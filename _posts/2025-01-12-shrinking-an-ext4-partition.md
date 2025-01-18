---
layout: post
title: Shrinking an ext4 partition
tags: nvme raspberry-pi ssd
---

I use a Raspberry Pi as a Time Machine server, and want to improve the setup to allow for offsite backups too. In order to do this, I've like to use ZFS snapshotting along with [syncoid](https://github.com/jimsalterjrs/sanoid#syncoid) to ship them to a remote server.

The RPi is using a single NVMe SSD, and holding the Time Machine data within the root `ext4` partition. I'd like to shrink down that partition, create a new partition for ZFS, and then transfer the data over. (See [part two]({% post_url 2025-01-17-getting-started-with-zfs %})!)

It is possible to shrink an `ext4` partition, but only when not mounted. Therefore, we'll need to boot the RPi from another medium temporarily. Using `rpi-eeprom-config`, we can adjust the boot order (read from the right) to boot from SD card (1) before NVMe (6):

```txt
BOOT_ORDER=0xf461
```

Before the root filesystem is unmounted, we can also use `df -h` to check how much of the partition's current capacity is being used by the filesystem:

```txt
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p2  1.8T  232G  1.5T  14% /
```

## Making some space

Once booted from the SD card, the first thing we need to do is shrink the filesystem that's using the partition. I've elected to shrink it down with the Time Machine data still in place, but you could move that data off first to reclaim maximum space if you wanted to.

```bash
# Run the prerequisite filesystem check:
sudo e2fsck -f /dev/nvme0n1p2

# Resize the filesystem down:
sudo resize2fs -p /dev/nvme0n1p2 256G
```

Once this is done, we can update the partition table to shrink the partition down around the filesystem. Raspberry Pi devices use an MBR scheme rather than GPT, so we can use `fdisk` to do this. Firstly, inspecting the output of `fdisk -l /dev/nvme0n1` we see the existing partition layout; we want to shrink down the latter partition to 256G, freeing up space for a third partition which we'll use for ZFS.

```txt
Device         Boot   Start        End    Sectors  Size Id Type
/dev/nvme0n1p1 *       2048    1050623    1048576  512M  c W95 FAT32 (LBA)
/dev/nvme0n1p2      1050624 3907029134 3905978511  1.8T 83 Linux
```
We then run `sudo fdisk /dev/nvme0n1` to start the interactive program, in which we can remove and re-add a smaller version of the data partition:

```txt
Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): d
Partition number (1,2, default 2): 2

Partition 2 has been deleted.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (2-4, default 2):
First sector (1050624-3907029167, default 1050624):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (1050624-3907029167, default 3907029167): +256G

Created a new partition 2 of type 'Linux' and of size 256 GiB.
Partition #2 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: N

Command (m for help): w

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Once the new partition table has been written, we can run `e2fsck` against our partition to double check that the filesystem hasn't been broken, and then use `fdisk` again to create a third partition; the defaults will create a 'Linux' partition consuming the rest of the space, which we can see by running `lsblk /dev/nvme0n1`:

```txt
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0  1.8T  0 disk
├─nvme0n1p1 259:6    0  512M  0 part
├─nvme0n1p2 259:7    0  256G  0 part
└─nvme0n1p3 259:8    0  1.6T  0 part
```

We should now be OK to boot back from the NVMe drive, with our now-smaller root partition. Next up, [let's get started with ZFS]({% post_url 2025-01-17-getting-started-with-zfs %})!
