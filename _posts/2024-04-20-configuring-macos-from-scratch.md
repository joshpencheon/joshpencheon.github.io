---
layout: post
title: Configuring macOS from scratch
tags: macOS dotfiles
---

After 13 years of using an original Retina MacBook Pro, I've finally made the move to an ARM-powered Mac. Having created a user account and signed into iCloud, what comes next when setting up a new Mac?


## Time Machine

Using [Migration Assistant](https://support.apple.com/en-gb/102613) to restore a macOS install is the quickest route to take, but if you prefer to start with a clean slate then setting up a new Time Machine target is a good thing to get done straight away. That's the route I took, given my previous install had over a decade of cruft.


## 1Password

This password manager is not available from the Mac App Store, but can be [downloaded directly](https://1password.com/downloads/mac/) instead. Get logged in to your account, and then the Safari extension can be installed from the App Store. If you use 1Password to hold your SSH keys as well, you'll also want to enable the SSH agent from `Settings -> Developer`.


## Homebrew

The unofficial package manager for macOS - it can be installed using the instructions [on the website](https://brew.sh). It will take care of installing the necessary Xcode development tools too.


## Dotfiles

I keep all my environment configuration under version control, and the repository contains [a quick install script](https://github.com/joshpencheon/dotfiles/blob/master/.dot/script/install) and then [a more comprehensive setup script](https://github.com/joshpencheon/dotfiles/blob/master/.dot/script/setup). The latter will install some essential packages, and configure Neovim.

## Default shell

In recent years, macOS has moved from Bash to Zsh. For the time being, I prefer to continue with Bash. It will have been installed as the dotfiles set up, but needs to be configured to be the default login shell. This process isn't scripted up.

First, add it to the allowlist:

```bash
sudo echo "$(brew --prefix)/bin/bash" >> /etc/shells
```

Then set it as the default for the user account:

```bash
sudo chsh -s "$(brew --prefix)/bin/bash" $USER
```

## Local configuration

There are some things that might be worth retrieving from the old computer, or a Time Machine backup from it:

* `~/.ssh` - this directory contains host configuration and any SSH keypair(s) not managed by 1Password.
* Any items in the local Keychain (that won't have been propagated via iCloud).
