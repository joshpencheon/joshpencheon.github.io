---
layout: post
title: Getting started with ZFS
tags: nvme raspberry-pi ssd zfs
---

Following on from [shrinking an ext4 partition]({% post_url 2025-01-12-shrinking-an-ext4-partition %}), we've got some free space to start making use of ZFS.

## Creating a pool and a dataset

To begin, we'll install the various utilities that are needed to manage ZFS:

```bash
sudo apt install zfsutils-linux
```

Next up, we'll create a ZFS pool. This is the underlying construct used to expose usable capacity to the system, by creating either ZFS datasets (for file storage) or "zvols" (for block storage).

A pool distributes the data it holds across one or more "vdevs". Simplistically, this is a bit like striping, but is actually more sophisticated; a pool may bias writes towards faster or emptier vdevs.

In turn, each vdev wraps one or more block device (drive/partition). Redundancy is achieved within individual vdevs, via mirroring or raidZ (storing parity data). It is possible to expand a pool by adding more vdevs, but individual vdevs are generally not expanded.

```bash
# Create a pool with a single vdev containing the single free single partition:
sudo zpool create main-pool /dev/nvme0n1p3

# Create a child 'time-machine' dataset within the root dataset:
sudo zfs create main-pool/time-machine
```

Hierarchical datasets inherit properties from their parents, but can be overridden - for example, you might want to have differing compression levels for different locations.

## Moving data over

There would be safer ways to copy data into ZFS, but I just moved it directly after stopping the samba daemon that was serving `/srv`:

```bash
# Stop Samba
sudo systemctl stop smbd.service

# Move the data in to ZFS
sudo mv /srv/time-machine/* /main-pool/time-machine/

# Ensure that the desired mountpoint is now empty
sudo rmdir /srv/time-machine/

# Configure ZFS to move the mount from /main-pool to /srv
sudo zfs set mountpoint=/srv main-pool

# Bring Samba back up
sudo systemctl start smbd.service
```

## Adding another vdev

In doing this, I'd freed up a lot of space within the ext4 partition, which would now go to waste. I therefore elected to [shrink it]({% post_url 2025-01-12-shrinking-an-ext4-partition %}) down again, and using `fdisk` to create a fourth partition in the empty space left behind.

This partition could then be taken over by ZFS, and used to grow the existing pool:

```bash
sudo zpool add main-pool /dev/nvme0n1p4
```

**Note:** this has expanded the capacity of the pool, but not added any resilience. Given these two vdevs exist on the same physical device, I don't think much is being given up; it's more like data fragmentation with extra steps / CPU cycles. In this case, it was just a bit of a hack to avoid faffing around trying to grow the existing partition to the left.

It's worth also bearing in mind that ZFS won't attempt to rebalance the existing data stored across the vdevs. If we generate some activity on the array, e.g. by running a Time Machine backup, we can monitor how the pool is being used. Here, we report stats for every 5 second period, and use verbose mode to give a breakdown by vdev:

```bash
zpool iostat -v main-pool 5
```

We can observe how it will start to use the new vdev, but at least initially the majority of the activity is going to be on the older vdev, with its larger capacity and more existing data:

```txt
               capacity     operations     bandwidth
pool         alloc   free   read  write   read  write
-----------  -----  -----  -----  -----  -----  -----
main-pool     223G  1.56T     44     16  5.45M  1.24M
  nvme0n1p3   223G  1.35T     44     10  5.44M   838K
  nvme0n1p4   255M   222G      0      7  14.5K   578K
-----------  -----  -----  -----  -----  -----  -----
```

