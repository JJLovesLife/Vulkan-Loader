<!-- markdownlint-disable MD041 -->
[![Khronos Vulkan][1]][2]

[1]: https://vulkan.lunarg.com/img/Vulkan_100px_Dec16.png "https://www.khronos.org/vulkan/"
[2]: https://www.khronos.org/vulkan/

# 调试 Vulkan 桌面加载器 <!-- omit from toc -->
[![Creative Commons][3]][4]

<!-- Copyright &copy; 2015-2023 LunarG, Inc. -->

[3]: https://i.creativecommons.org/l/by-nd/4.0/88x31.png "Creative Commons License"
[4]: https://creativecommons.org/licenses/by-nd/4.0/
## 目录 <!-- omit from toc -->

- [调试问题](#调试问题)
- [加载器日志](#加载器日志)
- [调试可能的层问题](#调试可能的层问题)
  - [启用层日志](#启用层日志)
  - [禁用层](#禁用层)
  - [有选择地重新启用层](#有选择地重新启用层)
- [允许 `VK_LOADER_LAYERS_DISABLE` 忽略特定层](#允许-vk_loader_layers_disable-忽略特定层)
- [调试可能的驱动问题](#调试可能的驱动问题)
  - [启用驱动日志](#启用驱动日志)
  - [有选择地启用特定驱动](#有选择地启用特定驱动)

## 调试问题

如果你的应用程序发生崩溃或行为异常，加载器提供了几种机制来帮助你调试这些问题。

**注意**：此功能全部仅适用于桌面版 Vulkan 加载器，不适用于 Android 加载器。

## 加载器日志

Vulkan 桌面加载器增加了日志功能，可通过设置 `VK_LOADER_DEBUG` 环境变量启用。
结果会输出到标准输出，同时也会传递给任何已存在的 `VK_EXT_debug_utils` messenger。
该变量可以设置为由逗号分隔的调试级别选项列表，包括：

  * error&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;报告加载器遇到的任何错误
  * warn&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;报告加载器遇到的任何警告
  * info&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;报告加载器生成的信息级消息
  * debug&nbsp;&nbsp;&nbsp;&nbsp;报告加载器生成的调试级消息
  * layer&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;报告加载器生成的所有层相关消息
  * driver&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;报告加载器生成的所有驱动相关消息
  * all&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;报告加载器生成的所有消息（包含以上全部）

如果你不确定问题来自哪里，至少应将其设置为输出到 `info` 级别为止的所有消息：

```
set VK_LOADER_DEBUG=error,warn,info
```

然后，你可以在输出列表中查找任何错误或警告，看看它们是否能提示你为何会遇到这些问题。

有关如何启用加载器日志的更多信息，请参阅下方的
[启用加载器调试层输出](LoaderApplicationInterface.md#enable-loader-debug-layer-output)
以及
[调试环境变量表](LoaderInterfaceArchitecture.md#table-of-debug-environment-variables)。

## 调试可能的层问题

### 启用层日志

如果你怀疑是层的问题，可将加载器日志设置为除了警告和错误之外，还专门输出层相关消息：

```
set VK_LOADER_DEBUG=error,warn,layer
```

大多数重要的层消息会在启用 error 或 warning 级别时输出，但这样还会提供更多层相关信息，例如：
  * 找到了哪些层
  * 在哪里找到它们
  * 如果它们是隐式层，可以使用哪些环境变量禁用它们
  * 如果某个层存在不兼容情况，可能包括：
    * 找不到层库文件（`.so`/`.dll`）
    * 对当前执行的应用程序而言，层库的位宽不正确（即 32 位与 64 位不匹配）
    * 该层本身不支持应用程序所需的 Vulkan 版本
  * 是否有任何环境变量正在禁用某些层

例如，加载器查找隐式层时的输出可能如下所示：

```
[Vulkan Loader] LAYER: Searching for implicit layer manifest files
[Vulkan Loader] LAYER:  In following locations:
[Vulkan Loader] LAYER:   /home/${USER}/.config/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /etc/xdg/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /usr/local/etc/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /etc/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /home/${USER}/.local/share/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /home/${USER}/.local/share/flatpak/exports/share/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /var/lib/flatpak/exports/share/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /usr/local/share/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:   /usr/share/vulkan/implicit_layer.d
[Vulkan Loader] LAYER:  Found the following files:
[Vulkan Loader] LAYER:   /home/${USER}/.local/share/vulkan/implicit_layer.d/renderdoc_capture.json
[Vulkan Loader] LAYER:   /home/${USER}/.local/share/vulkan/implicit_layer.d/steamfossilize_i386.json
[Vulkan Loader] LAYER:   /home/${USER}/.local/share/vulkan/implicit_layer.d/steamfossilize_x86_64.json
[Vulkan Loader] LAYER:   /home/${USER}/.local/share/vulkan/implicit_layer.d/steamoverlay_i386.json
[Vulkan Loader] LAYER:   /home/${USER}/.local/share/vulkan/implicit_layer.d/steamoverlay_x86_64.json
[Vulkan Loader] LAYER:   /usr/share/vulkan/implicit_layer.d/nvidia_layers.json
[Vulkan Loader] LAYER:   /usr/share/vulkan/implicit_layer.d/VkLayer_MESA_device_select.json
```

随后，层库的加载会以类似如下方式报告：

```
[Vulkan Loader] DEBUG | LAYER : Loading layer library libVkLayer_khronos_validation.so
[Vulkan Loader] INFO | LAYER : Insert instance layer VK_LAYER_KHRONOS_validation (libVkLayer_khronos_validation.so)
[Vulkan Loader] DEBUG | LAYER : Loading layer library libVkLayer_MESA_device_select.so
[Vulkan Loader] INFO | LAYER : Insert instance layer VK_LAYER_MESA_device_select (libVkLayer_MESA_device_select.so)
```

最后，在创建 Vulkan 实例时，你可以看到从功能角度展示的完整实例调用链，其输出如下：

```
[Vulkan Loader] LAYER: vkCreateInstance layer callstack setup to:
[Vulkan Loader] LAYER:  <Application>
[Vulkan Loader] LAYER:    ||
[Vulkan Loader] LAYER:  <Loader>
[Vulkan Loader] LAYER:    ||
[Vulkan Loader] LAYER:  VK_LAYER_MESA_device_select
[Vulkan Loader] LAYER:      Type: Implicit
[Vulkan Loader] LAYER:      Enabled By: Implicit Layer
[Vulkan Loader] LAYER:         Disable Env Var:  NODEVICE_SELECT
[Vulkan Loader] LAYER:      Manifest: /usr/share/vulkan/implicit_layer.d/VkLayer_MESA_device_select.json
[Vulkan Loader] LAYER:      Library:  libVkLayer_MESA_device_select.so
[Vulkan Loader] LAYER:    ||
[Vulkan Loader] LAYER:  VK_LAYER_KHRONOS_validation
[Vulkan Loader] LAYER:      Type: Explicit
[Vulkan Loader] LAYER:      Enabled By: By the Application
[Vulkan Loader] LAYER:      Manifest: /usr/share/vulkan/explicit_layer.d/VkLayer_khronos_validation.json
[Vulkan Loader] LAYER:      Library:  libVkLayer_khronos_validation.so
[Vulkan Loader] LAYER:    ||
[Vulkan Loader] LAYER:  <Drivers>
```

在这个场景中，使用了两个层（就是前面加载的那两个）：
* `VK_LAYER_MESA_device_select`
* `VK_LAYER_KHRONOS_validation`

这些信息现在表明，`VK_LAYER_MESA_device_select` 会先加载，随后是 `VK_LAYER_KHRONOS_validation`，然后继续进入任何可用驱动。
它还表明 `VK_LAYER_MESA_device_select` 是隐式层，这意味着它不是由应用程序直接启用的。
另一方面，`VK_LAYER_KHRONOS_validation` 被显示为显式层，这说明它很可能是由应用程序启用的。

### 禁用层

**注意：** 此功能仅适用于使用 Vulkan 头文件 1.3.234 及更高版本构建的加载器。

有时，隐式层可能会导致应用程序出现问题。
因此，下一步就是尝试禁用一个或多个列出的隐式层。
你可以使用过滤环境变量（`VK_LOADER_LAYERS_ENABLE` 和 `VK_LOADER_LAYERS_DISABLE`）来有选择地启用或禁用不同的层。
如果你不确定该怎么做，可以先手动禁用所有隐式层，将 `VK_LOADER_LAYERS_DISABLE` 设置为 `~implicit~`。

```
  set VK_LOADER_LAYERS_DISABLE=~implicit~
```

这会禁用所有隐式层，并且在启用层日志后，加载器会按如下方式在日志输出中报告被禁用的层：

```
[Vulkan Loader] WARNING | LAYER:  Implicit layer "VK_LAYER_MESA_device_select" forced disabled because name matches filter of env var 'VK_LOADER_LAYERS_DISABLE'.
[Vulkan Loader] WARNING | LAYER:  Implicit layer "VK_LAYER_AMD_switchable_graphics_64" forced disabled because name matches filter of env var 'VK_LOADER_LAYERS_DISABLE'.
[Vulkan Loader] WARNING | LAYER:  Implicit layer "VK_LAYER_Twitch_Overlay" forced disabled because name matches filter of env var 'VK_LOADER_LAYERS_DISABLE'.
```

### 有选择地重新启用层

**注意：** 此功能仅适用于使用 Vulkan 头文件 1.3.234 及更高版本构建的加载器。

在尝试诊断由层引起的问题时，一个有用的方法是先禁用所有层，再逐个重新启用它们。
如果问题再次出现，就可以立刻明确是哪一个层导致了该问题。

例如，根据上面给出的已禁用层列表，我们来有选择地重新启用其中一个：

```
set VK_LOADER_LAYERS_DISABLE=~implicit~
set VK_LOADER_LAYERS_ENABLE=*AMD*
```

这会让 `VK_LAYER_MESA_device_select` 和 `VK_LAYER_Twitch_Overlay` 保持禁用，同时启用 `VK_LAYER_AMD_switchable_graphics_64`。
如果一切继续正常工作，那么现有证据似乎表明问题很可能与 AMD 层无关。
接下来就可以再启用另一个层并再次尝试：

```
set VK_LOADER_LAYERS_DISABLE=~implicit~
set VK_LOADER_LAYERS_ENABLE=*AMD*,*twitch*
```

以此类推。

有关如何使用这些过滤环境变量的更多信息，请参阅 [LoaderLayerInterface](LoaderLayerInterface.md) 文档中的 [层过滤](LoaderLayerInterface.md#layer-filtering) 一节。

## 允许 `VK_LOADER_LAYERS_DISABLE` 忽略特定层

**注意：** `VK_LOADER_LAYERS_DISABLE` 仅适用于使用 Vulkan 头文件 1.3.262 及更高版本构建的加载器。

在使用 `VK_LOADER_LAYERS_DISABLE` 禁用隐式层时，可以使用 `VK_LOADER_LAYERS_ENABLE` 允许特定层被启用。
但是，这样会产生一个效果：*强制* 启用这些层，而这并不总是我们想要的。
隐式层本身能够仅在设置了某个层指定环境变量时才被启用，从而实现依赖上下文的启用逻辑。
`VK_LOADER_LAYERS_ENABLE` 会忽略这种上下文。

因此，需要另一个环境变量：`VK_LOADER_LAYERS_ALLOW`

`VK_LOADER_LAYERS_ALLOW` 的行为与 `VK_LOADER_LAYERS_ENABLE` 类似，只不过它不会强制启用层。
理解这个环境变量的一种方式是：所有匹配 `VK_LOADER_LAYERS_ALLOW` 的层，都会被排除在 `VK_LOADER_LAYERS_DISABLE` 的强制禁用范围之外。
这样一来，依赖上下文的隐式层就可以根据相关上下文决定是否启用，而不是被强制启用。

示例：禁用所有隐式层，但保留名称中包含 steam 或 mesa 的层不受影响。
```
set VK_LOADER_LAYERS_DISABLE=~implicit~
set VK_LOADER_LAYERS_ALLOW=*steam*,*Mesa*
```

## 调试可能的驱动问题

### 启用驱动日志

**注意：** 此功能仅适用于使用 Vulkan 头文件 1.3.234 及更高版本构建的加载器。

如果你怀疑是驱动问题，可将加载器日志设置为专门输出驱动相关消息：

```
set VK_LOADER_DEBUG=error,warn,driver
```

大多数重要的驱动消息会在启用 error 或 warning 级别时输出，但这样还会提供更多驱动相关信息，例如：
  * 找到了哪些驱动
  * 在哪里找到它们
  * 某个驱动是否存在不兼容情况
  * 是否有任何环境变量正在禁用某些驱动

例如，在 Linux 系统上，加载器查找驱动时的输出可能如下所示（注意：为了便于阅读，输出中额外的空格已被移除）：

```
[Vulkan Loader] DRIVER: Searching for driver manifest files
[Vulkan Loader] DRIVER:    In following folders:
[Vulkan Loader] DRIVER:       /home/$(USER)/.config/vulkan/icd.d
[Vulkan Loader] DRIVER:       /etc/xdg/vulkan/icd.d
[Vulkan Loader] DRIVER:       /etc/vulkan/icd.d
[Vulkan Loader] DRIVER:       /home/$(USER)/.local/share/vulkan/icd.d
[Vulkan Loader] DRIVER:       /home/$(USER)/.local/share/flatpak/exports/share/vulkan/icd.d
[Vulkan Loader] DRIVER:       /var/lib/flatpak/exports/share/vulkan/icd.d
[Vulkan Loader] DRIVER:       /usr/local/share/vulkan/icd.d
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d
[Vulkan Loader] DRIVER:    Found the following files:
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d/intel_icd.x86_64.json
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d/lvp_icd.x86_64.json
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d/radeon_icd.x86_64.json
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d/lvp_icd.i686.json
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d/radeon_icd.i686.json
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d/intel_icd.i686.json
[Vulkan Loader] DRIVER:       /usr/share/vulkan/icd.d/nvidia_icd.json
[Vulkan Loader] DRIVER: Found ICD manifest file /usr/share/vulkan/icd.d/intel_icd.x86_64.json, version "1.0.0"
[Vulkan Loader] DRIVER: Found ICD manifest file /usr/share/vulkan/icd.d/lvp_icd.x86_64.json, version "1.0.0"
[Vulkan Loader] DRIVER: Found ICD manifest file /usr/share/vulkan/icd.d/radeon_icd.x86_64.json, version "1.0.0"
[Vulkan Loader] DRIVER: Found ICD manifest file /usr/share/vulkan/icd.d/lvp_icd.i686.json, version "1.0.0"
[Vulkan Loader] DRIVER: Requested driver /usr/lib/libvulkan_lvp.so was wrong bit-type. Ignoring this JSON
[Vulkan Loader] DRIVER: Found ICD manifest file /usr/share/vulkan/icd.d/radeon_icd.i686.json, version "1.0.0"
[Vulkan Loader] DRIVER: Requested driver /usr/lib/libvulkan_radeon.so was wrong bit-type. Ignoring this JSON
[Vulkan Loader] DRIVER: Found ICD manifest file /usr/share/vulkan/icd.d/intel_icd.i686.json, version "1.0.0"
[Vulkan Loader] DRIVER: Requested driver /usr/lib/libvulkan_intel.so was wrong bit-type. Ignoring this JSON
[Vulkan Loader] DRIVER: Found ICD manifest file /usr/share/vulkan/icd.d/nvidia_icd.json, version "1.0.0"
```

随后，当应用程序选择要使用的设备时，你会看到 Vulkan 设备调用链按如下方式输出（注意：为了便于阅读，输出中额外的空格已被移除）：

```
[Vulkan Loader] DRIVER: vkCreateDevice layer callstack setup to:
[Vulkan Loader] DRIVER:    <Application>
[Vulkan Loader] DRIVER:      ||
[Vulkan Loader] DRIVER:    <Loader>
[Vulkan Loader] DRIVER:      ||
[Vulkan Loader] DRIVER:    <Device>
[Vulkan Loader] DRIVER:        Using "Intel(R) UHD Graphics 630 (CFL GT2)" with driver: "/usr/lib64/libvulkan_intel.so"
```


### 有选择地启用特定驱动

**注意：** 此功能仅适用于使用 Vulkan 头文件 1.3.234 及更高版本构建的加载器。

现在，你可以使用过滤环境变量（`VK_LOADER_DRIVERS_SELECT` 和 `VK_LOADER_DRIVERS_DISABLE`）来控制加载器会尝试加载哪些驱动。
对于驱动而言，传入上述环境变量的字符串 glob 会与驱动 JSON 文件名进行比较，因为直到 Vulkan 初始化流程的更后期阶段，加载器才知道驱动名称。

例如，要禁用除 Nvidia 之外的所有驱动，可以这样做：

```
set VK_LOADER_DRIVERS_DISABLE=*
set VK_LOADER_DRIVERS_SELECT=*nvidia*
```

使用这些环境变量时，加载器会输出如下消息：

```
[Vulkan Loader] WARNING | DRIVER: Driver "intel_icd.x86_64.json" ignored because not selected by env var 'VK_LOADER_DRIVERS_SELECT'
[Vulkan Loader] WARNING | DRIVER: Driver "radeon_icd.x86_64.json" ignored because it was disabled by env var 'VK_LOADER_DRIVERS_DISABLE'
```

有关如何使用这些过滤环境变量的更多信息，请参阅 [LoaderDriverInterface](LoaderDriverInterface.md) 文档中的 [驱动过滤](LoaderDriverInterface.md#driver-filtering) 一节。
