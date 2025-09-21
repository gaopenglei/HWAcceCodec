# NVENC 视频编码器 API 编程指南

## 概述

从 NVIDIA Kepler™ 架构开始，NVIDIA® 显卡便集成了基于硬件的 H.264/HEVC/AV1 视频编码器（下文简称 NVENC）。NVENC 硬件以 YUV/RGB 格式为输入，生成符合 H.264/HEVC/AV1 标准的视频码流。开发者可通过 NVIDIA 视频编解码 SDK 中提供的 NVENCODE API，调用 NVENC 的编码功能。

本文档将介绍如何通过 SDK 中暴露的 NVENCODE API 对 NVENC 进行编程。NVENCODE API 可在 Windows（Windows 10 及以上版本）和 Linux 系统中调用编码功能。

本文档默认开发者已掌握 H.264/HEVC/AV1 视频编码格式相关知识，并熟悉 Windows 和 / 或 Linux 开发环境。

NVENCODE API 保证二进制向后兼容性（若兼容性被破坏，会明确标注）。这意味着，使用旧版本 API 编译的应用程序，可在 NVIDIA 未来发布的驱动版本上正常运行。

## 一、简介

### 1.1 基础编码流程

开发者可创建客户端应用程序，调用 Windows 系统的 `nvEncodeAPI.dll` 或 Linux 系统的 `libnvidia-encode.so` 中暴露的 NVENCODE API 函数。这些库会随 NVIDIA 显示驱动一同安装。客户端应用程序可通过动态链接方式调用这些库：Windows 系统使用 `LoadLibrary()`，Linux 系统使用 `dlopen()`。

NVENCODE API 的函数、结构体及其他参数均在 `nvEncodeAPI.h` 中定义，该头文件包含在 SDK 中。

NVENCODE API 是 C 语言接口，采用类似 C++ 接口的设计模式：应用程序先创建 API 实例，再获取函数指针表，进而与编码器交互。若开发者偏好更高级的 API 及即用型代码，SDK 中还包含封装了核心 API 函数的 C++ 示例类。

本文档后续内容将聚焦 `nvEncodeAPI.h` 中暴露的 C 语言 API。NVENCODE API 设计用于接收原始视频帧（YUV 或 RGB 格式），输出 H.264、HEVC 或 AV1 码流。大致编码流程如下：



1.  初始化编码器

2.  配置所需的编码参数

3.  分配输入 / 输出缓冲区

4.  将帧数据复制到输入缓冲区，并从输出缓冲区读取码流（支持同步模式，Windows 和 Linux 均适用；或异步模式，仅 Windows 10 及以上版本适用）

5.  清理操作 —— 释放所有已分配的输入 / 输出缓冲区

6.  关闭编码会话

本文档后续章节将详细解释上述步骤，且视频编解码 SDK 软件包中的示例应用程序也对这些步骤进行了演示。

### 1.2 编码硬件配置

#### 1.2.1 开启编码会话

加载动态链接库（DLL）或共享对象库后，客户端与 API 的首次交互是调用 `NvEncodeAPICreateInstance` 函数。该函数会在传入的输入 / 输出缓冲区中填充接口函数指针，这些指针对应接口实现的功能。

加载 NVENC 接口后，客户端需先调用 `NvEncOpenEncodeSessionEx` 开启编码会话。该函数会返回一个编码会话句柄，后续调用当前会话的所有 API 函数时，均需使用此句柄。

#### 1.2.2 初始化编码设备

NVIDIA 编码器支持以下类型的设备，客户端需按对应方式初始化：



*   **DirectX 9**

    客户端需创建 DirectX 9 设备，并设置行为标志：`D3DCREATE_FPU_PRESERVE`、`D3DCREATE_MULTITHREADED` 和 `D3DCREATE_HARDWARE_VERTEXPROCESSING`。

    将创建的设备的 IUnknown 接口指针（强制转换为 `void *`）传入 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::device`，并将 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::deviceType` 设为 `NV_ENC_DEVICE_TYPE_DIRECTX`。

    仅 Windows 10 及以上版本支持 DirectX 设备。

*   **DirectX 10**

    将创建的设备的 IUnknown 接口指针（强制转换为 `void *`）传入 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::device`，并将 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::deviceType` 设为 `NV_ENC_DEVICE_TYPE_DIRECTX`。

    仅 Windows 10 及以上版本支持 DirectX 设备。

*   **DirectX 11**

    配置方式与 DirectX 10 一致：设备指针传入 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::device`，`deviceType` 设为 `NV_ENC_DEVICE_TYPE_DIRECTX`。

    仅 Windows 10 及以上版本支持 DirectX 设备。

*   **DirectX 12**

    配置方式与上述 DirectX 系列一致，但仅 Windows 10 20H1 及以上版本支持 DirectX 12 设备。

*   **CUDA**

    客户端需创建 CUDA 上下文（floating CUDA context），将 CUDA 上下文句柄传入 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::device`，并将 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::deviceType` 设为 `NV_ENC_DEVICE_TYPE_CUDA`。

    Linux 系统及 Windows 10 及以上版本均支持 CUDA 设备编码。

*   **OpenGL**

    客户端需创建 OpenGL 上下文，并将其与调用 NVENCODE API 的线程 / 进程关联（即设为当前上下文）。需将 `NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::device` 设为 `NULL`，`NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS::deviceType` 设为 `NV_ENC_DEVICE_TYPE_OPENGL`。

    仅 Linux 系统支持 OpenGL 设备编码。

#### 1.2.3 选择编码器编码格式 GUID

客户端需按以下步骤选择编码格式 GUID（代表视频序列的目标编码格式）：



1.  调用 `NvEncGetEncodeGUIDCount`，从 NVIDIA 视频编码器接口获取支持的编码器 GUID 数量。

2.  根据该数量分配足够大的缓冲区，用于存储支持的编码器 GUID。

3.  调用 `NvEncGetEncodeGUIDs`，填充缓冲区中的 GUID 列表。

4.  从列表中选择符合需求的 GUID，作为后续编码会话的 `encodeGUID`。

### 1.3 编码器优化信息与预设配置

NVIDIA 编码器接口提供四种不同的优化信息枚举值（高质量、低延迟、超低延迟、无损），以适配不同的视频编码场景。表 1 列出了适用于部分常见场景的推荐优化信息。

| 应用场景                           | 优化信息参数推荐值                                   |
| ------------------------------ | ------------------------------------------- |
| 高质量、可容忍延迟的转码视频归档OTT（互联网电视）流编码  | 超高画质（Ultra High Quality）/ 高质量（High Quality） |
| 云游戏流媒体视频会议（高带宽通道，可容忍偶尔较大的帧大小）  | 低延迟（Low Latency）+ CBR（恒定比特率）                |
| 云游戏流媒体视频会议（严格带宽受限通道）           | 超低延迟（Ultra-low Latency）+ CBR                |
| 保留原始视频素材用于后续编辑通用无损数据归档（视频或非视频） | 无损（Lossless）                                |

针对每种优化信息，接口提供从 P1（性能最高）到 P7（性能最低）的七种预设，用于控制性能与画质的平衡。使用这些预设会自动为所选优化信息设置所有相关编码参数，是 API 暴露的粗粒度控制方式。若有需求，可对预设中的特定属性 / 参数进行微调（详见后续小节）。

#### 1.3.1 枚举预设 GUID

客户端可按以下步骤，枚举所选 `encodeGUID` 对应的支持预设 GUID：



1.  调用 `NvEncGetEncodePresetCount`，获取支持的编码器 GUID 数量。

2.  根据该数量分配足够大的缓冲区，用于存储支持的预设 GUID。

3.  调用 `NvEncGetEncodePresetGUIDs`，填充缓冲区中的预设 GUID 列表。

#### 1.3.2 选择编码器预设配置

如前所述，客户端可直接使用 `presetGUID` 配置编码会话 —— 硬件编码器会根据优化信息与预设的组合，自动设置适配场景的参数。若有需求，客户端也可微调预设中的编码器配置参数，覆盖预设默认值。这种方式在编程上更便捷：开发者只需修改关注的配置参数，其余参数保持预设定义的配置即可。

获取预设编码配置并（可选）修改部分参数的步骤如下：



1.  按 1.3.1 节所述，枚举支持的预设。

2.  选择需获取编码配置的预设 GUID。

3.  调用 `NvEncGetEncodePresetConfigEx`，传入所选的 `encodeGUID`、`tuningInfo`（优化信息）和 `presetGUID`。

4.  通过 `NV_ENC_PRESET_CONFIG::presetCfg` 获取所需的预设编码器配置。

5.  （可选）使用对应的配置 API，覆盖默认编码器参数。

### 1.4 选择编码器配置文件

客户端可指定编码配置文件，以适配特定编码场景（例如，为 iPhone/iPod 播放编码视频、为蓝光光盘制作编码视频等）。

获取支持的编码器配置文件列表的步骤如下：



1.  调用 `NvEncGetEncodeProfileGUIDCount`，从 NVIDIA 视频编码器接口获取支持的编码器 GUID 数量。

2.  根据该数量分配足够大的缓冲区，用于存储支持的编码配置文件 GUID。

3.  调用 `NvEncGetEncodeProfileGUIDs`，填充缓冲区中的配置文件 GUID 列表。

4.  选择最符合需求的配置文件 GUID。

### 1.5 获取支持的输入格式列表

NVENCODE API 支持多种输入帧格式（如特定格式的 YUV 和 RGB），这些格式在 `NV_ENC_BUFFER_FORMAT` 中枚举定义。

获取支持的输入格式列表的步骤如下：



1.  调用 `NvEncGetInputFormatCount`，获取支持的输入格式数量。

2.  根据该数量分配缓冲区，用于存储支持的输入缓冲区格式（元素类型为 `NV_ENC_BUFFER_FORMAT`）。

3.  调用 `NvEncGetInputFormats`，获取支持的输入缓冲区格式。

4.  从列表中选择一种格式，用于创建输入缓冲区。

### 1.6 查询编码器功能特性

NVIDIA 视频编码器硬件历经多代演进，每代 GPU 均新增了多项功能。为方便应用程序动态识别系统底层硬件编码器的功能，NVENCODE API 提供了专门的查询接口。建议在使用目标编码器功能前，先通过该接口查询功能支持情况，这是良好的编程习惯。

查询编码器功能的步骤如下：



1.  在 `NV_ENC_CAPS_PARAM::capsToQuery` 参数中指定待查询的功能属性（需为 `NV_ENC_CAPS` 枚举值）。

2.  调用 `NvEncGetEncodeCaps`，判断目标属性是否受支持。

3.  参考 API 文档中 `NV_ENC_CAPS` 枚举的定义，解读各功能属性的含义。

## 二、初始化硬件编码器会话

客户端需调用 `NvEncInitializeEncoder` 初始化编码器会话，调用时需传入通过 `NV_ENC_INITIALIZE_PARAMS` 指定的有效编码器配置，以及开启编码会话时返回的编码器句柄。

### 2.1 编码会话属性

#### 2.1.1 配置编码会话属性

编码会话配置分为三部分：



*   **会话参数**

    输入格式、输出分辨率、显示宽高比、帧率、平均比特率等通用参数，在 `NV_ENC_INITIALIZE_PARAMS` 结构体中定义。客户端需将该结构体的实例作为 `NvEncInitializeEncoder` 的输入参数。

    为成功初始化编码会话，客户端必须填充 `NV_ENC_INITIALIZE_PARAMS` 结构体的以下成员：

1.  `NV_ENC_INITIALIZE_PARAMS::encodeGUID`：需按 1.2.3 节所述，选择合适的编码格式 GUID。

2.  `NV_ENC_INITIALIZE_PARAMS::encodeWidth`：指定编码视频的目标宽度。

3.  `NV_ENC_INITIALIZE_PARAMS::encodeHeight`：指定编码视频的目标高度。

4.  `NV_ENC_INITIALIZE_PARAMS::reportSliceOffsets`：可用于启用切片偏移量报告功能。启用该功能需将 `NV_ENC_INITIALIZE_PARAMS::enableEncodeAsync` 设为 0，且在 Kepler 显卡上不支持基于宏块（MB）和字节的切片模式。

*   **高级编码格式级参数**

    码流相关参数（如 GOP 长度、编码器配置文件、码率控制模式等）在 `NV_ENC_CONFIG` 结构体中暴露。客户端可通过 `NV_ENC_INITIALIZE_PARAMS::encodeConfig` 传递这些编码格式级参数（详见下文）。

*   **高级编码格式特定参数**

    H.264、HEVC 和 AV1 的高级特定参数，分别在 `NV_ENC_CONFIG_H264`、`NV_ENC_CONFIG_HEVC` 和 `NV_ENC_CONFIG_AV1` 结构体中定义。

    客户端可通过 `NV_ENC_CONFIG::encodeCodecConfig` 传递这些编码格式特定参数。

#### 2.1.2 最终确定编码配置



*   **基于预设的高级控制**

    这是配置 NVIDIA 视频编码器接口最简单的方式，客户端只需执行最少的设置步骤，适用于无需微调编码格式级参数的场景。

    此时客户端需遵循以下步骤：

1.  按 2.1.1 节所述，指定会话参数。

2.  （可选）按 1.2.3 节所述，枚举并选择最适配当前场景的预设 GUID，通过 `NV_ENC_INITIALIZE_PARAMS::presetGUID` 传入。这有助于 NVIDIA 视频编码器接口根据提供的 `encodeGUID`、优化信息（tuning info）和 `presetGUID`，正确配置编码会话。

3.  将高级编码格式级参数指针 `NV_ENC_INITIALIZE_PARAMS::encodeConfig::encodeCodecConfig` 设为 `NULL`。

*   **覆盖预设参数以实现精细控制**

    客户端可在预设参数的基础上，修改部分编码参数，步骤如下：

1.  按 2.1.1 节所述，指定会话参数。

2.  按 1.2.3 节所述，枚举并选择最适配当前场景的预设 GUID，再按 1.3.2 节所述获取预设编码配置。

3.  （可选）若需明确查询编码器是否支持特定功能或编码配置参数，需执行以下操作：

*   在 `NV_ENC_CAPS_PARAM::capsToQuery` 参数中指定目标功能属性（需为 `NV_ENC_CAPS` 枚举值）。

*   调用 `NvEncGetEncodeCaps`，判断目标属性是否受支持。参考 API 文档中 `NV_ENC_CAPS` 枚举的定义，解读各功能属性的含义。

1.  按 1.3 节所述，选择目标优化信息和预设 GUID，获取对应的预设编码配置。

2.  根据需求，修改预设 `NV_ENC_CONFIG` 中的任意参数，通过 `NV_ENC_INITIALIZE_PARAMS::encodeConfig::encodeCodecConfig` 指针传递微调后的 `NV_ENC_CONFIG` 结构体。

3.  此外，还需通过 `NV_ENC_INITIALIZE_PARAMS::presetGUID` 传递所选的预设 GUID。这是为了让 NVIDIA 视频编码器接口配置编码会话相关的内部参数，确保编码输出符合客户端需求。需注意，此处传递的预设 GUID 不会覆盖已微调的参数。

### 2.2 码率控制

NVENC 支持多种码率控制模式，并通过 `NV_ENC_INITIALIZE_PARAMS::encodeConfig::rcParams` 结构体，提供对码率控制算法相关参数的控制能力。码率控制算法在 NVENC 固件中实现。

NVENC 支持的码率控制模式如下：



*   **恒定比特率（CBR）**

    将 `rateControlMode` 设为 `NV_ENC_PARAMS_RC_CBR`，即可启用 CBR 模式。该模式下，仅需指定 `averageBitRate`，码率控制算法会将其作为目标输出比特率。客户端可通过 `NV_ENC_RC_PARAMS::lowDelayKeyFrameScale` 控制 I 帧与 P 帧的比特率比例，避免 I 帧生成过多比特导致通道拥塞。若需严格遵循目标比特率，需将 `NV_ENC_CONFIG_H264/NV_ENC_CONFIG_HEVC::enableFillerDataInsertion` 设为 1，或为 AV1 格式将 `NV_ENC_CONFIG_AV1::enableBitstreamPadding` 设为 1。

*   **可变比特率（VBR）**

    将 `rateControlMode` 设为 `NV_ENC_PARAMS_RC_VBR`，即可启用 VBR 模式。编码器会尝试在长期内遵循 `averageBitRate` 设定的平均比特率，且编码过程中任何时刻的比特率均不超过 `maxBitRate`。该模式下，必须指定 `averageBitRate`；若未指定 `maxBitRate`，NVENC 会将其设为内部默认值。建议客户端同时指定 `maxBitRate` 和 `averageBitRate`，以获得更精准的控制。

*   **恒定量化参数（Constant QP）**

    将 `rateControlMode` 设为 `NV_ENC_PARAMS_RC_CONSTQP`，即可启用恒定 QP 模式。该模式下，整帧均使用 `NV_ENC_RC_PARAMS::constQP` 中指定的 QP 值进行编码。

*   **目标画质（Target Quality）**

    将 `rateControlMode` 设为 VBR，并在 `targetQuality` 中指定目标画质，即可启用目标画质模式。H.264/HEVC 格式的目标画质取值范围为 0\~51，AV1 格式为 0\~63（Video Codec SDK 8.0 及以上版本支持小数取值）。该模式下，编码器会通过调整比特率（受 `maxBitRate` 限制），为每帧维持恒定画质，因此最终平均比特率可能因视频内容不同而显著变化。若设置 `maxBitRate`，其会成为实际比特率的上限；若 `maxBitRate` 设得过低，可能导致比特率受限，无法达到目标画质。

### 2.3 多 pass 帧编码

在确定编码帧的 QP 值时，若 NVENC 能了解帧的整体复杂度，便可更优地分配可用比特预算。此外，在某些场景下，多 pass 编码还能更好地捕捉帧间的大幅运动。为此，NVENC 支持以下几种多 pass 帧编码模式：



1.  每帧 1 pass 编码（`NV_ENC_MULTI_PASS_DISABLED`）

2.  每帧 2 pass 编码：首 pass 为 1/4 分辨率，次 pass 为全分辨率（`NV_ENC_TWO_PASS_QUARTER_RESOLUTION`）

3.  每帧 2 pass 编码：两次 pass 均为全分辨率（`NV_ENC_TWO_PASS_FULL_RESOLUTION`）

在 1 pass 码率控制模式下，NVENC 会估算宏块所需的 QP 值，并立即对宏块进行编码。在 2 pass 码率控制模式下，首 pass 会估算待编码帧的复杂度，确定帧内比特分配方案；次 pass 会根据首 pass 确定的方案，对帧内宏块进行编码。因此，2 pass 码率控制模式能更优地在帧内分配比特，尤其对 CBR 编码，更易接近目标比特率。但需注意，在其他条件相同的情况下，2 pass 模式的性能低于 1 pass 模式。客户端应用程序需评估各模式的优缺点，选择合适的多 pass 码率控制模式：`NV_ENC_TWO_PASS_FULL_RESOLUTION` 能为次 pass 生成更优的统计信息，而 `NV_ENC_TWO_PASS_QUARTER_RESOLUTION` 能捕捉更大的运动向量，并作为提示传递给次 pass。

### 2.4 设置编码会话属性

完成所有编码器设置后，客户端需填充 `NV_ENC_CONFIG` 结构体，并将其作为 `NvEncInitializeEncoder` 的输入参数，固定当前编码会话的编码设置。部分设置（如码率控制模式、平均比特率、分辨率等）可动态修改。

初始化编码会话时，客户端需明确指定以下内容：



*   **运行模式**

    若需启用异步模式，将 `NV_ENC_INITIALIZE_PARAMS::enableEncodeAsync` 设为 1；若需启用同步模式，设为 0。

    仅 Windows 10 及以上版本支持异步模式编码（详见第 6 章）。

*   **图像类型决策（Picture-Type Decision, PTD）**

    若客户端希望按显示顺序传递输入缓冲区，需将 `enablePTD` 设为 1—— 此时由 NVENCODE API 决定图像类型。

    若客户端希望按编码顺序传递输入缓冲区，需将 `enablePTD` 设为 0，且必须指定以下参数：

1.  `NV_ENC_PIC_PARAMS::pictureType`

2.  `NV_ENC_PIC_PARAMS_H264/NV_ENC_PIC_PARAMS_HEVC/NV_ENC_PIC_PARAMS_AV1::displayPOCSyntax`

3.  `NV_ENC_PIC_PARAMS_H264/NV_ENC_PIC_PARAMS_HEVC/NV_ENC_PIC_PARAMS_AV1::refPicFlag`

4.  `NV_ENC_PIC_PARAMS_AV1::goldenFrameFlag/arfFrameFlag/arf2FrameFlag/bwdFrameFlag/overlayFrameFlag`

## 三、创建存储输入 / 输出数据所需的资源

初始化编码会话后，客户端需分配缓冲区，用于存储输入 / 输出数据。

客户端可通过 NVIDIA 视频编码器接口的 `NvEncCreateInputBuffer` API 分配输入缓冲区。此时，客户端需在关闭编码会话前销毁已分配的输入缓冲区，且需按所选输入缓冲区格式，在输入缓冲区中填充有效的输入数据。

客户端需通过 `NvEncCreateBitstreamBuffer` API 分配缓冲区，用于存储输出编码码流。同样，需在关闭编码会话前销毁这些缓冲区。

此外，若客户端无法或不愿通过 NVIDIA 视频编码器接口分配输入缓冲区，也可使用任何外部分配的 DirectX 资源作为输入缓冲区，但需先执行简单处理，将这些资源映射为 NVIDIA 视频编码器接口可识别的资源句柄（详见 “外部分配的输入缓冲区” 小节）。

若客户端使用 CUDA 设备初始化编码器会话，且希望使用非 NVIDIA 视频编码器接口分配的输入缓冲区，则必须使用 `cuMemAlloc` 系列 API 分配的缓冲区。NVIDIA 视频编码器接口支持 `CUdeviceptr` 和 `CUarray` 输入格式。

若客户端使用 OpenGL 设备类型初始化编码器会话，且希望使用非 NVIDIA 视频编码器接口分配的输入缓冲区，则必须提供之前分配的纹理。客户端可通过 `glGenTextures()` 生成纹理，将其绑定到 `NV_ENC_INPUT_RESOURCE_OPENGL_TEX::GL_TEXTURE_RECTANGLE` 或 `NV_ENC_INPUT_RESOURCE_OPENGL_TEX::GL_TEXTURE_2D` 目标，通过 `glTexImage2D()` 为其分配存储空间，再向其中复制数据。需注意，NVENCODE API 的 OpenGL 接口仅支持 Linux 系统。

若客户端使用 DirectX 12 设备初始化编码器会话，则必须通过 `ID3D12Device::CreateCommittedResource()` API 分配输入和输出缓冲区，且需先执行简单处理，将这些输入 / 输出资源映射为 NVIDIA 视频编码器接口可识别的资源句柄（详见 “DirectX 12 输入 / 输出缓冲区分配” 小节）。

**注意**：客户端应至少分配（1 + NB）个输入和输出缓冲区，其中 NB 为连续 P 帧之间的 B 帧数量。

### 3.1 获取序列参数

配置编码会话后，客户端可随时调用 `NvEncGetSequenceParams` 获取序列参数信息（H.264/HEVC 格式为 SPS，AV1 格式为 Sequence Header OBU）。客户端需自行分配大小为 `MAX_SEQ_HDR_LEN` 的缓冲区，用于存储序列参数信息，并在后续释放该缓冲区。

默认情况下，H.264/HEVC 格式的 SPS/PPS 数据会附加到每个 IDR 帧，AV1 格式的 Sequence Header OBU 会附加到每个关键帧。此外，客户端也可请求编码器按需生成 SPS/PPS 或 Sequence Header OBU 数据：将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_OUTPUT_SPSPPS`，当前输入生成的输出码流中便会包含 H.264/HEVC 格式的 SPS/PPS 或 AV1 格式的 Sequence Header OBU。

