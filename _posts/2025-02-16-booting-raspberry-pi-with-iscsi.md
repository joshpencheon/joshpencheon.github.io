---
layout: post
title: Booting Raspberry Pi with iSCSI
tags: netboot networking raspberry-pi zfs iscsi
---

In order to get a Raspberry Pi booting without any physically attached storage, we need to consider a number of different phases:

* Where does the bootloader (RPi firmware) get a boot image / kernel from?
* How is that initial boot environment delivered to the RPi?
* How does the rest of the root filesystem become available?
* How can the booting device update its own kernel?

Let's address those one at a time.

## Enabling netboot

This is something [I've covered before]({% post_url 2023-12-17-enabling-netboot-on-raspberry-pi %}), but in summary we have to include and prioritise a `BOOT_ORDER` value of `2` in the config stored in the RPi's EEPROM, which will instruct it to try and source a kernel over the network via TFTP (Trivial File Transfer Protocol). While updating the EEPROM, we'll also set `TFTP_PREFIX=2` - more on that later.

For other hardware, you may find configuration in your BIOS that allows similar.

## Advertising a TFTP server

While it is possible to hard-code in the EEPROM config where the bootloader should go to find the TFTP server, I prefer to advertise this as part of [the DHCP offer](https://www.rfc-editor.org/rfc/rfc2132.html#section-9) - but watch out for potential issues if you start doing this for your whole network!

I use ISC's DHCP server on the server that is also acting as gateway for the network, and will additionally be taking on the role of TFTP server (and iSCSI server). Therefore, to the subnet configuration in `/etc/dhcp/dhcpd.conf` it is a simple matter of adding:

```diff
  range 192.168.20.10 192.168.20.200;
+ option tftp-server-name "192.168.20.1";
  option routers 192.168.20.1;
```

It's worth noting that DHCP also supports advertising a "next server" for clients to contact, but the RPi firmware doesn't support that, so I stuck with the BOOTP extension option 66.

We can then test this using one of `nmap`'s built-in scripts:

```
# sudo nmap --script broadcast-dhcp-discover
|   Response 1 of 1:
|     IP Offered: 192.168.20.26
|     DHCP Message Type: DHCPOFFER
|     Server Identifier: 192.168.20.1
|     [ ... ]
|_    TFTP Server Name: 192.168.20.1
```
## Hosting a TFTP server

Now the RPi will know where to look for its boot image, we now need to provision something for it to find! A TFTP server can be installed with:

```bash
sudo apt-get install tftpd-hpa
```

It is configured in `/etc/default/tftpd-hpa`; the defaults may be fine for you, but I configured the following starting settings:

```
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="192.168.20.1:69"
```

The `TFTP_PREFIX=2` we set earlier in the RPi's EEPROM will mean that it requests files it needs prefixed with its MAC address (easy to find) rather than its serial number (less easy to find).

### Extracting the boot partition

I'll be using ZFS to store both the block device to be served over iSCSI as well as the boot files to be served over TFTP. Setting up individual datasets will allow for snapshotting and rollback.

```bash
# Set up a dataset for the client with MAC address xx-xx-xx-xx-xx-xx:
CLIENT_DATASET="main-pool/netboot/xx-xx-xx-xx-xx-xx"
sudo zfs create -p $CLIENT_DATASET
sudo zfs set mountpoint=none main-pool/netboot

# Create a child zvol block device for iSCSI at '/vol':
sudo zfs create -V 64G $CLIENT_DATASET/vol

# Fill it with a disk image:
sudo dd \
  if=ubuntu-24.04.1-preinstalled-server-arm64+raspi.img \
  of=/dev/zvol/$CLIENT_DATASET/vol \
  status=progress

# Create a child dataset for TFTP at '/tftp':
sudo zfs create -o mountpoint=/srv/tftp/xx-xx-xx-xx-xx-xx $CLIENT_DATASET/tftp

# Scan for partitions in the new block device:
sudo fdisk -l /dev/zvol/$CLIENT_DATASET/vol

# Temporarily mount the first (boot) partition, and copy the contents to the TFTP directory:
sudo mount /dev/zvol/$CLIENT_DATASET/vol-part1 /mnt
sudo cp -r /mnt/* /srv/tftp/xx-xx-xx-xx-xx-xx/
sudo umount /mnt

# Finally, recursively snapshot this pristine state:
sudo zfs snapshot -r $CLIENT_DATASET@pristine-24.04
```

### Verifying it works

To observe things working, we can monitor DHCP logs and sniff for traffic on `:69` (in my case, shown here on the `vlan20` interface) when a client RPi tries to netboot:

```bash
# a DHCPREQUEST should arrive as per normal:
journalctl -u isc-dhcp-server.service -b -f

# then, boot files should be requested via TFTP:
sudo tcpdump -vv -i vlan20 port 69
```

## Fetching the root filesystem

At this point, our client RPi should be able to get a kernel started, but then panic as it won't be able to find a root filesystem. For Ubuntu 24.04, the kernel `cmdline.txt` that we will have delivered over TFTP will have instructed it to look for a filesystem labelled as `writeable` to use as the root:

```conf
... root=LABEL=writable rootfstype=ext4 rootwait ...
```

This can still work, but we'll need the kernel to act as an iSCSI initiator and fetch the block device over the network first before the relevant partition can be found. We can do that by editing `cmdline.txt` to include the following before `root=...` (again, substituting in the client's MAC address):

```conf
iscsi_initiator=iqn.2025-02.dev.pencheon:cluster:xx-xx-xx-xx-xx-xx iscsi_target_name=iqn.2025-02.dev.pencheon:root:xx-xx-xx-xx-xx-xx iscsi_target_ip=192.168.20.1
```

This will look for a iSCSI target called `iqn.2025-02.dev.pencheon:root:xx-xx-xx-xx-xx-xx` on the `192.168.20.1` host, self-identifying as `iqn.2025-02.dev.pencheon:cluster:xx-xx-xx-xx-xx-xx`.

In order to _serve_ this, we'll need to [set up the iSCSI target]({% post_url 2025-01-25-setting-up-isci-targets-on-ubuntu %}), with the following configuration in `/etc/tgt/conf.d/xx-xx-xx-xx-xx-xx.conf`:

```
<target iqn.2025-02.dev.pencheon:root:xx-xx-xx-xx-xx-xx>
	initiator-name iqn.2025-02.dev.pencheon:cluster:xx-xx-xx-xx-xx-xx
	backing-store /dev/zvol/main-pool/netboot/xx-xx-xx-xx-xx-xx/vol
</target>
```

### Verifying it works

All being well, the client should now fetch the initial boot image over TFTP, start the kernel, attach the block device over iSCSI, and continue booting:

```
[   13.451314] Loading iSCSI transport class v2.0-870.
[   13.462700] iscsi: registered transport (tcp)
[   14.474254] scsi host0: iSCSI Initiator over TCP/IP
[   14.483862] scsi 0:0:0:0: RAID              IET      Controller       0001 PQ: 0 ANSI: 5
[   14.495861] scsi 0:0:0:0: Attached scsi generic sg0 type 12
[   14.502170] scsi 0:0:0:1: Direct-Access     IET      VIRTUAL-DISK     0001 PQ: 0 ANSI: 5
[   14.514185] sd 0:0:0:1: Attached scsi generic sg1 type 0
[   14.514307] sd 0:0:0:1: Power-on or device reset occurred
[   14.525439] sd 0:0:0:1: [sda] 134217728 512-byte logical blocks: (68.7 GB/64.0 GiB)
[   14.533353] sd 0:0:0:1: [sda] Write Protect is off
[   14.538571] sd 0:0:0:1: [sda] Write cache: enabled, read cache: enabled, supports DPO and FUA
[   14.550184]  sda: sda1 sda2
[   14.553106] sd 0:0:0:1: [sda] Attached SCSI disk
```

Once logged in, we should see the two partitions on this device, labelled and mounted:

```
lsblk -o name,size,mountpoints,label
NAME    SIZE MOUNTPOINTS       LABEL
loop0  33.7M /snap/snapd/21761
sda      64G
├─sda1  512M /boot/firmware    system-boot
└─sda2 63.5G /                 writable
```

## Allowing the kernel to be updated

So far, so good! But there's one glaring issue that becomes apparent when a client tries to update its boot information at `/boot/firmware`. The bootloader is now getting this via TFTP, and being served an frozen extracted copy, rather than what it might have updated on `/dev/sda1`. This makes running system updates tricky!

### A note on NFS roots

If we were using NFS to serve the entire root filesystem, this would be easy enough to fix; we could bind mount the `boot/firmware` directory on the server to also be served via TFTP. Whilst this is technically circumventing the multi-access that the NFS server would manage, `tftp-hpa` is read-only by default, and the two access modes shouldn't be required at the same time.

### The problem with iSCSI roots

From the perspective of the server, the block device being used for the iSCSI target is nothing more than that; it's not mounted or readable by the server, nor should it be - unless the filesystem contained within is suitable for concurrent access (e.g. glusterfs), even a filesystem mounted a second time read-only won't work. A filesystem being mounted read-only is not the same as the underlying block device being read-only (ext4 writes to its journal regardless of the mount flag), and regardless of that a non-clustered filesystem mount would cache reads and thus go stale.

#### The workaround

NFS to the rescue again - but we'll instead use it to serve just the TFTP version of the boot partition back to the client, so it can re-mount it. With `nfs-kernel-server` installed, we can configure it:

```conf
# cat /etc/exports
/srv/tftp/xx-xx-xx-xx-xx-xx 192.168.20.0/24(rw,sync,no_subtree_check,no_root_squash)
```

...and then apply the config with `sudo exportfs -a`.

Because boot files need to be owned by `root` and sometimes not even world-readable, we unfortunately need the `no_root_squash` option on the mount, and to punch some holes in the TFTP server's configuration too:

```conf
# in /etc/default/tftpd-hpa

# allow serving of files accessible only to root by running as root:
TFTP_USERNAME="root"

# 'secure' tries to sandbox access to just the TFTP directory
# 'permissive' skips extra checks that require world-readable files
TFTP_OPTIONS="--secure --permissive"
```

On the client, we'll need to install `nfs-common`, then update `/etc/fstab`:

```diff
  LABEL=writable	/	ext4	defaults	0	1
- LABEL=system-boot	/boot/firmware	vfat	defaults	0	1
+ 192.168.20.1:/srv/tftp/xx-xx-xx-xx-xx-xx	/boot/firmware	nfs	defaults	0	1
```

...running `sudo mount -a` to apply.

It is a shame we have to manually install a package outside the base install on the client to get this working. Such provisioning is something that can be achieved with `cloud-init`, but that's for another day.

## Wrapping up

At this point, you might like to shut down the client, and take another set of snapshots with ZFS:

```
sudo zfs snapshot -r $CLIENT_DATASET@after-initial-config
```

In order to return to this state, we can use ZFS to roll back. Note that when rolling back, `-r` is used to remove any more recent snapshots than the targeted one, rather than meaning "recursive". Rollbacks have to be done to each child individually:

```
sudo zfs rollback -r $CLIENT_DATASET@after-initial-config
sudo zfs rollback -r $CLIENT_DATASET/vol@after-initial-config
sudo zfs rollback -r $CLIENT_DATASET/tftp@after-initial-config
```
