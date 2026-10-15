# NOTES — xtrace 的设计与取舍

本文件记录 xtrace **当前**的工作方式，以及每一项取舍的理由，给下一个改这个脚本的人看。
怎么用看 `readme.md`；这里只写"为什么是这样"。

---

## 1. 文件与前提

| 文件 | 说明 |
| --- | --- |
| `xtrace` | 工具本体，单文件 Python3 + BCC/eBPF |
| `readme.md` | 面向使用者 |
| `xtrace.enum` | **每次运行自动重写**：从 libAntares 读到的枚举表（C 风格），供事后和头文件对拍 |
| `lib/libAntares.so` | → `libAntares.so.1.21.0` 的软链接。枚举与帧结构偏移的**唯一**来源 |
| `Plugins/` | 插件目录；`--func` 靠它定位 |

**目标是 python3**（shebang `#!/usr/bin/env python3`，文件已 +x，可直接 `./xtrace`），不为 py2 妥协。
但**板上是老内核 + 老 bcc**，所以 bcc / 内核层面的保守写法必须保留。

---

## 2. 对外契约

hook 一个组件的帧处理函数，把它输入/输出**帧链**上每一帧的属性打成一列文本。

依据 `Antares/include/Antares/component/Antares_plugin.h`：

```c
typedef Antares_frame_list_handle (*Antares_plugin_fn_f)(void *hdl, Antares_frame_list_handle in_framelist);
```

**PARM2 恒为帧链** —— 整个工具只按帧链解析。

CLI **只有三个参数**：

| 参数 | 含义 |
| --- | --- |
| `--func` / `-f` | **必填**。源级函数名，必须形如 `<模块名>_frame`。**不能写正则，也不必写 mangled 名** |
| `--id` / `-i` | 精确匹配组件 ID；不设置则全捕获 |
| `--ret` / `-r` | 0 = 入口（默认），1 = 返回值 |


输出：一行一帧，空格分隔，无 ANSI 转义，逐行 flush；第一行 `#` 表头可当注释跳过。

固定 7 个前缀列（`#dir ktime_ns proc tid qlen type size`）+ 属性列 + 末尾的 `id`。表头挺长，因为**列名就是属性自己的名字**，可以拿头文件逐行对读：

```
#dir ktime_ns proc tid qlen type size FRAME_WIDTH FRAME_HEIGHT ... FRAME_LONG_PKT_TIMESTAMP id
enter 987654321012 3421 3427 3 ANTARES_VIDEO 2764800 1920 1080 ... 0 1234567 -1 0 0 0 ff_demuxer
enter 987654321012 3421 3427 3 ANTARES_AUDIO 2764800 - - ... - 0 100 0 0 0 0 ff_demuxer
```

- `id` 放在**最后一列**（ID 可能含空格，`awk '{print $NF}'` 才安全）。
- **中间的属性列整段由 DWARF 和 `FRAME_PROPERTY_COLUMNS` 表决定**：哪些属性、什么顺序、哪几类帧显示，
  改那张表就够了。不在表里的属性（DWARF 里新加的）排在最后，对所有帧类型都显示。
- 固定那几列是**一个列名对一个 `struct event` 字段**（打的字未必同：`dir`←`enter`、`ktime_ns`←
  `timestamp`、`proc`←`process_id`、`size`←`data_size`）。两边靠 `b["events"].event(data)` 生成的
  ctypes 类绑在一起，改 C 结构就得同步改 Python；**最险的是调换两个同类型字段的顺序** ——
  名字都在、类型也对，读出来却是另一个字段的值，不报错。
- 属性列**不在这份定义里**。`FRAME_WIDTH` / `FRAME_CSP` 只活在 `prop32[48]` / `prop64[14]` 两个整块
  数组里：`read_cell()` 先按名取数组、再按下标取格，而下标是运行时查 DWARF 查到的，跟 C 结构无关。
  所以**加/删/改属性列只动 `FRAME_PROPERTY_COLUMNS`**，BPF 一个字不用改，也不用重过 verifier。
- 统计与告警一律走 **stderr**；stdout 只有表头 + 数据行。
- `type` 是**枚举名**（`Antares_type_e`），不是数字；`FRAME_CSP` / `FRAME_CODEC_ID` /
  `FRAME_SAMPLE_FMT` / `FRAME_CHANNEL_LAYOUT` / `FRAME_COLOR_*` 等列同理，按表里点的 tag 查名。
  查不到写 `UNKNOWN(v)`（无空格，列结构不变）。