客户端可在编码器初始化（`NvEncInitializeEncoder`）完成且会话处于活跃状态后，随时调用 `NvEncGetSequenceParams`。

## 四、编码视频流

配置编码会话并分配输入 / 输出缓冲区后，客户端可开始流式传输输入数据进行编码。调用 NVIDIA 视频编码器接口对输入图像进行编码时，需传入有效的输入缓冲区句柄和有效的码流（输出）缓冲区句柄。

### 4.1 准备编码输入缓冲区

有两种方式可分配输入缓冲区并传递给视频编码器：

#### 4.1.1 通过 NVIDIA 视频编码器接口分配的输入缓冲区

若客户端通过 `NvEncCreateInputBuffer` 分配输入缓冲区，需先填充有效输入数据，再将其作为编码输入。具体步骤为：调用 `NvEncLockInputBuffer` 获取输入缓冲区的 CPU 指针，填充输入数据后，调用 `NvEncUnlockInputBuffer`。仅在解锁输入缓冲区后，才能将其传递给编码器；销毁 / 重新分配输入缓冲区前，也需调用 `NvEncUnlockInputBuffer` 解锁。

#### 4.1.2 外部分配的输入缓冲区

将外部分配的缓冲区传递给编码器，需遵循以下步骤：



1.  填充 `NV_ENC_REGISTER_RESOURCE` 结构体，填入外部分配缓冲区的属性。

2.  调用 `NvEncRegisterResource`，传入上述填充好的 `NV_ENC_REGISTER_RESOURCE`。

3.  `NvEncRegisterResource` 会在 `NV_ENC_REGISTER_RESOURCE::registeredResource` 中返回一个不透明句柄，需保存该句柄。

4.  调用 `NvEncMapInputResource`，传入上述返回的句柄。

5.  映射后的句柄会在 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource` 中可用。

6.  客户端需将该映射句柄（`NV_ENC_MAP_INPUT_RESOURCE::mappedResource`）作为 `NV_ENC_PIC_PARAMS` 中的输入缓冲区句柄。

7.  客户端使用完资源后，必须调用 `NvEncUnmapInputResource`。

8.  销毁已注册资源前，必须调用 `NvEncUnregisterResource`，传入 `NvEncRegisterResource` 返回的句柄。

需注意，处于映射状态的映射资源句柄（`NV_ENC_MAP_INPUT_RESOURCE::mappedResource`），不得用于 NVIDIA 视频编码器接口之外的其他用途，否则可能导致未定义行为。

#### 4.1.3 DirectX 12 输入 / 输出缓冲区分配

输入和输出缓冲区的分配需按以下方式进行：



*   **输入缓冲区**：通过 DirectX 12 的 `ID3D12Device::CreateCommittedResource()` API 创建，需指定 `D3D12_HEAP_PROPERTIES::Type = D3D12_HEAP_TYPE_DEFAULT` 和 `D3D12_RESOURCE_DESC::Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D`。

*   **输出缓冲区**：通过 DirectX 12 的 `ID3D12Device::CreateCommittedResource()` API 创建，需指定 `D3D12_HEAP_PROPERTIES::Type = D3D12_HEAP_TYPE_READBACK` 和 `D3D12_RESOURCE_DESC::Dimension = D3D12_RESOURCE_DIMENSION_BUFFER`。

对于 HEVC、H.264 或 AV1 编码，输出缓冲区的推荐大小为：

`输出缓冲区大小 = 2 × 输入 YUV 缓冲区大小`（单位：字节）。

将这些外部分配的输入 / 输出缓冲区传递给编码器，需遵循以下步骤：



1.  填充 `NV_ENC_REGISTER_RESOURCE` 结构体，填入外部分配缓冲区的属性。

2.  为启用 DirectX 12 中的显式同步，`NvEncRegisterResource` API 接收两个 `NV_ENC_FENCE_POINT_D3D12` 指针类型的对象（fence 点由 `ID3D12Fence` 指针和 `value` 组成）：`NV_ENC_REGISTER_RESOURCE_PARAMS_D3D12::pInputFencePoint` 和 `NV_ENC_REGISTER_RESOURCE_PARAMS_D3D12::pOutputFencePoint`，用于注册输入缓冲区。NVENC 引擎会等待 `pInputFencePoint` 达到后，再处理 `NV_ENC_REGISTER_RESOURCE::resourceToRegister`；处理完资源后，会发出 `pOutputFencePoint` 信号，以便需使用该资源的其他引擎开始处理。

3.  调用 `NvEncRegisterResource`，传入上述填充好的 `NV_ENC_REGISTER_RESOURCE`。

4.  `NvEncRegisterResource` 会在 `NV_ENC_REGISTER_RESOURCE::registeredResource` 中返回一个不透明句柄，需保存该句柄。

5.  客户端需将该已注册句柄（`NV_ENC_REGISTER_RESOURCE::registeredResource`），分别作为 `NV_ENC_INPUT_RESOURCE_D3D12::pInputBuffer` 和 `NV_ENC_INPUT_RESOURCE_D3D12::pOutputBuffer` 中的输入和输出缓冲区句柄。

6.  销毁已注册资源前，必须调用 `NvEncUnregisterResource`，传入 `NvEncRegisterResource` 返回的句柄。

需注意，处于注册状态的已注册资源句柄（`NV_ENC_REGISTER_RESOURCE::registeredResource`），不得用于 NVIDIA 视频编码器接口之外的其他用途，否则可能导致未定义行为。

### 4.2 配置逐帧编码参数

客户端需填充 `NV_ENC_PIC_PARAMS` 结构体，填入应用于当前输入图像的参数。客户端可按以下方式设置逐帧参数：



*   **强制当前帧为 intra 帧（I 帧）**：将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_FORCEINTRA`。

*   **强制当前帧作为参考帧**：将 `NV_ENC_PIC_PARAMS_H264/NV_ENC_PIC_PARAMS_HEVC/NV_ENC_PIC_PARAMS_AV1::refPicFlag` 设为 1。

*   **强制当前帧为 IDR 帧**：将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_FORCEIDR`。

*   **请求生成序列参数**：若需在当前编码帧中包含 SPS/PPS（H.264 和 HEVC）或 Sequence Header OBU（AV1），将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_OUTPUT_SPSPPS`。

### 4.3 提交输入帧进行编码

客户端需调用 `NvEncEncodePicture` 执行编码操作。输入图像数据会从指定的输入缓冲区读取，编码完成后，编码码流会存储在指定的码流（输出）缓冲区中。

时间戳、时长、输入缓冲区指针等与编码格式无关的参数，通过 `NV_ENC_PIC_PARAMS` 结构体传递；与编码格式相关的参数，根据所用编码格式，通过 `NV_ENC_PIC_PARAMS_H264`/`NV_ENC_PIC_PARAMS_HEVC`/`NV_ENC_PIC_PARAMS_AV1` 结构体传递。客户端需通过 `NV_ENC_PIC_PARAMS::codecPicParams` 成员，在 `NV_ENC_PIC_PARAMS` 中指定编码格式特定的结构体。

若客户端使用 DirectX 12 设备初始化编码器会话，需在 `NV_ENC_PIC_PARAMS::inputBuffer` 中传递 `NV_ENC_INPUT_RESOURCE_D3D12` 指针（包含已注册资源句柄和对应的输入 `NV_ENC_FENCE_POINT_D3D12`），供 NVENC 等待以启动编码；同时需在 `NV_ENC_PIC_PARAMS::outputBuffer` 中传递 `NV_ENC_OUTPUT_RESOURCE_D3D12` 指针（包含已注册资源句柄和对应的输出 `NV_ENC_FENCE_POINT_D3D12`）。NVENC 引擎会等待 `NV_ENC_INPUT_RESOURCE_D3D12::inputFencePoint` 达到后，再开始处理输入缓冲区；处理完资源后，会发出 `NV_ENC_OUTPUT_RESOURCE_D3D12::outputFencePoint` 信号，以便需使用这些输入 / 输出资源的其他引擎开始处理。

