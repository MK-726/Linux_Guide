# The ip and ss Commands

## Overview

`ip` (from the **iproute2** package) and `ss` replace the older `net-tools` commands. They are available on every current distribution, show more information, and are the tools used in this guide.

| Old command (`net-tools`) | Modern replacement |
|---------------------------|--------------------|
| `ifconfig` | `ip addr`, `ip link` |
| `route` | `ip route` |
| `arp` | `ip neigh` |
| `netstat` | `ss` (and `ip -s link`, `ip route`) |
| `iptunnel`, `vconfig`, `brctl` | `ip tunnel`, `ip link add ... type vlan`, `bridge` / `ip link ... type bridge` |
| `nameif`, `mii-tool` | `ip link`, `ethtool` |

## Key concepts: ip

`ip [OPTIONS] OBJECT COMMAND`

| Object | Purpose | Details in |
|--------|---------|-----------|
| `link` | Interfaces (layer 2) | [Network Interfaces](02-network-interfaces.md) |
| `addr` | IP addresses | [IPv4 and IPv6 Addressing](03-ipv4-and-ipv6-addressing.md) |
| `route` | Routing table | [Routing](06-routing-and-default-gateways.md) |
| `neigh` | ARP / NDP cache | [MAC Addresses and ARP](05-mac-addresses-and-arp.md) |
| `rule` | Policy routing rules | [Routing](06-routing-and-default-gateways.md) |
| `netns` | Network namespaces | Below |
| `maddr` | Multicast addresses | |
| `monitor` | Watch changes live | Below |

### Useful global options

| Option | Effect |
|--------|--------|
| `-br` | **Brief**, table-style output |
| `-c` | Color |
| `-4` / `-6` | Only IPv4 / IPv6 |
| `-s` | Statistics (use `-s -s` for more detail) |
| `-d` | Details |
| `-j` / `-j -p` | JSON / pretty JSON (for scripts: `ip -j addr \| jq`) |
| `-o` | One line per object (good with `grep`, `awk`) |
| `-n NS` | Run in network namespace `NS` |

Commands can be abbreviated: `ip a`, `ip r`, `ip l`, `ip n`. In documentation and scripts, write them out.

## Commands: ip

```console
$ ip -br addr                              # interfaces with addresses
$ ip -br -c link                           # link state with colors
$ ip -s link show enp3s0                   # counters
$ ip -4 -o addr show | awk '{print $2, $4}'
enp3s0 192.0.2.10/24
$ ip -j addr show enp3s0 | jq -r '.[0].addr_info[] | select(.family=="inet") | .local'
192.0.2.10
$ ip route get 203.0.113.9                 # which route would be used
```

### Watch changes live

```console
$ ip monitor                               # addresses, routes, links, neighbors
$ ip monitor link address route
```

Useful to see exactly when an interface flaps or DHCP changes a route.

### Network namespaces

A **network namespace** is an isolated copy of the network stack (interfaces, routes, firewall). Containers use them.

```console
# ip netns add test
# ip netns exec test ip -br addr               # run a command inside
# ip link add veth0 type veth peer name veth1
# ip link set veth1 netns test
# ip addr add 198.51.100.1/24 dev veth0 && ip link set veth0 up
# ip -n test addr add 198.51.100.2/24 dev veth1 && ip -n test link set veth1 up
# ip netns exec test ping -c 2 198.51.100.1
# ip netns del test
```

`ip netns list` shows the named namespaces. Docker and Podman namespaces are not under `/run/netns` by default. Use `nsenter -t PID -n`.

## Key concepts: ss

`ss` (socket statistics) shows **sockets**: listening ports and connections. It reads the kernel directly and is much faster than `netstat`.

### Options

