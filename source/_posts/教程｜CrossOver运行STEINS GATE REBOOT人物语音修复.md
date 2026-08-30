---
title: 教程｜在 Mac 上用 CrossOver 运行《STEINS;GATE RE:BOOT》：人物语音问题与解决办法
date: 2026-08-30 20:02:56
tags: [指南, macOS, CrossOver, Steam, 游戏]
lang: zh-cn
translation_key: crossover-steins-gate-reboot-voice-fix
permalink: 2026/08/30/教程｜CrossOver运行STEINS-GATE-REBOOT人物语音修复/
---

前些时候，我在 Apple Silicon Mac 上通过 CrossOver 26.3.0 运行 Steam 版《STEINS;GATE RE:BOOT》（AppID 4012810）时，遇到了一个颇为古怪的问题：游戏可以正常启动，BGM 与界面音效也都没有异常，偏偏人物开口时完全没有声音。

最初很容易怀疑语音文件有所损坏，但日志最后指向了另一个原因：游戏的人物语音使用 **Windows Media Audio 2（WMA v2）**，而当前 CrossOver 中恰好缺少能完成这一段解码的 GStreamer libav 插件。这里记录一下排查的过程与最后采用的办法，也算为以后升级 CrossOver 时留一份可以回看的笔记。

> 本文对应的环境是 Apple Silicon Mac、CrossOver 26.3.0 与 Steam 版《STEINS;GATE RE:BOOT》。CrossOver 升级后可能更换自带的 GStreamer，所以不应将本文中的动态库原样搬到其他版本。

## 1. 问题出在哪里

人物语音要在 CrossOver 中播放，大致需要经过下面这条链路：

```text
游戏中的 WMA v2 语音
        ↓
Wine / CrossOver 的 wmadmod 与 winegstreamer
        ↓
GStreamer
        ↓
libav（FFmpeg）中的 avdec_wmav2 解码器
        ↓
人物语音
```

CrossOver 已经带有 Wine、GStreamer 核心与 macOS 的音频输出，因此游戏的 BGM 和普通音效仍然能播放。但它们正常并不能证明所有音频格式都已经能够解码。在这个案例中，数据已经被交给媒体链路，可是当中没有可用的 `libgstlibav`，所以人物开口时才会单独沉默。