### 4.4 获取编码输出

输入图像编码完成后，客户端需调用 `NvEncLockBitstream` 获取编码码流的 CPU 指针。客户端可将编码数据本地复制，或传递该 CPU 指针进行后续处理（如写入媒体文件）。

CPU 指针的有效性会持续到客户端调用 `NvEncUnlockBitstream` 为止。客户端处理完输出数据后，应调用 `NvEncUnlockBitstream`。

若客户端使用 DirectX 12 设备初始化编码器会话，在获取输出时，需在 `NV_ENC_LOCK_BITSTREAM::outputBitstream` 中传递与 `NV_ENC_PIC_PARAMS::outputBuffer` 中相同的 `NV_ENC_OUTPUT_RESOURCE_D3D12` 指针。

客户端必须确保：在销毁 / 释放码流缓冲区（如关闭编码会话时），或在将其重新用作后续帧的输出缓冲区前，所有码流缓冲区均已解锁。

## 五、编码结束

### 5.1 通知输入流结束

若需通知输入流结束，客户端必须调用 `NvEncEncodePicture`，并将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_FLAGS_EOS`，同时将 `NV_ENC_PIC_PARAMS` 的其他所有成员设为 0。通知 EOS 时，无需传入输入缓冲区。

EOS 通知会有效清空编码器，在单个编码会话中可多次调用该操作，但必须在关闭编码会话前执行。

### 5.2 释放资源

编码完成后，客户端应销毁所有已分配的资源：



*   若通过 NVIDIA 视频编码器接口分配输入缓冲区，需调用 `NvEncDestroyInputBuffer` 销毁。销毁前，必须先调用 `NvEncUnlockInputBuffer` 解锁输入缓冲区。

*   需调用 `NvEncDestroyBitstreamBuffer` 销毁每个已分配的码流缓冲区。销毁前，必须先调用 `NvEncUnlockBitstream` 解锁码流缓冲区。

### 5.3 关闭编码会话

客户端需调用 `NvEncDestroyEncoder` 关闭编码会话。关闭前，必须确保与该编码会话相关的所有资源（包括输入缓冲区、码流缓冲区、SPS/PPS 缓冲区等）均已销毁，所有已注册事件均已注销，所有已映射的输入缓冲区句柄均已解除映射。

## 六、运行模式

NVIDIA 视频编码器接口支持以下两种运行模式：

### 6.1 异步模式

该模式用于异步处理输出缓冲区。客户端需分配事件对象，并将其与已分配的输出缓冲区关联，再将该事件对象作为 `NvEncEncodePicture` API 的参数传递给 NVIDIA 编码器接口。客户端可在单独的线程中等待该事件，事件触发后，调用 NVIDIA 视频编码器接口复制编码器生成的输出码流。需注意，仅 Windows 10 及以上版本的 WDDM 模式驱动支持异步模式；Linux 系统及 Windows TCC 模式（仅 Tesla 显卡支持 ¹）仅支持同步模式（详见 “同步模式” 小节）。

客户端需将 `NV_ENC_INITIALIZE_PARAMS::enableEncodeAsync` 设为 1，以启用异步模式。创建事件对象（每个已分配的输出码流缓冲区对应一个事件对象）后，需通过 `NvEncRegisterAsyncEvent` 将其注册到 NVIDIA 视频编码器接口。调用 `NvEncEncodePicture` 时，需传入码流缓冲区句柄和对应的事件句柄。当硬件编码器完成当前输入数据的编码后，NVIDIA 视频编码器接口会触发该事件。随后，客户端可将 `NV_ENC_LOCK_BITSTREAM::doNotWait` 标志设为 1，以非阻塞模式调用 `NvEncLockBitstream` 获取输出数据。

销毁事件对象前，客户端需调用 `NvEncUnregisterAsyncEvent` 注销事件句柄。NVIDIA 建议尽可能使用异步模式，而非同步模式。

异步模式的分步控制流程如下：



*   异步模式下，输出样本必须包含 “事件 + 输出缓冲区”，且客户端需采用多线程方式（创建 D3D9 设备时需设置 `MULTITHREADED` 标志）。

*   通过 `NvEncCreateBitstreamBuffer` API 分配输出缓冲区，NVIDIA 视频编码器接口会在 `NV_ENC_CREATE_BITSTREAM_BUFFER::bitstreambuffer` 中返回输出内存的不透明指针。该不透明输出指针需用于 `NvEncEncodePicture` 及 `NvEncLockBitstream`/`NvEncUnlockBitstream` 调用。若需通过 CPU 访问输出内存，客户端必须调用 `NvEncLockBitstream` API。IO 缓冲区数量应至少为 “4 + B 帧数量”。

*   事件为通过 Windows `CreateEvent` API 分配的 Windows 事件句柄，需在编码前通过 `NvEncRegisterAsyncEvent` 注册。每个编码会话仅需注册一次事件。销毁事件句柄前，客户端必须通过 `NvEncUnregisterAsyncEvent` 注销事件。事件句柄数量必须与输出缓冲区数量相同（每个输出缓冲区关联一个事件）。

*   客户端需创建辅助线程，用于等待完成事件并从输出样本中复制码流数据。客户端将有两个线程：主线程（应用程序线程）向 NVIDIA 编码器提交编码任务；辅助线程等待完成事件，并从输出缓冲区复制压缩码流数据。

*   调用 `NvEncEncodePicture` API 时，需在 `NV_ENC_PIC_PARAMS::outputBitstream` 和 `NV_ENC_PIC_PARAMS::completionEvent` 字段中，分别传递输出缓冲区和事件。

*   无论输入缓冲区是否重排序（编码顺序≠显示顺序），辅助线程都需按调用 `NvEncEncodePicture` 的顺序等待事件。若 `enablePTD = 1`，NVIDIA 编码器会自动处理 B 帧的重排序，客户端无需感知。对于 AV1 格式，NVIDIA 编码器还会自动执行帧码流打包：始终将 “前导非显示帧” 的码流与后续第一个显示帧的码流拼接为单个输出缓冲区，因此每个输出缓冲区始终包含一个待显示帧，以及自上一个待显示帧以来所有按编码顺序排列的非显示帧。

*   事件触发后，客户端需在 `NV_ENC_LOCK_BITSTREAM::outputBitstream` 字段中，传递等待事件对应的输出缓冲区，调用 `NvEncLockBitstream`。

*   NVIDIA 编码器接口会在 `NV_ENC_LOCK_BITSTREAM` 中返回 CPU 指针和码流字节大小。

*   复制码流数据后，客户端必须调用 `NvEncUnlockBitstream`，解锁已锁定的输出码流缓冲区。

**注意**：



1.  客户端会按事件和输出缓冲区的排队顺序接收触发信号和输出缓冲区。

2.  `NV_ENC_LOCK_BITSTREAM::pictureType` 会向客户端通知输出图像类型。

3.  一旦 NVIDIA 视频编码器接口触发事件，且客户端已从输出缓冲区复制数据，输入样本和输出样本（输出缓冲区及输出完成事件）即可重新使用。

### 6.2 同步模式

该模式用于同步处理输出缓冲区。客户端通过阻塞调用 NVIDIA 视频编码器接口，从编码器获取输出码流数据。启用同步模式需将 `NV_ENC_INITIALIZE_PARAMS::enableEncodeAsync` 设为 0。调用 `NvEncEncodePicture` 时，无需设置完成事件句柄；调用 `NvEncLockBitstream` 时，需将 `NV_ENC_LOCK_BITSTREAM::doNotWait` 设为 0，使锁定调用阻塞，直至硬件编码器完成输出码流写入。随后，客户端可处理生成的码流数据，并调用 `NvEncUnlockBitstream`。Linux 系统仅支持该模式。

### 6.3 线程模型

为实现最高编码性能，编码器客户端应创建单独的线程，用于等待事件或调用编码器接口的阻塞函数。

客户端应避免从主编码器处理线程调用任何阻塞函数。主编码器线程仅应用于编码器初始化，以及通过 `NvEncEncodePicture` API（非阻塞）向硬件编码器提交任务。

输出缓冲区处理（如异步模式下等待完成事件，或同步模式下调用 `NvEncLockBitstream`/`NvEncUnlockBitstream` 等阻塞 API）应在辅助线程中执行。这样可确保主编码器线程仅在客户端资源耗尽时才会阻塞。

此外，建议分配较多的输入和输出缓冲区，以避免资源冲突，提升编码器整体吞吐量。

在 Windows 系统中，若编码设备类型为 DirectX，从主线程调用 `IDXGIOutputDuplication::AcquireNextFrame` 等 DXGI API，同时从辅助线程调用 `NvEncLockBitstream`/`NvEncUnlockBitstream`，可能导致性能不佳或未定义行为 —— 因为 `NvEncLockBitstream` 可能在内部使用应用程序的 DirectX 设备。

此类应用程序若需优化性能，应使用以下编码器设置：



*   `NV_ENC_INITIALIZE_PARAMS::enableEncodeAsync = 1`

*   `NV_ENC_LOCK_BITSTREAM::doNotWait = 0`

*   `NV_ENC_INITIALIZE_PARAMS::enableOutputInVidmem = 0`

> ¹ 如需查看显卡运行模式，可运行驱动附带的命令行工具 nvidia-smi（Windows 系统为 nvidia-smi.exe）。

## 七、使用 CUDA 的编码器功能

尽管 GPU 上的核心视频编码器硬件与 CUDA 核心或图形引擎完全独立，但以下编码器功能会在内部使用 CUDA 进行硬件加速：

**注意**：启用这些功能对 CUDA 或图形性能的影响极小，提供此列表仅为参考。



*   高质量预设的多 pass 码率控制模式

*   前瞻（Look-ahead）功能

*   所有自适应量化（AQ）模式

*   加权预测（Weighted Prediction）

*   RGB 内容编码

*   时间滤波（Temporal Filtering）

### 7.1 仅运动估计模式（Motion Estimation Only Mode）

NVENC 可作为硬件加速器，执行运动搜索并生成运动向量和模式信息。生成的运动向量或模式决策可用于多种场景，例如运动补偿滤波、支持 NVENC 未完全支持的其他编码格式，或作为自定义编码器的运动向量提示。以下为该功能的使用步骤。

对于计算机视觉、AI 和帧插值等场景，Turing 及后续架构的 GPU 集成了另一款硬件加速器，可计算帧间光流向量 —— 相比运动向量，光流向量的视觉匹配效果更优。

#### 7.1.1 查询仅运动估计模式功能

使用仅运动估计（ME）模式前，客户端需明确查询编码器是否支持该模式，步骤如下：



1.  在 `NV_ENC_CAPS_PARAM::capsToQuery` 参数中，指定待查询的功能属性为 `NV_ENC_CAPS_SUPPORT_MEONLY_MODE`。

2.  调用 `NvEncGetEncoderCaps`，判断目标属性是否受支持。

`NV_ENC_CAPS_SUPPORT_MEONLY_MODE` 用于表示硬件对仅 ME 模式的支持情况：



*   0：不支持仅 ME 模式

*   1：支持仅 ME 模式

**注意**：使用 DirectX 12 设备时，不支持仅 ME 模式。

#### 7.1.2 创建输入 / 输出数据资源

客户端需通过 `NvEncCreateInputBuffer` API 至少分配一个输入图像缓冲区，同时通过该 API 分配一个参考帧缓冲区，并负责填充有效的输入数据。

创建输入资源后，客户端需通过 `NvEncCreateMVBuffer` API 分配输出数据资源（用于存储运动向量）。

#### 7.1.3 配置仅 ME 模式设置

`NV_ENC_CODEC_CONFIG::NV_ENC_CONFIG_H264_MEONLY` 结构体可用于控制 NVENC 硬件返回的运动向量分区类型和模式。具体而言，客户端可通过设置以下标志，禁用帧内模式（intra mode）和 / 或特定运动向量（MV）分区大小：



*   `NV_ENC_CONFIG_H264_MEONLY::disableIntraSearch`

*   `NV_ENC_CONFIG_H264_MEONLY::disablePartition16x16`

*   `NV_ENC_CONFIG_H264_MEONLY::disablePartition8x16`

*   `NV_ENC_CONFIG_H264_MEONLY::disablePartition16x8`

*   `NV_ENC_CONFIG_H264_MEONLY::disablePartition8x8`

API 还暴露了 `NV_ENC_CONFIG::NV_ENC_MV_PRECISION` 参数，用于控制硬件返回的运动向量精度：若为整像素精度，客户端需忽略运动向量的两个最低有效位（LSB）；若为亚像素精度，运动向量的两个最低有效位代表运动向量的小数部分。若需获取每个宏块的运动向量，建议将 `NV_ENC_CONFIG_H264_MEONLY::disableIntraSearch` 设为 1，让 NVENC 自动决定运动向量的最优分区大小。

#### 7.1.4 执行运动估计



1.  创建 `NV_ENC_MEONLY_PARAMS` 实例。

2.  将输入图像缓冲区和参考帧缓冲区的指针，分别传入 `NV_ENC_MEONLY_PARAMS::inputBuffer` 和 `NV_ENC_MEONLY_PARAMS::referenceFrame`。

3.  将 `NvEncCreateMVBuffer` API 在 `NV_ENC_CREATE_MV_BUFFER::mvBuffer` 字段中返回的指针，传入 `NV_ENC_MEONLY_PARAMS::mvBuffer`。

4.  若需启用异步模式，客户端需创建事件，并将其传入 `NV_ENC_MEONLY_PARAMS::completionEvent`—— 运动估计完成后，该事件会被触发。每个输出缓冲区应关联一个唯一的事件指针。

5.  调用 `NvEncRunMotionEstimationOnly`，在硬件编码器上执行运动估计。

6.  异步模式下，客户端需等待运动估计完成信号，再重新使用输出缓冲区或终止应用程序。

7.  客户端必须通过 `NvEncLockBitstream` 锁定 `NV_ENC_CREATE_MV_BUFFER::mvBuffer`，以获取运动向量数据。

8.  最后，需将包含输出运动向量的 `NV_ENC_LOCK_BITSTREAM::bitstreamBufferPtr`，分别强制转换为 H.264/HEVC 对应的 `NV_ENC_H264_MV_DATA*/NV_ENC_HEVC_MV_DATA*` 类型，再通过 `NvEncUnlockBitstream` 解锁 `NV_ENC_CREATE_MV_BUFFER::mvBuffer`。

#### 7.1.5 为立体视觉场景启用运动估计

对于需处理两个视角的立体视觉场景，建议采用以下方法，以提升运动向量的性能和质量：



1.  客户端应创建单个编码会话。

2.  在单独的线程中启动左视角和右视角的处理。

3.  对于左视角和右视角，分别将 `NV_ENC_MEONLY_PARAMS::viewID` 设为 0 和 1。

4.  主线程需等待为左视角和右视角启动的 NVENC 线程完成处理。

#### 7.1.6 释放创建的资源

完成运动估计后，客户端需调用 `NvEncDestroyInputBuffer` 销毁输入图像缓冲区和参考帧缓冲区，调用 `NvEncDestroyMVBuffer` 销毁运动向量数据缓冲区。

## 八、高级功能与设置

### 8.1 查询支持的最高视频编解码 SDK 版本

客户端应用程序可通过 `NvEncodeAPIGetMaxSupportedVersion`，获取底层 NVIDIA 显示驱动支持的最高视频编解码 SDK 版本。

`NvEncodeAPIGetMaxSupportedVersion` 可帮助客户端开发跨版本应用程序，使其能在支持不同视频编解码 SDK 版本的 NVIDIA 显示驱动上运行。

### 8.2 前瞻（Look-ahead）

前瞻功能通过让编码器缓存指定数量的帧、估算帧复杂度并按复杂度比例为帧分配比特，提升视频编码器的码率控制精度，同时还能动态分配 B 帧和 P 帧。

启用该功能需遵循以下步骤：



1.  通过 `NvEncGetEncodeCaps` 查询当前硬件是否支持该功能，需检查 `NV_ENC_CAPS_SUPPORT_LOOKAHEAD`。

2.  初始化时，将 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.enableLookahead` 设为 1，启用前瞻功能。