| Option | Meaning |
|--------|---------|
| `-t` | TCP |
| `-u` | UDP |
| `-x` | Unix sockets |
| `-l` | **Listening** sockets only |
| `-a` | All (listening and non-listening) |
| `-n` | **Numeric** (no name resolution; faster and clearer) |
| `-p` | Show the **process** (needs root for other users' processes) |
| `-s` | Summary statistics |
| `-o` | Timer information |
| `-i` | Internal TCP information (RTT, congestion window, retransmits) |
| `-e` | Extended information (UID, inode) |
| `-4` / `-6` | Only IPv4 / IPv6 |
| `-H` | No header (for scripts) |

The famous combination: **`ss -tulpn`** (TCP and UDP listening sockets, numeric, with processes).

## Commands: ss

```console
# ss -tulpn
Netid State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
tcp   LISTEN 0      128    0.0.0.0:22         0.0.0.0:*         users:(("sshd",pid=812,fd=3))
tcp   LISTEN 0      511    0.0.0.0:80         0.0.0.0:*         users:(("nginx",pid=1020,fd=6))
tcp   LISTEN 0      244    127.0.0.1:5432     0.0.0.0:*         users:(("postgres",pid=1130,fd=5))
tcp   LISTEN 0      128    [::]:22            [::]:*            users:(("sshd",pid=812,fd=4))
udp   UNCONN 0      0      127.0.0.53%lo:53   0.0.0.0:*         users:(("systemd-resolve",pid=620,fd=13))
```

### Reading the Local Address

| Local address | Reachable from |
|---------------|----------------|
| `0.0.0.0:22` | All IPv4 addresses of the host |
| `[::]:22` or `*:22` | All IPv6 addresses (and usually IPv4 as well) |
| `127.0.0.1:5432` | **Only the host itself** |
| `192.0.2.10:8080` | Only that one address |
| `127.0.0.53%lo:53` | Local stub resolver on the loopback |

### States

| State | Meaning |
|-------|---------|
| `LISTEN` | Waiting for connections |
| `ESTAB` | Connection established |
| `SYN-SENT` | We sent SYN, waiting for the answer (blocked or slow remote) |
| `SYN-RECV` | Received SYN, waiting for the final ACK. Many may indicate a SYN flood |
| `TIME-WAIT` | Connection closed, waiting to ensure no stray packets. Normal in large numbers on busy clients |
| `CLOSE-WAIT` | Remote closed, **the local application has not closed its side**. Many = application bug (connection leak) |
| `FIN-WAIT-1/2`, `LAST-ACK` | Closing sequence |

### Recv-Q and Send-Q

| For | Recv-Q | Send-Q |
|-----|--------|--------|
| `LISTEN` socket | Connections waiting to be accepted | Maximum backlog |
| Connected socket | Bytes received, not yet read by the application | Bytes sent, not yet acknowledged |

A **growing Recv-Q** on a listening socket means the application cannot accept connections fast enough.

### Filters and examples

```console
$ ss -tn state established                       # established TCP connections
$ ss -tn state established '( dport = :443 or sport = :443 )'
$ ss -tn dst 203.0.113.9                         # connections to one remote host
$ ss -tn sport = :22                             # local port 22
$ ss -tln '( sport = :80 or sport = :443 )'
$ ss -s                                          # totals by state
Total: 421
TCP:   96 (estab 41, closed 38, orphaned 0, timewait 36)
$ ss -ti dst 203.0.113.9                         # RTT, retransmits, congestion window
$ ss -tnp | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head    # top peers
$ ss -xl                                         # listening Unix sockets
$ ss -tn | awk 'NR>1 {c[$1]++} END {for (s in c) print s, c[s]}'    # count by state
```

State names with a space (`state established`) and filter expressions are `ss` syntax. Quote parentheses for the shell.

### Find out who uses a port

```console
# ss -tlnp 'sport = :8080'
# fuser 8080/tcp
# lsof -i :8080
```

## Common pitfalls

- **Forgetting `-n`.** Name resolution slows `ss` and obscures ports. Use `-n`.
- **`-p` shows no process.** Run as root (or with `sudo`) to see other users' processes.
- **Using `ifconfig` and `netstat` output from old tutorials.** Install `net-tools` only if a legacy tool requires it.
- **Reading `127.0.0.1` listeners as "exposed".** They are local only. A service bound to `127.0.0.1` cannot be reached from the network without a proxy or tunnel.
- **Ignoring `CLOSE-WAIT`.** It points to a leaking application, not a network problem.
- **Confusing `ip a` output.** Interface flags (`UP`, `LOWER_UP`) and address flags (`dynamic`, `tentative`, `dadfailed`) are different things. `dadfailed` means a duplicate address was detected.
- **`ip addr add` without bringing the link up.** The address exists but the interface is down.
- **`ip` changes are runtime only.** They vanish at reboot.

## Further reading

- [`ip(8)`](https://man7.org/linux/man-pages/man8/ip.8.html)
- [`ss(8)`](https://man7.org/linux/man-pages/man8/ss.8.html)
- [iproute2 project](https://wiki.linuxfoundation.org/networking/iproute2)
- [`ip-netns(8)`](https://man7.org/linux/man-pages/man8/ip-netns.8.html)
