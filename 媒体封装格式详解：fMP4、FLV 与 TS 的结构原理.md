# 媒体封装格式详解：fMP4、FLV 与 TS 的结构原理

## 一、什么是媒体封装格式

### 1.1 封装格式的本质

编码与封装解决的是两层问题：

- **编码（codec）**把原始像素或 PCM 采样压缩成编码访问单元，例如 AVC/H.264、HEVC/H.265、AAC。

- **封装（container/mux）**把一个或多个编码流组织成可解析的字节结构，并描述轨道、时间戳、解码配置、随机访问点和数据位置，例如 MP4、FLV、MPEG-2 TS。

- **传输协议**负责把这些字节或媒体消息送到接收端，例如 HTTP、RTMP、SRT、RTP。RTMP 自身属于这一层，虽然它的音视频消息负载通常复用 FLV 数据模型。

同一组 H.264/AAC 编码访问单元可以在不重新编码的前提下，从 FLV 重封装为 fMP4 或 TS；但前提是目标容器支持该编码，并且时间戳、参数集和随机访问信息能够正确映射。只改文件后缀既不会完成重封装，也不会改变编码格式。

### 1.2 常见封装格式一览表

|格式/协议|规范体系|定位|典型应用|
|---|---|---|---|
|MP4|ISO/IEC 14496-12/14|ISO BMFF 文件封装|本地播放、点播|
|fMP4|ISO BMFF Movie Fragment|分片式文件/流封装|DASH、HLS、MSE|
|CMAF|ISO/IEC 23000-19|对 ISO BMFF 分片、轨道与切换关系的应用约束|跨 HLS/DASH 复用、低延迟分段媒体|
|FLV|Adobe FLV File Format|Tag 式文件/流封装|HTTP-FLV、RTMP 媒体负载模型|
|RTMP|Adobe RTMP Specification|基于 TCP 的消息与分块传输协议|直播推流、传统直播拉流|
|TS|ITU-T H.222.0 / ISO/IEC 13818-1|固定包长的传输流封装|广播、传统 HLS|
|Matroska|Matroska Specifications|EBML 容器|本地多轨存储|

### 1.3 封装格式的核心职责

- 关联编码数据与解码配置，例如 SPS/PPS、`AVCDecoderConfigurationRecord`、`AudioSpecificConfig`

- 表达解码时间与呈现时间，支持 B 帧重排序和音视频同步

- 描述样本边界、轨道归属、字节位置与随机访问点

- 组织视频、音频、字幕和元数据等多种轨道或基本流

容器只负责**表达**这些信息，不保证源数据一定正确。例如把普通 P 帧误标成关键帧，仍会导致 Seek 后花屏；给音频写错采样率或时间戳，容器本身也无法修复同步。

### 1.4 NALU：编码数据的承载单元

在深入各格式之前，先区分三个经常混淆的概念：

- **NALU（Network Abstraction Layer Unit）**：H.264/H.265 码流的语法承载单元，用于把 VCL 切片数据或 SPS、PPS、SEI 等非 VCL 数据映射到存储和传输系统。

- **Slice（切片）**：一幅图像的一部分编码数据；一幅图像可以由一个或多个 Slice 组成。

- **Access Unit（访问单元）**：解码一幅图像所需的一组 NALU，可能包含 AUD、SPS/PPS、SEI 和多个 Slice NALU。

因此，“一个 NALU 就是一帧”并不成立。对 AVC/HEVC 视频，容器中的一个 video sample 通常对应一个 Access Unit，一个 sample 内可以有多个 NALU；对 AAC，sample 通常对应一个音频访问单元。这里的 “sample” 是**容器时间轴上的数据单元**，并不等于 PCM 的单个采样点。下面的 1 字节 Header 仅适用于 H.264；H.265/HEVC 的 NAL Header 为 2 字节，类型字段也不同。

**H.264 NAL Header（1 字节）结构：**

- `forbidden_zero_bit`（1 bit）

- `nal_ref_idc`（2 bit）：0 表示该 NALU 不用于后续图像参考，非 0 表示参考优先级

- `nal_unit_type`（5 bit）：类型

**常见 `nal_unit_type`：**

|Type|名称|说明|
|---|---|---|
|1|non-IDR Slice|非 IDR 图像的切片；可能属于 I/P/B 图像|
|5|IDR Slice|IDR 图像的切片；随机访问图像也可能含多个 Slice|
|6|SEI|附加信息|
|7|SPS|序列参数集|
|8|PPS|图像参数集|
|9|AUD|Access Unit Delimiter|

#### 两种 NALU 封装方式：Annex B vs AVCC

这是理解各容器存储视频的核心区别。

|封装方式|NALU 前缀|使用场景|特点|
|---|---|---|---|
|Annex B|Start Code：`00 00 01` 或 `00 00 00 01`|裸流（`.h264`）、TS 中常见的 AVC 字节流|适合连续字节流，可从 Start Code 重新寻找 NALU 边界|
|长度前缀（工程中常称 AVCC 格式）|1/2/4 字节 Length（大端，通常 4 字节）|MP4/fMP4、传统 H.264 FLV|无需扫描 Start Code，可直接得知当前 NALU 长度|

**示例：同一个 100 字节 IDR NALU：**

```text
Annex B: 00 00 00 01 65 [99字节]
AVCC:    00 00 00 64 65 [99字节]  (0x64=100)

```

注意：Length 字段长度由 `AVCDecoderConfigurationRecord.lengthSizeMinusOne + 1` 决定，规范允许 1、2 或 4 字节，工程中通常为 4 字节；Length 值包含 NAL Header，但不包含 Length 字段自身。严格地说，`avcC` 是 Box 类型，AVCDecoderConfigurationRecord 是其负载记录，“AVCC 格式”只是工程界对长度前缀样本表示法的常用简称。

#### HEVC 的 NALU、参数集与随机访问点

HEVC/H.265 同样使用 NALU，但它不是“把 H.264 的类型号换一下”：

- HEVC NAL Header 固定为 2 字节，包含 `forbidden_zero_bit`、6 bit `nal_unit_type`、`nuh_layer_id` 和 `nuh_temporal_id_plus1`。最后一个字段不能为 0。
- 非 VCL 参数集包括 VPS（Type 32）、SPS（33）和 PPS（34）；AUD 为 35，Prefix/Suffix SEI 分别为 39/40。
- Type 16～23 属于 IRAP（Intra Random Access Point）范围，其中 16～18 是 BLA，19～20 是 IDR，21 是 CRA，22～23 保留。**IRAP 是类别，不能与 IDR 画等号。**

IDR 会切断对 IDR 之前图像的参考，是最直接的随机访问起点。CRA 自身不依赖此前图像，但它可以关联在解码顺序上位于 CRA 之后、呈现顺序上位于 CRA 之前的 RASL 图像；从 CRA 随机加入时，这些 RASL 图像可能不可解码并应按规范不输出。因此，“sample 含 CRA”不自动等于“从该 sample 起所有下载到的图像都可立即显示”，打包器还要正确处理 leading picture、SAP 类型和分段边界。

HEVC 在 TS 字节流中通常仍使用 Start Code；在 MP4/fMP4 中通常使用长度前缀，并由 `hvcC` 中的 `HEVCDecoderConfigurationRecord` 给出 NALU Length 长度及 VPS/SPS/PPS 等配置。Sample Entry 的名称还约束参数集位置：

- `hvc1`：解码所需参数集只放在 Sample Entry 的配置记录中，sample 不携带参数集更新。
- `hev1`：参数集可以位于 Sample Entry，也可以出现在 sample 内。

AVC 有对应的 `avc1`/`avc3` 区别。转封装时不能只复制 NALU：还必须根据 in-band 参数集的实际存在位置选择正确 Sample Entry，并在配置变化时更新初始化信息。把含 in-band VPS/SPS/PPS 的流误标成 `hvc1`，即使多数播放器“碰巧能播”，也不代表文件符合其信令语义。

---

## 二、fMP4（Fragmented MP4）格式详解

### 2.1 设计哲学

fMP4 不是与 MP4 平行的另一套容器，而是 ISO BMFF（ISO/IEC 14496-12）的 **Movie Fragment** 组织方式。传统非分片 MP4 的逐 sample 索引主要位于 `moov` 的样本表中；fMP4 则把后续样本的索引拆到每个 `moof`，媒体字节仍放在 `mdat`。这样初始化信息可以先到达，后续分片可以边生成、边传输、边解析。

“Movie Fragment、CMAF Fragment、Segment、Chunk”来自不同抽象层，不能只按 Box 外形无条件互换：

- **Movie Fragment** 是 ISO BMFF 结构概念，核心是一个 `moof` 及其引用的媒体数据。

- **Media Segment** 是 HLS、DASH 或 MSE 的交付概念，具体允许包含几个 Fragment 由相应格式约束决定。例如 W3C MSE 的 ISO BMFF media segment 定义为可选 `styp`、一个 `moof` 和一个或多个 `mdat`。

- **CMAF Fragment** 是可独立解码的连续媒体对象，也是 CMAF Track 的切换边界；**CMAF Segment** 是同一 Track 中一个或多个连续 CMAF Fragment 的组合。

- **CMAF Chunk** 是某个 CMAF Fragment 中一段连续 sample 子集，用于在整个 Fragment/Segment 完成前渐进交付；**HLS Partial Segment（Part）**是 HLS 清单与交付层的对象，可以用 CMAF Chunk 组织，但二者不是脱离上下文就永远一一等价的同义词。

同一个 `moof + mdat` 字节组合在不同地址边界和应用格式中可能被称为 Movie Fragment、CMAF Fragment 或 CMAF Chunk。实现应以所遵循规范的对象边界、独立解码约束和清单引用方式为准，而不是用“看见一个 `moof` 就统一叫 Chunk”的启发式规则。

#### Box 通用格式（ISO BMFF 基础）

fMP4 的所有数据都以 Box（也称 Atom）为单位组织。每个 Box 有统一的头部格式：

```text
┌──────────────────────────────────────────────────────────┐
│  Box Header                                              │
├────────────┬────────────┬────────────────────────────────┤
│  Size (4B) │  Type (4B) │  [Extended Size (8B)]          │
│  Box总大小  │  Box类型    │  仅当 Size==1 时存在            │
├────────────┴────────────┴────────────────────────────────┤
│  Box Data (变长)                                         │
│  可以是纯数据，也可以嵌套子 Box                             │
└──────────────────────────────────────────────────────────┘

```

#### Size 字段的三种含义

|Size 值|含义|说明|
|---|---|---|
|普通值（至少能容纳 Header）|Box 总字节数|包含 Header 本身；普通 8B Header 的合法 Box 至少为 8。如 Size=100，则整个 Box 占 100 字节|
|1|使用扩展大小|Header 后紧跟 8 字节 uint64 表示真实总大小；32 bit 不够时必须使用，也允许生产者主动选择|
|0|延伸到文件末尾|该 Box 必须是文件中的最后一个 Box，内容延伸到文件末尾；通常只用于顶层 `mdat`|