3.  在 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.lookaheadDepth` 中设置前瞻帧数（最大支持 32 帧）。

4.  默认情况下，前瞻功能会启用帧内帧（intra frame）和 B 帧的自适应插入；若需禁用，可将 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.disableIadapt` 和 / 或 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.disableBadapt` 设为 1。

启用该功能后，帧会在编码器中排队，因此在编码器获取足够输入帧满足前瞻需求前，`NvEncEncodePicture` 会返回 `NV_ENC_ERR_NEED_MORE_INPUT`。需持续输入帧，直至 `NvEncEncodePicture` 返回 `NV_ENC_SUCCESS`。

### 8.3 B 帧作为参考帧（B-Frames As Reference）

将 B 帧作为参考帧可提升编码的主观和客观质量，且无性能损耗。因此，建议启用多 B 帧的用户同时启用该功能。

启用该功能需遵循以下步骤：



1.  通过 `NvEncGetEncodeCaps` API 查询功能支持情况，检查返回值中是否包含 `NV_ENC_CAPS_SUPPORT_BFRAME_REF_MODE`。

2.  初始化编码器时，将 `NV_ENC_CONFIG_H264/NV_ENC_CONFIG_HEVC/NV_ENC_CONFIG_AV1::useBFramesAsRef` 设为 `NV_ENC_BFRAME_REF_MODE_MIDDLE`：

*   H.264 和 HEVC 格式：会将第（N/2）个 B 帧设为参考帧（N 为 B 帧数量）；若 N 为奇数，则选择第（N-1）/2 个 B 帧作为参考帧。

*   AV1 格式：会将每隔一个 B 帧设为 Altref2 参考帧，但最后一个 B 帧（位于 Altref 区间内）除外。

### 8.4 重新配置 API（Reconfigure API）

`NvEncReconfigureEncoder` 允许客户端在不关闭现有编码器会话、不重新创建编码会话的情况下，修改 `NV_ENC_INITIALIZE_PARAMS` 中的编码器初始化参数。这可帮助客户端避免因销毁和重建编码会话带来的延迟，适用于视频会议、游戏流媒体等传输介质易不稳定的场景。

通过该 API，客户端可在同一编码会话中动态修改比特率、帧率、分辨率等参数。重新配置的参数通过 `NV_ENC_RECONFIGURE_PARAMS::reInitEncodeParams` 传递。

但需注意，该 API 目前不支持重新配置所有参数，以下为部分不支持的参数：



*   修改 GOP 结构（`NV_ENC_CONFIG_H264::idrPeriod`、`NV_ENC_CONFIG::gopLength`、`NV_ENC_CONFIG::frameIntervalP`）

*   在同步编码模式和异步编码模式之间切换

*   修改 `NV_ENC_INITIALIZE_PARAMS::maxEncodeWidth` 和 `NV_ENC_INITIALIZE_PARAMS::maxEncodeHeight`

*   修改 `NV_ENC_INITIALIZE_PARAMS::enablePTD` 中的图像类型决策（PTD）设置

*   修改位深

*   修改色度格式

*   修改 `NV_ENC_CONFIG_HEVC::maxCUSize`

*   修改 `NV_ENC_CONFIG::frameFieldMode`

若尝试重新配置不支持的参数，API 会执行失败。

仅当创建编码器会话时设置了 `NV_ENC_INITIALIZE_PARAMS::maxEncodeWidth` 和 `NV_ENC_INITIALIZE_PARAMS::maxEncodeHeight`，才能修改分辨率。

若客户端希望通过该 API 修改分辨率，建议将重新配置后的下一帧强制设为 IDR 帧，需将 `NV_ENC_RECONFIGURE_PARAMS::forceIDR` 设为 1。

若客户端希望重置内部码率控制状态，需将 `NV_ENC_RECONFIGURE_PARAMS::resetEncoder` 设为 1。

### 8.5 自适应量化（AQ）

该功能通过根据序列的空间和时间特性调整编码 QP（在码率控制算法计算的 QP 基础上），提升视觉质量。当前 SDK 支持以下两种自适应量化模式：

#### 8.5.1 空间自适应量化（Spatial AQ）

空间自适应量化模式根据帧的空间特性调整 QP 值。由于低复杂度平坦区域对质量差异的视觉感知比高复杂度细节区域更敏感，该模式会以减少高空间细节区域比特为代价，为平坦区域分配更多比特。尽管空间自适应量化能提升编码视频的视觉感知质量，但在多数情况下，比特的重新分配会导致 PSNR 下降 —— 因此在基于 PSNR 的评估中，应禁用该功能。

应用程序中启用空间自适应量化的步骤如下：



1.  初始化时，将 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.enableAQ` 设为 1，启用空间自适应量化。

2.  通过 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.aqStrength` 控制 QP 调整强度，取值范围为 1（最温和）\~15（最激进）；若不设置，驱动会自动选择强度值。

#### 8.5.2 时间自适应量化（Temporal AQ）

时间自适应量化模式根据序列的时间特性调整编码 QP（在码率控制算法计算的 QP 基础上）。对于帧间恒定或低运动但空间细节丰富的区域，该模式会调整其 QP 值，使其成为更优的未来帧参考帧。为参考帧的此类区域分配更多比特，比为被参考帧的残差分配比特更有效，有助于提升整体编码视频质量。若帧内大部分区域运动较少或无运动，但空间细节丰富（例如高细节静态背景），启用时间自适应量化的收益最大。

时间自适应量化的潜在缺点之一是：启用后，GOP（图像组）内每帧的比特消耗波动可能增大 ——I/P 帧的比特消耗会高于平均 P 帧大小，而 B 帧的比特消耗会更低。尽管在 GOP 级别会维持目标比特率，但 GOP 内帧大小的波动会比禁用该功能时更大。若需 GOP 内每帧严格遵循 CBR 配置文件，不建议启用时间自适应量化。此外，由于部分复杂度估算需通过 CUDA 执行，启用时间自适应量化可能会对性能产生轻微影响。

应用程序中启用时间自适应量化的步骤如下：



1.  调用 `NvEncGetEncodeCaps` API，检查 `NV_ENC_CAPS_SUPPORT_TEMPORAL_AQ`，查询当前硬件是否支持时间自适应量化。

2.  若支持，初始化时将 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.enableTemporalAQ` 设为 1，启用时间自适应量化。

时间自适应量化需使用 CUDA 预处理，因此会消耗一定的 CUDA 计算资源（具体取决于分辨率和内容），且可能导致编码器性能轻微下降。

### 8.6 高位深编码（High Bit Depth Encoding）

所有 NVIDIA GPU 均支持 8 位编码（8 位精度的 RGB/YUV 输入）。从 Pascal 架构开始，NVIDIA GPU 支持 HEVC 高位深编码（10 位精度输入的 HEVC main-10 配置文件）；从 Ada 架构开始，支持 AV1 高位深编码（8 位或 10 位精度输入的 AV1 主配置文件）；从 Blackwell 架构开始，支持 H.264 高位深编码（10 位精度输入的 H.264 high-10 配置文件）。

编码 10 位内容需遵循以下步骤：



1.  通过 `NvEncGetEncodeCaps` 查询功能支持情况，检查 `NV_ENC_CAPS_SUPPORT_10BIT_ENCODE`。

2.  创建编码器会话时，H.264 格式使用 `NV_ENC_H264_PROFILE_HIGH_10_GUID`，HEVC 格式使用 `NV_ENC_HEVC_PROFILE_MAIN10_GUID`，AV1 格式使用 `NV_ENC_AV1_PROFILE_MAIN_GUID`。

3.  初始化编码器时：

*   H.264 格式：若输入内容为 8 位，设 `encodeConfig->encodeCodecConfig.h264Config.outputBitDepth = NV_ENC_BIT_DEPTH_10` 且 `encodeConfig->encodeCodecConfig.h264Config.inputBitDepth = NV_ENC_BIT_DEPTH_8`；若输入内容为 10 位，设 `encodeConfig->encodeCodecConfig.h264Config.inputBitDepth = NV_ENC_BIT_DEPTH_10`。对于 8 位输入内容，NVENC 会在编码前通过硬件将输入从 8 位转换为 10 位。

*   HEVC 格式：若输入内容为 8 位，设 `encodeConfig->encodeCodecConfig.hevcConfig.outputBitDepth = NV_ENC_BIT_DEPTH_10` 且 `encodeConfig->encodeCodecConfig.hevcConfig.inputBitDepth = NV_ENC_BIT_DEPTH_8`；若输入内容为 10 位，设 `encodeConfig->encodeCodecConfig.hevcConfig.inputBitDepth = NV_ENC_BIT_DEPTH_10`。对于 8 位输入内容，Blackwell 架构之前的 GPU 会通过 CUDA 将输入从 8 位转换为 10 位，而 Blackwell 架构 GPU 会通过硬件完成这一转换。

*   AV1 格式：若输入内容为 8 位，设 `encodeConfig->encodeCodecConfig.av1Config.outputBitDepth = NV_ENC_BIT_DEPTH_10` 且 `encodeConfig->encodeCodecConfig.av1Config.inputBitDepth = NV_ENC_BIT_DEPTH_8`；若输入内容为 10 位，设 `encodeConfig->encodeCodecConfig.av1Config.inputBitDepth = NV_ENC_BIT_DEPTH_10`。对于 8 位输入内容，NVENC 会在编码前通过硬件将输入从 8 位转换为 10 位。

1.  预设、码率控制模式等其他编码参数，可根据需求设置。

### 8.7 加权预测（Weighted Prediction）

加权预测通过计算运动补偿预测的乘法加权因子和加法偏移量，为存在光照变化的内容带来显著的质量提升。从 Pascal 架构开始，NVIDIA GPU 的 NVENCODE API 支持 HEVC 和 H.264 格式的加权预测。

启用加权预测需遵循以下步骤：



1.  通过 `NvEncGetEncodeCaps` 查询功能支持情况，检查 `NV_ENC_CAPS_SUPPORT_WEIGHTED_PREDICTION`。

2.  初始化编码器时，将 `NV_ENC_INITIALIZE_PARAMS::enableWeightedPrediction` 设为 1。

**注意**：



*   若编码会话配置了 B 帧，不支持加权预测。

*   使用 DirectX 12 设备时，不支持加权预测。

*   加权预测需使用 CUDA 预处理，因此会消耗一定的 CUDA 计算资源（具体取决于分辨率和内容），且可能导致编码器性能轻微下降。

### 8.8 H.264、HEVC 和 AV1 中的长参考帧（Long-Term Reference, LTR）

NVENCODE API 支持将特定帧标记为长参考帧（LTR），后续可将其作为当前图像编码的参考帧。这有助于错误隐藏：若中间帧数据丢失，客户端解码器可通过长参考帧进行预测。该功能在视频流媒体应用中十分实用，可帮助接收端从帧丢失中恢复。

启用该功能需遵循以下步骤：

1.  初始化编码器时：

*   H.264 格式：将 `NV_ENC_CONFIG_H264::enableLTR` 设为 1。

*   HEVC 格式：将 `NV_ENC_CONFIG_HEVC::enableLTR` 设为 1。

*   AV1 格式：将 `NV_ENC_CONFIG_AV1::enableLTR` 设为 1。

1.  通过 `NvEncGetEncoderCaps` 查询 `NV_ENC_CAPS_NUM_MAX_LTR_FRAMES`，获取当前硬件支持的最大长参考帧数。

2.  正常编码过程中，标记特定帧为长参考帧需遵循以下步骤：

*   配置长参考帧数：


    *   H.264 格式：设置 `NV_ENC_CONFIG_H264::ltrNumFrames`。

    *   HEVC 格式：设置 `NV_ENC_CONFIG_HEVC::ltrNumFrames`。

    *   AV1 格式：设置 `NV_ENC_CONFIG_AV1::ltrNumFrames`。

*   标记帧为长参考帧：H.264 格式设 `NV_ENC_PIC_PARAMS_H264::ltrMarkFrame = 1`，HEVC 格式设 `NV_ENC_PIC_PARAMS_HEVC::ltrMarkFrame = 1`，AV1 格式设 `NV_ENC_PIC_PARAMS_AV1::ltrMarkFrame = 1`。每个长参考帧需分配一个长参考帧索引（取值范围为 0\~`ltrNumFrames - 1`）。

*   分配长参考帧索引：H.264 格式设 `NV_ENC_PIC_PARAMS_H264::ltrMarkFrameIdx`，HEVC 格式设 `NV_ENC_PIC_PARAMS_HEVC::ltrMarkFrameIdx`，AV1 格式设 `NV_ENC_PIC_PARAMS_AV1::ltrMarkFrameIdx`。

1.  使用已标记的长参考帧预测当前帧：

*   H.264 格式通过 `NV_ENC_PIC_PARAMS_H264::ltrUseFrameBitmap`，HEVC 格式通过 `NV_ENC_PIC_PARAMS_HEVC::ltrUseFrameBitmap`，AV1 格式通过 `NV_ENC_PIC_PARAMS_AV1::ltrUseFrameBitmap`，指定用于参考的长参考帧 —— 比特位位置代表用作参考的长参考帧索引。

**注意**：当前 SDK 不支持在配置了 B 帧的编码会话中使用长参考帧。

### 8.9 重点映射（Emphasis MAP）

NVENCODE API 的重点映射功能允许以宏块为粒度，指定帧中需以不同质量编码的区域。编码器会根据每个宏块的实际重点级别，调整用于编码该宏块的量化参数（QP）。调整幅度取决于以下因素：

1.  码率控制算法根据码率控制约束确定的 QP 绝对值 —— 通常，对于给定重点级别，码率控制确定的 QP 越高，（负向）调整幅度越大。

2.  宏块的重点级别值。

**注意**：QP 调整在码率控制算法执行后进行，因此使用该功能可能导致 VBV（视频缓冲验证器）/ 码率违规。

当客户端已知图像复杂度（例如通过 NVFBC 的分类映射功能），且需优先保证高复杂度区域的编码质量（降低 QP）—— 即使可能违反比特率 / VBV 缓冲区大小约束时，重点级别映射功能十分实用。**该功能在启用自适应量化（空间 / 时间 AQ）时不支持**。

启用该功能需遵循以下步骤：

1.  通过 `NvEncGetEncodeCaps` API 查询功能支持情况，检查 `NV_ENC_CAPS_SUPPORT_EMPHASIS_LEVEL_MAP`。

2.  将 `NV_ENC_RC_PARAMS::qpMapMode` 设为 `NV_ENC_QP_MAP_EMPHASIS`。

3.  在 `NV_ENC_PIC_PARAMS::qpDeltaMap`（按光栅扫描顺序存储每个宏块的有符号字节数组）中，填入 `enum NV_ENC_EMPHASIS_MAP_LEVEL` 中的值。

如前所述，`NV_ENC_EMPHASIS_MAP_LEVEL` 的值越高，代表对该宏块 QP 的（负向）调整幅度越大，以提升其编码质量。用户可对需高质量编码的区域设置更高的重点级别。

### 8.10 显存中的 NVENC 输出（NVENC Output in Video Memory）