- 一列对当前帧不适用时打 `-`，**不是把这一格省掉**：一条链上视频/音频/字幕帧是混着的，
  列数恒定才能对齐着看，awk 的列号也才稳定。
- 数字列是**有符号**的（只要是来自 `prop32_` 的列都是 `s32`）。

---

## 3. 内存布局

### 3.1 从哪来：全部从 libAntares 的 DWARF 读

**枚举索引、帧结构成员偏移（含 `std::deque` 的 `_M_start` / `_M_finish` 位置）、两个属性数组的
长度，都在启动时用 gdb 从 `lib/libAntares.so` 的 DWARF 里读**（`extract_dwarf()`，一次调用全拿完，
约 0.2s），不手抄头文件。读到的值注入成 BPF C 的 `#define`（`build_layout_defines()`），
并在 stderr 打一份供核对。

**为什么必须这样**：头文件里在枚举中间插一个属性，会让其后所有索引平移；改一个数组长度
（`MAX_FRAME_ELEMENT64` 14→13），会让它后面的成员整体平移 8 字节（如`data_`成员）。
这两种变化**抓帧是看不出假象的，只会静默输出错列**。跟着 DWARF 走就不会。

**代价与前提**：

- `lib/` 和 `Plugins/` 必须来自**同一修订**（用户约定：实际部署要么全 ASan、要么全普通）。
  ASan 与普通构建的布局**逐字节相同**（已实测），两者可互换。
- **gdb 是硬依赖**，没有"先生成快照、再到没 gdb 的机器上跑"这条路。找不到 gdb、或 .so 被 strip
  → 报错退出，不静默降级。

### 3.2 `AntaresFrame`（64 位 Itanium C++ ABI）

取值依据 `include_internal/component/Antares_frame.hpp`。下表是 libAntares 1.21.0 的实际值，
写在这里只为阅读方便，运行时一律以 DWARF 为准：

| 偏移 | 内容 |
| --- | --- |
| +0 | vptr |
| +8 | `Antares_type_t type_`（4 字节） |
| +12 | `int prop32_[48]`（`frame_property_e`） |
| +208 | `int64_t prop64_[14]`（`frame_property64_e`；8 字节对齐，+12+192=+204 后有 4 字节填充） |
| +320 | `std::shared_ptr prop_volatile_mem_[2]`（每个 16 字节） |
| +352 / +368 / +384 | `custom_prop32_` / `custom_prop64_` / `custom_prop_volatile_mem_` |
| +400 | `AntaresFrameData *data_` |

帧数据缓冲区大小在 `AntaresFrameData::buf_size_`（1.21.0 里是 `data_+8`，读 `s32`）。
读出来大于 `1<<31` 说明偏移取到的不是缓冲区大小（被 hook 的二进制与头文件不同修订时可能发生），
此时输出哨兵 `0xffffffff` 而不是垃圾数字。

### 3.3 帧链 `Antares_frame_list_`

```c
struct Antares_frame_list_ {                      // include_internal/component/Antares_frame_list.hpp
    std::deque<Antares_frame_handle> deqFrame;    // +0
    size_t infoFrameSize;                         // +80
    size_t deqFrameSize;                          // +88  <- 帧数，就是 qlen 列
};
```

`deqFrameSize` 同时是 `Antares_frame_list_size()` 的返回值来源。

遍历 `std::deque`（libstdc++）。1.21.0 这份上 `_M_start` 落在 +16（cur/first/last/node =
+16/24/32/40）、`_M_finish` 在 +48；`cur` 走到 `last` 就顺 `_M_node + 8` 取下一个块。

`_M_start` / `_M_finish` 的**位置**是成员偏移，从 DWARF 读 —— 表达式直接写
`deqFrame._M_impl._M_start`（`_M_impl` 是 `protected`，gdb 不管访问控制）。读到的是那个 `.so`
**编译时用的** libstdc++ 头文件的真实布局，比写死准。

**每个块放几个元素**这个数则哪里都没有：它来自 `<bits/stl_deque.h>` 的编译期常量
`_GLIBCXX_DEQUE_BUF_SIZE`（512 字节）/ `sizeof(元素)`，没有任何成员承载它；本机这份 `.so` 也没带
`.debug_macro` 段（producer 是纯 `-g`，不是 `-g3`），所以 gdb 问不出来。**工具不猜它，改成从这条链
自己量**：