解析不可信输入时，还要验证 Box 大小至少容纳自身 Header、没有越过父 Box/文件边界，并用足够宽的无符号整数检查 `offset + size` 溢出。`Size=1` 时 Header 已扩为 16 字节；不能先用 32 位加法计算末尾再读取 64 位扩展大小。未知 Box 可以按合法 Size 跳过，但非法 Size 不能靠“搜索下一个看似 ASCII 的 Type”静默恢复。

#### Type 字段

4 个 ASCII 字符，标识 Box 类型。常见类型：

|Type|Hex|含义|
|---|---|---|
|ftyp|66 74 79 70|File Type Box — 品牌标识|
|moov|6D 6F 6F 76|Movie Box — 元数据容器（嵌套子 Box）|
|moof|6D 6F 6F 66|Movie Fragment Box — 片段元数据|
|mdat|6D 64 61 74|Media Data Box — 实际音视频数据|
|mvhd|6D 76 68 64|Movie Header — 全局时间信息|
|trak|74 72 61 6B|Track Box — 单个轨道|
|traf|74 72 61 66|Track Fragment — 片段中的轨道数据|
|tfdt|74 66 64 74|Track Fragment Decode Time — 时间基准|
|trun|74 72 75 6E|Track Run — 样本详细信息|

#### FullBox 变体

部分 Box 需要版本和标志位控制行为，称为 FullBox：

```text
┌────────────┬────────────┬─────────────┬─────────────┬────────┐
│  Size (4B) │  Type (4B) │ Version (1B)│  Flags (3B) │  Data  │
└────────────┴────────────┴─────────────┴─────────────┴────────┘

示例：tfdt (Track Fragment Decode Time)
  Version=0: base_media_decode_time 为 uint32（最大约 13 小时 @90kHz）
  Version=1: base_media_decode_time 为 uint64（几乎无限）

示例：trun (Track Run)
  Version=0: sample_composition_time_offset 为 uint32
  Version=1: sample_composition_time_offset 为 int32，可表达负偏移
  Flags 位控制每个 sample 携带哪些字段：
    0x000001: data-offset present
    0x000004: first-sample-flags present
    0x000100: sample-duration present
    0x000200: sample-size present
    0x000400: sample-flags present
    0x000800: sample-composition-time-offsets present

```

#### Hex Dump 示例：一个完整的 ftyp Box

```text
偏移   十六进制                              含义
0000:  00 00 00 20                          Size = 32 字节（整个Box）
0004:  66 74 79 70                          Type = "ftyp"
0008:  69 73 6F 35                          Major Brand = "iso5"
000C:  00 00 02 00                          Minor Version = 512
0010:  69 73 6F 35                          Compatible: "iso5"
0014:  69 73 6F 36                          Compatible: "iso6"
0018:  64 61 73 68                          Compatible: "dash"
001C:  6D 70 34 31                          Compatible: "mp41"

```

#### 容器 Box vs 叶子 Box

- **容器 Box**（moov, trak, mdia, minf, stbl, moof, traf）：Data 区域只包含子 Box，自身无直接数据

- **字段型 Box**（mvhd, tfdt, trun, mdat）：Data 区域按各自语法解释；其中 `mdat` 是不透明媒体字节，`trun` 等是结构化字段

`stsd`、`meta` 等 Box 既有自身字段，又包含 Sample Entry/子 Box，不能仅靠“容器或叶子”二分法解析。通用解析器先读取 Size 和 Type，再根据已知 Box 语法决定是读取字段、递归解析子 Box，还是把内容视为不透明数据；Size 也让解析器能够跳过不认识的 Box。

### 2.2 整体架构

```text
Init Segment (同一初始化配置下通常缓存复用)
  [ftyp] + [moov]

Media Segments (持续到达)
  [moof][mdat]  [moof][mdat]  [moof][mdat] ...

```

#### Track、sample 与 timescale

理解 fMP4 的关键不是记 Box 名，而是先建立四个对象之间的关系：

1. **Track**：一条独立的定时媒体序列，例如一条视频轨、一条 AAC 音轨。`track_ID` 在 `tkhd` 中声明，分片里的 `tfhd.track_ID` 用它关联回初始化段。
2. **Sample Entry**：描述 sample 的编码类型和初始化配置，例如 `avc1 + avcC` 或 `mp4a + esds`。`sample_description_index` 选择具体描述。
3. **Sample**：Track 时间轴上的最小寻址数据单元。视频 sample 通常是一个 Access Unit，音频 sample 通常是一个编码音频访问单元。
4. **Timescale**：每秒包含多少时间单位。若 `mdhd.timescale = S`，时间值 `T` 对应的秒数为：

$$
t_{\text{seconds}} = \frac{T}{S}
$$

Movie 有自己的 `mvhd.timescale`，每个 Track 也有自己的 `mdhd.timescale`。`tfdt`、sample duration 和 composition time offset 使用的是对应 Track 的媒体时基；不同 Track 的整数时间戳不能直接比较。Edit List（`edts/elst`）可以再把 Track 的媒体时间映射到 Movie 的呈现时间。

### 2.3 Init Segment 详解

**Box 层级：**

```text
[ftyp] — 品牌标识（iso5/dash/cmfc）
[moov]
  ├── [mvhd] — Movie 时间轴的 timescale
  ├── [trak] — 视频轨道
  │     └── [mdia]
  │           ├── [mdhd] — 本轨 timescale；tfdt/duration 使用这个时基
  │           └── [minf] → [stbl]
  │                 ├── [stsd] → [avc1] → [avcC] — SPS/PPS
  │                 └── [stts/ctts/stsc/stsz/stco...] — 分段流初始化段中为空样本表
  ├── [trak] — 音频轨道
  │     └── [mdia] → [mdhd] + [minf] → [stbl] → [stsd] → [mp4a] → [esds]
  │                                             └── AudioSpecificConfig
  └── [mvex]
        └── [trex] — 标记为 fragmented 模式并提供默认 sample 参数

```

在 W3C MSE 和 HLS 的分段媒体约束下，初始化段不携带 sample，相关样本表的 entry count 必须为 0，`moov` 还需包含 `mvex`。HLS 进一步要求 `mvhd` 与 `tkhd` 的 duration 为 0，且 `mvex` 位于最后一个 `trak` 之后。一般 ISO BMFF 文件则可以先包含非分片样本、再追加 Fragment，不能把这些流媒体约束外推到所有 fMP4 文件。

视频 timescale 常取 90000，音频 timescale 常直接取采样率，但这只是常见选择，不是固定要求。编码参数、Sample Entry 或加密配置变化时，清单可能切换到新的初始化段，因此“Init Segment 只下载一次”应理解为“同一配置期间可以缓存复用”。

`ftyp`/`styp` 中的 brand 是内容对某套规则的**兼容性声明**，不是看到 `iso6`、`cmfc` 或 `dash` 就能忽略其余 Box 和 codec 配置。播放器仍需结合 Sample Entry、RFC 6381 `CODECS` 字符串、加密方案和自身解码能力判断是否可播；错误地声明兼容 brand 不会把不合规内容自动变得合规。

#### avcC Box：视频解码器的“说明书”（AVCDecoderConfigurationRecord）

|字段|大小|含义|
|---|---|---|
|configurationVersion|1B|固定 0x01|
|AVCProfileIndication|1B|如 0x64 = High Profile|
|`profile_compatibility`|1B|兼容性标志|
|AVCLevelIndication|1B|如 0x1F = Level 3.1|
|reserved(6bit)+lengthSizeMinusOne(2bit)|1B|NALU Length 字段=4字节时值为 0xFF|
|reserved(3bit)+numOfSPS(5bit)|1B|通常 0xE1 (1个SPS)|
|spsLength|2B|SPS 长度|
|spsNALU|变长|完整 SPS (含 NAL Header 0x67)|
|numOfPPS|1B|通常 0x01|
|ppsLength|2B|PPS 长度|
|ppsNALU|变长|完整 PPS (含 NAL Header 0x68)|

上表展示基础记录；部分 High Profile 配置还会在后面携带色度格式、位深和 SPS extension 等扩展字段。解析器应按 Profile 和记录长度继续解析或安全跳过，不能假设 PPS 之后必然立刻结束。

**Hex Dump 示例（1080p High@3.1）:**

```text
01 64 00 1F FF E1 00 1A 67 64 00 1F AC D9 40 50
05 BB 01 10 00 00 03 00 10 00 00 03 03 C0 F1 62
EE 01 00 04 68 EB E3 CB

```

#### esds Box：音频解码器的“说明书”

- `esds` 是一组 MPEG-4 Descriptor 的容器，其中 `DecoderSpecificInfo` 可以承载 AAC `AudioSpecificConfig`（ASC）

- ASC 的基础读取顺序是 `audioObjectType`、`samplingFrequencyIndex` 和 `channelConfiguration`。`audioObjectType=31` 时还要读取扩展 6 bit；`samplingFrequencyIndex=0xF` 时，真实频率由后续 24 bit 显式给出；`channelConfiguration=0` 不是“零声道”，而是声道布局要从 Program Config Element 等后续配置取得。

- 典型 2 字节 `12 10` 可拆成 AOT 2（AAC-LC）、频率索引 4（44.1 kHz）、声道配置 2（双声道）以及后续 GA 配置位。只有在这些默认条件成立时，ASC 才恰好短到 2 字节，解析器不能固定读取两个字节。

- HE-AAC/HE-AACv2 还涉及 SBR/PS 扩展、扩展采样率和底层 AOT；扩展既可能显式位于前部，也可能通过同步扩展表达。容器时基、解码器输出采样率、AAC core 采样率和每个访问单元的输出采样数必须按完整 ASC 解释，不能只看前五位 AOT。

- `esds` 的完整负载不等于 ASC；转封装时应抽取其中的 ASC，而不是把整个 `esds` 复制到 FLV

### 2.4 Media Segment 详解

```text
[styp] — 段类型标识
[prft] — 可选；把媒体时间映射到 NTP 墙钟时间
[moof]
  ├── [mfhd] — sequence_number
  └── [traf]
        ├── [tfhd] — track_ID + 默认属性
        ├── [tfdt] — base_media_decode_time（该轨解码时间轴的起点）
        └── [trun] — sample 数量以及按 flags 出现的字段
[mdat] — 被 trun/tfhd 引用的媒体 sample 字节

```

这是一般性结构示意，不是所有应用格式的唯一顶层顺序。例如 W3C MSE 严格把 media segment 定义为可选 `styp` 后接一个 `moof` 和一个或多个 `mdat`；`prft` 等其他顶层 Box 可以被接受和忽略，但不计入该 MSE media segment。具体打包器应按目标 profile 放置可选 Box。

**H.264 视频 sample 在 mdat 中的一种典型排列（长度前缀格式）：**

```text
[4B length][NALU₁] [4B length][NALU₂] [4B length][NALU₃] ...

```

一个 sample 可含多个 NALU（如 AUD + SEI + IDR）。

`mdat` 自身没有 sample 边界和 Track ID；这些信息来自 `moof` 中的偏移、大小与 Track 关联。音频 sample、字幕或其他 Track 的字节也可位于同一个 `mdat`，不能把整个 `mdat` 都按 H.264 NALU 扫描。

