<!-- markdownlint-disable MD041 -->

[![Khronos Vulkan][1]][2]

# Vulkan Loader 的驱动接口 <!-- omit from toc -->

[![Creative Commons][3]][4]

<!-- Copyright &copy; 2015-2023 LunarG, Inc. -->

<a id="table-of-contents"></a>
## 目录 <!-- omit from toc -->

- [概述](#overview)
- [驱动发现](#driver-discovery)
  - [覆盖默认驱动发现](#overriding-the-default-driver-discovery)
  - [附加驱动发现](#additional-driver-discovery)
  - [驱动过滤](#driver-filtering)
    - [驱动选择过滤](#driver-select-filtering)
    - [驱动禁用过滤](#driver-disable-filtering)
  - [提升权限时的例外情况](#exception-for-elevated-privileges)
    - [示例](#examples)
      - [在 Windows 上](#on-windows)
      - [在 Linux 上](#on-linux)
      - [在 macOS 上](#on-macos)
  - [驱动清单文件用法](#driver-manifest-file-usage)
  - [Windows 上的驱动发现](#driver-discovery-on-windows)
  - [Linux 上的驱动发现](#driver-discovery-on-linux)
    - [Linux 驱动搜索路径示例](#example-linux-driver-search-path)
  - [Fuchsia 上的驱动发现](#driver-discovery-on-fuchsia)
  - [macOS 上的驱动发现](#driver-discovery-on-macos)
    - [macOS 驱动搜索路径示例](#example-macos-driver-search-path)
    - [驱动调试的附加设置](#additional-settings-for-driver-debugging)
  - [使用 `VK_LUNARG_direct_driver_loading` 扩展的驱动发现](#driver-discovery-using-thevk_lunarg_direct_driver_loading-extension)
    - [如何使用 `VK_LUNARG_direct_driver_loading`](#how-to-use-vk_lunarg_direct_driver_loading)
    - [与其他驱动发现机制的交互](#interactions-with-other-driver-discovery-mechanisms)
    - [`VK_LUNARG_direct_driver_loading` 的限制](#limitations-of-vk_lunarg_direct_driver_loading)
  - [使用预生产 ICD 或软件驱动](#using-pre-production-icds-or-software-drivers)
  - [Android 上的驱动发现](#driver-discovery-on-android)
- [驱动清单文件格式](#driver-manifest-file-format)
  - [驱动清单文件版本](#driver-manifest-file-versions)
    - [驱动清单文件版本 1.0.0](#driver-manifest-file-version-100)
    - [驱动清单文件版本 1.0.1](#driver-manifest-file-version-101)
- [驱动 Vulkan 入口点发现](#driver-vulkan-entry-point-discovery)
- [驱动 API 版本](#driver-api-version)
- [混合驱动 instance 扩展支持](#mixed-driver-instance-extension-support)
  - [过滤掉 instance 扩展名](#filtering-out-instance-extension-names)
  - [loader 的 instance 扩展模拟支持](#loader-instance-extension-emulation-support)
- [驱动未知的物理设备扩展](#driver-unknown-physical-device-extensions)
  - [添加 `vk_icdGetPhysicalDeviceProcAddr` 的原因](#reason-for-adding-vk_icdgetphysicaldeviceprocaddr)
- [物理设备排序](#physical-device-sorting)
- [驱动可调度对象创建](#driver-dispatchable-object-creation)
- [在 WSI 扩展中处理 KHR Surface 对象](#handling-khr-surface-objects-in-wsi-extensions)
- [loader 与驱动接口协商](#loader-and-driver-interface-negotiation)
  - [Windows、Linux 和 macOS 驱动协商](#windows-linux-and-macos-driver-negotiation)
    - [loader 与驱动之间的版本协商](#version-negotiation-between-the-loader-and-drivers)
    - [与旧版驱动或 loader 对接](#interfacing-with-legacy-drivers-or-loaders)
    - [loader 与驱动接口版本 7 要求](#loader-and-driver-interface-version-7-requirements)
    - [loader 与驱动接口版本 6 要求](#loader-and-driver-interface-version-6-requirements)
    - [loader 与驱动接口版本 5 要求](#loader-and-driver-interface-version-5-requirements)
    - [loader 与驱动接口版本 4 要求](#loader-and-driver-interface-version-4-requirements)
    - [loader 与驱动接口版本 3 要求](#loader-and-driver-interface-version-3-requirements)
    - [loader 与驱动接口版本 2 要求](#loader-and-driver-interface-version-2-requirements)
    - [loader 与驱动接口版本 1 要求](#loader-and-driver-interface-version-1-requirements)
    - [loader 与驱动接口版本 0 要求](#loader-and-driver-interface-version-0-requirements)
    - [附加接口说明：](#additional-interface-notes)
  - [Android 驱动协商](#android-driver-negotiation)
- [loader 对 VK_KHR_portability_enumeration 的实现](#loader-implementation-of-vk_khr_portability_enumeration)
- [loader 与驱动策略](#loader-and-driver-policy)
  - [数字格式](#number-format)
  - [Android 差异](#android-differences)
  - [行为良好的驱动要求](#requirements-of-well-behaved-drivers)
    - [已移除的驱动策略](#removed-driver-policies)
  - [行为良好的 loader 要求](#requirements-of-a-well-behaved-loader)

<a id="overview"></a>
## 概述

这是使用 Vulkan loader 时以驱动为中心的视角。
有关 loader 各部分的完整概览，请参阅
[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文件。

**注意：** 尽管许多接口仍使用 "icd" 子串来标识与驱动相关的各种行为，
这纯粹出于历史原因，不应据此认为实现代码是通过传统 ICD 接口来完成这些行为的。
诚然，迄今为止大多数驱动确实都是面向特定 GPU 硬件的 ICD 驱动。

<a id="driver-discovery"></a>
## 驱动发现

Vulkan 允许多个驱动共同使用，每个驱动可包含一个或多个设备
（由 Vulkan `VkPhysicalDevice` 对象表示）。
loader 负责发现系统中可用的 Vulkan 驱动。
给定可用驱动列表后，loader 可以枚举应用可用的所有
物理设备，并将该信息返回给应用。
loader 在系统上发现可用驱动的过程依赖于平台。
下文列出了 Windows、Linux、Android 和 macOS 的驱动发现细节。

<a id="overriding-the-default-driver-discovery"></a>
### 覆盖默认驱动发现

有时开发者可能希望强制 loader 使用特定驱动。
这可能出于多种原因，包括使用 beta 驱动，或强制
loader 跳过存在问题的驱动。
为支持这种场景，可以通过 `VK_DRIVER_FILES` 或较旧的 `VK_ICD_FILENAMES`
环境变量，强制 loader 仅查看特定驱动。
这两个环境变量行为相同，但 `VK_ICD_FILENAMES`
应视为已弃用。
如果 `VK_DRIVER_FILES` 和 `VK_ICD_FILENAMES` 环境变量同时存在，
则会使用较新的 `VK_DRIVER_FILES`，并忽略
`VK_ICD_FILENAMES` 中的值。

`VK_DRIVER_FILES` 环境变量是一个指向驱动清单文件的路径列表，
其中可以包含驱动 JSON 清单文件的完整路径，和/或
包含驱动清单文件的文件夹路径。
在 Linux 和 macOS 上，该列表使用冒号分隔；在
Windows 上使用分号分隔。
通常，`VK_DRIVER_FILES` 只会包含某个单一驱动的一个信息
文件的完整路径名。
只有在需要多个驱动时才会使用分隔符（冒号或分号）。

<a id="additional-driver-discovery"></a>
### 附加驱动发现

有时开发者可能希望在标准驱动之外，再强制 loader 使用某个特定
驱动（而不替换标准搜索路径）。
可以使用 `VK_ADD_DRIVER_FILES` 环境变量添加一组
驱动清单文件，其中可以包含驱动 JSON 清单
文件的完整路径，和/或包含驱动清单文件的文件夹路径。
在 Linux 和 macOS 上，该列表使用冒号分隔；在
Windows 上使用分号分隔。
它会在标准驱动搜索文件之前加入。
如果存在 `VK_DRIVER_FILES` 或 `VK_ICD_FILENAMES`，则
loader 不会使用 `VK_ADD_DRIVER_FILES`，其中任何值都会被
忽略。

<a id="driver-filtering"></a>
### 驱动过滤

**注意：** 此功能仅适用于使用 Vulkan headers 1.3.234 及更高版本
构建的 loader。

loader 支持过滤环境变量，可强制选择或
禁用已知驱动。
已知驱动清单文件，是指 loader 在考虑默认搜索路径和其他环境变量（例如
`VK_ICD_FILENAMES` 或 `VK_ADD_DRIVER_FILES`）之后已经找到的那些文件。

过滤变量将与驱动的清单文件名进行比较。

这些过滤器还必须遵循
[Filter Environment Variable Behaviors](LoaderInterfaceArchitecture.md#filter-environment-variable-behaviors)
一节中定义的行为，该节位于 [LoaderLayerInterface](LoaderLayerInterface.md) 文档中。

<a id="driver-select-filtering"></a>
#### 驱动选择过滤

驱动选择环境变量 `VK_LOADER_DRIVERS_SELECT` 是一个
用于在已知驱动中搜索的 glob 模式逗号分隔列表。

如果在使用 `VK_LOADER_DRIVERS_SELECT` 过滤器时某个驱动未被选中，
并且 loader 日志被设置为输出警告或驱动消息，那么会为每个被忽略的驱动
显示一条消息。
该消息如下所示：

```
[Vulkan Loader] WARNING | DRIVER: Driver "intel_icd.x86_64.json" ignored because not selected by env var 'VK_LOADER_DRIVERS_SELECT'
```

如果没有任何驱动的清单文件名匹配所提供的任一
glob 模式，则不会启用任何驱动，这可能导致
运行任何 Vulkan 应用时失败。

<a id="driver-disable-filtering"></a>
#### 驱动禁用过滤

驱动禁用环境变量 `VK_LOADER_DRIVERS_DISABLE` 是一个
用于在已知驱动中搜索的 glob 模式逗号分隔列表。

当通过 `VK_LOADER_DRIVERS_DISABLE` 过滤器禁用某个驱动时，
并且 loader 日志被设置为输出警告或驱动消息，那么会为每个被强制禁用的驱动
显示一条消息。
该消息如下所示：

```
[Vulkan Loader] WARNING | DRIVER: Driver "radeon_icd.x86_64.json" ignored because it was disabled by env var 'VK_LOADER_DRIVERS_DISABLE'
```

如果没有任何驱动的清单文件名匹配所提供的任一
glob 模式，则不会禁用任何驱动。

<a id="exception-for-elevated-privileges"></a>
### 提升权限时的例外情况

出于安全原因，如果 Vulkan 应用以
提升权限运行，则会忽略 `VK_ICD_FILENAMES`、`VK_DRIVER_FILES` 和
`VK_ADD_DRIVER_FILES`。
这是因为它们可能将 loader 平常不会找到的新库插入到
可执行进程中。
因此，这些环境变量只能用于
未使用提升权限的应用。

更多信息请参见顶层
[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文档中的
[特权提升注意事项](LoaderInterfaceArchitecture.md#elevated-privilege-caveats)。

<a id="examples"></a>
#### 示例

要使用该设置，只需将它设为按正确分隔符分隔的
驱动清单文件列表。
在这种情况下，请为这些文件提供完整路径，以减少问题。

例如：

<a id="on-windows"></a>
##### 在 Windows 上

```
set VK_DRIVER_FILES=\windows\system32\nv-vk64.json
```

这是一个在 Windows 上使用 `VK_DRIVER_FILES` 覆盖设置、
指向 Nvidia Vulkan 驱动清单文件的示例。

```
set VK_ADD_DRIVER_FILES=\windows\system32\nv-vk64.json
```

这是一个在 Windows 上使用 `VK_ADD_DRIVER_FILES`、
指向 Nvidia Vulkan 驱动清单文件的示例，该驱动会在所有其他驱动之前
优先加载。

<a id="on-linux"></a>
##### 在 Linux 上

```
export VK_DRIVER_FILES=/home/user/dev/mesa/share/vulkan/icd.d/intel_icd.x86_64.json
```

这是一个在 Linux 上使用 `VK_DRIVER_FILES` 覆盖设置、
指向 Intel Mesa 驱动清单文件的示例。

```
export VK_ADD_DRIVER_FILES=/home/user/dev/mesa/share/vulkan/icd.d/intel_icd.x86_64.json
```

这是一个在 Linux 上使用 `VK_ADD_DRIVER_FILES`、
指向 Intel Mesa 驱动清单文件的示例，该驱动会在所有其他驱动之前
优先加载。

<a id="on-macos"></a>
##### 在 macOS 上

```
export VK_DRIVER_FILES=/home/user/MoltenVK/Package/Latest/MoltenVK/macOS/MoltenVK_icd.json
```

这是一个在 macOS 上使用 `VK_DRIVER_FILES` 覆盖设置、
指向 MoltenVK GitHub 仓库某个安装与构建位置的示例，
其中包含 MoltenVK 驱动。

更多细节请参见
[LoaderInterfaceArchitecture.md 文档](LoaderInterfaceArchitecture.md) 中的
[调试环境变量总表](LoaderInterfaceArchitecture.md#table-of-debug-environment-variables)

<a id="driver-manifest-file-usage"></a>
### 驱动清单文件用法

与 layer 一样，在 Windows、Linux 和 macOS 系统上，JSON 格式的清单
文件用于存储驱动信息。
为了找到系统已安装的驱动，Vulkan loader 会读取这些 JSON
文件，以识别每个驱动的名称和属性。
请注意，驱动清单文件比相应的
layer 清单文件简单得多。

更多细节请参见
[当前驱动清单文件格式](#driver-manifest-file-format)
一节。

<a id="driver-discovery-on-windows"></a>
### Windows 上的驱动发现

为了找到可用驱动（包括已安装的 ICD），
loader 会扫描显示适配器以及与这些适配器关联的所有软件组件所对应的特定注册表键，
以查找 JSON 清单文件的位置。
这些键位于驱动安装期间创建的设备键中，
并包含基础设置的配置信息，包括 OpenGL 和
Direct3D 的位置。

设备适配器和软件组件键路径将首先通过
枚举 DXGI 适配器获得。
如果失败，则会使用 PnP Configuration Manager API。
`000X` 键是一个编号键，每个设备都会分配不同的
编号。

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanDriverName
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{SoftwareComponent GUID}\000X\VulkanDriverName
```

此外，在 64 位系统上可能还存在另一组注册表值，
如下所示。
这些值以与 Windows-on-Windows 功能相同的方式，
记录了 64 位操作系统上 32 位 layer 的位置。

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{Adapter GUID}\000X\VulkanDriverNameWow
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Class\{SoftwareComponent GUID}\000X\VulkanDriverNameWow
```

如果上述任一值存在且类型为 `REG_SZ`，loader 将打开
该键值指定的 JSON 清单文件。
每个值都必须是 JSON 清单文件的完整绝对路径。
这些值也可以是 `REG_MULTI_SZ` 类型，在这种情况下，该值会被
解释为 JSON 清单文件路径列表。

此外，Vulkan loader 还会扫描以下 Windows
注册表键中的值：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\Drivers
```

对于运行在 64 位 Windows 上的 32 位应用，loader 会扫描 32 位
注册表位置：

```
HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Khronos\Vulkan\Drivers
```

这些位置中的每个驱动都应表示为一个值为 0 的 DWORD，
其值名是 JSON 清单文件的完整路径。
Vulkan loader 将尝试打开每个清单文件，以获取
驱动共享库（".dll"）文件的信息。

例如，假设注册表中包含以下数据：

```
[HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\Drivers\]

"C:\vendor a\vk_vendor_a.json"=dword:00000000
"C:\windows\system32\vendor_b_vk.json"=dword:00000001
"C:\windows\system32\vendor_c_icd.json"=dword:00000000
```

在这种情况下，loader 会依次处理每个条目并检查其值。
如果值为 0，则 loader 会尝试加载该文件。
在这个例子中，loader 会打开第一项和最后一项，
但不会打开中间那一项。
这是因为 vendor_b_vk.json 的值为 1，会禁用该驱动。

此外，Vulkan loader 还会扫描系统中已知的 Windows
AppX/MSIX 包。
如果找到某个包，loader 会扫描该已安装包的根目录，
查找 JSON 清单文件。目前唯一已知的包是
Microsoft 的
[OpenCL™, OpenGL®, and Vulkan® Compatibility Pack](https://apps.microsoft.com/store/detail/9NQPSL29BFFF?hl=en-us&gl=US)。

Vulkan loader 会打开找到的每个已启用清单文件，以获取
驱动共享库（".DLL"）文件的名称或路径名。

在可行情况下，驱动应使用来自 PnP Configuration
Manager 的注册表位置。
通常，这对于驱动最为重要，因为该位置会将驱动明确
关联到给定设备。
`SOFTWARE\Khronos\Vulkan\Drivers` 位置是定位驱动的较旧方法，
但它是基于软件的驱动的主要位置。

更多细节请参见
[驱动清单文件格式](#driver-manifest-file-format)
一节。

<a id="driver-discovery-on-linux"></a>
### Linux 上的驱动发现

在 Linux 上，Vulkan loader 会使用环境变量扫描驱动清单文件；
如果相应环境变量未定义，则使用对应的回退值：

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
    <td><b>在以 setuid、setgid 或文件系统能力等提升权限方式运行时，会忽略该路径</b>。<br/>
        这样做是因为在这些场景下，无法安全地信任
        环境变量不是恶意的。<br/>
        更多信息请参见 <a href="LoaderInterfaceArchitecture.md#elevated-privilege-caveats">
        特权提升注意事项</a>。
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
    <td>编译时选项，设置为来自非 Linux 发行版提供的软件包所安装驱动的
        可能位置。
    </td>
  </tr>
  <tr>
    <td>3</td>
    <td>EXTRASYSCONFDIR</td>
    <td>/etc</td>
    <td>编译时选项，设置为来自非 Linux 发行版提供的软件包所安装驱动的
        可能位置。
        通常仅在 SYSCONFDIR 被设为 /etc 之外的值时才设置
    </td>
  </tr>
  <tr>
    <td>4</td>
    <td>$XDG_DATA_HOME</td>
    <td>$HOME/.local/share</td>
    <td><b>在以 setuid、setgid 或文件系统能力等提升权限方式运行时，会忽略该路径</b>。<br/>
        这样做是因为在这些场景下，无法安全地信任
        环境变量不是恶意的。<br/>
        更多信息请参见 <a href="LoaderInterfaceArchitecture.md#elevated-privilege-caveats">
        特权提升注意事项</a>。
    </td>
  </tr>
  <tr>
    <td>5</td>
    <td>$XDG_DATA_DIRS</td>
    <td>/usr/local/share/:/usr/share/</td>
    <td></td>
  </tr>
</table>

目录列表会使用标准平台路径分隔符
（:）拼接在一起。
然后 loader 会选择每个路径，并在其后附加 "/vulkan/icd.d" 后缀，
再到该特定文件夹中查找清单文件。

Vulkan loader 会打开找到的每个清单文件，以获取驱动共享库
（".so"）文件的名称或路径名。

**注意：** 尽管搜索清单文件的文件夹顺序有明确定义，
但 loader 在每个目录中读取内容的顺序会因
[readdir 的行为而具有随机性](https://www.ibm.com/support/pages/order-directory-contents-returned-calls-readdir)。

更多细节请参见
[驱动清单文件格式](#driver-manifest-file-format)
一节。

还需要特别注意的是，尽管 `VK_DRIVER_FILES` 会让 loader 去查找
清单文件，但这并不保证清单中提到的库文件会立即被找到。
驱动清单文件通常会通过相对路径或绝对路径指向
库文件。
使用相对路径或绝对路径时，loader 通常可以在不查询
操作系统的情况下找到该库文件。
但是，如果库仅按名称列出，loader 可能找不到它，
除非驱动安装时已将该库放在操作系统
可搜索的默认位置。
如果在查找与驱动关联的库文件时出现问题，请尝试更新
`LD_LIBRARY_PATH` 环境变量，使其指向相应
`.so` 文件的位置。

<a id="example-linux-driver-search-path"></a>
#### Linux 驱动搜索路径示例

对于一个虚构用户 "me"，驱动清单搜索路径可能
如下所示：

```
  /home/me/.config/vulkan/icd.d
  /etc/xdg/vulkan/icd.d
  /usr/local/etc/vulkan/icd.d
  /etc/vulkan/icd.d
  /home/me/.local/share/vulkan/icd.d
  /usr/local/share/vulkan/icd.d
  /usr/share/vulkan/icd.d
```

<a id="driver-discovery-on-fuchsia"></a>
### Fuchsia 上的驱动发现

在 Fuchsia 上，Vulkan loader 会像
[Linux](#linux-driver-discovery) 一样，使用环境变量扫描清单文件；
如果相应环境变量未定义，则使用对应的回退值。
**唯一** 的区别是，Fuchsia 不允许对
*$XDG_DATA_DIRS* 或 *$XDG_HOME_DIRS* 使用回退值。

<a id="driver-discovery-on-macos"></a>
### macOS 上的驱动发现

在 macOS 上，Vulkan loader 会使用
应用资源文件夹以及环境变量来扫描驱动清单文件；如果相应环境变量未
定义，则使用对应的回退值。
其顺序与 Linux 上的搜索路径类似，但有一个例外：
会先搜索应用的 bundle 资源目录：
`(bundle)/Contents/Resources/`。

如果在应用 bundle 内找到驱动，则会忽略系统已安装的驱动。
这是因为目前没有标准机制来区分那些恰好是重复项的驱动。
例如，MoltenVK 通常会放在应用 bundle 中。
如果系统中还安装了 MoltenVK，loader 会同时加载
应用 bundle 内和系统安装的 MoltenVK，从而导致潜在问题或崩溃。
通过环境变量（如 `VK_DRIVER_FILES`）找到的驱动，
无论是否存在 bundle 内驱动，都会被使用。

<a id="example-macos-driver-search-path"></a>
#### macOS 驱动搜索路径示例

对于一个虚构用户 "Me"，驱动清单搜索路径可能
如下所示：

```
  <bundle>/Contents/Resources/vulkan/icd.d
  /Users/Me/.config/vulkan/icd.d
  /etc/xdg/vulkan/icd.d
  /usr/local/etc/vulkan/icd.d
  /etc/vulkan/icd.d
  /Users/Me/.local/share/vulkan/icd.d
  /usr/local/share/vulkan/icd.d
  /usr/share/vulkan/icd.d
```

<a id="additional-settings-for-driver-debugging"></a>
#### 驱动调试的附加设置

有时，驱动在加载时可能会遇到问题。
一个有用的选项是启用 `LD_BIND_NOW` 环境变量
来调试该问题。
这会强制每个动态库的所有符号在加载时就被完全解析。
如果当前系统上的驱动存在缺失符号问题，
这会暴露该问题，并导致 Vulkan loader 在加载驱动时失败。
建议同时使用 `LD_BIND_NOW` 和 `VK_LOADER_DEBUG=error,warn`
来暴露任何问题。

<a id="driver-discovery-using-thevk_lunarg_direct_driver_loading-extension"></a>
### 使用 `VK_LUNARG_direct_driver_loading` 扩展的驱动发现

`VK_LUNARG_direct_driver_loading` 扩展允许应用在
`vkCreateInstance` 期间向 loader 提供一个或多个驱动。
这使得驱动无需安装即可随应用一起提供，并且能够用于任何执行环境，
例如以提升权限运行的进程。

在启用 `VK_LUNARG_direct_driver_loading` 扩展并调用
`vkEnumeratePhysicalDevices` 时，来自系统已安装驱动和环境变量指定驱动的
`VkPhysicalDevice` 会先于任何来自
`VkDirectDriverLoadingListLUNARG::pDrivers` 列表中驱动的 `VkPhysicalDevice`
出现。

<a id="how-to-use-vk_lunarg_direct_driver_loading"></a>
#### 如何使用 `VK_LUNARG_direct_driver_loading`

要使用该扩展，必须先在 `VkInstance` 上启用它。
这要求通过 `VkInstanceCreateInfo` 的
`enabledExtensionCount` 和 `ppEnabledExtensionNames` members
启用 `VK_LUNARG_direct_driver_loading` 扩展。

```c
const char* extensions[] = {VK_LUNARG_DIRECT_DRIVER_LOADING_EXTENSION_NAME, <other extensions>};
VkInstanceCreateInfo instance_create_info = {};
instance_create_info.enabledExtensionCount = <size of extension list>;
instance_create_info.ppEnabledExtensionNames = extensions;
```

`VkDirectDriverLoadingInfoLUNARG` 结构包含一个
`VkDirectDriverLoadingFlagsLUNARG` 成员（保留供未来使用），以及一个
`PFN_vkGetInstanceProcAddrLUNARG` 成员，它向 loader 提供
驱动的 `vkGetInstanceProcAddr` 函数指针。

`VkDirectDriverLoadingListLUNARG` 结构包含计数和指针
成员，它们提供应用所提供的 `VkDirectDriverLoadingInfoLUNARG`
结构数组的大小和指针。

创建这些结构如下所示

```c
VkDirectDriverLoadingInfoLUNARG direct_loading_info = {};
direct_loading_info.sType = VK_STRUCTURE_TYPE_DIRECT_DRIVER_LOADING_INFO_LUNARG
direct_loading_info.pfnGetInstanceProcAddr = <put the PFN_vkGetInstanceProcAddr of the driver here>

VkDirectDriverLoadingListLUNARG direct_driver_list = {};
direct_driver_list.sType = VK_STRUCTURE_TYPE_DIRECT_DRIVER_LOADING_LIST_LUNARG;
direct_driver_list.mode = VK_DIRECT_DRIVER_LOADING_MODE_INCLUSIVE_LUNARG; // or VK_DIRECT_DRIVER_LOADING_MODE_EXCLUSIVE_LUNARG
direct_driver_list.driverCount = 1;
direct_driver_list.pDrivers = &direct_loading_info; // can include multiple drivers here if so desired
```

`VkDirectDriverLoadingListLUNARG` 结构包含枚举
`VkDirectDriverLoadingModeLUNARG`。
有两种模式：

* `VK_DIRECT_DRIVER_LOADING_MODE_EXCLUSIVE_LUNARG` - 指定仅会加载来自
  `VkDirectDriverLoadingListLUNARG` 结构的驱动。
* `VK_DIRECT_DRIVER_LOADING_MODE_INCLUSIVE_LUNARG` - 指定除了
  系统已安装驱动和环境变量指定驱动之外，还会使用来自
  `VkDirectDriverLoadingModeLUNARG` 结构的驱动。

然后，`VkDirectDriverLoadingListLUNARG` 结构 *必须* 追加到
`VkInstanceCreateInfo` 的 `pNext` 链中。

```c
instance_create_info.pNext = (const void*)&direct_driver_list;
```

最后，像平常一样创建 instance。

<a id="interactions-with-other-driver-discovery-mechanisms"></a>
#### 与其他驱动发现机制的交互

如果在 `VkDirectDriverLoadingListLUNARG` 结构中指定了
`VK_DIRECT_DRIVER_LOADING_MODE_EXCLUSIVE_LUNARG` 模式，则不会加载
任何系统已安装的驱动。
这在所有平台上都同样适用。
此外，以下环境变量不会产生任何作用：

* `VK_DRIVER_FILES`
* `VK_ICD_FILENAMES`
* `VK_ADD_DRIVER_FILES`
* `VK_LOADER_DRIVERS_SELECT`
* `VK_LOADER_DRIVERS_DISABLE`

Exclusive 模式还会禁用 macOS bundle 驱动清单发现。

<a id="limitations-of-vk_lunarg_direct_driver_loading"></a>
#### `VK_LUNARG_direct_driver_loading` 的限制

由于 `VkDirectDriverLoadingListLUNARG` 是在创建 instance 时提供给 loader 的，
因此 loader 无法在
`vkEnumerateInstanceExtensionProperties` 期间查询来自
`VkDirectDriverLoadingListLUNARG` 驱动的 instance 扩展列表。
应用可以改为使用每个驱动的 `pfnGetInstanceProcAddrLUNARG`，
直接从应用提供给 loader 的驱动中手动加载
`vkEnumerateInstanceExtensionProperties` 函数指针。
随后，应用可以调用每个驱动的
`vkEnumerateInstanceExtensionProperties`，并将非重复条目追加到
loader 的 `vkEnumerateInstanceExtensionProperties` 返回列表中，以获得完整的
受支持 instance 扩展列表。
另一种方式是，由于驱动是由应用提供的，因此可以合理地认为
应用已经知道这些驱动提供了哪些 instance 扩展，
从而无需手动查询它们。

但是，这里也存在限制。
如果有任何活动的隐式层会拦截
`vkEnumerateInstanceExtensionProperties` 以移除不受支持的扩展，那么
这些层将无法从应用提供的驱动中移除不受支持的扩展。
这是因为 `vkEnumerateInstanceExtensionProperties` 没有机制
对其进行扩展。

<a id="using-pre-production-icds-or-software-drivers"></a>
### 使用预生产 ICD 或软件驱动

软件驱动和预生产 ICD 都可以使用替代机制来
检测其驱动。
独立硬件供应商（IHV）可能不想完整安装某个预生产
ICD，因此无法在标准位置找到它。
例如，预生产 ICD 可能只是开发者构建树中的一个共享库。
在这种情况下，应当提供一种方式，使开发者无需修改系统上已安装的
ICD，即可指向这样的 ICD。

这一需求可通过使用 `VK_DRIVER_FILES` 环境变量来满足，
它会覆盖用于查找系统已安装驱动的机制。

换句话说，只会使用 `VK_DRIVER_FILES` 中列出的驱动。

更多信息请参见
[覆盖默认驱动发现](#overriding-the-default-driver-discovery)。

<a id="driver-discovery-on-android"></a>
### Android 上的驱动发现

Android loader 位于系统库文件夹中。
该位置无法更改。
loader 会通过 `hw_get_module`，使用 ID "vulkan" 来加载驱动。
**由于 Android 的安全策略，在正常使用情况下，这些内容都无法**
**被修改。**

<a id="driver-manifest-file-format"></a>
## 驱动清单文件格式

以下各节讨论驱动清单 JSON 文件格式的细节。
JSON 文件本身在命名上没有任何要求。
唯一要求是该文件的扩展名后缀必须为 ".json"。

下面是一个驱动 JSON 清单文件示例：

```json
{
   "file_format_version": "1.0.1",
   "ICD": {
      "library_path": "path to driver library",
      "api_version": "1.2.205",
      "library_arch" : "64",
      "is_portability_driver": false
   }
}
```

<table style="width:100%">
  <tr>
    <th>字段名</th>
    <th>字段值</th>
  </tr>
  <tr>
    <td>"file_format_version"</td>
    <td>该文件的 JSON 格式 major.minor.patch 版本号。<br/>
        支持的版本为：1.0.0 和 1.0.1。</td>
  </tr>
  <tr>
    <td>"ICD"</td>
    <td>用于将所有驱动信息归在一起的标识符。
        <br/>
        <b>注意：</b> 尽管这里标记为 <i>ICD</i>，但这只是历史遗留，
        对其他驱动同样适用。</td>
  </tr>
  <tr>
    <td>"library_path"</td>
    <td>"library_path" 指定驱动共享库文件的文件名、相对路径名或
        完整路径名。 <br />
        如果 "library_path" 指定的是相对路径名，则它相对于
        JSON 清单文件所在路径。 <br />
        如果 "library_path" 指定的是文件名，则该库必须位于
        系统的共享对象搜索路径中。 <br />
        对驱动共享库文件名没有其他规则要求，
        唯一要求是它应以适当的后缀结尾（Windows 上为 ".DLL"，
        Linux 上为 ".so"，macOS 上为 ".dylib"）。</td>
  </tr>
  <tr>
    <td>"library_arch"</td>
    <td>可选字段，用于指定与
        "library_path" 关联二进制的架构。 <br />
        它允许 loader 快速判断驱动的架构
        是否与当前运行应用匹配。 <br />
        唯一有效的值是 "32" 和 "64"。</td>
  </tr>
  <tr>
    <td>"api_version" </td>
    <td>驱动所支持的最大 Vulkan API 的 major.minor.patch
        版本号。
        但是，仅因为驱动支持某个特定 Vulkan API
        版本，并不保证用户系统上的硬件也能
        支持该版本。
        底层物理设备实际支持哪些能力，必须由用户通过
        <i>vkGetPhysicalDeviceProperties</i> API
        调用来查询。<br/>
        例如：1.0.33。</td>
  </tr>
  <tr>
    <td>"is_portability_driver" </td>
    <td>定义该驱动是否包含实现
        VK_KHR_portability_subset 扩展的 VkPhysicalDevices。<br/>
    </td>
  </tr>
</table>

**注意：** 如果同一个驱动共享库支持多个彼此不兼容的
清单文件格式版本，则必须为每个版本提供单独的 JSON 文件
（它们都可以指向同一个共享库）。

<a id="driver-manifest-file-versions"></a>
### 驱动清单文件版本

当前支持的最高驱动清单文件格式版本是 1.0.1。
有关各个版本的信息详见以下小节：

<a id="driver-manifest-file-version-100"></a>
#### 驱动清单文件版本 1.0.0

驱动清单文件的初始版本规定了 layer JSON 文件的基本
格式和字段。
文件格式 1.0.0 版本支持的字段包括：

 * "file\_format\_version"
 * "ICD"
 * "library\_path"
 * "api\_version"

<a id="driver-manifest-file-version-101"></a>
#### 驱动清单文件版本 1.0.1

为驱动添加了 `is_portability_driver` 布尔字段，以便它们自行报告
是否包含支持 VK_KHR_portability_subset
扩展的 VkPhysicalDevices。这是一个可选字段。省略该字段与
将其设为 `false` 的效果相同。

向驱动清单中添加了 "library\_arch" 字段，以便 loader 能够
快速判断驱动是否与当前运行应用的架构
匹配。该字段是可选的。
<a id="driver-vulkan-entry-point-discovery"></a>
## 驱动 Vulkan 入口点发现

驱动导出的 Vulkan 符号不得与 loader 导出的 Vulkan 符号发生冲突。
因此，所有驱动都必须导出以下函数，用于发现驱动的 Vulkan 入口点。
该入口点并不是 Vulkan API 本身的一部分，而只是用于版本 1 及更高版本接口中 loader 与驱动之间的私有接口。

```cpp
VKAPI_ATTR PFN_vkVoidFunction VKAPI_CALL
   vk_icdGetInstanceProcAddr(
      VkInstance instance,
      const char* pName);
```

该函数的语义与 `vkGetInstanceProcAddr` 非常相似。
`vk_icdGetInstanceProcAddr` 会为所有全局级和 instance 级 Vulkan 函数，以及 `vkGetDeviceProcAddr` 返回有效的函数指针。
全局级函数是指第一个参数中不包含可分发对象的函数，例如 `vkCreateInstance` 和 `vkEnumerateInstanceExtensionProperties`。
驱动必须支持通过向 `vk_icdGetInstanceProcAddr` 传入 `NULL` 的 `VkInstance` 参数来查询全局级入口点。
instance 级函数是指将 `VkInstance` 或 `VkPhysicalDevice` 作为第一个参数可分发对象的函数。
驱动支持的核心入口点以及任何 instance 扩展入口点，都应可通过 `vk_icdGetInstanceProcAddr` 获取。
未来的 Vulkan instance 扩展可能会定义并使用除 `VkInstance` 和 `VkPhysicalDevice` 之外的新 instance 级可分发对象；在这种情况下，使用这些新定义可分发对象的扩展入口点必须能够通过 `vk_icdGetInstanceProcAddr` 进行查询。

所有其他 Vulkan 入口点都必须满足以下二者之一：

 * 不得直接从驱动库中导出
 * 或者如果导出，则不得使用官方 Vulkan 函数名

这一要求适用于同时包含其他功能（例如 OpenGL）的驱动库，因为这类库可能会在应用加载 Vulkan loader 库之前就被应用加载。

如果使用官方 Vulkan 名称，请注意动态操作系统库加载器的 interposing。
在 Linux 上，如果使用官方名称，则驱动库必须使用 `-Bsymbolic` 进行链接。

<a id="driver-api-version"></a>
## 驱动 API 版本

当应用调用 `vkCreateInstance` 时，它可以选择传入一个 `VkApplicationInfo` 结构体，其中包含 `apiVersion` 字段。
Vulkan 1.0 驱动在用户传入其不支持的 API 版本时，必须返回 `VK_ERROR_INCOMPATIBLE_DRIVER`。
从 Vulkan 1.1 开始，驱动不得针对任何 `apiVersion` 值返回此错误。
当系统中存在多个驱动、其中一个是 1.0 驱动而另一个较新时，这会带来问题。

当应用调用 `vkEnumerateInstanceVersion` 时，新于 1.0 的 loader 总是会返回它自身所支持的版本，而不考虑系统中驱动支持的 API 版本。
这意味着，当应用调用 `vkCreateInstance` 时，为避免出错，loader 将不得不向任何 1.0 驱动传递一份 `VkApplicationInfo` 结构体副本，并将其中的 `apiVersion` 设为 1.0。
为了确定是否必须这样做，loader 会执行以下步骤：

1. 检查驱动 JSON 清单文件中的 "api_version" 字段。
2. 如果 JSON 中的版本大于或等于 1.1，则加载驱动的动态库
3. 调用驱动的 `vkGetInstanceProcAddr` 命令以获取指向 `vkEnumerateInstanceVersion` 的指针
4. 如果 `vkEnumerateInstanceVersion` 的指针不是 `NULL`，则调用它以获取驱动支持的 API 版本

如果满足以下任一条件，则该驱动会被视为 1.0 驱动：

- JSON 清单文件中的 "api_version" 字段小于 1.1
- 指向 `vkEnumerateInstanceVersion` 的函数指针为 `NULL`
- `vkEnumerateInstanceVersion` 返回的版本小于 1.1
- `vkEnumerateInstanceVersion` 返回的结果不是 `VK_SUCCESS`

如果驱动只支持 Vulkan 1.0，loader 将确保传递给驱动的任何 `VkApplicationInfo` 结构体都将其 `apiVersion` 字段设为 Vulkan 1.0。
否则，loader 会在不作任何修改的情况下将该结构体传递给驱动。

<a id="mixed-driver-instance-extension-support"></a>
## 混合驱动的 instance 扩展支持

在具有多个驱动的系统上，可能会出现一种特殊情况。
某些驱动可能会公开 loader 已经知晓的某个 instance 扩展。
同一系统上的其他驱动则可能不支持该 instance 扩展。

在这种场景下，loader 还需承担一些额外责任：

<a id="filtering-out-instance-extension-names"></a>
### 过滤 instance 扩展名

在调用 `vkCreateInstance` 期间，请求的 instance 扩展列表会向下传递给每个驱动。
由于驱动可能不支持其中一个或多个 instance 扩展，loader 会过滤掉驱动不支持的所有 instance 扩展。
这是按驱动分别完成的，因为不同驱动可能支持不同的 instance 扩展。

<a id="loader-instance-extension-emulation-support"></a>
### loader 对 instance 扩展的模拟支持

在相同场景下，对于每个不直接支持某个 instance 扩展的驱动，loader 必须尽其所能模拟该 instance 扩展的入口点。
当与其他原生支持该扩展的驱动组合调用时，这一机制也必须能正确工作。
通过这种方式，应用将不会察觉哪些驱动缺少对该扩展的支持。

<a id="driver-unknown-physical-device-extensions"></a>
## 驱动未知物理设备扩展

如果驱动实现了将 `VkPhysicalDevice` 作为第一个参数的入口点，则其 *应* 支持 `vk_icdGetPhysicalDeviceProcAddr`。
该函数是在 loader 与驱动接口版本 4 中加入的，使 loader 能够区分那些将 `VkDevice` 和 `VkPhysicalDevice` 作为第一个参数的入口点。
这样一来，loader 就能更妥善地支持那些它尚未知晓的入口点。
该入口点并不是 Vulkan API 本身的一部分，而只是 loader 与驱动之间的私有接口。
注意：loader 与驱动接口版本 7 使导出 `vk_icdGetPhysicalDeviceProcAddr` 变为可选。
相反，驱动 **必须** 通过 `vk_icdGetInstanceProcAddr` 暴露它。

```cpp
PFN_vkVoidFunction
   vk_icdGetPhysicalDeviceProcAddr(
      VkInstance instance,
      const char* pName);
```

该函数的行为与 `vkGetInstanceProcAddr` 和 `vkGetDeviceProcAddr` 类似，
但它只应返回物理设备扩展入口点的值。
也就是说，它会将 "pName" 与驱动中支持的每一个物理设备函数进行比较。

该函数的实现应具有如下行为：

* 如果 `pName` 是某个 Vulkan API 入口点的名称，而该入口点以 `VkPhysicalDevice` 作为其主调度句柄，且驱动支持该入口点，那么驱动 **必须** 返回指向该驱动该入口点实现的有效函数指针。
* 如果 `pName` 是某个 Vulkan API 入口点的名称，但该入口点的主调度句柄不是 `VkPhysicalDevice`，那么驱动 **必须** 返回 `NULL`。
* 如果驱动不知道名称为 `pName` 的任何入口点，则其 **必须** 返回 `NULL`。

如果驱动打算支持那些以 VkPhysicalDevice 作为可分发参数的函数，那么驱动就应支持 `vk_icdGetPhysicalDeviceProcAddr`。
这是因为，如果这些函数对 loader 来说是未知的，例如它们来自尚未发布的扩展，或者因为 loader 构建版本较旧、_尚未_ 知道它们，那么 loader 将无法区分这是 device 函数还是物理设备函数。

如果驱动确实实现了此支持，它必须使用 `vk_icdGetPhysicalDeviceProcAddr` 这个名称从驱动库中导出该函数，以便平台的动态链接工具能够定位该符号；或者，如果驱动支持 loader 与驱动接口版本 7，则改为通过 `vk_icdGetInstanceProcAddr` 暴露它。

在支持 `vk_icdGetPhysicalDeviceProcAddr` 函数的情况下，loader 的 `vkGetInstanceProcAddr` 行为如下：

  1. 检查是否为核心函数：
     - 如果是，则返回该函数指针
  2. 检查是否为已知的 instance 扩展函数或 device 扩展函数：
     - 如果是，则返回该函数指针
  3. 调用 layer/驱动 的 `GetPhysicalDeviceProcAddr`
     - 如果返回 `non-NULL`，则返回一个通用物理设备函数的 trampoline，并设置一个通用 terminator，将其传递给正确的驱动。
  4. 使用 `GetInstanceProcAddr` 继续向下调用
     - 如果返回非 `NULL`，则将其视为未知的逻辑设备命令。
这意味着要设置一个通用 trampoline 函数，将 `VkDevice` 作为第一个参数，并在从 `VkDevice` 获取调度表后，调整调度表以调用驱动/layer 的函数。
然后，返回对应 trampoline 函数的指针。
  5. 返回 `NULL`

其结果是，如果该命令后来被提升为 Vulkan 核心命令，就不再会通过 `vk_icdGetPhysicalDeviceProcAddr` 来设置。
另外，如果 loader 后续直接加入了对该扩展的支持，也不会再走到步骤 3，因为步骤 2 会直接返回有效函数指针。
不过，驱动仍应继续通过 `vk_icdGetPhysicalDeviceProcAddr` 支持该命令的查询，至少要持续到某次 Vulkan 版本提升之后，
因为旧版 loader 仍可能试图使用这些命令。

<a id="reason-for-adding-vk_icdgetphysicaldeviceprocaddr"></a>
### 添加 `vk_icdGetPhysicalDeviceProcAddr` 的原因

最初，当在 loader 中调用 `vkGetInstanceProcAddr` 时，其行为如下：

  1. loader 会检查它是否为核心函数：
     - 如果是，则返回该函数指针
  2. loader 会检查它是否为已知扩展函数：
     - 如果是，则返回该函数指针
  3. 如果 loader 对它一无所知，则会使用 `GetInstanceProcAddr` 继续向下调用
     - 如果返回 `non-NULL`，则将其视为未知的逻辑设备命令。
     - 这意味着要设置一个通用 trampoline 函数，将 `VkDevice` 作为第一个参数，并在从 `VkDevice` 获取调度表后，调整调度表以调用驱动/layer 的函数。
  4. 如果以上都失败，loader 会向应用返回 `NULL`。

当驱动试图暴露 loader 完全不知道、但应用知道的新物理设备扩展时，这种做法就会引发问题。
由于 loader 对它一无所知，它会在上述流程中走到步骤 3，并将该函数当作未知的逻辑设备命令来处理。
问题在于，这会创建一个通用的 `VkDevice` trampoline 函数，而该函数在首次调用时会尝试将 VkPhysicalDevice 按 `VkDevice` 进行解引用。
这会导致崩溃或数据损坏。

<a id="physical-device-sorting"></a>
## 物理设备排序

当应用选择要使用的 GPU 时，它必须枚举物理设备或物理设备分组。
这些 API 函数并未指定物理设备或物理设备分组的呈现顺序。
在 Windows 上，loader 会尝试对这些对象进行排序，以便将系统偏好项列在最前面。
该机制并不会强制应用使用任何特定 GPU &mdash; 它只是改变它们的呈现顺序。

该机制要求驱动支持 loader 与驱动接口版本 6。
此版本定义了一个新的导出函数 `vk_icdEnumerateAdapterPhysicalDevices`，详见下文，驱动可以在 Windows 上提供该函数。
该入口点并不是 Vulkan API 本身的一部分，而只是 loader 与驱动之间的私有接口。
注意：loader 与驱动接口版本 7 使导出 `vk_icdEnumerateAdapterPhysicalDevices` 变为可选。
相反，驱动 **必须** 通过 `vk_icdGetInstanceProcAddr` 暴露它。

```c
VKAPI_ATTR VkResult VKAPI_CALL
   vk_icdEnumerateAdapterPhysicalDevices(
      VkInstance instance,
      LUID adapterLUID,
      uint32_t* pPhysicalDeviceCount,
      VkPhysicalDevice* pPhysicalDevices);
```

该函数将适配器 LUID 作为输入，并枚举与该 LUID 关联的所有 Vulkan 物理设备。
它的工作方式与其他 Vulkan 枚举相同 &mdash; 如果 `pPhysicalDevices` 为 `NULL`，则会提供计数值。
否则，将提供与所查询适配器关联的物理设备。
当该 LUID 指向一个链接适配器时，该函数必须提供多个物理设备。
这使 loader 能够将该适配器转换为 Vulkan 物理设备分组。

尽管 loader 会尝试匹配系统对 GPU 排序的偏好，但仍存在一些限制。
由于该特性需要新的驱动接口，因此只有来自支持此函数的驱动的物理设备才会被排序。
所有未排序的物理设备都会列在列表末尾，顺序不确定。
此外，只有与某个适配器相对应的物理设备才可以被排序。
这意味着软件驱动很可能不会被排序。
最后，该 API 仅适用于 Windows 系统，并且只会在支持通过操作系统进行 GPU 选择的 Windows 10 版本上工作。
未来可能会纳入其他平台，但它们将需要单独的平台专有接口。

`vk_icdEnumerateAdapterPhysicalDevices` 的一项要求是，它 *必须* 为相同的物理设备返回与 `vkEnumeratePhysicalDevices` 相同的 `VkPhysicalDevice` 句柄值。
这是因为 loader 会在驱动上调用这两个函数，然后使用 `VkPhysicalDevice` 句柄对物理设备进行去重。
由于驱动中的并非所有物理设备都会有 LUID，例如软件实现，因此这一步是必需的，以便让驱动能够枚举所有可用物理设备。

<a id="driver-dispatchable-object-creation"></a>
## 驱动可分发对象创建

如前所述，loader 要求在 Vulkan 可分发对象内部能够访问调度表，例如：`VkInstance`、`VkPhysicalDevice`、`VkDevice`、`VkQueue` 和 `VkCommandBuffer`。
驱动创建的所有可分发对象的具体要求如下：

- 驱动创建的所有可分发对象都可以转换为 void \*\*
- loader 会将第一项替换为指向其自身拥有的调度表的指针
这对驱动意味着三点：
  1. 对于不透明可分发对象句柄，驱动必须返回一个指针
  2. 该指针指向一个普通的 C 结构体，其第一项为一个指针。
   * **注意：** 对于任何将 VK 对象直接实现为 C\++ 类的 C\++ 驱动：
      * 如果类因使用虚函数而成为非 POD，C\++ 编译器可能会在偏移量 0 处放置一个 vtable。
      * 在这种情况下，请使用普通的 C 结构体（见下文）。
  3. loader 会按如下方式检查所有已创建可分发对象中的 magic value（ICD\_LOADER\_MAGIC）（参见 `include/vulkan/vk_icd.h`）：

```cpp
#include "vk_icd.h"

union _VK_LOADER_DATA {
  uintptr loadermagic;
  void *  loaderData;
} VK_LOADER_DATA;

vkObj
   alloc_icd_obj()
{
  vkObj *newObj = alloc_obj();
  ...
  // Initialize pointer to loader's dispatch table with ICD_LOADER_MAGIC

  set_loader_magic_value(newObj);
  ...
  return newObj;
}
```

<a id="handling-khr-surface-objects-in-wsi-extensions"></a>
## 在 WSI 扩展中处理 KHR Surface 对象

通常，驱动负责处理各种 Vulkan 对象的创建和销毁。
Linux、Windows、macOS 和 QNX 的 WSI surface 扩展（"VK\_KHR\_win32\_surface"、"VK\_KHR\_xcb\_surface"、"VK\_KHR\_xlib\_surface"、"VK\_KHR\_wayland\_surface"、"VK\_MVK\_macos\_surface"、"VK\_QNX\_screen\_surface" 和 "VK\_KHR\_surface"）的处理方式则不同。
对于这些扩展，`VkSurfaceKHR` 对象的创建和销毁既可以由 loader 处理，也可以由驱动处理。

如果由 loader 管理 `VkSurfaceKHR` 对象：

  1. loader 会在不涉及驱动的情况下处理对 `vkCreateXXXSurfaceKHR` 和 `vkDestroySurfaceKHR` 函数的调用。
     * 其中 XXX 代表窗口系统名称：
       * Wayland
       * XCB
       * Xlib
       * Windows
       * Android
       * MacOS (`vkCreateMacOSSurfaceMVK`)
       * QNX (`vkCreateScreenSurfaceQNX`)
  2. loader 会为相应的 `vkCreateXXXSurfaceKHR` 调用创建一个 `VkIcdSurfaceXXX` 对象。
     * `VkIcdSurfaceXXX` 结构体定义在 `include/vulkan/vk_icd.h` 中。
  3. 驱动可以将任何 `VkSurfaceKHR` 对象转换为指向适当 `VkIcdSurfaceXXX` 结构体的指针。
  4. 所有 `VkIcdSurfaceXXX` 结构体的第一个字段都是一个 `VkIcdSurfaceBase` 枚举值，用于指示 surface 对象是 Win32、XCB、Xlib、Wayland 还是 Screen。

驱动也可以选择自行处理 `VkSurfaceKHR` 对象的创建。
如果驱动希望负责其创建和销毁，则必须做到以下几点：

  1. 支持 loader 与驱动接口版本 3 或更高版本。
  2. 暴露并处理所有接收 `VkSurfaceKHR` 对象作为参数的函数，包括：
      * `vkCreateXXXSurfaceKHR`
      * `vkGetPhysicalDeviceSurfaceSupportKHR`
      * `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`
      * `vkGetPhysicalDeviceSurfaceFormatsKHR`
      * `vkGetPhysicalDeviceSurfacePresentModesKHR`
      * `vkCreateSwapchainKHR`
      * `vkDestroySurfaceKHR`

由于 `VkSurfaceKHR` 对象是 instance 级对象，一个对象可以与多个驱动相关联。
因此，当 loader 接收到 `vkCreateXXXSurfaceKHR` 调用时，它仍会创建一个内部 `VkSurfaceIcdXXX` 对象。
该对象充当每个驱动版本 `VkSurfaceKHR` 对象的容器。
如果某个驱动不支持创建它自己的 `VkSurfaceKHR` 对象，那么 loader 的容器会为该驱动存储一个 `NULL`。
另一方面，如果驱动支持创建 `VkSurfaceKHR`，loader 就会向该驱动发起适当的 `vkCreateXXXSurfaceKHR` 调用，并将返回的指针存储在其容器对象中。
然后，loader 会将 `VkSurfaceIcdXXX` 作为 `VkSurfaceKHR` 对象沿调用链向上返回。
最后，当 loader 接收到 `vkDestroySurfaceKHR` 调用时，它随后会为每个内部 `VkSurfaceKHR` 对象不为 `NULL` 的驱动调用 `vkDestroySurfaceKHR`。
之后，loader 会在返回前销毁该容器对象。

<a id="loader-and-driver-interface-negotiation"></a>
## loader 与驱动接口协商

一般来说，对于应用发出的函数，loader 可以被视为一个透传层。
也就是说，loader 通常不会修改函数或其参数，而只是调用该函数对应的驱动入口点。
驱动还需要遵守一些额外的接口要求，而这些要求不属于 Vulkan 规范中的任何要求。
这些额外要求带有版本号，以便未来保留灵活性。

<a id="windows-linux-and-macos-driver-negotiation"></a>
### Windows、Linux 和 macOS 驱动协商

<a id="version-negotiation-between-the-loader-and-drivers"></a>
#### loader 与驱动之间的版本协商

所有支持 loader 与驱动接口版本 2 或更高版本的驱动，都必须导出以下函数，用于确定将使用的接口版本。
该入口点并不是 Vulkan API 本身的一部分，而只是 loader 与驱动之间的私有接口。
注意：loader 与驱动接口版本 7 使导出 `vk_icdNegotiateLoaderICDInterfaceVersion` 变为可选。
相反，驱动 **必须** 通过 `vk_icdGetInstanceProcAddr` 暴露它。

```cpp
VKAPI_ATTR VkResult VKAPI_CALL
   vk_icdNegotiateLoaderICDInterfaceVersion(
      uint32_t* pSupportedVersion);
```

该函数允许 loader 与驱动就要使用的接口版本达成一致。
"pSupportedVersion" 参数同时是输入参数和输出参数。
loader 会用其自身支持的、期望使用的最新接口版本（通常是最新版本）填充 "pSupportedVersion"。
驱动接收该值后，在同一字段中返回它期望使用的版本。
由于它是在设置 loader 与驱动之间的接口版本，因此这应是 loader 对驱动发出的第一个调用（甚至早于对 `vk_icdGetInstanceProcAddr` 的任何调用）。

如果接收该调用的驱动由于弃用而不再支持 loader 提供的接口版本，那么它应报告 `VK_ERROR_INCOMPATIBLE_DRIVER` 错误。
否则，它会将 "pSupportedVersion" 所指向的值设置为驱动和 loader 共同支持的最新接口版本，并返回 `VK_SUCCESS`。

如果 loader 提供的接口版本比驱动支持的版本更新，驱动也应报告 `VK_SUCCESS`，因为由 loader 负责确定它是否能够支持驱动所支持的旧接口版本。
如果驱动的接口版本高于 loader，它也应报告 `VK_SUCCESS`，但返回 loader 的版本。
因此，在返回 `VK_SUCCESS` 时，"pSupportedVersion" 将包含驱动要使用的目标接口版本。

如果 loader 从驱动接收到一个其由于弃用而不再支持的接口版本，或者它收到的是 `VK_ERROR_INCOMPATIBLE_DRIVER` 错误而不是 `VK_SUCCESS`，那么 loader 会将该驱动视为不兼容，并且不会加载它供使用。
在这种情况下，应用在枚举期间将看不到该驱动的 `vkPhysicalDevice`。

<a id="interfacing-with-legacy-drivers-or-loaders"></a>
#### 与旧版驱动或 loader 的接口交互

如果 loader 发现某个驱动没有导出或暴露 `vk_icdNegotiateLoaderICDInterfaceVersion` 函数，那么 loader 会假定对应驱动仅支持接口版本 0 或 1。

从接口的另一侧来看，如果驱动在收到对 `vk_icdNegotiateLoaderICDInterfaceVersion` 的调用之前先收到了对 `vk_icdGetInstanceProcAddr` 的调用，那么该 loader 要么是仅支持接口版本 0 或 1 的旧版 loader，要么是正在使用接口版本 7 或更高版本的 loader。

如果第一次对 `vk_icdGetInstanceProcAddr` 的调用是为了查询 `vk_icdNegotiateLoaderICDInterfaceVersion`，那么这意味着 loader 正在使用接口版本 7。
这只会在驱动不导出 `vk_icdNegotiateLoaderICDInterfaceVersion` 时发生。
对于导出 `vk_icdNegotiateLoaderICDInterfaceVersion` 的驱动，首先被调用的会是它。

如果第一次对 `vk_icdGetInstanceProcAddr` 的调用**不是**查询 `vk_icdNegotiateLoaderICDInterfaceVersion`，那么该 loader 就是一个仅支持版本 0 或 1 的旧版 loader。
在这种情况下，如果 loader 首先调用 `vk_icdGetInstanceProcAddr`，则它至少支持接口版本 1。
否则，loader 仅支持版本 0。

<a id="loader-and-driver-interface-version-7-requirements"></a>
#### loader 与驱动接口版本 7 要求

版本 7 放宽了 loader 与驱动接口函数必须被导出的要求。
相反，它只要求这些函数能够通过 `vk_icdGetInstanceProcAddr` 查询到。
这些函数是：
    `vk_icdNegotiateLoaderICDInterfaceVersion`
    `vk_icdGetPhysicalDeviceProcAddr`
    `vk_icdEnumerateAdapterPhysicalDevices`（仅 Windows）
出于获取目的，这些函数都被视为全局函数，因此 `vk_icdGetInstanceProcAddr` 的 `VkInstance` 参数将为 **NULL**。
尽管导出这些函数不再是要求，驱动仍然可以出于与旧版 loader 的兼容性而导出它们。
这一版本中的变化使通过 `VK_LUNARG_direct_driver_loading` 扩展提供的驱动能够支持完整的 loader 与驱动接口。

<a id="loader-and-driver-interface-version-6-requirements"></a>
#### loader 与驱动接口版本 6 要求

版本 6 提供了一种让 loader 对物理设备进行排序的机制。
只有在某个驱动支持接口版本 6 时，loader 才会尝试对该驱动上的物理设备进行排序。
此版本提供了本文前面定义的 `vk_icdEnumerateAdapterPhysicalDevices` 函数。

<a id="loader-and-driver-interface-version-5-requirements"></a>
#### loader 与驱动接口版本 5 要求

此接口版本对实际接口没有任何更改。
如果 loader 请求接口版本 5 或更高版本，这只是向驱动表明：loader 现在会评估传入 vkCreateInstance 的 API 版本 信息对于 loader 是否为有效版本。
如果不是，loader 会在 vkCreateInstance 期间捕获这一情况，并以 `VK_ERROR_INCOMPATIBLE_DRIVER` 错误失败。

另一方面，如果 loader 没有请求版本 5 或更高版本，那么这表明驱动所请求的 API 版本对 loader 来说是未知的。
因此，就需要由驱动来验证该 API 版本 是否不大于 major = 1 且 minor = 0。
如果超出该范围，那么驱动应自动以 `VK_ERROR_INCOMPATIBLE_DRIVER` 错误失败，因为该 loader 是 1.0 loader，并且不知道该版本。

以下表格给出了预期行为：

<table style="width:100%">
  <tr>
    <th>loader 支持的 I/f 版本</th>
    <th>驱动支持的 I/f 版本</th>
    <th>结果</th>
  </tr>
  <tr>
    <td>4 或更早</td>
    <td>任意版本</td>
    <td>对于所有 apiVersion 设置为 &gt; Vulkan 1.0 的 vkCreateInstance 调用，驱动<b>必须失败</b>并返回 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>，因为 loader 仍处于接口版本 &lt;= 4。<br/>
        否则，驱动应表现正常。
    </td>
  </tr>
  <tr>
    <td>5 或更新</td>
    <td>4 或更早</td>
    <td>如果 loader 无法处理该 apiVersion，则它<b>必须失败</b>并返回 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>。
        驱动可以对所有 apiVersion 都通过，但由于其接口版本
        &lt;= 4，最好假定它需要负责拒绝任何 &gt; Vulkan 1.0 的情况，并以 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b> 失败。
        <br/>
        否则，驱动应表现正常。
    </td>
  </tr>
  <tr>
    <td>5 或更新</td>
    <td>5 或更新</td>
    <td>如果 loader 无法处理该 apiVersion，则它<b>必须失败</b>并返回 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>；而驱动应仅在其不能支持指定 apiVersion 时，<i>才</i>以 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b> 失败。<br/>
        否则，驱动应表现正常。
    </td>
  </tr>
</table>

<a id="loader-and-driver-interface-version-4-requirements"></a>
#### loader 与驱动接口版本 4 要求

此接口版本第 4 版的主要变化，是使用 `vk_icdGetPhysicalDeviceProcAddr` 函数来支持[未知物理设备扩展](#driver-unknown-physical-device-extensions)。
该函数纯属可选。
但是，如果驱动支持某个物理设备扩展，它就必须提供 `vk_icdGetPhysicalDeviceProcAddr` 函数。
否则，loader 会继续将任何未知函数视为 VkDevice 函数，从而导致无效行为。

<a id="loader-and-driver-interface-version-3-requirements"></a>
#### loader 与驱动接口版本 3 要求

此接口版本中的主要变化，是允许驱动处理其自身 KHR_surfaces 的创建和销毁。
在此之前，loader 会创建一个由所有驱动共用的 surface 对象。
但是，某些驱动 *可以* 希望提供它们自己的 surface 句柄。
如果驱动选择启用此支持，它必须支持 loader 与驱动接口版本 3，以及任何使用 `VkSurfaceKHR` 句柄的 Vulkan 函数，例如：

- `vkCreateXXXSurfaceKHR`（其中 XXX 是平台专有标识符 [即 Windows 的 `vkCreateWin32SurfaceKHR`]）
- `vkDestroySurfaceKHR`
- `vkCreateSwapchainKHR`
- `vkGetPhysicalDeviceSurfaceSupportKHR`
- `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`
- `vkGetPhysicalDeviceSurfaceFormatsKHR`
- `vkGetPhysicalDeviceSurfacePresentModesKHR`

不参与该功能的驱动可以通过简单地不暴露上述 `vkCreateXXXSurfaceKHR` 和 `vkDestroySurfaceKHR` 函数来选择退出。

<a id="loader-and-driver-interface-version-2-requirements"></a>
#### loader 与驱动接口版本 2 要求

接口版本 2 要求驱动导出 `vk_icdNegotiateLoaderICDInterfaceVersion`。
更多信息，请参见[loader 与驱动之间的版本协商](#version-negotiation-between-loader-and-drivers)。

此外，版本 2 还要求驱动创建的 Vulkan 可分发对象必须按照[驱动可分发对象创建](#driver-dispatchable-object-creation)一节中的要求进行创建。

<a id="loader-and-driver-interface-version-1-requirements"></a>
#### loader 与驱动接口版本 1 要求

接口版本 1 添加了驱动专用入口点 `vk_icdGetInstanceProcAddr`。
由于这早于 `vk_icdNegotiateLoaderICDInterfaceVersion` 入口点的创建，loader 没有协商过程来确定驱动支持哪个接口版本。
因此，loader 通过缺少协商函数但存在 `vk_icdGetInstanceProcAddr` 的情况，来检测对接口版本 1 的支持。
驱动不需要导出其他入口点，因为 loader 会使用该函数查询相应的函数指针。

<a id="loader-and-driver-interface-version-0-requirements"></a>
#### loader 与驱动接口版本 0 要求

版本 0 不支持 `vk_icdGetInstanceProcAddr` 或 `vk_icdNegotiateLoaderICDInterfaceVersion`。
因此，除非存在其中之一，否则 loader 会假定驱动只支持接口版本 0。

此外，对于版本 0，驱动必须至少暴露以下核心 Vulkan 入口点，以便 loader 构建到驱动的接口：

- 驱动库中 **必须导出** 函数 `vkGetInstanceProcAddr`，并且它要为所有 Vulkan API 入口点返回有效的函数指针。
- 驱动库 **必须导出** `vkCreateInstance`。
- 驱动库 **必须导出** `vkEnumerateInstanceExtensionProperties`。

<a id="additional-interface-notes"></a>
#### 额外接口说明：

- loader 会在调用驱动之前，先过滤 `vkCreateInstance` 和 `vkCreateDevice` 中请求的扩展；所过滤的是由不同于相关驱动的实体（例如 layer）公开的扩展。
- loader 不会为 `vkEnumerate*LayerProperties` 调用驱动，因为 layer 属性是从 layer 库和 layer JSON 文件中获取的。
- 如果驱动库作者想要实现一个 layer，可以通过让相应的 layer JSON 清单文件引用该驱动库文件来实现。
- 如果 "pLayerName" 不等于 `NULL`，loader 将不会为 `vkEnumerate*ExtensionProperties` 调用驱动。
- 通过 device 扩展创建新可分发对象的驱动需要初始化新创建的可分发对象。
loader 对未知 device 扩展具有通用 *trampoline* 代码。
这段通用 *trampoline* 代码不会初始化新创建对象内部的调度表。
有关如何为 loader 不认识的扩展初始化新创建可分发对象的更多信息，请参见[驱动可分发对象创建](#driver-dispatchable-object-creation)一节。

<a id="android-driver-negotiation"></a>
### Android 驱动协商

Android loader 使用与上文所述相同的协议来初始化调度表。
唯一的区别在于，Android loader 会直接从各自的库中查询 layer 和扩展信息，而不使用 Windows、Linux 和 macOS loader 所使用的 JSON 清单文件。

<a id="loader-implementation-of-vk_khr_portability_enumeration"></a>
## loader 对 VK_KHR_portability_enumeration 的实现

loader 实现了 `VK_KHR_portability_enumeration` instance 扩展，该扩展会过滤掉任何报告支持 portability subset device 扩展的驱动。
除非应用通过在 `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` 的 VkInstanceCreateInfo::flags 中设置该位，显式请求枚举 portability 设备，否则 loader 不会加载任何声明自己为 portability 驱动的驱动。

驱动会在驱动清单 JSON 文件中通过 `is_portability_driver` 布尔字段声明自己是否为 portability 驱动。
[更多信息见此处](#driver-manifest-file-version-101)

对该扩展的初始支持只会在应用未启用 portability enumeration 特性时报告错误。
它不会过滤掉 portability 驱动。
这样做是为了给应用一段宽限期，使其能够更新 instance 创建逻辑，而不会直接破坏应用。

<a id="loader-and-driver-policy"></a>
## loader 和驱动策略

本节旨在定义 loader 与驱动之间应遵循的正确行为。
本节中的大部分内容都是对 Vulkan 规范的补充，并且对于维持跨平台一致性是必需的。
事实上，其中许多表述可以在本文档各处找到，这里为了方便而进行了汇总。
此外，还应有一种方法来识别驱动中的不良行为或不符合规范的行为，并尽快加以补救。
因此，这里提供了一个策略编号系统，以唯一方式清楚标识每一条策略陈述。

最后，基于使 loader 高效且高性能这一目标，这些用于定义正确驱动行为的策略陈述中，有些可能无法被测试（因此 loader 无法强制执行）。
不过，这不应削弱这些要求对于向最终用户和开发者提供最佳体验的重要性。

<a id="number-format"></a>
### 编号格式

loader 和驱动策略项以 `LDP_` 前缀开头（即 Loader and Driver Policy 的缩写），后跟一个标识符，该标识符基于策略所针对的组件。
在这里，只有两个可能的组件：

  - 驱动：其策略编号中将包含字符串 `DRIVER_`。
  - loader：其策略编号中将包含字符串 `LOADER_`。
<a id="android-differences"></a>
### Android 差异

如前所述，Android Loader 实际上独立于 Khronos Loader。
因此，出于这一点以及其他平台要求，并非这些策略声明中的所有内容都适用于 Android。
每个表格还包含一列标题为“Applicable to Android?”，
用于指示哪些策略声明适用于仅关注 Android 支持的驱动。
有关 Android loader 的更多信息可参见
<a href="https://source.android.com/devices/graphics/implement-vulkan">
Android Vulkan 文档</a>。

<a id="requirements-of-well-behaved-drivers"></a>
### 行为良好的驱动要求

<table style="width:100%">
  <tr>
    <th>要求编号</th>
    <th>要求说明</th>
    <th>不符合要求的结果</th>
    <th>适用于 Android？</th>
    <th>可由 loader 强制执行？</th>
    <th>参考章节</th>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_1</b></small></td>
    <td>驱动 <b>不得</b> 导致其他驱动失败、崩溃，
        或以其他方式行为异常。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是</td>
    <td>否</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_2</b></small></td>
    <td>当使用任何 Vulkan instance 或 physical device API
        对该驱动发起调用时，如果驱动检测到系统上不存在受支持的
        Vulkan Physical Device（<i>VkPhysicalDevice</i>），驱动 <b>不得</b> 崩溃。<br/>
        这是因为某些设备可以热插拔。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是</td>
    <td>否<br/>
        loader 并不直接知道某个给定驱动可能支持哪些设备（虚拟或物理）。</td>
    <td><small>N/A</small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_3</b></small></td>
    <td>驱动 <b>必须</b> 能够按照规定的协商流程，与 loader
        协商出受支持的 loader 与驱动接口版本。
    </td>
    <td>该驱动将不会被加载。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#loader-and-driver-interface-negotiation">
        接口协商</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_4</b></small></td>
    <td>驱动 <b>必须</b> 具有供 loader 处理的有效 JSON 清单文件，
        且该文件必须以 ".json" 后缀结尾。
    </td>
    <td>该驱动将不会被加载。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#driver-manifest-file-format">清单文件格式</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_5</b></small></td>
    <td>驱动 <b>必须</b> 先通过一致性测试，并且其结果已由 Khronos
        提交、验证并批准，然后才能通过 Vulkan 提供的任何机制报告一致性版本
        （示例包括在 <i>VkPhysicalDeviceVulkan12Properties</i> 和
        <i>VkPhysicalDeviceDriverProperties</i> 结构体内报告）。<br/>
        否则，当遇到此类包含一致性版本的结构体时，驱动 <b>必须</b> 返回
        0.0.0.0 的一致性版本，以表明其尚未经过上述验证和批准。
    </td>
    <td>是</td>
    <td>否</td>
    <td>loader 和/或应用程序可能会对驱动的能力作出假设，
        从而导致未定义行为，可能包括崩溃或损坏。
    </td>
    <td><small>
        <a href="https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/openglcts/README.md">
        Vulkan CTS 文档</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_6</b></small></td>
    <td>已移除 - 参见
        <a href="#removed-driver-policies">已移除的驱动策略</a>
    </td>
    <td>-</td>
    <td>-</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_7</b></small></td>
    <td>如果驱动希望支持 Vulkan API 1.1 或更新版本，则它 <b>必须</b>
        公开对 loader 与驱动接口版本 5 或更新版本的支持。
    </td>
    <td>该驱动会在不应被使用时仍被使用，并将导致
        未定义行为，可能包括崩溃或损坏。
    </td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#loader-version-5-interface-requirements">
        版本 5 接口要求</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_8</b></small></td>
    <td>如果驱动希望处理其自身的 <i>VkSurfaceKHR</i> 对象创建，
        则它 <b>必须</b> 实现 loader 与驱动接口版本 3 或更新版本，
        并支持通过 <i>vk_icdGetInstanceProcAddr</i> 查询所有相关 surface 函数。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td>是</td>
    <td><small>
        <a href="#handling-khr-surface-objects-in-wsi-extensions">
        处理 KHR Surface 对象</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_9</b></small></td>
    <td>如果版本协商导致驱动使用 loader 与驱动接口版本 4
        或更早版本，则驱动 <b>必须</b> 验证传入 <i>vkCreateInstance</i>
        的 Vulkan API 版本（通过 <i>VkInstanceCreateInfo</i> 的
        <i>VkApplicationInfo</i> 的 <i>apiVersion</i>）是否受支持。
        如果驱动无法支持所请求的 Vulkan API 版本，
        则它 <b>必须</b> 返回 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>。 <br/>
        如果接口版本为 5 或更新版本，则不要求这样做，因为该检查由
        loader 负责。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td>否</td>
    <td><small>
        <a href="#loader-version-5-interface-requirements">
        版本 5 接口要求</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_10</b></small></td>
    <td>如果版本协商导致驱动使用 loader 与驱动接口版本 5 或更新版本，则如果传入 <i>vkCreateInstance</i>
        的 Vulkan API 版本（通过 <i>VkInstanceCreateInfo</i> 的
        <i>VkApplicationInfo</i> 的 <i>apiVersion</i>）不受驱动支持，
        驱动 <b>不得</b> 返回 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>。
        该检查由 loader 代表驱动执行。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td>否</td>
    <td><small>
        <a href="#loader-version-5-interface-requirements">
        版本 5 接口要求</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_11</b></small></td>
    <td>驱动在卸载时 <b>必须</b> 移除所有清单文件以及对这些文件的引用
        （即 Windows 上的 Registry 条目）。
        <br/>
        同样，在更新驱动文件时，旧文件 <b>必须</b> 全部更新或移除。
    </td>
    <td>如果旧文件仍指向不正确的库，
        将导致未定义行为，可能包括崩溃或损坏。
    </td>
    <td>否</td>
    <td>否<br/>
        loader 无法知道哪些驱动文件是新的、旧的或不正确的。
        任何类型的驱动文件验证都会很快变得非常复杂，
        因为这将要求 loader 维护一个内部数据库，
        根据驱动供应商、驱动版本、目标平台以及可能的其他条件，
        跟踪行为不良的驱动。
    </td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_12</b></small></td>
    <td>为了与公开的 Khronos Loader 正常配合，驱动
        <b>不得</b> 在未先通过 Khronos 发布的情况下公开平台接口扩展。<br/>
        开发中的平台可以使用 Khronos Loader 的修改版本，直到设计变得稳定和/或公开。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是（特别是针对 Android 扩展）</td>
    <td>否</td>
    <td><small>N/A</small></td>
  </tr>
</table>

<a id="removed-driver-policies"></a>
#### 已移除的驱动策略

这些策略曾在某个时期存在于 loader 源代码中，但后来被移除。
这里将其记录下来以供参考。

<table>
  <tr>
    <th>要求编号</th>
    <th>要求说明</th>
    <th>移除原因</th>
  </tr>
  <tr>
    <td><small><b>LDP_DRIVER_6</b></small></td>
    <td>支持 loader 与驱动接口版本 1 或更新版本的驱动 <b>不得</b>
        直接导出标准 Vulkan 入口点。
        <br/>
        相反，它 <b>必须</b> 只导出其所支持接口版本要求的 loader 接口函数
        （例如 <i>vk_icdGetInstanceProcAddr</i>）。 <br/>
        这是因为某些平台上的动态链接过去曾出现问题，
        有时会错误地链接到错误动态库中导出的函数。 <br/>
        <b>注意：</b> 实际上，这一点适用于所有导出项。
        如果拿不准，不要从驱动中导出任何可能在其他库中引发冲突的项。<br/>
    </td>
    <td>
        由于在某些有效场景下，驱动需要导出核心入口点，
        因此该策略已被移除。
        此外，在实践中也未发现动态链接会导致很多问题。
    </td>
  </tr>
</table>

<a id="requirements-of-a-well-behaved-loader"></a>
### 行为良好的 loader 要求

<table style="width:100%">
  <tr>
    <th>要求编号</th>
    <th>要求说明</th>
    <th>不符合要求的结果</th>
    <th>适用于 Android？</th>
    <th>参考章节</th>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_1</b></small></td>
    <td>如果 loader 无法在系统上找到并加载有效的 Vulkan 驱动，
        则它 <b>必须</b> 返回 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是</td>
    <td><small>N/A</small></td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_2</b></small></td>
    <td>对于 loader 发现并认定其格式符合本文档的任何驱动清单文件，
        loader <b>必须</b> 尝试加载它。 
        <br/>
        <b>唯一</b> 的例外是平台通过其他机制确定驱动位置和功能的情况。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是</td>
    <td><small>
        <a href="#driver-discovery">驱动发现</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_3</b></small></td>
    <td>loader <b>必须</b> 支持某种机制，以从一个或多个非标准位置
        加载驱动。<br/>
        这是为了支持纯软件驱动，以及评估开发中的 ICD。 <br/>
        对该规则的 <b>唯一</b> 例外是 OS 因安全策略而不希望支持此功能。
    </td>
    <td>这将使某些工具和驱动开发者更难使用 Vulkan loader。</td>
    <td>否</td>
    <td><small>
        <a href="#using-pre-production-icds-or-software-drivers">
        预发布 ICD 或软件驱动</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_4</b></small></td>
    <td>loader <b>不得</b> 加载定义了与其自身不兼容 API 版本的 Vulkan 驱动。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是</td>
    <td><small>
        <a href="#driver-discovery">驱动发现</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_5</b></small></td>
    <td>对于无法协商出兼容 loader 与驱动接口版本的任何驱动，
        loader <b>必须</b> 忽略它。
    </td>
    <td>loader 会以不正确的方式加载驱动，从而导致未定义行为，
        可能包括崩溃或损坏。
    </td>
    <td>否</td>
    <td><small>
        <a href="#loader-and-driver-interface-negotiation">
        接口协商</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_6</b></small></td>
    <td>如果驱动协商导致 loader 使用 loader 与驱动接口版本 5 或更新版本，则 loader <b>必须</b> 验证传入
        <i>vkCreateInstance</i> 的 Vulkan API 版本（通过
        <i>VkInstanceCreateInfo</i> 的 <i>VkApplicationInfo</i> 的
        <i>apiVersion</i>）是否至少受一个驱动支持。
        如果任何驱动都无法支持所请求的 Vulkan API 版本，
        则 loader <b>必须</b> 返回
        <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>。<br/>
        如果 loader 与驱动接口版本为 4 或更早版本，则不要求这样做，
        因为该检查由驱动负责。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#loader-version-5-interface-requirements">
        版本 5 接口要求</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_7</b></small></td>
    <td>如果系统中存在多个驱动，并且其中一些驱动 <i>仅</i> 支持
        Vulkan API 版本 1.0，而其他驱动支持更新的 Vulkan API 版本，
        则 loader <b>必须</b> 将仅知道 Vulkan API 版本 1.0 的所有驱动所使用的
        <i>VkInstanceCreateInfo</i> 的 <i>VkApplicationInfo</i> 中的
        <i>apiVersion</i> 字段调整为版本 1.0。<br/>
        否则，仅支持 Vulkan API 版本 1.0 的驱动会在
        <i>vkCreateInstance</i> 期间返回 <b>VK_ERROR_INCOMPATIBLE_DRIVER</b>，
        因为 1.0 驱动并不了解未来版本。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#driver-api-version">驱动 API 版本</a>
        </small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_8</b></small></td>
    <td>如果存在多个驱动，并且至少有一个驱动 <i>不支持</i>
        其他驱动所支持的 instance 级功能；
        则 loader <b>必须</b> 以某种方式为这些不支持的驱动提供该 instance 级功能。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#loader-instance-extension-emulation-support">
        loader 的 instance 扩展模拟支持</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_9</b></small></td>
    <td>在调用驱动的 <i>vkCreateInstance</i> 期间，loader <b>必须</b>
        从 <i>VkInstanceCreateInfo</i> 结构体的 <i>ppEnabledExtensionNames</i>
        字段中过滤掉驱动不支持的实例扩展。<br/>
        这是因为应用程序无法知道哪些驱动支持哪些扩展。<br/>
        这也与上面的 <i>LDP_LOADER_8</i> 直接相关。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#filtering-out-instance-extension-names">
        过滤掉 instance 扩展名</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_10</b></small></td>
    <td>loader <b>必须</b> 支持创建可由所有底层驱动共享的
        <i>VkSurfaceKHR</i> 句柄。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是</td>
    <td><small>
        <a href="#handling-khr-surface-objects-in-wsi-extensions">
        处理 KHR Surface 对象</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_11</b></small></td>
    <td>如果驱动公开了适当的 <i>VkSurfaceKHR</i> 创建/处理入口点，
        则 loader <b>必须</b> 支持创建该驱动专用的 surface 对象句柄，
        并在请求时将其而不是共享的 <i>VkSurfaceKHR</i> 句柄回传给该驱动。
        <br/>
        否则，loader <b>必须</b> 提供由 loader 创建的
        <i>VkSurfaceKHR</i> 句柄。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>否</td>
    <td><small>
        <a href="#handling-khr-surface-objects-in-wsi-extensions">
        处理 KHR Surface 对象</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_12</b></small></td>
    <td>如果 <i>pLayerName</i> 不为 <b>NULL</b>，loader <b>不得</b>
        调用驱动中的任何 <i>vkEnumerate*ExtensionProperties</i> 入口点。
    </td>
    <td>其行为未定义，并且可能导致崩溃或损坏。</td>
    <td>是</td>
    <td><small>
        <a href="#additional-interface-notes">
        附加接口说明</a></small>
    </td>
  </tr>
  <tr>
    <td><small><b>LDP_LOADER_13</b></small></td>
    <td>当应用程序以提升权限（管理员/超级用户）运行时，
        loader <b>不得</b> 从用户定义的路径加载内容（包括使用
        <i>VK_ICD_FILENAMES</i>、<i>VK_DRIVER_FILES</i> 或
        <i>VK_ADD_DRIVER_FILES</i> 环境变量中的任何一个）。<br/>
        <b>这是出于安全原因。</b>
    </td>
    <td>其行为未定义，并且可能导致计算机安全漏洞、
        崩溃或损坏。
    </td>
    <td>否</td>
    <td><small>
        <a href="#exception-for-administrator-and-super-user-mode">
          管理员和超级用户模式的例外情况
        </a></small>
    </td>
  </tr>
</table>

<br/>

[返回顶层 LoaderInterfaceArchitecture.md 文件。](LoaderInterfaceArchitecture.md)
