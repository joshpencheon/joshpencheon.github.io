---
layout: post
title: Preparing Raspberry Pi 5 for PCIe boot
tags: nvme raspberry-pi ssd
---

SD Cards are fine for many applications, but they're not the fastest or most durable media. Particularly if you're wanting to set up something like a little NAS, or run more write-intensive loads like Kubernetes, it can help to soup up your storage!

## Getting connected

Whereas preparing a bootable SD card for a Raspberry Pi is easy enough, it's a little trickier if you're wanting to boot from an NVMe SSD attached via a Pi 5's exposed PCI lanes.

The first thing you'll need is an adaptor, like [this board from Pimoroni](https://shop.pimoroni.com/products/nvme-base), to provide an M.2 interface supporting "M key" SSDs up to 80mm in length.

If, once you've connected  drive it isn't recognised, you may need to add to your boot `config.txt` the following to ensure the connector is enabled:

```
dtparam=pciex1
```

This should be automatic though, particularly if you're on the latest Pi firmware.

## Preparing a new SSD

While external docks for NVMe SSDs are available, it's less likely you'll just have one kicking around compared to a MicroSD reader. However, it's easy enough to install a fresh OS image onto the drive once you've got it connected to your Pi.

*It's also possible to try and copy your existing installation over while it's running, but you can end up in a slighty broken state as data changes during the cloning.*

We'll start by downloading a fresh copy of our OS, in this case Ubuntu 23.10:

```
$ cd ~/Downloads
$ wget https://cdimage.ubuntu.com/releases/23.10/release/ubuntu-23.10-preinstalled
-server-arm64+raspi.img.xz
```

and then verifying the SHA256 signature of it:

```
$ SHASUM=81886cefc6b7abe5baf26dbc353bc69924dfc76416c15c3c3d03cf5ba30c90e8
$ echo "$SHASUM *ubuntu-23.10-preinstalled-server-arm64+raspi.img.xz" | shasum -a 256 --check
ubuntu-23.10-preinstalled-server-arm64+raspi.img.xz: OK
```

We can now copy this compressed image file directly onto the raw NVMe device:

```
$ xzcat ubuntu-23.10-preinstalled-server-arm64+raspi.img.xz | sudo dd of=/dev/nvme0n1
```

Once done, we can check that two partitions have been created on the disk:

```
$ lsblk -o NAME,TYPE,SIZE,FSTYPE
NAME        TYPE   SIZE FSTYPE
nvme0n1     disk 931.5G
├─nvme0n1p1 part   512M vfat
└─nvme0n1p2 part   931G ext4
```

The small boot partition is `vfat`, and the root partition is `ext4` - the former is more easily read from non-linux devices (allowing you to for example mount the media and tweak boot configuration from macOS).

## Booting from the NVMe drive

Before we're able to *boot* from our new SSD, we must update the Pi's bootloader to allow for NVMe boot. We do this using `sudo -E rpi-eeprom-config --edit`, and updating with the following:

```
BOOT_ORDER=0xf416
```

The numbers are read right-to-left, meaning this allows for NVMe boot (`6`) to be tried first, followed by SD Card (`1`) and finally USB (`4`). Depending on how fragile your PCIe connection is, you might prefer to leave SD boot as a higher priority - inserting/removing the MicroSD card might be preferable to disconnecting your SSD during troubleshooting!

With that done, you should be good to reboot into your fresh OS, running off your speedy SSD!

## Free speed

Although not officially supported, it is possible to get the single PCIe lane that connects your SSD to the Pi's CPU to run at gen3 speeds, rather than gen2. Assuming your SSD has at least a gen3x1 interface (which it probably does!) this should grant you a theoretical 8Gbps of throughput.

To enable it, head to your boot `config.txt` and add the following line at the end:

```
dtparam=pciex1_gen=3
```

After rebooting, you should be able to check the link status by running `lspci` in verbose mode as root:

```
$ sudo lspci -vvvv | grep -E '^\w+|LnkSta:'
0000:01:00.0 Non-Volatile memory controller: Micron Technology Inc Device 5416 (rev 01) (prog-if 02 [NVM Express])
		LnkSta:	Speed 8GT/s, Width x1 (downgraded)
```

Here, `(downgraded)` refers to the fact the link width is only `x1`, when the SSD itself theoretically supports in this case `x4` for an ideal `32GT/s`. Even in this "downgraded" state, things will be pretty speedy for a Pi! 🏎️

