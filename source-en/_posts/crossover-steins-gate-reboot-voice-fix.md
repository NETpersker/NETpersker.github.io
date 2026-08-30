---
title: Tutorial | Fixing Missing Character Voices in STEINS;GATE RE:BOOT on Mac with CrossOver
date: 2026-08-30 20:02:56
tags: [Guide, macOS, CrossOver, Steam, Games]
lang: en
translation_key: crossover-steins-gate-reboot-voice-fix
permalink: 2026/08/30/crossover-steins-gate-reboot-voice-fix/
ai_translation: true
---

Some time ago, while running the Steam edition of *STEINS;GATE RE:BOOT* (AppID 4012810) through CrossOver 26.3.0 on an Apple Silicon Mac, I ran into a rather peculiar problem. The game started normally, and both the background music and interface sound effects worked, yet the characters were completely silent whenever they spoke.

At first it was tempting to suspect damaged voice files. The diagnostic logs eventually pointed elsewhere: the character voices use **Windows Media Audio 2 (WMA v2)**, while this CrossOver installation lacked the GStreamer libav plugin needed to decode that part of the audio chain. This article records how I traced the problem and the solution I eventually adopted. It should also serve as a note to revisit after future CrossOver upgrades.

> This article describes an Apple Silicon Mac running CrossOver 26.3.0 and the Steam edition of *STEINS;GATE RE:BOOT*. A CrossOver update may change its bundled GStreamer version, so the dynamic libraries described here should not be copied unchanged into another version.

## 1. Locating the problem

For the voices to play through CrossOver, the data has to travel through roughly the following chain:

```text
WMA v2 voice data in the game
        ↓
Wine / CrossOver wmadmod and winegstreamer
        ↓
GStreamer
        ↓
The avdec_wmav2 decoder from libav (FFmpeg)
        ↓
Character voices
```

CrossOver already includes Wine, the GStreamer core and macOS audio output. That is why the BGM and ordinary sound effects can still play. Their working correctly, however, does not mean every audio format can be decoded. In this case, the voice data reached the media chain, but there was no usable `libgstlibav` component to decode it. The failure therefore appeared only when a character began to speak.