```c
/* _M_start._M_first 永远指块首, _M_last 被 libstdc++ 赋成 _M_first + _S_buffer_size(),
 * 所以这个差不随链上有几帧变, 恒等于一整块的字节数。两个值本来就读了, 不额外读内存。 */
u64 block_bytes = (u64)((char *)last - (char *)first);
```

于是元素多大、换哪个 libstdc++、32 位还是 64 位都自动对。这段只在一条链跨过块边界时才用到，
而每块 64 个元素、一次最多记 16 帧、遍历又总是从块首开始 —— 实际上跑不到，是防御性的。

### 3.4 组件 ID 的取值路径

```c
typedef struct Antares_plugin_id_interface {   // include/Antares/common/Antares_define.h
    const char *name;   // +0
    const char *id;     // +8
} *Antares_plugin_id_handle;
```

路径：`hdl`(PARM1) `+ PLUGIN_ID_OFF` → `Antares_plugin_id_handle` → `+8` → id 字符串。

**约定是"组件对象的第一个数据成员就是 `Antares_plugin_id_handle`"，但这个偏移逐组件不同。**
`PLUGIN_ID_OFF` 不能是一个固定的数（8），换组件就得手改。目前实现是按模块名查表。

#### 为什么不能是一个数

+8 只是"组件类带虚析构函数（`ff_demuxer.h: virtual ~FFDemuxer();`），编译器在 +0 放 vptr，第一个
数据成员 `id_` 紧跟在后"这一个情形。实测 57 个插件，这条约定只对**大约一半**成立：

| 偏移 | 情形 | 例子 |
| --- | --- | --- |
| +0 | 析构非虚，没有 vptr（`abs_demuxer.h: ~AbsDemuxer();`） | 21 个，`AbsDemuxer` / `DecoderMsdk` / `HlsPublish` … |
| **+8** | 带虚析构，vptr 占 +0（**多数**，也是默认值） | `FFDemuxer` 等 27 个 |
| +16 | 继承链上还压着基类的字节 | `BlackMagicSend` / `SdiSend`（各继承两个 DeckLink COM 接口） |
| +48 / +112 | 组件自己把 id 摆在结构体别处 | `ts_muxer_obj_t` / `RtspPusher`（`_runningStatus` 在 +0） |

#### 表是怎么定出来的

一次性的离线核对，不是运行时逻辑。对插件逐个：

1. **反汇编 `<模块名>_frame`**，看尾部直接调用的那个 mangled 符号 —— `call <_ZN9FFDemuxer5FrameEPv@plt>`
   里就带着类名（Itanium ABI 的长度前缀，不需要解名工具）。这一条覆盖 **37** 个。
   C 风格插件的帧函数不是 thunk，这条路不成立，退回第 2 条。
2. **反汇编 `<模块名>_create`**，认 `_ZN<类>C[1-3]E` 构造函数调用。再覆盖 **9** 个。
3. 剩下的从**帧函数调用到的那个函数在 DWARF 里的真签名**拿对象类型（C 函数的参数类型没有被冲掉）：
   `ts_muxer_obj_frame(ts_muxer_obj_t *)`、`dispatch_frame_to_writers(rtsp_publish_t *, …)`、
   `do_backup_before_source_arrival(audio_backup_t *, …)` 等，再查那个结构体里 id 成员的位置。覆盖 **5** 个。
4. `rtp_ts_demuxer` 单独一个：插件里唯一带 id 成员的结构体就是 `rtp_ts_demuxer_t`（+8）。

拿到类名后，偏移一律是 gdb 问 DWARF：`(long)&((类 *)0)->成员`，且**只认唯一一个类型是
`Antares_plugin_id_interface *` 的成员**（多个或没有就不下结论）。

**没进表的 若干个**（都走默认值 8），分两类：

- **没定出来**（4 个）：`rtp_demuxer` / `rtp_muxer` 的句柄类型是 `typedef void *`，类型信息断在这儿，
  帧函数也不是 thunk；`metrics_deal` / `http_send` 的帧函数自包含，不调用任何带对象类型的函数，
  构造函数也没认出来。这几个的 8 是**猜的**，不是查过的。