从 SDK 9.0 开始，NVENCODE API 支持在显存中输出码流和 H.264 仅 ME 模式的结果。这对需通过 CUDA 或 DirectX 着色器处理 NVENC 输出的场景十分有用 —— 将 NVENC 输出保留在显存中，可避免缓冲区不必要的 PCIe 传输。显存需由客户端应用程序分配为一维缓冲区。当前该功能支持 H.264、HEVC 和 AV1 编码，以及 H.264 仅 ME 模式，且支持 DirectX 11 和 CUDA 接口。

使输出存储在显存中的步骤如下：

1.  调用 `nvEncInitializeEncoder()` 时，将 `NV_ENC_INITIALIZE_PARAMS::enableOutputInVidmem` 设为 1。

2.  在显存中分配一维缓冲区，供 NVENC 写入输出。

*   AV1、HEVC 或 H.264 编码：该缓冲区的推荐大小为 `输出缓冲区大小 = 2 × 输入 YUV 缓冲区大小 + sizeof(NV_ENC_ENCODE_OUT_PARAMS)`。输出缓冲区的前 `sizeof(NV_ENC_ENCODE_OUT_PARAMS)` 字节存储 `NV_ENC_ENCODE_OUT_PARAMS` 结构体，后续字节存储编码码流数据。

*   H.264 仅 ME 模式输出：该缓冲区的推荐大小为 `输出缓冲区大小 = HeightInMbs × WidthInMbs × sizeof(NV_ENC_H264_MV_DATA)`，其中 `HeightInMbs` 和 `WidthInMbs` 分别为图像高度和宽度（以 16×16 宏块为单位）。

1.  缓冲区创建方式：

*   DirectX 11 接口：通过 DirectX 11 的 `CreateBuffer()` API 创建，需指定 `usage = D3D11_USAGE_DEFAULT`、`BindFlags = (D3D11_BIND_VIDEO_ENCODER | D3D11_BIND_SHADER_RESOURCE)`、`CPUAccessFlags = 0`。

*   CUDA 接口：通过 `cuMemAlloc()` 创建。

1.  通过 `nvEncRegisterResource()` 注册该缓冲区，需指定：

*   若输出为编码码流，设 `NV_ENC_REGISTER_RESOURCE::bufferUsage = NV_ENC_OUTPUT_BITSTREAM`。

*   若输出为 H.264 仅 ME 模式的运动向量，设 `NV_ENC_REGISTER_RESOURCE::bufferUsage = NV_ENC_OUTPUT_MOTION_VECTOR`。

    同时设 `NV_ENC_REGISTER_RESOURCE::bufferFormat = NV_ENC_BUFFER_FORMAT_U8`。`NvEncRegisterResource()` 会在 `NV_ENC_REGISTER_RESOURCE::registeredResource` 中返回已注册句柄。

1.  将 `NV_ENC_MAP_INPUT_RESOURCE::registeredResource` 设为上一步获取的 `NV_ENC_REGISTER_RESOURCE::registeredResource`。

2.  调用 `nvEncMapInputResource()`，该函数会在 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource` 中返回映射资源句柄。

3.  模式适配：

*   AV1/HEVC/H.264 编码模式：调用 `nvEncEncodePicture()` 时，将 `NV_ENC_PIC_PARAMS::outputBitstream` 设为 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource`。

*   H.264 仅 ME 模式：调用 `nvEncRunMotionEstimationOnly()` 时，将 `NV_ENC_MEONLY_PARAMS::mvBuffer` 设为 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource`。

读取输出缓冲区时需注意：

*   调用 `nvEncEncodePicture()` 或 `nvEncRunMotionEstimationOnly()` 后，客户端需先解除该输出缓冲区的映射，才能对其进行后续处理，且不应调用 `NvEncLockBitstream()`。

*   异步模式下，客户端应用程序需等待事件触发后再读取输出；同步模式下不触发事件，同步由 NVIDIA 驱动内部处理。

访问输出的步骤如下：

1.  调用 `nvEncUnmapInputResource()`，传入 `nvEncMapInputResource()` 返回的映射资源句柄 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource`，解除输入缓冲区映射。之后，即可对输出缓冲区进行后续处理 / 读取。

2.  编码场景：输出缓冲区的前 `sizeof(NV_ENC_ENCODE_OUT_PARAMS)` 字节需解读为 `NV_ENC_ENCODE_OUT_PARAMS` 结构体，后续为编码码流数据。编码码流的大小由 `NV_ENC_ENCODE_OUT_PARAMS::bitstreamSizeInBytes` 给出。

3.  模式适配：

*   CUDA 模式：对该缓冲区的所有 CUDA 操作必须使用默认流。若需在系统内存中获取输出，可通过任何 CUDA API（如 `cuMemcpyDtoH()`），使用默认流读取输出缓冲区。驱动会确保仅在 NVENC 完成输出写入后，才允许读取输出缓冲区。

*   DX11 模式：可通过任何 DirectX 11 API 读取输出。驱动会确保仅在 NVENC 完成输出写入后，才允许读取输出缓冲区。若需在系统内存中获取输出，可通过 DirectX 11 的 `CopyResource()` API，将数据复制到 CPU 可读取的暂存缓冲区（staging buffer），再调用 DirectX 11 的 `Map()` API 读取该暂存缓冲区。

### 8.11 HEVC 中的 Alpha 层编码支持（Alpha Layer Encoding support in HEVC）

NVENCODE API 支持在 HEVC 编码中加入 Alpha 层。该功能允许应用程序编码包含 YUV 数据的基础层，以及包含 Alpha 通道数据的辅助层。

启用该功能需遵循以下步骤：



1.  通过 `nvEncGetEncodeCaps()` 查询功能支持情况，检查 `NV_ENC_CAPS_SUPPORT_ALPHA_LAYER_ENCODING`。需注意，Alpha 层编码仅支持 `NV_ENC_BUFFER_FORMAT_NV12`、`NV_ENC_BUFFER_FORMAT_ARGB` 和 `NV_ENC_BUFFER_FORMAT_ABGR` 三种输入格式。

2.  初始化编码器时，将 `NV_ENC_CONFIG_HEVC::enableAlphaLayerEncoding` 设为 1。客户端还可通过设置 `NV_ENC_RC_PARAMS::alphaLayerBitrateRatio`，指定比特率在 YUV 层和 Alpha 辅助层之间的分配比例。例如，若 `NV_ENC_RC_PARAMS::alphaLayerBitrateRatio = 3`，则 75% 的比特率用于基础层编码，25% 用于 Alpha 层编码。

3.  正常编码过程中，Alpha 层编码需遵循以下步骤：

*   **在 **`nvEncEncodePicture()`** 中传递 Alpha 输入**：


    *   若输入格式为 `NV_ENC_BUFFER_FORMAT_NV12`：YUV 数据需传入 `NV_ENC_PIC_PARAMS::inputBuffer`，Alpha 输入数据需通过 `NV_ENC_PIC_PARAMS::alphaBuffer` 单独传递，且 `NV_ENC_PIC_PARAMS::alphaBuffer` 的格式需设为 `NV_ENC_BUFFER_FORMAT_NV12`—— 亮度（luma）平面存储 Alpha 数据，色度（chroma）分量需填充为 0x80。

    *   若输入格式为 `NV_ENC_BUFFER_FORMAT_ABGR` 或 `NV_ENC_BUFFER_FORMAT_ARGB`：输入数据直接传入 `NV_ENC_PIC_PARAMS::inputBuffer`，`NV_ENC_PIC_PARAMS::alphaBuffer` 需设为 `NULL`。

*   **获取编码输出**：调用 `NvEncLockBitstream` 可同时获取 YUV 层和 Alpha 层的编码输出。`NV_ENC_LOCK_BITSTREAM::bitstreamSizeInBytes` 包含总编码大小（YUV 层码流数据、Alpha 层码流数据及所有头部数据的总和）；客户端可通过 `NV_ENC_LOCK_BITSTREAM::alphaLayerSizeInBytes` 单独获取 Alpha 层的大小。

**注意**：以下场景不支持 Alpha 编码：



*   启用子帧模式（subframe mode）时；

*   输入图像为 YUV 4:2:2 或 4:4:4 格式时；

*   输入图像位深为 10 位时；

*   码流输出指定为显存存储时；

*   启用加权预测时。

### 8.12 H.264、HEVC 和 AV1 中的时间可伸缩视频编码（Temporal Scalable Video Coding, SVC）

NVENCODE API 支持 H.264/AVC 视频压缩标准附录 G 中定义的时间可伸缩视频编码（SVC）。时间 SVC 会生成包含基础层和多个辅助层的层级结构。

使用时间 SVC 需遵循以下步骤：



1.  调用 `NvEncGetEncodeCaps` API，检查 `NV_ENC_CAPS_SUPPORT_TEMPORAL_SVC`，查询当前硬件是否支持时间 SVC。

2.  若支持，通过 `nvEncGetEncodeCaps()` 查询 `NV_ENC_CAPS_NUM_MAX_TEMPORAL_LAYERS` 的值，获取 SVC 支持的最大时间层数。

3.  初始化编码器时：

*   H.264 格式：将 `NV_ENC_CONFIG_H264::enableTemporalSVC` 设为 1；通过 `NV_ENC_CONFIG_H264::numTemporalLayers` 指定时间层数，通过 `NV_ENC_CONFIG_H264::maxTemporalLayers` 指定最大时间层数。

*   HEVC 格式：将 `NV_ENC_CONFIG_HEVC::enableTemporalSVC` 设为 1；通过 `NV_ENC_CONFIG_HEVC::numTemporalLayers` 指定时间层数，通过 `NV_ENC_CONFIG_HEVC::maxTemporalLayersMinus1` 指定最大时间层数（值为 “最大层数 - 1”）。

*   AV1 格式：将 `NV_ENC_CONFIG_AV1::enableTemporalSVC` 设为 1；通过 `NV_ENC_CONFIG_AV1::numTemporalLayers` 指定时间层数，通过 `NV_ENC_CONFIG_AV1::maxTemporalLayersMinus1` 指定最大时间层数（值为 “最大层数 - 1”）。

1.  若最大时间层数大于 2，帧重排序所需的最小 DPB（解码图像缓冲区）大小为 `(maxTemporalLayers - 2) × 2`。因此，需将 `NV_ENC_CONFIG_H264::maxNumRefFrames`（H.264）、`NV_ENC_CONFIG_HEVC::maxNumRefFramesInDPB`（HEVC）或 `NV_ENC_CONFIG_AV1::maxNumRefFramesInDPB`（AV1）设为大于或等于该值。需注意，上述参数的默认值为 `NV_ENC_CAPS::NV_ENC_CAPS_NUM_MAX_TEMPORAL_LAYERS`。

2.  默认情况下，启用时间 SVC 时会添加 SVC 前缀 NALU；若需禁用，将 `NV_ENC_CONFIG_H264::disableSVCPrefixNalu` 设为 0。

3.  NVENCODE API 支持在码流中添加可伸缩信息 SEI 消息：将 `NV_ENC_CONFIG_H264::enableScalabilityInfoSEI` 设为 1 即可启用，该 SEI 消息会附加到编码码流的每个 IDR 帧中。需注意，当前该 SEI 仅支持与时间可伸缩性相关的部分字段。

4.  启用时间 SVC 时，仅基础层帧可标记为长参考帧（LTR）。

5.  当前时间 SVC 不支持 B 帧，启用后 `NV_ENC_CONFIG::frameIntervalP` 会被忽略。

### 8.13 错误恢复功能（Error Resiliency features）

在典型的视频流场景中，客户端解码器出现比特错误较为常见。为减少这些错误的影响并实现恢复，NVENCODE API 提供了以下错误恢复功能：

#### 8.13.1 参考图像失效（Reference Picture Invalidation）

当客户端解码器检测到某帧解码损坏时，NVENCODE API 允许通过 `NvEncInvalidateRefFrames` API 使该帧失效。客户端可请求流媒体服务器上的编码器使该帧失效，避免后续帧将其作为运动估计的参考帧。之后，服务器会根据可用的旧短期和长期参考帧进行参考；若无可用参考帧，则当前帧会编码为 intra 帧。

`NV_ENC_CONFIG_H264::maxNumRefFrames`（H.264）、`NV_ENC_CONFIG_HEVC::maxNumRefFramesInDPB`（HEVC）或 `NV_ENC_CONFIG_AV1::maxNumRefFramesInDPB`（AV1）参数决定了 DPB 中的帧数 —— 将其设为较大值，即使部分帧失效，DPB 中仍能保留旧帧，相比无参考帧时编码为 intra 帧，可获得更优的图像质量。

通过 `NvEncInvalidateRefFrames` API 失效的特定帧，由每帧的唯一标识（时间戳）确定。该时间戳是编码图像时，通过 `NV_ENC_PIC_PARAMS::inputTimeStamp` 字段传递给编码器的值，可为任何单调递增的唯一值（最常用的是图像的显示时间戳）。编码器会以 `inputTimeStamp` 作为唯一标识，将帧存储在 DPB 中；当通过 `NvEncInvalidateRefFrames` API 请求失效时，会根据该标识找到对应的帧并使其失效。

#### 8.13.2 帧内刷新（Intra Refresh）

8.13.1 节所述的参考图像失效功能，依赖带外上行通道报告客户端解码器的码流错误。若无此类上行通道，或码流易频繁出错，可使用帧内刷新作为错误恢复机制。此外，若使用无限 GOP 长度（无 intra 帧传输），帧内刷新也是从传输错误中恢复的实用手段。

NVENCODE API 提供帧内刷新的实现方式：将 `enableIntraRefresh` 设为 1 即可启用；`intraRefreshPeriod` 定义帧内刷新的周期（间隔多少帧后再次刷新）；`intraRefreshCnt` 定义帧内刷新持续的帧数（在多少帧内完成一次全帧刷新）。

帧内刷新会在 `intraRefreshCnt` 连续帧中，逐段将帧的部分区域用 intra 宏块编码；之后，从第一次帧内刷新帧开始，间隔 `intraRefreshPeriod` 帧后重复该周期。需根据传输过程中可能的错误概率，合理设置 `intraRefreshPeriod` 和 `intraRefreshCnt`：例如，对于高误码率网络，可将 `intraRefreshPeriod` 设为 30（对应 30 FPS 视频流，每秒恢复一次）；对于低误码率网络，可设为更大值。`intraRefreshPeriod` 越小，虽然能更快从网络错误中恢复，但由于帧内刷新周期内强制编码的 intra 宏块比例更高，可能导致画质略有下降。

`intraRefreshCnt` 定义一次帧内刷新周期内的刷新帧数：值越小，全帧刷新速度越快（无需逐段缓慢刷新），错误恢复更快，但每帧需编码的 intra 宏块数量更多，画质可能略有下降。

NVENCODE API 帧内刷新的默认行为：H.264/HEVC 格式基于切片（slice），AV1 格式基于切片组（tile）—— 即帧内刷新过程中，每帧会包含多个切片 / 切片组，且其中一个切片 / 切片组仅包含 intra 编码的宏块（MB）/ 编码树单元（CTU）/ 超块（SB）。

对于 AV1 格式，帧内刷新过程中使用的切片组数量，由驱动根据 `intraRefreshCnt` 和 `intraRefreshPeriod` 的值自动确定，应用程序指定的自定义切片组配置在帧内刷新期间会被忽略。

若应用程序未明确指定切片数量，或指定的切片数量少于 3，在帧内刷新过程中，驱动会将每帧的切片数设为 3。

对于 `NV_ENC_CONFIG_H264::sliceMode` 为 0（基于宏块的切片）、2（基于宏块行的切片）和 3（固定切片数）的场景，帧内刷新周期内的切片数会设为 “切片模式计算的切片数” 与 “`intraRefreshCnt`” 两者中的较小值。

