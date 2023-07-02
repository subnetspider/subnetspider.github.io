---
layout: post
title: "Setting up dynamic and static IPv6 addresses on FreeBSD"
date: 2023-07-02
categories: FreeBSD IPv6
---

# Setting up dynamic and static IPv6 addresses on FreeBSD

### Foreword

I assume that you, the reader, have a basic knowledge of networking and are familiar with IPv6 terms such as unique local addresses (ULA) and global unicast addresses (GUA).

### Background

In my journey to achieve IPv6-only networking in my homelab, I decided to replace my Sophos XGS firewall with OPNsense.
The reason for this is that while the Sophos XGS Firewall is great at filtering and blocking, it is still lacking in the IPv6 department.
One of the biggest limitations for me when using Sophos XGS Firewall with IPv6 was that I was forced to use NAT66.
This meant that my FreeBSD servers and jails, as well as my client devices, preferred IPv4 to NATed IPv6.

This was because SFOS v19.5 still does not support IPv6 prefix delegation, which is required to use your ISP's global IPv6 prefix.
As a result, there was no way around using ULA addresses within my network, with all hosts relying on NAT66 to access the internet.
Sophos appears to be planning to implement more IPv6 functionality in future firmware releases, but this is likely to be years away.

### The problem

Both of my ISPs (one FTTH, one cable) offer a dynamic /56 IPv6 prefix, which is fine for accessing the web, but a challenge for local networking.
I do not want to rely on static IPv4 addresses to manage my network devices, and using only dynamic IPv6 addresses is out of the question.

Possible solutions are
- Use dynamic DNS or create a script that changes the prefix for my AAAA records.
- Pay a lot of money for a business contract so that my ISP gives me a static IPv6 prefix.
- Get my own IPv6 Provider Independent (PI) IPv6 prefix and advertise it via BGP.

However, the first solution adds complexity, the second adds cost, and the third adds both.

### The solution

My solution: Use static ULA's via IP aliases for internal services/devices and dynamic GUA's via SLAAC for Internet access.
Achieving this was actually quite easy for me, as I was already relying on ULA addresses.
The only change is that all clients and servers now get a dynamic GUA address.

Currently, all my servers, jails, and network devices have a static IPv4 address from the 172.31.0.0/16 range and a static IPv6 address from the fd00::/8 range.
On the other hand, all clients have an IPv4 address assigned via DHCP and an IPv6 address generated from the IPv6 prefix via SLAAC.

### OPNsense setup

**WAN interface**

- Login to your OPNsense firewall with the webbrowser of your choice.
- Navigate to "Interfaces > WAN".
- Under "Generic configuration", select:
	- IPv6 Configuration Type: **DHCPv6**
- Under "DHCPv6 client configuration", select:
	- Configuration Mode: **Basic**
	- Request only an IPv6 prefix: ☑️
	- Prefix delegation size: **56**
	- Send IPv6 prefix hint: ☑️
	- Use IPv4 connectivity: ☑️
	- Use VLAN priority: **Disabled**
	- Save > Apply

