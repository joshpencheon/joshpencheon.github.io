---
layout: post
title: Mounting ZFS encrypted datasets on boot
tags: systemd zfs
---


I previously wrote about [enabling encryption for a ZFS dataset]({% post_url 2025-02-23-retrospectively-enabling-zfs-encryption %}), but didn't cover how to go about mounting such a dataset on boot. If using `keylocation=prompt` an interactive solution will be required, but as I'm using a keyfile held outside of ZFS an automatic solution is possible.

## Overview of the problem

Conceptually, there are three things that need to happen:

```bash
# The system needs to learn of the key's location:
zfs get keylocation main-pool/test-dataset

# Once available, this key needs to be loaded in to ZFS:
zfs load-key main-pool/test-dataset

# Once loaded, the requiring dataset can be mounted:
zfs mount main-pool/test-dataset
```

It turns out that this is a solved problem, and a solution is achievable using a combination of [the ZFS event daemon](https://openzfs.github.io/openzfs-docs/man/v2.2/8/zed.8.html) and a [systemd generator](https://www.freedesktop.org/software/systemd/man/latest/systemd.generator.html).

## Systemd to the rescue

OpenZFS [bundles a system generator](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-mount-generator.8.html) that gets installed at `/usr/lib/systemd/system-generators/zfs-mount-generator`. This runs very early in the boot process (before units are loaded) and has an opportunity to generate further units dynamically.

It is too early in the boot process to interrogate ZFS directly for mounts, so instead the mount generator is driven by a cached version of `zfs list` output. This cache is maintained by the ZEDLET `history_event-zfs-list-cacher`, which is in turn invoked by the ZFS Event Daemon `zed.service` when datasets are modified. From [the source](https://github.com/openzfs/zfs/blob/master/cmd/zed/zed.d/history_event-zfs-list-cacher.sh.in) of this ZEDLET, we can see it will keep a cached version of ZFS datasets' info written to `/etc/zfs/zfs-list.cache/<POOL_NAME>`, as long as the location exists.

This location doesn't exist by default, but we can ensure it does:

```bash
sudo mkdir -p /etc/zfs/zfs-list.cache
sudo touch /etc/zfs/zfs-list.cache/main-pool
```

Once we've triggered some cacheable activity (e.g. tweaking a dataset property), this cache should be written to by the ZEDLET.

With a populated cache in place, all we should have to do is reboot! Once the system comes back up, we should find new systemd units that have been generated as a result; a mount unit bound to a key loading service.

### Checking out the dynamic units

If we look at the new mount unit for our encrypted dataset (with `systemctl cat srv-test\\x2ddataset.mount`) we can see that it now binds to a service for loading the dataset's key:

```ini
# /run/systemd/generator/srv-test\x2ddataset.mount
# Automatically generated by zfs-mount-generator

[Unit]
SourcePath=/etc/zfs/zfs-list.cache/main-pool
Documentation=man:zfs-mount-generator(8)
Before=zfs-mount.service local-fs.target
After= zfs-load-key-main\x2dpool-test\x2ddataset.service
Wants=
BindsTo=zfs-load-key-main\x2dpool-test\x2ddataset.service

[Mount]
Where=/srv/test-dataset
What=main-pool/test-dataset
Type=zfs
Options=defaults,atime,relatime,dev,exec,rw,suid,nomand,zfsutil
```

Taking a look at _that_ service's definition, we can see it waits for the `keylocation` to be mounted, then runs `zfs load-key` for our dataset if it still needs it.

```ini
# /run/systemd/generator/zfs-load-key-main\x2dpool-test\x2ddataset.service
# Automatically generated by zfs-mount-generator

[Unit]
Description=Load ZFS key for main-pool/test-dataset
SourcePath=/etc/zfs/zfs-list.cache/main-pool
Documentation=man:zfs-mount-generator(8)
DefaultDependencies=no
Wants=
After=
RequiresMountsFor='/root/main-pool-test-dataset.key'

[Service]
Type=oneshot
RemainAfterExit=yes
# This avoids a dependency loop involving systemd-journald.socket if this
# dataset is a parent of the root filesystem.
StandardOutput=null
StandardError=null
ExecStart=/bin/sh -c 'set -eu;keystatus="$$(/sbin/zfs get -H -o value keystatus "main-pool/test-dataset")";[ "$$keystatus" = "unavailable" ] || exit 0;/sbin/zfs load-key "main-pool/test-dataset"'
ExecStop=/bin/sh -c 'set -eu;keystatus="$$(/sbin/zfs get -H -o value keystatus "main-pool/test-dataset")";[ "$$keystatus" = "available" ] || exit 0;/sbin/zfs unload-key "main-pool/test-dataset"'
```

All being well, these units will have done their job and the encrypted dataset should be mounted already:

```bash
# findmnt --mountpoint /srv/test-dataset
TARGET            SOURCE                 FSTYPE OPTIONS
/srv/test-dataset main-pool/test-dataset zfs    rw,relatime,xattr,noacl,casesensitive
```

This in turn should have triggered any [dependent downstream services (like Samba)]({% post_url 2025-02-24-binding-services-together-with-systemd %}) to start too.
