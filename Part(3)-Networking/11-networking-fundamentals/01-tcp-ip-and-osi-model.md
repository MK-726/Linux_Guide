# TCP/IP Networking Concepts and the OSI Model

## Overview

Networks are built in **layers**. Each layer solves one problem and relies on the layer below. When you troubleshoot, thinking in layers tells you where to look: cable, address, route, port, or application.

## Key concepts

### The OSI model and the TCP/IP model

| OSI layer | Name | TCP/IP layer | Examples | Data unit |
|-----------|------|--------------|----------|-----------|
| 7 | Application | **Application** | HTTP, DNS, SSH, SMTP | Data / message |
| 6 | Presentation | | TLS, encoding | |
| 5 | Session | | Sessions, RPC | |
| 4 | Transport | **Transport** | TCP, UDP | Segment (TCP), datagram (UDP) |
| 3 | Network | **Internet** | IP, ICMP | Packet |
| 2 | Data link | **Link** | Ethernet, Wi-Fi, ARP, VLAN | Frame |
| 1 | Physical | | Cables, radio, NIC | Bits |

The **OSI model** is a teaching and troubleshooting tool with 7 layers. The **TCP/IP model** is what the internet actually uses, with 4 layers. People often say "layer 3" for IP and "layer 2" for Ethernet.

### Encapsulation

Each layer wraps the data from the layer above with its own header.

```
Application data:                       [ HTTP request ]
Transport (TCP):                  [ TCP hdr | HTTP request ]
Internet (IP):             [ IP hdr | TCP hdr | HTTP request ]
Link (Ethernet): [ Eth hdr | IP hdr | TCP hdr | HTTP request | Eth trailer ]
```

The receiver removes the headers in reverse order.

### What each layer needs (a simple checklist)

| Layer | Question | Linux view |
|-------|----------|------------|
| Link | Is the interface up and connected? | `ip link`, `ethtool` |
| Internet | Does it have an IP address, a route, a gateway? | `ip addr`, `ip route` |
| Transport | Is the port open and reachable? | `ss`, `nc`, firewall |
| Application | Does the service answer correctly? | `curl`, logs |

### IP, ICMP

- **IP** (Internet Protocol) addresses hosts and routes packets. It is **connectionless** and offers **no delivery guarantee**.
- **ICMP** carries control and error messages: echo request/reply (`ping`), destination unreachable, time exceeded (`traceroute`). ICMP is not TCP or UDP and has no ports.

### TCP and UDP

| | **TCP** | **UDP** |
|---|---------|---------|
| Connection | Yes (handshake) | No |
| Reliable and ordered | Yes (retransmits lost data) | No |
| Flow and congestion control | Yes | No |
| Overhead | Higher | Low |
| Typical use | Web, SSH, email, databases, file transfer | DNS queries, DHCP, NTP, VoIP, video, VPNs, QUIC (HTTP/3) |

### The TCP three-way handshake

```
Client                        Server
  │ ── SYN ──────────────────► │   "I want to connect"
  │ ◄───────────── SYN, ACK ── │   "OK, I'm listening"
  │ ── ACK ──────────────────► │   "Connection established"
  │ ══════ data both ways ═══ │
  │ ── FIN ... ACK ... FIN ──► │   Connection closed
```

What you see when the handshake fails:

| Result of the SYN | Meaning |
|-------------------|---------|
| SYN-ACK comes back | Port open, service listening |
| **RST** comes back | Host reachable, **nothing listening** (or a reject firewall rule): "Connection refused" |
| **No answer** | Packet dropped by a firewall, wrong route, or host down: "Connection timed out" |

### Ports and sockets

A **port** (0–65535) identifies a service on a host. A **socket** is the combination of IP address, port, and protocol. A TCP connection is identified by four values: source IP, source port, destination IP, destination port.

| Range | Name | Use |
|-------|------|-----|
| 0–1023 | **Well-known** (system) ports | Standard services. Binding needs root or a capability |
| 1024–49151 | Registered ports | Applications (for example 3306 MySQL) |
| 49152–65535 | Dynamic / ephemeral | Short-lived client ports (Linux uses 32768–60999 by default) |

```console
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999
$ grep -E '^(ssh|http|https|domain)\s' /etc/services
ssh             22/tcp
domain          53/tcp
domain          53/udp
http            80/tcp
https           443/tcp
```

### Common ports

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 25, 465, 587 | TCP | SMTP (mail) |
| 53 | TCP/UDP | DNS |
| 67, 68 | UDP | DHCP server, client |
| 80, 443 | TCP | HTTP, HTTPS |
| 123 | UDP | NTP |
| 143, 993 | TCP | IMAP, IMAPS |
| 389, 636 | TCP | LDAP, LDAPS |
| 445 | TCP | SMB |
| 514 | TCP/UDP | Syslog |
| 2049 | TCP/UDP | NFS |
| 3306 | TCP | MySQL / MariaDB |
| 5432 | TCP | PostgreSQL |

### Other protocols you will meet

| Protocol | Layer | Purpose |
|----------|-------|---------|
| ARP | 2/3 | Find the MAC address for an IPv4 address |
| NDP (ICMPv6) | 3 | The IPv6 equivalent of ARP and more |
| VLAN (802.1Q) | 2 | Separate networks on the same wire |
| NAT | 3/4 | Rewrite addresses (home routers, cloud) |
| BGP, OSPF | 3 | Routing protocols |

### Latency, bandwidth, MTU

- **Bandwidth**: how much data per second (Mbit/s).
- **Latency**: how long one packet takes (ms). Also called RTT (round-trip time) when measured both ways.
- **MTU** (maximum transmission unit): the largest packet on a link. Ethernet default is **1500** bytes. VPNs and tunnels reduce it. Too-large packets get fragmented or dropped.

## Commands

```console
$ ip -br addr                  # layer 3 addresses
$ ip -br link                  # layer 2 state
$ ss -tuln                     # listening ports (layer 4)
$ cat /proc/net/dev            # per-interface counters
$ cat /etc/services | less     # name-to-port list
```

## Common pitfalls

- **Mixing up layers.** "The network is down" can mean cable, IP, routing, DNS, firewall, or a stopped service. Test layer by layer (see [Troubleshooting Methodology](../12-network-troubleshooting/01-troubleshooting-methodology.md)).
- **Confusing "refused" and "timed out".** Refused = reached the host, nothing listening. Timed out = no answer at all.
- **Assuming ICMP blocked means host down.** Many networks drop ping but allow other traffic.
- **UDP "connections".** UDP has no connection. Tools that show "connected" UDP sockets are showing local state only.
- **Ignoring MTU** in tunnels and VPNs.

## Further reading

- [RFC 1122: Requirements for Internet Hosts](https://datatracker.ietf.org/doc/html/rfc1122)
- [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [`ip(7)`](https://man7.org/linux/man-pages/man7/ip.7.html)
- [IANA service name and port registry](https://www.iana.org/assignments/service-names-port-numbers/)
