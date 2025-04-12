---
layout: post
title: Shipping ZFS snapshots with Syncoid
tags: zfs
---

[Sanoid](https://github.com/jimsalterjrs/sanoid) is a popular choice for maintaining a history of a ZFS dataset via snapshots. It gives us the ability to use ZFS `send`/`recv` to efficiently back up the dataset to a separate machine, transmitting just the deltas between snapshots. [Syncoid](https://github.com/jimsalterjrs/sanoid#syncoid) is a tool to help with this, bundled as part of `sanoid`. If used with something like [tailscale](https://tailscale.com/), it can make offsite backups a breeze!

## Prerequisites

We'll want a few ancillary optional packages installed (`pv`, `mbuffer`, `lzop`) to improve the experience. The main thing we'll want to get configured is non-root users on the source (user: `syncoid-sender`) and destination (user: `syncoid-receiver`) machines with sufficient delegated ZFS permissions to transfer the snapshots.

### Sending permissions

We'll allow the sending user to send data from the desired dataset(s), as well as to be able to place ZFS holds; this allows us to prevent snapshots that are being used by the process from being removed prematurely.

```bash
sudo zfs allow syncoid-sender hold,send main-pool/test-dataset
```

_Note that the granting of `send` does permit already-decrypted data to be sent. The is not currently a separate grant that only permits `send --raw`, although there is [a proposal to add one](https://github.com/openzfs/zfs/issues/13099)._

### Receiving permissions

On the receiving end, we'll want to configure a dataset in which to receive the backups:

```bash
sudo zfs create -o readonly=on main-pool/backup
```

...then allow our local user the ability to `zfs recv` into new datasets within it:

```bash
sudo zfs allow syncoid-receiver create,mount,receive main-pool/backup
```

## Pushing an initial backup

To start, we'll push the baseline of the dataset over SSH in order to establish a backup dataset on the receiving machine. Once SSH keys have been set up, we can run:

```bash
syncoid \
  --sendoptions=raw \
  --no-privilege-elevation \
  --no-sync-snap \
  --no-rollback \
  main-pool/test-dataset \
  syncoid-receiver@receiving-machine:main-pool/backup/test-dataset
```

Breaking down these options:

* `--sendoptions=raw` sends still-encrypted data. We don't need to load the decryption keys on the destination machine.
* `--no-privilege-elevation` prevents the use of `sudo`, as we've granted the necessary delegated permissions at both ends.
* `--no-sync-snap` avoids the creation of extra `syncoid`-specific snapshots; the existing snapshot from the preexisting `sanoid` policy will suffice.
* `--no-rollback` stops the backup dataset from being rolled back if snapshots go missing from the original dataset.

## Verifying the data has arrived

If we want, we can temporarily load the decryption key and mount the backup dataset to explore it:

```bash
zfs mount main-pool/backup/test-dataset
# cannot mount 'main-pool/backup/test-dataset': encryption key not loaded

sudo zfs load-key main-pool/backup/test-dataset
sudo zfs mount main-pool/backup/test-dataset

# [...] verify things are as expected

sudo zfs unmount main-pool/backup/test-dataset
sudo zfs unload-key main-pool/backup/test-dataset
```

## Pulling subsequent deltas

We'll pull data from the sending machine to the receiving machine as the `syncoid-receiver` user over SSH. With the appropriate SSH keys installed on the sending machine, we can incrementally pull any updates to our dataset with:

```bash
syncoid \
  --sendoptions=raw \
  --no-privilege-elevation \
  --no-sync-snap \
  --no-rollback \
  syncoid-sender@sending-machine:main-pool/test-dataset \
  main-pool/backup/test-dataset
```

## Pruning snapshots

The snapshot history from the source dataset will be preserved in the backup too, but we'll want to do similar housekeeping with `sanoid` to ensure they don't build up too much. We can even choose to keep an eye on the age of the latest snapshots present as a way of validating that our backups are continuing to take place.

Sample backup server `sanoid` configuration:

```ini
# /etc/sanoid/sanoid.conf
[main-pool/backup/test-dataset]
        frequently = 0
        hourly = 24
        daily = 5
        monthly = 1
        yearly = 0

        # Retain snapshots according to ^:
        autoprune = yes

        # Don't take any new snapshots:
        autosnap = no

        # If using nagios or similar, this has
        # sanoid --monitor-snapshots get worried
        # after a few days of missing dailies:
        monitor = yes
        daily_warn = 48h
        daily_crit = 60h
```

The final thing to do is to ensure that sanoid is run periodically to perform this maintenance:

```bash
sudo systemctl enable --now sanoid.timer
```
