# Routing, Neighbors, and Connection Troubleshooting

## Overview

This page puts the tools together. It first shows how to use `ip route` and `ip neigh` as **diagnostic** tools, then walks through the most common connection problems with a **symptom → test → cause** approach.

## Key concepts

Two questions solve most layer-3 problems:

1. **Which way does the kernel send this packet?** → `ip route get DEST`
2. **Can the next hop be reached on the local network?** → `ip neigh` (ARP/NDP)

## Commands

### ip route get

```console
$ ip route get 203.0.113.9
203.0.113.9 via 192.0.2.1 dev enp3s0 src 192.0.2.10 uid 1000
    cache

$ ip route get 10.20.5.9
10.20.5.9 via 192.0.2.254 dev enp3s0 src 192.0.2.10

$ ip route get 198.51.100.77 from 192.0.2.11 iif enp3s0      # test a forwarded packet's path
$ ip -6 route get 2001:db8::1
```

Check: the **gateway** (`via`), the **interface** (`dev`), and the **source address** (`src`). A wrong `src` or `dev` explains many failures (for example, traffic leaving through the wrong NIC with the wrong source IP). An error `RTNETLINK answers: Network is unreachable` means no route.

### ip neigh

```console
$ ip neigh show 192.0.2.1
192.0.2.1 dev enp3s0 lladdr 00:1a:2b:3c:4d:5e REACHABLE
```

| State of the gateway or neighbor | Meaning |
|----------------------------------|---------|
| `REACHABLE` / `STALE` with a MAC | Layer 2 works |
| `FAILED` / `INCOMPLETE` | No ARP/NDP reply. Wrong VLAN or subnet, cable, switch, host down, host firewall dropping ARP |

### The "connection" decision tree

```
Cannot reach host X?
 ├─ ip route get X fails → no route: fix routes or the gateway
 ├─ gateway neighbor FAILED → layer 2 problem
 ├─ ping X works, but TCP port fails?
 │    ├─ "refused" → nothing listening / wrong port / reject rule → check ss -tlnp on X
 │    └─ "timed out" → drop somewhere: host firewall on X, network firewall, security group, routing
 ├─ ping fails and TCP fails → check traceroute/mtr, route on X for the return path
 └─ works by IP but not by name → DNS
```

## Common scenarios

### 1. No connectivity at all

| Test | Finding | Next step |
|------|---------|-----------|
| `ip -br link` | `NO-CARRIER` / `DOWN` | Cable, switch port, VM NIC, driver |
| `ip -br addr` | No address, or `169.254.x.x` | DHCP failure, static config missing |
| `ip route` | No default route | Add/fix gateway, DHCP options |
| `ping <gateway>` | Fails | VLAN, subnet mask, ARP, gateway down |
| `ping 203.0.113.9` | Fails but gateway works | Upstream routing, NAT, provider |

### 2. IP works but names do not

```console
$ ping -c1 203.0.113.9            # works
$ getent hosts www.example.com    # fails
$ resolvectl status               # which DNS servers?
$ dig @192.0.2.53 www.example.com # does the DNS server answer?
```

Causes: no DNS configured or unreachable (UDP/TCP 53 blocked), wrong search domain, VPN changed DNS, local cache issue. See [DNS tools](03-dns-tools.md).

### 3. Reaches some networks, not others

Look for a wrong **netmask**, a missing **static route**, or a **VPN/overlapping subnet**:

```console
$ ip route get 10.20.5.9          # does it use the expected gateway?
$ ip route                        # overlapping prefixes?
```

### 4. "Connection refused"

The host is reachable and replied with RST.

```console
# on the server:
$ ss -tlnp | grep :8080           # listening? on which address?
$ systemctl status myapp
$ journalctl -u myapp -n 50
```

Possible causes: service stopped or crashed, wrong port, bound to `127.0.0.1`, or a firewall **REJECT** rule.

### 5. "Connection timed out"

The SYN got no answer. Capture on **both ends** to locate the drop:

```console
# server:  tcpdump -i any -nn 'host 192.0.2.10 and port 8080'
# client:  nc -zv -w 3 192.0.2.30 8080
```

| Server sees SYN? | Server sends SYN-ACK? | Meaning |
|------------------|----------------------|---------|
| No | — | Dropped before the server: network firewall, routing, security group |
| Yes | No | Host firewall dropping (`nft list ruleset`, `iptables -L -v -n`) or service not listening with a DROP rule |
| Yes | Yes, but the client never sees it | Return path dropped: asymmetric routing, NAT, stateful firewall |

### 6. Works for small requests, hangs on large ones (MTU)

