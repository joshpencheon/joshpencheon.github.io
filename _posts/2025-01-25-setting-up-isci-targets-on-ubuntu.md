---
layout: post
title: Setting up iSCSI targets on Ubuntu
tags: zfs iscsi
---

We can use iSCSI to expose block devices across the network, with the ultimate aim of storing the root volume of one system on another system. The client system (the "initiator") makes requests to the host system (serving the "target"), and can be granted access to one or more "LUN" (logical units) which appear as block devices.

## TGT server setup

To get started, we'll install the package necessary to serve the iSCSI targets:

```bash
sudo apt install tgt
```

This will configure a systemd service, as well as providing `tgt-admin` (for target configuration) and `tgtadm` (for service configuration).

### Restricting to specific interfaces

By default, the `tgtd` process listens on all interfaces, on `:3260`. In my lab environment, I'm using a dedicated network for experimentation. This allows me to skip iSCSI authentication, but instead I'd like to limit `tgtd` to listen only on the dedicated interface. We can override the systemd service with:

```bash
sudo systemctl edit tgt.service
```

In this override file, we clear out and replace the `ExecStart` command to provide iSCSI options to listen on just a single interface:

```txt
[Service]
ExecStart=
ExecStart=/usr/sbin/tgtd -f --iscsi portal=192.168.20.1:3260
```

After restarting the service, we can verify where `tgtd` is listening:

```bash
sudo tgtadm --lld iscsi --op show --mode portal
# Portal: 192.168.20.1:3260,1
```
## Provisioning a block device

We'll use the following to create a fixed-size, preallocated `zvol`:

```bash
sudo zfs create -V 64G main-pool/test-vol
```

This will be made available as a block device at `/dev/zvol/main-pool/test-vol`.

## LUN setup

To expose this block device over the network as part of an iSCSI target, we'll need to add the following configuration at `/etc/tgt/conf.d/test.conf`. This will create a `netboot-test` target with a single disk LUN, without any authentication:

```txt
<target iqn.2001-04.com.netboot-test>
	backing-store /dev/zvol/main-pool/test-vol
</target>
```

Then use `sudo tgt-admin --execute` to read in the configuration, and `sudo tgt-admin --show` to check it has been processed.

## Testing an initiator

We'll use another host as the client, or "initiator", and try and discover the advertised targets and attach a LUN.

```bash
# Have the server respond with the available targets:
sudo iscsiadm --mode discovery --type sendtargets --portal 192.168.20.1

# Attach the LUN(s) from the specified target:
sudo iscsiadm --mode node --targetname iqn.2001-04.com.netboot-test --login
```

Check `dmesg` or `lsblk --scsi` to see which device has been created:

```bash
sudo dmesg | grep "Attached SCSI disk"
# [ 1070.017963] sd 0:0:0:1: [sda] Attached SCSI disk

lsblk --scsi
# NAME
#     HCTL       TYPE VENDOR   MODEL         REV SERIAL TRAN
# sda 0:0:0:1    disk IET      VIRTUAL-DISK 0001 beaf11 iscsi
```
We've now got a block device we can do whatever we like with, located on another host. To disconnect again, log out:

```bash
sudo iscsiadm --mode node --targetname iqn.2001-04.com.netboot-test --login
```

Next up - can we boot over iSCSI?