CodeWeavers also documents the broader [Missing GStreamer 1.0 libav](https://support.codeweavers.com/en_US/missing-libraries/missinggstreamer1libav) problem. It is not unique to *STEINS;GATE RE:BOOT*; this game simply exposes it as missing voices while other audio remains intact.

## 2. Duplicate the bottle first

Before testing any media components, the most important step is not downloading a library but copying the working Steam bottle.

1. Open CrossOver.
2. Right-click the bottle in which Steam currently starts normally.
3. Select **Duplicate Bottle**.
4. Name the copy `Steam-SGRE-Voice-Test`.
5. Leave the main bottle untouched and perform every later experiment in the copy.

I kept the known stable graphics settings—Graphics Backend set to Auto and MSync enabled. As long as the main bottle is not altered, a failed experiment will not affect the Steam installation used every day.

## 3. Approaches that did not work

### 3.1 Do not modify CrossOver's shared directory

Do not copy files directly into:

```text
/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/
```

This environment is shared by every bottle. One incompatible dynamic library can affect Steam and other bottles, and a CrossOver update may overwrite the files anyway.

### 3.2 Do not mix Windows media DLLs

I tested the native Windows 7 `wmadmod.dll`. It called CrossOver's bundled `mfplat.dll` and crashed with a null-pointer write as soon as a character spoke. Replacing that file with the Windows 7 `mfplat.dll` did not help: the game then failed to start because that version lacks the newer `MFLockSharedWorkQueue` interface.

Media DLLs taken from different Windows versions, Wine and CrossOver are not interchangeable building blocks.

### 3.3 Do not use the ARM Homebrew plugin

On Apple Silicon, GStreamer installed through Homebrew is normally ARM64, while the media process used by CrossOver for this Windows game is x86_64. Those architectures cannot be mixed directly.

### 3.4 Do not apply the private decoder environment to all of Steam

If variables such as `GST_PLUGIN_PATH` are applied to Steam itself, Steam and its web components will also scan the private libav bundle. Steam may then remain stuck during startup. The decoder environment should be passed only to `sgre_steam.exe`.

## 4. The solution I used

Instead of replacing any Windows system DLL, I placed a private GStreamer libav bundle inside the test bottle and made it available only to SGRE:

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

The components came from the [official GStreamer macOS Universal 1.24.13 runtime](https://gstreamer.freedesktop.org/download/). I chose the 1.24 series because this CrossOver version bundles GStreamer 1.24.4, and staying within one stable series makes binary compatibility more likely. The corresponding upstream information is available in the [GStreamer 1.24 release notes](https://gstreamer.freedesktop.org/releases/1.24/).

There is an important limitation. The official 1.24.13 plugin declares a minimum compatible library version at the 1.24.14 level, while CrossOver supplies 1.24.4. Simply copying the files is therefore insufficient. I adapted the plugin's minimum compatibility version and dependency paths, then applied ad-hoc signatures to the modified dynamic libraries.

The SHA-256 values of the two critical files that worked in this environment are:

```text
libgstlibav.dylib
5133e1d0ef42d81f1e39f87dd4618c6f04679fadf2c6fa844a3c185440a00ff0

libgstpbutils-1.0.0.dylib
5a3f007aabde95632acc35ac16c4902b0060883e063e60e1bb32a1175cf3b6b4
```

> I do not recommend that inexperienced users modify `.dylib` files with a hex editor. Use components that match the CrossOver version, come from a known source and can be verified by checksum. Compatibility should be checked again after any CrossOver or GStreamer update.

## 5. Load the plugin only when starting the game

Even after the files are placed inside the bottle, launching from Steam's **Play** button still lets SGRE see only CrossOver's default plugin directory. I therefore made a dedicated launcher that sets the decoder environment only when `sgre_steam.exe` starts:

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

If the bottle has a different name, both `bottle_name` and `bottle_root` need to be changed. To avoid opening Terminal every time, I packaged and signed the script as a normal macOS application:

```text
STEINS;GATE REBOOT Voice Fix.app
```

For ordinary use, I first start Steam in the `Steam-SGRE-Voice-Test` bottle without clicking **Play**, then open the dedicated launcher. This keeps the variables limited to SGRE instead of placing all of Steam inside the private plugin environment.

## 6. Verifying that the fix is real

The fact that the game no longer crashes is not enough. I checked all of the following:

- the game window opens normally;
- BGM, interface sounds and ambient effects work;
- several consecutive spoken lines contain voices;
- voices remain after loading a save or changing scenes;
- the game can be closed and started again through the launcher.

The diagnostic log explicitly contained:

```text
avdec_wmav2
Decoded data
return flow ok
```

The running game process also loaded `libgstlibav.dylib`, `libavcodec.60.dylib` and the other components from the test bottle's `cx_gstreamer_libav` directory. The voices had not returned by accident: the WMA v2 decoding chain was actually connected.

## 7. Troubleshooting and rollback

### Voices are still missing

- Make sure the game was started through the dedicated launcher rather than Steam's **Play** button.
- Check that the bottle name in the script exactly matches the name shown in CrossOver.
- Confirm that `cx_gstreamer_libav` is still inside the test bottle.
- Confirm that `libgstlibav.dylib` is x86_64 or Universal, not ARM64 only.

### The game crashes when a character speaks

Check whether native `wmadmod.dll`, `mfplat.dll` or other media DLLs were copied into `drive_c/windows/system32`. If so, stop mixing components and restore the bottle backup made before testing.

### Steam remains stuck while starting

This usually means `GST_PLUGIN_PATH` and related variables were passed to Steam itself. Exit the test bottle, restore the normal Steam launch method, and keep those variables only in the dedicated SGRE launcher.

### Voices disappear after updating CrossOver

A CrossOver update may change its bundled GStreamer version. Do not copy the old plugin into the new shared directory. Duplicate the bottle first, then inspect the new GStreamer version and architecture before adapting the bundle again.

Because all changes are confined to the test bottle and the separate launcher, rollback is simple: stop using `STEINS;GATE REBOOT Voice Fix.app`, move the `cx_gstreamer_libav` directory out of the bottle, or restore the clean duplicated bottle. There is no need to delete the working main Steam bottle or reinstall CrossOver.

## 8. A short summary

The missing voices were caused by WMA v2 character audio meeting a CrossOver media chain without a usable GStreamer libav decoder. The final method can be reduced to the following sequence:

```text
Duplicate the test bottle
→ place a version- and architecture-matched private libav plugin inside it
→ expose the plugin path only to SGRE
→ run the game through a separate launcher
→ verify both spoken dialogue and decoder logs
```

This takes more work than dropping a few DLLs into the bottle, but it keeps the entire change inside a disposable copy. The main Steam environment and CrossOver's shared directory remain untouched.
