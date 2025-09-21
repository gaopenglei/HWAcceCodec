# NVDEC 视频解码器 API 编程指南

## 概述

从 NVIDIA® Fermi™ 系列开始，NVIDIA 显卡便集成了一款视频解码器引擎（本文档中称为 NVDEC），该引擎可提供完全硬件加速的视频解码能力。NVDEC 支持解码多种格式的码流，包括 AV1、H.264、HEVC（H.265）、VP8、VP9、MPEG-1、MPEG-2、MPEG-4 以及 VC-1。NVDEC 的运行完全独立于计算 / 图形引擎，不会占用其资源。

NVIDIA 提供用于 NVDEC 编程的软件 API 及库。下文提及的 “NVDECODE API” 便是这套软件 API，开发者可通过它调用 NVDEC 的视频解码功能，并实现 NVDEC 与显卡上其他引擎的协同工作。

NVDEC 对压缩视频流进行解码后，会将生成的 YUV 帧复制到显存中。由于帧数据存储在显存内，可通过 CUDA 对视频进行后处理。此外，NVDECODE API 还提供经 CUDA 优化的常用后处理操作实现，包括缩放、裁剪、宽高比转换、去隔行以及向多种主流输出视频格式的色彩空间转换。开发者可选择使用 NVDECODE API 提供的这些 CUDA 优化实现，也可自行对解码后的输出帧实现后处理逻辑。

解码后的视频帧可通过图形协同功能输出到显示器进行视频播放，也可直接传递至专用硬件编码器（NVENC）以实现高性能视频转码，还可用于 GPU 加速推理，或供 CUDA 及基于 CPU 的处理流程进一步使用。

## 支持的编码格式

NVDECODE API 支持的编码格式如下：

* MPEG-1

* MPEG-2

* MPEG-4

* VC-1（简易型 / 主型 / 高级配置文件）

* H.264（AVCHD）：基准型、主型、高级、High10（不含 MBAFF，即宏块自适应帧场编码）、High422（不含 MBAFF）配置文件

* H.265（HEVC）：主型 / 主型 10 / 主型 12 配置文件、Main 4:2:2/4:4:4 10/12 配置文件（不含 YUV400，即 4:0:0 采样格式）

* VP8

* VP9（8 位、10 位、12 位）

* AV1 主型配置文件

* 混合（CUDA + CPU）JPEG

有关不同显卡的视频功能详情，请参考第 2 章。

## 视频解码器功能特性

表 1 列出了各显卡架构对应的硬件视频解码器所支持的编码格式及功能特性。

| 显卡架构                        | MPEG-1 与 MPEG-2 | VC-1 与 MPEG-4                            | H.264/AVCHD                                                                         | H.265/HEVC                                                                           | VP8                 | VP9                                          | AV1                                     |
| --------------------------- | --------------- | ---------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------- | -------------------------------------------- | --------------------------------------- |
| Fermi（GF1xx）                | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 4.1 级）                                       | 不支持                                                                                  | 不支持                 | 不支持                                          | 不支持                                     |
| Kepler（GK1xx）               | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：主型、高级配置文件（最高支持 4.1 级）                                           | 不支持                                                                                  | 不支持                 | 不支持                                          | 不支持                                     |
| 第一代 Maxwell（GM10x）          | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 5.1 级）                                       | 不支持                                                                                  | 不支持                 | 不支持                                          | 不支持                                     |
| 第二代 Maxwell（GM20x，GM206 除外） | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048最大比特率：60 Mbps | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 5.1 级）                                       | 不支持                                                                                  | 最大分辨率：4096×4096     | 不支持                                          | 不支持                                     |
| GM206                       | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 5.1 级）                                       | 最大分辨率：4096×2304配置文件：主型配置文件（最高支持 5.1 级）、主型 10 配置文件                                    | 最大分辨率：4096×4096     | 最大分辨率：4096×2304配置文件：0 型配置文件                  | 不支持                                     |
| GP100                       | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 5.1 级）                                       | 最大分辨率：4096×4096配置文件：主型配置文件（最高支持 5.1 级）、主型 10、主型 12 配置文件                              | 最大分辨率：4096×4096     | 最大分辨率：4096×4096配置文件：0 型配置文件                  | 不支持                                     |
| GP10x/GV100/Turing/GA100    | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 5.1 级）                                       | 最大分辨率：8192×8192配置文件：主型配置文件（最高支持 6.0 级）、主型 10、主型 12 配置文件                              | 最大分辨率：4096×4096\[1] | 最大分辨率：8192×8192\[2]配置文件：0 型配置文件、10 位及 12 位解码 | 不支持                                     |
| Hopper                      | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 5.1 级）                                       | 最大分辨率：8192×8192配置文件：主型、主型 10、主型 12 配置文件（最高支持 6.0 级）、支持 4:4:4 色度格式                    | 最大分辨率：4096×4096     | 最大分辨率：8192×8192配置文件：0 型配置文件、10 位及 12 位解码     | 不支持                                     |
| GA10x/AD10x                 | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：4096×4096配置文件：基准型、主型、高级配置文件（最高支持 5.1 级）                                       | 最大分辨率：8192×8192配置文件：主型、主型 10、主型 12 配置文件（最高支持 6.0 级）、支持 4:4:4 色度格式                    | 最大分辨率：4096×4096     | 最大分辨率：8192×8192配置文件：0 型配置文件、10 位及 12 位解码     | 最大分辨率：8192×8192配置文件：0 型配置文件（最高支持 6.0 级） |
| Blackwell                   | 最大分辨率：4080×4080 | 最大分辨率：2048×1024 及 1024×2048              | 最大分辨率：8192×8192配置文件：基准型、主型、高级配置文件（最高支持 6.2 级）、High10 及 High422（不含 MBAFF，最高支持 6.2 级） | 最大分辨率：8192×8192配置文件：主型、主型 10、主型 12、Main 4:2:2/4:4:4 10/12 配置文件（不含 YUV400，最高支持 6.0 级） | 最大分辨率：4096×4096     | 最大分辨率：8192×8192配置文件：0 型配置文件、10 位及 12 位解码     | 最大分辨率：8192×8192配置文件：0 型配置文件（最高支持 6.0 级） |