Typical with VPNs, tunnels, PPPoE, and some cloud paths. The TCP handshake and small packets work. Full-size packets are dropped because ICMP "fragmentation needed" messages are blocked.

```console
$ ping -c 2 -M do -s 1472 203.0.113.9          # fails?
$ ping -c 2 -M do -s 1400 203.0.113.9          # works?
$ tracepath -n 203.0.113.9                      # shows pmtu
```

Fixes: lower the MTU on the interface (`ip link set dev wg0 mtu 1380`), enable **MSS clamping** on the router (`iptables ... -j TCPMSS --clamp-mss-to-pmtu`), and allow ICMP type 3 code 4 (and ICMPv6 Packet Too Big).

### 7. Slow or intermittent connections

| Test | What to look for |
|------|------------------|
| `mtr -rwzbc 100 host` | Loss that persists to the destination, jitter |
| `curl -w` timing | Which phase is slow (DNS, connect, TLS, TTFB) |
| `ip -s link`, `ethtool -S` | Errors, drops, CRC |
| `ethtool IFACE` | Speed and duplex mismatch |
| `ss -ti` | Retransmits per connection |
| `nstat -az \| grep -i retrans` | System-wide retransmissions |
| `top`, `vmstat` | CPU and I/O saturation on either end |
| `getent ahosts name` | Both IPv6 and IPv4: a dead IPv6 address causes delays |
| `ip neigh` | Flapping or duplicate IP (changing MAC for one IP) |

### 8. Works locally but not from other hosts

```console
$ ss -tlnp | grep :8080            # 127.0.0.1 only?
$ nft list ruleset | head -50      # or: iptables -L -n -v, firewall-cmd --list-all, ufw status verbose
$ curl -v http://192.0.2.30:8080   # from the client
```

### 9. Asymmetric routing and reverse-path filtering

A host with two interfaces receives a packet on one interface but would answer through the other. Strict `rp_filter` drops it, or a stateful firewall sees only half of the connection.

```console
$ sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.enp4s0.rp_filter
$ ip route get <client IP> from <server IP>
$ ip rule; ip route show table all
```

Fix with correct routes, policy routing (source-based rules), or `rp_filter=2` (loose) where appropriate.

### 10. Duplicate IP address

Symptoms: intermittent connectivity, ARP entries that change.

```console
$ ip neigh show 192.0.2.20                 # MAC changes between checks?
$ arping -D -I enp3s0 -c 3 192.0.2.20      # reply = another host has it
$ journalctl -k | grep -i -E 'duplicate|dad'
```

### 11. Too many connections: conntrack table full

```console
$ dmesg | grep -i conntrack
nf_conntrack: table full, dropping packet
$ conntrack -C                                  # current count (package conntrack)
$ sysctl net.netfilter.nf_conntrack_max net.netfilter.nf_conntrack_count
```

Increase `nf_conntrack_max` (and memory), reduce timeouts, or bypass tracking for high-volume traffic.

## A short toolbox summary

| Need | Command |
|------|---------|
| Interface and link | `ip -br link`, `ethtool IFACE` |
| Address | `ip -br addr` |
| Route chosen | `ip route get DEST` |
| Neighbor (ARP) | `ip neigh` |
| Reachability, loss | `ping`, `mtr` |
| Path MTU | `tracepath`, `ping -M do -s` |
| DNS | `getent hosts`, `dig`, `resolvectl` |
| Port reachability | `nc -zv`, `curl -v` |
| Listening ports | `ss -tlnp` |
| Firewall | `nft list ruleset`, `iptables -L -v -n`, `firewall-cmd --list-all`, `ufw status verbose` |
| Packets | `tcpdump -nn` |
| Counters | `ip -s link`, `nstat -az` |

## Common pitfalls

- **Skipping `ip route get`.** It gives the kernel's real decision in one line.
- **Testing only from one side.** Always check both client and server.
- **Believing "firewall" without proof.** A `tcpdump` on the server tells you whether packets arrive.
- **Missing the other firewalls** (cloud security groups, network ACLs, hardware firewalls, container networking rules).
- **Changing configuration while diagnosing** without recording it.
- **Ignoring IPv6 or IPv4 differences.** Test both explicitly (`-4`, `-6`).
- **Overlooking time-based causes.** Intermittent issues that match cron jobs, backups, or traffic peaks.

## Further reading

- [`ip-route(8)`](https://man7.org/linux/man-pages/man8/ip-route.8.html)
- [`conntrack(8)`](https://man7.org/linux/man-pages/man8/conntrack.8.html)
- [RFC 1191: Path MTU Discovery](https://datatracker.ietf.org/doc/html/rfc1191)
- [Linux Foundation networking wiki](https://wiki.linuxfoundation.org/networking/start)