`tfdt` 给出的是**轨道媒体时间轴**上的解码时间，不是 UTC 或系统墙钟时间。若需要估算直播端到端延迟，可用可选的 `prft`（Producer Reference Time）把媒体时间映射到 NTP 时间，或使用清单、协议层提供的节目日期时间。

`trun` 也不一定逐 sample 显式携带所有字段。duration、size、flags 可以来自 `trun`，也可以继承 `tfhd`，再不然使用初始化段 `trex` 的默认值；`sample_description_index` 也按 `tfhd → trex` 解析。`first_sample_flags` 只覆盖该 run 的第一个 sample，并且不能与逐 sample 的 `sample_flags` 同时出现。解析器必须按照 flags 和继承优先级取值，不能把字段缺省解释为数值 0。

#### sample 如何定位到 mdat

`mdat` 只是媒体字节仓库；真正的寻址链是 `traf → tfhd → trun → sample`：

1. `tfhd.track_ID` 确定这些 sample 属于哪条 Track，并关联相应 `trex` 和 Sample Entry。
2. 先按 `tfhd` 确定 base data offset：
   - `base-data-offset-present=1`：使用显式 64 位 `base_data_offset`；
   - 否则 `default-base-is-moof=1`：基址是当前 `moof` 的起始字节，这两个 flag 不能同时设置；
   - 两者都没有时，当前 `moof` 的首个 `traf` 以 `moof` 起始字节为隐式基址；后续 `traf` 则以前一个 Track Fragment 所引用数据的结束位置为基址。这种跨 `traf` 隐式状态更难流式解析，不应被误写成“永远相对当前 moof”。
3. 若 `trun` 设置 `data-offset-present`，其 `data_offset` 是相对上述基址的**有符号 32 位偏移**。常见的第一条 run 起点为 `moof_start + trun.data_offset`，但只有在基址确实为 `moof_start` 时这个简式才成立。
4. 若 `trun` 没有 `data_offset`，第一条 run 从 `tfhd` 基址开始；后续 run 则紧接前一条 run 所引用媒体数据的末尾。这里的“紧接”是媒体数据寻址状态，不是“紧跟前一个 `trun` Box 的字节”。
5. run 内第 $k$ 个 sample 的起始位置由此前 sample size 累加得到：

$$
\operatorname{offset}(k)
= \operatorname{run\_start}
+ \sum_{i=0}^{k-1}\operatorname{sample\_size}(i)
$$

6. sample size、duration 和 flags 优先取 `trun` 中的逐 sample 字段；第一帧还可能使用 `first_sample_flags`；字段缺省时再取 `tfhd` 默认值，最后取 `trex` 默认值。

解析器必须按 Box flags 建立每个 Fragment 的寻址状态，不能硬编码“媒体永远紧跟在 `moof` 后”，也不能在多个 `traf` 复用一个未经更新的游标。面向 MSE 的内容还需满足 movie-fragment relative addressing：每个 `traf` 的第一条 `trun` 带 `data_offset`，并且要么所有 `traf` 都设置 `default-base-is-moof`，要么当前 `moof` 仅有一个 `traf` 且未显式设置 `base_data_offset`。HLS fMP4 同样要求 movie-fragment-relative addressing。

### 2.5 时间戳机制

#### DTS 计算

`tfdt.base_media_decode_time` 是该 `traf` 第一条 run 中第一个 sample 的 DTS。后续 sample 的 DTS 由 duration 累加：

$$
DTS_n = B + \sum_{i=0}^{n-1}\Delta_i
$$

其中 $B$ 是 `tfdt.base_media_decode_time`，$\Delta_i$ 是第 $i$ 个 sample 的 duration；duration 按 `trun → tfhd → trex` 的优先级取得。

#### PTS 计算

$$
PTS_n = DTS_n + CTO_n
$$

其中 $CTO_n$ 是 `sample_composition_time_offset`。无重排序时通常为 0；存在 B 帧时通常不为 0。`trun` version 0 的 CTO 是无符号数，version 1 的 CTO 是有符号数；出现负 CTO 时必须使用 version 1，或通过合规的时间轴平移避免负值。

sample duration 决定相邻 DTS 的距离，CTO 决定同一个 sample 从解码时间移动到呈现时间的位置。把 PTS 当成下一个 sample 的 DTS，或者只保存 PTS 而丢掉 DTS，都会破坏含 B 帧视频的重排序。

#### 数值示例

以下是一个只为说明“解码顺序与显示顺序不同”的简化例子。`timescale=90000`，30 fps，因此每帧 `duration=3000`；存储/解码顺序为 I、P、B、B，显示顺序为 I、B、B、P。例子使用 `trun` version 1 表达负 CTO：

|存储/解码顺序|帧|Duration|CTO|DTS|PTS|显示顺序|
|---|---|---|---|---|---|---|
|1|I|3000|0|0|0|1|
|2|P|3000|6000|3000|9000|4|
|3|B|3000|-3000|6000|3000|2|
|4|B|3000|-3000|9000|6000|3|

`sample_flags` 还描述 sample 是否依赖其他 sample、是否为 non-sync sample 等。容器标记的 sync sample 应对应编码层可随机访问的起点，但二者不是凭名称自动等价：开放 GOP、恢复点、CRA/IDR 差异或错误标记都会影响真实可解码性。

解析时不要只看 `sample_is_non_sync_sample` 一个 bit。完整 flags 还含 `sample_depends_on`、`sample_is_depended_on`、`sample_has_redundancy` 和 degradation priority；其中值 0 往往表示“未知”，不是自动等于“无依赖”。打包器应依据编码访问单元真实依赖填写，播放器则要结合 SAP/协议层声明判断随机访问。

### 2.6 音视频排布

#### 分离式（自适应流媒体中常见）

- 视频和音频在不同 URL/文件

- 优势：ABR 切换互不影响、多语言独立切换

- DASH/HLS 的常见实现会将不同码率、音轨和字幕轨分开寻址，便于 ABR 与多语言选择；规范和实际系统也允许复用音视频的变体

#### 混合式

- 一个 moof 含两个 traf (Track ID 区分)

- 可用于复用音视频的文件或分段媒体，但 ABR 和多语言切换通常更偏好分离 Track 交付

#### 同步机制

- Track ID 关联：Init Segment 注册 → Media Segment 认领

- 时间对齐：各轨先按自身 timescale 换算为秒，再结合编辑列表、演示时间线和协议层时间映射对齐；不同轨的数值不应直接相减

### 2.7 Seek 机制

- `sidx` 可以记录子段的时间范围与字节偏移，但它是可选索引，不是 fMP4 Seek 的唯一入口。

- `sidx.earliest_presentation_time` 和 `subsegment_duration` 使用 `sidx.timescale`，不一定等于 Track timescale；`first_offset` 是从 **`sidx` Box 末尾**到第一项引用对象首字节的偏移，不是文件绝对偏移，也不是天然相对 `moof`。每个 reference 的 `referenced_size` 再把游标推进到下一对象。

- `reference_type=0` 表示引用媒体子段，`reference_type=1` 可以引用另一层 `sidx`；`starts_with_SAP/SAP_type/SAP_delta_time` 描述随机访问属性。实现不能把所有 reference 都直接当 `moof`，也不能只凭 `starts_with_SAP=1` 忽略 codec 的真实依赖。

- DASH/HLS 清单、HTTP Byte Range、`sidx`、文件尾的 `mfra/tfra`、服务端数据库或播放器自身索引，都可以先定位目标片段。

- 到达候选片段后，再根据 sample flags、SAP/随机访问点信息和编码依赖选择可解码起点。没有 `sidx` 并不必然意味着从文件头扫描全部 `moof`。

### 2.8 CMAF 低延迟扩展

CMAF 是基于 ISO BMFF 的分段媒体应用格式与约束集合，定义 Track、Fragment、Chunk、品牌和可切换编码表示等规则，但不规定清单格式、播放器或网络传输协议。它可以被 HLS、DASH 等交付体系复用，从而减少重复封装。

低延迟的关键是让尚未完成的较大 Segment 以更小 Chunk 尽早可用，并让服务端、CDN 与播放器及时传输和消费。实际延迟由 Chunk/GOP 时长、编码器、清单更新、网络传输和播放器缓冲共同决定，不能仅由“使用 CMAF”推导出固定秒数。

**一种常见的低延迟字节序列：**

```text
[styp]
[moof₁][mdat₁] ← 子对象 1 立即发送
[moof₂][mdat₂] ← 子对象 2 立即发送
[moof₃][mdat₃] ← 子对象 3 立即发送

```

这些子对象在具体 CMAF 结构中可能组成一个或多个 Fragment，不能仅凭三个 `moof` 就断言“一个 Fragment 有三个 Chunk”。对象名称由 CMAF Header/Track、地址范围和清单关系共同确定。

在 Low-Latency HLS 中，短对象由 `EXT-X-PART` 标识为 HLS Partial Segment，完整 Parent Segment 覆盖与其 Parts 相同的媒体时间范围；`EXT-X-PRELOAD-HINT` 只预告可能即将出现的资源，不保证该资源最终一定按预告发布。Part 只有在 `INDEPENDENT=YES` 等相应信令与真实编码依赖都成立时才可独立起播，不能从 Parent Segment 独立或 Playlist 含 `EXT-X-INDEPENDENT-SEGMENTS` 直接推导“每一个 Part 都是随机访问点”。

RFC 8216 要求每个 HLS fMP4 Segment 都有 `EXT-X-MAP` **作用于它**。这不等于必须在每个 Segment URI 前重复写一行：该 Tag 的作用会持续到新的 `EXT-X-MAP` 出现。配置、Track 集合或加密初始化信息改变时，应在正确边界发布新的 Map，并按变化类型配合 `EXT-X-DISCONTINUITY`；旧 Map 不能继续解释新 sample。

---

## 三、FLV（Flash Video）格式详解

### 3.1 设计哲学

FLV 是顺序 Tag 流：文件头之后依次出现 Script、Audio、Video Tag。Tag 自带类型、负载长度和毫秒时间戳，因此接收端无需先得到全局样本表就能增量解复用。它不是严格意义上的“链表”——Tag 中没有 next 指针；可顺序前进依赖 `DataSize`，可辅助向后定位依赖 `PreviousTagSize`。

### 3.2 整体架构

```text
[FLV Header (9字节)]
[PreviousTagSize₀ = 0]
[Tag₁ Script] — onMetaData
[PreviousTagSize₁]
[Tag₂ Video] — AVC Sequence Header (SPS/PPS)
[PreviousTagSize₂]
[Tag₃ Audio] — AAC Sequence Header (ASC)
[PreviousTagSize₃]
[Tag₄ Video] — 第一个关键帧 (IDR)
[PreviousTagSize₄]
[Tag₅ Audio] — 第一个音频帧
...

```

#### FLV Header 二进制格式（9 字节）

