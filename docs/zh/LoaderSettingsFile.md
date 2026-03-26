<!-- markdownlint-disable MD041 -->
[![Khronos Vulkan][1]][2]

[1]: https://vulkan.lunarg.com/img/Vulkan_100px_Dec16.png "https://www.khronos.org/vulkan/"
[2]: https://www.khronos.org/vulkan/

# Loader 设置文件 <!-- omit from toc -->

[![Creative Commons][3]][4]

<!-- Copyright &copy; 2025 LunarG, Inc. -->

[3]: https://i.creativecommons.org/l/by-nd/4.0/88x31.png "Creative Commons License"
[4]: https://creativecommons.org/licenses/by-nd/4.0/


## 目录 <!-- omit from toc -->

- [设置文件的用途](#设置文件的用途)
- [设置文件发现机制](#设置文件发现机制)
  - [Windows](#windows)
  - [Linux/MacOS/BSD/QNX/Fuchsia/GNU](#linuxmacosbsdqnxfuchsiagnu)
  - [其他平台](#其他平台)
  - [提升权限时的例外情况](#提升权限时的例外情况)
- [每应用程序设置文件](#每应用程序设置文件)
- [文件格式](#文件格式)
- [设置文件示例](#设置文件示例)
  - [字段](#字段)
- [行为](#行为)


## 设置文件的用途

Loader 设置文件的目的是让开发者能够对 Vulkan-Loader 的行为进行高度控制。

它可增强对以下内容的控制能力：加载哪些 Layer、调用链中 Layer 的顺序、日志记录，以及哪些驱动可用。

Loader 设置文件旨在供 Vulkan API 的“开发者控制面板”使用，例如 Vulkan Configurator，以替代设置调试环境变量。

## 设置文件发现机制

Loader 设置文件通过在特定文件系统路径中搜索，或通过平台特定机制（如 Windows 注册表）来定位。

### Windows

Vulkan Loader 首先会在注册表项 `HKEY_CURRENT_USER\SOFTWARE\Khronos\Vulkan\LoaderSettings` 中查找一个 DWORD 值，其名称必须是一个指向名为 `vk_loader_settings.json` 文件的有效路径。

如果没有匹配的值，或者该文件不存在，则 Vulkan Loader 会对注册表项 `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\LoaderSettings` 执行与上述相同的行为。

### Linux/MacOS/BSD/QNX/Fuchsia/GNU

Loader 设置文件通过在以下位置搜索名为 `vk_loader_settings.json` 的文件来定位：

`$HOME/.local/share/vulkan/loader_settings.d/`
`$XDG_DATA_HOME/vulkan/loader_settings.d/`
`/etc/vulkan/loader_settings.d/`

其中，`$HOME` 和 `%XDG_DATA_HOME` 指代同名环境变量中的值。

如果某个环境变量不存在，则忽略对应路径。

### 其他平台

上面未列出的平台目前不支持 Loader 设置文件，因为它们没有合适的搜索机制。

### 提升权限时的例外情况

由于 Loader 设置文件包含指向 Layer 清单文件和 ICD 清单文件的路径，而这些清单文件又包含指向各种可执行二进制文件的路径，因此在应用程序以提升后的权限运行时，必须限制 Loader 设置文件的使用。

实现方式是：不使用在非特权位置中找到的任何 Loader 设置文件。

在 Windows 上，以提升后的权限运行时，将忽略 `HKEY_CURRENT_USER\SOFTWARE\Khronos\Vulkan\LoaderSettings`。

在 Linux/MacOS/BSD/QNX/Fuchsia/GNU 上，以提升后的权限运行时，将使用安全方式查询 `$HOME` 和 `$XDG_DATA_HOME`，以防止恶意注入不安全的搜索目录。

## 每应用程序设置文件

## 文件格式

Loader 设置文件是一个 JSON 文件，其


## 设置文件示例


```json
{
   "file_format_version" : "1.0.1",
   "settings": {

   }
}
```

### 字段

<table style="width:100%">
  <tr>
    <th>JSON 节点</th>
    <th>说明与备注</th>
    <th>限制</th>
    <th>父节点</th>
    <th>内省查询</th>
  </tr>




## 行为
