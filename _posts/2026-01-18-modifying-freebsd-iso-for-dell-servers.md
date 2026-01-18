---
layout: post
title: "Modifying FreeBSD ISO for Dell Servers"
date: 2026-01-18
tags: FreeBSD ISO Install
---

# Modifying FreeBSD ISO for Dell Servers

## Background

My NAS that I built in 2019 to host my files on my home network has started to show its age, especially as the number of services running in jails increased.
It all started when I discovered the amazing features and data integrity that OpenZFS had to offer, and I tried out FreeNAS 11.2, which got released with a completely overhauled web interface at the time.
Back then, I already had my media on Plex Media Server and learned that FreeNAS had Plex as a Plugin, which was my first contact with FreeBSD Jails.

Over time, I added more Jails on my NAS (e.g. Nextcloud, MineOS - a Minecraft Server Manager, UniFi Controller, ...), which made me interested in learning more about Jails and FreeBSD itself.
When some time later FreeNAS became TrueNAS Core and a few years after that, iXsystems decided to move forward with Linux, I migrated my NAS to FreeBSD.
At the time of writing, the number of jails on my FreeBSD system has increased to almost 20, and the once plentiful 32 GiB of RAM seems smaller by the day.

While the old NAS is still capable of running my services and hosting files, the CPU (Intel Xeon E3-1231 v3) sometimes struggles when there is lots of usage.
Similary, the amount of RAM is still okay, but with 50% of constant usage doesn't leave a lot of space to grow.
To remedy there issues, I wanted to upgrade the NAS with a AM5 board and CPU, but right before I was able to do so, the prices for DDR5 absulutely exploded.

As I didn't want to spend 1200€ just for a 64 GiB DDR5 kit what only cost 180€ a couple weeks ago, I chose to buy a used Dell PowerEdge R530 instead.
It already came with 64 GiB of DDR4 preinstalled, which I promptly upgraded to 128 GiB, as prices for used DDR4 server RAM is fairly manageable.
When I first installed FreeBSD on the Dell R530, I noticed a lot of write and i/o errors on the console, which lead me to writing this post.

## Not all LSI SAS HBA are made equally

Searching for the error messages, I found [this blog post]([url](https://dan.langille.org/2023/02/24/booting-without-mrsas-driver-lots-of-errors/)) by Dan Langille, which explained the issue and a solution.
It turnes out that FreeBSD needs the [mrsas(4)]([url](https://man.freebsd.org/cgi/man.cgi?query=mrsas(4))) driver for the newer Broadcom/LSI SAS HBAs, which isn't used by default.
While I was able to get an old Install of FreeBSD to run on the Dell server, I wasn't able to boot a fresh install, as the FreeBSD installer used the old and imcompatible [mfi(4)]([url](https://man.freebsd.org/cgi/man.cgi?query=mfi(4))) driver.

To remedy this, I decided to modify the ISO of the FreeBSD installer, so it will use the correct driver during the installation process.
Unfortunately, I forgot to write down where I found the information I needed, so no credits this time. :/

## Modification steps

> The following steps were done on my Pop!_OS Linux desktop, because I was in a rush.
> If you want to do this on FreeBSD, you need to install `sysutils/dvd+rw-tools` to use the `growisofs` command (not tested).

First, the ISO installer to be modified needs to be downloaded:
```shell
wget https://download.freebsd.org/releases/amd64/amd64/ISO-IMAGES/15.0/FreeBSD-15.0-RELEASE-amd64-disc1.iso
```
---

To avoid confusion, the modified ISO file is renamed:
```bash
mv FreeBSD-15.0-RELEASE-amd64-disc1.iso FreeBSD-15.0-RELEASE-amd64-DELL.iso
```

Next, we install the program `growisofs` to modify the ISO file:
```bash
sudo apt install growisofs
```

Next, we read information from the ISO file and store the information in the variable `volid`:
```bash
volid=$(isoinfo -d -i FreeBSD-15.0-RELEASE-amd64-DELL.iso | awk '/Volume id/{print$3}')
```

> **Optional**: Check whether the variable has been created correctly:
> ```bash
> admin@pop-os:~$ echo $volid
> 15_0_RELEASE_AMD64_CD
> ```

Now we create the empty file `loader.conf.local` and store the following data in it:
```bash
touch loader.conf.local
```

```bash
echo 'hw.mfi.mrsas_enable="1"' > loader.conf.local
```

> **Optional**: Check whether the file was populated correctly:
> ```bash
> admin@pop-os:~$ cat loader.conf.local 
> hw.mfi.mrsas_enable="1"
> ```

Next, we'll extend our FreeBSD installer ISO file with `loader.conf.local`:
```bash
growisofs -M FreeBSD-15.0-RELEASE-amd64-DELL.iso -d -l -r -V "$volid" -graft-points /boot/loader.conf.local=loader.conf.local
```

Now we can copy the resulting ISO file to a USB stick. I used the software Ventoy for this.

FreeBSD can now be installed on the Dell server as usual.
Once the installation is complete, create the file `/boot/loader.conf.local` on the freh FreeBSD install:
```shell
ee /boot/loader.conf.local
```

Enter the following and then save + exit the file with pressing the Esc key.
```shell
hw.mfi.mrsas_enable="1"
```

After that, the USB stick can be removed and the server restarted—done.
