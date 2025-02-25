---
layout: post
title: "Expanding a ZFS pool with zpool-replace"
date: 2025-01-06
tags: FreeBSD OpenZFS Storage
---

# Expanding a ZFS pool with zpool-replace

### Goal

The goal of this post is to show how I replaced two old disks in one of my FreeBSD servers ZFS pool with larger disks to increase the storage capacity.

## Background

For the last month I've been working on migrating my storage to FreeBSD, which has been managed by the FreeNAS / TrueNAS CORE since the beginning of 2019.
[A year ago](https://x.com/subnetspider/status/1742303228438761484) I decided to consolidate two of my servers, one running FreeBSD 14.0 and hosting most of my services in bastille jails, the other running TrueNAS CORE for my storage.
This was mainly to save a bit of power, as the FreeBSD jail server was way overkill, using only 8 of 64 GiB of RAM and less than 30 of 500 GB of NVMe storage, with the i3-10100 CPU mostly idle, 1-2% usage.

Unfortunately, just as I finished migrating my bastille jails on FreeBSD to iocage on TrueNAS CORE, the community started questioning [the future of TrueNAS CORE](https://www.truenas.com/community/threads/what-is-the-future-of-truenas-core.116049/).
This was no surprise, as iXsystems' focus seemed to be shifting more and more to Linux with their new TrueNAS Scale appliance, and the FreeBSD base of TrueNAS CORE was nearing the end of its life.
While the community was promised that TrueNAS CORE wouldn't be deprecated, many saw the writing on the wall, which was later confirmed when iXsystems said that they have [no plans to upgrade to FreeBSD 14](https://www.theregister.com/2024/03/18/truenas_abandons_freebsd/).

As I have no plans to migrate away from FreeBSD and jails, which I've become very comfortable with over the last few years, to an unfamiliar system like Linux and Docker, I decided to leave TrueNAS behind and go full FreeBSD instead.
At the same time, I decided to build a second FreeBSD NAS for my parents to use as an off-site backup and to speed up their storage needs (mostly Windows clients), as they currently access my NAS via a site-to-site VPN, which is severely bottlenecked by their cable ISP's slow upload speeds.
Since I didn't have another pair of 18 TB disks available to easily migrate all the data from TrueNAS CORE, I dug out some old 4 TB disks that were actually from my first ever NAS, which ran OpenMediaVault before I switched to FreeNAS.

After migrating all of my storage and jails away from TrueNAS CORE to my new FreeBSD NAS, those 18 TB disks were now available again, so I could now replace the old 4 TB disks for my parents NAS.
As [this second FreeBSD server](https://x.com/subnetspider/status/1555282371448111104) which at one point was my TrueNAS CORE server, already had all of its four SATA ports used up (two SSDs as the boot pool, as well as two HDDs as the storage pool), I removed all the disks and installed them in another PC.
This way I can replace the old 4 TB disks one by one with the new 18 TB disks without having to remove them and degrade the ZFS pool, but still maintain the redundancy of the mirror until the very end.

## Commands

The commands I uses to accomplish the goal are the following:

- [camcontrol(8)](https://man.freebsd.org/cgi/man.cgi?query=camcontrol(8)) - CAM control program
- [doas(1)](https://man.freebsd.org/cgi/man.cgi?query=doas(1)) - execute commands as another user
- [glabel(8)](https://man.freebsd.org/cgi/man.cgi?query=glabel(8)) - disk labelization control utility
- [gpart(8)](https://man.freebsd.org/cgi/man.cgi?query=gpart(8)) - control utility for the disk partitioning GEOM class
- [zpool-get(8)](https://man.freebsd.org/cgi/man.cgi?query=zpool-get(8)) - retrieve properties of ZFS storage pools
- [zpool-replace(8)](https://man.freebsd.org/cgi/man.cgi?query=zpool-replace(8)) - replace one device with another in ZFS storage pool
- [zpool-list(8)](https://man.freebsd.org/cgi/man.cgi?query=zpool-list(8)) - list information about ZFS storage pools
- [zpool-status(8)](https://man.freebsd.org/cgi/man.cgi?query=zpool-status(8)) - show detailed health status for ZFS storage pools
- [zpool-online(8)](https://man.freebsd.org/cgi/man.cgi?query=zpool-online(8)) - take physical devices offline in ZFS storage pool

I have linked the corresponding man pages if you want to read more about them.
In this post I will be using doas (opendoas) for elevated privileges, you can also use sudo or run the commands as root. 

## Getting started

### Identifying your disks

First, shut down your server, install the new disks, and then turn it back on again.
> On some systems you can add the new disks while the server is running, YMMV.

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
The Toshiba disks `ada0` and `ada1` are the new 18 TB disks, the Hitachi `ada2` and `ada3` are the old disks.

> ℹ️ Note
> Your disks may be labelled differently, such as `da0`, `da1` and so on.
> Also keep in mind that these device nodes are not permanent and can change.

### Prepare your disks

Once you have identified which disk is which, we can prepare the new disks for the ZFS pool.
If your new disks already have a filesystem on them, delete it with the following command:

> ⚠️ **Warning!**
> **Make sure and double check that you have selected the correct disk!**
> **Selecting the wrong disk will result in permanent data loss!**

```shell
admin@nas01:~ % doas gpart destroy -F ada0
```
```shell
admin@nas01:~ % doas gpart destroy -F ada1
```

### Create partitions

Now we can create a GPT partition table on the new disks:
```shell
admin@nas01:~ % doas gpart create -s gpt ada0
```
```shell
admin@nas01:~ % doas gpart create -s gpt ada1
```

Next we will create partitions for ZFS to use.
It is also possible to use the raw disks (e.g. `ada0` and `ada1`) with ZFS, but as these names are assigned dynamically, it can be confusing to find the correct disk after a failure.
The reason I name my disks `HDD01, HDD02, ...` is because I have labelled them all with a label maker and documented their serial and model numbers in NetBox. 

If you want to learn more about using ZFS on disk partitions, I recommend reading the following blog posts:
- https://bsdbox.de/en/artikel/freebsd-server/freebsd-server-speicher
- https://vermaden.wordpress.com/2019/06/19/freebsd-enterprise-1-pb-storage/

To check the raw storage capacity of the new disks, run the following command:
```shell
admin@nas01:~ % gpart list ada0 | grep Mediasize
   Mediasize: 18000207937536 (16T)
```

> ℹ️ Note
> 
> You can also check the raw capacity of your new disk with the command `gpart show ada0`, but keep in mind that these values have to be multiplied by the sector size reported by your disk.
>
> Most disk lie about their sector size, which today is almost always 4K, but in my case, the disk reports a sector size of 512 bytes.

To make sure we don't run into a problem where ZFS refuses to use a new disk, because it's a few sectors smaller, we'll leave some space unused.
Because I like even numbers, I will reserve the remaining `207937536` bytes (about 208 MB or 0.00116%) at the end of the disk:
```shell
admin@nas01:~ % doas gpart add -s 18000000000000B -t freebsd-zfs -l "HDD22" -a 4K "ada0"
```
```shell
admin@nas01:~ % doas gpart add -s 18000000000000B -t freebsd-zfs -l "HDD23" -a 4K "ada1"
```

> ℹ️ Note
> The `-s` flag specifies the size of the partition.
> The `-t` flage the type, which is `freebsd-zfs`.
> The `-l` flag is for our custom label (you can use the serial number instead).
> The `-a` flag is to aligh the partition to 4K sectors on the disks.

I would have liked to just use `-s 18TB` for the partition size instead, but it seems that `gpart` expects `KiB, MiB, GiB` and `TiB` values that are powers of 2 instead of powers of 10, like `KB, MB, GB` and `TB`.
Since this doesn't seem to be possible with `gpart`, I just added twelve zeros after the 18 to create a partition that is exactly 18 TB big.

To check that you have created your partitions correctly, run the following commands:
```shell
admin@nas01:~ % gpart show ada0 ada1
=>         40  35156656048  ada0  GPT  (16T)
           40  35156250000     1  freebsd-zfs  (16T)
  35156250040       406048        - free -  (198M)

=>         40  35156656048  ada1  GPT  (16T)
           40  35156250000     1  freebsd-zfs  (16T)
  35156250040       406048        - free -  (198M)
```

As we can see, there are now two partitions, `ada0p1` and `ada1p1`:
```shell
admin@nas01:~ % glabel status ada0p1 ada1p1
     Name  Status  Components
gpt/HDD22     N/A  ada0p1
gpt/HDD23     N/A  ada1p1
```

### Replace your disks

The old 4 TB disks also use partitions for ZFS:
```shell
admin@nas01:~ % zpool status data-pool
NAME             STATE     READ WRITE CKSUM
data-pool        ONLINE       0     0     0
  mirror-0       ONLINE       0     0     0
    gpt/HDD09    ONLINE       0     0     0
    gpt/HDD10    ONLINE       0     0     0
```

To replace the first disk, run the following command:
```shell
admin@nas01:~ % doas zpool replace data-pool /dev/gpt/HDD09 /dev/gpt/HDD22
```

> ℹ️ Note
> This process will take some time to complete.
> Depending on the amount of data stored on your pool, you may have to wait from a few hours to a day or two.

To see the current progress of `zpool replace`, run the following command:
```shell
admin@nas01:~ % zpool status -v data-pool
  pool: data-pool
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
	continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Mon Jan  6 01:33:12 2025
	1.36T / 1.53T scanned at 1.37G/s, 76.2G / 1.53T issued at 76.9M/s
	76.3G resilvered, 4.87% done, 05:30:06 to go
config:

	NAME             STATE     READ WRITE CKSUM
	data-pool        ONLINE       0     0     0
	  mirror-0       ONLINE       0     0     0
	    replacing-0  ONLINE       0     0     0
	      gpt/HDD09  ONLINE       0     0     0
	      gpt/HDD22  ONLINE       0     0     0  (resilvering)
	    gpt/HDD10    ONLINE       0     0     0

errors: No known data errors
```

Once the first disk has been replaced, you can replace the second disk by running the following command:
```shell
admin@nas01:~ % doas zpool replace data-pool /dev/gpt/HDD10 /dev/gpt/HDD23
```

To see the current progress of `zpool replace`, run the following command again:
```shell
admin@nas01:~ % zpool status -v data-pool
  pool: data-pool
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Mon Jan  6 10:34:48 2025
        1.30T / 1.53T scanned at 22.9G/s, 5.93G / 1.53T issued at 105M/s
        5.92G resilvered, 0.38% done, 04:13:47 to go
config:

        NAME             STATE     READ WRITE CKSUM
        data-pool        ONLINE       0     0     0
          mirror-0       ONLINE       0     0     0
            gpt/HDD22    ONLINE       0     0     0
            replacing-1  ONLINE       0     0     0
              gpt/HDD10  ONLINE       0     0     0
              gpt/HDD23  ONLINE       0     0     0  (resilvering)

errors: No known data errors
```

### Grow your pool

By default, the zpool property `autoexpand` is disabled on FreeBSD:
```shell
admin@nas01:~ % zpool get autoexpand data-pool
NAME       PROPERTY    VALUE   SOURCE
data-pool  autoexpand  off     default
```

This means your ZFS pool won't use the additional capacity automatically: 
```shell
admin@nas01:~ % zpool list -v data-pool
NAME            SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
data-pool      3.62T  1.53T  2.10T        -         -     0%    42%  1.00x    ONLINE  -
  mirror-0     3.62T  1.53T  2.10T        -         -     0%  42.1%      -    ONLINE
    gpt/HDD22  16.4T      -      -        -         -      -      -      -    ONLINE
    gpt/HDD23  16.4T
```

Instead of setting the `zpool` property `autoexpand` to `on`, I want to do things manually, so I use the following command instead:
```shell
admin@nas01:~ % doas zpool online -e data-pool gpt/HDD22
admin@nas01:~ % doas zpool online -e data-pool gpt/HDD23
```

> ℹ️ Note
> If you grow your ZFS pool, you won't be able to shrink it afterwards unless you have created a `zpool checkpoint` beforehand.

Now you can make use of the extra space in your ZFS pool:
```shell
admin@nas01:~ % zpool list -v data-pool
NAME            SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
data-pool      16.4T  1.53T  14.8T        -         -     0%     9%  1.00x    ONLINE  -
  mirror-0     16.4T  1.53T  14.8T        -         -     0%  9.33%      -    ONLINE
    gpt/HDD22  16.4T      -      -        -         -      -      -      -    ONLINE
    gpt/HDD23  16.4T      -      -        -         -      -      -      -    ONLINE
```

## Conclusion

You have now (hopefully) successfully upgraded the disks in your ZFS pool and are ready to use the additional capacity.
The old disks can now be safely removed from your server and/or reused for other projects.
If you have SMART monitoring set up for your disks, don't forget to update your `/usr/local/etc/smartd.conf`.
