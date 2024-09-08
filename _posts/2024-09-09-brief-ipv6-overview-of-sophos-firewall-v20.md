---
layout: post
title: "Brief IPv6 overview of Sophos Firewall v20"
date: 2024-09-09
categories: Sophos IPv6
---

# Brief IPv6 overview of Sophos Firewall v20

For the past two months I have been using Sophos Firewall Home Edition as the firewall for my home network.
As I make extensive use of IPv6 on my home network, it is crucial that the firewall software I use supports it.
Today, I would like to share some insight from my perspective on using IPv6 on a day-to-day basis with Sophos Firewall.

### Why Sophos Firewall?

Although I am a big fan of OPNsense, I mainly use Sophos Firewall for my day job.
In order to gain more experience with it, I decided to use it for the time being.
After all, the most effective way to gain experience with something unfamiliar is to immerse yourself in it.

### What hardware am I using?

I installed Sophos Firewall v20 on two identical Fujitsu Futro S920 Thin Clients:

|     |               |
| --- | ------------- |
| CPU | AMD GX-222GC  |
| RAM | 8 GB DDR3     |
| SSD | 250 GB SSD    |
| NIC | Intel i350-T4 |

Both machines run in an active-passive HA cluster, which I didn't expect to be able to use.
However, the free license limits the usable CPU cores/threads to 4 and the usable RAM to 6GB.

### IPv6 Support

| Feature                | Implemented |
| ---------------------- | ----------- |
| Web Interface via IPv6 | Yes         |
| Prefix Delegation      | Yes         |
| IPv6 WAF               | No          |
| IPv6 Firewall rules    | Yes¹        |          
| IPv6 Static Routing    | Yes²        |
| IPv6 Dynamic Routing   | Yes³        |
| IPv6 SD-WAN            | No          |
| IPv6  Site-to-Site VPN | Yes         |
| IPv6 Remote Access VPN | Yes         |
| Router Advertisement   | Yes⁴ ⁵      |
| DHCPv6 Server          | Yes         |
| NAT64                  | No          |
| DNS64                  | No          |

1 - IPv4 and IPv6 Rules have their own seperate table, also IPv4/IPv6 rules and objects cant have the same name.
2 - It is not possible to use a IPv6 Link Local Address as Next Hop when using Route-Based IPSec VPN.
3 - OSPFv3 is very limited in the GUI, advanced features are only accessible from the CLI. 
4 - DNS information via Router Advertisements is not supported.
5 - It is not possible to send additional IPv6 prefixes via Router Advertisement on an interface which uses Prefix Delegation.

