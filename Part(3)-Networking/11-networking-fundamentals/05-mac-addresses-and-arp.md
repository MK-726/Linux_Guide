# MAC Addresses and ARP

## Overview

IP addresses identify hosts across networks. On the **local network** (the same Ethernet segment or VLAN), devices actually deliver frames using **MAC addresses**. **ARP** (Address Resolution Protocol) maps an IPv4 address to a MAC address so the host can send the frame to the right device.

## Key concepts

### MAC addresses

- 48 bits, written as six hex pairs: `52:54:00:12:34:56`.
- The first 3 bytes are the **OUI** (vendor ID). The rest is assigned by the vendor.
- Special bits in the first byte:

| Bit | Meaning |
|-----|---------|
| Least significant bit of the first byte = 1 | **Multicast** (broadcast `ff:ff:ff:ff:ff:ff` is a special case) |
| Second-least significant bit = 1 | **Locally administered** (set by software, not the vendor). Common for VMs, containers, and privacy-randomized Wi-Fi |

MAC addresses work only **within one local network segment**. A router replaces them at every hop.

### How ARP works

Host A (`192.0.2.10`) wants to send to host B (`192.0.2.20`) on the same subnet:

```
1. A checks its ARP cache. No entry for 192.0.2.20.
2. A broadcasts (ff:ff:ff:ff:ff:ff):  "Who has 192.0.2.20? Tell 192.0.2.10"
3. B replies (unicast to A):          "192.0.2.20 is at 52:54:00:aa:bb:cc"
4. A stores it in the cache and sends frames to that MAC.
```

For a **remote** destination, A ARPs for the **default gateway's** MAC and sends the frame to the gateway. The IP destination stays the remote host.

### The neighbor table

Linux keeps results in the **neighbor cache** (ARP table for IPv4, NDP for IPv6). Entries have states:

| State | Meaning |
|-------|---------|
| `REACHABLE` | Recently confirmed |
| `STALE` | Not recently confirmed. Still used, and checked on next use |
| `DELAY`, `PROBE` | Being re-verified |
| `INCOMPLETE` | ARP request sent, no reply yet |
| `FAILED` | No reply. The host is down or unreachable on layer 2 |
| `PERMANENT` | Static entry you added |

### IPv6 neighbor discovery

IPv6 does not use ARP. It uses **NDP** with ICMPv6 messages (neighbor solicitation and advertisement, router solicitation and advertisement). The commands are the same (`ip neigh`).

### Gratuitous ARP

A host may send an ARP announcement for **its own** address, to update others' caches after an IP or MAC change (failover with keepalived, VM migration) or to detect duplicate addresses.

## Commands

```console
$ ip neigh show
192.0.2.1 dev enp3s0 lladdr 00:1a:2b:3c:4d:5e REACHABLE
192.0.2.20 dev enp3s0 lladdr 52:54:00:aa:bb:cc STALE
192.0.2.30 dev enp3s0  FAILED
fe80::1 dev enp3s0 lladdr 00:1a:2b:3c:4d:5e router REACHABLE

$ ip neigh show dev enp3s0
$ ip -4 neigh show to 192.0.2.1
```

| Command | Purpose |
|---------|---------|
| `ip neigh show` | Show the neighbor table |
| `ip neigh add 192.0.2.50 lladdr aa:bb:cc:dd:ee:ff dev enp3s0 nud permanent` | Add a static entry (**root**) |
| `ip neigh del 192.0.2.50 dev enp3s0` | Delete an entry |
| `ip neigh flush dev enp3s0` | Clear dynamic entries |
| `ip neigh replace ...` | Add or update |
| `arping -I enp3s0 192.0.2.20` | Send ARP requests (`iputils-arping`; the `arping` tool) |
| `arping -D -I enp3s0 192.0.2.50` | **Duplicate address detection**: reply means someone else has that IP |
| `arping -U -I enp3s0 -c 3 192.0.2.10` | Send gratuitous ARP |
| `ip link show enp3s0` | See your own MAC (`link/ether`) |
| `cat /sys/class/net/enp3s0/address` | Your MAC |

```console
$ arping -c 3 -I enp3s0 192.0.2.20
ARPING 192.0.2.20 from 192.0.2.10 enp3s0
Unicast reply from 192.0.2.20 [52:54:00:AA:BB:CC]  0.612ms
Unicast reply from 192.0.2.20 [52:54:00:AA:BB:CC]  0.581ms
```

> [!NOTE]
> The old `arp -a` command (package `net-tools`) still works but is deprecated. Use `ip neigh`.

### Find the vendor of a MAC

The first three bytes (`52:54:00` is QEMU/KVM) can be looked up in the IEEE OUI list. This is useful to identify unknown devices on a network.

### Capture ARP traffic

```console
# tcpdump -i enp3s0 -nn -e arp
10:15:01.123 52:54:00:12:34:56 > ff:ff:ff:ff:ff:ff, ARP, Request who-has 192.0.2.20 tell 192.0.2.10
10:15:01.124 52:54:00:aa:bb:cc > 52:54:00:12:34:56, ARP, Reply 192.0.2.20 is-at 52:54:00:aa:bb:cc
```

## Common pitfalls

- **`FAILED` or `INCOMPLETE` entries.** The target does not answer ARP: it is down, on another VLAN, firewalled on layer 2, or the cable/switch port has a problem. A failed ARP shows as "No route to host" or "Destination Host Unreachable".
- **Duplicate IP addresses.** Two devices with one IP cause intermittent connectivity. Use `arping -D`, and check which MAC answers.
- **Stale neighbor entries after a failover or replacement.** The old MAC stays cached for a while. Flush it: `ip neigh flush to 192.0.2.20`.
- **ARP spoofing / poisoning.** A malicious host announces a false MAC to intercept traffic on the LAN. Defenses: switch features (dynamic ARP inspection, port security), static entries for critical hosts, VLAN separation, encryption (TLS, SSH).
- **Changing a MAC address** with the interface up. Bring the link down first.
- **Cloned VMs with the same MAC address.** Generate new MACs when cloning.
- **Large flat networks** produce lots of broadcast traffic. Segment with VLANs and routing.
- **Cloud networks** often handle ARP/NDP in the virtual network. The MAC you see may belong to the virtual router.

## Further reading

- [`ip-neighbour(8)`](https://man7.org/linux/man-pages/man8/ip-neighbour.8.html)
- [`arping(8)`](https://man7.org/linux/man-pages/man8/arping.8.html)
- [RFC 826: ARP](https://datatracker.ietf.org/doc/html/rfc826)
- [RFC 4861: IPv6 Neighbor Discovery](https://datatracker.ietf.org/doc/html/rfc4861)