You can find out more here: [Configure IPv6 for generic DSL dialup](https://docs.opnsense.org/manual/how-tos/ipv6_dsl.html)

> ℹ️ **Note**
> 
> The prefix delegation size may vary from ISP to ISP.

**LAN interface**

- Navigate to "Interfaces > LAN".
- Under "Generic configuration", select:
	- IPv6 Configuration Type: **Track Interface**
- Under "Track IPv6 Interface", select:
	- IPv6 Interface: **WAN**
	- IPv6 Prefix ID: **0**
	- Manual configuration: ☑️
	- Save > Apply

> ℹ️ **Note**
> 
> For each additional interface, increase the value of "IPv6 prefix ID" by one.

**Virtual IPs**

- Navigate to "Interfaces > - Virtual IPs > Settings".
- Click on the plus sign to add a virual IP.
- Under "Edit Virtual IP", select:
	-  Mode: **Alias**
	-  Interface: **LAN**
	- Network / Address: **fd00:10::1/64**
	- Save > Apply

> ℹ️ **Note**
> 
> If you have subnets where all hosts have static ULA addresses, you can check the box "Deny service binding".
> This way, OPNsense only sends router advertisements for the global prefix it gets from "track interface", so no additional ULA addresses are generated on your hosts.

**Firewall rules**

Create firewall rules for IPv6 traffic. This part varies greatly depending on your network and security requirements, so decide what works best for your environment.

**Router advertisement**

- Navigate to "Services > Router Advertisements > [LAN]".
- Select the following:
	- Router Advertisements: **Unmanaged**
	- Router Priority: **Normal**
	- Source Address: **Automatic**
	- Advertise Default Gateway: ☑️
	- Save > Apply

### FreeBSD setup

To change the network configuration on your FreeBSD host, edit the file `/etc/rc. conf` with your favorite text editor (or create a file at `/etc/rc.conf.d` if you prefer).

In this example, here is a snippet of the file `/etc/rc.conf` of my FreeBSD test VM:

```vim
# Network IPv4
ifconfig_vtnet0="inet 172.31.20.11/24"
defaultrouter="172.31.20.1"

# Network IPv6 (new)
ifconfig_vtnet0_ipv6="inet6 accept_rtadv"
ifconfig_vtnet0_alias0="inet6 fd00:20::11/64"

# Network IPv6 (old)
#ifconfig_vtnet0_ipv6="inet6 fd00:20::11/64"
#ipv6_defaultrouter="fd00:20::1"
```

The only things I had to change to make this work were two commented out lines.
Before, I had statically assigned the IPv6 address `fd00:20::11/64` to the interface `vtnet0`. With the second line, I set the IPv6 default gateway to `fd00:20::1`.

Now FreeBSD listens for router advertisements on the `vtnet0` interface and generates a global address from the prefix(es) provided by OPNsense. I then created an alias `alias0` so that I could continue to use the static ULA address I had previously assigned.

That's it - as simple as that.

All that remains is to apply the new configuration.
You can either reboot your FreeBSD host, or run the following commands:

```shell
sudo service netif restart vtnet0
```

```shell
sudo service routing restart
```

> ℹ️ **Note**
> 
> Change the interface name `vtnet0` to the interface used by your FreeBSD hosts.
> You can also omit the interface name in the first command, which will restart all interfaces.

> ❗ Important
>
> In some of my FreeBSD 13.2-RELEASE VNET jails, the default IPv6 route wouldn't be set no matter what commands I ran.
> Even if you're not experiencing connectivity problems, I recommend restarting VNET jails rather than running the commands above.
> At this point, I am not shure if this is caused by the OPNsense router advertisement daemon `radvd` or FreeBSD router solicitation daemon `rtsold`.
> As soon as I find out, I will update this post accordingly.

And, it works:

```shell
admin@freebsd-test:~ % ifconfig vtnet0
vtnet0: flags=8863<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=4c07bb<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,TSO4,TSO6,LRO,VLAN_HWTSO,LINKSTATE,TXCSUM_IPV6>
	ether 62:0e:59:e9:d1:d1
	inet 172.31.20.11 netmask 0xffffff00 broadcast 172.31.20.255
	inet6 fe80::600e:59ff:fee9:d1d1%vtnet0 prefixlen 64 scopeid 0x1
	inet6 fd00:20::11 prefixlen 64
	inet6 2XXX:XX:XXXX:e201:600e:59ff:fee9:d1d1 prefixlen 64 autoconf
	media: Ethernet autoselect (10Gbase-T <full-duplex>)
	status: active
	nd6 options=23<PERFORMNUD,ACCEPT_RTADV,AUTO_LINKLOCAL>
```

### Summary

Your FreeBSD host(s) should now have a global IPv6 address for accessing the Internet, and a local IPv6 address for accessing internal resources such as DNS.
This works the same way for FreeBSD VNET jails, and can be automated using Ansible.

---