- **定不了**（1 个）：`post_analyse`。`PostAnalyse` 对象里（`_vptr` +0、`_ctx` +8）**根本没有
  `Antares_plugin_id_handle` 成员**，所以这个组件 `--id` 无论取什么偏移都命中不了 —— 这不是表没配全，
  是它真的不按这个机制暴露 id。

#### 运行时的行为

表里列的是**核过的插件**，所以"在表里"的语义是"这个组件查过"，
真值照表取；表里没有的按默认值 8 处理，那才是"没核过、猜的"。

⚠️ 表里的键是**模块名**（`--func` 去掉 `_frame`），不是符号名也不是插件文件名；表只在一处定义，
加组件就往里加一行。

换组件不用改脚本 —— `--func` 推出模块名后自动查表，生效值和出处打进启动的布局块：

```
    组件 id: hdl +48   (模块 ts_muxer 在表里)   <- 这个偏移不来自 DWARF, 是按模块名查表
    组件 id: hdl +8    (模块 metrics_deal 不在表里, 用默认值)   <- ...
```

`PLUGIN_ID_OFF` 仍是整个脚本**唯一**的非 DWARF 常量（帧布局、deque 迭代器位置、枚举表全都读 DWARF）
—— 表是照着 1.21.0 这一版的插件定的，**换大版本时要重新核一遍**：布局块里显示"在表里"的
那几个如果和新库对不上，`--id` 会静默漏掉。这是拿掉"每次手改一个数"换来的代价，接受它的理由是
那张表是**静态**的，而其余所有偏移都由 DWARF 保证跟着库走。

读错的症状是 `--id` 一条都命中不了。**退出时的统计会如实反映**（`--id <id>: 命中 0 条, 未命中 N 条`），
过程中不告警 —— 中途告警在组件交错时必然是误报。

### 3.5 属性列

⚠️ `frame_property_e` 要求**从 0 开始连续定义**：在枚举中间插入属性会让其后所有属性的索引平移。
所以索引一律从 DWARF 查，绝不手抄。

**显示哪些属性、按什么顺序、哪几类帧显示，全由脚本开头的 `FRAME_PROPERTY_COLUMNS` 决定**，
每个属性一行四项元组：

| 项 | 含义 |
| --- | --- |
| 属性名 | 必须**逐字等于** `frame_property_e` / `frame_property64_e` 里的枚举项名（`FRAME_WIDTH` 这样）。索引是拿这个名字现查 DWARF 得到的 |
| 取值方式 | `int` / `int64` / `enum` |
| 枚举 tag | 取值方式为 `enum` 时用它查名字（`Antares_csp_e` 这样）；其它写 `None` |
| 适用帧类别 | `x`（**任何帧都不输出**，但名字算被认领），或 `v`/`a`/`s`/`d`/`o` 的组合 |

一个属性可以同时适用几类，比如 `FRAME_PID` 是 `avsd`

**列名就是属性名**，输出列直接能和头文件逐行对读，表里的名字和 DWARF 里的名字是同一套。

三条容错（都在启动时报到 stderr，不静默）：

- **类别字母写错**（不在 `v/a/s/d/o` 里）：报出来，认得出的那几类照旧生效。
- **同一个属性写两遍**：报出来，只留第一行 —— 否则会产出两个同名表头列，光看日志发现不了。
- **枚举 tag 写错**（DWARF 里没有这个类型）：那一列退化成打数字，并点名是哪个 tag 没找到。

表里有、DWARF 里没有 → 那列不输出；DWARF 里有、表里没有
（上游往枚举里加了新属性）→ 对所有帧类型都显示，补在最后几列，列名用它自己的枚举项名。

**索引每次运行现查**，所以头文件中间插属性导致的整体平移自动跟着走，不会静默打印错列。

帧类型 → 类别按**枚举名**对（`TYPE_CATEGORIES`），不按数值，所以头文件里加一个类型让数值平移了
也不会认错。不在表里的类型（`NONE` / `IMAGE` / `TEXT` / `INFO` / `STREAM_END`）
一律算"其它"（`o`）。

