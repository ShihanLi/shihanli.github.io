---
layout: post
title:  "Clipboard"
date:   2026-11-26 23:10:43 -0800
categories: projects
---

I built a tiny clipboard manager for Mac, and so far it's been a joy to use.

Tapping `⌘V` pastes as usual. Holding `⌘V` opens a popup of your last 50 copies next to your mouse where you can use arrow keys to go up and down to select entries.

<p align="center">
  <img src="/assets/photos/clipboard-demo.png" alt="Clipboard Demo" width="420" style="border-radius: 12px; box-shadow: 0 6px 24px rgba(0, 0, 0, 0.18);"/>
</p>

The latest copy gets saved at the top. Copying something that's already on the list also moves it to the top since you'll probably want it again.

The history is kept in memory only. It's cleared when you quit the app, or restart your machine. It doesn't save anything a password manager marks as secret.

It doesn't make network requests, use third-party packages, or collect telemetry of any kind.

It does require Accessibility permission, which macOS now calls Device Control and Data Access (in earlier versions, it's System Settings > Privacy & Security > Accessibility). It needs this permission to hold back `⌘V` from pasting and instead open the popup, and to send a `⌘V` keystroke after you pick something.

The code is here: https://github.com/ShihanLi/clipboard

Download the latest release: https://github.com/ShihanLi/clipboard/releases