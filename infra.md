---
title: Infra
author: subnetspider
description: subnetspider's infrastructure
---

## Devices and services of my home network

Last modified: 2025-11-11

# Hardware

## FreeBSD Server 1

> Used for hosting services with jails.

OS: FreeBSD 14.3-RELEASE-p5<br/>
Motherboard: Supermicro X10SLH-F<br/>
CPU: Intel Xeon E3-1231 v3<br/>
RAM: 4x Samsung 8 GiB DDR3L-1600 ECC<br/>
Storage:<br/>
- 2x Intel SSD DC S3500 120GB SATA (zroot, mirror)<br/>
- 2x HGST Ultrastar SSD400M 200GB SAS (pool01, special mirror)<br/>
- 2x Toshiba Enterprise Capacity MG09ACA 18TB SATA (pool01, mirror)<br/>
Case: Inter-Tech Inter-Tech IPC 4U-4408<br/>
HBA: Broadcom LSI 9211-8i (8x 6Gbit/s SAS)<br/>
NET: Intel X710-DA2 (2x 10 Gbit/s SFP+)<br/>

## FreeBSD Server 2

> Used for storing offline backups.

OS: FreeBSD 14.3-RELEASE-p5<br/>
Motherboard: Supermicro X10SLL-F<br/>
CPU: Intel Xeon E3-1220 v3<br/>
RAM: 2x Transcend 8 GiB DDR3-1600 ECC<br/>
Storage:<br/>
- 1x Intel SSD 320 80GB SATA (boot, mirror)<br/>
- 8x HGST Ultrastar 7K3000 3TB (pool01, raidz)<br/>
Case: Supermicro CSE-825<br/>
HBA: Broadcom LSI 9211-8i (8x 6Gbit/s SAS)<br/>
NET: Chelsio T520-CT (2x 10 Gbit/s SFP+)<br/>

## Proxmox Server 1

> Used for hosting services with KVM.

OS: Proxmox VE 8.4.12<br/>
Motherboard: ASUS Pro B460M-C<br/>
CPU: Intel Core i3-10100<br/>
RAM: 4x G.Skill 16 GiB DDR4-3000 UDIMM<br/>
Storage:<br/>
- 2x Intel SSD DC S3500 80GB SATA (boot, mirror)<br/>
- 2x Kingston A2000 NVMe PCIe SSD 500GB (vm, mirror)<br/>
Case: Inter-Tech 4088 Rev.2<br/>
NET: Intel i350-T4 (4x 1 Gbit/s RJ45)<br/>

