# Chapter 11: Linux Networking Fundamentals

| # | Topic | Summary |
|---|-------|---------|
| 1 | [TCP/IP and the OSI Model](01-tcp-ip-and-osi-model.md) | Layers, encapsulation, TCP vs. UDP, ports |
| 2 | [Network Interfaces](02-network-interfaces.md) | Interface names, types, link state, `ethtool` |
| 3 | [IPv4 and IPv6 Addressing](03-ipv4-and-ipv6-addressing.md) | Address types, private ranges, SLAAC |
| 4 | [Subnetting and CIDR](04-subnetting-and-cidr.md) | Prefix lengths, network and broadcast addresses |
| 5 | [MAC Addresses and ARP](05-mac-addresses-and-arp.md) | Local delivery, neighbor tables |
| 6 | [Routing Tables and Default Gateways](06-routing-and-default-gateways.md) | How Linux chooses where to send packets |
| 7 | [DNS Resolution and Hostname Configuration](07-dns-resolution-and-hostname.md) | `/etc/hosts`, resolvers, `systemd-resolved` |
| 8 | [DHCP Fundamentals](08-dhcp-fundamentals.md) | Leases, options, clients |
| 9 | [NetworkManager and nmcli](09-networkmanager-and-nmcli.md) | Connection profiles and commands |
| 10 | [The ip and ss Commands](10-ip-and-ss.md) | The modern replacements for `ifconfig` and `netstat` |
| 11 | [Network Configuration Persistence](11-network-configuration-persistence.md) | Netplan, networkd, ifupdown, NetworkManager files |

> [!NOTE]
> Examples use reserved documentation addresses: `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`, and `2001:db8::/32`. Replace them with your own.

**Previous:** [Chapter 10](../../part-2-system-administration/10-logging-time-and-scheduling/README.md) | **Next:** [Chapter 12](../12-network-troubleshooting/README.md)
