---
title: Windows Terminal
date: 2020-09-05 09:41:34
tags:
  - windows
  - terminal
---

[`Windows Terminal`](https://docs.microsoft.com/en-us/windows/terminal/) 是继 `WSL` 之后出品的又一个开发者友好的现代化应用程序。在支持原有的 `CMD`、`Powershell` 之外，还支持 `WSL` 子系统。其主要特性包括: 多 `tab`，多 `panes`，支持显示 `Unicode` 与 `UTF-8` 字符，使用 `GPU` 加速的文本渲染引擎，内置 `SSH` 客户端，更允许用户自定义主题及文本样式，颜色，背景，快捷键等。

更多内容请阅读[官方文档](https://docs.microsoft.com/en-us/windows/terminal/)。

## 我的配置文件

``` json
{
    "$schema": "https://aka.ms/terminal-profiles-schema",
    "defaultProfile": "{a5a97cb8-8961-5535-816d-772efe0c6a3f}",
    "copyOnSelect": false,
    "copyFormatting": false,
    "theme": "dark",
    "tabWidthMode": "titleLength",
    "profiles": {
        "defaults": {
            "colorScheme": "Dracula"
        },
        "list": [
            {
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "name": "Windows PowerShell",
                "commandline": "powershell.exe",
                "hidden": false
            },
            {
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "name": "Command Prompt",
                "commandline": "cmd.exe",
                "hidden": true
            },
            {
                "guid": "{a5a97cb8-8961-5535-816d-772efe0c6a3f}",
                "hidden": false,
                "name": "Arch",
                "tabTitle": "Arch",
                "suppressApplicationTitle": true,
                "antialiasingMode": "cleartype", // "grayscale"、"cleartype"、"aliased"
                "icon": "ms-appdata:///roaming/archlinux-512.webp",
                "source": "Windows.Terminal.Wsl",
                "acrylicOpacity": 0.8,
                "useAcrylic": true
            }
        ]
    },
    "schemes": [
        {
            "name": "Dracula",
            "cursorColor": "#F8F8F2",
            "selectionBackground": "#44475A",
            "background": "#282A36",
            "foreground": "#F8F8F2",
            "black": "#21222C",
            "blue": "#BD93F9",
            "cyan": "#8BE9FD",
            "green": "#50FA7B",
            "purple": "#FF79C6",
            "red": "#FF5555",
            "white": "#F8F8F2",
            "yellow": "#F1FA8C",
            "brightBlack": "#6272A4",
            "brightBlue": "#D6ACFF",
            "brightCyan": "#A4FFFF",
            "brightGreen": "#69FF94",
            "brightPurple": "#FF92DF",
            "brightRed": "#FF6E6E",
            "brightWhite": "#FFFFFF",
            "brightYellow": "#FFFFA5"
        }
    ],
    "keybindings": [
        {
            "command": {
                "action": "copy",
                "singleLine": false
            },
            "keys": "ctrl+c"
        },
        {
            "command": "paste",
            "keys": "ctrl+v"
        },
        {
            "command": "find",
            "keys": "ctrl+shift+f"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "splitMode": "duplicate"
            },
            "keys": "alt+shift+d"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "horizontal",
                "splitMode": "duplicate"
            },
            "keys": "alt+shift+-"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "vertical",
                "splitMode": "duplicate"
            },
            "keys": "alt+shift+|"
        }
    ]
}
```

## 全局设置

* [**defaultProfile**](https://docs.microsoft.com/en-us/windows/terminal/customize-settings/global-settings#default-profile): 默认使用的终端配置，可以是 `profiles.list` 中的某一个。
* [**theme**](https://docs.microsoft.com/en-us/windows/terminal/customize-settings/global-settings#darklight-theme): 窗体配色，可以是 `system`, `dark`, `light`
* [**tabWidthMode**](https://docs.microsoft.com/en-us/windows/terminal/customize-settings/global-settings#tab-width-mode): `tab` 显示长度，可以是 `equal`, `titleLength`, `compact`

## 配置终端

`profiles` 用于配置具体的终端，其包含如下两部分:

``` json
{
    "profiles": {
        "defaults": {
            // SETTINGS TO APPLY TO ALL PROFILES
        },
        "list": [
            // PROFILE OBJECTS
        ]
    }
}
```

### 默认终端配置

* **profiles.defaults**: 中的配置对所有的终端有效。
* **profiles.list**: 包含所有可用的终端，配置只对当前终端。

* **guid**: 配置文件可将 `GUID` 用作唯一标识符。 若要将某个配置文件设置为默认配置文件，则需要 `defaultProfile` 全局设置的 `GUID`，为必填项
* **colorScheme**: 终端使用的配色方案，需要在 `schemes` 中目标配色方案
* **hidden**: 在下拉列表中是否隐藏，默认为 `false`
* **name**: "在下拉列表中显示的名字
* **tabTitle**: 在 `tab` 上显示的名字，会覆盖 **name**，需要与 **suppressApplicationTitle** 一起使用才会生效。若要了解如何使 shell 设置标题，请访问[tab title tutorial](https://docs.microsoft.com/en-us/windows/terminal/tutorials/tab-title)
* **suppressApplicationTitle**: 设置为 **true** 时，**tabTitle** 会替代 `tab` 的默认标题，并将禁止应用程序的任何标题更改消息。 如果未设置 **tabTitle**，将改为使用 **name**
* **antialiasingMode**: 抗锯齿模式，可以为 **grayscale**、**cleartype**、**aliased**
* **icon**: 显示图标，"ms-appdata:///roaming/archlinux-512.webp"
* **acrylicOpacity**: 窗口透明度，**[0, 1]** 之内的浮点数，需要配合 **useAcrylic** 一起使用
* **useAcrylic**: 是否是否透明效果
* **startingDirectory**: 起始目录位置，比如 `\\\\wsl$\\Ubuntu-20.04\\home\\alice`

#### ms-appdata 在什么位置

`Windows Terminal` 是一个 `UWP` 应用(如果是在应用商店下载的话)，会有属于自己的 **appdata** 目录，位于:

``` powershell
%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\RoamingState
```

在文件管理器中打开这个目录放入数据，`Windows Terminal` 即可通过 `ms-appdata:\\\` 的方式获取到。

## 配色方案

可以在 **themes** 定义一系列的配色方案，要求每个配色方案要有一个唯一的 **name**。除 **name** 以外，每个设置都接受十六进制格式 (**#rgb** 或 **#rrggbb**) 的字符串形式的颜色。 **cursorColor** 和 **selectionBackground** 设置是可选的。

如果要在一个命令行配置文件中设置配色方案，请添加 **colorScheme** 属性，并将配色方案的 **name** 作为值。

## 自定义快捷键

内容太多，请查看[官方文档](https://docs.microsoft.com/en-us/windows/terminal/customize-settings/key-bindings)。

## 参考文档

* [Windows Terminal](https://docs.microsoft.com/en-us/windows/terminal/)
* [How to Customize the New Windows Terminal App](https://www.howtogeek.com/426346/how-to-customize-the-new-windows-terminal-app/)
* [Custom key bindings in Windows Terminal](https://docs.microsoft.com/en-us/windows/terminal/customize-settings/key-bindings)
* [draculatheme](https://draculatheme.com/windows-terminal/)