对于 `NV_ENC_CONFIG_H264::sliceMode` 为 1（基于字节的切片）的场景，帧内刷新过程中始终使用 3 个切片。

部分场景下，客户端可能希望避免一帧包含多个切片，此时可启用单切片帧内刷新：



1.  调用 `NvEncGetEncodeCaps` API，检查 `NV_ENC_CAPS_SINGLE_SLICE_INTRA_REFRESH`，查询当前驱动是否支持单切片帧内刷新。

2.  若支持，将 `NV_ENC_CONFIG_H264::singleSliceIntraRefresh`（H.264）或 `NV_ENC_CONFIG_HEVC::singleSliceIntraRefresh`（HEVC）设为 1，启用单切片帧内刷新。

若在帧内刷新过程中修改分辨率，当前刷新过程会立即终止，下一次刷新会在 `NV_ENC_CONFIG_H264::intraRefreshPeriod` 帧后开始。

帧内刷新按编码顺序执行，且仅作用于可作为参考的帧。

### 8.14 HEVC 和 AV1 中的多 NVENC 分帧编码（Multi NVENC Split Frame Encoding）

启用分帧编码后，每个输入帧会被划分为水平条带（horizontal strips），由多个独立的 NVENC 同时编码 —— 相比单 NVENC 编码，通常能提升编码速度。

**注意**：



*   该功能虽能提升编码速度，但会导致画质下降；

*   整体编码吞吐量（所有 NVENC 满负荷运行时，单位时间内编码的总帧数）保持不变；

*   仅 HEVC 和 AV1 格式支持该功能。

因此，该功能适用于 “单编码会话需更高编码速度” 的场景 —— 单流的水平条带通过多个 NVENC 同时编码，可实现单 NVENC 无法达到的速度。如前所述，当创建多个编码会话以满负荷利用所有 NVENC 时，该功能不会影响整体吞吐量。若 NVENC 数量少于请求的分帧模式对应的条带数，水平条带数会强制设为 NVENC 数量。

分帧编码支持以下两种启用模式：

#### 8.14.1 隐式模式（Implicit Mode）

满足以下条件时，会自动触发该模式：



*   GPU 上的 NVENC 数量 ≥ 2；

*   帧高度：HEVC 格式需 ≥ 2112 像素，AV1 格式需 ≥ 2048 像素；

*   预设与优化信息配置：表 2 列出了支持分帧编码的预设与优化信息组合。



| 优化信息（Tuning Info）       | 预设（Preset） |
| ----------------------- | ---------- |
|                         | P1         |
| 高质量（High Quality）       | 支持（Yes）    |
| 低延迟（Low Latency）        | 支持（Yes）    |
| 超低延迟（Ultra Low Latency） | 支持（Yes）    |

#### 8.14.2 显式模式（Explicit Mode）

HEVC 和 AV1 格式支持以下分帧编码模式：



*   `NV_ENC_SPLIT_ENCODE_MODE::NV_ENC_SPLIT_AUTO_MODE`（默认）：即上述隐式模式；

*   `NV_ENC_SPLIT_ENCODE_MODE::NV_ENC_SPLIT_AUTO_FORCED_MODE`：所有配置均启用分帧编码，驱动会自动选择水平条带数，以平衡性能和视觉质量；

*   `NV_ENC_SPLIT_ENCODE_MODE::NV_ENC_SPLIT_TWO_FORCED_MODE`：所有配置均启用分帧编码，当 NVENC 数量 > 1 时，水平条带数强制设为 2；

*   `NV_ENC_SPLIT_ENCODE_MODE::NV_ENC_SPLIT_THREE_FORCED_MODE`：所有配置均启用分帧编码，当 NVENC 数量 > 2 时，水平条带数强制设为 3，否则设为 NVENC 数量；

*   `NV_ENC_SPLIT_ENCODE_MODE::NV_ENC_SPLIT_FOUR_FORCED_MODE`：所有配置均启用分帧编码，当 NVENC 数量 > 3 时，水平条带数强制设为 4，否则设为 NVENC 数量；

*   `NV_ENC_SPLIT_ENCODE_MODE::NV_ENC_SPLIT_DISABLE`：所有配置均禁用分帧编码。

**注意**：部分编码功能与分帧编码不兼容 —— 启用以下任一功能时，分帧编码会自动禁用：



*   HEVC 格式的加权预测（Weighted Prediction）；

*   HEVC 格式的 Alpha 层编码（Alpha Layer Encoding）；

*   HEVC 格式的码流子帧回读模式（Bitstream Subframe Readback Mode）；

*   HEVC/AV1 格式的码流显存输出（Bitstream Output in Video Memory）。

### 8.15 NVENC 重建帧输出（NVENC Reconstructed Frame Output）

从 SDK 12.1 开始，NVENCODE API 支持 Turing 及后续架构 GPU 的 H.264、HEVC 和 AV1 编码重建帧输出。该功能对需通过重建帧评估编码质量的场景十分有用 —— 无需解码码流即可获取重建帧，大幅提升性能。重建帧缓冲区需由客户端应用程序分配为二维缓冲区。

可通过 `NvEncGetEncodeCaps()` 检查 `NV_ENC_CAPS_OUTPUT_RECON_SURFACE`，查询当前硬件是否支持该功能。支持的缓冲区格式包括：`NV_ENC_BUFFER_FORMAT_NV12` 和 `NV_ENC_BUFFER_FORMAT_YUV420_10BIT`；CUDA 接口额外支持 `NV_ENC_BUFFER_FORMAT_NV16`、`NV_ENC_BUFFER_FORMAT_P210`、`NV_ENC_BUFFER_FORMAT_YUV444` 和 `NV_ENC_BUFFER_FORMAT_YUV444_10BIT`。

获取重建帧输出的步骤如下：



1.  调用 `nvEncInitializeEncoder()` 时，将 `NV_ENC_INITIALIZE_PARAMS::enableReconFrameOutput` 设为 1。

2.  分配二维缓冲区，供 NVENC 写入重建帧输出。

*   CUDA 接口：通过 `cuMemAllocPitch()` 创建；

*   DirectX 9 接口：通过 `CreateOffscreenPlainSurface()` 或 `CreateSurface()` API 创建；

*   DirectX 11 接口：通过 DirectX 11 的 `CreateTexture2D()` API 创建，需指定 `usage = D3D11_USAGE_DEFAULT`、`BindFlags = (D3D11_BIND_SHADER_RESOURCE)`、`CPUAccessFlags = 0`。

1.  通过 `nvEncRegisterResource()` 注册该缓冲区，需指定 `NV_ENC_REGISTER_RESOURCE::bufferUsage = NV_ENC_OUTPUT_RECON`，并将 `NV_ENC_REGISTER_RESOURCE::bufferFormat` 设为目标格式。`NvEncRegisterResource()` 会在 `NV_ENC_REGISTER_RESOURCE::registeredResource` 中返回已注册句柄。

2.  将 `NV_ENC_MAP_INPUT_RESOURCE::registeredResource` 设为上一步获取的 `NV_ENC_REGISTER_RESOURCE::registeredResource`。

3.  调用 `nvEncMapInputResource()`，该函数会在 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource` 中返回映射资源句柄。

4.  调用 `nvEncEncodePicture()` 时，将 `NV_ENC_PIC_PARAMS::outputReconBuffer` 设为 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource`，并将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_OUTPUT_RECON_FRAME`。

读取重建帧输出时需注意：



*   调用 `nvEncEncodePicture()` 后，客户端需先解除该输出缓冲区的映射，才能对其进行后续处理。

*   异步模式下，客户端应用程序需等待事件触发后再读取输出；同步模式下不触发事件，同步由 NVIDIA 驱动内部处理。

访问重建帧输出的步骤如下：



1.  调用 `nvEncUnmapInputResource()`，传入 `nvEncMapInputResource()` 返回的映射资源句柄 `NV_ENC_MAP_INPUT_RESOURCE::mappedResource`，解除重建帧缓冲区映射。之后，即可对输出重建缓冲区进行后续处理 / 读取。

2.  模式适配：

*   CUDA 模式：若需在系统内存中获取输出，可通过任何 CUDA API（如 `cuMemcpyDtoH()`）读取重建输出缓冲区。驱动会确保仅在 NVENC 完成输出写入后，才允许读取输出缓冲区。

*   DX11 模式：可通过任何 DirectX 11 API 读取输出。驱动会确保仅在 NVENC 完成输出写入后，才允许读取输出缓冲区。若需在系统内存中获取输出，可通过 DirectX 11 的 `CopyResource()` API，将数据复制到 CPU 可读取的暂存缓冲区，再调用 DirectX 11 的 `Map()` API 读取该暂存缓冲区。

### 8.16 编码帧统计信息（Encoded Frame Stats）

从 SDK 12.1 开始，NVENCODE API 支持 Turing 及后续架构 GPU 的 H.264、HEVC 和 AV1 编码帧统计信息输出。可通过 `NvEncGetEncodeCaps()` 检查 `NV_ENC_CAPS_OUTPUT_ROW_STATS` 或 `NV_ENC_CAPS_OUTPUT_BLOCK_STATS`，查询当前硬件是否支持该功能。该功能对需获取行级或块级编码帧统计信息（QP 和比特数）的场景十分有用。

获取编码帧输出统计信息的步骤如下：



1.  调用 `nvEncInitializeEncoder()` 时，将 `NV_ENC_INITIALIZE_PARAMS::enableOutputStats` 设为 1，并将 `NV_ENC_INITIALIZE_PARAMS::outputStatsLevel` 设为 `NV_ENC_OUTPUT_STATS_ROW_LEVEL`（行级统计）或 `NV_ENC_OUTPUT_STATS_BLOCK_LEVEL`（块级统计）。其中，Turing 和 Ampere GPU 支持 `NV_ENC_OUTPUT_STATS_ROW_LEVEL`，Ada 及后续架构支持 `NV_ENC_OUTPUT_STATS_BLOCK_LEVEL`。

2.  设置统计信息缓冲区参数：

*   **行级统计**：


    *   `NV_ENC_LOCK_BITSTREAM::outputStatsPtrSize = sizeof(NV_ENC_OUTPUT_STATS_ROW) × 行数`；

    *   H.264 格式行数 = `(图像高度 + 15) >> 4`；

    *   HEVC 格式行数 = `(图像高度 + 31) >> 5`。

*   **块级统计**：


    *   `NV_ENC_LOCK_BITSTREAM::outputStatsPtrSize = sizeof(NV_ENC_OUTPUT_STATS_BLOCK) × 块数`；

    *   H.264 格式块数 = `((图像宽度 + 15) >> 4) × ((图像高度 + 15) >> 4)`；

    *   HEVC 格式块数 = `((图像宽度 + 31) >> 5) × ((图像高度 + 31) >> 5)`；

    *   AV1 格式块数 = `((图像宽度 + 63) >> 6) × ((图像高度 + 63) >> 6)`。

1.  分配大小为 `NV_ENC_LOCK_BITSTREAM::outputStatsPtrSize` 的系统内存缓冲区，将其赋值给 `NV_ENC_LOCK_BITSTREAM::outputStatsPtr`，然后调用 `NvEncLockBitstream()` API。

2.  从 `NV_ENC_LOCK_BITSTREAM::outputStatsPtr` 中，以 `NV_ENC_OUTPUT_STATS_BLOCK` 或 `NV_ENC_OUTPUT_STATS_ROW` 格式读取编码帧统计信息。

3.  调用 `NvEncUnlockBitstream()` API。

### 8.17 迭代编码（Iterative encoding）

从 SDK 12.1 开始，NVENCODE API 支持 Turing 及后续架构 GPU 的 H.264、HEVC 和 AV1 编码器迭代编码。可通过 `NvEncGetEncodeCaps()` 检查 `NV_ENC_CAPS_DISABLE_ENC_STATE_ADVANCE`，查询当前硬件是否支持该功能。使用该功能可对同一帧进行多次编码（例如每次使用不同的 QP 或 delta-QP），NVENC 会将每次迭代对应的新状态存储在内部状态缓冲区中；之后，可通过 `NvEncRestoreEncoderState()` API，将 NVENC 状态恢复到任意一次迭代的状态。

#### 8.17.1 启用迭代编码的步骤



1.  调用 `NvEncInitializeEncoder()` API 时，将 `NV_ENC_INITIALIZE_PARAMS::numStateBuffers` 设为目标值。H.264 和 HEVC 格式最大支持分配 16 个状态缓冲区，AV1 格式最大支持 32 个。

2.  每次帧迭代前，应用程序可调用 `NvEncReconfigureEncoder()` API 设置目标编码参数（例如不同的 QP 值），也可通过 `NV_ENC_PIC_PARAMS::qpDeltaMap` 数组设置当前帧迭代的目标值。

#### 8.17.2 应用程序决定图像类型（PTD）时的迭代编码

对同一帧进行多次编码需遵循以下步骤：



1.  将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE`。

2.  将 `NV_ENC_PIC_PARAMS::frameIdx` 设为有效值，且所有帧迭代的 `frameIdx` 必须相同。

3.  将 `NV_ENC_PIC_PARAMS::stateBufferIdx` 设为目标索引，用于将编码器状态存储到内部状态缓冲区中。每次迭代需指定不同的状态缓冲区索引，以便后续恢复编码器状态。

4.  调用 `NvEncEncodePicture()` API，该函数需返回 `NV_ENC_SUCCESS`。

5.  重复步骤 1\~4，根据需求对同一帧进行多次迭代编码（使用不同编码参数）。由于编码器状态存储在内部状态缓冲区中，帧的最大迭代次数取决于可用于存储状态的缓冲区数量。

6.  调用 `NvEncLockBitstream()` API 获取所有迭代的编码输出，也可在每次迭代后立即调用该 API。

7.  选择需恢复的帧迭代对应的内部状态缓冲区索引，将其赋值给 `NV_ENC_RESTORE_ENCODER_STATE_PARAMS::bufferIdx`；同时选择需更新的状态类型（`NV_ENC_STATE_RESTORE_TYPE`），将其赋值给 `NV_ENC_RESTORE_ENCODER_STATE_PARAMS::state`，然后调用 `NvEncRestoreEncoderState()` API。

8.  若应用程序选择的状态类型不是 `NV_ENC_STATE_RESTORE_FULL`，需调用两次 `NvEncRestoreEncoderState()` API：第一次将 `NV_ENC_RESTORE_ENCODER_STATE_PARAMS::state` 设为 `NV_ENC_STATE_RESTORE_ENCODE`，第二次设为 `NV_ENC_STATE_RESTORE_RATE_CONTROL`（两次调用的状态缓冲区索引需为目标索引，顺序可任意）。

9.  编码下一帧时，递增 `NV_ENC_PIC_PARAMS::frameIdx`，重复步骤 1\~8。

#### 8.17.3 NVENCODE API 决定图像类型（PTD）时 H.264 和 HEVC 的迭代编码

对同一帧进行多次编码需遵循以下步骤：



1.  按上述小节设置 `NV_ENC_PIC_PARAMS::encodePicFlags`、`NV_ENC_PIC_PARAMS::frameIdx` 和 `NV_ENC_PIC_PARAMS::stateBufferIdx` 为有效值，调用 `NvEncEncodePicture()` API。该 API 会返回 `NV_ENC_SUCCESS` 或 `NV_ENC_ERR_NEED_MORE_INPUT`。

2.  若返回 `NV_ENC_SUCCESS`，当前帧可立即进行迭代编码。

3.  若返回 `NV_ENC_ERR_NEED_MORE_INPUT`，当前帧暂无法进行迭代编码 —— 需继续传入下一帧进行编码，直至 API 返回 `NV_ENC_SUCCESS`，此时可对返回 `NV_ENC_SUCCESS` 的帧进行迭代编码。

4.  调用 `NvEncLockBitstream()` API 获取第一次迭代的编码输出。

