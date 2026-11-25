# IPv4 and IPv6 Addressing

## Overview

Every host on an IP network needs an **IP address**. IPv4 is the older and still dominant format. IPv6 is its successor, with a far larger address space. Most Linux systems run **dual stack**: both at the same time.

## Key concepts

### IPv4

- 32 bits, written as four decimal numbers (octets) separated by dots: `192.0.2.10`.
- Each octet is 0–255.
- About 4.3 billion addresses. Public addresses are scarce, so **NAT** is widespread.

### Special IPv4 ranges

| Range | Purpose |
|-------|---------|
| `10.0.0.0/8` | **Private** (RFC 1918) |
| `172.16.0.0/12` (172.16.0.0–172.31.255.255) | **Private** |
| `192.168.0.0/16` | **Private** |
| `100.64.0.0/10` | Carrier-grade NAT (shared address space) |
| `127.0.0.0/8` | **Loopback** (`127.0.0.1`) |
| `169.254.0.0/16` | **Link-local** (self-assigned when DHCP fails) |
| `224.0.0.0/4` | Multicast |
| `0.0.0.0` | "Any" or "this host" (used when binding or as a default route) |
| `255.255.255.255` | Limited broadcast |
| `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` | Reserved for **documentation** (used in this guide) |

Private addresses are not routed on the public internet. A router with NAT translates them.

### IPv6

- 128 bits, written as eight groups of four hex digits separated by colons.
- Leading zeros in a group can be dropped, and **one** run of all-zero groups can be replaced by `::`.

```
2001:0db8:0000:0000:0000:0000:0000:0001
2001:db8:0:0:0:0:0:1        (leading zeros dropped)
2001:db8::1                 (zeros compressed with ::)
```

An IPv6 address with a prefix: `2001:db8:10:1::25/64`. The first 64 bits are the **network prefix**, the last 64 the **interface identifier**. Standard subnets are **/64**.

### IPv6 address types

| Type | Prefix | Description |
|------|--------|-------------|
| **Global unicast** | `2000::/3` | Public, routable addresses |
| **Link-local** | `fe80::/10` | Valid only on the local link. Every interface has one. **Required** for IPv6 to work |
| **Unique local (ULA)** | `fc00::/7` (in practice `fd00::/8`) | Private, like RFC 1918 |
| **Loopback** | `::1/128` | Local host |
| **Multicast** | `ff00::/8` | Group delivery. Replaces broadcast |
| **Unspecified** | `::/128` | "No address yet" |
| **Documentation** | `2001:db8::/32` | Examples |

IPv6 has **no broadcast** and no ARP. It uses multicast and **NDP** (Neighbor Discovery Protocol, over ICMPv6).

### How addresses are assigned

| Method | IPv4 | IPv6 |
|--------|------|------|
| **Static** (configured by hand) | Yes | Yes |
| **DHCP** | DHCPv4 | **DHCPv6** (stateful) |
| **Self-configuration** | Link-local `169.254.x.x` fallback | **SLAAC**: the router announces the prefix (Router Advertisement) and the host builds its own address |
| **Privacy addresses** | — | Temporary random addresses for outgoing connections (RFC 4941) |

A Linux host can have **several addresses per interface**, for example one global IPv6, a temporary privacy address, a link-local, and one or more IPv4 addresses.

## Commands

```console
$ ip -4 addr show enp3s0
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.0.2.10/24 brd 192.0.2.255 scope global dynamic noprefixroute enp3s0
       valid_lft 85023sec preferred_lft 85023sec

$ ip -6 addr show enp3s0
    inet6 2001:db8:10:1:5054:ff:fe12:3456/64 scope global dynamic mngtmpaddr
    inet6 2001:db8:10:1:a1b2:c3d4:e5f6:7788/64 scope global temporary dynamic
    inet6 fe80::5054:ff:fe12:3456/64 scope link
```

| Field | Meaning |
|-------|---------|
| `inet` / `inet6` | IPv4 / IPv6 address and prefix length |
| `brd` | IPv4 broadcast address |
| `scope global / link / host` | Where the address is valid |
| `dynamic` | Assigned by DHCP or SLAAC (has a lease or lifetime) |
| `temporary` | IPv6 privacy address |
| `valid_lft`, `preferred_lft` | Remaining lifetimes (`forever` for static) |

Add and remove addresses (temporary, root):

```console
# ip addr add 192.0.2.11/24 dev enp3s0          # add a second IPv4 address
# ip addr add 2001:db8:10:1::25/64 dev enp3s0
# ip addr del 192.0.2.11/24 dev enp3s0
# ip addr flush dev enp3s0 scope global         # remove all global addresses (careful!)
```

Check the public address and test IPv6:

```console
$ curl -4 https://ifconfig.me          # your public IPv4 address (as seen from the internet)
$ curl -6 https://ifconfig.me          # public IPv6, if available
$ ping -6 -c 2 2001:db8::1
$ ping -c 2 fe80::1%enp3s0             # link-local needs the interface (%name)
```

### Sysctl settings for IPv6

| Setting | Meaning |
|---------|---------|
| `net.ipv6.conf.all.disable_ipv6=1` | Disable IPv6 (usually **not** recommended; some software breaks) |
| `net.ipv6.conf.IFACE.accept_ra` | Accept Router Advertisements |
| `net.ipv6.conf.IFACE.use_tempaddr` | Privacy addresses (0 off, 2 prefer temporary) |

## Common pitfalls

- **More than one `::` in an IPv6 address.** It is allowed only once.
- **Forgetting the zone ID for link-local addresses.** `ping fe80::1` fails. Use `fe80::1%enp3s0`.
- **Disabling IPv6 instead of securing it.** Many systems and services use it. If you do not manage IPv6, make sure the firewall covers it anyway (see [Firewall Concepts](../13-remote-access-firewalls-services/05-firewall-concepts.md)).
- **Firewall rules only for IPv4.** IPv6 traffic then bypasses the protection.
- **Services listening on `::`** are usually reachable on IPv4 as well (dual-stack socket). Check with `ss -tln`.
- **Using public-looking addresses in private networks** (for example, `1.2.3.4`) clashes with real internet hosts.
- **Overlapping private ranges** between sites and VPNs (everyone uses `192.168.1.0/24`). Plan your address space.
- **Applications hard-coding IPv4.** In dual-stack environments, a name may resolve to IPv6 first. If IPv6 is broken, connections are slow ("happy eyeballs" delay). Fix IPv6 or the DNS record.

## Further reading

- [`ip-address(8)`](https://man7.org/linux/man-pages/man8/ip-address.8.html)
- [RFC 1918: Private address space](https://datatracker.ietf.org/doc/html/rfc1918)
- [RFC 4291: IPv6 addressing architecture](https://datatracker.ietf.org/doc/html/rfc4291)
- [RFC 5737: IPv4 documentation addresses](https://datatracker.ietf.org/doc/html/rfc5737)
