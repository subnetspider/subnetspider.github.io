---
layout: post
title: "Trying out Netgraph VNET Jails with ngbuddy"
date: 2025-08-13
tags: FreeBSD VNET Jail Netgraph
---

## Context

After reading the latest FreeBSD Journal "[Netgraph for the Rest of Us](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/netgraph-for-the-rest-of-us/)" by Daniel J. Bell, I discovering the tool `ngbuddy(8)`, and immediately wanted to try it out myself.
I actually wanted to experiment with Netgraph for a while now, but I could never figure out how to configure it manually, as it is rather complicated.
However, this seemed like the perfect opportunity, so I installed FreeBSD 14.3-RELEASE in a fresh VM on my second Proxmox VE and created a couple of Bastille VNET jails, which I modified to use Netgraph.

---

## Steps

Here are the steps I've used to get everything up and running, minus setting up the FreeBSD hosts itself (pkg repos, doas, ssh, etc.):

**Installing the necessary packages:**
```shell
doas pkg install -y bastille ngbuddy
```

**Setting up ngbuddy:**
```shell
doas service ngbuddy enable
```
```
ngbuddy_enable:  -> YES
Adding default public and private bridges.
ngbuddy_public_if:  -> bridge0
ngbuddy_private_if:  -> nghost0
```

**Starting ngbuddy:**
```shell
doas service ngbuddy start
```
```
Starting ngbuddy.
Created 3 links.
```

**Running bastille setup:**
```shell
doas bastille setup
```

**Disabling pf, as I don't need NAT with bastille0 right now:**
```shell
doas service pf onedisable
```

**Bootstrapping and patching FreeBSD 14.3:**
```shell
doas bastille bootstrap 14.3-RELEASE update
```

**Creating a VNET jail:**
```shell
doas bastille create -V jail05 14.3-RELEASE DHCP vtnet0
```

**Stopping the VNET jail:**
```shell
doas bastille stop jail05
```

**Editing the VNEt jail's jail.conf:**
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

**Editing the VNET jail's rc.conf:**
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

**Starting the netgraph VNET jail:**
```shell
doas bastille start jail05
```
```
[jail05]:
jail05: created
```

**Checking if DHCP is working:**
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

---

## Conclusion

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

To see what kind of speeds I could achieve, I ran `iperf3` on two of the Netgraph VNET jails, and got ~10 Gbit/s of traffic as a result.
In comparison, the same VM with epair/bridge VNET jails only reached ~6 Gbit/s, but with significantly lower CPU load (~45% vs ~99%).
Since benchmarks with virtual Network interfaces such as `vtnet` often come with weird performance issues, take this result with a grain of salt.

Another thing that worked well is CARP, which still causes problems with FreeBSD jails whose epair interfaces are connected to a bridge.
For example, here is a snippet of `/var/log/messages` of one of my FreeBSD NAS servers:
```
Aug 13 20:06:16 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:01 vlan 0 moved from e0a_adguard03 to lagg0.60
Aug 13 20:06:16 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:02 vlan 0 moved from e0a_adguard03 to lagg0.60
Aug 13 20:06:16 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:03 vlan 0 moved from e0a_unbound03 to lagg0.60
Aug 13 20:06:16 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:04 vlan 0 moved from e0a_unbound03 to lagg0.60
Aug 13 20:06:17 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:02 vlan 0 moved from lagg0.60 to e0a_adguard03
Aug 13 20:06:18 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:03 vlan 0 moved from e0a_unbound03 to lagg0.60
Aug 13 20:06:18 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:01 vlan 0 moved from e0a_adguard03 to lagg0.60
Aug 13 20:06:18 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:02 vlan 0 moved from e0a_adguard03 to lagg0.60
Aug 13 20:06:18 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:04 vlan 0 moved from e0a_unbound03 to lagg0.60
Aug 13 20:06:18 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:01 vlan 0 moved from lagg0.60 to e0a_adguard03
Aug 13 20:06:19 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:01 vlan 0 moved from e0a_adguard03 to lagg0.60
Aug 13 20:06:19 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:02 vlan 0 moved from e0a_adguard03 to lagg0.60
Aug 13 20:06:19 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:03 vlan 0 moved from e0a_unbound03 to lagg0.60
Aug 13 20:06:19 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:04 vlan 0 moved from e0a_unbound03 to lagg0.60
Aug 13 20:06:20 freebsd-nas-2 kernel: bridge60: mac address 00:00:5e:00:01:01 vlan 0 moved from lagg0.60 to e0a_adguard03
```
> **Update** (2025-10-04)
> This problem was caused by missing firewall rules on the jails, which caused both jails's CARP VIP to become MASTER.
> After allowing CARP traffic through, the log messaged stopped and failover works flawlessly.

