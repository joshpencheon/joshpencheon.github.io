---
layout: post
title: Installing ZFS with DKMS
tags: arch zfs
---

For licencing reasons, ZFS is not able to integrated into the mainline linux kernel. Some distributions (like Ubuntu) do package the modules into their kernels, whereas others (like Arch) exclude it from their official repositories. In this latter case, you then have two choices - enable a third-party repository that provides built binary modules for specific kernel versions, or use [DKMS](https://en.wikipedia.org/wiki/Dynamic_Kernel_Module_Support) to build your own. We'll be exploring the latter option.

## The downside of DKMS

It's possible that a DKMS build may fail, or that the kernel being provided by Arch's rolling release model gets ahead and is temporarily unsupported by ZFS. For this reason, we'll install multiple kernels so we can (hopefully!) have one as a fallback should something go wrong.

## Required packages

We'll use both the standard `linux` kernel package, as well as `linux-lts`. We'll need to install the `*-headers` packages for both too, so we can build the ZFS modules against them.

```bash
# Some of these will already be installed
sudo pacman --needed -S linux linux-headers linux-lts linux-lts-headers dkms mkinitcpio git
```

## ZFS in the AUR

Packages for ZFS are available in the Arch User Repository (AUR), but these are just git repositories containing `PKGBUILD` recipes; you clone them and build yourself using the pacman-provided `makepkg`. You can layer additional tools like `yay` to automate more of the use of the AUR, but we'll stick with the basics for now.

For ZFS, we'll need two packages - the `zfs-dkms` kernel modules and the `zfs-utils` utilities.

```bash
git clone https://aur.archlinux.org/zfs-dkms.git
git clone https://aur.archlinux.org/zfs-utils.git
```

Within each checkout, we run `makepkg -si` to [s]ync any missing dependencies from the official repositories, download and build, then temporarily elevate to `root` in order to [i]nstall.

## Pacman hooks

At this point, a number of pacman hooks (in `/usr/share/libalpm/hooks`) will come in to play - the `dkms-install` hook will detect that the `zfs-dkms` package has placed kernel modules into `/usr/lib/modules/`, and will therefore build them for each of the installed kernels.

A following `mkinitcpio-install` pacman hook will then create a new initramfs for each kernel, ensuring the new modules are available during early stage boot. In fact, mkinitcpio will by default also create a second `-fallback.img` initramfs for each kernel; this skips the `autodetect` mkinitcpio hook that would normally prune modules deemed unnecessary for the hardware, leading to a larger but hopefully more-compatible environment.

## Bootloader

In order for our LTS kernel to give us a chance to easily recover into a bootable system should we have any issues with the latest `linux` package, we'll need to ensure it's selectable in the bootloader. I'm using `systemd-boot` with the EFI System Partition mounted at `/boot`, so we need to create entries as `/boot/loader/entries/*.conf`. As an example, here's the config for booting with the LTS kernel and its corresponding pared-down initramfs:

```conf
# /boot/loader/entries/linux-lts.conf
title   Arch Linux (LTS kernel)
linux   /vmlinuz-linux-lts
initrd  /initramfs-linux-lts.img
options root=PARTUUID=xxxx-xxxx-xxxx rw
```

We can then use `bootctl list` to check they're all showing up, and expect to see entries like:

```
         type: Boot Loader Specification Type #1 (.conf)
        title: Arch Linux (LTS kernel)
           id: arch-lts.conf
       source: /boot//loader/entries/arch-lts.conf (on the EFI System Partition)
        linux: /boot//vmlinuz-linux-lts
       initrd: /boot//initramfs-linux-lts.img
      options: root=PARTUUID=xxxx-xxxx-xxxx rw
```

At this point, we should be able to reboot into our ZFS-enabled kernel!
