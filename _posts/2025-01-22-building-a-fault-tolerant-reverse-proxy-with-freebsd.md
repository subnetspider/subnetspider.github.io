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

- [carp(4)](https://man.freebsd.org/cgi/man.cgi?query=carp) - Common Address Redundancy Protocol
- [doas(1)](https://man.freebsd.org/cgi/man.cgi?query=doas(1)) - execute commands as another user
- [pkg(7)](https://man.freebsd.org/cgi/man.cgi?query=pkg) - a utility for manipulating packages
- [haproxy(1)](https://man.freebsd.org/cgi/man.cgi?query=haproxy) - fast and reliable http	reverse	proxy and load balancer
- [ifconfig(8)](https://man.freebsd.org/cgi/man.cgi?query=ifconfig) - configure network interface parameters
- [kldload(8)](https://man.freebsd.org/cgi/man.cgi?query=kldload) - load a file into the kernel

## Getting started

To get started, you will need two FreeBSD hosts, which can be either physical hosts, virtual machines, or VNET jails.
I assume that you have already set up two FreeBSD systems, created a non-root user, and set up an ssh login and sudo/doas for elevated privileges.
I also assume that you have already set up DNS hostnames for all your hosts and services and know how to request TLS certificates from Let's Encrypt or ZeroSSL.
For commands that require root privileges I will use `doas`, you can also use `sudo` instead.

## Installation

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

To apply your new network settings, run the following command:
```shell
doas service netif restart && doas service routing restart
```

> ℹ️ Note
> This can disconnect you from your SSH session and possibly lock you out of your FreeBSD host, so be careful and check your `/etc/rc.conf` for typos beforehand.

Use `ifconfig` to check which host is MASTER and which is BACKUP:

**Host 1**
```
admin@haproxy01.example.net:~ % ifconfig vnet0
vnet0: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        options=8<VLAN_MTU>
        ether 02:3a:f8:fd:ed:0b
        inet 10.1.60.201 netmask 0xffffff00 broadcast 10.1.60.255
        inet 10.1.60.200 netmask 0xffffffff broadcast 10.1.60.200 vhid 1
        inet6 fe80::3a:f8ff:fefd:ed0b%vnet0 prefixlen 64 scopeid 0x38
        inet6 2003:abcd:ef12:3460:d:22ff:fe50:3d0b prefixlen 64
        inet6 fd5e:d1c2:c9de:60:d:22ff:fe50:3d0b prefixlen 64
        inet6 fd5e:d1c2:c9de:60::200 prefixlen 128 vhid 1
        groups: epair
        carp: BACKUP vhid 1 advbase 1 advskew 0
              peer 224.0.0.18 peer6 ff02::12
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
```

**Host 2**
```
admin@haproxy02.example.net:~ % ifconfig vnet0
vnet0: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        options=8<VLAN_MTU>
        ether 06:20:99:eb:29:69
        inet 10.1.60.202 netmask 0xffffff00 broadcast 10.1.60.255
        inet 10.1.60.200 netmask 0xffffffff broadcast 10.1.60.200 vhid 1
        inet6 fe80::420:99ff:feeb:2969%vnet0 prefixlen 64 scopeid 0x1e
        inet6 2003:abcd:ef12:3460:420:99ff:feeb:2969 prefixlen 64
        inet6 fd5e:d1c2:c9de:60:420:99ff:feeb:2969 prefixlen 64
        inet6 fd5e:d1c2:c9de:60::200 prefixlen 128 vhid 1
        groups: epair
        carp: MASTER vhid 1 advbase 1 advskew 100
              peer 224.0.0.18 peer6 ff02::12
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
```

Make sure you point the right DNS records (A / AAAA / CNAME) of your services to your CARP VIPs and check that clients can resolve them.
You can learn more about CARP in the FreeBSD Handbook.: [Common Address Redundancy Protocol (CARP)](https://docs.freebsd.org/en/books/handbook/advanced-networking/#carp)

### HAProxy

Install HAProxy with the following command:

```shell
doas pkg install -y haproxy
```

Next, create your HAProxy configurtion at `/usr/local/etc/haproxy.conf`:
``` shell
# generated 2025-01-22, Mozilla Guideline v5.7, HAProxy 3.0, OpenSSL 3.4.0, intermediate config, no HSTS
# https://ssl-config.mozilla.org/#server=haproxy&version=3.0&config=intermediate&openssl=3.4.0&hsts=false&guideline=5.7
global
    ssl-default-bind-curves X25519:prime256v1:secp384r1
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options prefer-client-ciphers ssl-min-ver TLSv1.2 no-tls-tickets

    ssl-default-server-curves X25519:prime256v1:secp384r1
    ssl-default-server-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
    ssl-default-server-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-server-options ssl-min-ver TLSv1.2 no-tls-tickets

  daemon
  maxconn 1024

defaults
  stats enable
  stats uri /report
  stats refresh 30s
  stats auth haproxy:very-secure-password
  log global 
  timeout connect 30s
  timeout client 30s
  timeout server 30s
  retries 3
  timeout tunnel        3600s
  timeout http-keep-alive  5s
  timeout http-request    15s
  timeout queue           30s
  timeout tarpit          60s
  default-server inter 3s rise 2 fall 3

frontend HTTP_frontend
  mode http
  bind ipv6@:443 ssl crt /usr/local/etc/ssl/example.net.pem alpn h2,http/1.1
  bind ipv6@:80
  bind :443 ssl crt /usr/local/etc/ssl/example.net.pem alpn h2,http/1.1
  bind :80
  http-request redirect scheme https unless { ssl_fc }
  option forwardfor if-none

  use_backend ADGUARD01 if  { hdr(host) -i adguard01.example.net }
  use_backend ADGUARD02 if  { hdr(host) -i adguard02.example.net }
  use_backend GITEA if      { hdr(host) -i gitea.example.net }
  use_backend HEIMDALL if   { hdr(host) -i heimdall.example.net }
  use_backend MINEOS if     { hdr(host) -i mineos.example.net }
  use_backend NETBOX if     { hdr(host) -i netbox.example.net }
  use_backend PROXMOX if    { hdr(host) -i proxmox.example.net }
  use_backend SYNCTHING if  { hdr(host) -i syncthing.example.net }
  use_backend TORRENT if    { hdr(host) -i transmission.example.net }
  use_backend UNIFI if      { hdr(host) -i unifi.example.net }
  use_backend ZABBIX if     { hdr(host) -i zabbix.example.net }

  timeout client 15m

backend ADGUARD01
  description adguard01
  mode http
  server adguard01 [fd5e:d1c2:c9de:60::13]:443 ssl verify none check
  stick-table type ip size 50k expire 30m  
  stick on src

backend ADGUARD02
  description adguard02
  mode http 
  server adguard02 [fd5e:d1c2:c9de:60::14]:80 check
  stick-table type ip size 50k expire 30m  
  stick on src

backend GITEA
  description gitea
  mode http
  server gitea [fd5e:d1c2:c9de:70::5]:3000 check
  stick-table type ip size 50k expire 30m  
  stick on src

backend HEIMDALL
  description heimdall
  mode http
  server heimdall [fd5e:d1c2:c9de:70::6]:443 ssl verify none check
  stick-table type ip size 50k expire 30m  
  stick on src

backend MINEOS
  description mineos
  mode http
  server mineos [fd5e:d1c2:c9de:60::12]:443 ssl verify none
  stick-table type ip size 50k expire 30m  
  stick on src

backend NETBOX
  description NetBox
  mode http
  server netbox [fd5e:d1c2:c9de:70::4]:443 ssl verify none check 
  stick-table type ip size 50k expire 30m  
  stick on src

backend PROXMOX
  description Proxmox VE
  mode http
  server proxmox [fd5e:d1c2:c9de:10::11]:8006 ssl verify none check
  stick-table type ip size 50k expire 30m  
  stick on src

backend SYNCTHING
  description syncthing
  mode http
  server syncthing [fd5e:d1c2:c9de:70::9]:80 check
  stick-table type ip size 50k expire 30m  
  stick on src

backend TORRENT
  description transmission
  mode http
  server torrent [fd5e:d1c2:c9de:60::10]:9091 check
  stick-table type ip size 50k expire 30m  
  stick on src

backend UNIFI
  description unifi controller
  mode http
  server unifi 10.1.12.2:8443 ssl verify none check # IPv4-only :(
  stick-table type ip size 50k expire 30m  
  stick on src

backend ZABBIX
  description Zabbix
  mode http
  server zabbix [fd5e:d1c2:c9de:11::2]:80 check
  stick-table type ip size 50k expire 30m  
  stick on src
```

To check your HAProxy configuration for errors, run the following command:
```shell
doas haproxy -f /usr/local/etc/haproxy.conf -c
```

**⚠️ Disclaimer**
Most of these settings are the result of reading many tutorials, man pages and trial and error.
I am not a HAProxy expert, so do a little research before you decide to expose this setup to the world wild web.
Here are some of the resources I haves used in the past:
- https://www.youtube.com/watch?v=CxamHNc3U4A
- https://www.haproxy.org/download/3.0/doc/configuration.txt
- https://andyleonard.com/2011/02/01/haproxy-and-keepalived-example-configuration/
- https://sleeplessbeastie.eu/2018/03/08/how-to-define-basic-authentication-on-haproxy/

If you want to run HAProxy on OPNsense, I highly recommend this HowTo:
- https://forum.opnsense.org/index.php?topic=23339.0

To start HAProxy, run the following command:

```shell
doas service haproxy enable && doas service haproxy start
```

Make sure you put a valid TLS certificate in `/usr/local/etc/ssl/example.net.pem' and secure it so that no unprivileged users can read the private key.
I currently request my TLS certificates with [acme.sh](https://github.com/acmesh-official/acme.sh) on a dedicated host, which copies them via SSH to my reverse proxies.

> ℹ️ Note
> If you are experiencing SSL handshake errors with HAProxy, check your TLS settings as these are used to communicate with clients **and** backend servers.

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

> ℹ️ Note
> If you don't allow `carp` in your FreeBSD host's firewall, both hosts will claim the MASTER role, which breaks things in interesting ways.

## Conclusion

You have now set up two very lightweight, high-performance, fault-tolerant reverse proxies on FreeBSD, with HAProxy as the only dependency.
This setup requires almost no maintenance other than updating HAProxy and FreeBSD, and requesting new TLS certificates from time to time.
If one of the two reverse proxies goes down, it is unlikely that you will notice it, so setting up some sort of monitoring is recommended, but not required.
If you are comfortable with managing your reverse proxy via SSH and not using a fancy WebUI, I recommend giving this a try :)
