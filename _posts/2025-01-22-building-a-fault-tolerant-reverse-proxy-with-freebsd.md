---
layout: post
title: "Building a fault-tolerant reverse proxy with FreeBSD"
date: 2025-01-22
tags: FreeBSD HAProxy CARP Reverse_Proxy
---

# Building a fault-tolerant reverse proxy with FreeBSD

### Goal

The goal of this blog post is to set up HAProxy as a reverse proxy on two FreeBSD hosts with CARP for fault tolerance, so that you can still access services if one of the hosts fails.

## Background

I have been running HAProxy on OPNsense as my reverse proxy for years, but after switching my firewall at home to Sophos XG, I needed to find an alternative.
Sophos XG has a built-in WAF (web application firewall) that can be set up as a reverse proxy with TLS termination, even with the free home license.
However, this wasn't an option for me as the WAF on Sophos XG does not support IPv6 as of yet, which is a mandatory requirement for me.
Although Sophos XG introduced the ability to request free TLS certificates from Let's Encrypt with SFOS 21, it does not support requesting wildcard certificates.

As I already have some familiarity with using HAProxy as a reverse proxy to get rid of invalid TLS certificate warnings, this is what I chose.
I have also tried out OpenBSD's `[relayd](https://man.openbsd.org/relayd.8)`, which I liked, but wasn't able to get it running the way I wanted.

## Commands



## Getting started

To get started, you will need two FreeBSD hosts, which can be either physical hosts, virtual machines, or VNET jails.
I assume that you have already set up two FreeBSD systems, created a non-root user, and set up an ssh login and sudo/doas for elevated privileges.
For commands that require root privileges I will use `doas`, you can also use `sudo` instead.

### CARP

To automatiacally load the `carp.ko` kernel module at boot, add the following line to the end of `/boot/loader.conf`:
```shell
carp_load="YES"
```

Next, run the following command to load the `carp.ko` kernel module:
```
doas kldload carp
```

> ℹ️ Note
> If you want to set up CARP in a VNET jail, you will need to edit `/boot/loader.conf` of the FreeBSD host.
> You will also need to load the `carp.ko` kernel module from the host, if you don't want to reboot it.

## Conclusion


