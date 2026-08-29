---
layout: post
title: Raising niri's default log level
tags: niri
---

[Niri](https://github.com/niri-wm/niri) releases are built with a default log level of `debug`, which can spoil an otherwise silent boot. To raise this to `error`, we can first create a wrapper script to set an environment variable that niri will use to configure the `tracing_subscriber` crate appropriately:

```bash
#!/bin/sh
# /usr/local/bin/niri-session

RUST_LOG=niri=error niri --session
```

Once this is made executable, we can specify it in the `greetd` configuration (other greeters are available), along with an optional `user` for auto-login:

```bash
# /etc/greetd/config.toml

[default_session]
command = "/usr/local/bin/niri-session"
user = josh
```

Happy silent booting!
