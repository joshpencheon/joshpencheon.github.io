---
layout: post
title: Managing AUR packages with a local repository
tags: arch
---

The Arch User Repository (AUR) provides thousands of user-submitted packages, but each package in the AUR is just a git repository containing at minimum a `PKGBUILD` recipe. To install a package, you first check out the repository and review it, then use `makepkg` to install dependencies, download/verify the source material, and build it. You can then install the built package directly with `pacman -U`.

## The option of an AUR helper

This process starts to get unwieldy once you're using lots of inter-related packages from the AUR. At this point, people often start using an AUR helper like `yay`, which abstracts away much of the manual work that's normally needed when using the AUR.

However, if you're trying to be as selective as possible when using the AUR, an AUR helper might be overkill; a slightly more primitive solution would be sufficient.

## Sticking with pacman

It's possible to set up a local repository, and configure `pacman` to use it. I do this in `/srv/packages` (a location that's readily accessible to pacman's `alpm` download user).

### Building packages into the local repository

The first step is to configure `makepkg` to set up the local repo as the destination for built packages:

```bash
grep -B 1 ^PKGDEST /etc/makepkg.conf

#-- Destination: specify a fixed directory where all packages will be placed
PKGDEST=/srv/packages
```

We then need to compile a repository database file that indexes all these packages:

```bash
cd /srv/packages
repo-add --new local-aur.db.tar.zst *.pkg.tar.zst
```

This needs to be re-run each time a package is [re-]built with `makepkg`.

### Configuring pacman

We can add the following to `/etc/pacman.conf` so it will synchronize packages from our local repo (as well as `core` and `extra`).

```conf
[local-aur]
SigLevel = Optional TrustAll
Server = file:///srv/packages
```

We can now search for and install packages from our local repository just like we would with any other! 🎉

## Checking for AUR updates

As AUR packages get updated, you have to manually update your git checkouts. I do this with a script to iterate over all my local versions, which I keep in `~/AUR`:

```bash
fd -td -d1 -x bash -c "echo -en '{}\n  ' && git -C {} pull" \; . ~/AUR
```

I can then review any changes, and run `makepkg` / `repo-add` / `pacman -Syu` to install.
