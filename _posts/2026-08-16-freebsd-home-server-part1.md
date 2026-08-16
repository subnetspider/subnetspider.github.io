---
layout: post
title: "FreeBSD Home Server - Part 1: OS Install"
date: 2026-08-16
tags: FreeBSD
---

# FreeBSD Home Server - Part 1: OS Install

This is part 1 of my FreeBSD Home Server how-to guide, and it will cover the installation of FreeBSD.
It will show how to install FreeBSD 15.1-RELEASE, configure it, and how to set up Jails, Samba and NFS for network file sharing, as well as Bhyve virual machines.
Follow the instructions at your own risk. I am still learning, so mistakes will be made.
If you have suggestions for improvements or want to give feedback, you can find my information here: [www.subnetspider.com/about]([url](https://www.subnetspider.com/about/))

### Download the FreeBSD Installer

Go to the FreeBSD website and download the Installer:
- https://download.freebsd.org/releases/amd64/amd64/ISO-IMAGES/15.1/
> You can also download the .iso via torrent, if you want to. 

I chose the file `FreeBSD-15.1-RELEASE-amd64-bootonly.iso.xz` because it contains all the stuff I need.
> All FreeBSD .iso files are usually compatible with booting from USB.
> To save a bit of bandwidth, I always download the compressed .xz archives.

Before you extract the .xz archive, verify that the checksum matches, by downloading the CHECKSUM file.
Open `CHECKSUM.SHA256-FreeBSD-15.1-RELEASE-amd64` with a text editor (or cat/grep the contents) and look for the line which contains your downloaded filename.

### Verify the checksum

> Important: The file hashes for the .xz files will only match the compressed archives.

If you are on Linux or FreeBSD, use the `sha256sum` command:

```shell
~$ sha256sum Downloads/FreeBSD-15.1-RELEASE-amd64-bootonly.iso.xz
```
```shell
807b31dc257f8c1b3979cb48f875bc9e198f615ea1dbf5a5ab3a0fefc2115d5f  Downloads/FreeBSD-15.1-RELEASE-amd64-bootonly.iso.xz
```

If you are on Windows, open PowerShell and use the `Get-FileHash` command:

```shell
> Get-FileHash .\Downloads\FreeBSD-15.1-RELEASE-amd64-bootonly.iso.xz
```
```shell
Algorithm       Hash                                                                   Path
---------       ----                                                                   ----
SHA256          807B31DC257F8C1B3979CB48F875BC9E198F615EA1DBF5A5AB3A0FEFC2115D5F       C:\Users\Admin\Downloads\Free...
```

If the checksum matches the hash, you are good to go.
Should they not match, the file might be corrupted or manipulated, so you better delete it and start over.

### Extract the .xz archive

If you are on Linux or FreeBSD, extract the .xz archive with the `unxz` command:
```shell
~$ unxz Downloads/FreeBSD-15.1-RELEASE-amd64-bootonly.iso.xz
```

If you are on Windows, you can right click on the file and extract it like you would a .zip file.

### Prepare the installation media

You can install FreeBSD by flashing it to a USB stick, burning it to a DVD or using PXE network boot.
If your host supports it, you can also use the IPMI/BMC to attach the .iso file remotely.

But if your system does have an IPMI / BMC, you can use dd on Linux or FreeBSD to flash it to your USB stick:
```shell
~$ doas dd if=Downloads/FreeBSD-15.1-RELEASE-amd64-bootonly.iso of=/dev/sdbX bs=1M conv=sync status=progress
```
> Warning: Make sure you select the correct device (e.g. /dev/sdb) or you will encur data loss.
> Only flash the .iso file onto a USB stick if you have backed up all the files on it first. 

If you are on Windows, you can use either BalenaEtcher or Rufus to create a bootable USB stick.

### Install FreeBSD

Because the Lenovo ST250 supports it, I chose to use the XCC (XClarity Controller IPMI / BMC) of the server.

---

**Login**

<img width="795" height="374" alt="image" src="https://github.com/user-attachments/assets/448d6d7d-2d9d-44fd-80cd-55c62eedf403" />

**Launch Remote Console**

<img width="815" height="248" alt="image" src="https://github.com/user-attachments/assets/f11b2fd5-fff4-4d80-8a7c-61385345fe3f" />

**Media**

<img width="1116" height="98" alt="image" src="https://github.com/user-attachments/assets/64a3c456-4363-4b63-b83f-f9c09b1e0129" />

**Activate**

<img width="1004" height="210" alt="image" src="https://github.com/user-attachments/assets/62341134-88cb-49fc-b62d-ae73bed1d552" />

**Browse and Mount**

<img width="1004" height="210" alt="image" src="https://github.com/user-attachments/assets/da0db3cb-5a92-48e0-ba3b-ba6e8d4a2829" />
<img width="1004" height="211" alt="image" src="https://github.com/user-attachments/assets/44468a54-4486-4200-babb-0a41690e18ff" />

Now the .iso file of the FreeBSD Installer is mounted on the remove system and it can be rebooted.

**Press F12**

<img width="963" height="126" alt="HostScreenShot(2)" src="https://github.com/user-attachments/assets/f89c213f-c8ce-4cc8-b6f5-5c29d17809ff" />

**Select XCC Virtual Media**

<img width="906" height="214" alt="image" src="https://github.com/user-attachments/assets/55501e96-f0c5-4c36-896d-b86d6d489c5d" />

**Install**

<img width="1114" height="224" alt="image" src="https://github.com/user-attachments/assets/4adc7433-400d-4a49-aecc-314fc621b9b1" />

**Select you Keymap**

<img width="1114" height="169" alt="image" src="https://github.com/user-attachments/assets/3d09d224-d0f0-41b6-b7a1-6f2d4683f3a8" />

> Select the fitting keymap for your physical keyboards language layout. 

**Set Hostname**

<img width="1114" height="331" alt="image" src="https://github.com/user-attachments/assets/093512ad-4cb9-45d7-8a56-cb8d06537958" />

**Select Installation Type**

<img width="1114" height="230" alt="image" src="https://github.com/user-attachments/assets/81883454-dd6e-488c-8222-337885d22b3c" />

> I chose `Packages` for pkgbase, as this will be the default for FreeBSD 16 and later releases.

**Network Configuration**

<img width="1114" height="302" alt="image" src="https://github.com/user-attachments/assets/c25181ef-34f3-43a8-ae30-896de8664fcf" />
<img width="1114" height="302" alt="image" src="https://github.com/user-attachments/assets/7a4c5a90-e033-442e-91d9-4405616d8bb4" />
<img width="1114" height="302" alt="image" src="https://github.com/user-attachments/assets/fec71a33-bc67-4d19-bd7f-00f78b8a0eaa" />

> For now, I will only configure IPv4 networking.

**Partitioning**

<img width="1114" height="325" alt="image" src="https://github.com/user-attachments/assets/9ac7ffa0-2c04-4352-b785-4813b675c5a6" />

**Pool Type/Disks**

<img width="1114" height="536" alt="image" src="https://github.com/user-attachments/assets/8bfde6cb-e1a4-4673-b7d2-b26320b6edff" />

**Select Virtual Device Type**

<img width="1114" height="378" alt="image" src="https://github.com/user-attachments/assets/5ff0e8d9-b654-48c0-8ffb-ca60665708e0" />

> I select ZFS RAID10 for the best speed with some redundancy.
> As this server only has SSDs, I will install FreeBSD on them.
> On most servers, I have two small dedicated SSDs for FreeBSD and two or more large HDDs for mass storage.

**Select Disks**

<img width="1114" height="329" alt="image" src="https://github.com/user-attachments/assets/c539d8da-d79a-481d-907a-6113a7643f94" />

**Proceed with Installation**

<img width="1114" height="535" alt="image" src="https://github.com/user-attachments/assets/a64b62c5-0eef-480a-a4a8-842572d839f4" />

**YES**

<img width="1114" height="277" alt="image" src="https://github.com/user-attachments/assets/4dd68f67-79b9-4748-8a65-c8c039bc5370" />

> All data on the selected disks will be destroyed.

**Select System Components**

<img width="1114" height="454" alt="image" src="https://github.com/user-attachments/assets/489ddd6c-e64e-48e5-8938-d450c3707d6a" />

> I only select `base`, because I don't need `kernel-debug` or `lib32`.
> After pressing "OK", the FreeBSD installer will download all `FreeBSD-*` base system packages over the Internet.

**Boot Configuration**

<img width="1114" height="223" alt="image" src="https://github.com/user-attachments/assets/9fff2086-2a0d-4b1a-8723-eb5b0038998f" />

> There once was another FreeBSD version installed on the SSDs, otherwise, you will not see this screen.
> Press "YES".

**Set root password**

<img width="1114" height="327" alt="image" src="https://github.com/user-attachments/assets/0cf3ea60-291f-434c-8956-cec0afd4864f" />

> Set a strong password for your root user. Save it in your password manager so you don't forget.

**Select your Timezone**

<img width="1114" height="507" alt="image" src="https://github.com/user-attachments/assets/eb27eab2-0794-412b-9bc1-16ad102221ff" />

**Time & Date**

<img width="1114" height="481" alt="image" src="https://github.com/user-attachments/assets/724520e6-8259-4d40-91cb-6f3a75e06cb0" />
<img width="1114" height="222" alt="image" src="https://github.com/user-attachments/assets/0b89c40a-61a7-42c6-9e06-9a5241fd0703" />

> We can skip both, because we will enable NTP later.

**System configuration**

<img width="1114" height="405" alt="image" src="https://github.com/user-attachments/assets/aa7a2a6c-983e-4010-8ca3-723b54be3ba0" />

> I selected `sshd`, `ntpd`, `ntpd_sync_on_start`, and `powerd`.
> Note that on system with modern CPUs, `powerd` might not be needed, as the CPU's automatically reduce the clock speed when idle.

**System Hardening**

<img width="1114" height="481" alt="image" src="https://github.com/user-attachments/assets/80a09098-de30-4e07-a9ac-4e5956cf8993" />

> I always enable all available hardening options.

**Add User Accounts**

<img width="1114" height="198" alt="image" src="https://github.com/user-attachments/assets/2baa7e9d-6d14-4663-bf47-7863d587e938" />
<img width="1024" height="768" alt="rpviewer(4)" src="https://github.com/user-attachments/assets/9bdb4b25-2335-4fb1-a71a-1c3656269f55" />

> Add a non-root user account for SSH login over the network.

**Final Configuration**

<img width="1114" height="533" alt="image" src="https://github.com/user-attachments/assets/29aceb87-32f9-4476-b800-0320f75322c9" />
<img width="1114" height="246" alt="image" src="https://github.com/user-attachments/assets/186f3112-6851-40de-bd81-ddfd4c2b7407" />
<img width="1114" height="222" alt="image" src="https://github.com/user-attachments/assets/ea4c7086-9f6d-4c20-913d-2069858cbc88" />

> Select "Finish > No > Reboot" to finish the installation and reboot the system.

You can now remove your USB stick, DVD, or unmount the .ISO in your IPMI / BMC.

---

### Conclusion 

The installation of FreeBSD is now complete.
As soon as the system has booted, you are now able to use SSH to log in and configure the system.
