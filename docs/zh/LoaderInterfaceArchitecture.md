<!-- markdownlint-disable MD041 -->
[![Khronos Vulkan][1]][2]

[1]: https://vulkan.lunarg.com/img/Vulkan_100px_Dec16.png "https://www.khronos.org/vulkan/"
[2]: https://www.khronos.org/vulkan/

# Vulkan Loader 接口架构（Architecture of the Vulkan Loader Interfaces） <!-- omit from toc -->
[![Creative Commons][3]][4]

<!-- Copyright &copy; 2015-2023 LunarG, Inc. -->

[3]: https://i.creativecommons.org/l/by-nd/4.0/88x31.png "Creative Commons License"
[4]: https://creativecommons.org/licenses/by-nd/4.0/
## 目录 <!-- omit from toc -->

- [概述](#overview)
  - [谁应该阅读本文档](#who-should-read-this-document)
  - [Loader](#the-loader)
    - [Loader 的目标](#goals-of-the-loader)
  - [Layer](#layers)
  - [Driver](#drivers)
    - [可安装客户端驱动（ICD）](#installable-client-drivers)
  - [VkConfig](#vkconfig)
- [重要 Vulkan 概念](#important-vulkan-concepts)
  - [Instance 与 Device](#instance-versus-device)
    - [Instance 相关](#instance-specific)
      - [Instance 对象](#instance-objects)
      - [Instance 函数](#instance-functions)
      - [Instance 扩展](#instance-extensions)
    - [Device 相关](#device-specific)
      - [Device 对象](#device-objects)
      - [Device 函数](#device-functions)
      - [Device 扩展](#device-extensions)
  - [调度表与调用链](#dispatch-tables-and-call-chains)
    - [Instance 调用链示例](#instance-call-chain-example)
    - [Device 调用链示例](#device-call-chain-example)
- [特权提升注意事项](#elevated-privilege-caveats)
- [应用与 Loader 的接口](#application-interface-to-the-loader)
- [Layer 与 Loader 的接口](#layer-interface-with-the-loader)
- [Driver 与 Loader 的接口](#driver-interface-with-the-loader)
- [问题调试](#debugging-issues)
- [Loader 策略](#loader-policies)
- [过滤环境变量行为](#filter-environment-variable-behaviors)
  - [比较字符串](#comparison-strings)
  - [逗号分隔列表](#comma-delimited-lists)
  - [Glob 模式](#globs)
  - [大小写不敏感](#case-insensitive)
  - [环境变量优先级](#environment-variable-priority)
- [调试环境变量总表](#table-of-debug-environment-variables)
  - [当前有效环境变量](#active-environment-variables)
  - [已弃用环境变量](#deprecated-environment-variables)
- [术语表](#glossary-of-terms)

<a id="overview"></a>
## 概述

Vulkan 是一种分层架构（layered architecture），由以下元素构成：
  * Vulkan 应用（Application）
  * [Vulkan Loader](#the-loader)
  * [Vulkan Layer](#layers)
  * [Driver](#drivers)
  * [VkConfig](#vkconfig)

![Loader 高层视图](./images/high_level_loader.png)

本文档中的通用概念适用于 Windows、Linux、Android 与 macOS 系统上的 Vulkan Loader。


<a id="who-should-read-this-document"></a>
### 谁应该阅读本文档

虽然本文档主要面向 Vulkan 应用、驱动与 Layer 的开发者，但其中的信息对任何希望更深入理解 Vulkan 运行时的人都可能有帮助。


<a id="the-loader"></a>
### Loader

应用位于栈顶，并直接与 Vulkan loader 交互。
栈底是驱动。
驱动可以控制一个或多个具备 Vulkan 渲染能力的物理设备，也可以将 Vulkan 转换为原生图形 API（例如 [MoltenVk](https://github.com/KhronosGroup/MoltenVK)），或者实现可在 CPU 上执行的软件路径来模拟 Vulkan 设备（例如 [SwiftShader](https://github.com/google/swiftshader) 或 LavaPipe）。
请记住，具备 Vulkan 能力的硬件可能是图形型、计算型，或两者兼具。
在应用与驱动之间，loader 可以注入任意数量的可选 [layer](#layers)，以提供特定功能。
loader 对于将 Vulkan 函数正确分发（dispatch）到对应的 layer 集合与驱动至关重要。
Vulkan 对象模型允许 loader 将 layer 插入调用链（call-chain），使 layer 能在驱动被调用之前处理 Vulkan 函数。

本文档旨在概述这些组件之间所需的接口。


<a id="goals-of-the-loader"></a>
#### Loader 的目标

loader 的设计目标如下：
  1. 在用户系统上支持一个或多个具备 Vulkan 能力的驱动，并避免它们彼此干扰。
  2. 支持 Vulkan Layer（可选模块），可由应用、开发者或系统标准设置启用。
  3. 将 loader 的整体开销保持在尽可能低的水平。


<a id="layers"></a>
### Layer

Layer 是用于增强 Vulkan 开发环境的可选组件。
它们可以在函数从应用向下传递到驱动、再向上返回的过程中拦截（intercept）、评估并修改现有 Vulkan 函数。
Layer 以库形式实现，可通过不同方式启用，并在 CreateInstance 期间加载。
每个 layer 可以选择 hook（或拦截）Vulkan 函数，进而对其忽略、检查或增强。
如果某个 layer 未 hook 某函数，则该 layer 会被跳过，控制流继续传递给下一个支持该函数的 layer 或驱动。
因此，layer 可以选择拦截所有已知 Vulkan 函数，或只拦截其关心的子集。

layer 可能提供的功能示例包括：
  * API 使用验证
  * API 调用跟踪
  * 调试辅助
  * 性能分析（Profiling）
  * 叠加层（Overlay）

由于 layer 是可选且动态加载的，因此可以按需启用或禁用。
例如，在开发和调试应用时，启用某些 layer 可帮助确保应用正确使用 Vulkan API。
但在发布应用时，这些 layer 通常不再需要，因此不会被启用，从而提升应用速度。


<a id="drivers"></a>
### Driver

实现 Vulkan 的库（无论是直接支持物理硬件设备、将 Vulkan 命令转换为原生命令，还是通过软件模拟 Vulkan）都被视为“驱动（driver）”。
最常见的驱动类型仍然是可安装客户端驱动（Installable Client Driver, ICD）。
loader 负责发现系统中可用的 Vulkan 驱动。
给定可用驱动列表后，loader 可以枚举所有可用物理设备，并将此信息提供给应用。


<a id="installable-client-drivers"></a>
#### 可安装客户端驱动（ICD）

Vulkan 允许多个 ICD，每个 ICD 可支持一个或多个设备。
这些设备中的每一个都由 Vulkan `VkPhysicalDevice` 对象表示。
loader 负责通过系统标准驱动搜索机制发现可用的 Vulkan ICD。


<a id="vkconfig"></a>
### VkConfig

VkConfig 是 LunarG 开发的工具，用于协助修改本地系统上的 Vulkan 环境。
它可用于查找 layer、启用 layer、修改 layer 设置以及其他实用功能。
VkConfig 可通过安装 [Vulkan SDK](https://vulkan.lunarg.com/) 获得，或从 [LunarG VulkanTools GitHub 仓库](https://github.com/LunarG/VulkanTools) 构建源码获得。

VkConfig 会生成三个输出，其中两个与 Vulkan loader 和 layer 协同工作。
这些输出包括：
  * Vulkan Override Layer
  * Vulkan Layer Settings File
  * VkConfig Configuration Settings

这些文件在不同平台上的位置如下：

<table style="width:100%">
  <tr>
    <th>平台</th>
    <th>输出</th>
    <th>位置</th>
  </tr>
  <tr>
    <th rowspan="3">Linux</th>
    <td>Vulkan Override Layer</td>
    <td>$USER/.local/share/vulkan/implicit_layer.d/VkLayer_override.json</td>
  </tr>
  <tr>
    <td>Vulkan Layer Settings</td>
    <td>$USER/.local/share/vulkan/settings.d/vk_layer_settings.txt</td>
  </tr>
  <tr>
    <td>VkConfig Configuration Settings</td>
    <td>$USER/.local/share/vulkan/settings.d/vk_layer_settings.txt</td>
  </tr>
  <tr>
    <th rowspan="3">Windows</th>
    <td>Vulkan Override Layer</td>
    <td>%HOME%\AppData\Local\LunarG\vkconfig\override\VkLayerOverride.json</td>
  </tr>
  <tr>
    <td>Vulkan Layer Settings</td>
    <td>(registry) HKEY_CURRENT_USER\Software\Khronos\Vulkan\LoaderSettings</td>
  </tr>
  <tr>
    <td>VkConfig Configuration Settings</td>
    <td>(registry) HKEY_CURRENT_USER\Software\LunarG\vkconfig </td>
  </tr>
</table>

[Override Meta-Layer](./LoaderLayerInterface.md#override-meta-layer) 是 VkConfig 工作机制中的关键组成部分。
当 loader 发现该 layer 时，它会强制加载在 VkConfig 中启用的目标 layer，并禁用被有意禁用的 layer（包括隐式 layer）。

Vulkan Layer Settings 文件可用于指定各已启用 layer 预期执行的某些行为和动作。
这些设置也可由 VkConfig 控制，或手动启用。
关于可用设置详情，请参考各个具体 layer 的文档。

未来，VkConfig 可能还会与 Vulkan loader 产生更多交互。

有关 VkConfig 的更多细节，请参阅其 [GitHub 文档](https://github.com/LunarG/VulkanTools/blob/main/vkconfig/README.md)。
<br/>
<br/>


<a id="important-vulkan-concepts"></a>
## 重要 Vulkan 概念

Vulkan 有一些构成其组织基础的核心概念。
任何尝试使用 Vulkan 或开发其组件的人都应理解这些概念。


<a id="instance-versus-device"></a>
### Instance 与 Device

本文档中会反复提到的一个重要概念是 Vulkan API 的组织方式。
Vulkan 中许多对象、函数、扩展和其他行为都可划分为两组：
  * [Instance 相关](#instance-specific)
  * [Device 相关](#device-specific)


<a id="instance-specific"></a>
#### Instance 相关

“Vulkan instance”（`VkInstance`）是用于提供 Vulkan 系统级信息与功能的高层构造。

<a id="instance-objects"></a>
##### Instance 对象

与 instance 直接相关的一些 Vulkan 对象包括：
  * `VkInstance`
  * `VkPhysicalDevice`
  * `VkPhysicalDeviceGroup`

<a id="instance-functions"></a>
##### Instance 函数

“instance function”指首个参数为 [instance 对象](#instance-objects) 或不带任何对象参数的 Vulkan 函数。

一些 Vulkan instance 函数包括：
  * `vkEnumerateInstanceExtensionProperties`
  * `vkEnumeratePhysicalDevices`
  * `vkCreateInstance`
  * `vkDestroyInstance`

应用可通过 Vulkan loader 的头文件直接链接所有核心 instance 函数。
或者，应用可以使用 `vkGetInstanceProcAddr` 查询函数指针。
`vkGetInstanceProcAddr` 除了可查询所有核心入口点外，还可查询任意 instance 或 device 入口点。

如果使用某个 `VkInstance` 调用 `vkGetInstanceProcAddr`，则返回的函数指针将特定于该 `VkInstance` 以及由其创建的其他对象。

<a id="instance-extensions"></a>
##### Instance 扩展

Vulkan 扩展同样会根据其提供的函数类型进行关联划分。
因此，扩展被分为 instance 扩展和 device 扩展，其中该扩展中的大多数（甚至全部）函数都属于对应类型。
例如，“instance extension”主要由“instance functions”组成，这些函数主要接收 instance 对象。
这些内容将在后文进一步讨论。


<a id="device-specific"></a>
#### Device 相关

另一方面，Vulkan device（`VkDevice`）是逻辑标识符，用于通过用户系统上的特定驱动，将函数与特定 Vulkan 物理设备（`VkPhysicalDevice`）关联起来。

<a id="device-objects"></a>
##### Device 对象

与 device 直接关联的一些 Vulkan 构造包括：
  * `VkDevice`
  * `VkQueue`
  * `VkCommandBuffer`

<a id="device-functions"></a>
##### Device 函数

“device function”指首个参数为任意 device 对象或其子对象的 Vulkan 函数。
绝大多数 Vulkan 函数都是 device 函数。
一些 Vulkan device 函数包括：
  * `vkQueueSubmit`
  * `vkBeginCommandBuffer`
  * `vkCreateEvent`

Vulkan device 函数可通过 `vkGetInstanceProcAddr` 或 `vkGetDeviceProcAddr` 查询。
如果应用选择使用 `vkGetInstanceProcAddr`，则每次调用的调用链会包含额外的函数调用，性能会略有下降。
如果改用 `vkGetDeviceProcAddr`，调用链会针对特定 device 更优化，但返回的函数指针**仅**能用于查询时所使用的 device。
与 `vkGetInstanceProcAddr` 不同，`vkGetDeviceProcAddr` 只能用于 Vulkan device 函数。

最佳方案是：使用 `vkGetInstanceProcAddr` 查询 instance 扩展函数，使用 `vkGetDeviceProcAddr` 查询 device 扩展函数。
更多信息参见 [LoaderApplicationInterface.md](LoaderApplicationInterface.md) 中的 [Best Application Performance Setup](LoaderApplicationInterface.md#best-application-performance-setup) 章节。

<a id="device-extensions"></a>
##### Device 扩展

与 instance 扩展类似，device 扩展是用于扩展 Vulkan 语言的一组 Vulkan device 函数。
有关 device 扩展的更多信息可在本文后续找到。


<a id="dispatch-tables-and-call-chains"></a>
### 调度表与调用链

Vulkan 使用对象模型来控制特定动作或操作的作用域。
被操作对象总是 Vulkan 调用的第一个参数，且是可分发对象（dispatchable object，参见 Vulkan 规范 3.3 Object Model 章节）。
在底层，可分发对象句柄是一个指向结构体的指针，该结构体又包含一个指向由 loader 维护的调度表（dispatch table）的指针。
该调度表包含适用于该对象的 Vulkan 函数指针。

loader 维护两类调度表：
  - Instance Dispatch Table
    - 在调用 `vkCreateInstance` 时由 loader 创建
  - Device Dispatch Table
    - 在调用 `vkCreateDevice` 时由 loader 创建

在此时，应用和系统各自都可指定要包含的可选 layer。
loader 会初始化指定 layer，为每个 Vulkan 函数创建调用链，并将调度表中每个条目指向该调用链的首元素。
因此，loader 会为每个创建出来的 `VkInstance` 构建 instance 调用链，并为每个创建出来的 `VkDevice` 构建设备调用链。

当应用调用 Vulkan 函数时，通常会先进入 loader 中的 *trampoline* 函数。
这些 *trampoline* 函数是小而简单的函数，用于跳转到给定对象对应的调度表条目。
此外，对于 instance 调用链中的函数，loader 还包含一个额外函数，称为 *terminator*，它在所有已启用 layer 之后被调用，用于将适当信息编组（marshall）到所有可用驱动。


<a id="instance-call-chain-example"></a>
#### Instance 调用链示例

例如，下图展示了 `vkCreateInstance` 的调用链中发生的过程。
在初始化链后，loader 调用第一个 layer 的 `vkCreateInstance`；该 layer 再调用下一个 layer 的 `vkCreateInstance`，最终再次回到 loader，由 loader 调用每个驱动的 `vkCreateInstance`。
这使调用链中每个已启用 layer 都能基于应用传入的 `VkInstanceCreateInfo` 结构体完成其所需设置。

![Instance 调用链](./images/loader_instance_chain.png)

这也突显了 loader 在 instance 调用链场景下必须管理的一些复杂性。
如图所示，当存在多个驱动时，loader 的 *terminator* 必须对来自多个驱动的信息进行聚合。
这意味着 loader 必须了解所有作用于 `VkInstance` 的 instance 级扩展，以便正确聚合。


<a id="device-call-chain-example"></a>
#### Device 调用链示例

device 调用链在 `vkCreateDevice` 中创建，通常更简单，因为它只处理单个 device。
这使得暴露该 device 的特定驱动总是可以成为该链的 *terminator*。

![Loader Device 调用链](./images/loader_device_chain_loader.png)
<br/>


<a id="elevated-privilege-caveats"></a>
## 特权提升注意事项

为了确保系统不被利用，使用提升权限（elevated privileges）运行的 Vulkan 应用会被限制执行某些操作，例如从不安全位置读取环境变量或在用户可控路径中搜索文件。
这样做是为了确保高权限应用不会使用未安装在经过批准位置的组件。

loader 会使用平台特定机制（如 `secure_getenv` 及其等价实现）查询敏感环境变量，以避免意外使用不可信结果。

这些行为还会导致某些环境变量被忽略，例如：

  * `VK_DRIVER_FILES` / `VK_ICD_FILENAMES`
  * `VK_ADD_DRIVER_FILES`
  * `VK_LAYER_PATH`
  * `VK_ADD_LAYER_PATH`
  * `VK_IMPLICIT_LAYER_PATH`
  * `VK_ADD_IMPLICIT_LAYER_PATH`
  * `XDG_CONFIG_HOME`（Linux/Mac 特有）
  * `XDG_DATA_HOME`（Linux/Mac 特有）

有关受影响搜索路径的更多信息，请参阅 [Layer Discovery](LoaderLayerInterface.md#layer-discovery) 和 [Driver Discovery](LoaderDriverInterface.md#driver-discovery)。
<br/>
<br/>


<a id="application-interface-to-the-loader"></a>
## 应用与 Loader 的接口

应用与 Vulkan loader 的接口细节现已在与本文件同目录的 [LoaderApplicationInterface.md](LoaderApplicationInterface.md) 文档中说明。
<br/>
<br/>


<a id="layer-interface-with-the-loader"></a>
## Layer 与 Loader 的接口

Layer 与 Vulkan loader 的接口细节在与本文件同目录的 [LoaderLayerInterface.md](LoaderLayerInterface.md) 文档中说明。
<br/>
<br/>


<a id="driver-interface-with-the-loader"></a>
## Driver 与 Loader 的接口

Driver 与 Vulkan loader 的接口细节在与本文件同目录的 [LoaderDriverInterface.md](LoaderDriverInterface.md) 文档中说明。
<br/>
<br/>


<a id="debugging-issues"></a>
## 问题调试


如果你的应用崩溃或行为异常，loader 提供了多种机制帮助你调试问题。
这些机制在与本文件同目录的 [LoaderDebugging.md](LoaderDebugging.md) 文档中有详细说明。
<br/>
<br/>


<a id="loader-policies"></a>
## Loader 策略

关于 loader 与驱动、layer 的交互策略，现已记录在相应章节中。
这些章节旨在明确界定 loader 在与这些组件交互时的预期行为。
在需要实现新型或专用 loader，且其行为需与现有 loader 一致的场景下，这些内容尤为有用。
因此，这些章节的主要关注点是定义相关组件的预期行为，以在各平台提供一致体验。
从长期来看，这些内容也可作为现有 Vulkan loader 的验证要求。

如需查看具体策略章节，请参考以下一个或两个部分：
  * [Loader And Driver Policy](LoaderDriverInterface.md#loader-and-driver-policy)
  * [Loader And Layer Policy](LoaderLayerInterface.md#loader-and-layer-policy)
<br/>
<br/>

<a id="filter-environment-variable-behaviors"></a>
## 过滤环境变量行为

在某些区域提供的过滤环境变量具有一些通用限制与行为，需在此说明。

<a id="comparison-strings"></a>
### 比较字符串

过滤变量会与驱动或 layer 的对应字符串进行比较。
对于 layer，对应字符串是 layer 清单文件中的 layer 名称。
由于驱动不像 layer 那样有名称，此处使用该子字符串与驱动清单文件名进行比较。

<a id="comma-delimited-lists"></a>
### 逗号分隔列表

所有过滤环境变量都接受逗号分隔输入。
因此，你可以串联多个字符串，loader 将使用这些字符串分别启用或禁用当前可用项列表中的相应项。

<a id="globs"></a>
### Glob 模式

为提供足够灵活性，使开发者仅匹配所需名称，loader 对字符串使用受限的 glob 格式。
可接受的 glob 包括：
  - 前缀：   `"string*"`
  - 后缀：   `"*string"`
  - 子串：  `"*string*"`
  - 全字符串： `"string"`
    - 对于全字符串场景，字符串会与每个 layer 名称或驱动文件名进行完整比较。
    - 因此，它只会匹配特定目标，例如：
      `VK_LAYER_KHRONOS_validation` 会匹配 layer 名称
      `VK_LAYER_KHRONOS_validation`，但**不会**匹配名为
      `VK_LAYER_KHRONOS_validation2` 的 layer（并不是说真有这个 layer）。

这尤其有用，因为有时很难确定驱动清单文件的完整名称，或某些常用 layer（如 `VK_LAYER_KHRONOS_validation`）的完整名称。

<a id="case-insensitive"></a>
### 大小写不敏感

所有过滤环境变量都假定 glob 内字符串不区分大小写。
因此，“Bob”、“bob” 和 “BOB” 作用等同。

<a id="environment-variable-priority"></a>
### 环境变量优先级

来自 *disable* 环境变量的值会在 *enable* 或 *select* 环境变量之前处理。
因此，可能出现通过 *disable* 环境变量禁用某个 layer/driver 后，又被 *enable*/*select* 环境变量重新启用的情况。
当你希望先禁用全部 layer/driver，再仅启用较小子集以进行问题定位时，这种行为很有用。

### 多重过滤

当提供多个 `VK_LOADER_<DEVICE|VENDOR|DRIVER>_ID_FILTER` 时，它们会累积生效。
只有同时匹配所有过滤器的设备才会呈现给应用。

<a id="table-of-debug-environment-variables"></a>
## 调试环境变量总表

下文列出了可与 Loader 一起使用的全部调试环境变量。
这些变量在正文中已有引用，此处集中列出便于检索。

<a id="active-environment-variables"></a>
### 当前有效环境变量

<table style="width:100%">
  <tr>
    <th>环境变量</th>
    <th>行为</th>
    <th>限制</th>
    <th>示例格式</th>
  </tr>
  <tr>
    <td><small>
        <i>VK_ADD_DRIVER_FILES</i>
    </small></td>
    <td><small>
        提供额外驱动 JSON 文件列表，loader 会在常规发现到的驱动之外一并使用。
        该列表会优先添加，即位于常规发现驱动列表之前。
        该值包含以分隔符分开的驱动 JSON 清单文件完整路径列表。<br/>
    </small></td>
    <td><small>
        如果未使用 JSON 文件的全局路径，可能会遇到问题。
        <br/> <br/>
        <a href="#elevated-privilege-caveats">
            以提升权限运行 Vulkan 应用时会被忽略。
        </a>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_ADD_DRIVER_FILES=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<folder_a>/intel.json:<folder_b>/amd.json
        <br/> <br/>
        set<br/>
        &nbsp;&nbsp;VK_ADD_DRIVER_FILES=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<folder_a>\nvidia.json;<folder_b>\mesa.json
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_ADD_LAYER_PATH</i>
    </small></td>
    <td><small>
        提供额外路径列表，loader 在查找 layer 清单文件时，除了标准 layer 库搜索路径外，也会搜索这些路径中的显式 layer。
        这些路径会优先添加，即位于常规搜索文件夹列表之前。
    </small></td>
    <td><small>
        <a href="#elevated-privilege-caveats">
            以提升权限运行 Vulkan 应用时会被忽略。
        </a>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_ADD_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;:&lt;path_b&gt;<br/><br/>
        set<br/>
        &nbsp;&nbsp;VK_ADD_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;;&lt;path_b&gt;</small>
    </td>
  </tr>
    <tr>
    <td><small>
        <i>VK_ADD_IMPLICIT_LAYER_PATH</i>
    </small></td>
    <td><small>
        提供额外路径列表，loader 在查找 layer 清单文件时，除了标准 layer 库搜索路径外，也会搜索这些路径中的隐式 layer。
        这些路径会优先添加，即位于常规搜索文件夹列表之前。
    </small></td>
    <td><small>
        <a href="#elevated-privilege-caveats">
            以提升权限运行 Vulkan 应用时会被忽略。
        </a>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_ADD_IMPLICIT_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;:&lt;path_b&gt;<br/><br/>
        set<br/>
        &nbsp;&nbsp;VK_ADD_IMPLICIT_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;;&lt;path_b&gt;</small>
    </td>
  </tr>
  <tr>
    <td><small>
        <i>VK_DRIVER_FILES</i>
    </small></td>
    <td><small>
        强制 loader 使用指定驱动 JSON 文件。
        该值包含以分隔符分开的驱动 JSON 清单文件完整路径列表，和/或包含驱动 JSON 文件的文件夹路径。<br/>
        <br/>
        该变量已取代旧的已弃用环境变量 <i>VK_ICD_FILENAMES</i>，但旧变量仍可继续使用。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.3.207 及更高版本构建的 Loader 中可用。<br/>
        建议对 JSON 文件使用绝对路径。
        由于 loader 将相对库路径转换为绝对路径的方式，相对路径可能带来问题。
        <br/> <br/>
        <a href="#elevated-privilege-caveats">
            以提升权限运行 Vulkan 应用时会被忽略。
        </a>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_DRIVER_FILES=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<folder_a>/intel.json:<folder_b>/amd.json
        <br/> <br/>
        set<br/>
        &nbsp;&nbsp;VK_DRIVER_FILES=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<folder_a>\nvidia.json;<folder_b>\mesa.json
        </small>
    </td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LAYER_PATH</i></small></td>
    <td><small>
        覆盖 loader 的标准显式 layer 搜索路径，使用提供的分隔文件和/或文件夹来定位 layer 清单文件。
    </small></td>
    <td><small>
        <a href="#elevated-privilege-caveats">
            以提升权限运行 Vulkan 应用时会被忽略。
        </a>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;:&lt;path_b&gt;<br/><br/>
        set<br/>
        &nbsp;&nbsp;VK_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;;&lt;path_b&gt;
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_IMPLICIT_LAYER_PATH</i></small></td>
    <td><small>
        覆盖 loader 的标准隐式 layer 搜索路径，使用提供的分隔文件和/或文件夹来定位 layer 清单文件。
    </small></td>
    <td><small>
        <a href="#elevated-privilege-caveats">
            以提升权限运行 Vulkan 应用时会被忽略。
        </a>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_IMPLICIT_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;:&lt;path_b&gt;<br/><br/>
        set<br/>
        &nbsp;&nbsp;VK_IMPLICIT_LAYER_PATH=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;path_a&gt;;&lt;path_b&gt;
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DEBUG</i>
    </small></td>
    <td><small>
        使用逗号分隔级别选项启用 loader 调试消息。
        这些选项包括：<br/>
        &nbsp;&nbsp;* error（仅错误）<br/>
        &nbsp;&nbsp;* warn（仅警告）<br/>
        &nbsp;&nbsp;* info（仅信息）<br/>
        &nbsp;&nbsp;* debug（仅调试）<br/>
        &nbsp;&nbsp;* layer（layer 特定输出）<br/>
        &nbsp;&nbsp;* driver（driver 特定输出）<br/>
        &nbsp;&nbsp;* all（输出所有消息）<br/><br/>
        若要启用多个选项（除 "all" 外），例如 info、warn 和 error，可设置为 "error,warn,info"。
    </small></td>
    <td><small>
        无
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_DEBUG=all<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_DEBUG=warn
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DEVICE_SELECT</i>
    </small></td>
    <td><small>
        允许用户强制将特定设备在 <i>vkGetPhysicalDevices<i> 与 <i>vkGetPhysicalDeviceGroups<i> 的返回顺序中优先于其他所有设备。<br/>
        该值应为 "&lt;hex vendor id&gt;:&lt;hex device id&gt;"。<br/>
        <b>注意：</b>这**不会**将设备从列表中移除，只会重新排序。
    </small></td>
    <td><small>
        <b>仅 Linux</b>
    </small></td>
    <td><small>
        set VK_LOADER_DEVICE_SELECT=0x10de:0x1f91
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DISABLE_SELECT</i>
    </small></td>
    <td><small>
        允许用户禁用 loader 在将物理设备集合返回给 layer 之前执行的一致性排序算法。<br/>
    </small></td>
    <td><small>
        <b>仅 Linux</b>
    </small></td>
    <td><small>
        set VK_LOADER_DISABLE_SELECT=1
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DISABLE_INST_EXT_FILTER</i>
    </small></td>
    <td><small>
        禁用对 loader 不认识的 instance 扩展的过滤。
        这将允许应用启用由驱动暴露、但 loader 本身不支持的 instance 扩展。<br/>
    </small></td>
    <td><small>
        <b>请谨慎使用！</b> 这可能导致 loader 或应用崩溃。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_DISABLE_INST_EXT_FILTER=1<br/><br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_DISABLE_INST_EXT_FILTER=1
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DRIVERS_SELECT</i>
    </small></td>
    <td><small>
        在已知驱动中按逗号分隔 glob 列表进行搜索，仅选择清单文件名匹配任一 glob 的驱动。<br/>
        由于驱动不像 layer 有名称，该 glob 用于与清单文件名比较。
        “已知驱动清单”指 loader 在考虑默认搜索路径与其他环境变量（如 <i>VK_ICD_FILENAMES</i> 或 <i>VK_ADD_DRIVER_FILES</i>）后已发现的文件。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.3.234 及更高版本构建的 Loader 中可用。<br/>
        若没有任何驱动清单文件名匹配所提供 glob，则不会启用任何驱动，且<b>可能</b>导致 Vulkan 应用无法正常运行。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_DRIVERS_SELECT=nvidia*<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_DRIVERS_SELECT=nvidia*<br/><br/>
        上述示例将只选择 Nvidia 驱动（前提是系统存在且 loader 可见）。
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DRIVERS_DISABLE</i>
    </small></td>
    <td><small>
        在已知驱动中按逗号分隔 glob 列表进行搜索，仅禁用清单文件名匹配任一 glob 的驱动。<br/>
        由于驱动不像 layer 有名称，该 glob 用于与清单文件名比较。
        “已知驱动清单”指 loader 在考虑默认搜索路径与其他环境变量（如 <i>VK_ICD_FILENAMES</i> 或 <i>VK_ADD_DRIVER_FILES</i>）后已发现的文件。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.3.234 及更高版本构建的 Loader 中可用。<br/>
        若使用该环境变量禁用了全部可用驱动，loader 将找不到任何驱动，并<b>会</b>导致 Vulkan 应用无法正常运行。<br/>
        该变量也会在其他驱动环境变量（如 <i>VK_LOADER_DRIVERS_SELECT</i>）之前检查，便于用户先禁用所有驱动，再用 enable 环境变量有选择地重新启用特定驱动。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_DRIVERS_DISABLE=*amd*,*intel*<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_DRIVERS_DISABLE=*amd*,*intel*<br/><br/>
        上述示例将禁用 Intel 与 AMD 驱动（前提是二者均存在且 loader 可见）。
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_LAYERS_ENABLE</i>
    </small></td>
    <td><small>
        在已知 layer 中按逗号分隔 glob 列表进行搜索，仅选择 layer 名称匹配任一 glob 的 layer。<br/>
        “已知 layer”指 loader 在考虑默认搜索路径与其他环境变量（如 <i>VK_LAYER_PATH</i>）后已发现的 layer。
        </i>
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.3.234 及更高版本构建的 Loader 中可用。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_LAYERS_ENABLE=*validation,*recon*<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_LAYERS_ENABLE=*validation,*recon*<br/><br/>
        上述示例将启用 Khronos validation layer 与 GfxReconstruct layer（前提是二者均存在且 loader 可见）。
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_LAYERS_DISABLE</i>
    </small></td>
    <td><small>
        在已知 layer 中按逗号分隔 glob 列表进行搜索，仅禁用 layer 名称匹配任一 glob 的 layer。<br/>
        “已知 layer”指 loader 在考虑默认搜索路径与其他环境变量（如 <i>VK_LAYER_PATH</i>）后已发现的 layer。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.3.234 及更高版本构建的 Loader 中可用。<br/>
        禁用应用有意启用的显式 layer <b>可能</b>导致应用无法正常工作。<br/>
        该变量也会在其他 layer 环境变量（如 <i>VK_LOADER_LAYERS_ENABLE</i>）之前检查，便于用户先禁用所有 layer，再用 enable 环境变量有选择地重新启用特定 layer。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_LAYERS_DISABLE=*MESA*,~implicit~<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_LAYERS_DISABLE=*MESA*,~implicit~<br/><br/>
        上述示例会禁用任何 Mesa layer，以及系统中原本会启用的所有其他隐式 layer。
    </small></td>
  </tr>
  <tr>
  <td><small>
    <i>VK_LOADER_LAYERS_ALLOW</i>
    </small></td>
    <td><small>
        在已知 layer 中按逗号分隔 glob 列表进行搜索，防止 layer 名称匹配任一 glob 的 layer 被 <i>VK_LOADER_LAYERS_DISABLE</i> 禁用。<br/>
        “已知 layer”指 loader 在考虑默认搜索路径与其他环境变量（如 <i>VK_LAYER_PATH</i>）后已发现的 layer。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.3.262 及更高版本构建的 Loader 中可用。<br/>
        如果正常启用机制未启用某 layer，该变量本身不会导致其被启用。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_LAYERS_ALLOW=*validation*,*recon*<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_LAYERS_ALLOW=*validation*,*recon*<br/><br/>
        上述示例将允许名称匹配 validation 或 recon 的 layer 在 `VK_LOADER_LAYERS_DISABLE` 存在时仍可被启用。
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DISABLE_DYNAMIC_LIBRARY_UNLOADING</i>
    </small></td>
    <td><small>
        若设为 "1"，会使 loader 在 vkDestroyInstance 期间不卸载动态库。
        该选项可让泄漏检测器获得完整调用栈。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.3.259 及更高版本构建的 Loader 中可用。<br/>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_DISABLE_DYNAMIC_LIBRARY_UNLOADING=1<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_DISABLE_DYNAMIC_LIBRARY_UNLOADING=1<br/><br/>
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DEVICE_ID_FILTER</i>
    </small></td>
    <td><small>
        若设置该变量，loader 仅枚举匹配过滤条件的物理设备。
        过滤器是逗号分隔的 device id 或 device id 范围列表；id 可用十进制或十六进制表示，范围使用冒号分隔。
        若设备的 id（由 <i>VkPhysicalDeviceProperties::deviceID<i> 提供）匹配任一过滤项，则该设备通过。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.4.326 及更高版本构建的 Loader 中可用。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_DEVICE_ID_FILTER=0x7460:0x747e,29827<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_DEVICE_ID_FILTER=0x7460:0x747e,29827<br/><br/>
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_VENDOR_ID_FILTER</i>
    </small></td>
    <td><small>
        若设置该变量，loader 仅枚举匹配过滤条件的物理设备。
        过滤器是逗号分隔的 vendor id 或 vendor id 范围列表；id 可用十进制或十六进制表示，范围使用冒号分隔。
        若设备的 vendor id（由 <i>VkPhysicalDeviceProperties::vendorID<i> 提供）匹配任一过滤项，则该设备通过。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.4.326 及更高版本构建的 Loader 中可用。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_VENDOR_ID_FILTER=65541<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_VENDOR_ID_FILTER=65541<br/><br/>
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_LOADER_DRIVER_ID_FILTER</i>
    </small></td>
    <td><small>
        若设置该变量，loader 仅枚举匹配过滤条件的物理设备。
        过滤器是逗号分隔的 driver id 或 driver id 范围列表；id 可用十进制或十六进制表示，范围使用冒号分隔。
        若设备的 driver id（由 <i>VkPhysicalDeviceDriverProperties::driverID<i> 提供）匹配任一过滤项，则该设备通过。
    </small></td>
    <td><small>
        该功能仅在基于 Vulkan 头文件 1.4.326 及更高版本构建的 Loader 中可用。
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_LOADER_DRIVER_ID_FILTER=1-3:13<br/>
        <br/>
        set<br/>
        &nbsp;&nbsp;VK_LOADER_DRIVER_ID_FILTER=1-3:13<br/><br/>
    </small></td>
  </tr>
</table>

<br/>

<a id="deprecated-environment-variables"></a>
### 已弃用环境变量

这些环境变量目前仍可用且受支持，但未来的 loader 版本可能移除支持。

<table style="width:100%">
  <tr>
    <th>环境变量</th>
    <th>行为</th>
    <th>被替代为</th>
    <th>限制</th>
    <th>示例格式</th>
  </tr>
  <tr>
    <td><small><i>VK_ICD_FILENAMES</i></small></td>
    <td><small>
            强制 loader 使用指定驱动 JSON 文件。
            该值包含以分隔符分开的驱动 JSON 清单文件完整路径列表。<br/>
            <br/>
            <b>注意：</b>若未使用 JSON 文件的全局路径，可能会遇到问题。<br/>
    </small></td>
    <td><small>
        已被 <i>VK_DRIVER_FILES</i> 替代。
    </small></td>
    <td><small>
        <a href="#elevated-privilege-caveats">
            以提升权限运行 Vulkan 应用时会被忽略。
        </a>
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_ICD_FILENAMES=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<folder_a>/intel.json:<folder_b>/amd.json
        <br/><br/>
        set<br/>
        &nbsp;&nbsp;VK_ICD_FILENAMES=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<folder_a>\nvidia.json;<folder_b>\mesa.json
    </small></td>
  </tr>
  <tr>
    <td><small>
        <i>VK_INSTANCE_LAYERS</i>
    </small></td>
    <td><small>
        强制 loader 将给定 layer 添加到通常传入 <b>vkCreateInstance</b> 的已启用 layer 列表。
        这些 layer 会优先添加，且 loader 会移除本列表与 <i>ppEnabledLayerNames</i> 中重复出现的 layer。
    </small></td>
    <td><small>
        它会覆盖通过 <i>VK_LOADER_LAYERS_DISABLE</i> 禁用的 layer。
    </small></td>
    <td><small>
        无
    </small></td>
    <td><small>
        export<br/>
        &nbsp;&nbsp;VK_INSTANCE_LAYERS=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;layer_a&gt;;&lt;layer_b&gt;<br/><br/>
        set<br/>
        &nbsp;&nbsp;VK_INSTANCE_LAYERS=<br/>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;layer_a&gt;;&lt;layer_b&gt;
    </small></td>
  </tr>
</table>
<br/>
<br/>

<a id="glossary-of-terms"></a>
## 术语表

<table style="width:100%">
  <tr>
    <th>字段名称</th>
    <th>字段值</th>
  </tr>
  <tr>
    <td>Android Loader</td>
    <td>主要为 Android OS 设计的 loader。
        它来自与 Khronos loader 不同的代码库。
        但在所有重要方面，两者应在功能上等价。
    </td>
  </tr>
  <tr>
    <td>Khronos Loader</td>
    <td>由 Khronos 发布的 loader，目前主要面向 Windows、Linux、macOS、Stadia 与 Fuchsia。
        它来自与 Android loader 不同的
        <a href="https://github.com/KhronosGroup/Vulkan-Loader">代码库</a>。
        但在所有重要方面，两者应在功能上等价。
    </td>
  </tr>
  <tr>
    <td>Core Function</td>
    <td>已属于 Vulkan 核心规范、而非扩展的一类函数。 <br/>
        例如：<b>vkCreateDevice()</b>。
    </td>
  </tr>
  <tr>
    <td>Device Call Chain</td>
    <td>设备函数所遵循的函数调用链。
        设备函数的调用链通常如下：应用先调用 loader trampoline，随后 loader trampoline 调用已启用 layer，最后一个 layer 再调用该设备对应驱动。 <br/>
        更多信息参见
        <a href="#dispatch-tables-and-call-chains">Dispatch Tables and Call
        Chains</a> 章节。
    </td>
  </tr>
  <tr>
    <td>Device Function</td>
    <td>设备函数是指首个参数为 <i>VkDevice</i>、<i>VkQueue</i>、<i>VkCommandBuffer</i> 或其任一子对象的 Vulkan 函数。 <br/><br/>
        一些 Vulkan 设备函数包括： <br/>
        &nbsp;&nbsp;<b>vkQueueSubmit</b>, <br/>
        &nbsp;&nbsp;<b>vkBeginCommandBuffer</b>, <br/>
        &nbsp;&nbsp;<b>vkCreateEvent</b>。 <br/><br/>
        更多信息参见 <a href="#instance-versus-device">Instance Versus Device</a> 章节。
    </td>
  </tr>
  <tr>
    <td>Discovery</td>
    <td>loader 搜索驱动和 layer 文件、以建立内部可用 Vulkan 对象列表的过程。<br/>
        在 <i>Windows/Linux/macOS</i> 上，发现过程通常聚焦于搜索 Manifest 文件。<br/>
        在 <i>Android</i> 上，该过程聚焦于搜索库文件。
    </td>
  </tr>
  <tr>
    <td>Dispatch Table</td>
    <td>一个函数指针数组（包含核心函数及可能的扩展函数），用于步进到调用链中的下一个实体。
        该实体可能是 loader、layer 或驱动。<br/>
        更多信息参见 <a href="#dispatch-tables-and-call-chains">Dispatch Tables and Call Chains</a>。
    </td>
  </tr>
  <tr>
    <td>Driver</td>
    <td>为 Vulkan API 提供支持的底层库。
        这种支持可实现为 ICD、API 翻译库或纯软件。<br/>
        更多信息参见 <a href="#drivers">Drivers</a> 章节。
    </td>
  </tr>
  <tr>
    <td>Extension</td>
    <td>Vulkan 中用于扩展核心功能的概念。
        扩展可以是 IHV 特定、平台特定或更广泛可用。 <br/>
        在使用扩展前，应先查询扩展是否存在，并在 <b>vkCreateInstance</b>（若为 instance 扩展）或 <b>vkCreateDevice</b>（若为 device 扩展）期间启用它。 <br/>
        扩展关联的每个结构体、枚举项、命令入口点或宏定义都会带有作者前缀或后缀修饰。
        例如，`KHR` 是 Khronos 编写扩展的前缀，也会出现在与这些扩展相关的结构体、枚举项和命令中。
    </td>
  </tr>
  <tr>
    <td>Extension Function</td>
    <td>定义在扩展中、而非 Vulkan 核心规范中的函数。 <br/>
        与其所属扩展一样，该函数会有后缀修饰以指示扩展作者。<br/>
        一些扩展后缀示例：<br/>
        &nbsp;&nbsp;<b>KHR</b>  - Khronos 编写扩展， <br/>
        &nbsp;&nbsp;<b>EXT</b>  - 多公司共同编写扩展， <br/>
        &nbsp;&nbsp;<b>AMD</b>  - AMD 编写扩展， <br/>
        &nbsp;&nbsp;<b>ARM</b>  - ARM 编写扩展， <br/>
        &nbsp;&nbsp;<b>NV</b>   - Nvidia 编写扩展。<br/>
    </td>
  </tr>
  <tr>
    <td>ICD</td>
    <td>“Installable Client Driver”的缩写。
        这类驱动由 IHV 提供，用于与其提供的硬件交互。 <br/>
        这是最常见的 Vulkan 驱动类型。 <br/>
        更多信息参见 <a href="#installable-client-drivers">Installable Client Drivers</a> 章节。
    </td>
  </tr>
  <tr>
    <td>IHV</td>
    <td>“Independent Hardware Vendor（独立硬件供应商）”的缩写。
        通常指构建底层硬件技术并被使用的公司。 <br/>
        图形 IHV 的典型示例包括（但不限于）：AMD、ARM、Imagination、Intel、Nvidia、Qualcomm
    </td>
  </tr>
  <tr>
    <td>Instance Call Chain</td>
    <td>instance 函数所遵循的函数调用链。
        instance 函数的调用链通常如下：应用先调用 loader trampoline，随后 loader trampoline 调用已启用 layer，最后一个 layer 调用 loader terminator，最后 loader terminator 调用所有可用驱动。 <br/>
        更多信息参见 <a href="#dispatch-tables-and-call-chains">Dispatch Tables and Call Chains</a> 章节。
    </td>
  </tr>
  <tr>
    <td>Instance Function</td>
    <td>instance 函数是指首个参数为 <i>VkInstance</i>、<i>VkPhysicalDevice</i> 或无参数对象的 Vulkan 函数。 <br/><br/>
        一些 Vulkan instance 函数包括：<br/>
        &nbsp;&nbsp;<b>vkEnumerateInstanceExtensionProperties</b>, <br/>
        &nbsp;&nbsp;<b>vkEnumeratePhysicalDevices</b>, <br/>
        &nbsp;&nbsp;<b>vkCreateInstance</b>, <br/>
        &nbsp;&nbsp;<b>vkDestroyInstance</b>。 <br/><br/>
        更多信息参见 <a href="#instance-versus-device">Instance Versus Device</a> 章节。
    </td>
  </tr>
  <tr>
    <td>Layer</td>
    <td>Layer 是用于增强 Vulkan 系统的可选组件。
        它们可以在函数从应用传递到驱动的过程中拦截、评估并修改现有 Vulkan 函数。<br/>
        更多信息参见 <a href="#layers">Layers</a> 章节。
    </td>
  </tr>
  <tr>
    <td>Layer Library</td>
    <td><b>Layer Library</b> 是 loader 能够发现的全部 layer 的集合。
        其中可包含隐式 layer 和显式 layer。
        除非以某种方式被禁用，否则这些 layer 可供应用使用。
        更多信息参见
        <a href="LoaderLayerInterface.md#layer-layer-discovery">Layer Discovery
        </a>。
    </td>
  </tr>
  <tr>
    <td>Loader</td>
    <td>作为 Vulkan 应用、Vulkan layer 与 Vulkan 驱动之间中介的中间件程序。<br/>
        更多信息参见 <a href="#the-loader">The Loader</a> 章节。
    </td>
  </tr>
  <tr>
    <td>Manifest Files</td>
    <td>Khronos loader 使用的 JSON 格式数据文件。
        这些文件包含 layer 或 driver 的特定信息，例如文件位置与默认设置。
        参见
        <a href="LoaderLayerInterface.md#layer-manifest-file-format">Layer</a>
        或
        <a href="LoaderDriverInterface.md#driver-manifest-file-format">Driver</a>
        清单格式。
    </td>
  </tr>
  <tr>
    <td>Terminator Function</td>
    <td>由 loader 拥有、位于 driver 之上的 instance 调用链末端函数。
        该函数在 instance 调用链中是必需的，因为所有 instance 功能都必须传递给所有能够接收该调用的驱动。 <br/>
        更多信息参见 <a href="#dispatch-tables-and-call-chains">Dispatch Tables and Call Chains</a>。
    </td>
  </tr>
  <tr>
    <td>Trampoline Function</td>
    <td>由 loader 拥有、位于 instance 或 device 调用链首端的函数，负责基于合适调度表进行设置与正确的调用链遍历。
        对于 device 函数（device 调用链中），该函数在某些情况下实际上可被跳过。<br/>
        更多信息参见 <a href="#dispatch-tables-and-call-chains">Dispatch Tables and Call Chains</a>。
    </td>
  </tr>
  <tr>
    <td>WSI Extension</td>
    <td>Windowing System Integration 的缩写。
        针对特定窗口系统的 Vulkan 扩展，旨在作为窗口系统与 Vulkan 之间的接口。<br/>
        更多信息参见
        <a href="LoaderApplicationInterface.md#wsi-extensions">WSI Extensions</a>。
    </td>
  </tr>
  <tr>
    <td>Exported Function</td>
    <td>计划通过平台特定动态链接器获取的函数，特指来自 Driver 或 Layer 库的函数。
        需要导出的函数主要是 Loader 对 Layer 或 Driver 库调用的最初一批函数。 <br/>
    </td>
  </tr>
  <tr>
    <td>Exposed Function</td>
    <td>计划通过 Querying Function（例如 `vkGetInstanceProcAddr`）获取的函数。
        某个 exposed function 所需的具体 Querying Function 会因 Layer/Driver 类型及接口版本不同而变化。 <br/>
    </td>
  </tr>
  <tr>
    <td>Querying Functions</td>
    <td>允许 Loader 从驱动和 layer 查询其他函数的一组函数。
        这些函数可能属于 Vulkan API，也可能来自 Loader 与 Driver 的私有接口，或 Loader 与 Layer 接口。 <br/>
        这些函数包括：
        `vkGetInstanceProcAddr`, `vkGetDeviceProcAddr`,
        `vk_icdGetInstanceProcAddr`, `vk_icdGetPhysicalDeviceProcAddr`, 以及
        `vk_layerGetPhysicalDeviceProcAddr`。
    </td>
  </tr>
</table>
