# xtrace

借助ebpf技术，在不阻塞程序运行的前提下hook组件的帧处理函数, 打印输入/输出帧链上每一帧的属性。

按 Antares 组件规范，帧处理函数的第 2 个参数就是帧链（`Antares_frame_list_handle`），工具固定按帧链解析，链上的每一帧打印一行。

## 部署布局

工具依赖两样东西，都按**脚本所在目录**去找（整个目录可以整体挪走）：

    xtrace                  这个脚本
    lib/libAntares.so       指向 lib/libAntares.so.<版本> 的软链接
    lib/libAntares.so.1.21.0
    Plugins/*.apf.*         各个组件

- **`Plugins/`**：`--func` 唯一确定了 hook 哪个插件，所以**不需要**指定 .so 路径。定位规则见下。
- **`lib/`**：属性索引和枚举名都从 `libAntares` 的调试信息里读，不手抄头文件。

### `--func` 怎么定位到插件

各插件的帧处理函数都叫 `<模块名>_frame`，插件文件叫 `<模块名>.apf.<版本>`，两者一一对应。所以：

1. 由 `--func` 去掉 `_frame` 后缀得到模块名；
2. 去找 `Plugins/<模块名>.apf.*`，找不到就报错，并给出拼写相近的候选。同一个模块有多个版本文件时报错。
3. 在这个插件自己的符号表里找**精确**等于 `--func` 的那个函数，取它的 mangled 名挂 probe。C 插件是裸名，C++ 插件是 `_Z`/`_ZL` 前缀，两种都认。

## 命令参数

| 参数 | 含义 |
| --- | --- |
| --func | 帧处理函数名，必须形如 `<模块名>_frame`（如 `ff_demuxer_frame`）。**不能写正则，也不必写 mangled 名** |
| --id | 组件 ID，精确匹配 `Antares_plugin_id_handle->id`（如 `ff_demuxer`）。不设置则捕获所有组件 |
| --ret | 默认 0，hook 函数入口；`--ret 1` hook 返回值（返回值同样是帧链） |

`--func` 必填。

## 输出格式

每帧一行，空格分隔的纯文本，不含颜色和光标控制字符，可直接重定向保存：

    ./xtrace --func ff_demuxer_frame > xtce.csv

第一行是表头，以 `#` 开头，grep/awk 可当注释跳过。表头挺长（数十个字段），因为列名用的是属性自己的名字：

    #dir ktime_ns proc tid qlen type size FRAME_WIDTH FRAME_HEIGHT FRAME_CSP FRAME_RATIO_NUM FRAME_RATIO_DEN FRAME_FPS_DEN FRAME_FPS_NUM FRAME_BITRATE FRAME_PID FRAME_CODEC_ID FRAME_KEY_FLAG FRAME_PROG_NUM FRAME_SAMPLE_RATE FRAME_SAMPLE_FMT FRAME_DATA_LOSS FRAME_CHANNEL_LAYOUT FRAME_PIC_STRUCTURE FRAME_PIC_TYPE FRAME_COLOR_PRIMARIES FRAME_COLOR_TRANSFER FRAME_COLOR_MATRIX FRAME_COLOR_RANGE FRAME_INIT_TIMEBASE_NUM FRAME_INIT_TIMEBASE_DEN FRAME_CUVA_PROVIDER_ORIENTED_CODE FRAME_SCTE35_COVERING FRAME_FORCE_IDR FRAME_FAKE FRAME_FIELD_ORDER FRAME_MONITOR FRAME_LONG_PTS FRAME_LONG_DTS FRAME_LONG_CAP_TIME FRAME_LONG_ABS_PLAY_TIME FRAME_LONG_PKT_TIMESTAMP id
    enter 987654321012 3421 3427 3 ANTARES_VIDEO 2764800 1920 1080 ANTARES_CSP_YUV420 16 9 1 25 0 256 ANTARES_CODEC_ID_H264 0 0 - - 0 - ANTARES_PICTURE_STRUCTURE_UNKNOWN 0 ANTARES_COL_PRI_RESERVED0 ANTARES_COL_TRC_RESERVED0 ANTARES_COL_MAT_RGB ANRARES_COL_RANGE_UNSPECIFIED - - 0 0 0 0 ANTARES_FIELD_PROGRESSIVE 0 1234567 -1 0 0 0 ff_demuxer
    enter 987654321012 3421 3427 3 ANTARES_AUDIO 2764800 - - - - - - - 0 256 ANTARES_CODEC_ID_AAC - 0 48000 ANTARES_SAMPLE_FMT_S16 0 ANTARES_CH_LAYOUT_STEREO - - - - - - 1 48000 - - - 0 - 0 100 0 0 0 0 ff_demuxer
    enter 987654321012 3421 3427 3 ANTARES_SUBTITTLE 2764800 - - - - - - - - 300 ANTARES_CODEC_ID_DVD_SUBTITLE - 0 - - - - - - - - - - - - - - - - - - - - - - -1 ff_demuxer

| 列 | 含义 |
| --- | --- |
| dir | `enter` 函数入口 / `exit` 返回值（`--ret 1` 时只有 exit） |
| ktime_ns | 采样时刻，内核单调时钟，单位 ns（不是墙上时间） |
| proc | 进程号 |
| tid | 线程号 |
| qlen | 本次调用时帧链上一共有多少帧 |
| type | 帧类型，**枚举名**，见 `Antares_type_e` |
| size | 帧数据缓冲区大小（`AntaresFrameData::buf_size_`，不是属性） |
| 中间的属性列 | 见下一节；列名就是属性名，顺序也取自那张表 |
| id | 组件 ID，放在最后一列（ID 可能含空格，`awk '{print $NF}'` 可直接取） |

### 中间的属性列

这一段的列**全部由脚本开头的 `FRAME_PROPERTY_COLUMNS` 表决定**，顺序就是表里的行序。
表里每个属性一行，写清楚四件事：**属性名**（要逐字等于头文件里的枚举项名，`FRAME_WIDTH` 这样）、怎么显示、按哪个枚举查名字、对哪些帧类型显示。要加列、减列、改显示方式，改那张表就够了，别的地方不用动。

显示方式有三种：打 `int` / 打 `int64` / 打**枚举名**。表里那个 tag 就是查名字用的枚举类型，写错了不会静默 —— 那一列退化成打数字，并在 stderr 点名是哪个 tag 没找到。

**列是定宽的**：一列对当前帧不适用时打 `-`，不是把这一格省掉。一个 demuxer 的帧链上视频、音频、字幕帧是混着的，行宽随类型伸缩的话日志就没法对齐着看，awk 的列号也不稳定。

帧类型分五类，由 `Antares_type_e` 决定：

| 类别 | `Antares_type_e` 里对应哪些取值 |
| --- | --- |
| v 视频 | `VIDEO` / `VIDEO_STREAM` / `VIDEO_MSDK` / `VIDEO_CUDA` / `VIDEO_EXTERNAL` / `VIDEO_AVFRAME` / `VIDEO_VPE` |
| a 音频 | `AUDIO` / `AUDIO_STREAM` |
| s 字幕 | `SUBTITTLE` / `SUB_STREAM` |
| d 数据 | `DATA` |
| o 其它 | 上面的都不算的（`NONE` / `IMAGE` / `TEXT` / `INFO` / `STREAM_END`），**以及头文件里将来新加的类型** |

所以比如音频帧只填表里标了 `a` 的那几列（`FRAME_SAMPLE_RATE` / `FRAME_SAMPLE_FMT` / `FRAME_CHANNEL_LAYOUT` / `FRAME_INIT_TIMEBASE_*` …），宽高那几列打 `-`。表里最后一列只有两种写法：`x`（任何帧都不显示），或 `v/a/s/d/o` 的组合（"avsdo" 就是这五类都显示）。写错了字母启动时会报出来，认得出的那几类照旧生效。

表里同一个属性写两遍也会报出来并只留第一行 —— 否则输出里会出现两个同名表头列，光看日志根本发现不了。

**DWARF 里出现了表里没有的帧属性**（上游往 `frame_property_e` 里加了新成员）时，工具不假装没看见：让它对所有帧类型都显示、打数字，排在最后几列，并在 stderr 提示补进表里。反过来，表里写了但 DWARF 里没有的属性只会让那一列不输出。

枚举名里**不含空格**，列结构不会被破坏。查到不认识的取值会写 `UNKNOWN(<数字>)`，便于发现上游加了新枚举。

注意以下特性：

- `FRAME_PIC_TYPE` 那一列本版 `libAntares` 的 DWARF 里**没有** `Antares_picture_type_t` / `Antares_picture_type_e` 这个类型，所以这一列退化成打数字。头文件里补上类型、或者把表里的 tag 改成实际存在的那个，就自动变回枚举名 —— 启动时 stderr 会说明。

## xtrace.enum

每次运行都会把从 `libAntares` 读到的枚举表按 C 的写法写进同目录的 **`xtrace.enum`**：

    enum Antares_csp_e
    {
        ANTARES_CSP_NONE = 0,
        ANTARES_CSP_YUV420,
        ANTARES_CSP_YUVJ420,
        ...
    };

**本工具用到的枚举全在里面**：必需的 （`frame_property_e` / `frame_property64_e` / `Antares_type_e`），加上 `FRAME_PROPERTY_COLUMNS` 第三列点过名的那些（`Antares_csp_e`、`Antares_codec_id_e`、`Antares_sample_fmt_e`、`Antares_channel_layout_e`、`Antares_pic_structure_e`、`Antares_ColorPrimaries` / `ColorTransfer` / `ColorMatrix`、`Antares_color_range_e`、`Antares_info_type_e`、`Antares_field_order_e`）。表里加一行、点了新枚举，这个文件跟着变，不用手抄。

把它和 `Antares_frame_define.h` 并排一看，就知道属性索引有没有平移、枚举值有没有被改过。
怀疑某一列对不上时，直接看这个文件就够了，不用重跑工具。

（表里点名的类型这一版确实没有——比如 `Antares_picture_type_t`——就不会出现：那一列已经退化成打数字、启动时也报过了，这里不重复。）

## 使用例子

### 查看组件入口、返回值

    ./xtrace --func ff_demuxer_frame
    ./xtrace --func ff_demuxer_frame --ret 1

由 `--func` 推出的模块名是 `ff_demuxer`，于是去 `Plugins/ff_demuxer.apf.*` 里找精确同名的那个函数。解析结果会先打到 stderr 给你确认：

    --func 'ff_demuxer_frame' -> Plugins/ff_demuxer.apf.2.1.2  (符号 _ZL16ff_demuxer_framePvS_)

写错时也会在 stderr 讲清楚：

    $ ./xtrace --func decoder_cpu_fram
    error: --func 必须形如 <模块名>_frame, 由 'decoder_cpu_fram' 推不出模块名。
      你是不是想传: decoder_cpu_frame, encoder_aud_frame, decoder_msdk_frame, ...

    $ ./xtrace --func ts_muxer_info_frame
    error: Plugins/ 下没有 'ts_muxer_info.apf.*' —— 由 --func 'ts_muxer_info_frame' 推出的模块名是 'ts_muxer_info'。
      你是不是想传: ts_muxer_frame, rtp_ts_muxer_frame, rtp_muxer_frame

### 只盯某一个组件实例

同一个 .so 可能被实例化出多个组件（多个流/多个节目），用 `--id` 精确指定：

    ./xtrace --func ff_demuxer_frame --id INPUT_0

## 注意事项

- 需要 root 运行（uprobe + bcc），需要 **python3**（脚本 shebang 指向 `/usr/bin/env python3`）。
- **需要 gdb**：枚举名、帧属性索引、帧结构成员偏移，全都是运行时从 `libAntares.so` 的 DWARF 里读的（约 0.2s，要求 `-g` 编译且没 strip），**没有别的来源**。板上没有 gdb 工具就用不了。
- 一次只能 hook 一个函数。
- 一条帧链最多记录 **16 帧**（受 eBPF 指令数上限限制，目标内核偏老）。`qlen` 大于 16 说明这条链被截断了。
- `--id` 取组件 ID 的路径是 `组件对象 + N -> Antares_plugin_id_handle -> +8 -> id 字符串`
- 帧属性列的**索引不用手工维护**：`FRAME_PROPERTY_COLUMNS` 表里只写属性**名字**（`FRAME_WIDTH` 这样），索引每次运行查 `frame_property_e` / `frame_property64_e` 拿。头文件里在中间插入属性会导致其后所有索引平移，工具自动跟着走，不会静默打印错列。
- **帧属性是整块读进来的**（`prop32_` 48 个 int + `prop64_` 14 个 int64，每帧两次 `bpf_probe_read_user`），"显示哪几列"的判断在 Python 侧做。逐个属性读不行：那个循环上面有 `#pragma unroll`，会展开成 16 份，几十次属性读取就把 4096 条指令的预算顶穿了。
- **帧结构的内存偏移也不用手工维护**：`type_` / `prop32_` / `prop64_` / `data_` / `AntaresFrameData::buf_size_` / `Antares_frame_list_::deqFrameSize`、帧链那个 `std::deque` 的 `_M_start` / `_M_finish` 位置，以及两个属性数组的长度，都是每次运行从 `libAntares` 的 DWARF 里读的。启动时会打到 stderr：

      帧布局(从 ./lib/libAntares.so 的 DWARF 读):
          sizeof(AntaresFrame) = 408
          type_   +8     prop32_ +12   (x48)  prop64_ +208  (x14)
          data_   +400   AntaresFrameData::buf_size_ +8
          Antares_frame_list_::deqFrameSize +88
          std::deque 帧链: _M_start +16  _M_finish +48  (每块几个元素运行时从 deque 量)
          组件 id: hdl +0    (encoder_aud.apf 查表)
          frame_property_e->MAX_FRAME_ELEMENT=48、frame_property64_e->MAX_FRAME_ELEMENT64=14

  头文件里改一个数组长度，它后面的成员会整体平移（例如 `MAX_FRAME_ELEMENT64` 从 14 变成 13 会让 `data_` 的偏移平移），工具跟着 DWARF 走就不会错。
- 帧链是 `std::deque<Antares_frame_handle>`。它的 `_M_start` / `_M_finish` 的**位置**是成员偏移，和上面那批一样从 DWARF 读（连 libstdc++ 的内部字段一起问，拿到的是那个 `.so` 编译时用的头文件的真实布局）。而**每个块放几个元素**这个数 DWARF 里确实没有 —— 它是 `<bits/stl_deque.h>` 里的编译期常量 `_GLIBCXX_DEQUE_BUF_SIZE`（512 字节）/ `sizeof(元素)` 算出来的，没有任何成员承载它。工具不猜这个数，改成**从这条链自己量**：起点迭代器的 `_M_last - _M_first` 恒等于一整块的字节数（`_M_first` 永远指块首），见 `proc_antares_list()` 里的 `block_bytes`。所以元素多大、libstdc++ 哪个版本、32 位还是 64 位，都自动对。（这段只在一条链超过一个块、需要跨块时才用到；每块 64 个元素而一次最多记 16 帧，实际上跑不到，是防御性的。）
- 抓帧前的闸门查的是**自洽性**：DWARF 里两个属性数组的长度必须等于枚举里的 `MAX_FRAME_ELEMENT` / `MAX_FRAME_ELEMENT64`。不等说明枚举和结构体不是同一份头文件编出来的（按枚举索引读数组会越界），这时直接报错而不是输出错数据（新的枚举表仍会照常写进 `xtrace.enum`，可以先看那个）。
- **`lib/` 和 `Plugins/` 必须来自同一修订**。布局以 `lib/libAntares.so` 的 DWARF 为准 —— 帧对象由 libAntares 分配，插件只传指针。用 ASan 构建还是普通构建无所谓（实测两者布局逐字节相同），但整套要一致。
- `size` 列取自帧的数据缓冲区（`AntaresFrame::data_`），不是枚举里的属性。
- 输出逐行即时 flush，Ctrl-C 中断不会丢已打印的内容。
