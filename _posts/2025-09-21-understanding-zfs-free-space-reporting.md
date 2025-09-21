---
layout: post
title: Understanding ZFS free space reporting
tags: zfs
---

Recently, I noted `btop` was showing some odd stats for a ZFS dataset; it was showing as 100% free, but with an unexpectedly small total capacity:

```
├─main-pool──────────────65,6─GiB─┤
│ IO% ........................... │
│ Used:  0% ■             128 KiB │
│ Free:100% ■■■■■■■■■■■■ 65,6 GiB │
├─────────────────────────────────┤
```

Checking with ZFS, this total capacity actually appeared to be the remaining space for the dataset:

```
$ zfs list main-pool
NAME        USED  AVAIL  REFER  MOUNTPOINT
main-pool   365G  65.7G   128K  /main-pool
```

At first, I guessed there might be some sort of off-by-one error in btop's reading of the ZFS statistics, so I had a poke around [in the source](https://github.com/aristocratos/btop/blob/v1.4.4/src/linux/btop_collect.cpp#L2032-L2049). While IO stats are read from the ZFS-provided interface `/proc/spl/kstat/zfs/main-pool/objset-*`, capacity stats are retrieved from a `statvfs` system call. 

## Reproducing the numbers

So in fact, I was able to reproduce `btop`'s numbers, both via a python script:

```python
>>> from os import statvfs
>>> stats = statvfs("/main-pool")
>>> "{:.1f}GB".format(stats.f_blocks * stats.f_bsize / (2**30))
'65.7GB'
```

...and via `df`:

```
$ df -h /main-pool
Filesystem      Size  Used Avail Use% Mounted on
main-pool        66G  128K   66G   1% /main-pool
```

## The penny drops

I then realised that I had nested datasets that weren't mounted, but were still consuming pool capacity. If I temporarily mounted them, things began to make sense:

```
├─main-pool─────────────────────────────65,6─GiB─┤
│ IO% .......................................... │
│ Used:  0% ■                            128 KiB │
│ Free:100% ■■■■■■■■■■■■■■■■■■■■■■■■■■■ 65,6 GiB │
│                                                │
├─time-machine───────────────────────────418─GiB─┤
│ IO% .......................................... │
│ Used: 84% ■■■■■■■■■■■■■■■■■■■■■■       353 GiB │
│ Free: 16% ■■■■■■■■■■■■■■■■■■■■■■■■■■■ 65,6 GiB │
│                                                │
├────────────────────────────────────────────────┤
```

So it turns out ZFS exposes each mounted dataset individually, and reports the pool's available space as free against each. Mystery solved!
