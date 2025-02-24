---
layout: post
title: Binding services together with systemd
tags: systemd zfs
---

I've got an SMB share that's serving files stored in ZFS. I don't want the samba server to start until the ZFS dataset is mounted, and nor do I want the server to continue running should the dataset get unmounted; this might risk data being written without the desired ZFS encryption in place.

Luckily, these scenarios can be prevented by associating the SMB systemd `.service` unit with the `.mount` unit that's created as a result of the ZFS setup.

We can use `sudo systemctl edit smbd.service` and then specify via an override that:

* the SMB daemon should not be started until the `time-machine` dataset is mounted (using `After=`), and
* should the dataset be unmounted, the daemon should also be stopped (using `BindsTo=`)

```conf
[Unit]
# Serve only when the ZFS mount is available
BindsTo=srv-time\x2dmachine.mount
After=srv-time\x2dmachine.mount
```

## Auto-restart

If we wanted to, we could go a step further and use `sudo systemctl edit srv-time\\x2dmachine.mount` to add a `Wants=smbd.service` declaration back in the other direction; this would attempt to automatically start the SMB daemon should the `time-machine` dataset get (re)mounted.
