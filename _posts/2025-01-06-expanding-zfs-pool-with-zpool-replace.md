---
layout: post
title: "Expanding a ZFS pool with zpool-replace"
date: 2025-01-06
tags: FreeBSD OpenZFS Storage
---

# Expanding a ZFS pool with zpool-replace

### Background

For the last month I've been working on migrating my storage to FreeBSD, which has been managed by the FreeNAS / TrueNAS CORE since the beginning of 2019.
[A year ago]([url](https://x.com/subnetspider/status/1742303228438761484)) I decided to consolidate two of my servers, one running FreeBSD 14.0 and hosting most of my services in bastille jails, the other running TrueNAS CORE for my storage.
This was mainly to save a bit of power, as the FreeBSD jail server was way overkill, using only 8 of 64 GiB of RAM and less than 30 of 500 GB of NVMe storage, with the i3-10100 CPU mostly idle, 1-2% usage.

Unfortunately, just as I finished migrating my bastille jails on FreeBSD to iocage on TrueNAS CORE, users started questioning [the future of TrueNAS CORE]([url](https://www.truenas.com/community/threads/what-is-the-future-of-truenas-core.116049/)).
This was no surprise, as iXsystems' focus seemed to be shifting more and more to Linux with their new TrueNAS Scale appliance, and the FreeBSD base of TrueNAS CORE was nearing the end of its life.
While the community was promised that TrueNAS CORE wouldn't be deprecated, many saw the writing on the wall, which was later confirmed when iXsystems said that they have [no plans to upgrade to FreeBSD 14]([url](https://www.theregister.com/2024/03/18/truenas_abandons_freebsd/)).

As I have no plans to migrate away from FreeBSD and jails, which I've become very comfortable with over the last few years, to an unfamiliar system like Linux and Docker, I decided to leave TrueNAS behind and go full FreeBSD instead.
At the same time, I decided to build a second FreeBSD NAS for my parents to use as an off-site backup and to speed up their storage needs (mostly Windows clients), as they currently access my NAS via a site-to-site VPN, which is severely bottlenecked by their cable ISP's slow upload speeds.
Since I didn't have another pair of 18TB disks available to easily migrate all the data from TrueNAS CORE, I dug out some old 4TB disks that were actually from my first ever NAS, which ran OpenMediaVault before I switched to FreeNAS.

After migrating all of my storage and jails away from TrueNAS CORE to my new FreeBSD NAS, those 18 TB disks were now available again, so I could now replace the old 4 TB disks for my parents NAS.
As [this second FreeBSD server]([url](https://x.com/subnetspider/status/1555282371448111104)) which at one point was my TrueNAS CORE server, already had all of its four SATA ports used up (two SSDs as the boot pool, as well as two HDDs as the storage pool), I removed all the disks and installed them in another PC.
This way I can replace the old 4TB disks one by one with the new 18TB disks without having to remove them and degrade the ZFS pool, but still maintain the redundancy of the mirror until the very end.