\[1] 仅部分 GP10x 显卡、所有 Turing 显卡及 GA100 支持\[2] VP9 10 位及 12 位解码仅部分 GP10x 显卡、所有 Turing 显卡及 GA100 支持

## 视频解码器流水线

解码器流水线包含三个核心组件：解复用器（Demuxer）、视频解析器（Video Parser）和视频解码器（Video Decoder）。这三个组件相互独立，可单独使用。NVDECODE API 提供了用于调用 NVIDIA 视频解析器和 NVIDIA 视频解码器的接口，其中 NVIDIA 视频解析器纯为软件组件，若有需求，开发者也可自行实现解析器以替代它。

**图 1：使用 NVDECODE API 的视频解码器流水线**

（此处对应原文中的 decode-pipeline.png 图片，因格式限制未展示）

使用 NVDECODE API 解码任意视频内容时，大致需遵循以下步骤：

1. 创建 CUDA 上下文。

2. 查询硬件解码器的解码功能特性。

3. 创建解码器实例。

4. 对内容进行解复用（如 .mp4 文件），可借助 FFmpeg 等第三方软件实现。

5. 使用 NVDECODE API 提供的解析器或 FFmpeg 等第三方解析器，对视频码流进行解析。

6. 调用 NVDECODE API 启动解码流程。

7. 获取解码后的 YUV 数据，以备后续处理。

8. 查询解码帧的状态。

9. 根据解码状态，将解码输出用于渲染、推理、后处理等后续操作。

10. 若应用需显示输出：
* 将解码后的 YUV 表面（Surface）转换为 RGBA 格式。

* 将 RGBA 表面映射为 DirectX 或 OpenGL 纹理。

* 将纹理绘制到屏幕上。
1. 解码流程完成后，销毁解码器实例。

2. 销毁 CUDA 上下文。

本文档后续章节将详细解释上述步骤，且视频编解码 SDK 软件包中包含的示例应用程序也对这些步骤进行了演示。

## 使用 NVIDIA 视频解码器（NVDECODE API）

所有 NVDECODE API 均通过两个头文件暴露：`cuviddec.h` 和 `nvcuvid.h`，这两个头文件位于视频编解码 SDK 软件包的 `Interface` 文件夹下。NVIDIA 视频编解码 SDK 中的示例程序会静态加载库（Windows 版 SDK 软件包附带该库）函数，并在源文件中包含 `cuviddec.h` 和 `nvcuvid.h`。其中，Windows 系统的 `nvcuvid.dll` 随 NVIDIA 显示驱动一同提供，Linux 系统的 `libnvcuvid.so` 也包含在 NVIDIA 显示驱动中。

本章后续小节将介绍使用 NVDECODE API 加速解码的具体流程。

### 1. 视频解析器

#### （1）创建解析器

填充 `CUVIDPARSERPARAMS` 结构体后，调用 `cuvidCreateVideoParser()` 即可创建解析器对象。该结构体需包含待解码码流的以下信息：

* `CodecType`：必须从 `enum cudaVideoCodec` 中取值，用于指定内容的编码格式（如 H.264、HEVC、VP9 等）。

* `ulMaxNumDecodeSurfaces`：表示解析器解码图像缓冲区（DPB）中的表面数量。初始化解析器时，该值可能未知，可先设为 1 等临时值以创建解析器对象。应用程序必须向驱动注册 `pfnSequenceCallback` 回调函数 —— 当解析器遇到第一个序列头或序列发生变化时，会调用该回调函数。回调函数会通过 `CUVIDEOFORMAT::min_num_decode_surfaces` 告知解析器 DPB 正确解码所需的最小表面数量。若序列回调函数的返回值大于 1，解析器会用该返回值覆盖 `CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces`（详见下文 `pfnSequenceCallback` 说明）。因此，为实现内存优化分配，建议延迟创建解码器对象，直至获取 `CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces` 的准确值，确保解码器对象创建时分配的缓冲区数量满足 `CUVIDDECODECREATEINFO::ulNumDecodeSurfaces` = `CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces`。

* `ulClockRate`：时间戳单位（单位：Hz），0 表示默认值（10000000 Hz）。

* `ulErrorThreshold`：控制解析器对非合规码流的检查严格程度，取值范围为 0\~100。0 表示严格检查，若发现非合规或错误，解析器会返回错误；100 表示忽略所有非合规码流检查。

* `ulMaxDisplayDelay`：最大显示回调延迟，0 表示无延迟。

* `bAnnexb`：对于 AV1 AnnexB 码流，必须设为 1。

* `pfnSequenceCallback`：应用程序必须注册此函数以处理序列变化。当解析器遇到初始序列头或视频格式变化时，会触发该回调。驱动对回调函数返回值的解读如下：

    *   0：失败
    
    *   1：成功，但驱动不应覆盖 `CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces`
    
    *   大于 1：成功，且驱动应使用该返回值覆盖 `CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces`

* `pfnDecodePicture`：当一帧的码流数据准备就绪时，解析器会触发该回调。对于场图像（Field Picture），每一次显示回调可能对应两次解码调用（因两场均需解码以构成一帧）。回调返回值解读如下：

    *   0：失败
    
    *   大于等于 1：成功

* `pfnDisplayPicture`：当一帧按显示顺序准备就绪时，解析器会触发该回调。回调返回值解读如下：

    *   0：失败
    
    *   大于等于 1：成功

* `pfnGetOperatingPoint`：解析器触发该回调以获取 AV1 可伸缩码流的工作点。若未设置 `pfnGetOperatingPoint`，或返回值为 -1 及无效工作点，解析器会默认选择工作点 0，且 `outputAllLayers` 标志设为 0。回调返回值解读如下：

    *   小于 0：失败
    
    *   大于等于 0：成功（第 0~9 位：当前工作点；第 10 位：`bOutputAllLayer` 标志）

* `pfnGetSEIMsg`：当解析器解析完一帧中所有未注册的用户补充增强信息（SEI）或元数据 OBU（按解码顺序）时，会触发该回调。目前，该回调支持 H.264、HEVC 及 AV1 编码格式。回调返回值解读如下：

    *   0：失败
    
    *   大于等于 1：成功

#### （2）解析数据包