⚠️ 数 `frame_property64_e` 的槽位数时的坑：`FRAME_UID = FRAME_INFO_UID` 是**别名**，不要数成两个
（真实槽位 14 个，最大索引 13）。数错会让后面成员的推算偏移对不上。表里用的是 `FRAME_INFO_UID`
那一个名字，另外一个名字在 `NOT_A_PROPERTY` 里，免得被当成"表里漏了的新属性"。

`FRAME_MONITOR` 那列**保持数字**（表里写的是 `int`，不是 `enum`）：目前`Antares_monitor_and_e`
不在任何 libAntares 里（只在用到它的插件里）。留作将来改进。

`FRAME_PIC_TYPE` 那列这一版**退化成打数字**：`Antares_picture_type_t` / `Antares_picture_type_e`
这两个类型在本版 `libAntares` 的 DWARF 里都没有（头文件里有，但没有 TU 用它，gcc 没生成）。
这是上面"tag 写错"那条容错的唯一实例，启动时 stderr 会说明；头文件里补上类型就自动变回枚举名。

`FRAME_LONG_PTS` / `FRAME_LONG_DTS` 这类"没设置"时读到的是 **0，不是 -1**：`AntaresFrame`
构造时把整个 `prop64_` 清 0，只有 `FRAME_LONG_PKT_TIMESTAMP` 被置 -1。插件显式设过 -1 才看得见 -1。

---

## 4. 运行方式与理由

### 4.1 `--func` 怎么定位到插件

各插件的帧处理函数都叫 `<模块名>_frame`，插件文件叫 `<模块名>.apf.<版本>`，两者一一对应
（实测 57/57）。所以：

1. `--func` 去掉 `_frame` 后缀得模块名；
2. 只去看 `Plugins/<模块名>.apf.*`。找不到 → 报错，并用 `difflib` 给出拼写相近的候选
   （候选给成**带 `_frame` 的可用形态**）；同一模块有多个版本文件 → 报错列出，不替你挑；
3. 在**这一个**插件自己的符号表里找源码级名字**精确**等于 `--func` 的函数（单次
   `nm --defined-only`，约 0.02s），取它的 mangled 名挂 probe。

因此 `--func` 只能是字面函数名。**代价是同插件里不叫 `<模块名>_frame` 的函数变得不可达**，
这是有意接受的：本工具的前提是被挂函数的第 2 个参数就是帧链，只有组件注册的帧处理函数满足这个
ABI。

三种符号形态（实测每个插件恰好 1 个 `<模块名>_frame`，无缺失无重复）：

| 形态 | 个数 | 例 |
| --- | --- | --- |
| C++ static（`_ZL<len><名字>PvS_`） | 48 | `_ZL16ff_demuxer_framePvS_` |
| C++ 全局（`_Z<len><名字>PvS_`） | 3 | `_Z17decoder_cpu_framePvS_` |
| C 裸名 | 6 | `ts_muxer_frame` |

解析 mangled 名不需要解名工具：按 Itanium 的长度前缀取名字比对即可，签名原样保留。
这些符号只在 `.symtab`（`nm -D` 一个都看不到），**实测 bcc 挂得上**。

### 4.2 bcc 的 `sym_re` 是**整串匹配**

实测（`ff_demuxer.apf.2.1.2`）：

| `sym_re` | 挂上的 probe 数 |
| --- | --- |
| `ff_demuxer_frame` | 0 |
| `ff_demuxer_framePvS_` | 0 |
| `_ZL16ff_demuxer_framePvS_` | 1 |
| `^_ZL16ff_demuxer_framePvS_$` | 1 |
| `.*ff_demuxer_frame.*` | 1 |

所以把用户的 `ff_demuxer_frame` 原样交给 bcc 是挂不上的。工具自己解析出精确符号，再用
`"^" + ere_quote(symbol) + "$"` 挂载 —— 这样在"整串匹配"和"非锚定搜索"两种语义下都成立，
也不要求用户写 mangled 名。同一组实测也确认了 bcc **会扫 `.symtab`**（`ff_demuxer_frame` 是
static 局部符号），这是整套接口成立的前提。

⚠️ 转义**不用** `re.escape`：bcc 用的是 POSIX 正则，不是 Python 的 `re`，`re.escape` 会多转义
`-` `&` `~` `#` 空格 这类在 POSIX ERE 里没有意义的字符（`\_` `\&` 之类是未定义行为）。
手写元字符表 `ERE_META`，只转义真正有意义的。

### 4.3 枚举表每次运行写 `xtrace.enum`