|偏移|大小|字段|值|说明|
|---|---|---|---|---|
|0|3B|Signature|0x46 4C 56 ("FLV")|魔数|
|3|1B|Version|0x01|版本号|
|4|1B|TypeFlags|bit[2]=Audio, bit[0]=Video|流类型|
|5|4B|DataOffset|通常 0x00000009|Header 长度；FLV v1 常见为 9 字节，解析时应按该字段跳过 Header|

**TypeFlags 位定义：**

```text
bit:  7  6  5  4  3  2  1  0
      0  0  0  0  0  A  0  V
0x05 = 有音频+有视频
0x04 = 仅音频
0x01 = 仅视频

```

**Hex Dump 示例（完整 FLV 文件开头 13 字节）：**

```text
46 4C 56 01 05 00 00 00 09 00 00 00 00
"FLV" v1 A+V  DataOffset=9  PrevTagSize₀=0

```

### 3.3 Tag Header（11 字节通用头）

第一个字节不是纯 `TagType`，而是 `Reserved(2) + Filter(1) + TagType(5)`。普通未加密 FLV 的 Reserved 和 Filter 都为 0。

|字段|大小|说明|
|---|---|---|
|Reserved + Filter + TagType|1B|TagType：8=音频、9=视频、18=脚本；Filter=1 表示需要预处理（如规范定义的加密 Tag）|
|Data Size|3B|Tag Data 长度|
|Timestamp|3B|媒体时间低 24 位，单位毫秒|
|TimestampExtended|1B|媒体时间高 8 位|
|Stream ID|3B|总是 0|

组合后的 32 位位模式为：

$$
T = \text{TimestampExtended}\times 2^{24} + \text{Timestamp}
$$

FLV 10.1 规范把组合结果定义为 `SI32`，且文件第一个 Tag 的时间戳为 0、后续时间相对它计时；大量直播实现则把同一位模式按无符号 32 位毫秒计数并做回绕展开。实现方必须明确采用哪种约定，不能只看 4 字节后按本地有符号整数直接比较。

### 3.4 Video Tag Data

|字段|大小|说明|
|---|---|---|
|FrameType（4 bit）+ CodecID（4 bit）|1B|对 AVC：FrameType 1=可 Seek 帧，2=不可 Seek 帧；CodecID 7=AVC|
|AVCPacketType|1B|0=SeqHeader, 1=NALU, 2=EndOfSeq|
|CompositionTime|3B|`SI24` 毫秒偏移；仅 AVC NALU 包有效，$PTS = DTS + CompositionTime$|
|Data|变长|取决于 AVCPacketType|

#### AVCPacketType=0：AVC Sequence Header（SPS/PPS）

传统 AVC FLV 的 Sequence Header payload 是 `AVCDecoderConfigurationRecord`。它与 fMP4 的 `avcC` **Box 负载**使用同一记录语法，但不包含 `avcC` Box 自身的 size/type 头。

```text
┌── Video Tag Data ──────────────────────────────────┐
│ 0x17                 ← FrameType=1,CodecID=7       │
│ 0x00                 ← AVCPacketType=0(SeqHeader)  │
│ 0x00 0x00 0x00       ← CompositionTime=0           │
│ [AVCDecoderConfigurationRecord]                    │
│   01 64 00 1F FF E1 00 1A 67...   ← 与 avcC 负载同语法 │
└────────────────────────────────────────────────────┘

```

#### AVCPacketType=1：NALU 数据（AVCC 格式）

```text
┌── Video Tag Data ──────────────────────────────────┐
│ 0x17 或 0x27        ← 关键帧/普通帧, H.264         │
│ 0x01                ← AVCPacketType=1(NALU)        │
│ 0xXX 0xXX 0xXX     ← CompositionTime=CTS          │
│ [4B length][NALU₁] [4B length][NALU₂] ...         │
│            ← AVCC 格式, 与 fMP4 mdat 内部相同!     │
└────────────────────────────────────────────────────┘

```

Adobe 的传统 FLV 语法允许一个 AVC Video Tag 承载一个或多个 NALU，甚至允许按独立 Slice 分包，并不强制每个 Tag 都是完整 Access Unit。很多直播系统约定“一条视频消息就是一帧”，但重封装器不能只依赖这个工程惯例：若输入按 Slice 分包，转为一个 sample 对应一个 Access Unit 的 fMP4 前，必须结合时间戳、NALU 语法和 Access Unit 边界先完成组帧。

### 3.5 Audio Tag Data

|字段|大小|说明|
|---|---|---|
|SoundFormat(4b)+Rate(2b)+Size(1b)+Type(1b)|1B|10=AAC；传统 FLV 中 AAC 常将后三个字段写为 3/1/1，真实采样配置以 ASC 为准|
|AACPacketType|1B|0=SeqHeader, 1=Raw|
|Data|变长|—|

#### AACPacketType=0：AAC Sequence Header（AudioSpecificConfig）

- 通常仅 2 字节：如 `12 10` = AAC-LC, 44100Hz, Stereo

- 其内容是 `AudioSpecificConfig`，可与 MP4 `esds` 内 `DecoderSpecificInfo` 所含 ASC 对应；它**不等于整个 `esds` Box**

- 没有这个 → 无声

#### 音频帧大小

传统 FLV 的一个 `AACPacketType=1` Tag 通常承载一个 Raw AAC access unit。AAC-LC 最常见的帧长是 1024 个采样点，因此在 44.1 kHz 下常见时长为：

$$
\frac{1024}{44100}\times 1000 \approx 23.22\ \text{ms}
$$

但 GA 配置可使用 960 samples，其他 AOT、SBR/PS 和多 raw data block 还会改变 core/output 采样关系，不能靠固定 23.22 ms 反推时间戳。以 128 kbit/s、1024 samples、44.1 kHz 粗略估算，编码负载平均约 372 字节/帧；瞬时帧大小仍会随码率模式和编码内容变化。

### 3.6 时间戳机制

- 对 AVC NALU Tag，Tag Header Timestamp 用作 DTS（毫秒）；CompositionTime 再把它映射到 PTS

- $PTS = DTS + CompositionTime$；该字段是有符号 24 位整数，因此 B 帧可能出现负偏移

- 对通常不做显示重排序的 AAC，容器层可视为 $PTS = DTS$

- 精度：毫秒级

- 低 24 位约每 4.66 小时进位到 `TimestampExtended`。若按直播系统常见的无符号 32 位计数解释，一个完整周期约 49.71 天；若严格按 FLV 规范的 `SI32` 正值范围解释，约 24.86 天就到达最大正值。长直播应在输入边界做时间戳展开，并在重连、回退或配置切换处显式处理 discontinuity

### 3.7 音视频排布

- 文件或流通常按时间大致交织音视频 Tag，以便增量播放；FLV 结构本身不要求所有 Tag 全局严格按 DTS 排序

- 音频密度更高：AAC 44.1kHz 每 23ms 一个 Tag vs 视频 30fps 每 33ms

- H.264 视频流应在媒体帧前发送 AVC Sequence Header；直播服务端通常会缓存并补发

- AAC 音频流应在 Raw AAC 帧前发送 AAC Sequence Header；否则中途加入可能无法立即解码

### 3.8 Seek 机制

- `PreviousTagSizeN`：位于第 N 个 Tag 之后，值为该 Tag 的 `11 + DataSize`，不包含 `PreviousTagSize` 字段自身；`PreviousTagSize0` 紧跟 FLV Header，值为 0

- 可选的 `onMetaData.keyframes`：常用 `times` 与 `filepositions` 数组记录关键帧时间和文件字节位置，但直播流或普通 FLV 文件未必提供

- 文件 Seek 的典型流程：查关键帧索引 → 跳转到可 Seek Video Tag → 补齐/恢复 Sequence Header → 从该点继续解码。没有索引时只能扫描或由服务端维护外部索引

### 3.9 FLV 的局限性

- H.265/AV1 等新编码不在 Adobe 传统 4-bit `CodecID` 定义中。Enhanced RTMP/FLV 使用扩展视频头和 FourCC 信令；厂商私有 CodecID 与该开放扩展不能混为一谈

- 传统 FLV 缺少像 ISO BMFF 那样的原生标准多轨与字幕 Track 模型；多语言、多字幕通常要依赖额外协议或扩展

- 时间戳精度低（毫秒 vs fMP4 可配置 timescale）

- Flash 退场后地位下降，但 HTTP-FLV 仍广泛使用

### 3.10 Enhanced RTMP/FLV：不是私有 CodecID 的别名

Enhanced RTMP v2 在保留传统消息类型的基础上增加扩展音视频头。以视频为例，传统首字节是 `FrameType(4) + CodecID(4)`；扩展模式把最高位作为 `isExVideoHeader`，随后 3 bit 表示 `VideoFrameType`，低 4 bit 改解释为 `VideoPacketType`，codec 由后续 FourCC 标识。因此旧解析器若仍把低 4 bit 当 `CodecID`，会从第一个字节起就错位。

常用视频 Packet Type 的含义如下：

|VideoPacketType|作用|关键边界|
|---|---|---|
|`SequenceStart=0`|携带相应 codec configuration record|FourCC 可标识 `avc1`、`hvc1`、`av01`、`vp09`、`vvc1` 等|
|`CodedFrames=1`|携带完整编码帧/时间单元|对 AVC/HEVC/VVC 等仍携带 `SI24` 毫秒 CompositionTimeOffset|
|`SequenceEnd=2`|结束当前编码序列|不是普通空媒体帧|
|`CodedFramesX=3`|携带编码帧但省略 CTS|只在 CompositionTimeOffset 隐含为 0 时使用|
|`Metadata=4`|AMF 编码的视频元数据|payload 不是视频码流|
|`Multitrack=6`|启用多轨结构|每条音/视频消息都要显式携带 track ID，track ID 不跨消息继承|
|`ModEx=7`|修饰/扩展当前 packet|可表达亚毫秒 timestamp offset 等扩展，需继续解析出实际 Packet Type|

扩展音频头采用相同思路，以 FourCC 信令 `mp4a`、`Opus`、`fLaC`、`ac-3`、`ec-3` 等 codec，并定义 SequenceStart/CodedFrames 等状态。发送端还要在 `connect` 命令中声明 FourCC 与扩展能力；接收端不能看到 FourCC 后就假定双方已经协商支持。

Enhanced RTMP/FLV 还定义标准多轨、HDR/颜色元数据和高精度时间戳扩展。这解决了传统 FLV 的部分局限，但**不会让传统 FLV v1 解析器自动获得这些能力**。转封装器应先区分 legacy header、Enhanced header 和厂商私有 CodecID，再选择对应配置记录、帧边界和时间戳解析路径。

---

## 四、RTMP（Real-Time Messaging Protocol）格式详解

### 4.1 先澄清：RTMP 不是媒体封装容器

RTMP 更准确地说是**实时传输协议**，不是像 MP4、FLV、TS 那样的文件封装格式。它常与 FLV Tag 数据模型配合使用：推流端把音视频按 RTMP Message 发送，消息 payload 通常承载与 FLV Audio/Video/Script Tag Data 高度一致的数据。

可以这样理解：

- **FLV**：定义文件/流里的 Tag 结构，常用于 HTTP-FLV 播放链路。