从解复用器提取的码流需连同其长度、时间戳、标志等辅助信息一同封装到 `CUVIDSOURCEDATAPACKET` 结构体（简称 “数据包”）中，再通过 `cuvidParseVideoData()` 将该数据包传入解析器。数据包的初始化方式如下：

* `flags`：由应用程序设置，解析器对各标志的解读如下：

    *   `CUVID_PKT_ENDOFSTREAM`：必须与该码流的最后一个数据包一同设置，解析器会触发显示回调以处理显示队列中所有未处理的缓冲区。
    
    *   `CUVID_PKT_TIMESTAMP`：表示数据包中的时间戳有效。
    
    *   `CUVID_PKT_DISCONTINUITY`：若存在码流不连续（如跳转后的数据包），需设置该标志。
    
    *   `CUVID_PKT_ENDOFPICTURE`：当数据包恰好包含一帧或一个场的数据时，必须设置该标志。基于 NALU（网络抽象层单元）的编码格式，其解码回调存在一帧延迟（因解析器需接收下一帧的部分非 VCL NALU 才能检测当前帧边界）。设置该标志可强制解析器跳过边界检查，立即触发解码回调。若数据包数据不完整，解码回调会接收到部分帧数据；若数据包包含多帧数据，解析器仅对第一帧数据触发解码回调，其余 NALU 会被丢弃。
    
    *   `CUVID_PKT_NOTIFY_EOS`：若与 `CUVID_PKT_ENDOFSTREAM` 同时设置，解析器会额外触发一次显示回调，且 `CUVIDPARSERDISPINFO` 取值为 NULL，以此表示码流结束。

* `payload_size`：表示有效载荷（payload）的字节数。

* `payload`：指向码流内存缓冲区的指针。

* `timestamp`：显示时间戳（时钟频率为 10 MHz），仅当 `CUVID_PKT_TIMESTAMP` 标志设置时有效。

在 `cuvidParseVideoData()` 函数内部，只要满足相应条件（如序列参数变化时触发 `pfnSequenceCallback`、帧准备好解码时触发 `pfnDecodePicture`），解析器就会同步触发创建解析器对象时注册的回调函数。若回调函数返回失败，`cuvidParseVideoData()` 会将该失败状态传递给应用程序。

解码结果会与 `CUVIDPICPARAMS` 结构体中的图像索引（picture-index）关联，该结构体也由解析器提供。后续可通过此图像索引将解码帧映射到 CUDA 内存。

#### （3）销毁解析器

用户需调用 `cuvidDestroyVideoParser()` 销毁解析器对象，并释放所有已分配的资源。

### 2. 视频解码器

#### （1）查询解码功能特性

`cuvidGetDecoderCaps()` API 可用于查询底层硬件视频解码器的功能特性。

如表格 1 所示，不同显卡的硬件解码器功能存在差异。因此，为确保应用程序能在各代显卡硬件上正常运行，强烈建议应用程序先查询硬件功能特性，再根据目标功能的支持情况做出相应处理。

调用 `cuvidGetDecoderCaps()` 前，调用线程需关联有效的 CUDA 上下文。用户需先填充 `CUVIDDECODECAPS` 结构体的以下字段：

* `eCodecType`：编码格式类型（如 AV1、H.264、HEVC、VP9、JPEG 等）。

* `eChromaFormat`：色度格式（取自 `enum cudaVideoChromaFormat`，包括单色、4:2:0、4:2:2、4:4:4）。

* `nBitDepthMinus8`：位深减 8 的值（8 位对应 0，10 位对应 2，12 位对应 4）。

调用 `cuvidGetDecoderCaps()` 后，底层驱动会填充 `CUVIDDECODECAPS` 的其余字段，包括目标功能的支持情况、支持的输出格式以及硬件支持的最大 / 最小分辨率。

以下伪代码演示了如何查询 NVDEC 的功能特性：

```c
CUVIDDECODECAPS decodeCaps = {};

// 为 decodeCaps 设置输入参数

decodeCaps.eCodecType = cudaVideoCodec_HEVC; // HEVC 编码格式
decodeCaps.eChromaFormat = cudaVideoChromaFormat_420; // YUV 4:2:0 色度格式
decodeCaps.nBitDepthMinus8 = 2; // 10 位位深
CUresult result = cuvidGetDecoderCaps(&decodeCaps);

// 解读 API 返回的参数，验证内容是否可在底层硬件上解码

// 检查编码格式是否支持
if (!decodeCaps.bIsSupported) {
    NVDEC_THROW_ERROR("该显卡不支持此编码格式", CUDA_ERROR_NOT_SUPPORTED);
}

// 验证内容分辨率是否在硬件支持范围内
if ((coded_width > decodeCaps.nMaxWidth) || (coded_height > decodeCaps.nMaxHeight)) {
    NVDEC_THROW_ERROR("该显卡不支持此分辨率", CUDA_ERROR_NOT_SUPPORTED);
}

// 验证最大支持的宏块数量（编码宽度×编码高度/256）是否不超过 nMaxMBCount
if ((coded_width >> 4) * (coded_height >> 4) > decodeCaps.nMaxMBCount) {
    NVDEC_THROW_ERROR("该显卡不支持此宏块数量", CUDA_ERROR_NOT_SUPPORTED);
}
```

多数情况下，解码器输出的位深和色度采样格式与输入码流（即内容本身）一致。但在某些场景下，可能需要解码器输出与输入码流不同的位深和色度采样格式。此时，建议在创建解码器前，先检查目标输出位深和色度采样格式是否受支持，具体方式如下：

```c
// 检查支持的输出格式
if (decodeCaps.nOutputFormatMask & (1 << cudaVideoSurfaceFormat_NV12)) {
    // 解码器支持 NV12 输出表面格式
}
if (decodeCaps.nOutputFormatMask & (1 << cudaVideoSurfaceFormat_P010)) {
    // 解码器支持 P010 输出表面格式
}
// ...（其他格式检查）
```

`cuvidGetDecoderCaps()` API 还会返回底层显卡的直方图相关功能。NVDEC 在解码过程中会自动收集直方图数据，且不会产生性能损耗。需注意，NVDEC 仅计算解码输出的亮度（luma）分量直方图，不计算后处理帧（如经过缩放、裁剪等操作的帧）的直方图；对于 AV1 编码格式，若启用胶片颗粒（film grain），直方图数据会在应用胶片颗粒前基于解码帧收集。

