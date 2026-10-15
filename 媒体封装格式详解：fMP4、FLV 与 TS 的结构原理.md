# 媒体封装与实时传输详解：fMP4、FLV、TS、RTMP 与 WebRTC

本文按“编码基础 → 文件/流容器 → 实时传输与连通性 → 格式转换 → 工程检查”组织。多字节整数默认按对应协议的字节序读取；示例中的 `[数据]` 为占位符，不代表完整可解码码流。

|章节|内容|
|---|---|
|一|编码、NALU、Access Unit 与参数集|
|二～四|fMP4/CMAF、FLV、MPEG-2 TS 容器结构|
|五|RTMP 消息与 Chunk|
|六|RTP 头、WebRTC H.264 打包、RTCP 统计与反馈|
|七|STUN 报文及 ICE 用法|
|八|重封装、时间戳换算与配置映射|
|九|首屏、同步、时移、延迟与选型|

## 一、基础概念：编码、封装与传输

### 1.1 三层分工

媒体系统需要区分三个层次：

- **编码（codec）**把原始像素或 PCM 采样压缩成编码访问单元，例如 AVC/H.264、HEVC/H.265、AAC。

- **封装（container/mux）**把一个或多个编码流组织成可解析的字节结构，并描述轨道、时间戳、解码配置、随机访问点和数据位置，例如 MP4、FLV、MPEG-2 TS。

- **传输协议**负责把这些字节或媒体消息送到接收端，例如 HTTP、RTMP、SRT、RTP。RTMP 自身属于这一层，虽然它的音视频消息负载通常复用 FLV 数据模型。

同一组 H.264/AAC 编码访问单元可以在不重新编码的前提下，从 FLV 重封装为 fMP4 或 TS；但前提是目标容器支持该编码，并且时间戳、参数集和随机访问信息能够正确映射。只改文件后缀既不会完成重封装，也不会改变编码格式。

### 1.2 格式与协议定位

|格式/协议|规范体系|定位|典型应用|
|---|---|---|---|
|MP4|ISO/IEC 14496-12/14|ISO BMFF 文件封装|本地播放、点播|
|fMP4|ISO BMFF Movie Fragment|分片式文件/流封装|DASH、HLS、MSE|
|CMAF|ISO/IEC 23000-19|对 ISO BMFF 分片、轨道与切换关系的应用约束|跨 HLS/DASH 复用、低延迟分段媒体|
|FLV|Adobe FLV File Format|Tag 式文件/流封装|HTTP-FLV、RTMP 媒体负载模型|
|RTMP|Adobe RTMP Specification|基于 TCP 的消息与分块传输协议|直播推流、传统直播拉流|
|TS|ITU-T H.222.0 / ISO/IEC 13818-1|固定包长的传输流封装|广播、传统 HLS|
|Matroska|Matroska Specifications|EBML 容器|本地多轨存储|
|RTP / RTCP|RFC 3550 及负载/反馈规范|实时媒体传输与质量反馈|WebRTC、实时音视频|
|STUN|RFC 8489；ICE 使用见 RFC 8445|地址发现与连通性检查基础协议|WebRTC ICE/NAT 穿透|

### 1.3 容器需要描述的信息

- 关联编码数据与解码配置，例如 SPS/PPS、`AVCDecoderConfigurationRecord`、`AudioSpecificConfig`

- 表达解码时间与呈现时间，支持 B 帧重排序和音视频同步

- 描述样本边界、轨道归属、字节位置与随机访问点

- 组织视频、音频、字幕和元数据等多种轨道或基本流

这些元数据必须与编码内容一致：把 P 帧误标为随机访问点、写错采样率或时间戳，都会造成解码或同步错误。

### 1.4 NALU：编码数据的承载单元

在深入各格式之前，先区分三个经常混淆的概念：

- **NALU（Network Abstraction Layer Unit）**：H.264/H.265 码流的语法承载单元，用于把 VCL 切片数据或 SPS、PPS、SEI 等非 VCL 数据映射到存储和传输系统。

- **Slice（切片）**：一幅图像的一部分编码数据；一幅图像可以由一个或多个 Slice 组成。

- **Access Unit（访问单元）**：解码一幅图像所需的一组 NALU，可能包含 AUD、SPS/PPS、SEI 和多个 Slice NALU。

因此，“一个 NALU 就是一帧”并不成立。对 AVC/HEVC 视频，容器中的一个 video sample 通常对应一个 Access Unit，一个 sample 内可以有多个 NALU；对 AAC，sample 通常对应一个音频访问单元。这里的 “sample” 是**容器时间轴上的数据单元**，并不等于 PCM 的单个采样点。下面的 1 字节 Header 仅适用于 H.264；H.265/HEVC 的 NAL Header 为 2 字节，类型字段也不同。

**H.264 NAL Header（1 字节）结构：**

- `forbidden_zero_bit`（1 bit）

- `nal_ref_idc`（2 bit）：指示参考重要性；对 VCL NALU，0 表示所属图像不用于后续参考，非 0 表示参考图像。参数集等非 VCL NALU 另有取值约束

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

#### 两种常见字节流/样本表示：Annex B vs AVCC

这是理解各容器存储视频的核心区别。RTP 另有编码相关的打包语法，H.264 的单 NALU、聚合与分片格式见第 6.4 节。

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

- `hvc1`：参数集放在 Sample Entry 的配置记录中，sample 不应携带 VPS/SPS/PPS NALU。
- `hev1`：参数集可以位于 Sample Entry，也可以出现在 sample 内。

AVC 有对应的 `avc1`/`avc3` 区别。转封装时不能只复制 NALU：还必须根据 in-band 参数集的实际存在位置选择正确 Sample Entry，并在配置变化时更新初始化信息。把含 in-band VPS/SPS/PPS 的流误标成 `hvc1`，即使多数播放器“碰巧能播”，也不代表文件符合其信令语义。

---

## 二、fMP4（Fragmented MP4）格式详解

### 2.1 Box 基础与分片模型

fMP4 不是与 MP4 平行的另一套容器，而是 ISO BMFF（ISO/IEC 14496-12）的 **Movie Fragment** 组织方式。传统非分片 MP4 的逐 sample 索引主要位于 `moov` 的样本表中；fMP4 则把后续样本的索引拆到每个 `moof`，媒体字节仍放在 `mdat`。这样初始化信息可以先到达，后续分片可以边生成、边传输、边解析。

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

解析时检查 Size 不小于实际头部、不越过父 Box/文件边界，且 `offset + size` 不溢出。普通类型 `Size=1` 时头部为 16B，`uuid` 还需再加 16B。未知 Box 可按合法 Size 跳过。

#### Type 字段

Type 是 32 bit 标识，通常写成四字符码（FourCC）；`uuid` 还需额外的 16B user type。常见类型：

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

`stsd`、`meta` 等既有自身字段，又有 Sample Entry/子 Box；递归解析位置必须由具体语法决定。

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