- **RTMP**：定义客户端与服务器之间如何建连、握手、分块、复用和传输消息。

- **RTMP + FLV 数据模型**：直播推流最常见组合之一，H.264/AAC 的 Sequence Header、NALU、AAC Raw 等与 FLV 章节中的描述可以对应起来。

### 4.2 RTMP 整体链路

```text
TCP 连接
  └── RTMP Handshake: C0/C1/C2 <-> S0/S1/S2
        └── RTMP Chunk Stream
              ├── Control Message：窗口确认、带宽、Set Chunk Size、Ping 等
              ├── Command Message：connect、createStream、publish、play 等
              ├── Data Message：onMetaData 等
              └── Audio/Video Message：承载音视频负载

```

RTMP 工作在 TCP 之上，默认端口通常是 `1935`。它的核心不是“文件结构”，而是**把不同类型的消息切成 Chunk，在一条 TCP 连接中复用传输**。

### 4.3 握手格式

RTMP 建连后先进行握手：

```text
Client -> Server: C0 + C1
Server -> Client: S0 + S1 + S2
Client -> Server: C2

```

|阶段|大小|说明|
|---|---|---|
|C0/S0|1B|RTMP 版本号，常见为 0x03|
|C1/S1|1536B|基础规范为 4B time + 4B zero + 1528B random|
|C2/S2|1536B|对端 time、接收时刻 time2 与 1528B random echo|

公开的 Adobe RTMP 1.0 规范定义的是固定长度的基础握手：C1/S1 为 `time + zero + random`，其中 4 字节 `zero` 必须为 0；C2/S2 回显对端的时间与随机区。工程中所谓带 HMAC 摘要的 “complex handshake” 来自历史实现扩展，并未写入该公开 RTMP 1.0 语法，不能在没有互操作约定时把 C1/S1 的第二个 4 字节字段擅自解释为版本号或摘要索引。普通直播链路通常由 SDK 或服务器框架封装这两条路径。

### 4.4 Chunk：RTMP 的传输基本单元

RTMP Message 可能很大，协议会把 Message 切成多个 Chunk 发送。每个 Chunk 由 Basic Header、Message Header、可选 Extended Timestamp 和 Chunk Data 组成。

```text
┌──────────────────────────────────────────────────────────┐
│ Basic Header │ Message Header │ Extended Timestamp? │ Data │
└──────────────────────────────────────────────────────────┘

```

#### Basic Header

Basic Header 主要包含：

- **fmt**：2 bit，表示 Message Header 的压缩格式。

- **csid**：Chunk Stream ID，用于区分不同 Chunk Stream。

|fmt|Header 类型|含义|
|---|---|---|
|0|Type 0，11B|完整消息头；Chunk Stream 起始处或时间戳回退时必须使用|
|1|Type 1，7B|省略 Message Stream ID，携带 timestamp delta、消息长度和类型|
|2|Type 2，3B|只携带 timestamp delta，复用长度、类型和 Message Stream ID|
|3|Type 3，0B|复用同一 Chunk Stream 上一 Header；常用于同一 Message 的后续 Chunk，也可用于字段与时间间隔均可复用的新 Message|

#### Message Header 关键字段

|字段|大小|说明|
|---|---|---|
|Timestamp / Delta|3B|时间戳或时间戳增量；当值为 0xFFFFFF 时使用 Extended Timestamp|
|Message Length|3B|完整 RTMP Message payload 长度，不是当前 Chunk 长度|
|Message Type ID|1B|消息类型，例如音频、视频、命令、控制消息|
|Message Stream ID|4B|消息流 ID，采用小端序；只在 Type 0 Header 中出现|

RTMP Header 有一个容易踩坑的点：多数多字节字段按网络序/大端处理，但 **Message Stream ID 是小端序**。

当 24 位 timestamp 或 timestamp delta 字段为 `0xFFFFFF` 时，后面必须出现 4 字节 Extended Timestamp。若上一条 Type 0/1/2 Header 使用了 Extended Timestamp，同一 Chunk Stream 上继承它的 Type 3 Chunk 也必须携带该字段。接收端应先按 Chunk Stream ID 重组完整 Message，再把 Message payload 交给 FLV 音视频负载解析器。

#### 每个 CSID 都有独立的 Header 与重组状态

`Chunk Stream ID`（CSID）与 `Message Stream ID` 不是同一个字段。接收端至少要为每个 CSID 保存：

- 上一次完整 Header 的 Message Stream ID、Message Length 和 Message Type；
- 上一次绝对 timestamp、最近的 timestamp delta，以及是否启用了 Extended Timestamp；
- 当前正在重组的 Message 已接收多少字节。

收到新 Chunk 时，再结合该 CSID 的状态解释 `fmt`：

1. Type 0 建立完整新状态，携带**绝对** timestamp；切换 Message Stream ID 或 timestamp 回退时应重新使用它。
2. Type 1 复用 Message Stream ID，但更新长度、类型和 timestamp delta。
3. Type 2 继续复用长度、类型和 Message Stream ID，只更新 timestamp delta。
4. Type 3 有两种完全不同的用途：
   - 当前 Message 尚未收满：它是同一 Message 的后续 Chunk，timestamp 不再推进；
   - 当前 Message 已收满：它可以开始一条字段和时间间隔都可继承的新 Message，此时用已保存的 delta 推进 timestamp。若它直接继承 Type 0，规范把该 Type 0 的 timestamp 值作为可复用的 delta。

因此，不能把“看到 Type 3 就沿用上一条绝对 timestamp”写成统一规则，也不能跨 CSID 继承 Header。不同 CSID 的 Chunk 可以交织，但同一 CSID 上一条 Message 未结束时，后来的字节必须继续完成该 Message，除非收到针对该 CSID 的 Abort Message。

Chunk Size 也按**发送方向分别维护**：默认最大 Chunk Data 为 128 字节；某端发送 `Set Chunk Size` 后，新值约束的是该端随后发出的 Chunk。它只改变分块边界，不改变 Message Length、媒体帧边界或时间戳。

### 4.5 Message Type：RTMP 承载什么

|Type ID|名称|典型内容|
|---|---|---|
|0x01|Set Chunk Size|设置后续 Chunk Data 的最大大小，默认通常为 128 字节|
|0x04|User Control Message|Stream Begin、Ping Request、Ping Response 等|
|0x08|Audio Message|Legacy AAC 或 Enhanced FourCC 音频负载，与相应 FLV Audio Tag Data 对应|
|0x09|Video Message|Legacy AVC 或 Enhanced FourCC 视频负载，与相应 FLV Video Tag Data 对应|
|0x0F|Data Message AMF3|AMF3 数据消息|
|0x11|Command Message AMF3|AMF3 命令消息|
|0x12|Data Message AMF0|onMetaData 等 AMF0 数据消息|
|0x14|Command Message AMF0|connect、createStream、publish、play 等 AMF0 命令|

### 4.6 RTMP 与 FLV 的对应关系

RTMP 的音视频 Message payload 与 FLV Tag Data 高度对应，但二者边界和时间戳位置不同：

|维度|FLV|RTMP|
|---|---|---|
|定位|封装格式/流格式|实时传输协议|
|基本单元|Tag|Message，被切分成 Chunk 传输|
|视频负载|Video Tag Data|Video Message payload，通常复用 Video Tag Data 结构|
|音频负载|Audio Tag Data|Audio Message payload，通常复用 Audio Tag Data 结构|
|元数据|Script Tag|Data Message，如 onMetaData|
|时间戳|位于 FLV Tag Header|位于 RTMP Chunk Message Header；重组后属于 Message 元数据，不在音视频 payload 内|
|传输边界|Tag Header + Tag Data + PreviousTagSize|Chunk Header + Message payload；没有 FLV Tag Header 和 PreviousTagSize|

因此，RTMP 转 HTTP-FLV 通常不需要重新编码，只需要做**协议解包 + 封装重组**：重组 RTMP Message，把其音视频 payload 写入 FLV Tag Data，把 RTMP Message timestamp 映射到 FLV Tag timestamp，并补齐 FLV Header、Tag Header、PreviousTagSize 等结构。

### 4.7 工程特点与局限

- **低延迟**：长连接持续推送，通常比传统 HLS 更低延迟。

- **推流生态成熟**：OBS、FFmpeg、SRS、Nginx-RTMP 等工具链长期支持。

- **浏览器播放不友好**：Flash 退场后，浏览器端通常不直接播放 RTMP，需要转成 HTTP-FLV、HLS、DASH、WebRTC 等。

- **基于 TCP**：应用看到的是有序、可靠字节流；网络丢包会触发重传，而前面的缺口会阻塞后续字节交付，因此弱网下可能放大延迟和抖动。

- **编解码扩展依赖约定**：H.264/AAC 最常见；H.265 等通常依赖 Enhanced RTMP/Enhanced FLV 或厂商约定。

## 五、TS（MPEG-2 Transport Stream）格式详解

### 5.1 设计目标

MPEG-2 TS 面向广播、复用和连续传输场景。固定 188 字节包长与同步字节便于硬件处理、在字节流中重新找到包边界；PID、连续计数器和周期性 PSI 则帮助接收端识别流、检测丢包并在后续数据处恢复。基础 TS 本身不提供前向纠错或重传，“容易检测并重新同步”不等于“不会丢包”。

ITU-T H.222.0 / ISO/IEC 13818-1 定义的核心 TS packet 是 188 字节。工程上看到的 192 字节 M2TS 常在前面增加 4 字节到达时间戳，204 字节广播包常在后面增加 16 字节纠错数据；解析前要先识别外层变体，不能直接把它们当成 188 字节 TS 头。

### 5.2 核心结构

```text
[TS Packet 1 (188B)] [TS Packet 2 (188B)] [TS Packet 3 (188B)] ...
 └── 4B Header + [Adaptation Field] + [Payload]
     sync_byte=0x47 + PID + continuity_counter + ...

```

4 字节 TS Header 的位布局如下：

|字段|位数|作用|
|---|---:|---|
|sync_byte|8|固定 `0x47`|
|transport_error_indicator（TEI）|1|上游检测到不可纠正错误|
|payload_unit_start_indicator（PUSI）|1|本包 payload 中存在 PSI section 或 PES packet 的起点；具体解释随 PID 类型变化|
|transport_priority|1|同 PID 包的传输优先级提示|
|PID|13|标识 PSI、PES 或其他数据流|
|transport_scrambling_control|2|传输层加扰状态|
|adaptation_field_control|2|`01`=仅 payload，`10`=仅 adaptation field，`11`=两者都有，`00` 保留|
|continuity_counter|4|每个 PID 独立的模 16 连续计数|

- PID 区分不同流。`0x0000` 固定用于 PAT，PMT PID 由 PAT 给出，音视频 PID 由 PMT 给出，`0x1FFF` 是 Null Packet

- 一个视频帧可能被拆到多个 TS 包

- 每个 PID 各自维护 4 bit `continuity_counter`。`adaptation_field_control=01/11`、即包内含 payload 时，对同 PID 的新包递增 1（模 16）；`10`、即仅有 adaptation field 时不递增；`00` 为保留值，接收端应丢弃。Null PID `0x1FFF` 的连续计数值未定义，不能据此报丢包

