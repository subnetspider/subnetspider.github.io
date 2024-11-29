---
layout: post
title: "Using IPv6 with Sophos XG Firewall"
date: 2024-11-29
categories: Sophos IPv6
---

# Using IPv6 with Sophos XG Firewall

I've been using Sophos XG Firewall for the past six months, and today I'd like to share some of the insights I've gained about using IPv6.

---

### Why Sophos?

Since 2017, I've been using OPNsense almost exclusively as my firewall, with only short periods of time when I used an Ubiquiti EdgeRouter or pfSense Firewall instead.
Although I really like OPNsense, my day job mainly involves administering 200+ Sophos firewalls, so I decided to use it as my daily driver to gain more experience with it.

---

### Hardware

I installed Sophos XG Firewall Home Edition SFOS v20 on two Fujitsu Futro S920 thin clients.
In November 2024, I upgraded both firewalls to SFOS v21, but everything remained the same as far as IPv6 was concerned.

**Hardware used**:

| Type | Details                  | 
| --- | ------------------------- |
| CPU | AMD GX-222GC (2x 2.2 GHz) |
| RAM | 8 GB DDR3 SO-DIMM         |
| SSD | Kingston 256 GB mSATA SSD |
| NIC | Intel i350-T4 (4x RJ-45)  |

The two hosts run in an active-passive HA cluster, which you are actually to do with the free Home licence.
However, the Home license limits you to 4 CPU cores and 6GB of RAM, which is more than enough for my use case.

Both firewalls are connected to a router doing NAT and PPPoE, which in turn is connected to my ISP's GPON modem as its uplink.
Although the CPUs in the firewalls are rather weak, they can push 1Gbps of traffic without too much trouble.

---

### Feature overview