5.  若 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 与步骤 1 或 3 中的 `NV_ENC_PIC_PARAMS::frameIdx` 相同，需调用 `NvEncLockBitstream()` API 获取所有剩余迭代的编码输出。

6.  部分场景下，`NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 可能与步骤 1 或 3 中的 `NV_ENC_PIC_PARAMS::frameIdx` 不同，表明当前接收的帧并非进行迭代编码的帧 —— 此时需对 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 对应的帧进行迭代编码（若需），调用 `NvEncLockBitstream()` API 获取该帧所有迭代的编码输出，再调用 `NvEncRestoreEncoderState()` API 恢复状态。NVENCODE API 会保存步骤 1 或 3 中 `NV_ENC_PIC_PARAMS::frameIdx` 对应帧的所有迭代编码参数，待所有返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧编码完成且状态恢复后，再将其传入编码。

7.  调用 `NvEncRestoreEncoderState()` API 恢复目标帧迭代的状态。

8.  若部分帧返回 `NV_ENC_ERR_NEED_MORE_INPUT`，NVENCODE API 会将其中一帧传入编码。

9.  若传入编码的帧未将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE`，后续帧也会传入编码。

10. 若存在返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧，需调用 `NvEncLockBitstream()` API，`NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 会指示当前可进行迭代编码的帧。

11. 对所有返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧，重复上述步骤。

##### 缓冲区重排序

存在 B 帧时，驱动会对缓冲区进行重排序，涉及输出码流缓冲区、完成事件和重建帧缓冲区。重排序的目的是让应用程序能按解码顺序获取输出，无需关注图像类型。

需注意，状态缓冲区索引不会进行重排序。

下表展示了 API 调用与对应存储编码输出、重建帧输出和内部状态的缓冲区：



| 序号 | API 调用                                                | 返回参数                                            | 说明                                                                                                         |
| -- | ----------------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 1  | `NvEncEncodePicture (I1, N1=1, O1, E1, R1, F1=0)`     | `NV_ENC_SUCCESS`                                | I1 = 输入缓冲区，N1 = 帧索引，O1 = 输出缓冲区，E1 = 完成事件，R1 = 重建缓冲区，F1=0 表示未设置 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE` |
| 2  | `NvEncLockBitstream(O1)`                              | `frameIdxDisplay=1, picType: NV_ENC_PIC_TYPE_I` | -                                                                                                          |
| 3  | `NvEncEncodePicture (I2, N2=2, O2, E2, R2, S1, F2=1)` | `NV_ENC_ERR_NEED_MORE_INPUT`                    | S1 = 状态缓冲区索引，F2=1 表示设置 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE`                                         |
| 4  | `NvEncEncodePicture (I3, N3=3, O3, E3, R3, S2, F3=1)` | `NV_ENC_SUCCESS`                                | -                                                                                                          |
| 5  | `NvEncEncodePicture (I3, N3=3, O4, E4, R4, S3, F4=1)` | `NV_ENC_SUCCESS`                                | -                                                                                                          |
| 6  | `NvEncLockBitstream(O2)`                              | `frameIdxDisplay=3, picType: NV_ENC_PIC_TYPE_P` | 帧 3 第 1 次迭代的输出，重建输出在 R3 中，内部状态存储在 S2 中                                                                     |
| 7  | `NvEncLockBitstream(O3)`                              | `frameIdxDisplay=3, picType: NV_ENC_PIC_TYPE_P` | 帧 3 第 2 次迭代的输出，重建输出在 R4 中，内部状态存储在 S3 中                                                                     |
| 8  | `NvEncRestoreEncoderState(S2 或 S3)`                   | `NV_ENC_SUCCESS`                                | 帧 2 会传入编码                                                                                                  |
| 9  | `NvEncLockBitstream (O4)`                             | `frameIdxDisplay=2, picType: NV_ENC_PIC_TYPE_B` | 帧 2 第 1 次迭代的输出，重建输出在 R2 中，内部状态存储在 S1 中                                                                     |
| 10 | `NvEncEncodePicture (I2, N2=2, O5, E5, R5, S4, F5=1)` | `NV_ENC_SUCCESS`                                | -                                                                                                          |
| 11 | `NvEncLockBitstream(O5)`                              | `frameIdxDisplay=2, picType: NV_ENC_PIC_TYPE_B` | 帧 2 第 2 次迭代的输出，重建输出在 R5 中，内部状态存储在 S4 中                                                                     |
| 12 | `NvEncRestoreEncoderState(S1 或 S4)`                   | `NV_ENC_SUCCESS`                                | -                                                                                                          |

#### 8.17.4 NVENCODE API 决定图像类型（PTD）时 AV1 的迭代编码

由于 AV1 格式存在非显示帧（当 `NV_ENC_CONFIG::frameIntervalP` 设为大于 1 时启用），其迭代编码与 H.264 和 HEVC 存在差异，以下为详细说明：

对同一帧进行多次编码需遵循以下步骤：



1.  按上述小节设置 `NV_ENC_PIC_PARAMS::encodePicFlags`、`NV_ENC_PIC_PARAMS::frameIdx` 和 `NV_ENC_PIC_PARAMS::stateBufferIdx` 为有效值，调用 `NvEncEncodePicture()` API。该 API 会返回 `NV_ENC_SUCCESS` 或 `NV_ENC_ERR_NEED_MORE_INPUT`。

2.  若返回 `NV_ENC_SUCCESS`，当前帧可立即进行迭代编码。

3.  若返回 `NV_ENC_ERR_NEED_MORE_INPUT`，当前帧暂无法进行迭代编码 —— 需继续传入下一帧进行编码，直至 API 返回 `NV_ENC_SUCCESS`，此时可对返回 `NV_ENC_SUCCESS` 的帧进行迭代编码。

4.  调用 `NvEncLockBitstream()` API 获取第一次迭代的编码输出。

5.  若 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 与步骤 1 或 3 中的 `NV_ENC_PIC_PARAMS::frameIdx` 相同，需调用 `NvEncLockBitstream()` API 获取所有剩余迭代的编码输出。

6.  部分场景下，`NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 可能与步骤 1 或 3 中的 `NV_ENC_PIC_PARAMS::frameIdx` 不同，表明当前接收的帧并非进行迭代编码的帧 —— 此时需对 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 对应的帧进行迭代编码（若需），调用 `NvEncLockBitstream()` API 获取该帧所有迭代的编码输出，再调用 `NvEncRestoreEncoderState()` API 恢复状态。NVENCODE API 会保存步骤 1 或 3 中 `NV_ENC_PIC_PARAMS::frameIdx` 对应帧的所有迭代编码参数，待所有返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧编码完成且状态恢复后，再将其传入编码。

7.  若 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE` 的帧（对应 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay`）存在之前返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧，则当前接收的编码帧为非显示帧。

8.  对于任何非显示帧，无论其迭代次数多少，对应的 OVERLAY 帧仅会编码一次 —— 需在所有之前返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧编码完成后进行。

9.  应用程序需调用 `NvEncRestoreEncoderState()` API 恢复该帧状态，该 API 会返回 `NV_ENC_ERR_NEED_MORE_OUTPUT` 或 `NV_ENC_SUCCESS`。

10. 若返回 `NV_ENC_ERR_NEED_MORE_OUTPUT`，需在 `NV_ENC_RESTORE_ENCODER_STATE_PARAMS::outputBitstream` 中传入输出缓冲区，再次调用 `NvEncRestoreEncoderState()` API；若启用异步编码模式，还需在 `NV_ENC_RESTORE_ENCODER_STATE_PARAMS::completionEvent` 中传入完成事件。

11. 若 `NvEncRestoreEncoderState()` API 返回 `NV_ENC_SUCCESS`，NVENCODE API 会将之前返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧之一传入编码。

12. 若传入编码的帧未将 `NV_ENC_PIC_PARAMS::encodePicFlags` 设为 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE`，后续帧也会传入编码。

13. 若存在返回 `NV_ENC_ERR_NEED_MORE_OUTPUT` 的帧，需调用 `NvEncLockBitstream()` API。

14. 应用程序应按帧传入编码的顺序，接收这些帧的编码输出。

15. 若 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 的顺序与传入顺序不同，表明接收的是 non-displayable frame（非显示帧）。

16. 此时可对 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 对应的帧进行迭代编码（若需）。

17. 对于任何非显示帧，对应的 OVERLAY 帧仅会编码一次 —— 需在所有之前返回 `NV_ENC_ERR_NEED_MORE_INPUT` 的帧编码完成后进行。

18. 对所有返回 `NV_ENC_ERR_NEED_MORE_OUTPUT` 的帧，重复步骤 5\~6。

下表展示了 API 调用与对应存储编码输出、重建帧输出和内部状态的缓冲区：



| 序号 | API 调用                                                | 返回参数                                            | 说明                                                                                                         |
| -- | ----------------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 1  | `NvEncEncodePicture (I1, N1=1, O1, E1, R1, F1=0)`     | `NV_ENC_SUCCESS`                                | I1 = 输入缓冲区，N1 = 帧索引，O1 = 输出缓冲区，E1 = 完成事件，R1 = 重建缓冲区，F1=0 表示未设置 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE` |
| 2  | `NvEncLockBitstream(O1)`                              | `frameIdxDisplay=1, picType: NV_ENC_PIC_TYPE_I` | 重建输出在 R1 中                                                                                                 |
| 3  | `NvEncEncodePicture (I2, N2=2, O2, E2, R2, S1, F2=1)` | `NV_ENC_ERR_NEED_MORE_INPUT`                    | S1 = 状态缓冲区索引，F2=1 表示设置 `NV_ENC_PIC_FLAG_DISABLE_ENC_STATE_ADVANCE`                                         |
| 4  | `NvEncEncodePicture (I3, N3=3, O3, E3, R3, S2, F3=1)` | `NV_ENC_SUCCESS`                                | -                                                                                                          |
| 5  | `NvEncEncodePicture (I3, N3=3, O4, E4, R4, S3, F4=1)` | `NV_ENC_SUCCESS`                                | -                                                                                                          |
| 6  | `NvEncLockBitstream(O2)`                              | `frameIdxDisplay=3, picType: NV_ENC_PIC_TYPE_P` | 帧 3 第 1 次迭代的输出，重建输出在 R3 中，内部状态存储在 S2 中，当前为非显示帧                                                             |
| 7  | `NvEncLockBitstream(O3)`                              | `frameIdxDisplay=3, picType: NV_ENC_PIC_TYPE_P` | 帧 3 第 2 次迭代的输出，重建输出在 R4 中，内部状态存储在 S3 中，当前为非显示帧                                                             |
| 8  | `NvEncRestoreEncoderState(S2 或 S3)`                   | `NV_ENC_ERR_NEED_MORE_OUTPUT`                   | 需再次调用该 API                                                                                                 |
| 9  | `NvEncRestoreEncoderState(O5, E5, S2 或 S3)`           | `NV_ENC_SUCCESS`                                | 帧 2 会传入编码                                                                                                  |
| 10 | `NvEncLockBitstream (O4)`                             | `frameIdxDisplay=2, picType: NV_ENC_PIC_TYPE_B` | 帧 2 第 1 次迭代的输出，重建输出在 R2 中，内部状态存储在 S1 中                                                                     |
| 11 | `NvEncEncodePicture (I2, N2=2, O6, E6, R5, S4, F5=1)` | `NV_ENC_SUCCESS`                                | -                                                                                                          |
| 12 | `NvEncLockBitstream(O5)`                              | `frameIdxDisplay=2, picType: NV_ENC_PIC_TYPE_B` | 帧 2 第 2 次迭代的输出，重建输出在 R5 中，内部状态存储在 S4 中                                                                     |
| 13 | `NvEncRestoreEncoderState(S1 或 S4)`                   | `NV_ENC_SUCCESS`                                | 帧 3 对应的 OVERLAY 帧会传入编码                                                                                     |
| 14 | `NvEncLockBitstream(O6)`                              | `frameIdxDisplay=3, picType: NV_ENC_PIC_TYPE_P` | 帧 3 对应的 OVERLAY 帧                                                                                          |

**注意**：当应用程序决定图像类型时，重建缓冲区不会进行重排序，应用程序需通过 `NV_ENC_LOCK_BITSTREAM::frameIdxDisplay` 跟踪该缓冲区。

### 8.18 外部前瞻（External lookahead）

从 SDK 12.1 开始，NVENCODE API 支持 Turing 及后续架构 GPU 的 H.264、HEVC 和 AV1 编码器外部前瞻。外部前瞻与内部前瞻（仅需设置 `NV_ENC_RC_PARAMS::lookaheadDepth` 即可启用）的效果相同，但内部前瞻不支持迭代编码 —— 因此，外部前瞻的优势在于可与迭代编码配合使用。

使用外部前瞻的步骤如下：



1.  调用 `nvEncInitializeEncoder()` API 时，将 `NV_ENC_RC_PARAMS::enableExtLookahead` 设为 1，并将 `NV_ENC_RC_PARAMS::lookaheadDepth` 设为目标值。

2.  对每帧需执行以下操作：

    调用 `NvEncLookaheadPicture()`，将 `NV_ENC_LOOKAHEAD_PIC_PARAMS::inputBuffer` 设为通过 `::NvEncCreateInputBuffer()` 或 `::NvEncMapInputResource()` API 获取的指针。

3.  若前瞻深度为 N，需先调用 N+1 次 `NvEncLookaheadPicture()` API，再调用 `NvEncEncodePicture()` API 编码第一帧。

**示例**：若前瞻深度为 4：



1.  调用 `NvEncLookaheadPicture()` API 处理帧 0

2.  调用 `NvEncLookaheadPicture()` API 处理帧 1

3.  调用 `NvEncLookaheadPicture()` API 处理帧 2

4.  调用 `NvEncLookaheadPicture()` API 处理帧 3

5.  调用 `NvEncLookaheadPicture()` API 处理帧 4

6.  调用 `NvEncEncodePicture()` API 编码帧 0

7.  调用 `NvEncLookaheadPicture()` API 处理帧 5

8.  调用 `NvEncEncodePicture()` API 编码帧 1

9.  调用 `NvEncLookaheadPicture()` API 处理帧 6

10. 以此类推……

### 8.19 单向 B 帧（Unidirectional B Frames）

单向 B 帧的 L0 和 L1 参考列表均仅使用过去的帧，避免了传统 B 帧的延迟问题，因此可替代 P 帧用于低延迟编码场景。单向 B 帧能提升视频编码质量，且对性能影响极小。

启用该功能需遵循以下步骤：



1.  通过 `NvEncGetEncodeCaps` API 查询功能支持情况，检查返回值中是否包含 `NV_ENC_CAPS_SUPPORT_UNIDIRECTIONAL_B`。

2.  初始化编码器时，将 `enableUniDirectionalB` 设为 1。

**注意**：当前该功能仅支持 HEVC 格式。

### 8.20 前瞻级别（Lookahead Level）

前瞻级别通过让编码器缓存指定数量的帧、估算帧复杂度并按复杂度比例为帧分配比特，提升视频编码质量；同时，它还能确定编码树单元（CTB）的传播路径，为传播范围最大的 CTB 分配更低的 QP 值，进而提升编码器的码率控制精度。

前瞻级别分为 4 种，分别对应不同的质量与性能平衡：`NV_ENC_LOOKAHEAD_LEVEL_0` 性能最高，`NV_ENC_LOOKAHEAD_LEVEL_3` 质量最高。用户可根据自身质量 / 性能需求选择合适的前瞻级别。

启用该功能需遵循以下步骤：



1.  通过 `NvEncGetEncodeCaps` 查询当前硬件是否支持该功能，检查 `NV_ENC_CAPS_SUPPORT_LOOKAHEAD_LEVEL`。