- 规范允许同 PID 的原包与**一份**重复包连续出现：两包使用相同 CC，除 PCR 可更新为有效值外，其余字节相同，且二者的 adaptation control 为 `01` 或 `11`。因此 CC 重复只有在满足重复包条件时才不是错误；payload 不同却沿用同一 CC，不能被静默当成合法重传

- 同 PID 的 CC 无故跳变通常表示丢包、乱序或拼接错误；`discontinuity_indicator=1` 可以显式宣布连续性/系统时基中断，但接收端仍应丢弃已残缺的 PES 或 section，并从下一个可验证边界重组。`TEI=1` 的包也应作为损坏数据丢弃或明确传播错误状态，不能继续喂给解码器

- Adaptation Field 可携带 PCR、`random_access_indicator`、`discontinuity_indicator`、OPCR 和填充等信息，因此有效 payload 不一定是 184 字节。随机访问指示只是信令，接收端仍需核对编码访问单元和参数集

### 5.3 PES（Packetized Elementary Stream）

编码 Elementary Stream 先按 PES 组织，再切到同一 PID 的多个 TS packet 中：

```text
Elementary Stream Access Units
  → PES Header + PES Payload
  → TS Packet payload + TS Packet payload + ...
```

- 对承载 PES 的 PID，`PUSI=1` 表示当前 TS payload 的第一个字节就是新 PES packet 的起点；PES 路径没有 PSI 的 `pointer_field`。它不表示“这个 TS 包等于一帧”

- PES Header 可以携带 PTS/DTS；并非每个 TS 包甚至并非每个 PES Header 都同时包含二者

- `PTS_DTS_flags=10` 表示只带 PTS，`11` 表示同时带 PTS 和 DTS，`00` 表示两者都没有，`01` 是禁止值。DTS 不能脱离 PTS 单独出现；没有 B 帧重排时通常只需 PTS，有重排且两者不同时才同时写入

- 一个 PES packet 可以包含一个或多个 Access Unit，也可能只包含某个 Access Unit 的一部分；Access Unit 边界要结合编码语法和 PES/时间戳信息判断

- `PES_packet_length` 是 16 位；某些视频 PES 允许写 0 表示长度未在该字段中界定，因此不能只靠它切出所有视频 PES

### 5.4 PSI 表

- **PAT（PID 0）**：把 `program_number` 映射到对应 PMT PID；`program_number=0` 的条目用于 Network PID，不是普通节目

- **PMT**：列出节目中的基本流 PID、`stream_type`、描述符以及 `PCR_PID`。一些私有或扩展 `stream_type` 必须结合 registration/codec descriptor 才能确定真实 codec，不能只按单个类型字节猜流内容

常见 PMT `stream_type` 示例：

|stream_type|典型映射|
|---|---|
|`0x1B`|AVC/H.264 video|
|`0x24`|HEVC/H.265 video|
|`0x0F`|ISO/IEC 13818-7 AAC with ADTS transport syntax|
|`0x11`|ISO/IEC 14496-3 AAC with LATM transport syntax|
|`0x06`|PES private data；必须结合 descriptor/注册标识继续判断|

- PAT/PMT 以 section 形式承载。PUSI 为 1 时，payload 开头的 `pointer_field` 给出从它之后到第一个新 section 起点的字节数；这些字节可以是前一个跨包 section 的结尾或 stuffing，不能一律丢弃。section 可以跨多个 TS packet，单个 TS packet 也可能容纳多个 section

- `version_number`、`current_next_indicator`、`section_number/last_section_number` 共同管理表版本、当前/下一版本与分节。只有收齐同一版本所需 section 并通过校验后，才能原子替换当前 Program 映射，不能收到一个新版 PMT section 就混用新旧 PID

- section 末尾使用 H.222.0 Annex A 定义的 MPEG-2 CRC-32，使整个 section 通过同一 CRC 模型后的余数为 0。它的位序/初值处理不能直接套用常见“反射式 Ethernet/ZIP CRC-32”API 默认参数

广播接收端可在任意时刻加入，因此 PAT/PMT 通常周期性重发。RFC 8216 对 HLS 的要求更具体：一个 TS Segment 只能包含单个 Program；每个 Segment 必须含 PAT 与 PMT，或由适用的 `EXT-X-MAP` 提供媒体初始化段。没有 `EXT-X-MAP` 时，前两个 TS packet 应当是 PAT 和 PMT。

### 5.5 PCR、PTS、DTS：三种时间各管什么

TS 的同步不能只看 PTS/DTS，还必须理解 PCR：

|字段|典型位置|作用|时基|
|---|---|---|---|
|PCR（Program Clock Reference）|`PCR_PID` 的 TS Adaptation Field|让接收端恢复编码器的系统时钟，驱动去抖动与解码缓冲|33 bit base（90kHz）+ 9 bit extension（27MHz）|
|DTS|PES Header|指定 Access Unit 进入解码器的时刻；有重排序且 PTS 与 DTS 不同时需要|90kHz|
|PTS|PES Header|指定 Access Unit 的呈现时刻，用于音视频同步|90kHz|

PCR 的 42 位值由两部分组成：

$$
PCR_{27M} = 300 \times PCR_{base} + PCR_{ext}
$$

$$
t_{\text{seconds}} = \frac{PCR_{27M}}{27\,000\,000}
$$

`PCR_ext` 虽占 9 bit，但合规取值范围是 0～299；与 `PCR_base` 合并后形成 27 MHz 计数。

PCR 是系统时钟恢复基准，PTS/DTS 是相对于节目时间线的解码与呈现调度值。广播接收机可用 PCR 驱动 T-STD/去抖动时钟，再按 DTS 解码、按 PTS 呈现；某些文件或 HTTP 播放器会采用自己的时钟策略，但仍必须正确处理这些时间值。

不要用“最近一个 PCR 就是当前视频帧 PTS”的方式配对：PCR 可以位于单独 PID，也可以与某个 PES 共用 PID，它采样的是系统时钟而非 Access Unit 呈现时间。`discontinuity_indicator` 出现在 `PCR_PID` 时还可能宣布系统时基切换；接收端必须让下一 PCR 建立新时钟段，并同步重置相关抖动/展开状态。

PTS/DTS 和 `PCR_base` 都是 33 bit 循环计数。90 kHz 下完整周期为：

$$
\frac{2^{33}}{90000} \approx 95\,443.72\ \text{s} \approx 26.51\ \text{h}
$$

跨回绕点比较时要先做模数展开；遇到 `discontinuity_indicator` 或 Playlist 的 `EXT-X-DISCONTINUITY` 时，还要重建时间线，不能把它误判成普通回绕。

### 5.6 AVC/HEVC 参数集在 TS 中的携带方式：Annex B 内联

TS 中 AVC 视频通常按 H.264 byte stream format（工程中常称 Annex B）承载，以 Start Code 分隔 NALU。PES 和 TS packet 边界都不保证与 NALU 或 Access Unit 边界重合。

**一种常见的随机访问 Access Unit 组织方式：**

```text
[00 00 00 01] [AUD]            ← 帧边界
[00 00 00 01] [SPS: 67 ...]   ← 在随机访问/分段边界重复
[00 00 00 01] [PPS: 68 ...]   ← 在随机访问/分段边界重复
[00 00 00 01] [IDR: 65 ...]   ← 关键帧

```

**非关键帧：**

```text
[00 00 00 01] [Non-IDR: 41/61 ...]

```

在每个 IDR 或 Segment 边界前重复 SPS/PPS 是常见且有利于随机加入的 muxing 策略，但不是“所有 TS 必须在每个 IDR 前重复”的语法定律。能否从某处独立开始播放，还取决于 PAT/PMT、正确的随机访问点、参数集及音频配置是否可获得。fMP4 的参数集通常来自初始化段，FLV 通常来自 Sequence Header；编码配置发生变化或新会话加入时，这些初始化信息也可能重发，而不是整个流永远只发一次。

HEVC 的 TS 字节流也使用 Start Code，但随机访问 Access Unit 前常见的是 VPS/SPS/PPS 加 IDR 或 CRA：

```text
[00 00 00 01] [VPS type 32]
[00 00 00 01] [SPS type 33]
[00 00 00 01] [PPS type 34]
[00 00 00 01] [IDR type 19/20 或 CRA type 21]
```

CRA 与 IDR 的 leading-picture 语义不同，不能看到 HEVC `nal_unit_type=21` 就照搬 AVC IDR 的独立性标记。TS `random_access_indicator=1` 也只是说明当前位置存在随机访问信息的传输层提示；真正声明 HLS 全部分段独立的是清单的 `EXT-X-INDEPENDENT-SEGMENTS`，它要求各 Media Segment 的 sample 不依赖其他 Segment。编码依赖、参数集、容器标志与清单声明必须一致。

### 5.7 音频在 TS 中的携带

- 传统 HLS/TS 中的 AAC 常使用 ADTS Header（`protection_absent=1`、无 CRC 时 7 字节）+ AAC 原始访问单元

- ADTS 每帧携带 profile、采样率索引、声道配置和帧长等信息，比单次 ASC 更冗余但便于重新加入

- MPEG-2 TS 也定义了 LATM/LOAS 等其他 AAC 映射，不能把“TS 中 AAC 必然是 ADTS”当成通用规则

### 5.8 特点

- **优势：**固定包长、容易重新同步、能检测连续性错误、广播硬件与标准生态成熟

- **边界：**基础 TS 不自带重传或 FEC；固定 188 字节包会带来额外开销（仅 4B TS Header 占 $4/188 \approx 2.13\%$，若包含 adaptation field、PAT/PMT 重复和 PES Header，实际开销更高）；随机访问通常依赖节目指南、HLS Playlist、外部索引或接收端扫描

- **适用性：**在广播、传统 HLS 和既有硬件链路中仍很重要；在许多 HTTP 自适应点播和低延迟流媒体系统中，则常选择 CMAF/fMP4

---

## 六、四类格式/协议对比总结

|维度|fMP4|FLV|RTMP|TS|
|---|---|---|---|---|
|定位|分片封装格式|封装/流格式|实时传输协议|传输流封装|
|基本结构|Box 嵌套 + Movie Fragment|顺序 Tag 流|Message 切为 Chunk 复用|固定 188B packet|
|音视频排布|可分轨，片段化组织|按 DTS 交织|多消息流复用|按 PID 复用交织|
|NALU 表示|AVC/HEVC 通常为长度前缀|传统 FLV 仅 AVC；Enhanced 可承载 HEVC 等，NALU codec 通常仍为长度前缀|通常复用 legacy/Enhanced FLV Video Message payload|AVC/HEVC 通常为 Annex B/byte stream Start Code|
|初始化信息|Init Segment 中的 Sample Entry/配置 Box|Legacy AVC/AAC 或 Enhanced FourCC Sequence Header|Video/Audio Message 中的 Sequence Header|PAT/PMT + in-band 参数集/音频头|
|时间戳机制|tfdt + sample duration + CTO；可选 prft 映射墙钟|Tag Header Timestamp + CompositionTime|RTMP Message Timestamp / Delta|PCR 恢复时钟，PES PTS/DTS 调度呈现与解码|
|媒体时间精度|Track timescale 可配置|毫秒|常见媒体消息为毫秒|PTS/DTS 为 90 kHz，PCR 为 27 MHz|
|Seek 支持|清单、sidx/mfra、Byte Range 或应用索引|keyframes 元数据 + fileposition|协议本身不负责文件 Seek，播放侧通过命令和服务端状态实现|通常依赖外部 Playlist/索引|
|典型场景|DASH/HLS/CMAF|HTTP-FLV|直播推流、传统直播拉流|广播/传统 HLS|
|延迟属性|由分片/Chunk、GOP、清单、传输和播放器缓冲决定|由 GOP、服务端与播放器缓冲、网络决定|由 GOP、Chunk、TCP 与收发缓冲决定|由分段、缓冲、传输系统与播放器策略决定|