The following is my experience of using IPv6 with the Sophos Firewall features that I normally use.
If you want to learn more, here is the official IPv6 compatibility list for all features: [Sophos Firewall - IPv6 support](https://docs.sophos.com/nsg/sophos-firewall/21.0/help/en-us/webhelp/onlinehelp/AdministratorHelp/AdvancedServices/IPv6FeaturesServices/index.html)

---

### Web interface

The Sophos Firewall WebAdmin interface is accessible over IPv6, so you don't need to use IPv4 to manage your firewalls.
Although you could technically access your Sophos Firewall over the internet using Global Unicast Addresses (GUA), I recommend using a VPN instead.
Alternatively, you could use Sophos Central for remote management, but I prefer to manage my firewalls locally as Central is quite slow.

---

### Firewall rules

IPv6 firewall rules have a separate rules table, so you can't create single firewall rules for dual stack hosts.
Also, both firewall rules and firewall objects cannot have the same name, which is a downgrade compared to OPNsense.
One thing I really like though is the Zones feature, which makes it very easy to block or allow traffic when using a dynamic IPv6 prefix.

> ℹ️ As of November 2024, you can't use FQDN firewall objects in IPv6 firewall rules, which would be very useful for dynamic IPv6 hosts.

---

### Web filter

The web filter supports IPv6 for blocking and allowing different URL categories.
I use this feature extensively to filter out unwanted websites and it works pretty well.
It's actually one of my favourite features of Sophos Firewall, but its blocking can be a little aggressive at times.

---

### Web application firewall

Unfortunately, the Sophos Firewall's WAF doesn't support IPv6 and is only available for IPv4.
Instead, I use HAProxy in a FreeBSD jail as my reverse proxy, even though I lose some features.

---

### Prefix Delegation

After many years of waiting, Sophos has finally included DHCPv6 prefix delegation in v20.
This feature was first requested in 2017, so many Sophos Firewall users have been stuck with NAT66 for the last 7 years.

As most ISPs sadly only assign dynamic IPv6 prefixes, this feature is very important as you can now use them.

> ℹ️ You can only request prefixes with a length of /48, /52, /56, or /60.

---

### Static Routing

IPv6 Static Routing is supported, but you are unable to use an IPv6 link local address (e.g. `fe80::1`) as the next hop.
This means you have to use a GUA or Unique Local Address (ULA, e.g. `fd12:abcd::1`) for the next hop, which is a bit limiting.

---

### SD-WAN

Currently, IPv6 is not supported by SD-WAN Rules and Profiles.
This means that routing decisions can only be made via static or dynamic routing, and you lose the ability to route traffic based on type, source, and destination.

---

### Site-to-Site VPN

As WireGuard is not supported, I use IPSec for Site-to-Site VPNs, which does support IPv6.
You can also use IPv6 GUA and ULA prefixes inside the SA of your tunnel, or use a route based tunnel instead.

Sophos XG Firewall uses strongSwan under the hood, but with some "tweaks" made by Sophos. 
This can sometimes cause interesting behaviour when using IPSec with different vendors.

> ℹ️ You can only use IPSec if your WAN interface has a static IPv6 address assigned to it.

---

### Remote Access VPN

As WireGuard is - again - not supported, I decided to use SSLVPN for my remote access VPN, which supports IPv6 inside and outsite the tunnel.
I could have used IPSec for this, but I'm not a big fan of configuring IPSec on devices that can't run the Sophos Connect VPN client.
Sophos uses only OpenVPN under the hood, so you can simply use a OpenVPN client on Linux, Android or iOS.

---

### Router Advertisement

IPv6 router advertisement is supported and works fine, but for some reason you don't have the option to send DNS server information.
All your IPv6 enabled clients will receive is the IPv6 prefix(es) and the default IPv6 gateway - thats it.
This means that you can't use IPv6 DNS servers on Android clients, as Google still refuses to implement a DHCPv6 client.

Also, you cannot advertise additional prefixes (e.g. ULA for internal addressing) while using IPv6 prefix delegation on an interface.
I really don't understand why this isnt't available on Sophos Firewall, as this is something I would always use on OPNsense.
As this is the only way to get static ULA IPv6 addresses via SLAAC if your ISP only issues dynamic IPv6 prefixes, this is quite a bummer.

---

### DHCPv6

If you want to hand out IPv6 addresses to clients the traditional way, you can use DHCPv6 with Sophos Firewall.
In my case, I only use it to send IPv6 DNS servers to my devices, except Androud, as mentioned before.

> ℹ️ You can only use DHCPv6 to send DNS servers when using prefix delegation on an interface.

---

### NAT64 + DNS64

Neither of these features are available on Sophos Firewall, which is one of the biggest downgrades for me in switching from OPNsense to Sophos.
If you still want to use NAT64 and DNS64, you will need to set up another device to do so.

---

# Let's Encrypt

As of SFOS v21, Sophos has added Support for requesting free TLS certificates via Let's Encrypt.
This is one of the most, if not the most, requested feature by Sophos users.

Unfortunately, this feature is not available over IPv6.
Sophos Firewall only supports the Lets Encrypt HTTP-01 challenge, which creates a temporary web server on port 80 on your WAN interface.

So if you want to request Let's Encrypt certificates and you don't have a public IPv4 address, you won't be able to use this feature.
Since I like to use wildcard TLS certificates that require the DNS-01 challenge, I use Certbot on my HAProxy jail.

---

### Conclusion

While it is possible to use IPv6 with Sophos Firewall, there is still **a lot** of room for improvement.
The biggest issues I have with Sophos Firewall is definitely the lack of sending DNS and additional IPv6 prefixes via Router Advertisements.
This almost made me switch back to OPNsense, but now that I have a static IPv6 prefix, it's just about bearable.

Although Sophos XG Firewall is now 9 years old, it is clear that IPv6 was an afterthought.
From my perspective, all the IPv6 features available today appear to have been added in response to customer demand.
This is because many of the users migrating from the older Sophos UTM to Sophos XG firewalls were often left without many of the IPv6 features they needed.

If you are using IPv6 and looking for a firewall for your home network, I would strongly recommend using OPNsense or pfSense instead:
- You don't have to wait half a decade for IPv6 or Let's Encrypt features to be added.
- If you run into problems, you can work them out with the developers in a matter of weeks instead of your requests falling on deaf ears.
- In addition to IPSec and OpenVPN, you can use WireGuard, ZeroTier, TailScale and many more VPN solutions.
- If you want to run additional software, there are plenty of plug-ins available, or you can even install it in a jail.
