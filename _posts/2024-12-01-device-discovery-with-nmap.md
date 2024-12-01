---
layout: post
title: Device discovery with nmap
tags: networking
---

## Some background

I have some managed Netgear switches on my network that should by default acquire an IP address via DHCP. In order to find them, Netgear provide some discovery tools to locate their switches; depending on the firmware of the given switch, this can place either [via NSDP](https://en.wikipedia.org/wiki/Netgear_Switch_Discovery_Protocol) or [via UPnP](https://en.wikipedia.org/wiki/Universal_Plug_and_Play). However, this all means installation of some (Rosetta-requiring) additional software. Ideally, we'd avoid this.

While another place to get the IP address for them would be to check the lease list on the DHCP server, if this means your ancient router (as it does for me) you may struggle!

## Enter nmap

We can use `nmap` to enumerate IP addresses within a range, and detect hosts and/or services. To do a "ping scan" against a block of IPs:

```bash
nmap -sn 192.168.1.0/24
```

It might also be helpful to scan for specific ports being open. For example, the managed switch I'm searching for has a web interface, so should have TCP ports 80/443 open:

```bash
nmap -p 80 --open 192.168.1.0/24
```

...and in the output, we find our Netgear switch, along with its IP address:

```
Nmap scan report for GS308EPP (192.168.1.101)
Host is up (0.010s latency).

PORT   STATE SERVICE
80/tcp open  http
```

This is just scratching the surface of what `nmap` is capable of.