CodeWeavers 也单独列出过 [Missing GStreamer 1.0 libav](https://support.codeweavers.com/en_US/missing-libraries/missinggstreamer1libav) 这类问题。它并非《STEINS;GATE RE:BOOT》独有，只是在这款游戏中恰好以“只有人声消失”的方式表现出来。

## 2. 先复制一个 bottle

在试验各种媒体组件之前，最重要的一步不是下载文件，而是复制当前能用的 Steam bottle。

1. 打开 CrossOver。
2. 右键当前可以正常启动 Steam 的 bottle。
3. 选择 **Duplicate Bottle**。
4. 将副本命名为 `Steam-SGRE-Voice-Test`。
5. 主 bottle 保持不动，后续操作全部在副本中完成。

我在测试时仍然使用 Graphics Backend 为 Auto、MSync 开启的组合。只要主 bottle 没有被修改，即使实验失败，也不会影响平时使用的 Steam。

## 3. 排查中走过的弯路

### 3.1 不要修改 CrossOver 的共享目录

不要把文件直接复制到：

```text
/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/
```

这里是所有 bottle 共用的运行环境。一个不兼容的动态库就可能影响 Steam 与其他 bottle，而且 CrossOver 更新时还可能覆盖这些文件。

### 3.2 不要混用 Windows 媒体 DLL

我曾尝试过 Windows 7 的原生 `wmadmod.dll`，它会调用 CrossOver 自带的 `mfplat.dll`，并在角色开口时因空指针写入而崩溃。如果继续替换为 Windows 7 的 `mfplat.dll`，游戏又会因为缺少较新的 `MFLockSharedWorkQueue` 接口而无法启动。

来自不同 Windows 版本、Wine 与 CrossOver 的媒体 DLL 并不是可以随意拼接的积木。

### 3.3 不要使用 Homebrew 的 ARM 插件

Apple Silicon 上通过 Homebrew 安装的 GStreamer 通常是 ARM64，而 CrossOver 运行这款 Windows 游戏时用到的媒体进程是 x86_64。两种架构不能直接混用。

### 3.4 不要让整个 Steam 扫描私有插件

如果将 `GST_PLUGIN_PATH` 一类变量直接加给 Steam，Steam 与它的网页组件也会扫描这套私有 libav，结果可能是长时间停在启动阶段。这些设置只应当传给 `sgre_steam.exe`。

## 4. 最后采用的方案

最后我没有替换任何 Windows 系统 DLL，而是在测试 bottle 内放入一套仅供 SGRE 使用的 GStreamer libav 组件：

```text
~/Library/Application Support/CrossOver/Bottles/
└── Steam-SGRE-Voice-Test/
    └── cx_gstreamer_libav/
        └── lib/
            ├── gstreamer-1.0/
            │   └── libgstlibav.dylib
            ├── libgstpbutils-1.0.0.dylib
            ├── libavcodec.60.dylib
            ├── libavformat.60.dylib
            ├── libavfilter.9.dylib
            ├── libavutil.58.dylib
            ├── libswresample.4.dylib
            ├── libz.1.dylib
            └── libbz2.1.dylib
```

组件来自 [GStreamer 官方 macOS Universal 1.24.13 运行时](https://gstreamer.freedesktop.org/download/)。选择 1.24 系列，是因为这个 CrossOver 版本自带 GStreamer 1.24.4，同一稳定系列更容易保持二进制兼容。对应的官方说明可见 [GStreamer 1.24 release notes](https://gstreamer.freedesktop.org/releases/1.24/)。

不过，1.24.13 插件声明需要 1.24.14 级别的库兼容版本，而 CrossOver 提供的是 1.24.4，所以不能只把文件复制进去。我对插件的最低兼容版本与依赖路径进行了适配，并对修改后的动态库做了 ad-hoc 签名。

成功使用的两个关键文件 SHA-256 如下：

```text
libgstlibav.dylib
5133e1d0ef42d81f1e39f87dd4618c6f04679fadf2c6fa844a3c185440a00ff0

libgstpbutils-1.0.0.dylib
5a3f007aabde95632acc35ac16c4902b0060883e063e60e1bb32a1175cf3b6b4
```

> 不建议基础用户自行使用十六进制编辑器修改 `.dylib`。更稳妥的方式是使用与 CrossOver 版本匹配、来源清楚且可以校验的组件。CrossOver 或 GStreamer 升级后，也应当重新检查兼容性。

## 5. 只在启动游戏时加载插件

即使文件已经放入 bottle，从 Steam 的“开始游戏”按钮启动时，SGRE 默认仍然只能看到 CrossOver 自带的插件目录。因此我又做了一个专用启动器，只在启动 `sgre_steam.exe` 的一刻设置解码环境：

```zsh
#!/bin/zsh
set -eu

bottle_name='Steam-SGRE-Voice-Test'
bottle_root="$HOME/Library/Application Support/CrossOver/Bottles/Steam-SGRE-Voice-Test"
plugin_root="$bottle_root/cx_gstreamer_libav"
wine_bin='/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/bin/wine'

export SteamAppId='4012810'
export SteamGameId='4012810'
export GST_PLUGIN_PATH="$plugin_root/lib/gstreamer-1.0"
export GST_PLUGIN_PATH_1_0="$plugin_root/lib/gstreamer-1.0"
export GST_PLUGIN_SYSTEM_PATH='/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/lib64/gstreamer-1.0'
export GST_PLUGIN_SYSTEM_PATH_1_0="$GST_PLUGIN_SYSTEM_PATH"
export GST_REGISTRY="$plugin_root/registry.bin"
export GST_REGISTRY_FORK='no'
export GST_PLUGIN_FEATURE_RANK='avdec_wmav2:MAX'

exec "$wine_bin" \
  --bottle "$bottle_name" \
  --no-wait \
  --cx-app 'C:\Program Files (x86)\Steam\steamapps\common\SGRE\sgre_steam.exe'
```

如果 bottle 名称不同，需要同时修改 `bottle_name` 和 `bottle_root`。为了不必每次打开终端，我把这个脚本封装并签名成了普通的 macOS 应用：

```text
STEINS;GATE REBOOT Voice Fix.app
```

日常启动时，先打开 `Steam-SGRE-Voice-Test` bottle 中的 Steam，但不要点击其中的“开始游戏”；然后双击这个专用启动器。这样变量只会影响 SGRE，不会让 Steam 整体进入私有插件环境。

## 6. 如何确认修复生效

“游戏没有崩溃”并不等于问题已经解决。我最后按照下面几点做了验证：

- 游戏窗口可以正常出现；
- BGM、界面与环境音效正常；
- 角色连续说多句台词时都有声音；
- 切换存档或场景后语音仍然正常；
- 退出后可以再次通过专用启动器进入。

诊断日志中明确出现了：

```text
avdec_wmav2
Decoded data
return flow ok
```

同时，运行中的游戏进程确实从测试 bottle 的 `cx_gstreamer_libav` 目录加载了 `libgstlibav.dylib`、`libavcodec.60.dylib` 等组件。人物语音的恢复因此不是偶然，而是 WMA v2 解码链路已经真正接通。

## 7. 故障排查与回滚

### 仍然没有语音

- 确认是通过专用启动器运行，而不是 Steam 的“开始游戏”。
- 确认脚本中的 bottle 名称与 CrossOver 左侧显示的名称完全一致。
- 确认 `cx_gstreamer_libav` 仍在测试 bottle 内。
- 确认 `libgstlibav.dylib` 是 x86_64 或 Universal，而不是仅 ARM64。

### 人物开口时崩溃

优先检查是否曾把原生 `wmadmod.dll`、`mfplat.dll` 或其他媒体 DLL 复制进 `drive_c/windows/system32`。如果有，应当停止继续混装，恢复测试前的 bottle 备份。

### Steam 卡在启动中

这通常是因为把 `GST_PLUGIN_PATH` 等变量传给了 Steam 本身。退出测试 bottle，恢复普通的 Steam 启动方式，只在 SGRE 专用启动器中设置这些变量。

### 升级 CrossOver 后再次失效

CrossOver 更新可能改变自带的 GStreamer 版本。不要将旧插件直接放进新版本的共享目录；应先复制 bottle，再检查新版本的 GStreamer 版本与架构。

本方案的改动都被限制在测试 bottle 和独立启动器中，回滚也很简单：停止使用 `STEINS;GATE REBOOT Voice Fix.app`，移走 bottle 内的 `cx_gstreamer_libav` 文件夹，或者直接恢复此前复制的干净 bottle。不需要删除仍然正常的主 Steam bottle，也不需要重装 CrossOver。

## 8. 简单的总结

这次问题的根因，是人物语音使用 WMA v2，而现有 CrossOver 媒体链路中缺少可用的 GStreamer libav 解码插件。最后的处理思路可以简化为：

```text
复制测试 bottle
→ 在 bottle 内放入版本与架构匹配的私有 libav 插件
→ 只给 SGRE 设置插件路径
→ 通过独立启动器运行
→ 用人物台词与解码日志双重验证
```

这样做虽然比直接丢几个 DLL 进去麻烦一些，但影响范围一直被限制在副本 bottle 之内，主 Steam 环境与 CrossOver 共享目录都能保持原状。
