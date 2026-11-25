# Subnetting and CIDR

## Overview

A **subnet** is a range of IP addresses that share the same network part. **CIDR** (Classless Inter-Domain Routing) writes it as an address plus a **prefix length**: `192.0.2.0/24`. Subnetting lets you divide a network, plan address space, and decide which hosts can talk directly.

## Key concepts

### Prefix length and netmask

The prefix length is the number of leading bits that identify the **network**. The remaining bits identify the **host**.

```
192.0.2.10/24
11000000.00000000.00000010.00001010
└──────── 24 network bits ────────┘└ 8 host bits ┘
Netmask: 11111111.11111111.11111111.00000000 = 255.255.255.0
```

| CIDR | Netmask | Addresses | Usable hosts |
|------|---------|-----------|--------------|
| `/8` | 255.0.0.0 | 16,777,216 | 16,777,214 |
| `/16` | 255.255.0.0 | 65,536 | 65,534 |
| `/20` | 255.255.240.0 | 4,096 | 4,094 |
| `/22` | 255.255.252.0 | 1,024 | 1,022 |
| `/23` | 255.255.254.0 | 512 | 510 |
| `/24` | 255.255.255.0 | 256 | 254 |
| `/25` | 255.255.255.128 | 128 | 126 |
| `/26` | 255.255.255.192 | 64 | 62 |
| `/27` | 255.255.255.224 | 32 | 30 |
| `/28` | 255.255.255.240 | 16 | 14 |
| `/29` | 255.255.255.248 | 8 | 6 |
| `/30` | 255.255.255.252 | 4 | 2 |
| `/31` | 255.255.255.254 | 2 | 2 (point-to-point links, RFC 3021) |
| `/32` | 255.255.255.255 | 1 | 1 (a single host) |

**Formula:** addresses = 2^(32 − prefix). Usable hosts = addresses − 2 (the **network** address and the **broadcast** address), except for /31 and /32.

### Special addresses in a subnet

| Address | Role | Example for `192.0.2.0/24` |
|---------|------|---------------------------|
| **Network address** | First address, all host bits 0. Identifies the subnet | `192.0.2.0` |
| **First usable** | Network + 1 | `192.0.2.1` |
| **Last usable** | Broadcast − 1 | `192.0.2.254` |
| **Broadcast address** | Last address, all host bits 1 | `192.0.2.255` |

### Same subnet or not?

Two hosts are on the **same subnet** if their addresses, ANDed with the netmask, give the same network address. Hosts on the same subnet talk directly (ARP). Hosts on different subnets need a **router**.

```
Host A: 192.0.2.10/24  → network 192.0.2.0
Host B: 192.0.2.200/24 → network 192.0.2.0   → same subnet, direct
Host C: 192.0.3.5/24   → network 192.0.3.0   → different, needs gateway
```

### Step by step: split a /24 into four subnets

Goal: split `192.0.2.0/24` into four equal subnets.

1. Four subnets need 2 extra bits (2² = 4), so the new prefix is **/26**.
2. Each /26 has 64 addresses (block size 64).
3. The subnets start at multiples of 64:

| Subnet | Network | Usable range | Broadcast |
|--------|---------|--------------|-----------|
| 1 | `192.0.2.0/26` | .1 – .62 | .63 |
| 2 | `192.0.2.64/26` | .65 – .126 | .127 |
| 3 | `192.0.2.128/26` | .129 – .190 | .191 |
| 4 | `192.0.2.192/26` | .193 – .254 | .255 |

**Block-size shortcut:** block size = 256 − the interesting octet of the netmask. For `/26` (mask `.192`), block size = 256 − 192 = 64.

### Larger example

`10.0.0.0/22` covers `10.0.0.0` – `10.0.3.255` (1,024 addresses), because a /22 spans four /24s in the third octet.

### Supernets and route aggregation

Several adjacent subnets can be combined into one larger prefix, **if they align**: `192.0.2.0/24` and `192.0.3.0/24` combine into `192.0.2.0/23`. Aggregation keeps routing tables small.

### IPv6 subnetting

Subnetting is simpler: use **/64** for every LAN. A site normally receives a **/48** or **/56**, which gives 65,536 or 256 /64 subnets. Subnet by changing hex digits in the prefix: `2001:db8:10:1::/64`, `2001:db8:10:2::/64`.

## Commands

```console
$ ipcalc 192.0.2.10/26                      # RHEL-based: ipcalc; Debian-based: ipcalc or sipcalc
Address:   192.0.2.10
Netmask:   255.255.255.192 = 26
Network:   192.0.2.0/26
Broadcast: 192.0.2.63
HostMin:   192.0.2.1
HostMax:   192.0.2.62
Hosts/Net: 62

$ python3 -c "import ipaddress as i; n=i.ip_network('192.0.2.0/24'); print([str(s) for s in n.subnets(prefixlen_diff=2)])"
['192.0.2.0/26', '192.0.2.64/26', '192.0.2.128/26', '192.0.2.192/26']

$ python3 -c "import ipaddress as i; print(i.ip_address('192.0.2.77') in i.ip_network('192.0.2.64/26'))"
True
```

Output formats of `ipcalc` differ between distributions. The `ipaddress` module in Python 3 works everywhere.

```console
$ ip route                       # connected subnets appear as routes automatically
192.0.2.0/24 dev enp3s0 proto kernel scope link src 192.0.2.10
```

## Planning tips

1. **Count hosts, then add growth** (at least 30–50%). Pick the prefix from the table.
2. Keep subnets **aligned** to their block size.
3. Use **different private ranges** at different sites (and for VPN, cloud, and container networks) to avoid overlap.
4. Document the plan (IPAM tools such as NetBox or phpIPAM help).
5. Reserve space: gateway, servers, DHCP ranges, static devices.
6. Separate by role (servers, users, management, guests) so firewalls can control traffic between them.

## Common pitfalls

- **Wrong netmask** on a host: it thinks remote hosts are local and fails to reach them via the gateway (or the reverse).
- **Overlapping subnets** (a VPN range that overlaps the office LAN). Traffic goes to the wrong place.
- **Counting the network and broadcast addresses as usable.**
- **Misaligned subnet boundaries.** `192.0.2.100/26` is in the subnet `192.0.2.64/26`, not a network of its own.
- **Using /24 for everything** and running out of addresses, or wasting address space. Cloud VPCs and Kubernetes pod networks need careful sizing.
- **Gateway outside the subnet.** The gateway must be in the same subnet as the interface (except special on-link routes).
- **Forgetting IPv6 prefix size.** Subnets smaller than /64 break SLAAC and some features.

## Further reading

- [RFC 4632: CIDR](https://datatracker.ietf.org/doc/html/rfc4632)
- [RFC 3021: /31 point-to-point links](https://datatracker.ietf.org/doc/html/rfc3021)
- [Python `ipaddress` module](https://docs.python.org/3/library/ipaddress.html)
