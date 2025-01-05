---
layout: post
title: Manually updating Raspberry Pi EEPROM
tags: raspberry-pi
---

Whether you're using Raspberry Pi OS or another distribution (e.g. Ubuntu), the `rpi-eeprom-update` tool is likely available, and in the case of Raspberry Pi OS will update the firmware of the Pi automatically.

However, on Ubuntu the `rpi-eeprom` package tends to lag behind a little (or a lot, if you're using an LTS release). As an example, the latest firmware for an RPi4B [packaged in the 24.04 LTS](https://packages.ubuntu.com/noble/arm64/rpi-eeprom/filelist) is from May 2023, with the `default` release channel sticking with a build from January 2023.

If you'd like to stay a bit more up-to-date than this, it's easy to manually update the firmware of the Pi though. Raspberry Pi host the firmware [on GitHub](https://github.com/raspberrypi/rpi-eeprom), so you just need to pick the appropriate release for your hardware (`-2711` being for RPi4B, and `-2712` being for RPi5B), and the desired release channel - either `default`, or `latest` if you're after the cutting edge. All you then need do is download, schedule the update, and reboot:

```bash
wget https://github.com/raspberrypi/rpi-eeprom/raw/refs/heads/master/firmware-2712/default/pieeprom-2024-11-12.bin
sudo rpi-eeprom-update -f pieeprom-2024-11-12.bin
sudo reboot now
```

Once rebooted, you can check that the new firmware flashed OK by just running `rpi-eeprom-update`:

```
BOOTLOADER: up to date
   CURRENT: Tue Nov 12 16:10:44 UTC 2024 (1731427844)
    LATEST: Wed Dec  6 18:29:25 UTC 2023 (1701887365)
   RELEASE: default (/lib/firmware/raspberrypi/bootloader-2712/default)
            Use raspi-config to change the release.
```
Here, `LATEST` refers to the latest firmware that's been packaged in `rpi-eeprom` and unpacked to `/lib/firmware/raspberrypi/`, rather than the latest available from Raspberry Pi directly. Note that on Ubuntu, the mentioned `raspi-config` tool is not available; instead, the `/etc/default/rpi-eeprom-update` file can be used to configure which release channel to use if you _are_ content with sticking to packaged firmwares.
