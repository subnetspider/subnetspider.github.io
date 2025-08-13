---
layout: post
title: "Trying out Netgraph VNET Jails with ngbuddy.md"
date: 2025-08-13
tags: FreeBSD VNET Jail Netgraph
---

After reading the latest FreeBSD Journal "[Netgraph for the Rest of Us]([url](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/netgraph-for-the-rest-of-us/))" by Daniel J. Bell and discovering the tool `ngbuddy(8)`, I wanted to try it myself.
I actually wanted to try Netgraph for a while now, but I could never figure out how to configure it manually, as it is rather complicated.
However, this seemed like the perfect opportunity, so I installed FreeBSD 14.3-RELEASE in a fresh VM on my second Proxmox VE and created a couple of Bastille VNET jails.
I then modified these to use Netgraph interfaces:

Installing the necessary packages:
```shell
doas pkg install -y bastille ngbuddy
```

Setting up ngbuddy:
```shell
doas service ngbuddy enable
```
```
ngbuddy_enable:  -> YES
Adding default public and private bridges.
ngbuddy_public_if:  -> bridge0
ngbuddy_private_if:  -> nghost0
```

Starting ngbuddy:
```shell
doas service ngbuddy start
```
```
Starting ngbuddy.
Created 3 links.
```

Running bastille setup:
```shell
doas bastille setup
```

Disabling pf, as I don't need NAT with bastille0 right now:
```shell
doas service pf onedisable
```

Bootstrapping and patching FreeBSD 14.3:
```shell
doas bastille bootstrap 14.3-RELEASE update
```

Creating a VNET jail:
```shell
doas bastille create -V jail05 14.3-RELEASE DHCP vtnet0
```

Stopping the VNET jail:
```shell
doas bastille stop jail05
```

Edit the VNEt jail:
```shell
doas bastille edit jail05
```
```
jail05 {
  $if_name = "$name";
  $bridge = "public";
  enforce_statfs = 2;
  devfs_ruleset = 13;
  exec.clean;
  exec.consolelog = /var/log/bastille/jail05_console.log;
  exec.prestart = "service ngbuddy jail $if_name $bridge";
  exec.start = '/bin/sh /etc/rc';
  exec.stop = '/bin/sh /etc/rc.shutdown';
  exec.prestop = "service ngbuddy unjail $if_name $name";
  host.hostname = jail05;
  mount.devfs;
  mount.fstab = /usr/local/bastille/jails/jail05/fstab;
  path = /usr/local/bastille/jails/jail05/root;
  securelevel = 2;
  osrelease = 14.3-RELEASE;
  vnet;
  vnet.interface = "$if_name";
}
```

Edit the VNET jail's rc.conf:
```shell
doas bastille edit jail05 root/etc/rc.conf
```
```
ifconfig_jail05_name="vnet0"
ifconfig_vnet0="SYNCDHCP"
syslogd_flags="-ss"
sendmail_enable="NO"
sendmail_submit_enable="NO"
sendmail_outbound_enable="NO"
sendmail_msp_queue_enable="NO"
cron_flags="-J 60"
```

Start the netgraph VNET jail:
```shell
doas bastille start jail05
```
```
[jail05]:
jail05: created
```

Testing if DHCP is working:
```shell
doas bastille cmd jail05 ifconfig vnet0
```
```
[jail05]:
vnet0: flags=1008843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        options=28<VLAN_MTU,JUMBO_MTU>
        ether 58:9c:fc:10:ff:dd
        inet 10.2.70.131 netmask 0xffffff00 broadcast 10.2.70.255
        media: Ethernet autoselect (1000baseT <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

One nice side effect of using netgraph instead of epair and bridge interfaces is that the output of `ifconfig` on the FreeBSD host isn't nearly as cluttered:
```shell
vtnet0: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        options=4800bb<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,LINKSTATE,TXCSUM_IPV6>
        ether bc:24:11:5c:4d:b5
        inet 10.2.70.107 netmask 0xffffff00 broadcast 10.2.70.255
        media: Ethernet autoselect (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
lo0: flags=1008049<UP,LOOPBACK,RUNNING,MULTICAST,LOWER_UP> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        inet 127.0.0.1 netmask 0xff000000
        inet6 ::1 prefixlen 128
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
```

For comparison, here is the output of `ifconfig` on another FreeBSD jail host which uses epair and a bridge:
```
vtnet0: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        options=c00b9<RXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,VLAN_HWTSO,LINKSTATE>
        ether bc:24:11:44:8a:6d
        media: Ethernet autoselect (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
lo0: flags=1008049<UP,LOOPBACK,RUNNING,MULTICAST,LOWER_UP> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        inet 127.0.0.1 netmask 0xff000000
        inet6 ::1 prefixlen 128
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
bridge0: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        options=0
        ether 58:9c:fc:10:ff:db
        inet 10.2.70.109 netmask 0xffffff00 broadcast 10.2.70.255
        id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
        maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
        root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
        member: e0a_jail04 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
                ifmaxaddr 0 port 14 priority 128 path cost 2000
        member: e0a_jail03 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
                ifmaxaddr 0 port 11 priority 128 path cost 2000
        member: e0a_jail02 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
                ifmaxaddr 0 port 8 priority 128 path cost 2000
        member: e0a_jail01 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
                ifmaxaddr 0 port 5 priority 128 path cost 2000
        member: vtnet0 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
                ifmaxaddr 0 port 1 priority 128 path cost 2000
        groups: bridge
        nd6 options=9<PERFORMNUD,IFDISABLED>
bastille0: flags=8008<LOOPBACK,MULTICAST> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
e0a_jail01: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        description: vnet0 host interface for Bastille jail jail01
        options=8<VLAN_MTU>
        ether 02:d6:ec:ad:52:0a
        groups: epair
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
e0a_jail02: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        description: vnet0 host interface for Bastille jail jail02
        options=8<VLAN_MTU>
        ether 02:10:ec:6c:e7:0a
        groups: epair
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
e0a_jail03: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        description: vnet0 host interface for Bastille jail jail03
        options=8<VLAN_MTU>
        ether 02:21:90:a4:5f:0a
        groups: epair
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
e0a_jail04: flags=1008943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        description: vnet0 host interface for Bastille jail jail04
        options=8<VLAN_MTU>
        ether 02:6b:4b:c6:ea:0a
        groups: epair
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

---

## Commands

The commands I have used:

- [ngbuddy(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=ngbuddy) - Simplified netgraph(4)	manager	for jail(8) and	bhyve(8)
- [bastille(8)](https://man.freebsd.org/cgi/man.cgi?query=bastille) - Bastille is an open-source system for automating deployment and management of containerized	applications on	FreeBSD.
- [doas(1)](https://man.freebsd.org/cgi/man.cgi?query=doas) - execute commands as another user




