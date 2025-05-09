---
layout: post
title: Retrospectively enabling ZFS encryption
tags: zfs
---

Suppose we have an existing ZFS dataset that was created without encryption, and we'd now like to encrypt it. This is something that can't be changed once a dataset has been created, so we have to get a little creative.

```bash
# Here's our existing dataset in the "main-pool" pool, without encryption:
sudo zfs create main-pool/test-dataset
```

## Establishing a baseline

So we're working on something consistent, we can either unmount the dataset, or take a snapshot. We'll do both; the former preventing further changes, and the latter so we can use the full `--replicate` option later on:

```bash
sudo zfs unmount main-pool/test-dataset
sudo zfs snapshot main-pool/test-dataset@encryption-time
```

## Key management

We then need a key, which we'll choose to store on the host filesystem (rather than require it to be provided interactively when mounting):

```bash
sudo openssl rand -hex -out /root/main-pool-test-dataset.key 32
```

It may be prudent to keep a copy of this somewhere safe, too.

## Replicating the dataset

We'll now use ZFS's `send`/`receive` to create a new dataset with encryption enabled:

```bash
sudo zfs send --replicate --verbose \
  main-pool/test-dataset@encryption-time \
  | sudo zfs receive \
  -s \
  -o encryption=on \
  -o keylocation=file:///root/main-pool-test-dataset.key \
  -o keyformat=hex \
  main-pool/encrypted-test-dataset
```

### Dealing with interruptions

Depending on the size of the source dataset and capabilities of the hardware, it may may take a long time to create the encrypted version of the dataset. This process may get interrupted; if so, the `-s` flag given to `zfs receive` will have caused a "progress" token to be saved against the new dataset. We can extract that, and resume the `zfs send` from that point:

```bash
# extract the resume token from the target dataset:
token=$(zfs get -o value -H receive_resume_token main-pool/encrypted-test-dataset)

# restart sending the source dataset from that point:
sudo zfs send --verbose -t $token \
  | sudo zfs receive -s main-pool/encrypted-test-dataset
```

### Swapping datasets

Once happy, we can rename the new encrypted version in place of the original:

```bash
sudo zfs rename main-pool/test-dataset main-pool/unencrypted-test-dataset
sudo zfs rename main-pool/encrypted-test-dataset main-pool/test-dataset
```

## Reclaiming space

If desired, we can then destroy the unencrypted version of the dataset to release the capacity back to the pool:

```bash
sudo zfs destroy -r main-pool/unencrypted-test-dataset
```

### Secure clean up

In order make this a secure erase, we can either:

* run `sudo zpool initialize -w main-pool` to zero over the freed space, or
* issue a secure TRIM erase instruction to the physical devices with `sudo zpool trim --secure -w main-pool` (hardware support allowing).

## Further reading

I've written separately on [automatically mounting encrypted dataset on boot]({% post_url 2025-03-08-mounting-zfs-encrypted-datasets-on-boot %}), which is possible here due to the use of a file as the dataset's `keylocation`.