With Netgraph, there were no such log entries, and `ping` to the CARP VIP configure on two of the Netgraph VNET jails worked without a hitch.

As you can generate Graphs to visualize Netgraph deployments, I went ahead and made one myself:
```shell
doas ngctl dot
```

Since the command `ngctl dot | dot -T png -o netgraph.png` I found on the Klara Systems article "[Using Netgraph for FreeBSD’s Bhyve Networking](https://klarasystems.com/articles/using-netgraph-for-freebsds-bhyve-networking/)" only threw me the error "dot: graph is too large for cairo-renderer bitmaps. Scaling by 0.00850981 to fit", I used a online version of `graphviz` and converted the SVG file to a PNG:
<img alt="ngctl dot output rendered with graphviz" src="https://www.subnetspider.com/assets/img/graphviz.png" />

Some other useful commands are:

```shell
doas ngctl list
```
```
There are 8 total nodes:
  Name: jail02          Type: eiface          ID: 00000080   Num hooks: 1
  Name: vtnet0          Type: ether           ID: 00000001   Num hooks: 2
  Name: jail04          Type: eiface          ID: 00000062   Num hooks: 1
  Name: public          Type: bridge          ID: 00000006   Num hooks: 7
  Name: ngctl82484      Type: socket          ID: 00000089   Num hooks: 0
  Name: jail05          Type: eiface          ID: 0000006a   Num hooks: 1
  Name: jail03          Type: eiface          ID: 00000015   Num hooks: 1
  Name: jail01          Type: eiface          ID: 00000079   Num hooks: 1
```

```shell
doas service ngbuddy status
```
```
public
  jail02: RX 942B, TX 64.44 KB
  jail01: RX 56.41 KB, TX 38.32 KB
  jail05: RX 384B, TX 104.07 KB
  jail04: RX 42B, TX 117.36 KB
  jail03: RX 4.88 GB, TX 9.65 GB
  vtnet0 (upper): RX 2.33 MB, TX 2.37 MB
  vtnet0 (lower): RX 2.28 MB, TX 2.39 MB
```

In conclusion, thanks to the ngbuddy ([Netgraph Buddy](https://github.com/bellhyve/ngbuddy)) package, it was very easy to get started with Netgraph.
Compared to running VNET jails with `bridge` and `epair`, I found this approach much cleaner, so I may end up using this in future instead.
I am looking forward to trying it out with FreeBSD installed on physical hardware to see what performance I can achieve.

---

## Commands

The commands I have used:

- [ngbuddy(8)](https://man.freebsd.org/cgi/man.cgi?query=ngbuddy) - Simplified netgraph(4)	manager	for jail(8) and	bhyve(8)
- [bastille(8)](https://man.freebsd.org/cgi/man.cgi?query=bastille) - Bastille is an open-source system for automating deployment and management of containerized	applications on	FreeBSD.
- [doas(1)](https://man.freebsd.org/cgi/man.cgi?query=doas) - execute commands as another user
- [ngctl(8)](https://man.freebsd.org/cgi/man.cgi?query=ngctl) - netgraph control utility
