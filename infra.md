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

- OS: FreeBSD 14.3-RELEASE-p5
- Motherboard: Supermicro X10SLH-F
- CPU: Intel Xeon E3-1231 v3
- RAM: 4x Samsung 8 GiB DDR3L-1600 ECC
- Storage:
  - 2x Intel SSD DC S3500 120GB SATA (zroot, mirror)
  - 2x HGST Ultrastar SSD400M 200GB SAS (pool01, special mirror)
  - 2x Toshiba Enterprise Capacity MG09ACA 18TB SATA (pool01, mirror)
- Case: Inter-Tech Inter-Tech IPC 4U-4408
- HBA: Broadcom LSI 9211-8i (8x 6Gbit/s SAS)
- NET: Intel X710-DA2 (2x 10 Gbit/s SFP+)

## FreeBSD Server 2

> Used for storing offline backups.

- OS: FreeBSD 14.3-RELEASE-p5
- Motherboard: Supermicro X10SLL-F
- CPU: Intel Xeon E3-1220 v3
- RAM: 2x Transcend 8 GiB DDR3-1600 ECC
- Storage:
  - 1x Intel SSD 320 80GB SATA (boot, mirror)
  - 8x HGST Ultrastar 7K3000 3TB (pool01, raidz)
- Case: Supermicro CSE-825
- HBA: Broadcom LSI 9211-8i (8x 6Gbit/s SAS)
- NET: Chelsio T520-CT (2x 10 Gbit/s SFP+)

## Proxmox Server 1

> Used for hosting services with KVM.

- OS: Proxmox VE 8.4.12
- Motherboard: ASUS Pro B460M-C
- CPU: Intel Core i3-10100
- RAM: 4x G.Skill 16 GiB DDR4-3000 UDIMM
- Storage:
  - 2x Intel SSD DC S3500 80GB SATA (boot, mirror)
  - 2x Kingston A2000 NVMe PCIe SSD 500GB (vm, mirror)
- Case: Inter-Tech 4088 Rev.2
- NET: Intel i350-T4 (4x 1 Gbit/s RJ45)