---

## 七、直播传输链路与格式转换原理

### 7.1 从采集到播放：每一层做什么

```text
Camera / Microphone
  ↓ raw video / PCM
Encoder
  ↓ H.264/HEVC Access Units + AAC Access Units + DTS/PTS
Muxer / Packager
  ├─ FLV Tag / RTMP Message
  ├─ fMP4 Init + Fragments
  └─ PAT/PMT + PES + TS packets
Transport / Delivery
  ├─ RTMP over TCP
  ├─ HTTP-FLV
  ├─ HLS/DASH over HTTP
  └─ UDP/RTP/SRT 等承载 TS
Player
  ↓ demux → decode → synchronize → render
```

这里有三个容易混淆的动作：

- **Remux/Transmux（重封装）**：编码数据不变，只重建容器、索引和时间戳表示。例如 H.264/AAC 的 RTMP 输入转成 HLS fMP4。

- **Transcode（转码）**：先解码再编码，编码格式、分辨率、码率或帧率发生变化。它开销更大，也会引入新的 GOP 和编码延迟。

- **Protocol gateway（协议网关）**：只改变传输层，例如 RTMP Message 转 HTTP-FLV Tag；可能同时做极轻量的封装补头，但不必改媒体编码。

判断能否纯重封装的第一步是核对**目标容器和播放端是否支持原编码、Profile、Level、像素格式、音频对象类型及加密方式**。容器支持 H.264 并不等于终端一定支持源流的 High 4:4:4 Profile。

### 7.2 时间戳换算：先展开，再缩放，最后重建

四种常见时间基不同：

|来源|常见时间单位|
|---|---|
|FLV / RTMP|毫秒|
|MPEG-2 TS PTS/DTS|1/90000 秒|
|fMP4 视频 Track|由 `mdhd.timescale` 决定，常见 90000|
|fMP4 AAC Track|由 `mdhd.timescale` 决定，常见等于采样率|

从源时基 $S_{src}$ 转到目标时基 $S_{dst}$ 的基本公式是：

$$
T_{dst} = \operatorname{round}\left(
\frac{T_{src}\,S_{dst}}{S_{src}}
\right)
$$

工程实现还必须遵守以下顺序：

1. **重建并展开计数**：RTMP 先根据 fmt Header 的绝对 timestamp 或 delta 重建 Message 时间，再把 TS 的 33 bit、RTMP/FLV 的 32 bit 位模式恢复成连续的较宽整数。若先缩放再判断回绕，边界附近很容易被误判。
2. **识别 discontinuity**：时间戳回退可能是回绕、重连、Seek、广告拼接或编码器重启；只有回绕适合按模数连续展开，其余情况通常要开启新时间线。
3. **分别保存 DTS 与 PTS**：对视频计算 $CTO = PTS-DTS$；不能只保留一个“timestamp”后猜另一个。
4. **用整数有理数换算**：避免浮点累计误差；实现应先约分或使用足够宽的中间整数，防止乘法溢出。逐帧 duration 的余数应累计到后续帧，或者直接缩放累计时间边界，再由相邻边界相减。
5. **按 Track 独立量化**：视频和音频分别在自己的 timescale 上生成单调 DTS，再映射到共同呈现时间。音频 44.1 kHz 与视频 90 kHz 不能逐包硬凑成相同整数。

对 $n$ bit 循环计数器，若已经排除 discontinuity，且相邻真实间隔不可能达到半个周期，可用最短有符号差展开：

$$
d_k =
\left(
\left(x_k-x_{k-1}+2^{n-1}\right)\bmod 2^n
\right)-2^{n-1}
$$

$$
X_k = X_{k-1}+d_k
$$

其中 $x_k$ 是线上的循环值，$X_k$ 是展开后的宽整数。该算法对 TS 的 33 bit 或常见 RTMP/FLV 32 bit 计数都适用，但前提很重要：长时间断流超过半周期、Seek、编码器重启或广告拼接时，必须借助 discontinuity 和外部时间线重新定锚，不能盲目选择“最短距离”。

例如 FLV 中 `DTS=1000 ms`、`CompositionTime=67 ms` 的 AVC Tag，转到 90 kHz 视频 Track 后可得到近似：

$$
DTS=90000,\qquad CTO=6030,\qquad PTS=96030
$$

若源毫秒时间戳已经量化，转成 90 kHz 只会改变单位，不会恢复被毫秒精度丢失的亚毫秒信息。

#### 负 CTO 不能截断成 0

源流含 B 帧时，$CTO_i=PTS_i-DTS_i$ 可能为负。fMP4 `trun` version 1 使用有符号 32 bit，传统 AVC FLV 使用有符号 `SI24` 毫秒，都可以直接表达各自范围内的负 CTO。最安全的策略是保留它，并确保 Sample Entry、播放器和应用格式允许。

若某条目标路径只接受非负 CTO，不能逐帧执行 `max(CTO, 0)`，因为这会改变呈现顺序和音视频同步。可先选：

$$
K \ge -\min_i(CTO_i)
$$

再把视频解码时间轴整体提前：

$$
DTS'_i=DTS_i-K,\qquad PTS'_i=PTS_i
$$

于是：

$$
CTO'_i=PTS'_i-DTS'_i=CTO_i+K\ge 0
$$

若这样产生负 DTS，再给**所有相关轨道的共同时间线**增加足够大的全局偏移，或通过合规的 Edit List/清单时间映射补偿。核心不变量是：每轨 DTS 顺序、每帧 PTS、跨轨呈现对齐和随机访问边界必须同时保持；只修视频字段、不修共同时间原点同样会造成 A/V 偏移。

### 7.3 H.264/HEVC 转封装：边界转换不等于重新编码

#### Annex B → 长度前缀

TS 转 fMP4/FLV 时，通常需要：

1. 从 PES payload 中组装完整 Access Unit，不能把每个 TS packet 当成一帧。
2. 扫描 3 或 4 字节 Start Code，取出每个 NALU。
3. 只去掉 Start Code，按目标配置写入 1/2/4 字节大端长度，再写 NALU 本体。NALU 内防止伪 Start Code 的 `emulation_prevention_three_byte`（常见 `00 00 03` 中的 `03`）属于 EBSP，不应在单纯转封装时删除。
4. AVC 收集 SPS/PPS 并生成 `AVCDecoderConfigurationRecord`；HEVC 收集 VPS/SPS/PPS 并生成 `HEVCDecoderConfigurationRecord`。根据 `avc1/avc3`、`hvc1/hev1` 的 Sample Entry 规则决定参数集允许出现的位置，不能把 in-band 更新静默藏在“仅配置记录”的类型中。
5. 根据真实随机访问语义设置 FLV FrameType、ISO BMFF sample flags/SAP 或 HLS 独立性信令，而不是只凭“包含 I Slice”判断；HEVC 还要区分 IDR、CRA、BLA 和 leading picture。

```text
Annex B:
00 00 00 01 [SPS] 00 00 00 01 [PPS] 00 00 01 [IDR]

Length-prefixed:
[len(SPS)][SPS] [len(PPS)][PPS] [len(IDR)][IDR]
```

#### 长度前缀 → Annex B

fMP4/FLV 转 TS 时，AVC 按 `lengthSizeMinusOne + 1`、HEVC 按 `HEVCDecoderConfigurationRecord` 中相应 Length 字段读取每个 NALU，验证它没有越过 sample 边界，再把长度字段替换为 Start Code。不要硬编码 4 字节长度，也不要在不可信输入上按声明长度无限读取。

对于 `avc1`/`hvc1` 样本条目，解码参数来自初始化记录；为了让 TS 分段可独立加入，muxer 常在随机访问 Access Unit 前注入最新 AVC SPS/PPS 或 HEVC VPS/SPS/PPS。对于 `avc3`/`hev1` 等允许 in-band 参数集的样本条目，还要按 sample 顺序处理更新，避免继续使用过期配置。

### 7.4 AAC 转封装：ASC、Raw AAC 与 ADTS

- **FLV**：Sequence Header 承载 ASC，后续 AAC Tag 承载 Raw AAC access unit。

- **fMP4**：Sample Entry 的 `esds`/其他适用配置 Box 描述解码配置，sample 承载不含 ADTS 的编码访问单元。

- **传统 TS/HLS**：常见做法是每个 AAC access unit 前带 ADTS Header。

TS/ADTS 转 FLV 或 fMP4 时，应解析 ADTS 的 profile、sampling frequency index、channel configuration、13 bit `aac_frame_length`、CRC 标志和 `num_raw_data_blocks_in_frame`；`aac_frame_length` 包含 ADTS Header 与编码负载总长。完成边界校验后再剥掉 7/9 字节 Header。最常见的 `num_raw_data_blocks_in_frame=0` 表示该 ADTS frame 含一个 raw data block；其他值不能继续按“一个 ADTS frame 固定等于一个 1024-sample sample”处理。

ADTS Header 的 2 bit profile 只直接表达有限的基础 AOT，无法完整携带任意 ASC。HE-AAC 的 SBR/PS 可能通过编码负载中的隐式/同步扩展出现，`channel_configuration=0` 还需要 Program Config Element。因此，从 ADTS 生成 ASC 时必须结合码流或可靠的上游配置；只由头部三项拼一个 ASC，可能把 HE-AAC 错标成 AAC-LC。反向转换也要先确认目标 AOT、频率和声道布局能够用 ADTS 表达。

不能把 ADTS Header 当成 AAC 编码负载写入 MP4 sample，也不能只复制一个固定的 `12 10` 去处理所有音频。若目标格式无法无损表达源配置，应保留为适用的原格式、转码或显式拒绝，而不是静默降级配置。

在成功确认配置后，反向转换才可根据 ASC 为每个适用的 access unit 重建 ADTS Header，并以目标规范要求的 sample/帧边界输出。

### 7.5 常见转换路径的字段映射