```c
// 检查是否支持直方图功能
if (decodeCaps.bIsHistogramSupported) {
    int nCounterBitDepth = decodeCaps.nCounterBitDepth; // 直方图计数器位深
    int nMaxHistogramBins = decodeCaps.nMaxHistogramBins; // 直方图最大分箱数
}
// ...（后续处理）
```

直方图数据的计算方式为：`Histogram_Bin[pixel_value >> (pixel_bitDepth - log2(nMaxHistogramBins))]++`。

#### （2）创建解码器

创建解码器实例前，用户需先创建有效的 CUDA 上下文，该上下文将用于整个解码流程。

填充 `CUVIDDECODECREATEINFO` 结构体后，调用 `cuvidCreateDecoder()` 即可创建解码器实例。该结构体需包含待解码码流的以下信息：

* `CodecType`：必须从 `enum cudaVideoCodec` 中取值，表示内容的编码格式（如 H.264、HEVC、VP9 等）。

* `ulWidth, ulHeight`：编码宽度和编码高度（单位：像素）。

* `ulMaxWidth, ulMaxHeight`：解码器支持的最大宽度和最大高度（用于应对分辨率变化场景）。若视频码流分辨率发生变化（新分辨率 ≤ 设定的最大宽高），应用程序可通过 `cuvidReconfigureDecoder()` API 重新配置解码器，无需销毁并重建实例。若 `ulMaxWidth` 或 `ulMaxHeight` 设为 0，则默认与 `ulWidth` 或 `ulHeight` 取值相同。

* `ChromaFormat`：必须从 `enum cudaVideoChromaFormat` 中取值，表示内容的色度格式（如 4:2:0、4:4:4 等）。

* `bitDepthMinus8`：待解码视频码流的位深减 8 的值（8 位对应 0，10 位对应 2，12 位对应 4）。

* `ulNumDecodeSurfaces`：即本文档中提及的 “解码表面” 数量，指驱动在内部分配用于存储解码帧的表面数量。该值越大，流水线效率越高，但会增加 GPU 内存占用。为确保正常运行，其最小值由 `CUVIDEOFORMAT::min_num_decode_surfaces` 定义，可通过 NVIDIA 解析器的第一个序列回调获取。NVDEC 引擎会将解码数据写入这些表面中的一个，但 NVDECODE API 用户无法直接访问这些表面；映射阶段（含解码器输出格式转换、缩放、裁剪等操作）会将这些表面作为输入表面使用。

* `ulNumOutputSurfaces`：表示客户端可通过 `cuvidMapVideoFrame()` 同时映射到解码表面以进行后续处理的最大输出表面数量。这些表面存储经过后处理的解码输出，供客户端使用，驱动会在内部分配相应数量的表面（本文档中称为 “输出表面”）。关于 “映射” 的定义，可参考 “为后续处理准备解码帧” 小节。

* `OutputFormat`：输出表面格式（取自 `enum cudaVideoSurfaceFormat`），该格式必须是 `cuvidGetDecoderCaps()` 中 `decodecaps.nOutputFormatMask` 字段所支持的格式之一。若传入不支持的输出格式，API 会返回 `CUDA_ERROR_NOT_SUPPORTED` 错误。

* `ulTargetWidth, ulTargetHeight`：输出表面的分辨率。若无需缩放，应分别设为 `ulWidth` 和 `ulHeight`。

* `DeinterlaceMode`：对于逐行扫描内容，应设为 `cudaVideoDeinterlaceMode_Weave` 或 `cudaVideoDeinterlaceMode_Bob`；对于隔行扫描内容，应设为 `cudaVideoDeinterlaceMode_Adaptive`。`cudaVideoDeinterlaceMode_Adaptive` 可提供更高画质，但会增加内存占用。

* `ulCreationFlags`：取自 `enum cudaVideoCreateFlags`，可选填。若不明确指定，驱动会自动选择合适的模式。

* `ulIntraDecodeOnly`：若设为 1，需确保待解码内容仅包含 I/IDR 帧（ intra 帧），此设置可帮助驱动优化内存占用。若内容包含非 intra 帧，则不应设置该标志。

* `enableHistogram`：若设为 1，将启用直方图数据收集功能。

调用 `cuvidCreateDecoder()` 后，`CUvideodecoder` 会被填充为解码器句柄，该句柄需在整个解码会话期间保留，并在调用其他 NVDECODE API 时传入。

用户还可在 `CUVIDDECODECREATEINFO` 中指定以下参数，以控制最终输出效果：

* 缩放尺寸

* 裁剪尺寸

* 宽高比转换后的尺寸

以下代码分别演示了在缩放、裁剪、宽高比转换场景下的解码器设置方式：

**示例 1：缩放（源尺寸 1280×960，缩放到 1920×1080）**

```c
CUresult rResult;
unsigned int uScaleWidth = 1920, uScaleHeight = 1080;
// ...（其他初始化操作）

CUVIDDECODECREATEINFO stDecodeCreateInfo;
memset(&stDecodeCreateInfo, 0, sizeof(CUVIDDECODECREATEINFO));
// ...（设置结构体其他成员）
stDecodeCreateInfo.ulTargetWidth = uScaleWidth;
stDecodeCreateInfo.ulTargetHeight = uScaleHeight;

rResult = cuvidCreateDecoder(&hDecoder, &stDecodeCreateInfo);
// ...（后续操作）
```

**示例 2：裁剪（源尺寸 1280×960）**

```c
CUresult rResult;
unsigned int uCropL = 30, uCropR = 700, uCropT = 20, uCropB = 500;
// ...（其他初始化操作）

CUVIDDECODECREATEINFO stDecodeCreateInfo;
memset(&stDecodeCreateInfo, 0, sizeof(CUVIDDECODECREATEINFO));
// ...（设置结构体其他成员）
stDecodeCreateInfo.display_area.left = uCropL;
stDecodeCreateInfo.display_area.right = uCropR;
stDecodeCreateInfo.display_area.top = uCropT;
stDecodeCreateInfo.display_area.bottom = uCropB; // 原文此处疑似笔误，修正为 bottom

rResult = cuvidCreateDecoder(&hDecoder, &stDecodeCreateInfo);
// ...（后续操作）
```

