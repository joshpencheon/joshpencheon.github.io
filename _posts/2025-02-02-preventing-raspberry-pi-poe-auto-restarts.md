---
layout: post
title: Preventing Raspberry Pi PoE auto restarts
tags: raspberry-pi networking
---

I have a number of Raspberry Pis on my network, powered via PoE. I noticed that some of them would boot back up on their own after having been instructed to shut down, whereas others would remain off as intended - only booting back up when PoE power was cycled from the switch.

I have some Pi4s that are powered by [the official PoE+ HAT](https://www.raspberrypi.com/products/poe-plus-hat/) that all behave properly, and some Pi5s with [Waveshare's G HAT](https://www.waveshare.com/poe-hat-g.htm) that exhibited mixed behaviour; some would power down as intended, whereas others would reboot after a few seconds.

I narrowed it down to the Pi5's `POWER_OFF_ON_HALT` setting, which went set to `1` would cause the Waveshare HATs to reboot the Pi. After resetting the value to be `0` using `sudo rpi-eeprom-config --edit`, the problem was solved. It's a shame that the energy consumption will now hover around 1W for these Pis, even when off.