|转换路径|可以直接复用的内容|必须重建的内容|
|---|---|---|
|RTMP → HTTP-FLV|Audio/Video Message payload|FLV Header、Tag Header、timestamp、PreviousTagSize|
|FLV/RTMP → fMP4|受双方支持的 AVC/HEVC NALU、Raw AAC 和 codec configuration record|区分 legacy/Enhanced 头，重建 `ftyp/moov`、Track/Sample Entry、`moof/trun/tfdt`、sample flags、mdat 偏移|
|TS → fMP4|解出的 AVC/HEVC NALU、可兼容的 AAC 编码负载|PAT/PMT/PES 解复用、时间戳展开、Annex B→长度前缀、ADTS→ASC、Fragment 索引|
|fMP4 → TS|sample 内编码数据与 DTS/PTS|长度前缀→Annex B、ASC→ADTS、PES、PAT/PMT、PCR、PID/continuity counter|
|RTMP → HTTP-FLV → MSE fMP4|首段可复用 AVC/AAC 编码负载|网关先生成 FLV，浏览器侧仍需再次 transmux 为 ISO BMFF 才能喂给多数 MSE SourceBuffer|

#### FLV/RTMP → fMP4 的最小状态机

```text
判断 legacy FLV/RTMP 或 Enhanced FourCC 模式
  ↓
等待并解析相应 SequenceStart/Sequence Header
  ↓
解析 AVC/HEVC Configuration Record / AudioSpecificConfig
  ↓
生成并发布 Init Segment
  ↓
媒体消息 → sample(size, duration, flags, DTS, CTO)
  ↓
在随机访问点规划 Fragment/Segment 边界
  ↓
生成 moof（先确定字段与大小）
  ↓
计算 trun.data_offset → 写 mdat
```

`trun.data_offset` 依赖最终 `moof` 大小，所以常见实现会先构建/测量 `moof`，再回填偏移。sample duration 则通常要等下一个 DTS 才能精确知道；实时打包器需要保留至少一个待定 sample，或采用已知恒定 duration，结束时再处理最后一个 sample。

#### TS → fMP4 的额外状态

TS 输入还要维护：

- PAT/PMT 版本、Program、stream PID 与 PCR PID
- 每个 PID 的 continuity counter 和 PES 重组缓冲
- 33 bit PTS/DTS/PCR 的展开状态
- AVC/HEVC Access Unit 组装器、随机访问状态与最新参数集
- 音视频各自的 Track 时间线和分片边界

丢失带 `PUSI=1` 的 PES 起始包后，不应把后续中间 payload 冒充完整 sample；应丢弃到可恢复边界，并把缺口传播成 discontinuity 或损坏状态。

### 7.6 初始化信息、关键帧与配置变化

“先发初始化信息、再发可随机访问媒体”是三类封装共同的首屏原则，但具体载体不同：

|格式|初始化/信令|随机访问起点|配置变化时|
|---|---|---|---|
|fMP4|Init Segment：`ftyp + moov`|sample flags/SAP 与编码随机访问图像|发布匹配的新 Init Segment，清单切换 `EXT-X-MAP`/表示；必要时声明 discontinuity|
|FLV/RTMP|Legacy AVC/AAC Sequence Header，或 Enhanced FourCC SequenceStart，常配合 metadata|实际可随机访问的 Video Tag/Message|先发新的配置消息，再发使用新配置的媒体；服务端缓存也要原子更新|
|TS|PAT/PMT + in-band 参数集/音频头|随机访问 Access Unit，可辅以 `random_access_indicator`|若 PMT 内容变化则递增版本；发送新参数集，并按实际变化处理 PID、continuity 与时间基 discontinuity|

配置切换的危险窗口是“旧初始化信息配上新媒体”。如果分辨率或 SPS 已改变，却仍把新 sample 挂在旧 `avc1` Sample Entry 下，Box 结构即使完全合法，解码器仍可能失败。发布端应确保清单、初始化段和首个媒体分片的可见顺序一致，播放器则应按清单关系选择 Init Segment，而不是永远缓存第一个。

关键帧也要区分三层：

- 编码层：IDR、CRA、恢复点、开放/闭合 GOP 等具有不同依赖语义。
- 容器层：FLV FrameType、ISO BMFF sample flags/SAP、TS `random_access_indicator`。
- 协议层：HLS Segment/Partial Segment、DASH Segment/Representation 的独立性声明。

只有三层信令与实际依赖一致，播放器才能安全首播、Seek 和 ABR 切换。

### 7.7 延迟来自整条流水线

端到端直播延迟可以粗略拆成：

$$
L \approx L_{\text{capture}}
+ L_{\text{encode}}
+ L_{\text{GOP/packaging}}
+ L_{\text{network/CDN}}
+ L_{\text{player buffer}}
+ L_{\text{decode/render}}
$$

缩短 fMP4 Chunk、HLS Partial Segment 或 RTMP Chunk 只能影响其中一部分。编码器 B 帧重排、长 GOP、源站聚合、TCP 丢包重传、Playlist 刷新和播放器安全缓冲都可能成为主导项。低延迟优化必须同时观测采集时间、编码时间、媒体时间、服务端发布时间和客户端呈现时间，不能仅凭“格式名称”判断延迟。

---

## 八、直播场景常见工程问题

### 8.1 首屏时间（首帧出图）优化

核心瓶颈：拿到解码参数 + 等到关键帧

- **fMP4：**确保初始化段可快速获取，并从含随机访问点的目标 Segment/Chunk 开始；初始化段大小和首屏时间没有固定常数

- **FLV：**服务器缓存 Sequence Header + 最近 GOP → 新用户加入立即发送

- **TS：**获得 PAT + PMT、解码参数及可用的随机访问点；工程上常在分段边界重复这些初始化信息

- **关键：**若只能等待下一处 IDR，固定 2 秒 IDR 间隔带来的等待上界约为 2 秒；实际首屏还叠加建连、清单、网络、解复用、解码和播放器缓冲

### 8.2 中途加入问题

缺少解码参数 → 黑屏/无声

- **fMP4：**先获得与目标媒体段匹配的初始化段；源站、清单和播放器必须保证它可寻址且版本正确

- **FLV：**服务端必须缓存并补发 Sequence Header

- **TS：**等待可用随机访问点，并确保 PAT/PMT、参数集和音频配置都已获得；关键帧不保证天然携带 SPS/PPS

### 8.3 音视频不同步诊断

- 先把不同轨道的时间戳按各自 timescale 归一化到同一时间单位，再比较呈现时间。告警阈值应结合业务的可感知标准、设备链路和连续偏差窗口设置，不宜把 200ms 当作格式规范

- **fMP4：**检查 `tfdt`、sample duration、CTO、编辑列表和协议层时间线；不要直接比较不同 timescale 的整数值

- **FLV：**分别重建音频呈现时间和视频 $PTS=DTS+CompositionTime$，再放到同一时间轴比较；不能只取文件中恰好相邻的两个 Tag 做差

- **TS：**检查 PCR 连续性/抖动、PTS/DTS 展开、continuity counter 和 discontinuity 标志

- 常见原因：采集或编码时钟漂移、错误的 timescale 换算、丢包/缺片、时间戳回绕、重排序处理错误和 discontinuity 未重置

### 8.4 时间戳回绕/溢出

- **FLV：**低 24 位约 4.66 小时进位；完整位模式的解释需区分规范 `SI32` 与直播栈常见无符号计数，不能一概写成 49.7 天

- **TS：**33 bit PTS/DTS 在 90kHz 下约 26.5 小时回绕；PCR base 也需按循环计数处理

- **fMP4：**`tfdt` version 0 是 32 位，在 90kHz 下约 13.26 小时到达上限；version 1 是 64 位。解析器必须先读 version，再按对应位宽解析

### 8.5 直播 Seek（时移回看）

- **fMP4 + DASH/HLS：**通常由清单、Segment/Byte Range、随机访问点和服务端保留窗口共同实现，并非容器自动提供时移能力

- **FLV：**需要服务端维护关键帧索引

- **TS + HLS：**通过 Media Playlist 的 Segment 列表、时间线与服务端保留窗口实现

---

## 九、总结

不存在一种格式覆盖所有媒体系统。选择容器与协议时，应先看交付方式、随机访问、浏览器/硬件兼容、延迟预算、弱网策略和既有生态。

- **fMP4/CMAF** 是现代 HTTP 自适应流媒体的重要基础，可被 DASH 和 HLS 复用，但 CMAF 本身不规定清单与传输协议

- **FLV/RTMP** 是直播时代的经典组合：FLV 负责音视频 Tag 数据模型，RTMP 负责实时传输；HTTP-FLV 仍常用于直播播放链路

- **TS** 在广播、传统 HLS 和既有硬件链路中仍占重要位置；fMP4/CMAF 在点播、跨 HLS/DASH 复用和低延迟 HTTP 流媒体中更常见

- **NALU 的两种封装**（长度前缀 vs Annex B）是 AVC/HEVC 格式转换的核心

- **趋势：**点播与标准化分段流媒体大量采用 fMP4/CMAF；实时互动则常采用 WebRTC，可靠低延迟贡献链路还可能选择 SRT/RIST。QUIC/HTTP/3 是传输层/HTTP 映射，可以承载分段媒体，但不是 fMP4、FLV 或 TS 的直接“容器替代品”

---

## 参考资料

- [ISO/IEC 14496-12:2026：ISO Base Media File Format](https://www.iso.org/standard/85596.html)

- [W3C：ISO BMFF Byte Stream Format for Media Source Extensions](https://www.w3.org/TR/mse-byte-stream-format-isobmff/)

- [RFC 8216：HTTP Live Streaming](https://www.rfc-editor.org/rfc/rfc8216)

- [ISO/IEC 23000-19:2024：Common Media Application Format](https://www.iso.org/standard/85623.html)

- [Apple：HLS Authoring Specification for Apple Devices](https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices/)

- [Apple：About the Common Media Application Format with HLS](https://developer.apple.com/documentation/http-live-streaming/about-the-common-media-application-format-with-http-live-streaming-hls)

- [Apple：Enabling Low-Latency HLS](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-http-live-streaming-hls)

- [ITU-T H.264：Advanced Video Coding（Annex B 定义 Byte Stream Format）](https://www.itu.int/rec/T-REC-H.264)

- [ITU-T H.265 (01/2026)：High Efficiency Video Coding](https://www.itu.int/rec/T-REC-H.265)

- [ISO/IEC 14496-15:2024：NAL Unit Structured Video in ISO BMFF](https://www.iso.org/standard/89118.html)

- [ISO/IEC 14496-3:2019：MPEG-4 Audio](https://www.iso.org/standard/76383.html)

- [ITU-T H.222.0 (04/2025) / ISO/IEC 13818-1：MPEG-2 Systems](https://www.itu.int/rec/T-REC-H.222.0)

- [MPEG-4 Registration Authority：ISO BMFF Box、品牌与 Sample Entry 注册表](https://mp4ra.org/)

- [Adobe Flash Video File Format Specification 10.1（存档副本）](https://veovera.org/docs/legacy/video-file-format-v10-1-spec.pdf)

- [Adobe RTMP Specification 1.0（存档副本）](https://veovera.org/docs/legacy/rtmp-v1-0-spec.pdf)

- [Veovera Enhanced RTMP v2 Specification](https://veovera.org/docs/enhanced/enhanced-rtmp-v2.html)