在 [MSE ISO BMFF](https://www.w3.org/TR/mse-byte-stream-format-isobmff/#initialization-segments) 和 [HLS fMP4](https://www.rfc-editor.org/rfc/rfc8216#section-3.3) 中，初始化段的轨道不含媒体样本，但 `stsd` 仍需保留 Sample Entry；不能把“空样本表”理解成所有表项都清零。`moov` 还需包含 `mvex`；HLS 要求 `mvhd/tkhd.duration=0`，且 `mvex` 位于最后一个 `trak` 之后。一般 ISO BMFF 文件可以先有非分片样本、再追加 Fragment，不受这套初始化段约束。

视频 timescale 常取 90000，音频常取采样率，但并非固定要求。Init Segment 可在同一配置期间缓存；配置切换见第 9.1 节。

`ftyp/styp` 的 brand 声明格式兼容性；可播放性还取决于 Sample Entry、`CODECS`、加密方案与终端能力。

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

**基础记录的字段示意（非完整可解码配置）：**

```text
01 42 00 1E FF E1 [spsLength:2B] [SPS NALU]
01 [ppsLength:2B] [PPS NALU]
```

这里 `0x42` 表示 Baseline，`0x1E` 表示 Level 3.0，`0xFF` 指定 4B 长度前缀。分辨率必须解析 SPS，不能从 Profile/Level 字节直接推出；SPS/PPS 长度须按实际字节数填写。

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

`trun` 的 duration、size、flags 可逐 sample 给出，也可按 `tfhd → trex` 继承默认值；`sample_description_index` 按 `tfhd → trex` 选择。`first_sample_flags` 只覆盖 run 的第一个 sample，且不能与逐 sample 的 `sample_flags` 同时出现。缺省字段不等于 0。

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

自适应流媒体常将音视频分别寻址，以便独立选择码率和语言；也可以在同一 `moof` 中用多个 `traf` 复用轨道。`tfhd.track_ID` 关联初始化段中的 Track。同步时先用各自 timescale 换算，再结合编辑列表和协议层时间原点对齐。

### 2.7 Seek 机制

- `sidx` 可以记录子段的时间范围与字节偏移，但它是可选索引，不是 fMP4 Seek 的唯一入口。

- `sidx.earliest_presentation_time` 和 `subsegment_duration` 使用 `sidx.timescale`，不一定等于 Track timescale；`first_offset` 是从 **`sidx` Box 末尾**到第一项引用对象首字节的偏移，不是文件绝对偏移，也不是天然相对 `moof`。每个 reference 的 `referenced_size` 再把游标推进到下一对象。

- `reference_type=0` 表示引用媒体子段，`reference_type=1` 可以引用另一层 `sidx`；`starts_with_SAP/SAP_type/SAP_delta_time` 描述随机访问属性。实现不能把所有 reference 都直接当 `moof`，也不能只凭 `starts_with_SAP=1` 忽略 codec 的真实依赖。

- DASH/HLS 清单、HTTP Byte Range、`sidx`、文件尾的 `mfra/tfra`、服务端数据库或播放器自身索引，都可以先定位目标片段。

- 到达候选片段后，再根据 sample flags、SAP/随机访问点信息和编码依赖选择可解码起点。没有 `sidx` 并不必然意味着从文件头扫描全部 `moof`。

### 2.8 CMAF、Segment 与低延迟交付

CMAF 是 ISO BMFF 的应用约束，可由 HLS/DASH 复用；它不定义清单或网络协议。以下对象不能仅凭名称互换：

|对象|含义|
|---|---|
|Movie Fragment|ISO BMFF 的分片结构，以 `moof` 描述关联媒体样本|
|CMAF Fragment|满足 CMAF 约束、可独立解码的媒体对象；同一 Switching Set 可在相应边界切换|
|CMAF Chunk|Fragment 中连续的 sample 子集，通常由一对 `moof + mdat` 承载，不要求独立解码|
|CMAF Segment|同一 Track 的一个或多个完整 Fragment|
|HLS/DASH Media Segment|交付层对象，其边界与内部结构由相应应用格式规定|
|HLS Partial Segment（Part）|LL-HLS 的部分分段对象，可使用 CMAF Chunk 组织|

低延迟交付让较大 Segment 尚未完成时，其已编码的子对象就可传输和消费。

**一种常见的低延迟字节序列：**

```text
[styp]
[moof₁][mdat₁] ← 子对象 1 立即发送
[moof₂][mdat₂] ← 子对象 2 立即发送
[moof₃][mdat₃] ← 子对象 3 立即发送

```

这些子对象可能组成一个或多个 Fragment，须结合随机访问边界和具体 CMAF 约束判断。

LL-HLS 用 `EXT-X-PART` 描述 Partial Segment，Parent Segment 覆盖各 Part 的总时间范围；`EXT-X-PRELOAD-HINT` 是预告，并不保证最终发布。只有满足真实独立解码条件的 Part 才能独立起播；Parent Segment 独立不代表每个 Part 独立。

HLS fMP4 Segment 必须有适用的 `EXT-X-MAP`；该 Tag 持续生效直到被新的 Map 替换，无需逐段重复。配置切换时更新 Map，并按变化类型配合 `EXT-X-DISCONTINUITY`。

---

## 三、FLV（Flash Video）格式详解

### 3.1 顺序 Tag 模型

FLV 是顺序 Tag 流：文件头之后依次出现 Script、Audio、Video Tag。Tag 自带类型、负载长度和毫秒时间戳，因此接收端无需先得到全局样本表就能增量解复用。它不是严格意义上的“链表”——Tag 中没有 next 指针；可顺序前进依赖 `DataSize`，可辅助向后定位依赖 `PreviousTagSize`。

### 3.2 整体架构

```text
[FLV Header (通常 9B，长度由 DataOffset 指定)]
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

#### FLV Header 二进制格式（基础字段 9 字节）

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
│ [Length][NALU₁] [Length][NALU₂] ...               │
│ ← Length 宽度由配置指定；与 AVC sample 语法对应    │
└────────────────────────────────────────────────────┘

```

[Adobe FLV 规范 E.4.3.2](https://veovera.org/docs/legacy/video-file-format-v10-1-spec.pdf) 对 AVC NALU 媒体包明确要求完整帧（“Full frames are required”）：一个包可包含组成该帧的多个 NALU，不能把任意独立 Slice 分包称为合规 FLV。若需兼容这类非标准输入，应额外组帧或明确拒绝。

### 3.5 Audio Tag Data

|字段|大小|说明|
|---|---|---|
|SoundFormat(4b)+Rate(2b)+Size(1b)+Type(1b)|1B|10=AAC；传统 FLV 中 AAC 常将后三个字段写为 3/1/1，真实采样配置以 ASC 为准|
|AACPacketType|1B|0=SeqHeader, 1=Raw|
|Data|变长|—|

#### AACPacketType=0：AAC Sequence Header

负载为第 2.3 节介绍的 ASC，例如 AAC-LC/44.1 kHz/双声道常见 `12 10`，但长度并不固定。接收端须先获得适用的配置，再解码 Raw AAC；不能把整个 `esds` Box 复制到此处。

#### 音频帧大小

传统 FLV 的一个 `AACPacketType=1` Tag 通常承载一个 Raw AAC access unit。AAC-LC 最常见的帧长是 1024 个采样点，因此在 44.1 kHz 下常见时长为：

$$
\frac{1024}{44100}\times 1000 \approx 23.22\ \text{ms}
$$

但 GA 配置可使用 960 samples，其他 AOT、SBR/PS 和多 raw data block 还会改变 core/output 采样关系，不能靠固定 23.22 ms 反推时间戳。以 128 kbit/s、1024 samples、44.1 kHz 粗略估算，编码负载平均约 372 字节/帧；瞬时帧大小仍会随码率模式和编码内容变化。

### 3.6 时间戳与交织

AVC NALU Tag 的 Header Timestamp 为 DTS，`PTS = DTS + CompositionTime`，后者是有符号 24 bit 毫秒偏移；AAC 通常无显示重排序，可视为 `PTS=DTS`。音视频 Tag 通常大致按时间交织，但 FLV 不要求所有 Tag 全局严格按 DTS 排序。时间戳范围与回绕集中见第 8.2 节。

### 3.7 Seek 与索引

- `PreviousTagSizeN`：位于第 N 个 Tag 之后，值为该 Tag 的 `11 + DataSize`，不包含 `PreviousTagSize` 字段自身；`PreviousTagSize0` 紧跟 FLV Header，值为 0

- 可选的 `onMetaData.keyframes`：常用 `times` 与 `filepositions` 数组记录关键帧时间和文件字节位置，但直播流或普通 FLV 文件未必提供

- 文件 Seek 的典型流程：查关键帧索引 → 跳转到可 Seek Video Tag → 补齐/恢复 Sequence Header → 从该点继续解码。没有索引时只能扫描或由服务端维护外部索引

### 3.8 Enhanced RTMP/FLV 扩展

传统 FLV 的编码标识、多轨模型与毫秒时间精度有限。新编码可通过 Enhanced RTMP/FLV 扩展承载；厂商私有 CodecID 不能视为该规范的一部分。

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

## 四、MPEG-2 TS：固定包与节目复用

### 4.1 设计目标

MPEG-2 TS 面向广播、复用和连续传输场景。固定 188 字节包长与同步字节便于硬件处理、在字节流中重新找到包边界；PID、连续计数器和周期性 PSI 则帮助接收端识别流、检测丢包并在后续数据处恢复。基础 TS 本身不提供前向纠错或重传，“容易检测并重新同步”不等于“不会丢包”。

ITU-T H.222.0 / ISO/IEC 13818-1 定义的核心 TS packet 是 188 字节。192B M2TS 常在前面增加 4B 到达时间信息，204B 广播包常在后面增加 16B 纠错数据，解析前需识别变体。核心包仅 4B Header 的开销为 `4/188 ≈ 2.13%`，尚未计入 PES、PSI 与 adaptation field。

### 4.2 核心结构

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

### 4.3 PES（Packetized Elementary Stream）

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

### 4.4 PSI 表

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

### 4.5 PCR、PTS、DTS：三种时间各管什么

TS 的同步不能只看 PTS/DTS，还必须理解 PCR：

|字段|典型位置|作用|时基|
|---|---|---|---|
|PCR（Program Clock Reference）|`PCR_PID` 的 TS Adaptation Field|让接收端恢复编码器的系统时钟，驱动去抖动与解码缓冲|33 bit base（90kHz）+ 9 bit extension（27MHz）|
|DTS|PES Header|在系统解码模型中指定 Access Unit 的解码时刻|90kHz|
|PTS|PES Header|指定 Access Unit 的呈现时刻，用于音视频同步|90kHz|

PCR 字段实际占 6B：`base:33 + reserved:6 + extension:9`；其中 42 个有效位并非普通的二进制拼接计数：

$$
PCR_{27M} = 300 \times PCR_{base} + PCR_{ext}
$$

$$
t_{\text{seconds}} = \frac{PCR_{27M}}{27\,000\,000}
$$

`PCR_ext` 虽占 9 bit，但合规取值范围是 0～299；与 `PCR_base` 合并后形成 27 MHz 计数。

PCR 是系统时钟恢复基准，PTS/DTS 是相对于节目时间线的解码与呈现调度值。广播接收机可用 PCR 驱动 T-STD/去抖动时钟，再按 DTS 解码、按 PTS 呈现；某些文件或 HTTP 播放器会采用自己的时钟策略，但仍必须正确处理这些时间值。

不要用“最近一个 PCR 就是当前视频帧 PTS”的方式配对：PCR 可以位于单独 PID，也可以与某个 PES 共用 PID，它采样的是系统时钟而非 Access Unit 呈现时间。`discontinuity_indicator` 出现在 `PCR_PID` 时还可能宣布系统时基切换；接收端必须让下一 PCR 建立新时钟段，并同步重置相关抖动/展开状态。

PTS/DTS 与 `PCR_base` 都是 33 bit 循环计数；回绕周期、展开及 discontinuity 处理见第 8.2 节。

### 4.6 AVC/HEVC 字节流与参数集

AVC/HEVC 使用带 Start Code 的字节流表示；PES 和 TS packet 边界不保证与 NALU 或 Access Unit 边界重合。随机访问处常见如下排列：

```text
AVC： [Start Code][AUD] [Start Code][SPS] [Start Code][PPS]
      [Start Code][IDR Slice 1] [Start Code][IDR Slice 2] ...
HEVC：[Start Code][VPS] [Start Code][SPS] [Start Code][PPS]
      [Start Code][IDR 或 CRA 的 Slice NALU] ...
```

在随机访问点前重复参数集便于中途加入，但不是每个 IDR 都必须重复的通用语法要求。HEVC 的 CRA/IDR 差异见第 1.4 节；`random_access_indicator` 也不能替代真实编码依赖检查。

### 4.7 音频在 TS 中的携带

- 传统 HLS/TS 中的 AAC 常使用 ADTS Header（`protection_absent=1`、无 CRC 时 7 字节）+ AAC 原始访问单元

- ADTS 每帧携带 profile、采样率索引、声道配置和帧长等信息，比单次 ASC 更冗余但便于重新加入

- MPEG-2 TS 也定义了 LATM/LOAS 等其他 AAC 映射，不能把“TS 中 AAC 必然是 ADTS”当成通用规则

---

## 五、RTMP：消息分块与 FLV 负载传输

### 5.1 TCP 上的消息与分块传输

RTMP 在 TCP 连接中复用 Command、Control、Data、Audio 和 Video Message。音视频 payload 通常复用第三章的 FLV Tag Data；RTMP 线上没有 FLV 文件头、Tag Header 和 PreviousTagSize。

### 5.2 RTMP 整体链路

```text
TCP 连接
  └── RTMP Handshake: C0/C1/C2 <-> S0/S1/S2
        └── RTMP Chunk Stream
              ├── Control Message：窗口确认、带宽、Set Chunk Size、Ping 等
              ├── Command Message：connect、createStream、publish、play 等
              ├── Data Message：onMetaData 等
              └── Audio/Video Message：承载音视频负载

```

普通 RTMP 的默认端口为 1935；消息按 Chunk 在连接中交织。

### 5.3 握手格式

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

此处为 Adobe RTMP 1.0 的基础握手。带 HMAC 的 complex handshake 属于历史实现扩展，不能与表中的 `zero` 字段语义混用。

### 5.4 Chunk：RTMP 的传输基本单元

RTMP Message 可能很大，协议会把 Message 切成多个 Chunk 发送。每个 Chunk 由 Basic Header、Message Header、可选 Extended Timestamp 和 Chunk Data 组成。

```text
┌──────────────────────────────────────────────────────────┐
│ Basic Header │ Message Header │ Extended Timestamp? │ Data │
└──────────────────────────────────────────────────────────┘

```

#### Basic Header

首字节为 `fmt:2 + csid 编码:6`；Basic Header 共 1～3B：

|低 6 bit|额外字节|实际 CSID|
|---|---|---|
|2～63|无|等于该值；2 保留用于底层协议控制|
|0|1B：b1|`64 + b1`|
|1|2B：b1、b2|`64 + b1 + 256×b2`|

因此 CSID 扩展字节也有自己的低字节优先规则，不能把整个 RTMP 头都按大端读取。`fmt` 决定 Message Header：

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

### 5.5 Message Type：RTMP 承载什么

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

### 5.6 与 HTTP-FLV 的转换

重组 RTMP Message 后，把兼容的音视频 payload 写入 FLV Tag Data，将消息 timestamp 写入 Tag Header，再补 FLV Header、PreviousTagSize 等结构即可，无需重新编码。Script/Data Message 则需按具体 AMF 语义适配，不能保证原样复制。

RTMP 的 TCP 字节流按序可靠交付，但丢包后的缺口会阻塞后续数据，可能增加延迟。浏览器播放一般需要网关转换；实际延迟还取决于编码、排队和播放缓冲，不能仅由协议名称比较。

---

## 六、RTP / RTCP：媒体报文、H.264 打包与反馈

### 6.1 在 WebRTC 中的位置

RTP 按编码对应的 Payload Format 携带媒体；RTCP 提供统计、时钟映射和反馈；STUN 为 ICE 提供地址发现与连通性检查。它们都不是 MP4/FLV 一类的文件容器。WebRTC 的典型 UDP 媒体路径如下：

```text
H.264 Access Unit → NALU → RTP packetizer → SRTP → UDP/IP
                                      ↑
接收质量、丢包、刷新请求 ← RTCP/SRTCP

ICE：使用 STUN 检查候选地址对，必要时通过 TURN 中继
DTLS-SRTP：协商 SRTP/SRTCP 所需密钥材料
```

媒体通常由 SRTP 保护，控制报文由 SRTCP 保护；DTLS 用于密钥协商，并不是把每个 RTP 包作为 DTLS 应用数据发送。下文展示的是 RTP/RTCP 的逻辑格式；线上密文还须按协商的保护配置处理认证信息、索引等开销。RTP/RTCP 可复用同一传输端口，WebRTC 还会与 STUN、DTLS 复用底层传输，不能简单按 UDP 端口号判定协议。参见 [RFC 8834](https://www.rfc-editor.org/rfc/rfc8834)。

### 6.2 RTP 固定头：最少 12 字节

以下多字节整数均采用网络字节序（大端）。每行共 32 bit：

```text
| V:2 | P:1 | X:1 | CC:4 | M:1 | PT:7 | Sequence Number:16 |
|                       Timestamp:32                      |
|                          SSRC:32                        |
|                    CSRC:32 × CC（可选）                 |
|                    Header Extension（X=1）               |
|                    Codec-specific Payload               |
|                    Padding（P=1）                       |
```

|字段|含义|
|---|---|
|V|版本，当前格式为 2|
|P|末尾存在 RTP padding；最后 1 字节给出 padding 总字节数，包含该计数字节|
|X|CSRC 列表后存在扩展头|
|CC|CSRC 个数，0～15；每个 CSRC 为 4 字节|
|M|Marker，语义由负载格式定义；H.264 表示 Access Unit 的最后一个 RTP 包|
|PT|7 bit Payload Type，通过 SDP 映射到编码及其时钟频率；不是 NALU Type|
|Sequence Number|16 bit，每发送一个原始 RTP 数据包递增 1，用于检测丢失、乱序和重复，按模 65536 回绕|
|Timestamp|32 bit 媒体采样时间；时钟频率由负载格式决定，不是发送时刻或 Unix 时间|
|SSRC|32 bit 同步源标识；不同源独立维护序号和媒体时间线|
|CSRC|混音等场景下的贡献源列表，不是每条轨道都必带|

RTP 头长度为 `12 + 4×CC + 扩展头长度`。解析前必须逐段检查剩余字节，不能固定跳过 12 字节就当成媒体负载。P=1 时还要从尾部扣除合法的 padding；受保护的报文应先经过对应的安全处理。格式依据 [RFC 3550 §5.1](https://www.rfc-editor.org/rfc/rfc3550#section-5.1)。

**示例：单包承载一个 IDR NALU，并且它恰好结束该 Access Unit：**

```text
80 E0 12 34  00 01 5F 90  11 22 33 44  65 [NALU 剩余字节]
│  │  └─ seq=0x1234       └─ SSRC=0x11223344
│  └─ M=1, PT=96
└─ V=2, P=0, X=0, CC=0
Timestamp=0x00015F90=90000；Payload 首字节 0x65 为 H.264 NAL Header
```

此例假定 SDP 将 PT=96 映射为 H.264/90000；96 本身不具有固定的 H.264 含义。90000 代表相对媒体时钟的一秒刻度，但初始时间戳通常是随机偏移，不能据此推断“会话已开始一秒”。

### 6.3 RTP 扩展头与时间线

X=1 时，CSRC 列表后接以下结构：

```text
| Profile / Extension Identifier:16 | Length:16 |
| Extension Data：Length × 4 字节，包含对齐填充    |
```

这里 Length 以 **32 bit word** 为单位，不包含扩展头自身的 4 字节。扩展数据末尾对齐填充与 P 位指示的 RTP 尾部 padding 是两个独立概念。

[RFC 8285](https://www.rfc-editor.org/rfc/rfc8285) 定义的常用元素格式：

|形式|扩展标识|元素头|元素数据长度|
|---|---|---|---|
|One-byte|`0xBEDE`|`ID:4 + len:4`|`len+1`，即 1～16 字节；ID=0 用于 padding；遇 ID=15 时停止解析本扩展区|
|Two-byte|`0x1000～0x100F`，低 4 bit 为 appbits|`ID:8 + len:8`|len 字节，可为 0；ID=0 是单字节 padding|

MID、音频电平和用于传输反馈的序号等可通过扩展携带；元素 ID 必须按 SDP `a=extmap` 解释，不能写死“ID=3 永远是某字段”。

H.264 使用 90 kHz RTP 时钟，同一 Access Unit 的所有包使用相同 Timestamp；恒定 30 fps 时相邻图像通常相差 3000。音频使用各自编码规定的时钟频率，不能直接比较音频和视频的 Timestamp。RTP 时间戳不直接携带 DTS；含重排序的视频也不能把包序号或时间戳当成容器 DTS。

### 6.4 WebRTC 中 H.264 NALU 的 RTP 打包

#### 6.4.1 先确定编码和 packetization-mode

“NALU 在 RTP 中怎么打包”取决于编码。以下详细说明 **H.264 / RFC 6184**，不能直接套用于 H.265，也不能套用于 VP8、VP9 或 AV1。WebRTC 的 H.264 实现必须支持 `packetization-mode=1`，其他模式可协商使用，见 [RFC 7742 §6.2](https://www.rfc-editor.org/rfc/rfc7742#section-6.2)。

```text
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=42e01f;level-asymmetry-allowed=1
```

这只是相关 SDP 行示例，不是完整 SDP。`packetization-mode` 未提供时默认是 0，不能因为“这是 WebRTC”就默认为 1。

|模式|名称|允许的主要 RTP payload 结构|
|---|---|---|
|0|Single NAL Unit Mode|Single NALU；不能使用 STAP-A 或 FU-A|
|1|Non-interleaved Mode|Single NALU、STAP-A、FU-A；按 NALU 解码顺序发送|
|2|Interleaved Mode|STAP-B、MTAP16/24、FU-A/FU-B 等，涉及 DON；不是常见 WebRTC H.264 路径|

RTP payload **不携带 Annex B Start Code，也不直接沿用 MP4/FLV 的长度前缀**。先从源样本提取 NALU，再按协商模式打包；NALU 内部的 EBSP 字节（包括 `emulation_prevention_three_byte`，即防止伪 Start Code 的 `03`）保持不变。

#### 6.4.2 Single NALU：一个包放一个完整 NALU

当一个 NALU 能放入负载预算时：

```text
[RTP Header][NAL Header:1B][NAL Body]
             F:1 | NRI:2 | Type:5
```

第一个 payload 字节就是原始 NAL Header，Type 为 1～23 范围内的适用 NALU 类型。包里不再额外写 NALU 长度，因为剩余 payload 边界已由 RTP 包确定。

```text
Annex B 输入：00 00 00 01 65 AA BB CC
RTP payload：            65 AA BB CC
```

即使使用 Single NALU，也不意味着“一个 RTP 包就是一帧”：一个 Access Unit 可以包含多个 Slice NALU，它们可分别占包。M 只应在整个 Access Unit 的最后一个包置 1。

#### 6.4.3 STAP-A：多个小 NALU 聚合进一个包

STAP-A 的外层 Type=24，不是编码器输出的 Slice 类型。格式为：

```text
[RTP Header]
[STAP-A Header:1B = F:1 | NRI:2 | Type=24:5]
[NALU 1 Size:2B][完整 NALU 1]
[NALU 2 Size:2B][完整 NALU 2]
...
```

每个 Size 是大端 16 bit，包含该 NALU 自己的 1B Header，不包括 Size 字段。外层 NRI 取内部 NALU 的最大 NRI；F 为内部 F 的逻辑或，正常无错误码流为 0。STAP-A 内 NALU 共享同一个 RTP 时间戳，常用于聚合较小的参数集，不能把不同采样时间的图像任意塞入同一 STAP-A。

```text
78 00 04 67 AA BB CC 00 03 68 DD EE
│  │     └─4B NALU  └─3B  └─PPS Header
│  └─第一个 Size=4
└─F=0, NRI=3, Type=24
```

以上只演示字节边界，SPS/PPS body 是占位字节，不是可解码参数集。接收端拆出 `67 AA BB CC` 和 `68 DD EE`；STAP-A 的外层 Header 和两个 Size 不送入视频解码器。若后面还有同一 Access Unit 的 Slice 包，此聚合包 M=0。

#### 6.4.4 FU-A：一个大 NALU 拆成多个 RTP 包

FU-A 的负载头为 2 字节：

```text
[RTP Header][FU Indicator:1B][FU Header:1B][NAL Body Fragment]

FU Indicator：F:1 | NRI:2 | Type=28:5
FU Header：   S:1 | E:1 | R:1 | Original NAL Type:5
```

S=1 表示 NALU 的起始分片，E=1 表示结束分片，R 发送时置 0。S 与 E 不可同时为 1；能放入单包时直接采用 Single NALU。原始 NAL Header 被拆分表示在这两个字段中，**不能再次作为首分片数据写入**。规则见 [RFC 6184 §5.8](https://www.rfc-editor.org/rfc/rfc6184#section-5.8)。

假设原始 NAL Header 为 `0x65`（F=0、NRI=3、Type=5），分成三包：

```text
原始 NALU：65 [A][B][C]

seq=1000, timestamp=T, M=0：7C 85 [A]  ← S=1,E=0,Type=5
seq=1001, timestamp=T, M=0：7C 05 [B]  ← S=0,E=0,Type=5
seq=1002, timestamp=T, M=1：7C 45 [C]  ← S=0,E=1,Type=5

恢复 Header：(0x7C & 0xE0) | (0x85 & 0x1F) = 0x65
恢复 NALU： 65 [A][B][C]
```

这里假定该 NALU 是 Access Unit 的最后一个 NALU；如果后面还有同一图像的其他 Slice，则第三包虽 E=1，仍必须 M=0。**E 标识 NALU 结束，M 标识 Access Unit 结束。**同一 NALU 的分片共享时间戳，按连续 RTP 序号发送；网络到达顺序则可能乱序。

#### 6.4.5 MTU 预算、重组与丢包

设 RTP payload 预算为 B，FU-A 每包最多携带 `B−2` 字节原始 NAL Body。若完整 NALU 长度为 N（包含 1B Header），分片数量为：

```text
ceil((N − 1) / (B − 2))
```

例如预算 B=1200B、原始 NALU=3001B，3000B Body 拆成 `1198 + 1198 + 604`；三个 RTP payload 分别为 `1200、1200、606B`。若 RTP 头均为 12B，则保护前 RTP 包分别为 `1212、1212、618B`。

1200B 只是此处设定的 **payload 预算**，不是协议固定值。真实预算应从路径 MTU 扣掉 IP、UDP、RTP 固定头、CSRC、扩展头以及 SRTP 和可能的中继/隧道开销。不要把“UDP 数据报大小”“RTP 包大小”和“媒体片段大小”混作同一值。

接收路径通常为：

```text
SRTP 校验/解密 → 按 SSRC 与序号缓存、去重、处理乱序
 → 按 payload 首字节 Type 选择 Single / STAP-A / FU-A
 → 验证长度或分片连续性，重组 NALU
 → 按时间戳、M 和编码语法组装 Access Unit
 → 结合参数集和播放截止时间送入解码器
```

遇到 FU-A 缺片，不能把前后残片直接拼成一个“完整 NALU”。应等待截止时间内的恢复，或按损坏处理策略丢弃该 NALU；必要时通过 PLI 请求恢复画面。M=1 不能证明此前的包均已收到；[RFC 6184 §5.1](https://www.rfc-editor.org/rfc/rfc6184#section-5.1) 也要求解码端不能只依赖 M 判断 AU 边界，应结合时间戳和编码语法。序号要展开并区分重复、乱序和源重启；RTX 包还需按协商的映射恢复原序号及媒体流归属后参与重组。

#### 6.4.6 参数集与 HEVC 边界

WebRTC H.264 要求通过 in-band 方式发送参数集，因此 SPS/PPS 可以通过 Single NALU 或 STAP-A 发送；生成的 SDP 不应包含 `sprop-parameter-sets`，参见 [RFC 7742 §6.2](https://www.rfc-editor.org/rfc/rfc7742#section-6.2)。它们需要在引用这些参数集的图像解码前到达；不能把 FLV AVC Sequence Header 或整个 `avcC` Box 当作 RTP 媒体 payload 原样发送。

向容器转换时须先恢复完整 NALU 与 Access Unit，再建立 sample 和时间线，具体映射见第 8.3、8.5 节。

HEVC/H.265 有独立的 [RFC 7798](https://www.rfc-editor.org/rfc/rfc7798)：NAL Header 为 2B，聚合 AP 使用 Type=48，分片 FU 使用 Type=49，且在适用配置下涉及 DONL 等字段。不能用 H.264 的 Type=24/28 或 `0x65 → 0x7C` 规则解析 HEVC；是否能在具体 WebRTC 会话使用 HEVC，还取决于双方能力与 SDP 协商。

### 6.5 RTCP 通用头与 SR / RR

RTCP 通用头为 4 字节，不能按 RTP 的 12 字节固定头解析：

```text
| V:2 | P:1 | Count / FMT:5 | Packet Type:8 | Length:16 |
|                  Packet-specific Body               |
```

`Length = 本 RTCP 包总字节数 / 4 - 1`，包括通用头和本包 padding；它不是字节数，也不是整个 compound RTCP 的长度。Count 在 SR/RR 中为 report block 数，在反馈包中为 FMT。常见 Packet Type：

|PT|名称|主体结构 / 用途|
|---|---|---|
|200|SR，Sender Report|发送者 SSRC + 20B Sender Info + 0～31 个 24B Report Block|
|201|RR，Receiver Report|报告者 SSRC + 0～31 个 24B Report Block|
|202|SDES|源描述 chunk，如 CNAME，用于源关联|
|203|BYE|离开会话的源列表，可带原因|
|204|APP|应用自定义内容|
|205|RTPFB|传输层反馈，例如 Generic NACK|
|206|PSFB|负载相关反馈，例如 PLI|

无 padding、无额外 profile 扩展时，SR 总长为 `28 + 24×RC`，RR 为 `8 + 24×RC`。SR 通用头之后的主体如下（Sender Info 为其中 NTP 至 Octet Count 的 20B）：

```text
Sender SSRC:32
NTP Timestamp:64          ← 同一时刻的墙钟表示
RTP Timestamp:32          ← 该时刻对应的媒体时钟
Sender's Packet Count:32
Sender's Octet Count:32   ← RTP payload 字节累计，不含 RTP 头和 padding
[Report Block × RC]
```

每个 24 字节 Report Block：

```text
|                  被报告的源 SSRC:32                    |
| Fraction Lost:8 | Cumulative Packets Lost:24（有符号） |
|          Extended Highest Sequence Number:32          |
|                 Interarrival Jitter:32                |
|                         LSR:32                        |
|                        DLSR:32                        |
```

Fraction Lost 表达上次报告以来的丢包比例，单位为 1/256；累计丢包数是有符号值，重复包可能使它为负。Jitter 使用该 RTP 流的时间戳单位。LSR 是最近收到 SR 的 NTP 时间戳中间 32 bit；DLSR 是从收到该 SR 到发送此报告的间隔，单位为 1/65536 秒。发送方在收到反馈时，用同样的短 NTP 表示计算 `RTT ≈ A − LSR − DLSR`，须处理回绕和没有有效 SR 的情况。

音视频同步需要各流 SR 中的 NTP↔RTP 映射，并结合源关联信息把不同媒体时钟对齐；不能只比较 RTP Timestamp。SR/RR 格式与统计定义见 [RFC 3550 §6.4](https://www.rfc-editor.org/rfc/rfc3550#section-6.4)。

### 6.6 NACK、PLI 与复合 RTCP

反馈报文的公共结构来自 [RFC 4585](https://www.rfc-editor.org/rfc/rfc4585)：

```text
[RTCP Header：FMT + PT + Length]
[Sender SSRC:32]
[Media Source SSRC:32]
[Feedback Control Information（FCI）]
```

- **Generic NACK：PT=205，FMT=1。**每个 FCI 项为 `PID:16 + BLP:16`。PID 指出一个丢失序号，BLP 的 bit i 指出 `PID+i+1` 是否也丢失，序号按 16 bit 回绕；bit=0 并不保证对应包已经收到。
- **PLI：PT=206，FMT=1。**FCI 为空，总长 12B，Length=2。它告知发送者图像依赖可能损坏，请求恢复可解码画面；不指定某个丢失包，也不强制发送端每次立即产生 IDR。

```text
81 CD 00 03  11 22 33 44  AA BB CC DD  12 34 00 05
   PT=205    Sender SSRC  Media SSRC   PID   BLP
FMT=1, Length=3 → 16B；请求序号 0x1234、0x1235、0x1237

81 CE 00 02  11 22 33 44  AA BB CC DD
   PT=206    Sender SSRC  Media SSRC
FMT=1, Length=2 → 12B；PLI，无 FCI
```

基础 RTCP 常将多个包串成一个 compound，例如 `[SR][SDES]`；解析器应按每个包自身 Length 前进。支持并协商 reduced-size RTCP 时，某些反馈可以单独发送，不应把“所有 RTCP 数据报都必须以 SR/RR 开头”作为 WebRTC 的绝对判定规则。NACK 是反馈，是否重传及采用何种 RTX 映射还由协商和发送端策略决定；RTP 本身不提供可靠交付保证。

---

## 七、STUN：报文格式与 ICE 连通性检查

### 7.1 固定头：20 字节

STUN 是请求/响应及指示消息协议，核心格式由 [RFC 8489 §5](https://www.rfc-editor.org/rfc/rfc8489#section-5) 定义：

```text
| 0:2 |       Message Type:14       | Message Length:16 |
|                 Magic Cookie:32 = 0x2112A442           |
|                                                       |
|                   Transaction ID:96                   |
|                                                       |
|               Attributes：TLV 序列，4 字节对齐          |
```

|字段|规则|
|---|---|
|Message Type|最高 2 bit 固定为 0；其余位交织编码 Method 与 Class|
|Message Length|头后属性区字节数，包含属性 padding，不包含 20B 固定头；必须为 4 的倍数|
|Magic Cookie|固定 `21 12 A4 42`，用于识别报文及 XOR 地址计算|
|Transaction ID|96 bit 事务标识；响应匹配请求，同一事务重传保持此标识|

Type 的布局从最高位到最低位为：

```text
0 0 M11 M10 M9 C1 M8 M7 M6 C0 M5 M4 M3 M2 M1 M0

Class C1C0：00=Request，01=Indication，10=Success，11=Error
Binding Method=0x001：
  Request=0x0001   Indication=0x0011
  Success=0x0101   Error=0x0111
```

因此 Type 不能按“高字节是方法、低字节是类型”拆开。仅凭最高两位是 0 也不足以证明是合法 STUN，还要核验 Cookie、长度、事务上下文与相应认证。

### 7.2 Attribute：TLV 与 padding

```text
| Attribute Type:16 | Attribute Length:16 |
| Value：Attribute Length 字节            |
| Padding：0～3 字节，对齐到 4B            |
```

Attribute Length 只计算 Value，不包括 4B TLV 头，也不包括 padding。下一个属性偏移为 `当前偏移 + 4 + ((Length+3) & ~3)`。例如 5B Value 的属性实际占 12B：4B 头 + 5B 值 + 3B padding。

Type `0x0000～0x7FFF` 属于 comprehension-required，未知时按消息处理规则处理，不能一律忽略；`0x8000～0xFFFF` 属于 comprehension-optional。常见属性如下：

|Type|属性|Value 格式 / 用途|
|---|---|---|
|0x0006|USERNAME|变长用户名；ICE 检查中使用 ufrag 组合|
|0x0008|MESSAGE-INTEGRITY|20B HMAC-SHA1，证明持有共享凭据|
|0x0009|ERROR-CODE|前 4B 为保留位、Class 和 Number，后接原因文本；错误码为 `Class×100+Number`|
|0x0014 / 0x0015|REALM / NONCE|长期凭据认证使用的域与挑战值|
|0x001C|MESSAGE-INTEGRITY-SHA256|HMAC-SHA256；按适用规则允许截断，不应总按 32B 固定解析|
|0x0020|XOR-MAPPED-ADDRESS|响应中报告观察到的源 IP/端口|
|0x0024|PRIORITY|4B，若产生 peer-reflexive candidate 时赋给它的优先级|
|0x0025|USE-CANDIDATE|0B，ICE controlling 端用于提名候选地址对|
|0x8028|FINGERPRINT|4B，CRC-32 与 `0x5354554E` 异或|
|0x8029 / 0x802A|ICE-CONTROLLED / ICE-CONTROLLING|8B tie-breaker，声明 ICE 角色并处理角色冲突|

MESSAGE-INTEGRITY 与 FINGERPRINT 的覆盖范围不同：前者计算 HMAC 时覆盖该属性之前的消息字节，并将头部 Message Length 临时调整到该完整性属性末尾；不把该属性本身纳入 HMAC 输入。FINGERPRINT 必须位于最后，CRC 覆盖它之前的消息字节，头部 Length 则包含 FINGERPRINT。FINGERPRINT 不是密码学认证；XOR 地址编码也不是加密。完整性计算的细节见 [RFC 8489 §14.5～14.7](https://www.rfc-editor.org/rfc/rfc8489#section-14.5)。

### 7.3 XOR-MAPPED-ADDRESS 的位布局与实例

```text
| Reserved:8=0 | Family:8 | X-Port:16 |
| X-Address：IPv4 为 32 bit，IPv6 为 128 bit |
```

- Family=1：IPv4；`Port = X-Port XOR 0x2112`，`IPv4 = X-Address XOR 0x2112A442`。
- Family=2：IPv6；端口规则相同，地址与 `Magic Cookie || Transaction ID` 拼成的 128 bit 掩码异或。
- IPv4 属性 Value 长 8B、整个 TLV 长 12B；IPv6 分别为 20B 和 24B。

以下是仅含一个地址属性的 Binding Success 格式示例，**省略认证属性，不是完整 ICE 成功响应**：

```text
01 01 00 0C  21 12 A4 42  01 02 03 04 05 06 07 08 09 0A 0B 0C
└─Success    └─Cookie      └─Transaction ID（12B）
      └─Message Length=12B；整个消息=32B

00 20 00 08  00 01 F5 23  EA 12 D5 45
Type  Len    IPv4 X-Port   X-Address

0xF523 XOR 0x2112 = 0xD431 = 54321
EA 12 D5 45 XOR 21 12 A4 42 = CB 00 71 07 = 203.0.113.7
```

这里报告的是服务端从该事务观察到的源地址 `203.0.113.7:54321`，并不保证其他对端使用这个地址就一定能穿透 NAT。

### 7.4 STUN、ICE、TURN 的分工

ICE 将本地、server-reflexive 和 relayed 等候选地址配对，通过 STUN Binding 请求验证真实路径，并提名使用的候选地址对。向 STUN 服务端查询映射地址，与向对端执行 ICE connectivity check，虽然都用 Binding，但事务对象、属性与认证上下文不同。

ICE 检查的 USERNAME 通常为 `接收方 ufrag:发送方 ufrag`，请求完整性校验使用接收方的 ICE password；这些凭据由信令交换，不是用户登录密码。PRIORITY、角色属性和 USE-CANDIDATE 也属于 ICE 对 STUN 的具体用法，不能要求所有普通 STUN Binding 请求都携带它们。参见 [RFC 8445](https://www.rfc-editor.org/rfc/rfc8445)。

STUN 不转发媒体。直连不可用时，TURN 提供中继；ICE 负责候选检查与选择，而 SRTP/SRTCP 负责选定路径上的媒体保护。连通性检查成功也不代表解码器已经获得 SPS/PPS 或可解码图像，这些仍属于媒体链路。

---

## 八、格式转换：边界、配置与时间线

### 8.1 先判断重封装还是转码

```text
采集 → 编码访问单元 + 时间信息
        ├─ fMP4 / FLV / TS 封装 → HTTP / RTMP / SRT 等交付
        └─ 编码专用 RTP 打包 → SRTP → WebRTC 接收端
接收 → 解复用或 RTP 重组 → 解码 → 同步呈现
```

**重封装**保留编码数据，重建边界、配置与时间戳；**转码**先解码再编码。协议网关可能同时完成重封装，例如 RTMP→HTTP-FLV。应先核对目标编码、Profile/Level、像素格式、音频配置与加密能力；支持 H.264 并不等于支持其所有 Profile。RTP 既可按本章所述直接承载编码媒体，也可以按另一种 Payload Format 承载 TS，二者不是同一条打包路径。

### 8.2 时间戳换算：先展开，再缩放，最后重建

先确定源时基、时间原点及字段语义：

|来源|常见时间单位|
|---|---|
|FLV / RTMP|毫秒|
|MPEG-2 TS PTS/DTS|1/90000 秒|
|fMP4 视频 Track|由 `mdhd.timescale` 决定，常见 90000|
|fMP4 AAC Track|由 `mdhd.timescale` 决定，常见等于采样率|
|H.264 RTP|1/90000 秒；随机初值，Timestamp 不直接提供 DTS|

常见范围如下；**循环计数器的回绕与文件字段容量耗尽不同**：

|字段|边界|
|---|---|
|FLV / RTMP 毫秒时间|FLV 低 24 bit 约 4.66 小时进位；无符号 32 bit 周期约 49.71 天；FLV 规范 SI32 正值范围约 24.86 天|
|TS PTS/DTS/PCR base|33 bit、90 kHz，约 26.51 小时回绕|
|H.264 RTP Timestamp|32 bit、90 kHz，约 13.26 小时回绕；序号另按 16 bit 回绕|
|fMP4 `tfdt` v0|32 bit；90 kHz 下约 13.26 小时耗尽，不能直接按 RTP 方式静默回绕，应使用 v1 的 64 bit 字段|

仅换算单位且保持同一时间原点时：

$$
T_{dst} = \operatorname{round}\left(
\frac{T_{src}\,S_{dst}}{S_{src}}
\right)
$$

若源与目标原点不同，应换算时间差：`T_dst = O_dst + round((T_src−O_src)×S_dst/S_src)`。RTP→文件还要建立连续时间原点，音视频之间利用 SR 等映射对齐。实现时依次处理：

1. **重建并展开计数**：RTMP 先根据 fmt Header 的绝对 timestamp 或 delta 重建 Message 时间，再把 TS 的 33 bit、RTP 和 RTMP/FLV 的 32 bit 位模式恢复成连续的较宽整数。若先缩放再判断回绕，边界附近很容易被误判。
2. **识别 discontinuity**：先排除包乱序和视频呈现重排序；含 B 帧的 PTS/RTP Timestamp 按解码顺序观察时可能正常回退。确认是回绕后才连续展开；重连、Seek、拼接或源重启则按会话与外部信令判断是否建立新时间线。
3. **分别保存 DTS 与 PTS**：对视频计算 $CTO = PTS-DTS$；不能只保留一个“timestamp”后猜另一个。
4. **用整数有理数换算**：避免浮点累计误差；实现应先约分或使用足够宽的中间整数，防止乘法溢出。逐帧 duration 的余数应累计到后续帧，或者直接缩放累计时间边界，再由相邻边界相减。
5. **按 Track 独立量化**：视频和音频分别在自己的 timescale 上生成单调 DTS，再映射到共同呈现时间。音频 44.1 kHz 与视频 90 kHz 不能逐包硬凑成相同整数。

对 $n$ bit 循环计数器，若已经排除 discontinuity，且相邻真实时间差的绝对值小于半个周期，可用最短有符号差展开：

$$
d_k =
\left(
\left(x_k-x_{k-1}+2^{n-1}\right)\bmod 2^n
\right)-2^{n-1}
$$

$$
X_k = X_{k-1}+d_k
$$

其中 $x_k$ 是循环值，$X_k$ 是展开后的宽整数。使用 RTP 时还须先处理包乱序；超过半周期的断流或源重启应借助外部时间线重新定锚。该公式不适用于把 `tfdt` 溢出解释成合法回绕。

例如 FLV 中 `DTS=1000 ms`、`CompositionTime=67 ms` 的 AVC Tag，保持时间原点转到 90 kHz 视频 Track 后得到：

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

### 8.3 H.264/HEVC 转封装：边界转换不等于重新编码

#### Annex B → 长度前缀

TS 转 fMP4/FLV 时，通常需要：

1. 从 PES payload 中组装完整 Access Unit，不能把每个 TS packet 当成一帧。
2. 扫描 3 或 4 字节 Start Code，取出每个 NALU。
3. 解析 Annex B 的 Start Code 及其外层 leading/trailing zero 字节，提取准确的 NALU 边界；按目标配置写入长度前缀，再写 NALU 本体。不要把 Start Code 附近的外层零字节计入 NALU，也不要删除 NALU 内的 `emulation_prevention_three_byte`（`03`）。
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

#### RTP ↔ 容器

发送前从 sample/字节流提取 NALU，按第 6.4 节打包；接收时先去聚合或重组，再加 Start Code 或长度前缀。RTP Timestamp 不直接提供 DTS，含 B 帧时不能无条件写 `DTS=PTS`；必须结合可靠的解码顺序与时间信息构建 sample。

### 8.4 AAC 转封装：ASC、Raw AAC 与 ADTS

FLV/fMP4 用初始化配置携带 ASC、媒体单元承载 Raw AAC；传统 TS/HLS 常用 ADTS。转换重点是配置表达能力和 raw data block 边界。

TS/ADTS 转 FLV 或 fMP4 时，应解析 ADTS 的 profile、sampling frequency index、channel configuration、13 bit `aac_frame_length`、CRC 标志和 `num_raw_data_blocks_in_frame`；`aac_frame_length` 包含 ADTS Header 与编码负载总长。当 `num_raw_data_blocks_in_frame=0` 时，帧内为单个 raw data block，无 CRC/有 CRC 的头部处理分别为 7B/9B。多 raw data block 且带 CRC 时，还涉及位置字段与各 block 的校验，不能统一只剥 9B；应按完整 ADTS 语法提取，再按目标格式重建访问单元。

ADTS Header 的 2 bit profile 只直接表达有限的基础 AOT，无法完整携带任意 ASC。HE-AAC 的 SBR/PS 可能通过编码负载中的隐式/同步扩展出现，`channel_configuration=0` 还需要 Program Config Element。因此，从 ADTS 生成 ASC 时必须结合码流或可靠的上游配置；只由头部三项拼一个 ASC，可能把 HE-AAC 错标成 AAC-LC。反向转换也要先确认目标 AOT、频率和声道布局能够用 ADTS 表达。

不能把 ADTS Header 当成 AAC 编码负载写入 MP4 sample，也不能只复制一个固定的 `12 10` 去处理所有音频。若目标格式无法无损表达源配置，应保留为适用的原格式、转码或显式拒绝，而不是静默降级配置。

### 8.5 常见转换路径的字段映射

|转换路径|可以直接复用的内容|必须重建的内容|
|---|---|---|
|RTMP → HTTP-FLV|Audio/Video Message payload|FLV Header、Tag Header、timestamp、PreviousTagSize|
|FLV/RTMP → fMP4|受双方支持的 AVC/HEVC NALU、Raw AAC 和 codec configuration record|区分 legacy/Enhanced 头，重建 `ftyp/moov`、Track/Sample Entry、`moof/trun/tfdt`、sample flags、mdat 偏移|
|TS → fMP4|解出的 AVC/HEVC NALU、可兼容的 AAC 编码负载|PAT/PMT/PES 解复用、时间戳展开、Annex B→长度前缀、ADTS→ASC、Fragment 索引|
|fMP4 → TS|sample 内编码数据与 DTS/PTS|长度前缀→Annex B、ASC→ADTS、PES、PAT/PMT、PCR、PID/continuity counter|
|H.264 RTP → fMP4|完整重组后的 NALU|去聚合/重组、AU 边界、SPS/PPS、长度前缀、DTS/PTS 与 sample flags|

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

在媒体紧随 `moof`、寻址基址为 `moof` 的常见布局中，`trun.data_offset` 依赖最终 `moof` 大小和 `mdat` 头长度，因此通常先测量结构再回填。sample duration 则通常要等下一个 DTS 才能精确知道；实时打包器需要保留至少一个待定 sample，或采用已知恒定 duration，结束时再处理最后一个 sample。

#### TS → fMP4 的额外状态

TS 输入还要维护：

- PAT/PMT 版本、Program、stream PID 与 PCR PID
- 每个 PID 的 continuity counter 和 PES 重组缓冲
- 33 bit PTS/DTS/PCR base 的展开状态，以及 PCR extension
- AVC/HEVC Access Unit 组装器、随机访问状态与最新参数集
- 音视频各自的 Track 时间线和分片边界

丢失带 `PUSI=1` 的 PES 起始包后，不应把后续中间 payload 冒充完整 sample；应丢弃到可恢复边界，并把缺口传播成 discontinuity 或损坏状态。

---

## 九、工程检查与格式选择

### 9.1 首屏、中途加入与配置切换

首屏与中途加入都需要匹配的解码配置和可随机访问的媒体。配置切换必须保持二者版本一致：

|格式|初始化/信令|随机访问起点|配置变化时|
|---|---|---|---|
|fMP4|Init Segment：`ftyp + moov`|sample flags/SAP 与编码随机访问图像|发布匹配的新 Init Segment，清单切换 `EXT-X-MAP`/表示；必要时声明 discontinuity|
|FLV/RTMP|Legacy AVC/AAC Sequence Header，或 Enhanced FourCC SequenceStart，常配合 metadata|实际可随机访问的 Video Tag/Message|先发新的配置消息，再发使用新配置的媒体；服务端缓存也要原子更新|
|TS|PAT/PMT + in-band 参数集/音频头|随机访问 Access Unit，可辅以 `random_access_indicator`|PMT 内容改变时更新版本；处理参数集及时间基变化|
|WebRTC H.264|SDP 协商 + in-band SPS/PPS|收到完整可解码 AU；必要时请求 PLI|在媒体使用前发送新参数集；超出协商能力时重新协商|

例如新分辨率的 sample 不能继续使用不匹配的旧 `avc1` 配置。服务端应协调初始化信息、媒体与清单的发布/缓存更新；FLV 可缓存配置与可解码 GOP，TS 可在分段边界重复 PSI/参数集，WebRTC 还需完成 ICE 与媒体密钥协商。

关键帧也要区分三层：

- 编码层：IDR、CRA、恢复点、开放/闭合 GOP 等具有不同依赖语义。
- 容器层：FLV FrameType、ISO BMFF sample flags/SAP、TS `random_access_indicator`。
- 协议层：HLS Segment/Partial Segment、DASH Segment/Representation 的独立性声明。

只有三层信令与实际依赖一致，播放器才能安全首播、Seek 和 ABR 切换。

若无缓存或主动刷新，只能等待周期性 IDR，2 秒 IDR 间隔意味着理想条件下最多约 2 秒的额外等待；丢包、建连与缓冲仍会增加实际首屏时间。

### 9.2 音视频不同步与损坏定位

先按各轨时基和时间原点对齐呈现时间，再定位异常层：

|链路|主要检查点|
|---|---|
|fMP4|`tfdt`、duration、CTO、Edit List 与清单时间线；初始化配置是否匹配|
|FLV/RTMP|Chunk 重组、DTS 与有符号 CompositionTime；不能只比较相邻音视频 Tag|
|TS|CC/TEI、PES 完整性、PCR 连续性、PTS/DTS 及 discontinuity|
|WebRTC|序号缺口、FU 重组、播放截止时间、RTCP SR 映射与源关联；STUN 成功不代表解码条件已齐备|

遇到计数边界按第 8.2 节展开；重连、Seek、源重启或拼接则应明确重建时间线。持续同步漂移还应检查采集时钟及重采样策略。

### 9.3 Seek 与直播时移

Seek 需要“时间→数据位置”的索引、对应解码配置和随机访问点。fMP4 可用清单、`sidx/tfra` 或外部索引；FLV 可用关键帧索引；TS/HLS 常用 Playlist。直播时移还需要服务端保留历史媒体窗口，单靠格式无法实现。实时 RTP 会话若要回看，通常另建录制和索引链路。

### 9.4 端到端延迟

端到端直播延迟可以粗略拆成：

$$
L \approx L_{\text{capture}}
+ L_{\text{encode}}
+ L_{\text{GOP/packaging}}
+ L_{\text{network/CDN}}
+ L_{\text{player buffer}}
+ L_{\text{decode/render}}
$$

CMAF Chunk/HLS Part 的时长影响媒体何时可交付；RTMP Chunk Size 则是字节分块大小，主要影响消息交织与调度，两者不能等同。B 帧重排、等待随机访问点、网络重传和播放缓冲都可能主导延迟，应沿采集、编码、发布、接收、呈现各节点测量。

### 9.5 按交付需求选择

|需求|常见组合|主要约束|
|---|---|---|
|HTTP 点播 / 自适应直播|fMP4/CMAF + HLS/DASH|初始化段、随机访问边界、清单和终端 codec 支持|
|既有直播推流与 HTTP-FLV|RTMP + FLV 数据模型|Legacy/Enhanced 兼容性、TCP 队头阻塞、播放缓冲|
|广播与既有 TS 系统|TS + 广播传输，或经 SRT 等交付|PSI、PCR、固定包开销及外层恢复机制|
|实时互动|WebRTC：ICE/STUN + DTLS-SRTP + RTP/RTCP|连通性、协商、拥塞控制、丢包恢复与播放截止时间|

QUIC/HTTP/3 是传输及 HTTP 映射，不是 fMP4、FLV 或 TS 的容器替代品。具体方案应结合兼容性、延迟预算、随机访问和运维需求选择。

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

- [RFC 3550：RTP / RTCP 基础格式](https://www.rfc-editor.org/rfc/rfc3550)

- [RFC 8285：RTP Header Extensions](https://www.rfc-editor.org/rfc/rfc8285)

- [RFC 4585：RTCP Feedback：NACK / PLI](https://www.rfc-editor.org/rfc/rfc4585)

- [RFC 5506：Reduced-Size RTCP](https://www.rfc-editor.org/rfc/rfc5506)

- [RFC 8489：STUN](https://www.rfc-editor.org/rfc/rfc8489)

- [RFC 8445：ICE](https://www.rfc-editor.org/rfc/rfc8445)

- [RFC 8834：WebRTC 媒体传输](https://www.rfc-editor.org/rfc/rfc8834)

- [RFC 7742：WebRTC 视频与 H.264 要求](https://www.rfc-editor.org/rfc/rfc7742)

- [RFC 6184：H.264 RTP Payload Format](https://www.rfc-editor.org/rfc/rfc6184)

- [RFC 7798：HEVC RTP Payload Format](https://www.rfc-editor.org/rfc/rfc7798)