**示例 3：宽高比转换（源尺寸 1280×960，4:3 转 16:9）**

```c
CUresult rResult;
unsigned int uDispAR_L = 0, uDispAR_R = 1280, uDispAR_T = 70, uDispAR_B = 790;
// ...（其他初始化操作）

CUVIDDECODECREATEINFO stDecodeCreateInfo;
memset(&stDecodeCreateInfo, 0, sizeof(CUVIDDECODECREATEINFO));
// ...（设置结构体其他成员）
stDecodeCreateInfo.target_rect.left = uDispAR_L;
stDecodeCreateInfo.target_rect.right = uDispAR_R;
stDecodeCreateInfo.target_rect.top = uDispAR_T;
stDecodeCreateInfo.target_rect.bottom = uDispAR_B;

rResult = cuvidCreateDecoder(&hDecoder, &stDecodeCreateInfo);
// ...（后续操作）
```

#### （3）解码帧 / 场

完成解复用和解析后，客户端可将包含一帧或一个场数据的码流提交至硬件进行解码，具体步骤如下：

1. 填充 `CUVIDPICPARAMS` 结构体：客户端需填入解析过程中获取的参数，且该结构体中针对各支持编码格式的专用子结构体也需一并填充。
2. 调用 `cuvidDecodePicture()`，传入解码器句柄和 `CUVIDPICPARAMS` 指针，触发 NVDEC 开始解码。

#### （4）为后续处理准备解码帧

用户需调用 `cuvidMapVideoFrame()` 以获取输出表面的 CUDA 设备指针和行间距（pitch），该输出表面存储了解码并经过后处理的帧数据。

需注意，`cuvidDecodePicture()` 仅指示 NVDEC 硬件引擎启动帧 / 场的解码，而 `cuvidMapVideoFrame()` 执行成功才表示解码流程完成，且解码得到的 YUV 帧已从 NVDEC 生成的格式转换为 `CUVIDDECODECREATEINFO::OutputFormat` 中指定的 YUV 格式。

`cuvidMapVideoFrame()` API 以解码表面索引（`nPicIdx`）为输入，将其映射到某个可用的输出表面，对解码帧进行后处理并复制到输出表面，最终返回输出表面的 CUDA 设备指针及对应的行间距。本文档中将 `cuvidMapVideoFrame()` 执行的上述操作称为 “映射”。

用户完成对帧的处理后，必须调用 `cuvidUnmapVideoFrame()`，释放输出表面以供存储其他解码并后处理后的帧。若在调用 `cuvidMapVideoFrame()` 后持续不调用对应的 `cuvidUnmapVideoFrame()`，最终 `cuvidMapVideoFrame()` 会执行失败 —— 同一时间最多可映射 `CUVIDDECODECREATEINFO::ulNumOutputSurfaces` 个帧。

`cuvidMapVideoFrame()` 是阻塞调用，会等待解码完成。若在调用 `cuvidDecodePicture()` 的同一 CPU 线程上调用 `cuvidMapVideoFrame()`，会同时阻塞 `cuvidDecodePicture()`，导致应用程序需等待映射完成后才能向 NVDEC 提交新的解码数据包。为避免此问题，可将映射操作放在与调用 `cuvidDecodePicture()` 不同的 CPU 线程（称为 “映射线程”）中执行，而 `cuvidDecodePicture()` 所在线程称为 “解码线程”。

若使用 NVDECODE API 中的 NVIDIA 解析器，应用程序可在解码线程（生产者）与映射线程（消费者）之间实现一个生产者 - 消费者队列，队列中存储待解码帧的图像索引（或其他唯一标识）。解析器可在解码线程上运行，解码线程在显示回调中将图像索引加入队列后立即返回，继续处理后续待解码帧；映射线程则持续监听队列，若队列非空，便取出队列项，传入图像索引调用 `cuvidMapVideoFrame(…)`。需注意，解码线程需确保：直到映射线程消费并释放对应的队列项前，不重复使用该解码图像缓冲区存储新的解码输出。

以下代码演示了 `cuvidMapVideoFrame()` 和 `cuvidUnmapVideoFrame()` 的使用方式：

```c
// MapFrame：调用 cuvidMapVideoFrame 获取设备指针和行间距，通过 CUDA 设备到主机的内存复制，将表面（设备内存）复制到主机内存

bool MapFrame()
{
    CUVIDPARSEDISPINFO stDispInfo;
    CUVIDPROCPARAMS stProcParams;
    CUresult rResult;
    unsigned long long cuDevPtr = 0;&#x20;
    int nPitch, nPicIdx, frameSize;
    unsigned char* pHostPtr = nullptr;

    memset(&stDispInfo, 0, sizeof(CUVIDPARSEDISPINFO));
    memset(&stProcParams, 0, sizeof(CUVIDPROCPARAMS));

 /****************************************************************************************************
*   配置 stProcParams&#x20;
****************************************************************************************************/

// 从帧显示队列中获取帧（该队列在 HandlePictureDisplay 中填充）
if (g_pFrameQueue->dequeue(&stDispInfo))
{
    nPicIdx = stDispInfo.picture_index;
    // 调用 cuvidMapVideoFrame 获取设备指针和行间距
    rResult = cuvidMapVideoFrame(&hDecoder, nPicIdx, &cuDevPtr, &nPitch, &stProcParams);

    // 计算帧大小：4:4:4 格式为 nPitch×3×高度，其他格式为 nPitch×(高度 + (高度 + 1)/2)
    frameSize = (ChromaFormat == cudaVideoChromaFormat_444) ?&#x20;
                (nPitch * 3 * nHeight) :&#x20;
                (nPitch * (nHeight + (nHeight + 1) / 2));

    // 使用 CUDA 设备到主机的内存复制
    rResult = cuMemAllocHost((void**)&pHostPtr, frameSize);
    if (pHostPtr != nullptr)
    {
        rResult = cuMemcpyDtoH(pHostPtr, cuDevPtr, frameSize);
    }

    // 调用 cuvidUnmapVideoFrame 释放输出表面
    rResult = cuvidUnmapVideoFrame(&hDecoder, cuDevPtr);
}

// ...（将 YUV 数据写入文件等后续操作）

// 释放主机内存
if (pHostPtr != nullptr)
{
    cuMemFreeHost(pHostPtr);
}

// ...（其他操作）
return true; // 此处根据实际逻辑返回成功/失败状态
}
```

