---
title: "Chrome Wayland Fcitx 漏字修复"
date: 2024-02-14T20:48:00+08:00
lastmod: 2024-02-14T20:48:00+08:00
draft: false
tags: ["Linux","Wayland"]
categories: ["Linux"]

---

最近在自用的 Fedora 系统上切换了Wayland显示，输入法就有一些奇怪的漏字现象。

具体表现为，输入一句汉字后，会有一些拼音的字母跳过输入法直接进入程序。形成类似于 `我说` => `我s活` 这样的奇怪句子。

翻了一下 [ArchWiki](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Chromium_.2F_Electron)，发现已经有现成的修复方案了。

如果你和我一样是 Fcitx5+Wayland 的搭配，编辑Chrome的 `.desktop` 文件，在命令后添加以下参数即可。
`--enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime`

Chrome和VSCode都可以这么修复。