每次运行都重新调 gdb 读一遍，读到的表按 C 的写法落到 `<脚本目录>/xtrace.enum`。规则：4 空格缩进、
一行一个、行尾逗号；**仅当是首个枚举值、或值不等于前一项 +1 时才打印 `= v`**（连续的隐式值省略，
正是 C 的语义，也让"中间插入属性"这类改动一眼可见）。

**只写 `enum X { … };`，不写头文件收尾的那个 typedef 别名。** 别名是纯装饰，而且头文件里
**没有统一规律** —— 多数是 `typedef enum X_e {…} X_t;`，但 `Antares_info_type_e` 的 tag 和
typedef 同名（没有 `_t`），`Antares_color_range_e` 的 tag 其实叫 `Antares_color_range_`。
原先是从 DWARF 里查别名、查不到就按 `_e → _t` 推导，结果是：14 个枚举里**只查得出 1 个**
（`Antares_type_t`，且和推导结果一字不差），而推不出来的那 2 个正好推出**错名字**。
去掉这个位置之后就没有"对不对"这回事了，14 个枚举一视同仁。

**落盘的是 `enum_tags()` 的全集**，也就是"本工具用到的每一个枚举"：必需的 3 个
（`frame_property_e` / `frame_property64_e` / `Antares_type_e`）加上 `FRAME_PROPERTY_COLUMNS`
第 3 列点过名的那些。表里点名、但这份 DWARF 里真没有的类型（`Antares_picture_type_t`）
跳过，不在文件里出现（那一列已经退化成数字、启动时也报过了，不重复）。

**为什么落文件而不是打屏**：这是"工具认的枚举"的唯一可查记录。事后怀疑某列错位时，直接和
`Antares_frame_define.h` 对拍这个文件就够了，不用重跑工具；打屏则每次都要重定向、还会混进日志。
stderr 只留两行元信息（来源路径 + 写入路径）。写不进去（只读挂载点等）**不致命**：告警一次并把表
改打到 stderr，信息不丢，工具照跑。

不做缓存：提取仅 0.2s，缓存换来的是"陈旧缓存"这个新失败面。

### 4.4 布局闸门

查的是**自洽性**：DWARF 里 `prop32_` / `prop64_` 的元素个数，必须等于枚举里的
`MAX_FRAME_ELEMENT` / `MAX_FRAME_ELEMENT64`。不等说明 `libAntares.so` 或者本脚本存在BUG。

⚠️ 调用顺序：`check_layout` 排在 `write_enum_dump` **之后**。布局一变就该拦，但枚举表必须先落盘 ——
报错信息里也就顺手指向那个文件，让"先看看新枚举表长什么样"这条路还通。

（不拿写死的值和 DWARF 比：那样既会误伤，又漏得掉真正的改动 —— `MAX_FRAME_ELEMENT64` 14→13 时
`MAX_FRAME_ELEMENT` 仍是 48，写死的闸门会直接放行。）

### 4.5 命名映射只在 Python 侧

**BPF 侧一行都不改**（省 4096 指令预算，不碰 verifier 风险）：`format_cell()` 只对 `kind == "enum"`
的列，拿这一列的 `tag` 去 `name_maps` 里查一次名字，`type` 那列也不例外（它就是表里 tag 为
`Antares_type_e` 的一个普通 `enum` 列）。所以**不再有"对某几列特别处理"的硬编码** ——
表里把哪一列的 tag 点成 `enum`，哪一列就查名字。

`name_maps` 按 **tag 名**索引，每张表必须是**值→名字**：JSON 把 `(name, value)` 元组变成
`[name, value]` 列表，`dict(pairs)` 建出来的是名字→值，反查会**全部落空、整列变成 UNKNOWN**。

⚠️ 还有一条同源的老坑：查名字用的枚举如果**没人去 DWARF 里取**（只列在 `NEEDED_ENUMS` 之外），
那一列的 `tag` 查不到，`build_frame_columns` 会把它退化成打数字 —— 而且只报一条告警。
现在 tag 一律从表的第 3 列现推（`enum_tags()`），不会再漏。

⚠️ gdb 脚本自身是 Python 字符串拼出来的，新加的代码里出现 `%s` 会和外层的百分号格式化打架
（`TypeError: not enough arguments for format string`）。所以那个脚本里**一个百分号都不留**，
用字符串拼接 —— 比逐个转义成 `%%` 稳。

