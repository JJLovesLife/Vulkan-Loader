<!-- markdownlint-disable MD041 -->
[![Khronos Vulkan][1]][2]

[1]: https://vulkan.lunarg.com/img/Vulkan_100px_Dec16.png "https://www.khronos.org/vulkan/"
[2]: https://www.khronos.org/vulkan/

# 应用程序与 Loader 的接口（Application Interface to Loader） <!-- omit from toc -->
[![Creative Commons][3]][4]

<!-- Copyright &copy; 2015-2023 LunarG, Inc. -->

[3]: https://i.creativecommons.org/l/by-nd/4.0/88x31.png "Creative Commons License"
[4]: https://creativecommons.org/licenses/by-nd/4.0/

## 目录 <!-- omit from toc -->

- [概述](#overview)
- [与 Vulkan 函数交互](#interfacing-with-vulkan-functions)
  - [Vulkan 直接导出](#vulkan-direct-exports)
  - [直接链接到 Loader](#directly-linking-to-the-loader)
    - [动态链接](#dynamic-linking)
    - [静态链接](#static-linking)
  - [间接链接到 Loader](#indirectly-linking-to-the-loader)
  - [应用程序的最佳性能设置](#best-application-performance-setup)
  - [ABI 版本控制](#abi-versioning)
    - [Windows 动态库用法](#windows-dynamic-library-usage)
    - [Linux 动态库用法](#linux-dynamic-library-usage)
    - [MacOs 动态库用法](#macos-dynamic-library-usage)
  - [随应用程序一同打包 Loader](#bundling-the-loader-with-an-application)
- [应用程序中的 Layer 使用](#application-layer-usage)
  - [元层](#meta-layers)
  - [隐式层与显式层](#implicit-vs-explicit-layers)
    - [覆盖层](#override-layer)
  - [强制指定 Layer 源文件夹](#forcing-layer-source-folders)
    - [提升权限时的例外情况](#exception-for-elevated-privileges)
  - [在 Windows、Linux 和 macOS 上强制启用 Layer](#forcing-layers-to-be-enabled-on-windows-linux-and-macos)
  - [Layer 的整体顺序](#overall-layer-ordering)
  - [调试可能的 Layer 问题](#debugging-possible-layer-issues)
- [应用程序对扩展的使用](#application-usage-of-extensions)
  - [实例扩展和设备扩展](#instance-and-device-extensions)
  - [WSI 扩展](#wsi-extensions)
  - [未知扩展](#unknown-extensions)
  - [过滤掉未知的实例扩展名称](#filtering-out-unknown-instance-extension-names)
- [物理设备排序](#physical-device-ordering)

<a id="overview"></a>
## 概述

本文从以应用程序为中心的视角介绍如何使用 Vulkan loader。
有关 loader 各个部分的完整概览，请参阅
[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文件。

<a id="interfacing-with-vulkan-functions"></a>
## 与 Vulkan 函数交互

可以通过 loader 以多种方式与 Vulkan 函数交互：

<a id="vulkan-direct-exports"></a>
### Vulkan 直接导出

Windows、Linux、Android 和 macOS 上的 loader 库会导出所有核心
Vulkan 入口点，以及所有适用的窗口系统接口（WSI）
入口点。
这样做是为了让开发者更容易开始进行 Vulkan 开发。
当应用程序以这种方式直接链接到 loader 库时，
Vulkan 调用就是简单的 *trampoline* 函数，它们会跳转到为所给对象对应的
调度表条目。

<a id="directly-linking-to-the-loader"></a>
### 直接链接到 Loader

<a id="dynamic-linking"></a>
#### 动态链接

loader 以动态库形式分发（Windows 上为 `.dll`，Linux 上为 `.so`，
macOS 上为 `.dylib`），并安装到系统的动态库搜索路径中。
此外，该动态库通常会作为驱动安装的一部分安装到 Windows
系统中，在 Linux 上通常也会通过系统包管理器提供。
这意味着应用程序通常可以预期系统中已经存在一份 loader。
如果应用程序希望完全确保 loader 存在，
可以随应用程序一起提供 loader 或运行时安装程序。

<a id="static-linking"></a>
#### 静态链接

在 loader 的早期版本中，可以对 loader 进行静态链接。
**该功能已被移除，且不再可用。**
移除静态链接的原因是驱动发生了变化，
这会导致采用静态链接的旧应用程序无法找到较新的
驱动。

此外，静态链接还带来了若干问题：
 - 若不重新链接应用程序，就永远无法更新 loader
 - 两个被包含的库可能各自携带不同版本 loader 的可能性
   - 这可能导致不同 loader 版本之间发生冲突

唯一的例外是 macOS，但该用法不受支持，也未经测试。

<a id="indirectly-linking-to-the-loader"></a>
### 间接链接到 Loader

应用程序并不要求必须直接链接到 loader 库，
它们也可以通过特定平台的动态符号查找机制，在
loader 库上初始化应用程序自己的调度表。
这样一来，如果找不到 loader，应用程序也能优雅地失败。
这也是应用程序调用 Vulkan
函数的最快机制。
应用程序只需要从 loader 库中查询（通过 `dlsym` 之类的系统调用）
`vkGetInstanceProcAddr` 的地址。
然后应用程序使用 `vkGetInstanceProcAddr` 以平台无关的方式
加载所有可用函数，例如 `vkCreateInstance`、`vkEnumerateInstanceExtensionProperties`
和 `vkEnumerateInstanceLayerProperties`。

<a id="best-application-performance-setup"></a>
### 应用程序的最佳性能设置

为了在 Vulkan 应用程序中获得尽可能好的性能，应用程序
应当为每个 Vulkan API 入口点建立自己的调度表。
对于调度表中的每一个实例级 Vulkan 命令，
都应使用 `vkGetInstanceProcAddr` 的返回结果查询并填充其函数指针。
此外，对于每一个设备级 Vulkan 命令，
都应使用 `vkGetDeviceProcAddr` 的返回结果查询并填充其函数指针。

*为什么要这样做？*

答案在于实例函数调用链的实现方式
与设备函数调用链的实现方式不同。
请记住，[Vulkan 实例是用于提供 Vulkan
系统级信息的高级构造](LoaderInterfaceArchitecture.md#instance-specific)。
因此，实例函数需要广播到系统上的每一个可用
驱动。
下图展示了在启用了三个 layer 时，实例调用链的大致情况：

![Instance Call Chain](./images/loader_instance_chain.png)

如果 Vulkan 设备函数是通过
`vkGetInstanceProcAddr` 查询得到的，其调用链也是这样。
另一方面，设备函数不需要考虑广播问题，
因为它明确知道调用应当终止于哪个关联驱动以及哪个关联的
物理设备。
正因如此，在任何已启用的
layer 与驱动之间，loader 无需介入。
因此，在上述相同场景下，使用由 loader 导出的 Vulkan 设备函数时，
调用链将如下所示：

![Loader Device Call Chain](./images/loader_device_chain_loader.png)

更好的方案是让应用程序对所有设备函数都执行
`vkGetDeviceProcAddr` 调用。
这会进一步优化调用链，在大多数场景下
将 loader 完全移除：

![Application Device Call Chain](./images/loader_device_chain_app.png)

另外请注意，如果没有启用任何 layer，应用程序的函数指针会
**直接指向驱动**。
对于大量函数调用来说，每次调用中少掉的一层间接跳转
累积起来会带来不可忽视的性能收益。

**注意：** 仍有一些设备函数需要 loader
通过 *trampoline* 和 *terminator* 进行拦截。
这类函数非常少，但通常是那些 loader
会用自己的数据进行包装的函数。
在这些情况下，即使是设备调用链，也仍会看起来像
实例调用链。
一个需要 *terminator* 的设备函数示例是
`vkCreateSwapchainKHR`。
对于该函数，loader 在将函数的其余信息继续传递给驱动之前，
可能需要先将 KHR_surface 对象转换为驱动特定的
KHR_surface 对象。

请记住：
 * `vkGetInstanceProcAddr` 用于查询实例函数和物理设备
   函数，但它也可以查询所有函数。
 * `vkGetDeviceProcAddr` 仅用于查询设备函数。

<a id="abi-versioning"></a>
### ABI 版本控制

Vulkan loader 库会通过多种方式分发，包括 Vulkan
SDK、操作系统软件包发行版以及独立硬件供应商（IHV）驱动
软件包。
这些细节超出了本文档的范围。
不过，Vulkan loader 库的名称和版本控制方式是有规定的，
以便应用程序能够链接到正确的 Vulkan ABI 库版本。
对于主版本号相同的所有版本（例如 1.0 和 1.1），都保证 ABI
向后兼容。

<a id="windows-dynamic-library-usage"></a>
#### Windows 动态库用法

在 Windows 上，loader 库会将 ABI 版本编码进其名称中，
从而使多个 ABI 不兼容版本的 loader 能够在同一系统上
和平共存。
Vulkan loader 库文件名格式为 `vulkan-<ABI version>.dll`。
例如，在 Windows 上，对于 Vulkan 版本 1.X，库文件名为
`vulkan-1.dll`。
该库文件通常位于 `windows\system32`
目录中（在 64 位 Windows 安装中，
同名的 32 位 loader 版本可在 `windows\sysWOW64` 目录中找到）。

<a id="linux-dynamic-library-usage"></a>
#### Linux 动态库用法

在 Linux 上，共享库通过后缀进行版本控制。
因此，ABI 号不会像 Windows 那样编码在
库文件名的基础部分中。

在 Linux 上，若应用程序对 Vulkan 有硬依赖，
应在其构建系统中请求链接到无版本名 `libvulkan.so`。
例如，可以导入 CMake 目标 `Vulkan::Vulkan`，或使用
`pkg-config --cflags --libs vulkan` 的输出作为编译器标志。
与 Linux 库的通常做法一样，编译器和链接器会将其解析为
对正确带版本 SONAME 的依赖，目前为 `libvulkan.so.1`。
在运行时动态加载 Vulkan-Loader 的 Linux 应用程序
无法从该机制中获益，因此应确保传递给 `dlopen()` 的是
带版本的名称，例如 `libvulkan.so.1`，
以确保加载兼容版本。

<a id="macos-dynamic-library-usage"></a>
#### MacOs 动态库用法

MacOs 上的链接方式与 Linux 类似，不同之处在于标准
动态库名称为 `libvulkan.dylib`，而 ABI 带版本的库
当前名为 `libvulkan.1.dylib`。

<a id="bundling-the-loader-with-an-application"></a>
### 随应用程序一同打包 Loader

Khronos loader 通常会以平台特定方式安装在各个平台上
（例如 Linux 上的软件包），或作为驱动安装的一部分
（例如 Windows 上的 Vulkan Runtime 安装程序）。
应用程序或引擎可能希望在其自身的安装过程中，
将 Vulkan loader 安装到其执行目录树中。
这可能是因为提供特定版本的 loader：

  1) 能保证 loader 中提供某些 Vulkan API 导出
  2) 能确保某些 loader 行为是已知且稳定的
  3) 能在用户安装之间提供一致性

但是，**强烈不建议** 这样做，因为：

  1) 打包的 loader 可能与未来的驱动版本不兼容
(这一点在 Windows 上尤其明显，因为驱动安装位置可能会在
操作系统更新期间发生变化)
  2) 它可能阻止应用程序/引擎利用新的 Vulkan
API 版本/扩展导出
  3) 应用程序/引擎将错过重要的 loader 缺陷修复
  4) 打包的 loader 将不包含有用的功能更新（例如
改进的 loader 调试能力)

当然，即使应用程序/引擎最初确实随产品发布了某个特定版本的
Khronos loader，它也可能会在未来某个时间点
更新或移除该 loader。
这可能是因为随着时间推移，loader 暴露出了所需功能。
但是，这依赖最终用户在未来正确执行所需的更新流程，
而这可能会导致不同用户系统上的行为出现差异。

另一个更好的替代方案，至少在 Windows 上，是将所需版本的 Vulkan Runtime
安装程序随你的产品一起打包。
这样，安装过程就可以使用它来确保最终用户的系统
是最新的。
Runtime 安装程序会检测系统中已安装的版本，
并且仅在必要时安装更新的 runtime。

另一个替代方案是编写应用程序，使其能够回退到较早版本的
Vulkan，同时显示警告，表明在用户将其系统更新到特定 runtime/driver 之前，
某些功能将被禁用。

<a id="application-layer-usage"></a>
## 应用程序中的 Layer 使用

如果应用程序需要其系统上 Vulkan 驱动
当前尚未暴露的 Vulkan 功能，可以使用各种 layer 来扩展 API。
layer 不能添加 `Vulkan.h` 中未暴露的新的 Vulkan 核心 API 入口点。
但是，layer 可以提供某些扩展的实现，
从而引入超出未启用这些 layer 时可用范围之外的附加入口点。
这些附加扩展入口点可以通过 Vulkan
扩展接口进行查询。

layer 的一个常见用途是 API 验证，它可以在
应用程序开发期间启用，而在应用程序发布时省略。
这样就能方便地控制因启用应用程序 API 使用验证所带来的
额外开销，而这在以前的图形 API 中并不总是可行的。

要了解应用程序可用的 layer，请使用
`vkEnumerateInstanceLayerProperties`。
它会报告 loader 已发现的所有 layer。
loader 会在系统中的多个位置查找 layer。
更多信息请参见
[Layer discovery](LoaderLayerInterface.md#layer-discovery)
小节，位于
[LoaderLayerInterface.md document](LoaderLayerInterface.md) 文档中。

要启用特定 layer，只需在调用
`vkCreateInstance` 时，将要启用的 layer 名称传递给 `VkInstanceCreateInfo`
中的 `ppEnabledLayerNames` 字段即可。
完成后，所有使用所创建 `VkInstance`
及其任何子对象的 Vulkan 函数，都会启用这些 layer。

**注意：** 在若干情况下，layer 的顺序很重要，因为某些 layer
会彼此交互。
启用 layer 时请务必小心，这种情况可能会发生。
更多信息请参见 [Layer 的整体顺序](#overall-layer-ordering) 一节。

下面的代码段展示了如何启用
`VK_LAYER_KHRONOS_validation` layer。

```
char *instance_layers[] = {
    "VK_LAYER_KHRONOS_validation"
};
const VkApplicationInfo app = {
    .sType = VK_STRUCTURE_TYPE_APPLICATION_INFO,
    .pNext = NULL,
    .pApplicationName = "TEST_APP",
    .applicationVersion = 0,
    .pEngineName = "TEST_ENGINE",
    .engineVersion = 0,
    .apiVersion = VK_API_VERSION_1_0,
};
VkInstanceCreateInfo inst_info = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pNext = NULL,
    .pApplicationInfo = &app,
    .enabledLayerCount = 1,
    .ppEnabledLayerNames = (const char *const *)instance_layers,
    .enabledExtensionCount = 0,
    .ppEnabledExtensionNames = NULL,
};
err = vkCreateInstance(&inst_info, NULL, &demo->inst);
if (VK_ERROR_LAYER_NOT_PRESENT == err) {
  // Couldn't find the validation layer
}
```

在 `vkCreateInstance` 和 `vkCreateDevice` 时，loader 会构建包含
应用程序指定（启用）layer 的调用链。
`ppEnabledLayerNames` 数组中的顺序很重要；数组元素 0 是调用链中
最顶层（最接近应用程序）的 layer，而最后一个
数组元素最接近驱动。
有关 layer 排序的更多信息，
请参见 [Layer 的整体顺序](#overall-layer-ordering) 一节。

**注意：** *设备 Layer 现已弃用*
> `vkCreateDevice` 最初能够以与
`vkCreateInstance` 类似的方式选择 layer。
> 这引出了“instance layers”和“device layers”的概念。
> Khronos 决定弃用“device layer”功能，
> 仅保留“instance layers”。
> 因此，`vkCreateDevice` 将使用在
`vkCreateInstance` 中指定的 layer。
> 因此，以下内容已被弃用：
> * `VkDeviceCreateInfo` 字段：
>   * `ppEnabledLayerNames`
>   * `enabledLayerCount`
> * `vkEnumerateDeviceLayerProperties` 函数

<a id="meta-layers"></a>
### 元层

元层是包含其他 layer 的有序启用列表的 layer。
这样可以按指定顺序将多个 layer 组合在一起，
以便它们能够正确交互。
最初，这用于以正确顺序组合各个 Vulkan Validation
layer，从而避免冲突。
之所以有这个需求，是因为验证功能最初并不是一个单独的 Validation layer，
而是拆分成了多个组成 layer。
新的 `VK_LAYER_KHRONOS_validation` layer 将所有内容整合到了单个
layer 中，因此不再需要元层。
虽然验证场景如今已不再需要它们，但 VkConfig 确实会使用元层，
按照用户偏好对 layer 进行分组。
有关该功能的更多信息，可以参阅
[VkConfig documentation](https://github.com/LunarG/VulkanTools/blob/main/vkconfig/README.md)
以及后文的 [Override Layer](#override-layer) 一节。

关于元层的更多细节，请参见本文件夹中
[LoaderLayerInterface.md](LoaderLayerInterface.md) 文件的
[Meta-Layers](LoaderLayerInterface.md#meta-layers) 一节。

<a id="implicit-vs-explicit-layers"></a>
### 隐式层与显式层

![Different Types of Layers](./images/loader_layer_order.png)

显式层是由应用程序启用的 layer（例如前面提到的通过
vkCreateInstance 函数启用）。

隐式层则会因其自身存在而自动启用，除非
它们还要求额外的手动启用步骤；这与必须显式启用的显式层不同。
例如，某些应用程序运行环境（如 Steam 或汽车
信息娱乐系统）可能希望其启动的所有应用程序
始终启用某些 layer。
其他隐式层则可能适用于某个系统上启动的所有应用程序
（例如叠加显示 frames-per-second 的 layer）。

隐式层相较于显式层还有一个额外要求，
即它们必须能够通过环境变量被禁用。
这是因为它们对应用程序不可见，
并且可能引发问题。
一个值得牢记的好原则是同时定义启用和禁用
环境变量，以便用户可以确定性地启用该
功能。
在桌面平台（Windows、Linux 和 macOS）上，这些启用/禁用设置
定义在 layer 的 JSON 文件中。

系统安装的隐式层和显式层的发现方式将在后文
[Layer discovery](LoaderLayerInterface.md#layer-discovery)
小节中说明，位于
[LoaderLayerInterface.md](LoaderLayerInterface.md) 文档中。

根据底层操作系统的不同，隐式层和显式层
可能位于不同的位置。
下表给出了更多信息：

<table style="width:100%">
  <tr>
    <th>操作系统</th>
    <th>隐式层识别方式</th>
  </tr>
  <tr>
    <td>Windows</td>
    <td>隐式层位于与显式层不同的 Windows 注册表位置。</td>
  </tr>
  <tr>
    <td>Linux</td>
    <td>隐式层位于与显式层不同的目录位置。</td>
  </tr>
  <tr>
    <td>Android</td>
    <td>Android 上**不支持隐式层**。</td>
  </tr>
  <tr>
    <td>macOS</td>
    <td>隐式层位于与显式层不同的目录位置。</td>
  </tr>
</table>

<a id="override-layer"></a>
#### 覆盖层

“Override Layer” 是由
[VkConfig](https://github.com/LunarG/VulkanTools/blob/main/vkconfig/README.md)
工具创建的一种特殊隐式元层，并且在该工具运行时默认可用。
一旦 VkConfig 退出，override layer 就会被移除，
系统也应恢复到标准的 Vulkan 行为。
只要 override layer 出现在 layer 搜索路径中，loader 就会
将它与标准隐式层以及其待加载 layer 列表中包含的所有 layer
一起拉入 layer 调用栈。
这使最终用户或开发者能够通过 VkConfig
轻松强制启用任意数量的 layer 和设置。

有关 override layer 的更多讨论，请参见本文件夹中
[LoaderLayerInterface.md](LoaderLayerInterface.md) 文件的
[Override Meta-Layer](LoaderLayerInterface.md#override-meta-layer) 一节。

<a id="forcing-layer-source-folders"></a>
### 强制指定 Layer 源文件夹

开发者可能需要使用特殊的、预生产的 layer，
而不修改系统中已安装的 layer。

这可以通过以下两种方式之一实现：

  1. 使用 Vulkan SDK 随附的
[VkConfig](https://github.com/LunarG/VulkanTools/blob/main/vkconfig/README.md)
工具选择特定的 layer 路径。
  2. 使用
`VK_LAYER_PATH` 和/或 `VK_IMPLICIT_LAYER_PATH` 环境变量，指示 loader 在特定文件和/或文件夹中查找 layer。

`VK_LAYER_PATH` 和 `VK_IMPLICIT_LAYER_PATH` 环境变量都可以包含多个
路径，它们之间以操作系统特定的路径分隔符分隔。
在 Windows 上是分号（`;`），而在 Linux 和 macOS 上是冒号
（`:`）。

如果存在 `VK_LAYER_PATH`，则会扫描其中列出的文件和/或文件夹，
以查找显式 layer 清单文件。
隐式层发现过程不受该环境变量影响。

如果存在 `VK_IMPLICIT_LAYER_PATH`，则会扫描其中列出的文件和/或文件夹，
以查找隐式 layer 清单文件。
显式层发现过程不受该环境变量影响。

`VK_LAYER_PATH` 和 `VK_IMPLICIT_LAYER_PATH` 中列出的每个目录都应当是
包含 layer 清单文件的文件夹的完整路径名。

更多细节请参见
[调试环境变量表](LoaderInterfaceArchitecture.md#table-of-debug-environment-variables)
中的对应内容，位于 [LoaderInterfaceArchitecture.md document](LoaderInterfaceArchitecture.md)。

<a id="exception-for-elevated-privileges"></a>
#### 提升权限时的例外情况

出于安全原因，如果以提升权限运行，
`VK_LAYER_PATH` 和 `VK_IMPLICIT_LAYER_PATH` 会被忽略。
因此，这些环境变量只能用于
不使用提升权限的应用程序。

更多信息请参见顶层
[LoaderInterfaceArchitecture.md][LoaderInterfaceArchitecture.md] 文档中的
[Elevated Privilege Caveats](LoaderInterfaceArchitecture.md#elevated-privilege-caveats)。

<a id="forcing-layers-to-be-enabled-on-windows-linux-and-macos"></a>
### 在 Windows、Linux 和 macOS 上强制启用 Layer

开发者可能想启用其正在使用的应用程序
本身未启用的 layer。

这同样可以通过以下两种方式之一实现：

  1. 使用 Vulkan SDK 随附的
[VkConfig](https://github.com/LunarG/VulkanTools/blob/main/vkconfig/README.md)
工具选择特定 layer。
  2. 使用
`VK_INSTANCE_LAYERS` 环境变量，按名称指示 loader 查找附加 layer。

这两种方式都可用于启用那些未由应用程序在
`vkCreateInstance` 时指定（启用）的附加 layer。

`VK_INSTANCE_LAYERS` 环境变量是一个要启用的 layer 名称列表，
列表项之间以操作系统特定的路径分隔符分隔。
在 Windows 上是分号（`;`），而在 Linux 和 macOS 上是冒号
（`:`）。
这些名称的顺序是有意义的，其中列表中的第一个 layer 名称
是最顶层（最接近应用程序）的 layer，最后一个 layer 名称
是最底层（最接近驱动）的 layer。
更多信息请参见 [Layer 的整体顺序](#overall-layer-ordering) 一节。

在启用 layer 时，应用程序指定的 layer 与用户指定的 layer（通过环境
变量）会由 loader 聚合，并移除重复项。
通过环境变量指定的 layer 位于最顶层（最接近
应用程序），而应用程序指定的 layer 位于最底层。

下面是在 Linux 或 macOS 上使用这些环境变量激活验证
layer `VK_LAYER_KHRONOS_validation` 的示例：

```
> $ export VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation
```

更多细节请参见
[调试环境变量表](LoaderInterfaceArchitecture.md#table-of-debug-environment-variables)
中的对应内容，位于 [LoaderInterfaceArchitecture.md document](LoaderInterfaceArchitecture.md)。

<a id="overall-layer-ordering"></a>
### Layer 的整体顺序

根据上述规则，loader 对所有 layer 的整体排序
如下所示：

![Loader Layer Ordering](./images/loader_layer_order_calls.png)

显式层列表内部的顺序也可能很重要。
有些 layer 可能依赖于在 loader 调用它之前或之后
先实现某些行为。
例如：某个 overlay layer 可能希望使用 `VK_LAYER_KHRONOS_validation`
来验证 overlay layer 的行为是否正确。
这就要求将 overlay layer 放在更靠近应用程序的位置，
这样 validation layer 才能拦截 overlay layer 为实现其功能
所需发出的任何 Vulkan API 调用。

<a id="debugging-possible-layer-issues"></a>
### 调试可能的 Layer 问题

如果可能是某个 layer 导致了问题，可以尝试几种方法，
这些方法记录在
[Debugging Possible Layer Issues](LoaderDebugging.md#debugging-possible-layer-issues)
一节中，位于 docs
文件夹下的 [LoaderDebugging.mg](LoaderDebugging.md) 文档中。

<a id="application-usage-of-extensions"></a>
## 应用程序对扩展的使用

扩展是由 layer、loader 或
驱动提供的可选功能。
扩展可以修改 Vulkan API 的行为，并且需要由 Khronos
进行指定和注册。
这些扩展可以由 Vulkan 驱动、loader 或 layer 实现，
以暴露核心 API 中不可用的功能。
有关各种扩展的信息可以在 Vulkan 规范以及
`vulkan.h` 头文件中找到。

<a id="instance-and-device-extensions"></a>
### 实例扩展和设备扩展

正如主
[LoaderInterfaceArchitecture.md](LoaderInterfaceArchitecture.md) 文档中的
[Instance Versus Device](LoaderInterfaceArchitecture.md#instance-versus-device)
一节所暗示的那样，
扩展分为两种类型：
 * 实例扩展
 * 设备扩展

实例扩展会修改实例级对象（例如 `VkInstance` 和 `VkPhysicalDevice`）上的
现有行为或实现新行为。
设备扩展则对设备级对象（例如 `VkDevice`、`VkQueue` 和 `VkCommandBuffer`）
以及这些对象的任何子对象执行相同的操作。

了解扩展的类型 **非常** 重要，
因为实例扩展必须通过 `vkCreateInstance` 启用，而设备
扩展必须通过 `vkCreateDevice` 启用。

调用 `vkEnumerateInstanceExtensionProperties` 和
`vkEnumerateDeviceExtensionProperties` 时，loader 会先发现并聚合其各自类型的所有
扩展，这些扩展来自 layer（显式层和隐式层）、
驱动以及 loader，然后再向应用程序报告。

查看 `vulkan.h` 可以发现，这两个函数非常相似，
例如，`vkEnumerateInstanceExtensionProperties` 的原型如下：

```
VkResult
   vkEnumerateInstanceExtensionProperties(
      const char *pLayerName,
      uint32_t *pPropertyCount,
      VkExtensionProperties *pProperties);
```

而 `vkEnumerateDeviceExtensionProperties` 的原型如下：

```
VkResult
   vkEnumerateDeviceExtensionProperties(
      VkPhysicalDevice physicalDevice,
      const char *pLayerName,
      uint32_t *pPropertyCount,
      VkExtensionProperties *pProperties);
```

这些函数中的 “pLayerName” 参数用于选择某个单独的
layer，或 Vulkan 平台实现。
如果 “pLayerName” 为 NULL，则会枚举 Vulkan 实现组件中的扩展
（包括 loader、隐式层和驱动）。
如果 “pLayerName” 等于某个已发现 layer 模块的名称，则只会枚举该
layer 的扩展（该 layer 可以是隐式层，也可以是显式层）。

**注意：** 虽然设备层已被弃用，但通过实例启用的 layer
仍然存在于设备调用链中。

重复的扩展（例如某个隐式层和驱动可能都报告支持同一个
扩展）会由 loader 去重。
对于重复项，会报告驱动版本，而 layer 版本会被剔除。

此外，在可以使用与扩展相关的函数之前，
扩展 **必须先被启用**（在 `vkCreateInstance` 或 `vkCreateDevice` 中）。
如果使用 `vkGetInstanceProcAddr` 或
`vkGetDeviceProcAddr` 查询了某个扩展函数，
但该扩展并未启用，则可能导致未定义行为。
Validation layer 会捕获这种无效的 API 用法。

<a id="wsi-extensions"></a>
### WSI 扩展

Khronos 批准的 WSI 扩展可用，并为各种执行环境提供窗口系统
集成支持。
需要理解的是，有些 WSI 扩展对所有
目标都有效，但另一些则仅适用于特定执行环境（以及
loader）。
当前这个 Khronos loader（目前面向 Windows、Linux、macOS、Stadia 和
Fuchsia）只会启用并直接导出那些适合当前环境的
WSI 扩展。
在大多数情况下，这一选择是通过 loader 中的编译期
预处理器标志完成的。
当前 Khronos loader 的所有版本至少都会暴露以下 WSI
扩展支持：
- VK_KHR_surface
- VK_KHR_swapchain
- VK_KHR_display

此外，loader 针对以下各个 OS 目标还支持目标特定扩展：

| 窗口系统 | 可用扩展 |
| -------- | -------- |
| Windows | VK_KHR_win32_surface |
| Linux (Wayland) | VK_KHR_wayland_surface |
| Linux (X11) | VK_KHR_xcb_surface and VK_KHR_xlib_surface |
| macOS (MoltenVK) | VK_MVK_macos_surface |
| QNX (Screen) | VK_QNX_screen_surface |

需要理解的是，尽管 loader 可能支持这些扩展的各种
入口点，但要真正使用它们还需要一次握手过程：
* 至少有一个物理设备必须支持该扩展
* 应用程序在创建设备时必须使用这样的物理设备
* 应用程序在创建实例或逻辑设备时，
  必须请求启用该扩展（这取决于给定扩展
  适用于实例还是设备）

只有这样，WSI 扩展才能在 Vulkan 程序中被正确使用。

<a id="unknown-extensions"></a>
### 未知扩展

由于 Vulkan 很容易扩展，因此将会出现一些
loader 完全不了解的扩展。
如果该扩展是设备扩展，loader 会将这个未知
入口点沿设备调用链向下传递，最终到达相应的
驱动入口点。
如果该扩展是一个实例扩展，并且其第一个参数
是物理设备参数，也会发生同样的情况。
但是，对于所有其他实例扩展，loader 将无法加载它。

*但为什么 loader 不支持未知的实例扩展？*
<br/>
让我们再来看一次实例调用链：

![Instance call chain](./images/loader_instance_chain.png)

注意，对于普通的实例函数调用，loader 必须处理
将函数调用传递给可用驱动的过程。
如果 loader 完全不知道该实例调用的参数或返回值，
它就无法正确地将信息传递给驱动。
未来也许会探索实现这一点的方法。
但目前，loader 不支持那些
其暴露入口点不以物理设备作为第一个参数的实例扩展。

由于设备调用链通常不会经过 loader
*terminator*，因此这对设备扩展不是问题。
此外，由于一个物理设备只关联一个驱动，loader
可以使用一个指向单个驱动的通用 *terminator*。
这是因为这两类扩展都会直接终止于其所关联的
驱动。

*这是个大问题吗？*
<br/>
不是！
大多数扩展功能只会影响物理设备或逻辑设备，
而不会影响实例。
因此，绝大多数扩展都应当能够通过 loader 的直接支持
得到支持。

<a id="filtering-out-unknown-instance-extension-names"></a>
### 过滤掉未知的实例扩展名称

在某些情况下，驱动可能支持一些 loader
并不支持的实例扩展。
基于上述原因，当应用程序调用
`vkEnumerateInstanceExtensionProperties` 时，loader 会过滤掉这些未知
实例扩展的名称。
此外，如果应用程序仍然尝试在
`vkCreateInstance` 时使用这些扩展，这种行为还会导致 loader
发出错误。
这样做的目的是保护应用程序，防止它们无意中使用
可能导致崩溃的功能。

另一方面，如果必须强制启用该扩展，则可以通过将
`VK_LOADER_DISABLE_INST_EXT_FILTER` 环境变量定义为非零数值
来禁用此过滤。
这样会有效关闭 loader 对实例扩展
名称的过滤。

<a id="physical-device-ordering"></a>
## 物理设备排序

在 1.3.204 之前的 loader 中，Linux 上返回的物理设备顺序可能并不一致。
为解决此问题，Vulkan loader 现在会在从驱动接收到设备之后（并在将信息返回给任何已启用的 layer 之前），按如下方式对设备进行排序：
 * 按设备类型排序（离散、集成、虚拟，以及其他所有类型）
 * 在各类型内部，再根据 PCI 信息（Domain、Bus、Device 和 Function）进行排序。

这样一来，在同一系统上多次运行时，物理设备的顺序将保持一致，除非底层实际硬件发生变化。

新定义了一个环境变量，使用户能够强制指定特定设备：`VK_LOADER_DEVICE_SELECT`。
该环境变量应设置为目标设备的 Vendor Id 和 Device Id 的十六进制值（由 `vkGetPhysicalDeviceProperties` 在 `VkPhysicalDeviceProperties` 结构中返回）。
其格式如下所示：

```
set VK_LOADER_DEVICE_SELECT=0x10de:0x1f91
```

这会强制选中 vendor ID 为 `0x10de`、device ID 为 `0x1f91` 的设备。
如果未找到该设备，则会直接忽略此设置。

通过将环境变量 `VK_LOADER_DISABLE_SELECT` 设置为非零值，可以禁用 loader 中执行的所有设备选择工作。
此设置主要用于调试，以便缩小与 loader 设备选择机制相关问题的范围，但其他场景也可使用。

[返回顶层 LoaderInterfaceArchitecture.md 文件。](LoaderInterfaceArchitecture.md)
