---
layout: post
title: Enabling Netboot on Raspberry Pi
tags: netboot networking raspberry-pi
---

SD Cards are not the most durable things in the world, and using them as the boot media can cause problems over time. It's possible to boot in other ways too, such as from an SSD connected via USB, but a good option when you've got lots of devices is to boot over the network. This allows you to central your storage, and can allow for fast re-imaging of devices.

The Raspberry Pi bootloader supports [PXE boot](https://en.wikipedia.org/wiki/Preboot_Execution_Environment), but not by default. In order to enable it, we have to set a flag in the Pi's built-in EEPROM. There is a tool to do this which comes installed by default on Raspbian, and availble from the repositories on Ubuntu:

```bash
sudo apt-get install rpi-eeprom
```

With the tool installed, we can run the following to open an editor and update the EEPROM configuration:

```bash
sudo -E rpi-eeprom-config --edit
```

We'll make two additions to the file:

```bash
BOOT_ORDER=0x21
TFTP_PREFIX=2
```

Breaking this down, setting `BOOT_ORDER=0x21` means the Pi will first try to boot from the SD Card (`1`), but if there isn't one installed it will fall back to netboot (`2`).

Setting `TFTP_PREFIX=2` means that the bootloader will request the boot files it needs from the TFTP server prefixing the filepaths with the Pi's mac address rather than its device serial number. This is convenience as mac addresses are generally easier to discover and keep track of.

Once we've written the file, it's important to reboot the Pi to apply the changes to the EEPROM.
