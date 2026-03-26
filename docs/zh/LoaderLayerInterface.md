<!-- markdownlint-disable MD041 -->
[![Khronos Vulkan][1]][2]

[1]: https://vulkan.lunarg.com/img/Vulkan_100px_Dec16.png "https://www.khronos.org/vulkan/"
[2]: https://www.khronos.org/vulkan/

# Layer 与 Loader 的接口（Layer Interface to the Loader） <!-- omit from toc -->
[![Creative Commons][3]][4]

<!-- Copyright &copy; 2015-2023 LunarG, Inc. -->

[3]: https://i.creativecommons.org/l/by-nd/4.0/88x31.png "Creative Commons License"
[4]: https://creativecommons.org/licenses/by-nd/4.0/

## 目录 <!-- omit from toc -->

- [概述](#overview)
- [Layer 发现](#layer-discovery)
  - [Layer 清单文件的用法](#layer-manifest-file-usage)
  - [Android 上的 Layer 发现](#android-layer-discovery)
  - [Windows 上的 Layer 发现](#windows-layer-discovery)
  - [Linux 上的 Layer 发现](#linux-layer-discovery)
    - [Linux 显式层搜索路径示例](#example-linux-explicit-layer-search-path)
  - [Fuchsia 上的 Layer 发现](#fuchsia-layer-discovery)
  - [macOS 上的 Layer 发现](#macos-layer-discovery)
    - [macOS 隐式层搜索路径示例](#example-macos-implicit-layer-search-path)
  - [Layer 过滤](#layer-filtering)
    - [Layer 启用过滤](#layer-enable-filtering)
    - [Layer 禁用过滤](#layer-disable-filtering)
    - [Layer 特殊禁用情况](#layer-special-case-disable)
    - [Layer 禁用警告](#layer-disable-warning)
    - [允许某些 Layer 忽略 Layer 禁用](#allow-certain-layers-to-ignore-layer-disabling)
      - [`VK_INSTANCE_LAYERS`](#vk_instance_layers)
  - [提升权限时的例外情况](#exception-for-elevated-privileges)
- [Layer 版本协商](#layer-version-negotiation)
- [Layer 调用链与分布式分发](#layer-call-chains-and-distributed-dispatch)
- [Layer 未知物理设备扩展](#layer-unknown-physical-device-extensions)
  - [添加`vk_layerGetPhysicalDeviceProcAddr`的原因](#reason-for-adding-vk_layergetphysicaldeviceprocaddr)
- [Layer 拦截要求](#layer-intercept-requirements)
- [分布式分发要求](#distributed-dispatching-requirements)
- [Layer 约定与规则](#layer-conventions-and-rules)
- [Layer 调度初始化](#layer-dispatch-initialization)
- [`CreateInstance`示例代码](#example-code-for-createinstance)
- [`CreateDevice`示例代码](#example-code-for-createdevice)
- [元层](#meta-layers)
  - [覆盖元层](#override-meta-layer)
- [预实例函数](#pre-instance-functions)
- [特殊注意事项](#special-considerations)
  - [在 Layer 内将私有数据与 Vulkan 对象关联](#associating-private-data-with-vulkan-objects-within-a-layer)
    - [包装](#wrapping)
    - [关于包装的注意事项](#cautions-about-wrapping)
    - [哈希表](#hash-maps)
  - [创建新的可分发对象](#creating-new-dispatchable-objects)
  - [版本控制与激活的交互](#versioning-and-activation-interactions)
- [Layer 清单文件格式](#layer-manifest-file-format)
  - [Layer 清单文件版本历史](#layer-manifest-file-version-history)
  - [Layer 清单文件版本 1.2.1](#layer-manifest-file-version-121)
    - [Layer 清单文件版本 1.2.0](#layer-manifest-file-version-120)
    - [Layer 清单文件版本 1.1.2](#layer-manifest-file-version-112)
    - [Layer 清单文件版本 1.1.1](#layer-manifest-file-version-111)
    - [Layer 清单文件版本 1.1.0](#layer-manifest-file-version-110)
    - [Layer 清单文件版本 1.0.1](#layer-manifest-file-version-101)
    - [Layer 清单文件版本 1.0.0](#layer-manifest-file-version-100)
- [Layer 接口版本](#layer-interface-versions)
  - [Layer 接口版本 2](#layer-interface-version-2)
  - [Layer 接口版本 1](#layer-interface-version-1)
  - [Layer 接口版本 0](#layer-interface-version-0)
- [Loader 与 Layer 接口策略](#loader-and-layer-interface-policy)
  - [编号格式](#number-format)
  - [Android 差异](#android-differences)
  - [行为良好的 Layer 的要求](#requirements-of-well-behaved-layers)
  - [行为良好的 Loader 的要求](#requirements-of-a-well-behaved-loader)

<a id="overview"></a>
## 概述

本文从以 Layer 为中心的视角介绍如何与 Vulkan loader 配合工作。有关 loader 各个部分的完整概览，请参阅[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文件。

<a id="layer-discovery"></a>
## Layer 发现

如[LoaderApplicationInterface.md](LoaderApplicationInterface.md) 文档中[隐式层与显式层](LoaderApplicationInterface.md#implicit-vs-explicit-layers) 一节所述，layer 可分为两类：
  * 隐式层
  * 显式层

两者的主要区别在于，隐式层默认会自动启用，除非被覆盖；而显式层则必须显式启用。请注意，并非所有操作系统都支持隐式层（例如 Android）。

在任何系统上，loader 都会在特定位置查找其可按用户请求加载的 layer 相关信息。查找系统中可用 layer 的过程称为 Layer 发现（Layer Discovery）。在发现过程中，loader 会确定有哪些 layer 可用、layer 名称、 layer 版本以及该 layer 支持的扩展。这些信息会通过`vkEnumerateInstanceLayerProperties`返回给应用程序。

loader 可用的整组 layer 被称为`Layer Library`。本节定义了一种可扩展接口，用于发现`Layer Library`中包含哪些 layer。

本节还规定了 layer 必须遵循的最低限度约定与规则，尤其是 layer 与 loader 及其他 layer 交互时应遵守的要求。

当查找某个 layer 时，loader 会按照检测到它们的顺序遍历`Layer Library`，并在名称匹配时加载对应的 layer。如果同一个库在用户系统的不同位置存在多个实例，则使用搜索顺序中最先出现的那个。每个操作系统都有自己的搜索顺序，其定义见下文对应的 Layer 发现小节。如果同一目录中的多个清单文件定义了同一个 layer，但指向不同的库文件，则 layer 的加载顺序会因为[`readdir`的行为而是随机的](https://www.ibm.com/support/pages/order-directory-contents-returned-calls-readdir)。

此外，在调用`vkCreateInstance`或`vkCreateDevice`时，无论是在组件 layer 列表中，还是在所有已启用 layer 的全局范围内，任何重复的 layer 名称都会被 loader 直接忽略。同一个 layer 名称只会使用第一次出现的那个。

<a id="layer-manifest-file-usage"></a>
### Layer 清单文件的用法

在 Windows、Linux 和 macOS 系统上，使用 JSON 格式的清单文件来存储 layer 信息。为了查找系统中已安装的 layer，Vulkan loader 会读取这些 JSON 文件，以识别 layer 及其扩展的名称和属性。使用清单文件可以让 loader 在应用既不查询也不请求任何扩展时，避免加载任何共享库文件。[Layer 清单文件](#layer-manifest-file-format) 的格式详见下文。

Android loader 不使用清单文件。相反，loader 会通过称为 “introspection” 函数的特殊函数来查询 layer 属性。这些函数的目的，是获取与读取清单文件时相同的必需信息。 Khronos loader 本身不会使用这些 introspection 函数，但为了保持一致性，layer 仍应提供它们。具体有哪些 introspection 函数，已在[Layer 清单文件格式](#layer-manifest-file-format) 表格中列出。

<a id="android-layer-discovery"></a>
### Android 上的 Layer 发现

在 Android 上，loader 会在`/data/local/debug/vulkan`文件夹中查找可枚举的 layer。

启用了调试的应用能够枚举并启用该位置中的任何 layer。

<a id="windows-layer-discovery"></a>
### Windows 上的 Layer 发现

为了查找系统中已安装的 layer，Vulkan loader 会扫描以下 Windows 注册表项中的值：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ExplicitLayers
HKEY_CURRENT_USER\SOFTWARE\Khronos\Vulkan\ExplicitLayers
HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers
HKEY_CURRENT_USER\SOFTWARE\Khronos\Vulkan\ImplicitLayers
```

但在 64 位 Windows 上运行 32 位应用时， loader 会改为扫描 32 位注册表位置：

```
HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Khronos\Vulkan\ExplicitLayers
HKEY_CURRENT_USER\SOFTWARE\WOW6432Node\Khronos\Vulkan\ExplicitLayers
HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Khronos\Vulkan\ImplicitLayers
HKEY_CURRENT_USER\SOFTWARE\WOW6432Node\Khronos\Vulkan\ImplicitLayers
```

对于这些注册表项中的每个值，只要其 DWORD 数据为 0， loader 就会打开该值名称指定的 JSON 清单文件。每个名称都必须是清单文件的绝对路径。此外，只有当应用不是以管理员权限执行时，才会搜索`HKEY_CURRENT_USER`位置。这样做是为了确保具备管理员权限的应用不会运行那些安装时并不需要管理员权限的 layer。

由于某些 layer 会与驱动一同安装，loader 还会扫描与显示适配器相关、以及与这些适配器关联的所有软件组件专用的注册表项，以获取 JSON 清单文件的位置。这些注册表项位于驱动安装时创建的设备键中，包含基础设置的配置信息，包括 Vulkan、OpenGL 和 Direct3D 的 ICD 位置。

设备适配器和软件组件的注册表路径应通过 PnP Configuration Manager API 获取。`000X`键是一个编号键，每个设备都会分配不同的编号。

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanExplicitLayers
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanImplicitLayers
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanExplicitLayers
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanImplicitLayers
```

此外，在 64 位系统上还可能存在另一组注册表值，如下所示。这些值会以与 Windows-on-Windows 功能相同的方式，记录 64 位操作系统上的 32 位 layer 位置。

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanExplicitLayersWow
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanImplicitLayersWow
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanExplicitLayersWow
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Software Component GUID}\000X\VulkanImplicitLayersWow
```

如果上述任意值存在，且其类型为`REG_SZ`， loader 就会打开该键值指定的 JSON 清单文件。每个值都必须是 JSON 清单文件的绝对路径。某个键值也可以是`REG_MULTI_SZ`类型，此时该值会被解释为一个 JSON 清单文件路径列表。

一般来说，应用应将 layer 安装到`SOFTWARE\Khronos\Vulkan`路径中。 PnP 注册表位置专门用于作为驱动安装一部分分发的 layer。应用安装程序不应修改设备专用注册表，而设备驱动也不应修改系统注册表。

此外，Vulkan loader 还会扫描系统中已知的 Windows AppX/MSIX 包。如果发现此类包，loader 会扫描该已安装包的根目录以查找 JSON 清单文件。目前唯一已知的包是 Microsoft 的[OpenCL™, OpenGL®, and Vulkan® Compatibility Pack](https://apps.microsoft.com/store/detail/9NQPSL29BFFF?hl=en-us&gl=US)。

Vulkan loader 会打开每个清单文件以获取 layer 信息，包括共享库（`.dll`）文件的名称或路径名。

如果定义了`VK_LAYER_PATH`，loader 就会在该变量指定的路径中查找显式 layer 清单文件，而不是使用显式 layer 注册表项中提供的信息。

如果定义了`VK_ADD_LAYER_PATH`，loader 则会在使用显式 layer 注册表项中信息的基础上，额外在所提供的路径中查找显式 layer 清单文件。`VK_ADD_LAYER_PATH`提供的路径会被加到标准搜索文件夹列表之前，因此会优先被搜索。

如果存在`VK_LAYER_PATH`， loader 将不会使用`VK_ADD_LAYER_PATH`，其任何值都会被忽略。

如果定义了`VK_IMPLICIT_LAYER_PATH`，loader 就会在该变量定义的路径中查找隐式层清单文件，而不是使用隐式层注册表项中提供的信息。

如果定义了`VK_ADD_IMPLICIT_LAYER_PATH`， loader 则会在使用隐式层注册表项中信息的基础上，额外在所提供的路径中查找隐式层清单文件。`VK_ADD_IMPLICIT_LAYER_PATH`提供的路径会被加到标准搜索文件夹列表之前，因此会优先被搜索。

出于安全原因，如果以提升权限运行，则会忽略`VK_LAYER_PATH`、`VK_ADD_LAYER_PATH`、`VK_IMPLICIT_LAYER_PATH`和`VK_ADD_IMPLICIT_LAYER_PATH`。更多信息请参阅[提升权限时的例外情况](#exception-for-elevated-privileges)。

有关这方面的更多信息，请参阅[LoaderApplicationInterface.md](LoaderApplicationInterface.md) 文档中的[强制指定 Layer 源文件夹](LoaderApplicationInterface.md#forcing-layer-source-folders)。

<a id="linux-layer-discovery"></a>
### Linux 上的 Layer 发现

在 Linux 上，Vulkan loader 会使用环境变量来扫描清单文件；如果相应环境变量未定义，则使用对应的回退值：

<table style="width:100%">
  <tr>
    <th>搜索顺序</th>
    <th>目录/环境变量</th>
    <th>回退值</th>
    <th>附加说明</th>
  </tr>
  <tr>
    <td>1</td>
    <td>$XDG_CONFIG_HOME</td>
    <td>$HOME/.config</td>
    <td><b>在以 setuid、setgid 或文件系统 capability 等提升权限方式运行时，
           会忽略该路径</b>。<br/>
        这是因为在这些场景下，无法安全地信任这些环境变量一定不是恶意的。
    </td>
  </tr>
  <tr>
    <td>1</td>
    <td>$XDG_CONFIG_DIRS</td>
    <td>/etc/xdg</td>
    <td></td>
  </tr>
  <tr>
    <td>2</td>
    <td>SYSCONFDIR</td>
    <td>/etc</td>
    <td>编译时选项，设置为非 Linux 发行版提供的软件包所安装 layer 的可能位置。
    </td>
  </tr>
  <tr>
    <td>3</td>
    <td>EXTRASYSCONFDIR</td>
    <td>/etc</td>
    <td>编译时选项，设置为非 Linux 发行版提供的软件包所安装 layer 的可能位置。
        通常仅在 SYSCONFDIR 被设置为不同于 /etc 的值时才会设置。
    </td>
  </tr>
  <tr>
    <td>4</td>
    <td>$XDG_DATA_HOME</td>
    <td>$HOME/.local/share</td>
    <td><b>在以 setuid、setgid 或文件系统 capability 等提升权限方式运行时，
           会忽略该路径</b>。<br/>
        这是因为在这些场景下，无法安全地信任这些环境变量一定不是恶意的。
    </td>
  </tr>
  <tr>
    <td>5</td>
    <td>$XDG_DATA_DIRS</td>
    <td>/usr/local/share/:/usr/share/</td>
    <td></td>
  </tr>
</table>

这些目录列表会使用平台标准路径分隔符（`:`）拼接起来。随后，loader 会依次选择每个路径，并根据所查找的 layer 类型附加特定后缀，然后在相应文件夹中查找清单文件：

  * 隐式层：后缀 = `/vulkan/implicit_layer.d`
  * 显式层：后缀 = `/vulkan/explicit_layer.d`

如果定义了`VK_LAYER_PATH`，loader 就会在该变量指定的路径中查找显式 layer 清单文件，而不是使用上面提到的标准显式 layer 路径信息。

如果定义了`VK_ADD_LAYER_PATH`，loader 则会在使用上面提到的标准显式 layer 路径信息的基础上，额外在所提供的路径中查找显式 layer 清单文件。`VK_ADD_LAYER_PATH`提供的路径会被加到标准搜索文件夹列表之前，因此会优先被搜索。

如果存在`VK_LAYER_PATH`， loader 将不会使用`VK_ADD_LAYER_PATH`，其任何值都会被忽略。

如果定义了`VK_IMPLICIT_LAYER_PATH`，loader 就会在该变量定义的路径中查找隐式层清单文件，而不是使用上面提到的标准隐式层路径信息。

如果定义了`VK_ADD_IMPLICIT_LAYER_PATH`， loader 则会在使用上面提到的标准隐式层路径信息的基础上，额外在所提供的路径中查找隐式层清单文件。`VK_ADD_IMPLICIT_LAYER_PATH`提供的路径会被加到标准搜索文件夹列表之前，因此会优先被搜索。

如果存在`VK_IMPLICIT_LAYER_PATH`， loader 将不会使用`VK_ADD_IMPLICIT_LAYER_PATH`，其任何值都会被忽略。

出于安全原因，如果以提升权限运行，则会忽略`VK_LAYER_PATH`、`VK_ADD_LAYER_PATH`、`VK_IMPLICIT_LAYER_PATH`和`VK_ADD_IMPLICIT_LAYER_PATH`。更多信息请参阅[提升权限时的例外情况](#exception-for-elevated-privileges)。

**注意：** 虽然搜索清单文件的文件夹顺序有明确规定，但 loader 在每个目录中读取内容的顺序会因为[`readdir`的行为而是随机的](https://www.ibm.com/support/pages/order-directory-contents-returned-calls-readdir)。

有关这方面的更多信息，请参阅[LoaderApplicationInterface.md](LoaderApplicationInterface.md) 文档中的[强制指定 Layer 源文件夹](LoaderApplicationInterface.md#forcing-layer-source-folders)。

还需要注意的是，虽然`VK_LAYER_PATH`、`VK_ADD_LAYER_PATH`、`VK_IMPLICIT_LAYER_PATH`和`VK_ADD_IMPLICIT_LAYER_PATH`会让 loader 去这些路径中查找清单文件，但这并不保证清单文件中提到的库文件一定能立即被找到。很多时候，layer 清单文件会使用相对路径或绝对路径指向库文件。当使用相对路径或绝对路径时，loader 通常可以在不查询操作系统的情况下找到库文件。但是，如果库仅以名称形式列出，loader 可能找不到它。如果在查找与某个 layer 关联的库文件时出现问题，请尝试更新`LD_LIBRARY_PATH`环境变量，使其指向相应`.so`文件所在的位置。

<a id="example-linux-explicit-layer-search-path"></a>
#### Linux 显式层搜索路径示例

对于一个虚构用户 “me”，layer 清单搜索路径可能如下所示：

```
  /home/me/.config/vulkan/explicit_layer.d
  /etc/xdg/vulkan/explicit_layer.d
  /usr/local/etc/vulkan/explicit_layer.d
  /etc/vulkan/explicit_layer.d
  /home/me/.local/share/vulkan/explicit_layer.d
  /usr/local/share/vulkan/explicit_layer.d
  /usr/share/vulkan/explicit_layer.d
```

<a id="fuchsia-layer-discovery"></a>
### Fuchsia 上的 Layer 发现

在 Fuchsia 上，Vulkan loader 会像[Linux](#linux-layer-discovery) 一样，使用环境变量及其对应的回退值来扫描清单文件，前提是相应环境变量未定义。 **唯一** 的区别是，Fuchsia 不允许对 *$XDG_DATA_DIRS* 或 *$XDG_HOME_DIRS* 使用回退值。

<a id="macos-layer-discovery"></a>
### macOS 上的 Layer 发现

在 macOS 上，Vulkan loader 会使用应用资源文件夹，以及环境变量或其对应的回退值来扫描清单文件，前提是相应环境变量未定义。其顺序与 Linux 上的搜索路径类似，但有一个例外：会先搜索应用 bundle 的资源目录：`(bundle)/Contents/Resources/`。

<a id="example-macos-implicit-layer-search-path"></a>
#### macOS 隐式层搜索路径示例

对于一个虚构用户 “Me”，layer 清单搜索路径可能如下所示：

```
  <bundle>/Contents/Resources/vulkan/implicit_layer.d
  /Users/Me/.config/vulkan/implicit_layer.d
  /etc/xdg/vulkan/implicit_layer.d
  /usr/local/etc/vulkan/implicit_layer.d
  /etc/vulkan/implicit_layer.d
  /Users/Me/.local/share/vulkan/implicit_layer.d
  /usr/local/share/vulkan/implicit_layer.d
  /usr/share/vulkan/implicit_layer.d
```

<a id="layer-filtering"></a>
### Layer 过滤

**注意：** 该功能仅在使用 Vulkan 头文件版本 1.3.234 及更高版本构建的 Loader 中可用。

loader 支持过滤环境变量，可强制启用或禁用已知的 layer。已知 layer 是指 loader 在考虑默认搜索路径以及环境变量`VK_LAYER_PATH`、`VK_ADD_LAYER_PATH`、`VK_IMPLICIT_LAYER_PATH`和`VK_ADD_IMPLICIT_LAYER_PATH`之后已经找到的那些 layer。

这些过滤器会与 layer 清单文件中提供的 layer 名称进行比较。

这些过滤器还必须遵循[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文档中[过滤环境变量行为](LoaderInterfaceArchitecture.md#filter-environment-variable-behaviors) 一节定义的行为。

<a id="layer-enable-filtering"></a>
#### Layer 启用过滤

layer 启用环境变量`VK_LOADER_LAYERS_ENABLE`是一个用逗号分隔的 glob 列表，用于在已知 layer 中进行匹配搜索。 layer 名称会与该环境变量中的 glob 模式进行比较；如果匹配，这些 layer 就会自动被加入 loader 为每个应用维护的已启用 layer 列表中。这些 layer 会在隐式层之后、其他显式层之前启用。

当使用`VK_LOADER_LAYERS_ENABLE`过滤器启用某个 layer 时，如果 loader 日志被设置为输出警告或 layer 消息，则会为每个被强制启用的 layer 输出一条消息。该消息如下所示：

```
[Vulkan Loader] WARNING | LAYER:  Layer "VK_LAYER_LUNARG_wrap_objects" force enabled due to env var 'VK_LOADER_LAYERS_ENABLE'
```

<a id="layer-disable-filtering"></a>
#### Layer 禁用过滤

layer 禁用环境变量`VK_LOADER_LAYERS_DISABLE`是一个用逗号分隔的 glob 列表，用于在已知 layer 中进行匹配搜索。 layer 名称会与该环境变量中的 glob 模式进行比较；如果匹配，它们就会被自动禁用（无论该 layer 是隐式还是显式）。这意味着它们不会被加入 loader 为每个应用维护的已启用 layer 列表中。这也意味着某些由应用请求的 layer 也可能不会被启用，例如`VK_KHRONOS_LAYER_synchronization2`，从而导致某些应用行为异常。

当使用`VK_LOADER_LAYERS_DISABLE`过滤器禁用某个 layer 时，如果 loader 日志被设置为输出警告或 layer 消息，则会为每个被强制禁用的 layer 输出一条消息。该消息如下所示：

```
[Vulkan Loader] WARNING | LAYER:  Layer "VK_LAYER_LUNARG_wrap_objects" disabled because name matches filter of env var 'VK_LOADER_LAYERS_DISABLE'
```

<a id="layer-special-case-disable"></a>
#### Layer 特殊禁用情况

由于 layer 有不同类型，因此在使用`VK_LOADER_LAYERS_DISABLE`环境变量时，还提供了 3 个额外的特殊禁用选项。

它们是：

  * `~all~`
  * `~implicit~`
  * `~explicit~`

`~all~`会有效禁用所有 layer。这使开发者能够禁用系统上的全部 layer。`~implicit~`会有效禁用所有隐式层（但显式层仍会保留在应用调用链中）。`~explicit~`会有效禁用所有显式层（但隐式层仍会保留在应用调用链中）。

<a id="layer-disable-warning"></a>
#### Layer 禁用警告

无论是通过正常使用`VK_LOADER_LAYERS_DISABLE`，还是通过`~all~`或`~explicit~`之类的特殊禁用选项来禁用 layer，如果应用依赖一个或多个显式层提供的功能，都可能导致应用出错。

<a id="allow-certain-layers-to-ignore-layer-disabling"></a>
#### 允许某些 Layer 忽略 Layer 禁用

**注意：** `VK_LOADER_LAYERS_DISABLE`仅在使用 Vulkan 头文件版本 1.3.262 及更高版本构建的 Loader 中可用。

layer 允许环境变量`VK_LOADER_LAYERS_ALLOW`是一个用逗号分隔的 glob 列表，用于在已知 layer 中进行匹配搜索。 layer 名称会与该环境变量中的 glob 模式进行比较；如果匹配，它们就不会被`VK_LOADER_LAYERS_DISABLE`禁用。

隐式层可以设计为仅在设置了 layer 指定的环境变量时才启用，从而支持依赖上下文的启用方式。`VK_LOADER_LAYERS_ENABLE`会忽略这种上下文。`VK_LOADER_LAYERS_ALLOW`的行为与`VK_LOADER_LAYERS_ENABLE`类似，但同时也会尊重通常用于判断某个隐式层是否应启用的上下文条件。

`VK_LOADER_LAYERS_ALLOW`实际上会抵消`VK_LOADER_LAYERS_DISABLE`的行为。列在`VK_LOADER_LAYERS_ALLOW`中的显式层不会因此被启用。列在`VK_LOADER_LAYERS_ALLOW`中、且始终处于活动状态的隐式层，也就是不需要任何外部上下文即可启用的那些隐式层，则会被启用。

<a id="vk_instance_layers"></a>
##### `VK_INSTANCE_LAYERS`

最初的`VK_INSTANCE_LAYERS`可以视为`VK_LOADER_LAYERS_ENABLE`的一种特殊情况。因此，任何通过`VK_INSTANCE_LAYERS`启用的 layer 都会被视为与通过`VK_LOADER_LAYERS_ENABLE`启用的 layer 相同，从而覆盖`VK_LOADER_LAYERS_DISABLE`中提供的任何禁用设置。

<a id="exception-for-elevated-privileges"></a>
### 提升权限时的例外情况

出于安全原因，如果以提升权限运行 Vulkan 应用程序，则会忽略`VK_LAYER_PATH`、`VK_ADD_LAYER_PATH`、`VK_IMPLICIT_LAYER_PATH`和`VK_ADD_IMPLICIT_LAYER_PATH`。这是因为它们可能会将 loader 平时无法发现的新库插入到可执行进程中。正因如此，这些环境变量只能用于未使用提升权限的应用程序。

更多信息请参阅顶层[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文档中的[特权提升注意事项](LoaderInterfaceArchitecture.md#elevated-privilege-caveats)。

<a id="layer-version-negotiation"></a>
## Layer 版本协商

现在 layer 已经被发现，应用可以选择加载它；或者在隐式层的情况下，它也可能默认被加载。当 loader 尝试加载该 layer 时，它首先会尝试协商 loader 与 layer 接口的版本。为了协商 loader/layer 接口版本， layer 必须实现`vkNegotiateLoaderLayerInterfaceVersion`函数。该接口在`include/vulkan/vk_layer.h`中定义如下：

```cpp
typedef enum VkNegotiateLayerStructType {
    LAYER_NEGOTIATE_INTERFACE_STRUCT = 1,
} VkNegotiateLayerStructType;

typedef struct VkNegotiateLayerInterface {
    VkNegotiateLayerStructType sType;
    void *pNext;
    uint32_t loaderLayerInterfaceVersion;
    PFN_vkGetInstanceProcAddr pfnGetInstanceProcAddr;
    PFN_vkGetDeviceProcAddr pfnGetDeviceProcAddr;
    PFN_GetPhysicalDeviceProcAddr pfnGetPhysicalDeviceProcAddr;
} VkNegotiateLayerInterface;

VkResult
   vkNegotiateLoaderLayerInterfaceVersion(
      VkNegotiateLayerInterface *pVersionStruct);
```

`VkNegotiateLayerInterface`结构体与其他 Vulkan 结构体类似。其中的`sType`字段在这里采用了一个新的枚举值，专门用于 loader/layer 内部接口交互。将来`sType`的有效值可能会扩展，但目前只有一个值：`LAYER_NEGOTIATE_INTERFACE_STRUCT`。

该函数（`vkNegotiateLoaderLayerInterfaceVersion`）应由 layer 导出，这样在 Windows 上使用 “GetProcAddress”，或在 Linux 和 macOS 上使用 “dlsym” 时，都应能返回指向它的有效函数指针。 loader 获取到该 layer 函数的有效地址后，会创建一个`VkNegotiateLayerInterface`类型的变量，并按如下方式初始化：

  1. 将结构体的`sType`设为`LAYER_NEGOTIATE_INTERFACE_STRUCT`
  2. 将`pNext`设为`NULL`
      - 这是为未来扩展预留
  3. 将`loaderLayerInterfaceVersion`设为 loader 当前希望使用的接口版本
      - loader 发送的最小值将是 2，因为这是第一个支持该函数的版本

随后，loader 会分别调用每个 layer 的`vkNegotiateLoaderLayerInterfaceVersion`函数，并传入已填充好的`VkNegotiateLayerInterface`。

该函数允许 loader 与 layer 就将要使用的接口版本达成一致。`loaderLayerInterfaceVersion`字段同时是输入参数和输出参数。该字段由 loader 填入 loader 所支持、且期望使用的最新接口版本（通常就是最新版本）。 layer 接收到该值后，会通过同一个字段返回它希望使用的版本。由于它负责建立 loader 与 layer 之间的接口版本，因此这应当是 loader 对 layer 发出的第一个调用（甚至早于对`vkGetInstanceProcAddr`的任何调用）。

如果接收调用的 layer 已经不再支持 loader 提供的接口版本（例如因为该版本已被弃用），那么它应返回`VK_ERROR_INITIALIZATION_FAILED`错误。否则，它应将`loaderLayerInterfaceVersion`的值设为 layer 与 loader 共同支持的最新接口版本，并返回`VK_SUCCESS`。

如果 loader 提供的接口版本比 layer 支持的版本新， layer 也应返回`VK_SUCCESS`，因为由 loader 负责判断自己是否能够支持该 layer 所支持的较旧接口版本。如果 layer 的接口版本高于 loader 的版本， layer 也应返回`VK_SUCCESS`，但返回 loader 的版本。因此，当返回`VK_SUCCESS`时，`loaderLayerInterfaceVersion`中将包含该 layer 应使用的目标接口版本。

如果 loader 收到的是`VK_ERROR_INITIALIZATION_FAILED`而不是`VK_SUCCESS`，那么 loader 会将该 layer 视为不可用，并且不会加载它。在这种情况下，应用在枚举时将看不到这个 layer。 *请注意，loader 当前向后兼容所有 layer 接口版本，因此 layer 不应能够请求比 loader 所支持版本更旧的版本。*

该函数 **MUST NOT** 向下调用 layer 链中的下一个 layer。 loader 会分别与每个 layer 单独交互。

如果 layer 支持新接口并报告版本为 2 或更高，则 layer 应将函数指针值填入其内部函数：

    - `pfnGetInstanceProcAddr`应设为 layer 内部的`GetInstanceProcAddr`函数。
    - `pfnGetDeviceProcAddr`应设为 layer 内部的`GetDeviceProcAddr`函数。
    - `pfnGetPhysicalDeviceProcAddr`应设为 layer 内部的`GetPhysicalDeviceProcAddr`函数。
      - 如果 layer 不支持任何物理设备扩展，可将该值设为`NULL`。
      - 关于该函数的更多内容将在后文说明

loader 将使用`VkNegotiateLayerInterface`结构中的`pfnGetInstanceProcAddr`和`pfnGetDeviceProcAddr`函数。在这些更改之前，loader 会在 Windows 上通过 “GetProcAddress”，或在 Linux 和 macOS 上通过 “dlsym”，分别查询这些函数。

<a id="layer-call-chains-and-distributed-dispatch"></a>
## Layer 调用链与分布式分发

有两个关键的架构特性决定了 loader 需要采用`Layer Library`接口：

1. 相互分离且彼此独立的 instance 调用链与 device 调用链
2. 分布式分发（distributed dispatch）

有关更多信息，请阅读前文[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文档中[调度表与调用链](LoaderInterfaceArchitecture.md#dispatch-tables-and-call-chains) 一节。

这里需要特别注意的是，Layer 可以拦截 Vulkan instance 函数、device 函数，或者同时拦截两者。如果某个 Layer 要拦截 instance 函数，它就必须参与 instance 调用链。如果某个 Layer 要拦截 device 函数，它就必须参与 device 调用链。

请记住，Layer 不必拦截所有 instance 或 device 函数，它可以只选择拦截其中的一个子集。

通常，当某个 Layer 拦截了给定的 Vulkan 函数时，它会根据需要继续向下调用 instance 或 device 调用链。 loader 与参与调用链的所有 Layer 库会协同工作，以确保调用能按照正确顺序从一个实体传递到下一个实体。这种共同维护调用链顺序的机制，下文称为 **分布式分发**。

在分布式分发中，每个 Layer 都负责正确调用调用链中的下一个实体。这意味着，对于 Layer 所拦截的所有 Vulkan 函数，都需要相应的分发机制。如果某个 Vulkan 函数没有被某个 Layer 拦截，或者某个 Layer 选择不继续向下调用调用链而直接终止该函数，那么该函数就不需要进行分发。

例如，如果启用的 Layer 只拦截了部分 instance 函数，则调用链如下所示：
![Instance Function Chain](./images/function_instance_chain.png)

同样地，如果启用的 Layer 只拦截了少数几个 device 函数，则调用链可能如下所示：
![Device Function Chain](./images/function_device_chain.png)

loader 负责将所有核心 Vulkan 函数以及 instance 扩展函数分发到调用链中的第一个实体。

<a id="layer-unknown-physical-device-extensions"></a>
## Layer 未知物理设备扩展

如果 Layer 拦截的入口点以`VkPhysicalDevice`作为第一个参数，则该 Layer *应当* 支持`vk_layerGetPhysicalDeviceProcAddr`。此函数是在 Layer Interface Version 2 中加入的，它使 loader 能够区分那些以`VkDevice`为第一个参数的入口点和以`VkPhysicalDevice`为第一个参数的入口点。这样一来，loader 就能更优雅地支持那些它自身并不认识的入口点。

```cpp
PFN_vkVoidFunction
   vk_layerGetPhysicalDeviceProcAddr(
      VkInstance instance,
      const char* pName);
```

这个函数的行为与`vkGetInstanceProcAddr`和`vkGetDeviceProcAddr`类似，但它只应返回物理设备扩展入口点的值。也就是说，它会将`pName`与 Layer 所支持的每一个物理设备函数进行比较。

该函数的实现应具有如下行为：

* 如果名称对应的是该 Layer 支持的某个物理设备函数，则应返回指向该 Layer 相应函数的指针。
* 如果名称对应的是一个有效函数，但**不是**物理设备函数（例如该 Layer 实现的 instance、device 或其他函数），则应返回`NULL`。
  * 因为该命令不是物理设备扩展，所以 Layer 不应继续向下调用。
* 如果 Layer 完全不知道这个函数是什么，则它应继续沿 Layer 链向下调用下一个`vk_layerGetPhysicalDeviceProcAddr`。
  * 这可以通过以下两种方式之一获取：
    * 在`vkCreateInstance`期间，它会通过传给 Layer 的`VkLayerInstanceCreateInfo`结构体中的链信息传入。
      * 使用`get_chain_info()`获取指向`VkLayerInstanceCreateInfo`结构体的指针。这里称之为`chain_info`。
      * 该地址位于`chain_info->u.pLayerInfo->pfnNextGetPhysicalDeviceProcAddr`
      * 参见[CreateInstance 示例代码](#example-code-for-createinstance)
    * 使用下一个 Layer 的`GetInstanceProcAddr`函数查询`vk_layerGetPhysicalDeviceProcAddr`。

如果某个 Layer 打算支持那些以`VkPhysicalDevice`作为可分发参数的函数，那么它就应当支持`vk_layerGetPhysicalDeviceProcAddr`。这是因为，如果这些函数对 loader 来说是未知的，例如它们来自尚未发布的扩展，或者 loader 版本较旧、*尚未* 知道它们，那么 loader 将无法判断这是 device 函数还是物理设备函数。

如果某个 Layer 实现了`vk_layerGetPhysicalDeviceProcAddr`，那么它应在[Layer 版本协商](#layer-version-negotiation) 期间，通过`VkNegotiateLayerInterface`结构体中的`pfnGetPhysicalDeviceProcAddr`成员返回其`vk_layerGetPhysicalDeviceProcAddr`函数地址。此外，该 Layer 还应确保`vkGetInstanceProcAddr`在查询`vk_layerGetPhysicalDeviceProcAddr`时返回有效的函数指针。

注意：如果某个 Layer 包装了`VkInstance`句柄，那么对`vk_layerGetPhysicalDeviceProcAddr`的支持就*不是*可选项，而是必须实现。

在支持`vk_layerGetPhysicalDeviceProcAddr`的情况下，loader 的`vkGetInstanceProcAddr`行为如下：

1. 检查是否为核心函数：
   - 如果是，则返回该函数指针
2. 检查是否为已知的 instance 扩展函数或 device 扩展函数：
   - 如果是，则返回该函数指针
3. 调用 Layer/driver 的`GetPhysicalDeviceProcAddr`
   - 如果返回非`NULL`，则返回一个通用物理设备函数的 trampoline，并设置一个通用 terminator，将其传递给正确的 driver。
4. 使用`GetInstanceProcAddr`继续向下调用
   - 如果返回非`NULL`，则将其视为未知的逻辑设备命令。这意味着要设置一个通用 trampoline 函数，将`VkDevice`作为第一个参数，并在从`VkDevice`获取调度表后，调整调度表以调用 driver/Layer 的函数。然后返回对应 trampoline 函数的指针。
5. 返回`NULL`

这样一来，如果该命令后来被提升为核心命令，就不再会通过`vk_layerGetPhysicalDeviceProcAddr`来设置。另外，如果 loader 后续直接加入了对该扩展的支持，也不会再走到步骤 3，因为步骤 2 会直接返回有效函数指针。不过，Layer 仍应继续通过`vk_layerGetPhysicalDeviceProcAddr`支持该命令的查询，至少要持续到某次 Vulkan 版本提升之后，因为旧版 loader 仍可能尝试使用这些命令。

<a id="reason-for-adding-vk_layergetphysicaldeviceprocaddr"></a>
### 添加`vk_layerGetPhysicalDeviceProcAddr`的原因

最初，当在 loader 中调用`vkGetInstanceProcAddr`时，其行为如下：

1. loader 检查是否为核心函数：
   - 如果是，则返回该函数指针
2. loader 检查是否为已知扩展函数：
   - 如果是，则返回该函数指针
3. 如果 loader 对它一无所知，则使用`GetInstanceProcAddr`继续向下调用
   - 如果返回非`NULL`，则将其视为未知的逻辑设备命令。
   - 这意味着要设置一个通用 trampoline 函数，将`VkDevice`作为第一个参数，并在从`VkDevice`获取调度表后，调整调度表以调用 Driver/Layer 的函数。
4. 如果以上都失败，则 loader 向应用返回`NULL`。

当某个 Layer 试图暴露新的物理设备扩展，而应用认识这些扩展、loader 却完全不知道时，这种做法就会引发问题。由于 loader 对此一无所知，它会在上述流程中走到步骤 3，并将该函数当作未知的逻辑设备命令来处理。问题在于，这会创建一个通用的`VkDevice` trampoline 函数，而该函数在首次调用时会尝试将`VkPhysicalDevice`按`VkDevice`进行解引用。这会导致崩溃或数据损坏。

<a id="layer-intercept-requirements"></a>
## Layer 拦截要求

* Layer 通过定义一个与该 Vulkan API 函数签名**完全一致**的 C/C++ 函数来拦截某个 Vulkan 函数。
* Layer 若要参与 instance 调用链，**至少必须拦截** `vkGetInstanceProcAddr`和`vkCreateInstance`。
* Layer 若要参与 device 调用链，**也可以拦截** `vkGetDeviceProcAddr`和`vkCreateDevice`。
* 对于 Layer 所拦截、且具有非`void`返回值的任何 Vulkan 函数，Layer 拦截函数**必须返回适当的值**。
* Layer 拦截的大多数函数**都应继续向下调用调用链**，调用下一个实体中的对应 Vulkan 函数。
  * Layer 的常见行为是先拦截调用、执行某些处理，然后再将其传递给下一个实体。
    * 如果 Layer 不向下传递这些信息，就可能出现未定义行为。
    * 这是因为调用链中更下方的 Layer，或任何 driver，都将收不到该函数调用。
  * 有一个函数**绝不能继续向下调用**：
    * `vkNegotiateLoaderLayerInterfaceVersion`
  * 有三个常见函数**可以不继续向下调用**：
    * `vkGetInstanceProcAddr`
    * `vkGetDeviceProcAddr`
    * `vk_layerGetPhysicalDeviceProcAddr`
    * 这些函数只会在遇到自己不拦截的 Vulkan 函数时，才继续向下调用。
* Layer 的拦截函数**可以在拦截之外插入额外的 Vulkan 函数调用**。
  * 例如，某个拦截`vkQueueSubmit`的 Layer 可能希望在沿调用链继续调用`vkQueueSubmit`之后，再额外调用一次`vkQueueWaitIdle`。
  * 这样会在调用链中产生两次向下调用：第一次是沿`vkQueueSubmit`调用链，第二次是沿`vkQueueWaitIdle`调用链。
  * Layer 插入的任何附加调用都必须位于同一条调用链上
    * 如果该函数是 device 函数，则只能添加其他 device 函数
    * 同样地，如果该函数是 instance 函数，则只能添加其他 instance 函数

<a id="distributed-dispatching-requirements"></a>
## 分布式分发要求

- 对于 Layer 拦截的每个入口点，它都必须跟踪调用链中下一个实体所对应的入口点，以便向下调用。
  * 换句话说，Layer 必须维护一个函数指针列表，这些函数指针的类型应与调用下一个实体所需的类型相匹配。
  * 这可以通过多种方式实现，不过为了表述清晰，以下统称其为调度表。
- Layer 可以使用`VkLayerDispatchTable`结构体作为 device 调度表（参见`include/vulkan/vk_dispatch_table_helper.h`）。
- Layer 可以使用`VkLayerInstanceDispatchTable`结构体作为 instance 调度表（参见`include/vulkan/vk_dispatch_table_helper.h`）。
- Layer 的`vkGetInstanceProcAddr`函数会使用下一个实体的`vkGetInstanceProcAddr`，以便对未知函数（即未拦截的函数）继续沿调用链向下调用。
- Layer 的`vkGetDeviceProcAddr`函数会使用下一个实体的`vkGetDeviceProcAddr`，以便对未知函数（即未拦截的函数）继续沿调用链向下调用。
- Layer 的`vk_layerGetPhysicalDeviceProcAddr`函数会使用下一个实体的`vk_layerGetPhysicalDeviceProcAddr`，以便对未知函数（即未拦截的函数）继续沿调用链向下调用。

<a id="layer-conventions-and-rules"></a>
## Layer 约定与规则

当某个 Layer 被插入到原本符合规范的 Vulkan 驱动中时，最终结果<b>必须</b>仍然是一个符合规范的 Vulkan 驱动。其目的是让 Layer 具备明确定义的基线行为。因此，它必须遵循下面定义的一些约定和规则。

为了让 Layer 拥有唯一名称，并降低 loader 在尝试加载这些 Layer 时发生冲突的概率，Layer <b>必须</b>遵循以下命名标准：

* 以`VK_LAYER_`为前缀
* 在此前缀之后，跟一个全大写的组织名或公司名（LunarG）、唯一公司标识符（Nvidia 的 NV），或软件产品名（RenderDoc）
* 再跟上该 Layer 的具体名称（通常为小写，但并非强制）
  * 注意：如果具体名称由多个单词组成，<b>必须</b>使用下划线分隔

有效的 Layer 名称示例如下：

* <b>VK_LAYER_KHRONOS_validation</b>
  * Organization = "KHRONOS"
  * Specific name = "validation"
* <b>VK_LAYER_RENDERDOC_Capture</b>
  * Application = "RENDERDOC"
  * Specific name = "Capture"
* <b>VK_LAYER_VALVE_steam_fossilize_32</b>
  * Organization = "VALVE"
  * Application = "steam"
  * Specific name = "fossilize"
  * OS-modifier = "32"  (32 位版本)
* <b>VK_LAYER_NV_nsight</b>
  * Organization Acronym = "NV"（Nvidia）
  * Specific name = "nsight"

有关 Layer 命名的更多细节，可参见[Vulkan style-guide](https://www.khronos.org/registry/vulkan/specs/1.2/styleguide.html#extensions-naming-conventions) 中的 3.4 节“Version, Extension, and Layer Naming Conventions”。

Layer 总是与其他 Layer 链接在一起。它不能对下层 Layer 发起无效调用，也不能依赖下层 Layer 的未定义行为。当它改变了某个函数的行为时，必须确保它的上层 Layer 不会因为这种行为变化，而对下层 Layer 发起无效调用或依赖其未定义行为。例如，当某个 Layer 拦截对象创建函数，以包装其下层 Layer 创建的对象时，它必须确保下层 Layer 永远不会直接从该 Layer 或间接通过其上层 Layer 看到这些包装对象。

当 Layer 需要主机内存时，可以忽略提供的分配器。如果某个 Layer 预期会运行在生产环境中，则更推荐它使用提供的内存分配器。例如，这通常适用于始终启用的隐式层。这样应用程序就可以将该 Layer 的内存使用情况纳入统计。

其他规则还包括：

- `vkEnumerateInstanceLayerProperties` **必须** 枚举且**只能**枚举它自身这个 Layer。
- `vkEnumerateInstanceExtensionProperties` **必须**处理`pLayerName`为其自身名称的情况。
  - 否则它**必须**返回`VK_ERROR_LAYER_NOT_PRESENT`，包括`pLayerName`为`NULL`时。
- `vkEnumerateDeviceLayerProperties` **已弃用，可以省略**。
  - 使用它会导致未定义行为。
- `vkEnumerateDeviceExtensionProperties` **必须**处理`pLayerName`为其自身名称的情况。
  - 在其他情况下，它应继续链接到其他 Layer。
- `vkCreateInstance` **不得**因未识别的 Layer 名称或扩展名称而生成错误。
  - 它可以假定这些 Layer 名称和扩展名称已经过验证。
- `vkGetInstanceProcAddr`通过返回本地入口点来拦截 Vulkan 函数
  - 否则，它返回沿 instance 调用链向下调用所得的值。
- `vkGetDeviceProcAddr`通过返回本地入口点来拦截 Vulkan 函数
  - 否则，它返回沿 device 调用链向下调用所得的值。
  - 如果 Layer 实现了 device 级调用链，则还必须拦截以下附加函数：
    - `vkGetDeviceProcAddr`
    - `vkCreateDevice`（仅在进行任何 device 级调用链处理时需要）
       - **注意：** 较旧的 Layer 库可能预期`vkGetInstanceProcAddr`忽略`instance`，当`pName`为`vkCreateDevice`时也是如此。
- 规范**要求**对于已禁用的函数，`vkGetInstanceProcAddr`和`vkGetDeviceProcAddr`必须返回`NULL`。
  - Layer 可以自己返回`NULL`，也可以依赖后续 Layer 来这样做。
- 当查询`vkCreateInstance`时，Layer 实现的`vkGetInstanceProcAddr` **应当**无论`instance`参数取何值，都返回有效函数指针。
  - 规范**要求** `instance`参数**必须**为`NULL`。不过，较早版本的规范并无此要求，因此允许传入非`NULL`的`instance`句柄并返回有效的`vkCreateInstance`函数指针。 Vulkan-Loader 本身就是这样做的，并且会继续如此，以维持与那些发布于此规范变更之前的 Layer 的兼容性。

<a id="layer-dispatch-initialization"></a>
## Layer 调度初始化

- Layer 会在其`vkCreateInstance`函数内部初始化 instance 调度表。
- Layer 会在其`vkCreateDevice`函数内部初始化 device 调度表。
- 对于`vkCreateInstance`和`VkCreateDevice`，loader 会通过`VkInstanceCreateInfo`和`VkDeviceCreateInfo`结构体中的`pNext`字段，向 Layer 传递一个初始化结构体链表。
- 该链表的头节点，对于 instance 是`VkLayerInstanceCreateInfo`类型，对于 device 是`VkLayerDeviceCreateInfo`类型。详见`include/vulkan/vk_layer.h`。
- loader 会在`VkLayerInstanceCreateInfo`的`sType`字段中使用`VK_STRUCTURE_TYPE_LOADER_INSTANCE_CREATE_INFO`。
- loader 会在`VkLayerDeviceCreateInfo`的`sType`字段中使用`VK_STRUCTURE_TYPE_LOADER_DEVICE_CREATE_INFO`。
- `function`字段指示在`VkLayer*CreateInfo`中应如何解释联合体字段`u`。 loader 会将`function`字段设置为`VK_LAYER_LINK_INFO`。这表示`u`字段应为`VkLayerInstanceLink`或`VkLayerDeviceLink`。
- `VkLayerInstanceLink`和`VkLayerDeviceLink`结构体是该链表的节点。
- `VkLayerInstanceLink`包含 Layer 所使用的下一个实体的`vkGetInstanceProcAddr`。
- `VkLayerDeviceLink`包含 Layer 所使用的下一个实体的`vkGetInstanceProcAddr`和`vkGetDeviceProcAddr`。
- 在 loader 按上述方式设置好这些结构体后，Layer 必须按如下方式初始化其调度表：
  - 在`VkInstanceCreateInfo`/`VkDeviceCreateInfo`结构体中找到`VkLayerInstanceCreateInfo`/`VkLayerDeviceCreateInfo`结构体。
  - 从`pLayerInfo`字段中获取下一个实体的`vkGet*ProcAddr`。
  - 对于 CreateInstance，通过调用`pfnNextGetInstanceProcAddr`获取下一个实体的`vkCreateInstance`：`pfnNextGetInstanceProcAddr(NULL, "vkCreateInstance")`。
  - 对于 CreateDevice，通过调用`pfnNextGetInstanceProcAddr`获取下一个实体的`vkCreateDevice`：`pfnNextGetInstanceProcAddr(instance, "vkCreateDevice")`，并传入已经创建好的 instance 句柄。
  - 将链表前进到下一个节点：`pLayerInfo = pLayerInfo->pNext`。
  - 继续向下调用链中的`vkCreateDevice`或`vkCreateInstance`
  - 对于调度表中所需的每个 Vulkan 函数，通过下一个实体的 Get*ProcAddr 函数逐个调用一次，以初始化 Layer 调度表

<a id="example-code-for-createinstance"></a>
## CreateInstance 示例代码

```cpp
VkResult
   vkCreateInstance(
      const VkInstanceCreateInfo *pCreateInfo,
      const VkAllocationCallbacks *pAllocator,
      VkInstance *pInstance)
{
   VkLayerInstanceCreateInfo *chain_info =
        get_chain_info(pCreateInfo, VK_LAYER_LINK_INFO);

    assert(chain_info->u.pLayerInfo);
    PFN_vkGetInstanceProcAddr fpGetInstanceProcAddr =
        chain_info->u.pLayerInfo->pfnNextGetInstanceProcAddr;
    PFN_vkCreateInstance fpCreateInstance =
        (PFN_vkCreateInstance)fpGetInstanceProcAddr(NULL, "vkCreateInstance");
    if (fpCreateInstance == NULL) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    // Advance the link info for the next element of the chain.
    // This ensures that the next layer gets it's layer info and not
    // the info for our current layer.
    chain_info->u.pLayerInfo = chain_info->u.pLayerInfo->pNext;

    // Continue call down the chain
    VkResult result = fpCreateInstance(pCreateInfo, pAllocator, pInstance);
    if (result != VK_SUCCESS)
        return result;

    // Init layer's dispatch table using GetInstanceProcAddr of
    // next layer in the chain.
    instance_dispatch_table = new VkLayerInstanceDispatchTable;
    layer_init_instance_dispatch_table(
        *pInstance, my_data->instance_dispatch_table, fpGetInstanceProcAddr);

    // Other layer initialization
    ...

    return VK_SUCCESS;
}
```

<a id="example-code-for-createdevice"></a>
## CreateDevice 示例代码

```cpp
VkResult
   vkCreateDevice(
      VkPhysicalDevice gpu,
      const VkDeviceCreateInfo *pCreateInfo,
      const VkAllocationCallbacks *pAllocator,
      VkDevice *pDevice)
{
    VkInstance instance = GetInstanceFromPhysicalDevice(gpu);
    VkLayerDeviceCreateInfo *chain_info =
        get_chain_info(pCreateInfo, VK_LAYER_LINK_INFO);

    PFN_vkGetInstanceProcAddr fpGetInstanceProcAddr =
        chain_info->u.pLayerInfo->pfnNextGetInstanceProcAddr;
    PFN_vkGetDeviceProcAddr fpGetDeviceProcAddr =
        chain_info->u.pLayerInfo->pfnNextGetDeviceProcAddr;
    PFN_vkCreateDevice fpCreateDevice =
        (PFN_vkCreateDevice)fpGetInstanceProcAddr(instance, "vkCreateDevice");
    if (fpCreateDevice == NULL) {
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    // Advance the link info for the next element on the chain.
    // This ensures that the next layer gets it's layer info and not
    // the info for our current layer.
    chain_info->u.pLayerInfo = chain_info->u.pLayerInfo->pNext;

    VkResult result = fpCreateDevice(gpu, pCreateInfo, pAllocator, pDevice);
    if (result != VK_SUCCESS) {
        return result;
    }

    // initialize layer's dispatch table
    device_dispatch_table = new VkLayerDispatchTable;
    layer_init_device_dispatch_table(
        *pDevice, device_dispatch_table, fpGetDeviceProcAddr);

    // Other layer initialization
    ...

    return VK_SUCCESS;
}
```

这里会调用`GetInstanceFromPhysicalDevice`函数来获取 instance 句柄。在实际实现中，Layer 可以采用任意方式，从物理设备中取得 instance 句柄。

<a id="meta-layers"></a>
## 元层

元层是一种特殊的 layer，仅通过 Khronos loader 提供。普通 layer 关联到某一个特定库，而元层实际上是一种集合 layer，其中包含按顺序排列的其他 layer 列表（称为组成 layer）。

元层的优点包括：
  1. 只需使用一个 layer 名称激活元层，即可通过将多个 layer 归入同一个元层来一次性激活多个 layer。
  2. 各个组成 layer 的加载顺序可以在元层中定义。
  3. layer 配置（位于元层清单文件内部）可以方便地与他人共享。
  4. loader 会自动汇总元层中各组成 layer 的所有 instance 扩展和 device 扩展，并在应用程序查询时将其作为元层的属性返回给应用程序。

定义和使用元层时有以下限制：
  1. 元层清单文件**必须**格式正确，并且包含一个或多个组成 layer。
  3. 要使用元层，系统中**必须存在**所有组成 layer。
  4. 要使用元层，所有组成 layer 的 Vulkan API 主版本号和次版本号都**必须与**元层一致。

元层中的组成 layer 在 instance 或 device 调用链中的顺序很简单：
  * 列表中的第一个 layer 最靠近应用程序。
  * 列表中的最后一个 layer 最靠近驱动。

在元层清单文件中，每个组成 layer 都通过其 layer 名称列出。这个名称就是各组成 layer 清单文件中`layer`或`layers`节点下`name`字段的值。这也是通常在`vkCreateInstance`期间激活某个 layer 时所使用的名称。

无论是在组成 layer 列表内部，还是在全局所有已启用 layer 中，只要出现重复的 layer 名称，loader 都会直接忽略。每个 layer 名称只会使用第一次出现的那个实例。

例如，如果某个 layer 通过环境变量`VK_INSTANCE_LAYERS`启用，同时该 layer 也列在某个元层中，那么通过环境变量启用的 layer 会被使用，而作为组成 layer 的那个实例会被丢弃。同样地，如果某人启用了一个元层，随后又单独启用了其中某个组成 layer，那么第二次出现的该 layer 名称也会被忽略。

定义元层所需的清单文件格式可在[Layer 清单文件格式](#layer-manifest-file-format)一节中找到。

<a id="override-meta-layer"></a>
### 覆盖元层

如果系统中发现了一个名为`VK_LAYER_LUNARG_override`的隐式元层，loader 会将其用作“覆盖”layer。它用于有选择地启用或禁用其他 layer 的加载。它既可以全局生效，也可以只对某一个或某几个特定应用程序生效。覆盖元层可以包含以下额外键：
  * `blacklisted_layers` - 即使应用程序请求加载，也不应被加载的显式层名称列表。
  * `app_keys` - 覆盖 layer 适用的可执行文件路径列表。
  * `override_paths` - 用作组成 layer 搜索位置的路径列表。

当应用程序启动且覆盖 layer 存在时，loader 会先检查该应用程序是否在列表中。如果不在，则不会应用该覆盖 layer。如果列表为空，或者`app_keys`不存在，则 loader 会将覆盖 layer 视为全局生效，并在每个应用程序启动时都应用它。

如果覆盖 layer 包含`override_paths`，那么组成 layer 只会使用这组路径。因此，它会忽略默认的显式层和隐式层搜索位置，以及`VK_LAYER_PATH`等环境变量设置的路径。如果提供的覆盖路径中缺少任何一个组成 layer，该元层就会被禁用。

覆盖元层主要在使用 Vulkan SDK 附带的[VkConfig](https://github.com/LunarG/VulkanTools/blob/main/vkconfig/README.md) 工具时启用。通常只有在 VkConfig 工具实际运行期间，它才会存在。更多信息请参阅该文档。

<a id="pre-instance-functions"></a>
## 预实例函数

Vulkan 包含少量无需任何可分发对象即可调用的函数。 <b>大多数 layer 都不会拦截这些函数</b>，因为 layer 是在创建 instance 时启用的。但是，在某些条件下，layer 也可以拦截这些函数。

layer 可能希望拦截这些预实例函数的一个原因，是过滤掉原本会由 Vulkan 驱动返回给应用程序的扩展。[RenderDoc](https://renderdoc.org/) 就是这样一个 layer，它会拦截这些预实例函数，以便禁用自己不支持的扩展。

要拦截这些预实例函数，需要满足几个条件：
* 该 layer 必须是隐式层
* 该 layer 的清单文件版本必须为 1.1.2 或更高
* 该 layer 必须导出每个被拦截函数的入口点符号
* 该 layer 的清单文件必须在`pre_instance_functions` JSON 对象中指定每个被拦截函数的名称

可以通过这种方式拦截的函数有：
* `vkEnumerateInstanceExtensionProperties`
* `vkEnumerateInstanceLayerProperties`
* `vkEnumerateInstanceVersion`

预实例函数与其他所有 layer 拦截函数的工作方式不同。其他拦截函数的函数原型与其所拦截函数的原型完全相同。然后，它们依赖在创建 instance 或 device 时传递给 layer 的数据，以便 layer 能继续向下调用链。由于调用预实例函数时还不需要先创建 instance，因此这些函数必须使用另一种机制来构造调用链。该机制是在调用 layer 拦截函数时，额外向其传递一个参数。这个参数是一个指向结构体的指针，其定义如下：

```cpp
typedef struct Vk...Chain
{
    struct {
        VkChainType type;
        uint32_t version;
        uint32_t size;
    } header;
    PFN_vkVoidFunction pfnNextLayer;
    const struct Vk...Chain* pNextLink;
} Vk...Chain;
```

这些结构体定义在`vk_layer.h`文件中，因此外部代码无需重新定义这些链结构体。每个结构体的名称都与其对应函数的名称相似，但开头的`V`会大写，并在末尾添加`Chain`一词。例如，`vkEnumerateInstanceExtensionProperties`对应的结构体名为`VkEnumerateInstanceExtensionPropertiesChain`。此外，结构体成员`pfnNextLayer`实际上并不是真正的 void 函数指针 &mdash; 它的类型会是该调用链中对应函数的真实类型。

每个 layer 拦截函数的原型都必须与被拦截函数的原型相同，只是第一个参数必须是该函数对应的链结构体（以 const 指针形式传入）。例如，若某个函数希望拦截`vkEnumerateInstanceExtensionProperties`，其原型应为：

```cpp
VkResult
   InterceptFunctionName(
      const VkEnumerateInstanceExtensionPropertiesChain* pChain,
      const char* pLayerName,
      uint32_t* pPropertyCount,
      VkExtensionProperties* pProperties);
```

函数名本身可以任意指定；只要该名称在 layer 清单文件中给出即可（参见[Layer 清单文件格式](#layer-manifest-file-format)）。每个拦截函数的实现都负责利用该链参数调用调用链中的下一个项。具体做法是调用链结构体中的`pfnNextLayer`成员，将`pNextLink`作为第一个参数传入，再依次传入其余函数参数。例如，下面这个`vkEnumerateInstanceExtensionProperties`的简单实现除了继续向下调用链之外不做任何事：

```cpp
VkResult
   InterceptFunctionName(
      const VkEnumerateInstanceExtensionPropertiesChain* pChain,
      const char* pLayerName,
      uint32_t* pPropertyCount,
      VkExtensionProperties* pProperties)
{
   return pChain->pfnNextLayer(
      pChain->pNextLink, pLayerName, pPropertyCount, pProperties);
}
```

使用 C++ 编译器时，每种链类型还会定义一个名为`CallDown`的函数，可用于自动处理第一个参数。采用这种方式实现上述函数时，代码如下：

```cpp
VkResult
   InterceptFunctionName(
      const VkEnumerateInstanceExtensionPropertiesChain* pChain,
      const char* pLayerName,
      uint32_t* pPropertyCount,
      VkExtensionProperties* pProperties)
{
   return pChain->CallDown(pLayerName, pPropertyCount, pProperties);
}
```

与 layer 中的其他函数不同，layer 不能在这些函数调用之间保存任何全局数据。由于 Vulkan 在创建 instance 之前不会保存任何状态，因此所有 layer 库都会在每次预实例调用结束后被释放。这意味着隐式层可以利用预实例拦截来修改这些函数返回的数据，但不能用它们来记录这些数据。

<a id="special-considerations"></a>
## 特殊注意事项

<a id="associating-private-data-with-vulkan-objects-within-a-layer"></a>
### 在 Layer 内将私有数据与 Vulkan 对象关联

layer 可能希望将自己的私有数据与一个或多个 Vulkan 对象关联起来。常见的两种方法是哈希映射和对象包装。

<a id="wrapping"></a>
#### 包装

loader 支持 layer 包装任意 Vulkan 对象，包括可分发对象。对于那些返回对象句柄的函数，每个 layer 都不会修改沿调用链向下传递的值。这是因为更底层的项可能仍然需要使用原始值。但是，当该值从更低层的 layer（也可能是驱动）返回后，layer 会保存这个句柄，并将它自己的句柄返回给上层 layer（也可能是应用程序）。当 layer 收到一个 Vulkan 函数调用，其中使用的是它先前返回过句柄的对象时，layer 必须先解包该句柄，再将先前保存的句柄传给下层 layer。这意味着，layer **必须拦截每一个会使用到该对象的 Vulkan 函数**，并按需要对对象进行包装或解包。这包括支持所有使用该 layer 所包装对象的扩展函数，以及`vk_layerGetPhysicalDeviceProcAddr`等 loader-layer 接口函数。

位于对象包装 layer 之上的 layer 会看到包装后的对象。包装可分发对象的 layer 必须确保包装结构体中的第一个字段是指向`vk_layer.h`中定义的调度表的指针。具体来说，一个包装后的 instance 级可分发对象可以如下所示：

```cpp
struct my_wrapped_instance_obj_ {
    VkLayerInstanceDispatchTable *disp;
    // whatever data layer wants to add to this object
};
```

一个包装后的 device 级可分发对象可以如下所示：

```cpp
struct my_wrapped_instance_obj_ {
    VkLayerDispatchTable *disp;
    // whatever data layer wants to add to this object
};
```

包装可分发对象的 layer 必须遵循下文关于创建新可分发对象的指导原则。

<a id="cautions-about-wrapping"></a>
#### 关于包装的注意事项

通常不鼓励 layer 包装对象，因为这可能会与新的扩展产生不兼容问题。例如，假设某个 layer 包装了`VkImage`对象，并且已经为所有核心函数正确处理了`VkImage`对象句柄的包装和解包。如果后来出现了一个新扩展，其函数参数中包含`VkImage`对象，而该 layer 又不支持这些新函数，那么同时使用该 layer 和该新扩展的应用程序在调用这些新函数时就会产生未定义行为（例如应用程序可能崩溃）。这是因为更底层的 layer 和驱动无法收到它们自己生成的那个句柄。相反，它们收到的会是仅被包装该对象的 layer 所识别的句柄。

由于与未支持扩展产生不兼容的风险，包装对象的 layer 必须检查应用程序正在使用哪些扩展，并在 layer 与不受支持的扩展一同使用时采取适当措施，例如向用户发出警告或错误消息。

验证层之所以会包装对象，是为了跟踪每个对象的正确使用和销毁。当它们与不受支持的扩展一起使用时，会发出验证错误，提醒用户存在潜在的未定义行为风险。

<a id="hash-maps"></a>
#### 哈希映射

另一种做法是，layer 可以使用哈希映射将数据与某个对象关联起来。映射的键可以直接就是该对象。或者，对于某一层级（例如 device 或 instance）的可分发对象，layer 也可能希望将数据关联到`VkDevice`或`VkInstance`对象上。但由于同一个`VkInstance`或`VkDevice`下会有多个可分发对象，因此`VkDevice`或`VkInstance`对象本身并不是很理想的映射键。更合适的做法是使用`VkDevice`或`VkInstance`内部的调度表指针，因为对于某个给定的`VkInstance`或`VkDevice`，这个指针是唯一的。

<a id="creating-new-dispatchable-objects"></a>
### 创建新的可分发对象

创建可分发对象的 layer 必须格外小心。请记住，loader 的 *trampoline* 代码通常会负责填充新创建对象中的调度表指针。因此，如果 loader 的 *trampoline* 不会执行这一步，layer 就必须自己填充该调度表指针。 layer（或驱动）可能在没有 loader *trampoline* 代码参与的情况下创建可分发对象，常见情形如下：
- 包装可分发对象的 layer
- 添加会创建可分发对象的扩展的 layer
- 在从应用程序拦截到的函数流中插入额外 Vulkan 函数调用的 layer
- 添加会创建可分发对象的扩展的驱动

Khronos loader 提供了一个可用于初始化可分发对象的回调函数。创建 instance（`VkInstanceCreateInfo`）或 device（`VkDeviceCreateInfo`）时，这个回调会作为扩展结构通过创建信息结构体中的`pNext`字段传入。该回调的原型分别针对 instance 和 device 定义如下（参见`vk_layer.h`）：

```cpp
VKAPI_ATTR VkResult VKAPI_CALL
   vkSetInstanceLoaderData(
      VkInstance instance,
      void *object);

VKAPI_ATTR VkResult VKAPI_CALL
   vkSetDeviceLoaderData(
      VkDevice device,
      void *object);
```

要获取这些回调，layer 必须遍历`VkInstanceCreateInfo`和`VkDeviceCreateInfo`参数中`pNext`字段所指向的结构体链表，以查找任何由 loader 插入的回调结构体。关键点如下：
- 对于`VkInstanceCreateInfo`，`pNext`指向的回调结构体是`VkLayerInstanceCreateInfo`，其定义位于`include/vulkan/vk_layer.h`中。
- `VkInstanceCreateInfo`参数中若存在`sType`字段值为`VK_STRUCTURE_TYPE_LOADER_INSTANCE_CREATE_INFO`，则表示这是 loader 结构体。
- 在`VkLayerInstanceCreateInfo`中，`function`字段用于指示联合字段`u`应如何解释。
- 若`function`等于`VK_LOADER_DATA_CALLBACK`，则表示`u`字段中的`pfnSetInstanceLoaderData`包含该回调。
- 对于`VkDeviceCreateInfo`，`pNext`指向的回调结构体是`VkLayerDeviceCreateInfo`，其定义位于`include/vulkan/vk_layer.h`中。
- `VkDeviceCreateInfo`参数中若存在`sType`字段值为`VK_STRUCTURE_TYPE_LOADER_DEVICE_CREATE_INFO`，则表示这是 loader 结构体。
- 在`VkLayerDeviceCreateInfo`中，`function`字段用于指示联合字段`u`应如何解释。
- 若`function`等于`VK_LOADER_DATA_CALLBACK`，则表示`u`字段中的`pfnSetDeviceLoaderData`包含该回调。

另一种情况是，如果使用的是较旧的 loader，而它并不提供这些回调，layer 也可以手动初始化新创建的可分发对象。要为新创建的可分发对象填充调度表指针，layer 应从同层级（instance 或 device）的现有父对象中复制调度指针；该指针始终是结构体中的第一个成员。

例如，如果新创建了一个`VkCommandBuffer`对象，那么应将其父对象`VkDevice`中的调度指针复制到这个新创建的对象中。

<a id="versioning-and-activation-interactions"></a>
### 版本控制与激活的交互

关于 layer 的激活有若干彼此交互的规则，其结果并不总是显而易见。下面并非完整列表，但有助于更清楚地说明 loader 在复杂场景中的行为。

* 1.3.228 及以上版本的 Vulkan Loader 会启用隐式层，而不考虑应用程序在`VkApplicationInfo::apiVersion`中指定的 API 版本。此前版本的 loader（1.3.227 及以下）曾要求：只有当隐式层的 API 版本大于或等于应用程序的 API 版本时，该 layer 才会被启用。之所以放宽隐式层的加载要求，是因为人们认为，阻止旧 layer 在新应用程序中运行这一看似提供保护的做法，并不足以抵消它带来的阻力。这是因为旧 layer 往往会在没有明显原因的情况下无法与较新的应用程序一起工作，而且旧 layer 还必须更新清单文件才能与较新的应用程序配合工作。除此之外，layer 不需要做任何其他事情就能重新恢复工作，这意味着 layer 并不需要真正证明自己能够在较新的 API 版本上正常工作。因此，这种禁用机制会让用户感到困惑，却并不能保护他们免受潜在行为异常的 layer 的影响。

* 如果某个隐式层是某个已激活元层中的组成部分，那么即使设置了它的禁用环境变量，它也会忽略该设置。

* 环境变量`VK_LAYER_PATH`只影响显式层搜索，不影响隐式层。在该路径下发现的 layer 都会被视为显式层，即使它们包含了成为隐式层所需的全部字段也是如此。这意味着它们不会被隐式启用。

* 元层不一定必须是隐式层——它们也可以是显式层。因此，不能因为某个元层存在，就假定它一定会处于激活状态。

* 覆盖元层中的`blacklisted_layers`成员会阻止隐式启用和显式启用的 layer 激活。应用程序的`VkInstanceCreateInfo::ppEnabledLayerNames`中凡是出现在黑名单里的 layer，都不会被启用。

* 覆盖元层中的`app_keys`成员会使某个元层仅适用于该列表中的应用程序。如果`app_keys`列表中存在任何项，那么该元层只会对列表中的应用程序启用，对其他应用程序都不会启用。

* 如果覆盖元层中存在`override_paths`成员，它将替换 loader 用来查找组成 layer 的搜索路径。如果任一组成 layer 不在这些覆盖路径中，覆盖元层就不会被应用。因此，如果某个覆盖元层希望同时混合默认 layer 位置和自定义 layer 位置，那么`override_paths`中必须同时包含默认 layer 位置和自定义 layer 位置。

* 如果覆盖 layer 存在且包含`override_paths`，那么在搜索显式层时，会忽略环境变量`VK_LAYER_PATH`中的路径。例如，当元层覆盖路径和`VK_LAYER_PATH`同时存在时，`VK_LAYER_PATH`中的所有 layer 都将无法被发现，loader 也就无法找到它们。

<a id="layer-manifest-file-format"></a>
## Layer 清单文件格式

Khronos loader 使用清单文件来发现可用的 layer 库和 Layer。除构建调用链期间外，它不会直接查询 layer 的动态库。这样做是为了降低将恶意 layer 加载到内存中的可能性。 loader 会改为从清单文件中读取详细信息，再将这些信息提供给应用程序，用于判断实际应加载哪些 Layer。

下面的小节将讨论 Layer 清单 JSON 文件格式的细节。 JSON 文件本身对命名没有要求。唯一的要求是文件扩展名后缀必须为`.json`。

下面是一个仅包含单个 Layer 的 layer JSON 清单文件示例：

```json
{
   "file_format_version" : "1.2.1",
   "layer": {
       "name": "VK_LAYER_LUNARG_overlay",
       "type": "INSTANCE",
       "library_path": "vkOverlayLayer.dll",
       "library_arch" : "64",
       "api_version" : "1.0.5",
       "implementation_version" : "2",
       "description" : "LunarG HUD layer",
       "functions": {
           "vkNegotiateLoaderLayerInterfaceVersion":
               "OverlayLayer_NegotiateLoaderLayerInterfaceVersion"
       },
       "instance_extensions": [
           {
               "name": "VK_EXT_debug_report",
               "spec_version": "1"
           },
           {
               "name": "VK_VENDOR_ext_x",
               "spec_version": "3"
            }
       ],
       "device_extensions": [
           {
               "name": "VK_EXT_debug_marker",
               "spec_version": "1",
               "entrypoints": ["vkCmdDbgMarkerBegin", "vkCmdDbgMarkerEnd"]
           }
       ],
       "enable_environment": {
           "ENABLE_LAYER_OVERLAY_1": "1"
       },
       "disable_environment": {
           "DISABLE_LAYER_OVERLAY_1": ""
       }
   }
}
```

下面的片段展示了在每个清单文件中支持多个 Layer 所需的变更：

```json
{
   "file_format_version" : "1.0.1",
   "layers": [
      {
           "name": "VK_LAYER_layer_name1",
           "type": "INSTANCE",
           ...
      },
      {
           "name": "VK_LAYER_layer_name2",
           "type": "INSTANCE",
           ...
      }
   ]
}
```

下面是一个元层清单文件示例：

```json
{
   "file_format_version" : "1.1.1",
   "layer": {
       "name": "VK_LAYER_META_layer",
       "type": "GLOBAL",
       "api_version" : "1.0.40",
       "implementation_version" : "1",
       "description" : "LunarG Meta-layer example",
       "component_layers": [
           "VK_LAYER_KHRONOS_validation",
           "VK_LAYER_LUNARG_api_dump"
       ]
   }
}
```

<table style="width:100%">
  <tr>
    <th>JSON 节点</th>
    <th>说明与注释</th>
    <th>限制</th>
    <th>父节点</th>
    <th>自省查询</th>
  </tr>
  <tr>
    <td>"api_version"</td>
    <td>该 Layer 所支持的 Vulkan API 主.次.补丁版本号。
        这并不要求应用程序必须使用该 API 版本。
        它只是表明该 Layer 能够支持直到并包括该 API 版本的 Vulkan API
        instance 和 device 函数。</br>
        例如：1.0.33。
    </td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerateInstanceLayerProperties</small></td>
  </tr>
  <tr>
    <td>"app_keys"</td>
    <td>该元层适用的可执行文件路径列表。
    </td>
    <td><b>仅元层</b></td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"blacklisted_layers"</td>
    <td>显式 layer 名称列表。即使应用程序请求加载这些 layer，也不应加载。
    </td>
    <td><b>仅元层</b></td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"component_layers"</td>
    <td>指示属于某个元层的组件 layer 名称。
        列出的名称必须是每个组件 layer 清单文件中 "name" 标签所标识的
        "name"（这也就是传给 `vkCreateInstance` 命令的 layer 名称）。
        只有当所有组件 layer 都存在于系统中，且均能被 loader 找到时，
        此元层才可用并可激活。<br/>
        <b>如果定义了 "library_path"，则不得出现此字段</b>。
    </td>
    <td><b>仅元层</b></td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"description"</td>
    <td>对该 Layer 及其预期用途的高层描述。</td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerateInstanceLayerProperties</small></td>
  </tr>
  <tr>
    <td>"device_extensions"</td>
    <td><b>可选：</b>包含该 Layer 支持的 device 扩展名称列表。
        如果某个 Layer 支持任何 device 扩展，则必须有一个
        "device_extensions" 节点，其数组中包含一个或多个元素；否则该节点可选。
        数组中的每个元素都必须包含 "name" 和 "spec_version" 节点，
        分别对应 `VkExtensionProperties` 的 "extensionName" 和
        "specVersion"。
        此外，如果某个 device 扩展增加了 Vulkan API 函数，
        那么该 device 扩展数组中的每个元素都必须包含 "entrypoints" 节点；
        否则该节点不是必需的。
        "entrypoint" 节点是一个数组，包含该受支持扩展新增的所有入口点名称。
    </td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerateDeviceExtensionProperties</small></td>
  </tr>
  <tr>
    <td>"disable_environment"</td>
    <td><b>必需：</b>指示一个用于禁用隐式层的环境变量
        （当其被定义为任意非空字符串值时）。<br/>
        在少数应用程序无法与某个隐式层协同工作的情况下，
        应用程序可以设置该环境变量（在调用 Vulkan 函数之前），
        以将该 layer “加入黑名单”。
        该环境变量（不同变体的 layer 可能不同）必须被设置
        （不特定要求某个值）。
        如果同时设置了 "enable_environment" 和 "disable_environment" 变量，
        则该隐式层会被禁用。
    </td>
    <td><b>仅隐式层</b></td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"enable_environment"</td>
    <td><b>可选：</b>指示一个用于启用隐式层的环境变量
        （当其被定义为任意非空字符串值时）。<br/>
        该环境变量（不同变体的 layer 可能不同）必须被设置为给定值，
        否则不会加载该隐式层。
        这适用于某些应用环境（例如 Steam），它们希望只对自己启动的应用启用
        某个或某些 layer，并使在该应用环境之外运行的应用不会获得这些隐式层。
    </td>
    <td><b>仅隐式层</b></td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"file_format_version"</td>
    <td>清单格式的主.次.补丁版本号。<br/>
        支持的版本有：1.0.0、1.0.1、1.1.0、1.1.1、1.1.2 和 1.2.0。
    </td>
    <td>无</td>
    <td>无</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"functions"</td>
    <td><b>可选：</b>该部分可用于指定一个不同的函数名，
        让 loader 用它代替标准 Layer 接口函数。
        如果 Layer 为 `vkNegotiateLoaderLayerInterfaceVersion` 使用了替代名称，
        则必须提供 "functions" 节点。
    </td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkGet*ProcAddr</small></td>
  </tr>
  <tr>
    <td>"implementation_version"</td>
    <td>该 Layer 的实现版本。
        如果 Layer 自身有任何重大变更，则应修改该数字，
        以便 loader 和/或应用程序能够正确识别它。
    </td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerateInstanceLayerProperties</small></td>
  </tr>
  <tr>
    <td>"instance_extensions"</td>
    <td><b>可选：</b>包含该 Layer 支持的 instance 扩展名称列表。
        如果某个 Layer 支持任何 instance 扩展，则必须有一个
        "instance_extensions" 节点，其数组中包含一个或多个元素；否则该节点可选。
        数组中的每个元素都必须包含 "name" 和 "spec_version" 节点，
        分别对应 `VkExtensionProperties` 的 "extensionName" 和
        "specVersion"。
    </td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerateInstanceExtensionProperties</small></td>
  </tr>
  <tr>
    <td>"layer"</td>
    <td>用于将单个 Layer 的信息组织在一起的标识符。
    </td>
    <td>无</td>
    <td>无</td>
    <td><small>vkEnumerateInstanceLayerProperties</small></td>
  </tr>
  <tr>
    <td>"layers"</td>
    <td>用于将多个 Layer 的信息组织在一起的标识符。
        这要求清单文件格式版本至少为 1.0.1。
    </td>
    <td>无</td>
    <td>无</td>
    <td><small>vkEnumerateInstanceLayerProperties</small></td>
  </tr>
  <tr>
    <td>"library_path"</td>
    <td>指定 layer 共享库文件的文件名、相对路径名或完整路径名。
        如果 "library_path" 指定的是相对路径名，则它相对于 JSON 清单文件的路径
        （例如应用程序提供的 layer 与其他应用文件位于同一文件夹层级时）。
        如果 "library_path" 指定的是文件名，则该库必须位于系统的共享对象搜索路径中。
        对 layer 共享库文件的名称没有特殊规则，只要求以适当后缀结尾
        （Windows 上为 ".DLL"，Linux 上为 ".so"，macOS 上为 ".dylib"）。<br/>
        <b>如果定义了 "component_layers"，则不得出现此字段</b>。
    </td>
    <td><b>对元层无效</b></td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"library_arch"</td>
    <td>可选字段，用于指定与 "library_path" 关联的二进制文件架构。<br />
        它允许 loader 快速判断该 Layer 的架构是否与当前运行的应用程序匹配。<br />
        仅有效的值为 "32" 和 "64"。</td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"name"</td>
    <td>应用程序用于唯一标识该 Layer 的字符串。</td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerateInstanceLayerProperties</small></td>
  </tr>
  <tr>
    <td>"override_paths"</td>
    <td>将用作组件 layer 搜索位置的路径列表。
    </td>
    <td><b>仅元层</b></td>
    <td>"layer"/"layers"</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td>"pre_instance_functions"</td>
    <td><b>可选：</b>指示该 Layer 希望拦截、且不要求已创建 instance 的函数。
        这应为一个对象，其中每个要拦截的函数都定义为一个字符串条目：
        键为 Vulkan 函数名，值为该 Layer 动态库中的拦截函数名。
        该字段在 Layer 清单版本 1.1.2 及以上版本中可用。<br/>
        更多信息请参见 <a href="#pre-instance-functions">预实例函数</a>。
    </td>
    <td><b>仅隐式层</b></td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerateInstance*Properties</small></td>
  </tr>
  <tr>
    <td>"type"</td>
    <td>此字段指示 layer 的类型。取值可以是：GLOBAL 或 INSTANCE。<br/>
        <b>注意：</b>在弃用之前，"type" 节点用于指示应在哪些 layer 链中激活该 layer：
        instance、device，或两者。
        独立的 instance layer 和 device layer 已被弃用；现在只剩 instance layer。
        最初允许的值是 "INSTANCE"、"GLOBAL" 和 "DEVICE"。
        但现在 loader 会像未找到一样直接跳过 "DEVICE" layer。
    </td>
    <td>无</td>
    <td>"layer"/"layers"</td>
    <td><small>vkEnumerate*LayerProperties</small></td>
  </tr>
</table>

<a id="layer-manifest-file-version-history"></a>
### Layer 清单文件版本历史

当前支持的最高 Layer 清单文件格式版本是 1.2.0。各版本的详细信息见下面的小节：

<a id="layer-manifest-file-version-121"></a>
### Layer 清单文件版本 1.2.1

向 layer 清单中添加了`library_arch`字段，以便 loader 能快速判断该 Layer 是否与当前运行应用程序的架构匹配。

<a id="layer-manifest-file-version-120"></a>
#### Layer 清单文件版本 1.2.0

增加了定义 layer 设置的能力，如[layer manifest schema](https://github.com/LunarG/VulkanTools/blob/main/vkconfig_core/layers/layers_schema.json) 所定义。

还可通过以下字段对 layer 进行简要说明：
 * `introduction`：用一段文字介绍该 layer 的用途。
 * `url`：指向 layer 主页的链接。
 * `platforms`：该 layer 支持的平台列表
 * `status`：该 layer 的生命周期状态：Alpha、Beta、Stable 或 Deprecated

这些变更是为了让第三方 layer 能够在[Vulkan Configurator](https://github.com/LunarG/VulkanTools/blob/main/vkconfig/README.md) 或其他工具中暴露其功能。

<a id="layer-manifest-file-version-112"></a>
#### Layer 清单文件版本 1.1.2

1.1.2 版本引入了让 Layer 拦截那些不带 instance 的函数调用的能力。

<a id="layer-manifest-file-version-111"></a>
#### Layer 清单文件版本 1.1.1

增加了定义自定义元层的能力。为支持元层，添加了`component_layers`部分，并且当存在`component_layers`部分时，不再要求必须存在`library_path`部分。

<a id="layer-manifest-file-version-110"></a>
#### Layer 清单文件版本 1.1.0

Layer 清单文件版本 1.1.0 与 Loader/Layer 接口版本 2 暴露出的变更相关联。
  1. `functions`部分中对`vkGetInstanceProcAddr`的重命名已被弃用，因为 loader 不再需要直接向 Layer 查询`vkGetInstanceProcAddr`。它现在会在 layer 协商期间返回，因此该字段将被忽略。
  2. `functions`部分中对`vkGetDeviceProcAddr`的重命名已被弃用，因为 loader 不再需要直接向 Layer 查询`vkGetDeviceProcAddr`。它同样会在 layer 协商期间返回，因此该字段将被忽略。
  3. 增加了在`functions`部分中重命名`vkNegotiateLoaderLayerInterfaceVersion`函数的能力，因为这现在是 loader 唯一需要通过操作系统特定调用来查询的函数。
      - 注意：这是一个可选字段，并且与前两个字段一样，仅当 Layer 因某种原因需要更改该函数名称时才需要它。

如果所列函数的名称没有变化，则无需更新 layer 清单文件。

<a id="layer-manifest-file-version-101"></a>
#### Layer 清单文件版本 1.0.1

增加了使用`layers`数组定义多个 Layer 的能力。在定义单个 Layer 或多个 Layer 时，都可以使用这个 JSON 数组字段。对于单个 Layer 定义，`layer`字段仍然存在且有效。

<a id="layer-manifest-file-version-100"></a>
#### Layer 清单文件版本 1.0.0

Layer 清单文件的初始版本定义了 layer JSON 文件的基本格式和字段。 1.0.0 文件格式中的字段包括：
 * `file_format_version`
 * `layer`
 * `name`
 * `type`
 * `library_path`
 * `api_version`
 * `implementation_version`
 * `description`
 * `functions`
 * `instance_extensions`
 * `device_extensions`
 * `enable_environment`
 * `disable_environment`

也是在这一时期，`type`字段中的`DEVICE`值被弃用了。

<a id="layer-interface-versions"></a>
## Layer 接口版本

当前的 loader/layer 接口版本为 2。下面的小节详细说明了各版本之间的差异。

<a id="layer-interface-version-2"></a>
### Layer 接口版本 2

引入了通过`vkNegotiateLoaderLayerInterfaceVersion`函数进行[loader 与 layer 接口](#layer-version-negotiation)协商的概念。此外，还引入了[Layer 未知物理设备扩展](#layer-unknown-physical-device-extensions) 以及相关的`vk_layerGetPhysicalDeviceProcAddr`函数。最后，它将清单文件定义更新为 1.1.0。

注意：如果某个 Layer 包装了`VkInstance`句柄，则`vk_layerGetPhysicalDeviceProcAddr`的支持*不是*可选项，必须实现。

<a id="layer-interface-version-1"></a>
### Layer 接口版本 1

支持接口版本 1 的 Layer 具有以下行为：
  1. 直接导出`vkGetInstanceProcAddr`和`vkGetDeviceProcAddr`
  2. layer 清单文件可以覆盖`GetInstanceProcAddr`和`GetDeviceProcAddr`函数的名称。

<a id="layer-interface-version-0"></a>
### Layer 接口版本 0

支持接口版本 0 的 Layer 必须定义并导出以下这些自省函数。尽管它们的名称、签名以及其他方面与 Vulkan 函数相似，但它们与任何 Vulkan 函数都无关：

- `vkEnumerateInstanceLayerProperties`：枚举`Layer Library`中的所有 layer。
  - 此函数永不失败。
  - 当`Layer Library`只包含一个 layer 时，该函数可以是该 layer 的`vkEnumerateInstanceLayerProperties`的别名。
- `vkEnumerateInstanceExtensionProperties`：枚举`Layer Library`中 layer 的 instance 扩展。
  - `pLayerName`始终是有效的 layer 名称。
  - 此函数永不失败。
  - 当`Layer Library`只包含一个 layer 时，该函数可以是该 layer 的`vkEnumerateInstanceExtensionProperties`的别名。
- `vkEnumerateDeviceLayerProperties`：枚举`Layer Library`中 layer 的一个子集（可以是全集、真子集或空子集）。
  - `physicalDevice`始终为`VK_NULL_HANDLE`。
  - 此函数永不失败。
  - 如果某个 layer 未被此函数枚举到，它将不会参与 device 函数拦截。
- `vkEnumerateDeviceExtensionProperties`：枚举`Layer Library`中 layer 的 device 扩展。
  - `physicalDevice`始终为`VK_NULL_HANDLE`。
  - `pLayerName`始终是有效的 layer 名称。
  - 此函数永不失败。

它还必须为库中的每个 Layer 各定义并导出一次以下函数：

- `<layerName>GetInstanceProcAddr(instance, pName)`的行为与某个 layer 的`vkGetInstanceProcAddr`完全一致，只是它是导出的。

当`Layer Library`只包含一个 layer 时，该函数也可以改名为`vkGetInstanceProcAddr`。

- `<layerName>GetDeviceProcAddr`的行为与某个 layer 的`vkGetDeviceProcAddr`完全一致，只是它是导出的。

当`Layer Library`只包含一个 layer 时，该函数也可以改名为`vkGetDeviceProcAddr`。

库中包含的所有 Layer 都必须支持`vk_layer.h`。它们不需要实现自己未拦截的函数。建议它们不要导出任何函数。

<a id="loader-and-layer-interface-policy"></a>
## Loader 与 Layer 接口策略

本节旨在定义 loader 与 Layer 之间应遵循的正确行为。本节的大部分内容是对 Vulkan 规范的补充，并且对于保持跨平台一致性是必要的。实际上，本文档中许多地方都能看到相关表述，这里只是为了方便统一汇总。此外，还应当有办法识别 layer 中不良或不符合规范的行为，并尽快加以纠正。因此，这里提供了一套策略编号系统，用于以唯一方式清晰标识每条策略声明。

最后，基于让 loader 高效且高性能的目标，这里定义的某些 Layer 正确行为策略可能无法测试（因此 loader 也无法强制执行）。但这并不应削弱这些要求本身对最终用户和开发者体验的重要性。

<a id="number-format"></a>
### 编号格式

Loader/Layer 策略项以`LLP_`为前缀（即 Loader/Layer Policy 的缩写），后面跟一个基于该策略针对哪个组件的标识符。在这里，只有两个可能的组件：
  - Layer：其策略编号中会包含字符串`LAYER_`。
  - Loader：其策略编号中会包含字符串`LOADER_`。

<a id="android-differences"></a>
### Android 差异

如前所述，Android Loader 实际上与 Khronos Loader 是分离的。正因如此，再加上其他平台要求，这些策略声明并不全部适用于 Android。每个表还都有一列名为“适用于 Android 吗？” 它用于指示哪些策略声明适用于只面向 Android 支持的 Layer。有关 Android loader 的更多信息，请参见 <a href="https://source.android.com/devices/graphics/implement-vulkan"> Android Vulkan 文档</a>。

<a id="requirements-of-well-behaved-layers"></a>
### 行为良好的 Layer 的要求

<table style="width:100%">
  <tr>
    <th>要求编号</th>
    <th>要求说明</th>
    <th>不符合要求的结果</th>
    <th>适用于 Android 吗？</th>
    <th>可由 Loader 强制执行吗？</th>
    <th>参考章节</th>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_1</b></small></td>
    <td>一个 Layer 在被插入到原本符合规范的 Vulkan 环境中时，<b>必须</b>
        仍然使该环境保持符合规范，除非它本来就打算模拟不符合规范的行为
        （例如设备模拟 layer）。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td>否<br/>
        loader 很难在 layer 调用链中简单定位故障根因。</td>
    <td><small>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_2</b></small></td>
    <td>Layer <b>不得</b>导致其他 layer 或驱动失败、崩溃或出现其他异常行为。<br/>
        它<b>不得</b>向其下层的 layer 或驱动作出无效调用，也<b>不得</b>依赖其未定义行为。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td>否<br/>
        loader 很难在 layer 调用链中简单定位故障根因。</td>
    <td><small>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_3</b></small></td>
    <td>任何新开发的 Layer <b>应当</b>遵循“Layer 约定与规则”一节中定义的命名规则，
        这些规则也与 Vulkan Style Guide 第 3.4 节
        “Version, Extension, and Layer Naming Conventions” 中定义的命名规则一致。
    </td>
    <td>Layer 开发者可能会产生冲突名称，导致在用户平台上存在多个同名 layer 时出现意外行为。
    </td>
    <td>是</td>
    <td>是<br/>
        目前无法立即强制执行，因为这样会导致一些已发布的 layer 停止工作。</td>
    <td><small>
        <a href="https://www.khronos.org/registry/vulkan/specs/1.2/styleguide.html#extensions-naming-conventions">
            Vulkan Style Guide 第 3.4 节</a> <br/>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_4</b></small></td>
    <td>Layer <b>应当</b>导出
        <i>vkNegotiateLoaderLayerInterfaceVersion</i> 入口点，以协商接口版本。<br/>
        使用接口版本 2 或更新版本的 Layer <b>必须</b>导出此函数。<br/>
    </td>
    <td>该 Layer 将不会被加载。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#layer-version-negotiation">Layer 版本协商</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_5</b></small></td>
    <td>Layer <b>必须</b>能够按照规定的协商流程，与 loader 协商出一个受支持的
        loader/layer 接口版本。
    </td>
    <td>该 Layer 将不会被加载。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#layer-version-negotiation">
        接口协商</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_6</b></small></td>
    <td>Layer <b>必须</b>提供一个有效的 JSON 清单文件供 loader 处理，
        且文件名必须以 `.json` 后缀结尾。
        建议在发布前，针对
        <a href="https://github.com/LunarG/VulkanTools/blob/main/vkconfig_core/layers/layers_schema.json">
        layer schema</a> 验证 layer 清单文件。</br>
        <b>唯一</b>的例外是在 Android 上，它通过
        <a href="#layer-interface-version-0">Layer 接口版本 0</a>
        一节以及
        <a href="#layer-manifest-file-format">Layer 清单文件格式</a>
        表中定义的自省函数来确定 layer 功能。
    </td>
    <td>该 Layer 将不会被加载。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#layer-manifest-file-usage">清单文件用法</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_7</b></small></td>
    <td>如果某个 Layer 是元层，则其清单文件中的每个组件 layer <b>必须</b>
        存在于系统中。
    </td>
    <td>该 Layer 将不会被加载。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#meta-layers">元层</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_8</b></small></td>
    <td>如果某个 Layer 是元层，则其清单文件中的每个组件 layer <b>必须</b>
        报告与该元层相同或更新的 Vulkan API 主版本和次版本。
    </td>
    <td>该 Layer 将不会被加载。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#meta-layers">元层</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_9</b></small></td>
    <td>如果某个 Layer 作为隐式层安装，则它<b>必须</b>定义一个禁用环境变量，
        以便能够被全局禁用。
    </td>
    <td>如果它未定义该环境变量，则该 Layer 不会被加载。
    </td>
    <td>是</td>
    <td>是</td>
    <td><small>
        <a href="#layer-manifest-file-format">清单文件格式</a>，参见
        `disable_environment` 变量</small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_10</b></small></td>
    <td>如果某个 Layer 包装了单个对象句柄，则它在将这些句柄沿调用链向下传递给下一个 layer 时，
        <b>必须</b>先将它们解包。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    </td>
    <td>是</td>
    <td>否</td>
    <td><small>
      <a href="#cautions-about-wrapping">关于包装的注意事项</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_11</b></small></td>
    <td>任何与驱动一同发布的 Layer <b>必须</b>针对相应驱动完成一致性验证。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td>否</td>
    <td><small>
        <a href="https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/openglcts/README.md">
        Vulkan CTS 文档</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_12</b></small></td>
    <td>在 <i>vkCreateInstance</i> 期间，Layer <b>必须</b>正确处理
         <i>VkLayerInstanceCreateInfo</i> 链接。<br/>
         这包括获取下一个 layer 的 <i>vkGetInstanceProcAddr</i>
         函数以构建调度表，以及在向下调用下一个 layer 的
         <i>vkCreateInstance</i> 函数之前，将
         <i>VkLayerInstanceCreateInfo</i> 链接更新为指向链中的下一个结构体。<br/>
         这种用法的详细示例见
         <a href=#example-code-for-createinstance>CreateInstance 示例代码</a>
         一节。
    </td>
    <td>这种行为会导致崩溃或数据损坏，因为后续所有 layer 都会访问到错误内容。</td>
    <td>是</td>
    <td>否<br/>
        在当前 loader/layer 设计下，loader 很难在不增加可能影响性能的额外开销的情况下诊断此问题。<br/>
        这是因为 loader 会一次调用所有 layer，而无法获得 <i>pNext</i>
        链内容的中间状态数据。
        未来或许可以做到，但这需要重新设计 layer 初始化过程。
    </td>
    <td><small>
        <a href="#layer-dispatch-initialization">
           Layer 调度初始化</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_13</b></small></td>
    <td>在 <i>vkCreateDevice</i> 期间，Layer <b>必须</b>正确处理
         <i>VkLayerDeviceCreateInfo</i> 链接。<br/>
         这包括在向下调用下一个 layer 的 <i>vkCreateDevice</i> 函数之前，
         将 <i>VkLayerDeviceCreateInfo</i> 链接更新为指向链中的下一个结构体。<br/>
         这种用法的详细示例见
         <a href="#example-code-for-createdevice">CreateDevice 示例代码</a>
         一节。
    </td>
    <td>这种行为会导致崩溃或数据损坏，因为后续所有 layer 都会访问到错误内容。</td>
    <td>是</td>
    <td>否<br/>
        在当前 loader/layer 设计下，loader 很难在不增加可能影响性能的额外开销的情况下诊断此问题。</td>
    <td><small>
        <a href="#layer-dispatch-initialization">
           Layer 调度初始化</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_14</b></small></td>
    <td>当应用程序提供了内存分配器函数时，Layer <b>应当</b>使用这些函数，
        以便应用程序能够跟踪所分配的内存。
    </td>
    <td>这些分配器函数可能用于限制或跟踪 Vulkan 组件所使用的内存。
        因此，如果某个 Layer 忽略这些分配器，可能导致未定义行为，
        包括崩溃或数据损坏。
    </td>
    <td>是</td>
    <td>否</td>
    <td><small>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_15</b></small></td>
    <td>当 <i>pLayerName</i> 指向自身时，Layer 在调用
        <i>vkEnumerateInstanceExtensionProperties</i> 时<b>必须</b>
        只枚举它自己的扩展属性。<br/>
        否则，它<b>必须</b>返回 <i>VK_ERROR_LAYER_NOT_PRESENT</i>，
        包括 <i>pLayerName</i> 为 <b>NULL</b> 的情况。
    </td>
    <td>loader 可能会对某个特定 Layer 实际支持的内容产生混淆，
        从而导致未定义行为，包括崩溃或数据损坏。
    </td>
    <td>是</td>
    <td>否</td>
    <td><small>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_16</b></small></td>
    <td>当 <i>pLayerName</i> 指向自身时，Layer 在调用
        <i>vkEnumerateDeviceExtensionProperties</i> 时<b>必须</b>
        只枚举它自己的扩展属性。<br/>
        否则，除了按标准调用链向下传递之外，它<b>必须</b>忽略该调用。
    </td>
    <td>loader 可能会对某个特定 Layer 实际支持的内容产生混淆，
        从而导致未定义行为，包括崩溃或数据损坏。
    </td>
    <td>是</td>
    <td>否</td>
    <td><small>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_17</b></small></td>
    <td>Layer 的 <i>vkCreateInstance</i> <b>不得</b>因无法识别的扩展名称而报错，
        因为该扩展可能由更下层的 layer 或驱动实现。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td>是</td>
    <td><small>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_18</b></small></td>
    <td>对于其不支持的入口点，或尚未正确启用的入口点，
        Layer <b>必须</b>从 <i>vkGetInstanceProcAddr</i> 或
        <i>vkGetDeviceProcAddr</i> 返回 <b>NULL</b>
        （例如，若未启用某些入口点所属的扩展，则请求这些入口点时，
        <i>vkGetInstanceProcAddr</i> 应返回 <b>NULL</b>）。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td>否<br/>
        在当前 loader/layer 设计下，loader 很难在不增加可能影响性能的额外开销的情况下判断这一点。</td>
    <td><small>
        <a href="#layer-conventions-and-rules">Layer 约定与规则</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_19</b></small></td>
    <td>如果某个 Layer 创建了可分发对象，
        无论是因为它包装对象，还是实现了 loader 或底层驱动不支持的扩展，
        它都<b>必须</b>为所有创建出的可分发对象正确创建调度表。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td>否</td>
    <td><small>
        <a href="#creating-new-dispatchable-objects">
          创建新的可分发对象</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_20</b></small></td>
    <td>Layer 在卸载时<b>必须</b>移除所有清单文件以及对这些文件的引用
        （例如 Windows 上的注册表项）。<br/>
        同样，在更新 Layer 文件时，旧文件<b>必须</b>全部更新或移除。
    </td>
    <td>loader 会忽略加载同一清单文件的重复尝试，但如果旧文件仍指向错误的库，
        将导致未定义行为，包括崩溃或数据损坏。
    </td>
    <td>否</td>
    <td>否<br/>
        loader 无法知道哪些 layer 文件是新的、旧的或不正确的。
        任何类型的 layer 文件验证都会迅速变得非常复杂，
        因为这要求 loader 维护一个内部数据库，根据 layer 名称、版本、
        目标平台以及可能的其他标准来跟踪行为不良的 layer。
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td><small><b>LLP_LAYER_21</b></small></td>
    <td>在 <i>vkCreateInstance</i> 期间，Layer <b>不得</b>在向下调用更低层之前修改
        <i>pInstance</i> 指针。<br/>
        这是因为 loader 会通过该指针传递 loader 终止器函数初始化代码所需的信息。<br/>
        因此，如果 Layer 要覆盖 <i>pInstance</i> 指针，则<b>必须</b>仅在调用下层返回后再进行。
    </td>
    <td>loader 很可能会崩溃。</td>
    <td>否</td>
    <td>是</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
  <td><small><b>LLP_LAYER_22</b></small></td>
    <td>在 <i>vkCreateDevice</i> 期间，Layer <b>不得</b>在向下调用更低层之前修改
        <i>pDevice</i> 指针。<br/>
        这是因为 loader 会通过该指针传递 loader 终止器函数初始化代码所需的信息。<br/>
        因此，如果 Layer 要覆盖 <i>pDevice</i> 指针，则<b>必须</b>仅在调用下层返回后再进行。
    </td>
    <td>loader 很可能会崩溃。</td>
    <td>否</td>
    <td>是</td>
    <td><small>N/A</small></td>
  </tr>
</table>

<a id="requirements-of-a-well-behaved-loader"></a>
### 行为良好的 Loader 的要求

<table style="width:100%">
  <tr>
    <th>要求编号</th>
    <th>要求说明</th>
    <th>不符合要求的结果</th>
    <th>适用于 Android 吗？</th>
    <th>参考章节</th>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_1</b></small></td>
    <td>Loader <b>必须</b>支持 Vulkan Layer。</td>
    <td>用户将无法使用 Vulkan 生态中的关键部分，例如 Validation Layers、
        GfxReconstruct 或 RenderDoc。</td>
    <td>是</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_2</b></small></td>
    <td>Loader <b>必须</b>支持一种机制，以便从一个或多个非标准位置加载 layer。<br/>
        这是为了支持应用程序/引擎专用的 layer，以及在不全局安装的情况下评估开发中的 layer。
    </td>
    <td>这会使某些工具和驱动开发者更难使用 Vulkan loader。</td>
    <td>否</td>
    <td><small><a href="#layer-discovery">Layer 发现</a></small></td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_3</b></small></td>
    <td>Loader <b>必须</b>过滤掉各种启用列表中的重复 layer 名称，仅保留首次出现的项。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td><small><a href="#layer-discovery">Layer 发现</a></small></td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_4</b></small></td>
    <td>Loader <b>不得</b>加载那些定义了与自身不兼容 API 版本的 Vulkan Layer。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>是</td>
    <td><small><a href="#layer-discovery">Layer 发现</a></small></td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_5</b></small></td>
    <td>对于无法协商出兼容接口版本的 Layer，Loader <b>必须</b>忽略它。
    </td>
    <td>loader 会错误加载该 Layer，从而导致未定义行为，包括崩溃或数据损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#layer-version-negotiation">
        接口协商</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_6</b></small></td>
    <td>如果某个 Layer 是隐式层，且它带有启用环境变量，
        则除非该启用环境变量已定义，否则 Loader <b>不得</b>将该 Layer 视为已启用。<br/>
        如果某个隐式层没有启用环境变量，则默认视为已启用。
    </td>
    <td>某些 layer 可能会在非预期情况下被使用。</td>
    <td>否</td>
    <td><small>
        <a href="#layer-manifest-file-format">清单文件格式</a>，参见
        `enable_environment` 变量</small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_7</b></small></td>
    <td>如果某个隐式层已启用，但又被其他机制禁用
        （例如定义了该 layer 的禁用环境变量，或通过 Override Layer 的黑名单机制禁用），
        那么 Loader <b>不得</b>加载该 Layer。
    </td>
    <td>某些 layer 可能会在非预期情况下被使用。</td>
    <td>否</td>
    <td><small>
        <a href="#layer-manifest-file-format">清单文件格式</a>，参见
        `disable_environment` 变量</small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_8</b></small></td>
    <td>Loader <b>必须</b>通过 <i>VkInstanceCreateInfo</i> 结构体
        <i>pNext</i> 字段中的 <i>VkLayerInstanceCreateInfo</i> 结构体，
        向每个 Layer 传递一个初始化结构体链表。
        其中包含建立 instance 调用链所需的信息，包括提供指向下一链路
        <i>vkGetInstanceProcAddr</i> 的函数指针。
    </td>
    <td>Layer 在尝试加载无效数据时会崩溃。</td>
    <td>是</td>
    <td><small>
        <a href="#layer-dispatch-initialization">
           Layer 调度初始化</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_9</b></small></td>
    <td>Loader <b>必须</b>通过 <i>VkDeviceCreateInfo</i> 结构体
        <i>pNext</i> 字段中的 <i>VkLayerDeviceCreateInfo</i> 结构体，
        向每个 Layer 传递一个初始化结构体链表。
        其中包含建立 device 调用链所需的信息，包括提供指向下一链路
        <i>vkGetDeviceProcAddr</i> 的函数指针。
    </td>
    <td>Layer 在尝试加载无效数据时会崩溃。</td>
    <td>是</td>
    <td><small>
        <a href="#layer-dispatch-initialization">
           Layer 调度初始化</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_10</b></small></td>
    <td>在加载元层之前，Loader <b>必须</b>验证所有元层都包含 loader 能在系统中找到的有效组件 layer，
        并且这些组件 layer 还必须报告与元层自身相同的 Vulkan API 版本。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#meta-layers">元层</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_11</b></small></td>
    <td>如果覆盖元层存在，Loader <b>必须</b>在将所有其他隐式层添加到调用链之后，
        再加载它及其对应的组件 layer。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#override-meta-layer">覆盖元层</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_12</b></small></td>
    <td>如果覆盖元层存在，且它带有待移除 layer 的黑名单，
        则 Loader <b>必须</b>禁用黑名单中列出的所有 layer。
    </td>
    <td>行为未定义，并且可能导致崩溃或数据损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#override-meta-layer">覆盖元层</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LLP_LOADER_13</b></small></td>
    <td>当运行提升权限的应用程序（Administrator/Super-user）时，
        Loader <b>不得</b>从用户定义路径加载
        （包括使用 <i>VK_LAYER_PATH</i>、<i>VK_ADD_LAYER_PATH</i>、<i>VK_IMPLICIT_LAYER_PATH</i>
        或 <i>VK_ADD_IMPLICIT_LAYER_PATH</i> 环境变量）。<br/>
        <b>这是出于安全原因。</b>
    </td>
    <td>行为未定义，并且可能导致计算机安全漏洞、崩溃或数据损坏。
    </td>
    <td>否</td>
    <td><small><a href="#layer-discovery">Layer 发现</a></small></td>
  </tr>
</table>

<br/>

[返回顶层 LoaderInterfaceArchitecture.md 文件。](LoaderInterfaceArchitecture.md)