在多实例解码场景中，NVDEC 可能成为性能瓶颈，此时在不同 CPU 线程上调用 `cuvidMapVideoFrame()` 和 `cuvidDecodePicture()` 难以带来显著性能提升 —— 若驱动内部 NVDEC 的等待队列已满，`cuvidDecodePicture()` 会陷入停滞。为简化实现，视频编解码 SDK 中的示例应用程序将映射和解码调用放在同一 CPU 线程中。

#### （5）获取直方图数据缓冲区

NVDEC 在解码过程中会自动收集直方图数据，且无性能损耗。需注意，NVDEC 仅计算解码输出的亮度分量直方图，不计算后处理帧（如经过缩放、裁剪等操作的帧）的直方图；对于 AV1 编码格式，若启用胶片颗粒（film grain），直方图数据会在应用胶片颗粒前基于解码帧收集。

若在调用 `cuvidCreateDecoder()` 创建解码器时，将 `CUVIDDECODECREATEINFO::enableHistogram` 标志设为 1，则 `cuvidMapVideoFrame()` API 会在返回输出表面的同时，返回直方图数据缓冲区的 CUDA 设备指针。可通过 `CUVIDPROCPARAMS::histogram_dptr` 获取该指针。

驱动会将直方图缓冲区与输出缓冲区关联，因此调用 `cuvidUnmapVideoFrame()` 时，会同时解除直方图缓冲区与输出表面的映射。

以下代码演示了如何通过 `cuvidMapVideoFrame()` 和 `cuvidUnmapVideoFrame()` 访问直方图缓冲区：

```c
// MapFrame：调用 cuvidMapVideoFrame 获取输出帧及对应的直方图缓冲区 CUDA 设备指针
void MapFrameWithHistogram(int nPicIdx, const CUVIDDECODECAPS& decodeCaps)
{
    CUVIDPROCPARAMS stProcParams;
    CUresult rResult;
    unsigned long long cuOutputFramePtr = 0, cuHistogramPtr = 0;&#x20;
    int nPitch;
    // 计算直方图缓冲区大小：（计数器位深 / 8）× 最大分箱数
    int histogramSize = (decodeCaps.nCounterBitDepth / 8) * decodeCaps.nMaxHistogramBins;
    unsigned char* pHistogramPtr = nullptr;

    memset(&stProcParams, 0, sizeof(CUVIDPROCPARAMS));

    /****************************************************************************************************
    *   配置 stProcParams&#x20;
    *****************************************************************************************************/
    stProcParams.histogram_dptr = &cuHistogramPtr; // 绑定直方图缓冲区指针

    // 调用 cuvidMapVideoFrame 获取输出帧和直方图缓冲区
    rResult = cuvidMapVideoFrame(&hDecoder, nPicIdx, &cuOutputFramePtr, &nPitch, &stProcParams);
    // 为直方图数据分配主机内存
    rResult = cuMemAllocHost((void**)&pHistogramPtr, histogramSize);
    if (pHistogramPtr != nullptr)
    {
        // 将直方图数据从设备内存复制到主机内存
        rResult = cuMemcpyDtoH(pHistogramPtr, cuHistogramPtr, histogramSize);
        // ...（直方图数据处理操作）
    }

    // 解除输出帧的映射（同时解除直方图缓冲区映射）
    rResult = cuvidUnmapVideoFrame(&hDecoder, cuOutputFramePtr);

    // 释放直方图主机内存
    if (pHistogramPtr != nullptr)
    {
        cuMemFreeHost(pHistogramPtr);
    }

    // ...（其他操作）
}
```

#### （6）查询解码状态

启动解码后，可随时调用 `cuvidGetDecodeStatus()` 查询帧的解码状态，底层驱动会将解码状态填入 `CUVIDGETDECODESTATUS::*pDecodeStatus`。

目前，NVDECODE API 支持返回以下状态：

1. 解码中

2. 帧解码成功完成

3. 帧码流损坏，但 NVDEC 已修复（ conceal ）

4. 帧码流损坏，且 NVDEC 无法修复

该 API 可帮助客户端根据帧的解码状态做出后续决策（例如，判断是否对该帧执行推理）。需注意，NVDEC 对错误的检测能力因编码格式而异，且该 API 仅在 Maxwell 及后续架构的显卡上支持 HEVC、H.264 和 JPEG 编码格式。

#### （7）重新配置解码器

若码流的分辨率和 / 或后处理参数发生变化，无需销毁当前解码器实例并重建，可通过 `cuvidReconfigureDecoder()` 重新配置解码器，从而节省操作时间（降低延迟）。

在早期版本的 SDK 中，若需调整解码器分辨率或后处理参数（如缩放比例、裁剪尺寸等），需销毁现有解码器实例并创建新实例；而 `cuvidReconfigureDecoder()` 可简化这一流程，适用于码流分辨率频繁变化的场景（例如，服务器端编码器为满足服务质量（QoS）约束而频繁调整图像分辨率）。

使用 `cuvidReconfigureDecoder()` 需遵循以下步骤：

1. 调用 `cuvidCreateDecoder()` 时，需指定 `CUVIDDECODECREATEINFO::ulMaxWidth` 和 `CUVIDDECODECREATEINFO::ulMaxHeight`，且需确保在整个解码流程中，码流分辨率始终不超过这两个值。需注意，同一解码会话中无法修改 `ulMaxWidth` 和 `ulMaxHeight`，若需修改，需销毁并重建解码会话。

2. 解码过程中，若需修改码流或后处理参数，需调用 `cuvidReconfigureDecoder()`，且建议在码流变化时从 `CUVIDPARSERPARAMS::pfnSequenceCallback` 中调用。需将待重新配置的参数填入 `CUVIDRECONFIGUREDECODERINFO` 结构体，且 `CUVIDRECONFIGUREDECODERINFO::ulWidth` 和 `CUVIDRECONFIGUREDECODERINFO::ulHeight` 必须分别小于等于 `CUVIDDECODECREATEINFO::ulMaxWidth` 和 `CUVIDDECODECREATEINFO::ulMaxHeight`，否则 `cuvidReconfigureDecoder()` 会执行失败。