---

## 5. 关键设计决策

- **`--ret 1` 怎么拿到组件 ID**：uretprobe 读不到第 1 个参数。入口 probe **恒挂**，按 tid 把 hdl
  存进 `BPF_HASH(hdl_by_tid, u32, u64, 4096)`，出口 probe 查回来。入口是否提交事件由注入的编译期
  开关 `EMIT_ON_ENTER` 控制（`--ret 1` 时为 0，入口只缓存不提交）。不删 hash 条目，条数以线程数为界。
- **`--id` 过滤放在 Python 侧**：BPF 里做字符串比较要注入字面量 + 逐字节比较，在 4096 指令预算下
  不划算。代价是未命中的事件也会走 perf buffer，所以 `page_cnt=64`。
- **编译期开关 / 索引 / 偏移一律用前置 `#define` 注入**，不用 `.replace()` 占位符 —— 漏替换会让 C
  编译不过，而本地无法编译 C。
- **帧链上限 16**（`MAX_FRAMES_PER_CALL`）：脚本里有 `bpf_probe_read_user` → `bpf_probe_read` 的低内核
  兼容宏，说明目标是 pre-5.5（大概率 4.14/4.19 LTS），这些内核 `BPF_MAXINSNS = 4096` 且 verifier
  不支持有界循环（5.3+）。16 帧约 1300~1900 条指令安全，**64 帧会 "program too large"**。
  截断由 Python 侧按 `qlen > 16` 告警一次。
- **帧属性整块读，不逐个属性读**（`fill_frame`）：`prop32_` 48 个 int + `prop64_` 14 个 int64
  各一次 `bpf_probe_read_user`（304 字节），每帧总共两次。逐个读是 `每次调用约 6~8 条指令 ×
  35 列 × 16 帧`，在 `#pragma unroll` 展开后**直接顶穿 4096 指令预算**。还有一个更根本的理由：
  **"这一帧该显示哪几列"是随帧类型变的，C 里做不出来那个判断**（要按类别查一遍表），
  所以整块搬到用户态，判断交给 Python。
- **保留 `BPF_ARRAY(m_event, struct event, 1)` 作 scratch**（板上已验证；注意它是**普通数组，不是
  per-CPU**，多核共享，窗口极小但存在）。每次 `perf_submit` 拷贝整个结构，所以每次都必须
  `__builtin_memset` 清零，否则泄漏内核栈内容。
- **probe 数检查用 `matched > expected` 而不是 `!=`**：`num_open_uprobes()` 在不同 bcc 版本是否统计
  uretprobe 不确定，用 `>` 可以避免 `--ret 1` 下误报。
- **`lost_cb(*args)`**：不同 bcc 版本回调参数个数不一样，取最后一个 int，认不出按 1 条计。
  **静默丢事件是这个工具最坏的失败模式**，所以退出时一定打印。
- **`--id` 的过滤结果只在退出时统计一次**（`report_stats()`：`--id <id>: 命中 N 条, 未命中 M 条`），
  过程中**不告警** —— 组件交错时中途告警必然是误报，而退出时的真实计数已经足够判断 ID 写错
  或 `PLUGIN_ID_OFF` 取的不对（命中 0 条）—— 后者在布局块里显示成"不在表里, 用默认值"。
- **`std::deque` 的块大小不写死，从活的 deque 量**：`_M_last - _M_first` 恒等于一整块的
  字节数。DWARF 表达不了这个编译期常量，而写死它意味着换 libstdc++ / 换元素大小 / 换位数都可能
  静默读错；量出来的值不额外读一次内存，且自动正确。
- **输出契约**：`#` 表头；属性列定宽、不适用打 `-`；id 在最后一列；每行 `sys.stdout.flush()`
  （重定向到文件时 stdout 全缓冲，不 flush 会丢掉最后一段）；统计/告警走 stderr。
  表头和数据行的列名都取自同一张 `FRAME_PROPERTY_COLUMNS`，所以两者不会对不上。

---

## 6. 已验证 / 未验证

**已在 Linux 本机实跑验证**（都是真跑，不是推演）：

