---
layout: post
title: "Netgraph Crash with vtnet Workaround"
date: 2025-10-04
tags: FreeBSD Netgraph VNET Jail Network
---

# Netgraph Crash with vtnet Workaround

## Goal

Prevent a FreeBSD host from crashing when restarting Netgraph VNET jails, which are bridged to a vtnet network interface.

## Background

A few weeks ago I experimented with VNET jails and Netgraph, and noticed that one of my FreeBSD 14.3 VMs kept kernel panicking, everytime I stopped or restarted the VNET jails.
These crashes only occurred on this one FreeBSD VM, which uses the `vtnet` VirtIO Ethernet driver for the virtual NICs it gets from Proxmox VE.

## Workaround

After not being able to find a solution for the kernel panics myself, I searched on the Internet™️ and found this FreeBSD bug report:
https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=238326#c41

Some of the commenters recommended removing the VNET interface from the jails, **before** shutting them down, to prevend the Host from crashing.
Here are the changes I made to my VNET jail's `jail.conf` to stop my FreeBSD VM from crashing:

```
...
  vnet;
  vnet.interface += ng0_nsd04;
  exec.prestart += "jng bridge nsd04 vtnet1";
  exec.prestart += "ifconfig ng0_nsd04 ether 58:9c:fc:2a:1c:9b";
+ exec.prestop += "/usr/sbin/jexec ${name} /bin/sh /etc/rc.shutdown";
+ exec.prestop += "jng shutdown nsd04";
- exec.poststop += "jng shutdown nsd04";
}
```
> The lines with a `+` represend the added lines, the line with the `-` represents the remved line.

Now that the services inside the jail are stopped and the Netgraph VNET interface is detached **before** the jail is stopped, kernel panics on the FreeBSD host have ceased for now.