该 API 支持 NVDECODE API 所支持的所有编码格式。

#### （8）销毁解码器

用户需调用 `cuvidDestroyDecoder()` 销毁解码会话，并释放所有已分配的解码器资源。

### 3. NVIDIA 库的运行时动态链接

视频编解码 SDK 示例应用程序主要使用两个 NVIDIA 库：nvcuvid 和 cuda。这两个库既支持加载时动态链接，也支持运行时动态链接，示例应用程序默认使用加载时动态链接。若有需求，用户也可采用运行时动态链接，以下代码片段可帮助理解对应的编程风格调整：

#### （1）运行时动态链接

运行时动态链接指在程序运行时将库加载到内存，以下代码片段演示了在 Windows 和 Linux 系统上动态加载 nvcuvid 库的方式：

```c
#if defined(WIN32) || defined(_WIN32) || defined(WIN64) || defined(_WIN64)
#include <Windows.h>

#ifdef UNICODE
static LPCWSTR __DriverLibName = L"nvcuvid.dll";
#else
static LPCSTR __DriverLibName = "nvcuvid.dll";
#endif

typedef HMODULE DLLDRIVER;

// 加载库的函数
static CUresult LOAD_LIBRARY(DLLDRIVER\* pInstance)
{
*pInstance = LoadLibrary(__DriverLibName);
    if (*pInstance == NULL)
    {
        printf("加载库 \"%s\" 失败！\n", __DriverLibName);
        return CUDA_ERROR_UNKNOWN;
    }
    return CUDA_SUCCESS;
}

#elif defined(__unix__) || defined(__APPLE__) || defined(__MACOSX)
#include <dlfcn.h>

static char __DriverLibName[] = "libnvcuvid.so";

typedef void* DLLDRIVER;

// 加载库的函数
static CUresult LOAD_LIBRARY(DLLDRIVER* pInstance)
{
    *pInstance = dlopen(__DriverLibName, RTLD_NOW);
    if (*pInstance == NULL)
    {
        printf("加载库 "%s" 失败！\n", __DriverLibName);
        return CUDA_ERROR_UNKNOWN;
    }
    return CUDA_SUCCESS;
}
#endif
```

#### （2）获取函数指针

在 Windows 系统中，可通过 `GetProcAddress()` 获取函数指针；在 Linux 系统中，可通过 `dlsym()` 获取，具体代码如下：

```c
// 定义函数指针类型

typedef CUresult CUDAAPI tcuvidCreateVideoParser(CUvideoparser* pObj, CUVIDPARSERPARAMS* pParams);
typedef CUresult CUDAAPI tcuvidParseVideoData(CUvideoparser obj, CUVIDSOURCEDATAPACKET* pPacket);
typedef CUresult CUDAAPI tcuvidDestroyVideoParser(CUvideoparser obj);

typedef CUresult CUDAAPI tcuvidGetDecoderCaps(CUVIDDECODECAPS* pdc);
typedef CUresult CUDAAPI tcuvidCreateDecoder(CUvideodecoder* phDecoder, CUVIDDECODECREATEINFO* pdci);
typedef CUresult CUDAAPI tcuvidDestroyDecoder(CUvideodecoder hDecoder);
typedef CUresult CUDAAPI tcuvidDecodePicture(CUvideodecoder hDecoder, CUVIDPICPARAMS* pPicParams);

// 声明函数指针变量
tcuvidCreateVideoParser* cuvidCreateVideoParser;
tcuvidParseVideoData* cuvidParseVideoData;
tcuvidDestroyVideoParser* cuvidDestroyVideoParser;
tcuvidGetDecoderCaps* cuvidGetDecoderCaps;
tcuvidCreateDecoder* cuvidCreateDecoder;
tcuvidDestroyDecoder* cuvidDestroyDecoder;
tcuvidDecodePicture* cuvidDecodePicture;

// 定义获取函数指针的宏（Windows 系统）
#if defined(WIN32) || defined(_WIN32) || defined(WIN64) || defined(_WIN64)
#define GET_PROC_EX(name, alias, required)                     \
    alias = (t##name*)GetProcAddress(DriverLib, #name);        \
    if (alias == NULL && required) {                           \
        printf("在 %s 中找不到必需函数 "%s"\n",               \
               __DriverLibName, #name);                        \
    return CUDA_ERROR_UNKNOWN;                             \

}

// 定义获取函数指针的宏（Linux/Mac 系统）

#elif defined(__unix__) || defined(__APPLE__) || defined(__MACOSX)
#define GET_PROC_EX(name, alias, required)                     \
    alias = (t##name*)dlsym(DriverLib, #name);                 \
    if (alias == NULL && required) {                           \
        printf("在 %s 中找不到必需函数 "%s"\n",               \
               __DriverLibName, #name);                        \
        return CUDA_ERROR_UNKNOWN;                             \
    }
#endif

// 简化宏定义

#define GET_PROC_REQUIRED(name) GET_PROC_EX(name, name, 1) // 必需函数

#define GET_PROC_OPTIONAL(name) GET_PROC_EX(name, name, 0) // 可选函数

#define GET_PROC(name)          GET_PROC_REQUIRED(name)     // 默认获取必需函数

// 检查调用结果的宏

#define CHECKED_CALL(call)              \
   do {                                \
      CUresult result = (call);       \
       if (CUDA_SUCCESS != result) {   \
           return result;              \
   }                               \
} while (0)

// 初始化 cuvid 库（获取所有必需函数指针）

CUresult CUDAAPI cuvidInit(unsigned int Flags)
{

   DLLDRIVER DriverLib;

   // 加载库
   CHECKED_CALL(LOAD_LIBRARY(&DriverLib));

   // 获取视频解析器相关函数指针
   GET_PROC(cuvidCreateVideoParser);
   GET_PROC(cuvidParseVideoData);
   GET_PROC(cuvidDestroyVideoParser);

   // 获取解码器相关函数指针
   GET_PROC(cuvidGetDecoderCaps);
   GET_PROC(cuvidCreateDecoder);
   GET_PROC(cuvidDestroyDecoder);
   GET_PROC(cuvidDecodePicture);

   // ...（获取其他函数指针）
   return CUDA_SUCCESS;

}
```