1. `python3 -m py_compile` 通过；`./xtrace`（走 shebang）也能直接跑。
2. **枚举表与权威头文件逐项一致**：`xtrace.enum` 里落盘的 **14 个枚举、291 项全部一致**。
   ⚠️ 两个对拍本身的坑：一是要先剥掉头文件里的多行 `/* */` 注释再按逗号切分，
   否则会算出一堆假差异；二是**头文件里 tag 名和 typedef 名不一定同名**
   （`typedef enum Antares_color_range_ {...} Antares_color_range_e;`），
   只按 tag 名建索引会把这个枚举漏掉、看起来像"dump 里多了一个"。
3. **插件定位全中**：三种符号形态各验一个；`ts_muxer_frame` 定位到的符号名里不含
   `_info_frame` / `_obj_frame`；6 条错误路径（`frame` / `Frame` / `_frame` / `ts_muxer_frame(` /
   mangled 名 / 不存在的模块）都给出可操作的报错；定位行在 stderr，不污染 stdout。
4. **来源解析**：`lib/` 改名 → 报错并指出试过的路径（不是崩栈）；建 `libAntares.so` 软链接 →
   优先命中它。
5. **端到端加载**：四种符号形态各跑一次真机，全部 BPF 加载通过（只有 bcc 自带的
   `__HAVE_BUILTIN_BSWAP*` macro-redefined warning）、表头正常输出、进入读循环直到 timeout（rc=124）。
   **即确认 bcc 能 attach 到 static 局部符号**。
6. **帧布局从 DWARF 派生**：布局块注入若干个 `#define`，其中基本来自 DWARF（含
   `DEQ_START_OFF` / `DEQ_FINISH_OFF`），与 gdb 独立读出的值逐个相同，`PLUGIN_ID_OFF`
   来自查表；注入后的 C 源码里**已无裸偏移**，`DEQ_BLOCK_ELEMS` 这个宏已不存在，
   跨块处用的是运行时量出来的 `block_bytes`。闸门的反向路径也测了（自洽放行；
   `MAX_FRAME_ELEMENT=49` vs 数组 48、缺 `FRAME_DATA_OFF`、缺 `DEQ_START_OFF`、缺 `FRAME_SIZE`
   → 都拦住并点名）。⚠️ 这个用例必须切出函数体单测 —— 枚举和数组来自同一个 .so，用真实文件
   构造不出不一致。
7. **`PLUGIN_ID_OFF` 按模块名查表**：`ts_muxer` / `abs_demuxer` / `rtsp_pusher` / `sdi_send`
   分别注入 48 / 0 / 112 / 16，不在表里的 `ff_demuxer` 注入默认 8；布局块里查表命中的显示
   "模块 X 在表里"、吃默认值的显示"不在表里, 用默认值"，两者能一眼分开。表的键逐个在
   `Plugins/` 下有对应 `.apf.*`，且表里不出现值等于默认值的项。
8. **ASan 与普通构建等价**：四个结构逐字节相同；用户提供的普通版整套实跑通过，stderr 的布局块与
   `xtrace.enum` 都和 ASan 部署逐行一致。
9. **属性表本身的容错**（离线构造表跑 `build_frame_columns`）：类别字母写错只丢那一类；表里少
   写的属性补到末尾；表里多写 DWARF 里没有的属性则那列不出现；
   **同一个属性写两遍只留第一行**；tag 写错那一列退化成 `int`。
10. **`xtrace.enum` 的覆盖度**（这是补掉的一个漏）：落盘集合 == `enum_tags()` 里查得到的那些、
   必需的 3 个在内、表里点过名且 DWARF 里有的**全在内**、每个枚举都能解析出名字和结尾、
   无空枚举、首成员带 `= 0`、连续值不重复写 `= N`。

**还没做、需要在目标板上验收的**：

1. **加载通过** —— bcc 接受 + verifier 通过。⚠️ 本机是 x86_64 通用内核 5.19，**目标板可能是
   aarch64 + 老内核**，本机的加载成功**不能**替代板上的验证。

**依赖**：python3、root（uprobe + bcc）、`nm`、**gdb**（运行时读 DWARF，硬依赖）。

---

## 7. 迁移注意事项
- 改工具时，`readme.md` 的参数表与输出列说明必须同步；**属性列的增删改一律只动
  `FRAME_PROPERTY_COLUMNS`**，动了它 `readme.md` 的表头样例、每类列数、
  `xtrace.enum` 的枚举清单都会跟着变，别漏。
