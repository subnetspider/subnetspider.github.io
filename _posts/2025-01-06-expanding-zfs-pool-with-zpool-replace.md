---
layout: post
title: "Expanding a ZFS pool with zpool-replace"
date: 2025-01-06
tags: FreeBSD OpenZFS Storage
---

# Expanding a ZFS pool with zpool-replace

### Goal

The goal of this post is to show how I replaced two old disks in one of my FreeBSD servers ZFS pool with larger disks to increase the amount of storage available.

---

### Background

For the last month I've been working on migrating my storage to FreeBSD, which has been managed by the FreeNAS / TrueNAS CORE since the beginning of 2019.
[A year ago]([url](https://x.com/subnetspider/status/1742303228438761484)) I decided to consolidate two of my servers, one running FreeBSD 14.0 and hosting most of my services in bastille jails, the other running TrueNAS CORE for my storage.
This was mainly to save a bit of power, as the FreeBSD jail server was way overkill, using only 8 of 64 GiB of RAM and less than 30 of 500 GB of NVMe storage, with the i3-10100 CPU mostly idle, 1-2% usage.

Unfortunately, just as I finished migrating my bastille jails on FreeBSD to iocage on TrueNAS CORE, users started questioning [the future of TrueNAS CORE]([url](https://www.truenas.com/community/threads/what-is-the-future-of-truenas-core.116049/)).
This was no surprise, as iXsystems' focus seemed to be shifting more and more to Linux with their new TrueNAS Scale appliance, and the FreeBSD base of TrueNAS CORE was nearing the end of its life.
While the community was promised that TrueNAS CORE wouldn't be deprecated, many saw the writing on the wall, which was later confirmed when iXsystems said that they have [no plans to upgrade to FreeBSD 14]([url](https://www.theregister.com/2024/03/18/truenas_abandons_freebsd/)).

As I have no plans to migrate away from FreeBSD and jails, which I've become very comfortable with over the last few years, to an unfamiliar system like Linux and Docker, I decided to leave TrueNAS behind and go full FreeBSD instead.
At the same time, I decided to build a second FreeBSD NAS for my parents to use as an off-site backup and to speed up their storage needs (mostly Windows clients), as they currently access my NAS via a site-to-site VPN, which is severely bottlenecked by their cable ISP's slow upload speeds.
Since I didn't have another pair of 18 TB disks available to easily migrate all the data from TrueNAS CORE, I dug out some old 4 TB disks that were actually from my first ever NAS, which ran OpenMediaVault before I switched to FreeNAS.

After migrating all of my storage and jails away from TrueNAS CORE to my new FreeBSD NAS, those 18 TB disks were now available again, so I could now replace the old 4 TB disks for my parents NAS.
As [this second FreeBSD server]([url](https://x.com/subnetspider/status/1555282371448111104)) which at one point was my TrueNAS CORE server, already had all of its four SATA ports used up (two SSDs as the boot pool, as well as two HDDs as the storage pool), I removed all the disks and installed them in another PC.
This way I can replace the old 4 TB disks one by one with the new 18 TB disks without having to remove them and degrade the ZFS pool, but still maintain the redundancy of the mirror until the very end.

---

### Commands

The commands I uses to accomplish the goal are the following:

- [camcontrol(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=camcontrol(8))) - CAM control program
- [doas(1)]([url](https://man.freebsd.org/cgi/man.cgi?query=doas(1))) - execute commands as another user
- [glabel(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=glabel(8))) - disk labelization control utility
- [gpart(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=gpart(8))) - control utility for the disk partitioning GEOM class
- [zfs-list(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=zfs-list(8))) - list properties of ZFS datasets
- [zpool-replace(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=zpool-replace(8))) - replace one device with another in ZFS storage pool
- [zpool-list(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=zpool-list(8))) - list information about ZFS storage pools
- [zpool-status(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=zpool-status(8))) - show detailed health status for ZFS storage pools
- [zpool-online(8)]([url](https://man.freebsd.org/cgi/man.cgi?query=zpool-online(8))) - take physical devices offline in ZFS storage pool

I have linked the corresponding man pages if you want to read more about them.

---

### The process

First, shut down your server, install the new disks, and then turn it back on again.
On some systems you can add the new disks while the server is running, YMMV.

Next, list all your available disks to see which is which:
```shell
admin@nas01:~ % doas camcontrol devlist
<TOSHIBA MG09ACA18TE 0104>         at scbus0 target 0 lun 0 (pass0,ada0)
<TOSHIBA MG09ACA18TE 0105>         at scbus1 target 0 lun 0 (pass1,ada1)
<Hitachi HDS724040ALE640 MJAOA3B0>  at scbus2 target 0 lun 0 (pass2,ada2)
<Hitachi HDS724040ALE640 MJAOA3B0>  at scbus3 target 0 lun 0 (pass3,ada3)
<INTEL SSDSC2BB080G4 D2012370>     at scbus4 target 0 lun 0 (pass4,ada4)
<INTEL SSDSC2BB080G4 D2012370>     at scbus5 target 0 lun 0 (pass5,ada5)
<AHCI SGPIO Enclosure 2.00 0001>   at scbus6 target 0 lun 0 (ses0,pass6)
```
The Toshiba disks `ada0` and `ada1` are the new 18 TB disks, the Hitagi `ada2` and `ada3` are the old disks.
The two Intel disks are where FreeBSD is installed on.

> ℹ️ Note
> Your discs may be labelled differently, such as `da0`, `da1` and so on.
> Also keep in mind that these device nodes are not permanent and can change.

After you indentified which disk is which, we can prepare the new disk for the ZFS pool.
If your new disks already have a file system, delete them with the following command:

> ⚠️ Warning
> Make sure and double check that you have selected the correct disk!
> Selecting the wrong disk will result in permanent data loss!

```shell
doas gpart destroy -F ada0 # This is the first new 18 TB disk.
```

```shell
doas gpart destroy -F ada1 # This is the second new 18 TB disk.
```