## 编写高效的解码应用程序

NVIDIA 显卡上的 NVDEC 引擎是专用硬件模块，可解码支持格式的输入视频码流。典型的视频解码应用程序大致包含以下阶段：

1. 解复用（De-Muxing）

2. 视频码流解析与解码

3. 为后续处理准备帧数据

其中，解复用和解析不属于硬件加速范畴，不在本文档讨论范围内。解复用可借助 FFmpeg 等第三方组件实现（FFmpeg 支持多种复用视频格式），SDK 中的示例应用程序也演示了如何使用 FFmpeg 进行解复用。

同样，解码后处理或视频后处理（如缩放、色彩空间转换、降噪、色彩增强等）可通过用户自定义的 CUDA 核函数高效实现。

若需显示，经后处理的帧可传递至显示引擎输出到屏幕，但该操作不属于 NVDECODE API 的范畴。

### 1. 多线程优化实现

为实现优化，应将解复用、解析、码流解码、处理等阶段分配到独立线程，具体如下：

* **解复用线程**：对媒体文件进行解复用，为解析器提供原始码流。

* **解析与解码线程**：解析码流，并调用 `cuvidDecodePicture()` 启动解码。

* **映射与帧准备线程**：检查是否有已解码帧，若有则调用 `cuvidMapVideoFrame()` 获取帧的 CUDA 设备指针和行间距，供后续处理使用。

NVDEC 驱动内部维护一个包含 4 帧的队列，以实现操作的高效流水线。需注意，该流水线不会引入解码延迟 —— 第一帧加入队列后即开始解码，应用程序可在队列有空间时继续加入输入帧，无需停滞。通常，当应用程序加入 2\~3 帧后，第一帧的解码已完成，流水线可持续运行，确保硬件解码器的利用率最大化。

### 2. 性能与延迟优化建议

* **PCIe 链路宽度**：对于高性能、低延迟的视频编解码应用，需确保 PCIe 链路宽度设为最大可用值。可通过运行 `nvidia-smi -q` 命令查看当前配置的 PCIe 链路宽度，也可在系统 BIOS 设置中调整该参数。

* **解码器重新配置**：若需频繁修改解码分辨率和 / 或后处理参数，建议使用 `cuvidReconfigureDecoder()`，而非销毁并重建解码器实例，以节省操作时间。

### 3. 显存使用优化

#### （1）基础优化步骤

1. 设 `CUVIDDECODECREATEINFO::ulNumDecodeSurfaces = CUVIDEOFORMAT::min_num_decode_surfaces`：确保驱动分配解码序列所需的最小解码表面数量。若解码器性能下降，可适当增大 `ulNumDecodeSurfaces`，建议通过调试选择兼顾解码吞吐量与内存占用的最优值。

2. 合理设置 `CUVIDDECODECREATEINFO::ulNumOutputSurfaces`：需通过实验找到平衡解码吞吐量与内存占用的最优值。

3. 选择合适的 `CUVIDDECODECREATEINFO::DeinterlaceMode`：
* 逐行扫描内容：设为 `cudaVideoDeinterlaceMode::cudaVideoDeinterlaceMode_Weave` 或 `cudaVideoDeinterlaceMode::cudaVideoDeinterlaceMode_Bob`。

* 隔行扫描内容：`cudaVideoDeinterlaceMode::cudaVideoDeinterlaceMode_Adaptive` 可提供更高画质，但会增加内存占用；`Weave` 或 `Bob` 模式内存占用最低，但画质可能下降。

* 若未指定 `DeinterlaceMode`，驱动会默认设为 `Adaptive`，导致内存占用较高。因此，需根据需求明确设置该参数。
1. 共享 CUDA 上下文：多码流解码时，建议尽量减少 CUDA 上下文的创建数量，并在多个会话间共享上下文，以降低上下文创建带来的内存开销。

2. 启用 `ulIntraDecodeOnly` 标志：若明确待解码序列仅包含 Intra 帧，可将 `CUVIDDECODECREATEINFO::ulIntraDecodeOnly` 设为 1（仅支持 HEVC、H.264、VP9）。需注意，若支持格式的常规码流包含 P/B 帧，启用该标志会导致解码失败。

需注意，视频编解码 SDK 中的示例应用程序仅用于演示各 API 功能，可能未完全优化。因此，开发者需自行设计应用程序架构，优化解码 - 后处理 - 显示流水线的各阶段，以实现目标性能与内存占用。

#### （2）动态分配解码表面

码流序列（SPS，序列参数集）中定义的 DPB 大小通常大于实际需求，多数情况下无需使用全部表面。因此，可通过以下方法减少解码会话中的解码表面数量，降低每个会话的内存占用，从而支持更多同时进行的解码会话（但单个会话的解码吞吐量可能略有下降，因表面复用频率增加）：

1. 调用 `cuvidCreateVideoParser()` 前，设 `CUVIDPARSERPARAMS::bMemoryOptimize = 1`，确保解析器传递最小的 `CUVIDPICPARAMS::CurrPicIdx`。

2. `pfnSequenceCallback()` 包含 `min_num_decode_surfaces` 信息，从该回调返回 `min_num_decode_surfaces`，将 DPB 大小覆盖为该值。

3. 设 `CUVIDDECODECREATEINFO::ulNumDecodeSurfaces` 小于 `min_num_decode_surfaces`（最小值为 1），调用 `cuvidCreateDecoder()` 创建解码器（表面数量减少）。

4. 在 `pfnDecodeCallback()` 中，检查解析器传递的 `CUVIDPICPARAMS::CurrPicIdx`：
* 若 `CurrPicIdx < ulNumDecodeSurfaces`，直接调用 `cuvidDecodePicture()` 继续解码。

* 若 `CurrPicIdx ≥ ulNumDecodeSurfaces`，调用 `cuvidReconfigureDecoder()` 增大 `CUVIDRECONFIGUREDECODERINFO::ulNumDecodeSurfaces`，再调用 `cuvidCreateDecoder()` 解码该帧。

* 注意：若 `cuvidReconfigureDecoder()` 因 `CUDA_ERROR_OUT_OF_MEMORY` 失败，应用程序可重试；若持续失败，需自行处理该错误。
