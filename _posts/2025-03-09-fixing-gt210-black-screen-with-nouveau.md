---
layout: post
title: Fixing GT210 black screen with Nouveau
tags: nvidia
---

I've got an ancient Nvidia GT210, a GPU that was old even when it was new. Recently, I've been having issues with kernel and driver support. Nvidia stopped supporting this card a long time ago ([340.x](https://www.nvidia.com/en-us/geforce/drivers/results/113161/) being the last driver series that supported it), so I've always used the reverse-engineered nouveau drivers instead.

In trying to update to Ubuntu 24.10, I was getting just a black screen (sometimes with a cursor) whenever trying to start `gdm`, followed by hangs / long pauses before a different virtual terminal could be switched to. A pre-release build of 25.04 exhibited the same symptoms, as did a very pared down Arch + GNOME install. No end of kernel flags were tried without success, and with no smoking gun in any logs I could find I became resigned to the idea that my GT210 had finally become obsolete.

However, I stumbled across a suggestion that the issue might be with Mutter, and found this minimal addition to `/etc/environment` fixed everything:

```bash
# cat /etc/environment
# Hail Mary to try and get GT210 working with GNOME:
GSK_RENDERER=gl
MUTTER_DEBUG_DISABLE_TRIPLE_BUFFERING=1
MUTTER_DEBUG_USE_KMS_MODIFIERS=0
```
```bash
# pacman -Q linux mesa gdm mutter
linux 6.13.5.arch1-1
mesa 1:24.3.4-1
gdm 47.0-2
mutter 47.6-1
```

The GT210 and its 1080p maximum resolution lives on, hurray!