2.  初始化时，将 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.enableLookahead` 设为 1，启用前瞻功能。

3.  初始化时，将 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.lookaheadLevel` 设为 `NV_ENC_LOOKAHEAD_LEVEL_0`\~`NV_ENC_LOOKAHEAD_LEVEL_3`，指定前瞻级别。

4.  在 `NV_ENC_INITIALIZE_PARAMS::encodeconfig->rcParams.lookaheadDepth` 中设置前瞻帧数（最大支持 32 帧）。

启用该功能后，帧会在编码器中排队，因此在编码器获取足够输入帧满足前瞻需求前，`NvEncEncodePicture` 会返回 `NV_ENC_ERR_NEED_MORE_INPUT`。需持续输入帧，直至 `NvEncEncodePicture` 返回 `NV_ENC_SUCCESS`。

### 8.21 时间滤波（Temporal Filter）

时间滤波会根据相邻的过去帧和未来帧对当前帧进行滤波，对相机拍摄的自然视频内容（可能包含传感器噪声或其他噪声）效果显著。时间滤波能提升视频的客观质量，尤其适用于可容忍延迟的编码场景。

应用程序中启用时间滤波的步骤如下：



1.  调用 `NvEncGetEncodeCaps` API，检查 `NV_ENC_CAPS_SUPPORT_TEMPORAL_FILTER`，查询当前硬件是否支持时间滤波。

2.  若支持，初始化时将 `NV_ENC_CONFIG_HEVC::tfLevel` 设为 `NV_ENC_TEMPORAL_FILTER_LEVEL_4`，启用时间滤波。

时间滤波需使用 CUDA 预处理，因此会消耗一定的 CUDA 计算资源（具体取决于分辨率和内容），且可能导致编码器性能轻微下降。

### 8.22 HEVC 多视角视频编码（MV-HEVC）

HEVC 视频压缩标准包含在单个码流中编码多个相关视角或层的扩展功能。HEVC 规范（ITU-T H.265 / ISO/IEC 23008-2）的附录 G（基于附录 F）定义了使用 HEVC 表示和解码多个层的语法。这些扩展的关键应用之一是多视角 HEVC（MV-HEVC）—— 通过将左右眼视角作为同一码流中的独立层进行编码，实现立体 3D 视频的高效编码。该方式可利用两个视角间的冗余，高效传输立体 3D 视频所需的两个视角数据。

应用程序中使用 MV-HEVC 需遵循以下步骤：



1.  调用 `NvEncGetEncodeCaps` API，检查 `NV_ENC_CAPS_SUPPORT_MVHEVC_ENCODE`，查询当前硬件是否支持 MV-HEVC。

2.  若支持，初始化编码器时将 `NV_ENC_CONFIG_HEVC::enableMVHEVC` 设为 1。

3.  当前仅支持 2 个视角。

4.  NVENCODE API 要求视角按 “帧 0（视角 0）、帧 0（视角 1）、帧 1（视角 0）、帧 1（视角 1）……” 的顺序传入，因此需合理设置 `NV_ENC_PIC_PARAMS_HEVC::viewId`。

5.  NVENCODE API 支持在码流中添加 3D 参考显示信息 SEI 消息：将 `NV_ENC_CONFIG_HEVC::outputHevc3DReferenceDisplayInfo` 设为 1 即可启用，该 SEI 消息会附加到编码码流的每个 IDR 帧中。需注意，也可通过填充 `NV_ENC_PIC_PARAMS_HEVC::p3DReferenceDisplayInfo` 结构体，插入用户指定的 HEVC_3D_REFERENCE_DISPLAY_INFO。

**注意**：当前 MV-HEVC 不支持与以下编码功能同时使用：



*   长参考帧（LTR）

*   Alpha 层编码（Alpha Layer Encoding）

*   单向 B 帧（UniDirectionalB）

*   前瞻（Lookahead）

*   时间滤波（Temporal Filter）

*   分帧编码（Split encoding）

*   双 pass 编码（2 pass encoding）

*   除 `NV_ENC_TUNING_INFO_HIGH_QUALITY` 外的其他 `NV_ENC_TUNING_INFO`

### 8.23 HDR10/HDR10+：MaxCLL、主显示信息（Mastering Display）及 ITU-T T.35 SEI / 元数据

HDR10 和 HDR10+ 的 SEI / 元数据会嵌入 HEVC/AV1 码流中，确保 HDR 内容在不同设备上的准确显示。这些元数据包括 MaxCLL（最大内容光亮度）和主显示信息（描述峰值亮度和色彩范围，支持正确的色调映射）；HDR10+ 还会使用 ITU-T T.35 SEI / 元数据进行逐帧动态调整，进一步提升视觉质量。嵌入这些元数据可确保 HDR 内容在支持 HDR 的显示设备上的兼容性和最佳性能。

API 对 HDR10 和 HDR10+ SEI / 元数据的支持方式如下：



*   **HEVC 格式**：

1.  内容光亮度信息（Content Light Level）SEI 消息：将 `NV_ENC_CONFIG_HEVC::outputMaxCll` 设为 1 即可启用，该 SEI 消息会附加到编码码流的每个 IDR 帧中。也可通过填充 `NV_ENC_PIC_PARAMS_HEVC::pMaxCll` 结构体，插入用户指定的 CONTENT\_LIGHT\_LEVEL。

2.  主显示色彩范围（Mastering Display Colour Volume）SEI 消息：将 `NV_ENC_CONFIG_HEVC::outputMasteringDisplay` 设为 1 即可启用，该 SEI 消息会附加到编码码流的每个 IDR 帧中。也可通过填充 `NV_ENC_PIC_PARAMS_HEVC::pMasteringDisplay` 结构体，插入用户指定的 MASTERING\_DISPLAY\_INFO。

3.  ITU-T T.35 SEI 消息：可作为用户 SEI 消息嵌入码流，通过 `NV_ENC_PIC_PARAMS_HEVC::seiPayloadArray` 并更新正确的有效载荷类型，写入 ITU-T T.35 数据。

*   **AV1 格式**：

1.  内容光亮度元数据：将 `NV_ENC_CONFIG_AV1::outputMaxCll` 设为 1 即可启用，该元数据会附加到编码码流的每个关键帧（Keyframe）中。也可通过填充 `NV_ENC_PIC_PARAMS_AV1::pMaxCll` 结构体，插入用户指定的 CONTENT\_LIGHT\_LEVEL。

2.  主显示色彩范围元数据：将 `NV_ENC_CONFIG_AV1::outputMasteringDisplay` 设为 1 即可启用，该元数据会附加到编码码流的每个关键帧中。也可通过填充 `NV_ENC_PIC_PARAMS_AV1::pMasteringDisplay` 结构体，插入用户指定的 MASTERING\_DISPLAY\_INFO。

3.  ITU-T T.35 元数据：可作为用户 OBU（有序比特单元）嵌入码流，通过 `NV_ENC_PIC_PARAMS_AV1::obuPayloadArray` 并更新正确的有效载荷类型，写入 ITU-T T.35 数据。

### 8.24 外部运动估计提示（External ME Hints）

可将运动向量提示传递给 NVENC，用于指导运动搜索。对于每个宏块（MB）/ 编码树单元（CTU）/ 超块（SB），可传递 L0 和 L1（B 帧场景）方向的提示向量。

#### 8.24.1 H.264 和 HEVC 格式启用该功能的步骤



1.  初始化编码器时，将 `NV_ENC_INITIALIZE_PARAMS::enableExternalMEHints` 设为 1，并通过填充 `NV_ENC_INITIALIZE_PARAMS::maxMEHintCountsPerBlock` 中的对应字段，指定每个宏块 / 编码树单元（CTU）每种方向（L0/L1）、每种分区类型的最大提示候选数。其中，`NV_ENC_INITIALIZE_PARAMS::maxMEHintCountsPerBlock[0]` 对应 L0 预测器，`NV_ENC_INITIALIZE_PARAMS::maxMEHintCountsPerBlock[1]` 对应 L1 预测器。

2.  对每帧，通过填充 `NV_ENC_PIC_PARAMS::meHintCountsPerBlock` 中的对应字段，指定每个块每种方向、每种分区类型的提示候选数；通过 `NV_ENC_PIC_PARAMS::meExternalHints` 传递存储提示的缓冲区指针。对于每个宏块 / 编码树单元，提示需按以下分区类型顺序提供：16x16、16x8、8x16、8x8；每种分区类型需先提供 L0 提示，再提供 L1 提示（若有）。

3.  需注意：HEVC 格式仅支持 16x16 和 8x8 分区大小；H.264 和 HEVC 格式的外部提示范围（整像素单位）为：水平方向 \[-2048, 2047]，垂直方向 \[-512, 511]；H.264 和 HEVC 格式的提示结构体为 `NVENC_EXTERNAL_ME_HINT`。

#### 8.24.2 AV1 格式启用该功能的步骤



1.  初始化编码器时，将 `NV_ENC_INITIALIZE_PARAMS::enableExternalMEHints` 设为 1，并通过填充 `NV_ENC_INITIALIZE_PARAMS::maxMEHintCountsPerBlock` 中的对应字段，指定每个超块（SB）每种方向（L0/L1）的最大提示候选数。其中，`NV_ENC_INITIALIZE_PARAMS::maxMEHintCountsPerBlock[0]` 对应 L0 预测器，`NV_ENC_INITIALIZE_PARAMS::maxMEHintCountsPerBlock[1]` 对应 L1 预测器。

2.  对每帧，通过填充 `NV_ENC_PIC_PARAMS::meHintCountsPerBlock` 中的对应字段，指定每个块每种方向的提示候选数；通过 `NV_ENC_PIC_PARAMS::meExternalSbHints` 传递存储提示的缓冲区指针，并通过 `NV_ENC_PIC_PARAMS::meSbHintsCount` 字段指定帧的外部运动估计超块（ME SB）提示候选总数。对于每个超块，需先提供 L0 提示，再提供 L1 提示（若有）。

3.  需注意：AV1 格式的外部提示范围（整像素单位）为：水平方向 \[-1023, 1023]，垂直方向 \[-511, 511]；AV1 格式的提示结构体为 `NVENC_EXTERNAL_ME_SB_HINT`。

## 九、推荐的 NVENC 设置

NVIDIA 硬件视频编码器可用于多种应用场景，常见场景包括视频录制（归档）、游戏直播（在线广播 / 多播游戏视频）、转码（直播和视频点播）及流媒体（游戏或直播内容）。每种场景对质量、比特率、延迟容忍度、性能约束等均有独特需求。尽管 NVIDIA 编码器接口提供了丰富的 API 用于控制设置，但下表可作为通用指南，为部分常见场景提供推荐设置，以实现最佳编码码流质量。这些推荐设置尤其适用于第二代 Maxwell 及后续架构的 GPU；对于早期 GPU（Kepler 和第一代 Maxwell），建议以表 5 为起点，调整设置以实现合适的性能 - 质量平衡。



| 应用场景                                       | 优化质量与性能的推荐设置                                                                                                                                                                                                                                                                                               |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 录制 / 归档（Recording/Archiving）               | - 优化信息（Tuning Info）：高质量（High Quality）/ 超高画质（Ultra High Quality）- 码率控制模式：VBR（可变比特率）- VBV 缓冲区大小：极大（4 秒）- 启用 B 帧 \*- 启用前瞻（Look-ahead）- 启用 B 帧作为参考帧- 有限 GOP 长度（2 秒）- 启用自适应量化（AQ）\*\*                                                                                                                           |
| 游戏直播与云转码（Game-casting & Cloud Transcoding） | - 优化信息：高质量（High Quality）/ 超高画质（Ultra High Quality）- 码率控制模式：CBR（恒定比特率）- VBV 缓冲区大小：中等（1 秒）- 启用 B 帧 \*- 启用前瞻（Look-ahead）- 启用 B 帧作为参考帧- 有限 GOP 长度（2 秒）- 启用自适应量化（AQ）\*\*                                                                                                                                        |
| 低延迟场景（如游戏流媒体、视频会议等）（Low-latency use cases） | - 优化信息：超低延迟（Ultra-low Latency）/ 低延迟（Low Latency）- 码率控制模式：CBR（恒定比特率）- 多 pass 模式：1/4 分辨率或全分辨率（评估后选择）- VBV 缓冲区大小：极小（例如单帧 = 比特率 / 帧率）- 启用单向 B 帧（Unidirectional B Frames）- 无限 GOP 长度- 启用自适应量化（AQ）**- 启用长参考帧（LTR）***- 启用帧内刷新（Intra Refresh）***- 启用非参考 P 帧（Non-reference P frames）***- 启用强制 IDR 帧（Force IDR）*\*\* |
| 无损编码（Lossless Encoding）                    | - 优化信息：无损（Lossless）                                                                                                                                                                                                                                                                                        |

\*：推荐用于低运动游戏和自然视频。

\*\*：推荐用于第二代 Maxwell 及后续架构的 GPU。

\*\*\*：这些功能适用于在噪声较大的传输介质中实现错误恢复。

若客户端需降低显存占用，建议遵循以下指南：

1.  避免使用 B 帧：B 帧需额外缓冲区用于重排序，因此避免 B 帧可减少显存占用。

2.  减少最大参考帧数：减少最大参考帧数可使 NVIDIA 显示驱动减少内部分配的缓冲区数量，从而降低显存占用。

3.  使用单 pass 码率控制模式：双 pass 码率控制模式（尤其是首 pass 为全分辨率的模式）比单 pass 模式占用更多显存，因需为第一 pass 编码分配额外资源。

4.  避免启用自适应量化 / 加权预测：自适应量化（AQ）、加权预测（Weighted Prediction）等功能需在显存中分配额外缓冲区，禁用这些功能可避免此类分配。

5.  避免启用前瞻（Look-ahead）：前瞻功能需为前瞻队列中的帧在显存中分配额外缓冲区。

6.  避免启用时间滤波（Temporal Filter）：时间滤波需使用相邻帧，因此需在显存中分配额外缓冲区。

7.  避免使用 UHQ 优化信息（UHQ Tuning Info）：UHQ 优化信息会启用前瞻和时间滤波，这两项功能均需较高显存。

### 9.1 NVENC 性能优化

NVENC 针对高性能 / 高吞吐量编码进行了优化。为最大限度提升其利用率，应用程序需能以足够快的速率提供帧数据。NVENC 的常见瓶颈取决于所实现的视频编码流水线，具体如下：

#### 9.1.1 从磁盘编码帧（Encoding frames from disk）

*   文件读取速度：从磁盘加载原始帧的速度。

*   PCIe 速度：原始帧从主机到设备的传输速度，或压缩帧从设备到主机的传输速度。

#### 9.1.2 从磁盘转码帧（Transcoding frames from disk）

*   解码器速度：解码器可能成为瓶颈（尤其是在主机端运行时）。建议尽可能使用 NVDEC；若 NVDEC 不支持目标格式，建议在设备端运行解码器（例如 NVJPEG 2000）。

*   计算速度：解码器与编码器之间的计算步骤（如像素格式转换、缩放或 AI 滤波）的速度。

#### 9.1.3 通用推荐

1.  应用程序应实现完整（或大部分）视频流水线，即解码、计算和编码均在 GPU 上执行，以减少 PCIe 传输的数据量。

2.  流水线的不同步骤应使用多线程实现并发 —— 例如，编码帧 0 时，将帧 1 加载到设备，同时从磁盘加载帧 2。

3.  使用分帧编码（SFE）等多 NVENC 技术进行转码时，NVDEC 可能成为瓶颈。单个 NVDEC 解码帧的速度通常足以供 2 个 NVENC 使用；若使用 3 路（或更多）分帧编码，建议应用程序利用多个解码器会话（若可能），以增加 NVDEC 的使用数量。

**注意**：上述指南可能导致编码质量略有下降。因此，建议客户端进行充分评估，以实现编码质量、速度和显存占用的平衡。


