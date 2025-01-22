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

For commands that require root privileges I will use `doas`, you can also use `sudo` instead.

## Getting started

To get started, you will need two FreeBSD hosts, which can be either physical hosts, virtual machines, or VNET jails.
I assume that you have already set up two FreeBSD systems, created a non-root user, and set up an ssh login and sudo/doas for elevated privileges.
I also assume that you have already set up DNS hostnames for all your hosts and services and know how to request TLS certificates from Let's Encrypt or ZeroSSL.

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

#### Networking

Next, you will need to set up CARP VIPs (virtual IP addresses) in `/etc/rc.conf`:

**Host 1**
```
hostname="haproxy01.example.net"

# IPv4
ifconfig_vnet0="inet 10.1.60.201/24"
defaultrouter="10.1.60.1"

# IPv6
ifconfig_vnet0_ipv6="inet6 2003:abcd:ef12:3460:d:22ff:fe50:3d0b/64"
ifconfig_vnet0_alias0="inet6 fd5e:d1c2:c9de:60:d:22ff:fe50:3d0b/64"
ipv6_defaultrouter="2003:abcd:ef12:3460::1"

# CARP VIPs
ifconfig_vnet0_alias1="inet vhid 1 advskew 0 pass very-secure-password alias 10.1.60.200/32"
ifconfig_vnet0_alias2="inet6 vhid 1 advskew 0 pass very-secure-password alias fd5e:d1c2:c9de:60::200/128"
```

**Host 2**
```
hostname="haproxy02.example.net"

# IPv4
ifconfig_vnet0="inet 10.1.60.202/24"
defaultrouter="10.1.60.1"

# IPv6
ifconfig_vnet0_ipv6="inet6 2003:abcd:ef12:3460:420:99ff:feeb:2969/64"
ifconfig_vnet0_alias0="inet6 fd5e:d1c2:c9de:60:420:99ff:feeb:2969/64"
ipv6_defaultrouter="2003:abcd:ef12:3460::1"

# CARP VIPs
ifconfig_vnet0_alias1="inet vhid 1 advskew 100 pass very-secure-password alias 10.1.60.200/32"
ifconfig_vnet0_alias2="inet6 vhid 1 advskew 1ß0 pass very-secure-password alias fd5e:d1c2:c9de:60::200/128"
```

You can learn more about CARP in the FreeBSD Handbook.: [Common Address Redundancy Protocol (CARP)](https://docs.freebsd.org/en/books/handbook/advanced-networking/#carp)

### HAProxy




### Firewall

If you run a firewall like PF, make sure to add HTTP, HTTPS and CARP to your `/etc/pf.conf`:

```shell
# Macros
ext_if = vnet0
icmp_types = "{ unreach, squench, echoreq, timex, paramprob }"
icmp6_types = "{ unreach, toobig, timex, paramprob, echoreq, neighbradv, neighbrsol, routeradv, routersol }"

# Default Rules
set block-policy drop
set skip on lo
block in log all
pass out all

# SSH
pass in on $ext_if inet6 proto tcp from 2003:abcd:ef12:3420::/60 to any port = ssh
pass in on $ext_if inet6 proto tcp from 2003:abcd:ef12:34a1::/64 to any port = ssh
pass in on $ext_if inet6 proto tcp from fd5e:d1c2:c9de:20::/64 to any port = ssh
pass in on $ext_if inet6 proto tcp from fd5f:a747:658d:20::/64 to any port = ssh

# Ping and IPv6 requirements
pass in on $ext_if inet proto icmp icmp-type $icmp_types
pass in on $ext_if inet6 proto icmp6 icmp6-type $icmp6_types

# CARP
pass quick on $ext_if proto carp

# HAProxy
pass in proto tcp to port { http, https }
```

## Conclusion


