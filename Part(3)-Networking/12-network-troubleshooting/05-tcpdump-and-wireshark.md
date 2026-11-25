# Packet Capture and Analysis: tcpdump and Wireshark

## Overview

When logs and simple tools cannot explain a problem, capture the packets. A **packet capture** shows exactly what is sent and received on an interface: it settles arguments about who sent what, and reveals loss, resets, delays, and misconfigurations. **tcpdump** captures on the command line (best on servers). **Wireshark** and **tshark** analyze captures with a powerful interface.

> [!WARNING]
> Captures can contain **passwords, tokens, personal data, and business data** (anything not encrypted). Capture only where you are authorized, keep captures minimal, store them securely, and delete them when finished.

## Key concepts

- Capturing needs **root** (or the `CAP_NET_RAW` capability).
- A **filter** (BPF syntax) selects which packets are captured. Use filters on busy systems.
- Captured data is saved in **pcap** or **pcapng** files that Wireshark reads.
- On a normal host you see only traffic to and from that host (plus broadcast and multicast). Seeing others' traffic needs a **mirror (SPAN) port** or tap.
- Encrypted payloads (TLS, SSH) show only metadata, but the **handshake and flow behavior** are still useful.
- **Offload features** (checksum, TSO/GRO) make local captures show wrong checksums or oversized packets. This is cosmetic.

## Commands

### tcpdump basics

```console
# tcpdump -i enp3s0 -nn                      # all traffic on enp3s0, numeric
# tcpdump -i any -nn port 53                 # DNS on all interfaces
# tcpdump -i enp3s0 -nn -c 20 host 203.0.113.9
# tcpdump -i enp3s0 -nn -w /tmp/capture.pcap port 443    # write to file
$ tcpdump -nn -r /tmp/capture.pcap            # read a file (no root needed)
```

| Option | Meaning |
|--------|---------|
| `-i IFACE` | Interface (`any` = all) |
| `-nn` | No name or port-name resolution. Always use |
| `-c N` | Stop after N packets |
| `-w FILE` | Write raw packets to a file |
| `-r FILE` | Read a capture file |
| `-s 0` | Capture whole packets (default in new versions is 262144) |
| `-v`, `-vv`, `-vvv` | More protocol detail |
| `-A` | Print payload as ASCII (HTTP and other clear text) |
| `-X` | Payload in hex and ASCII |
| `-e` | Show link-layer (MAC) headers |
| `-tttt` | Readable timestamps with date |
| `-q` | Quieter output |
| `-C SIZE -W N` | Rotate files: new file every SIZE MB, keep N files |
| `-Z root` | Drop privileges to this user after opening |
| `--print` | With `-w`, also print on screen |

### BPF filters

| Filter | Matches |
|--------|---------|
| `host 203.0.113.9` | Traffic to or from the address |
| `src host A`, `dst host B` | Direction |
| `net 192.0.2.0/24` | A subnet |
| `port 443`, `portrange 8000-8100` | Ports |
| `src port 22`, `dst port 80` | Port by direction |
| `tcp`, `udp`, `icmp`, `icmp6`, `arp` | Protocol |
| `ether host 52:54:00:aa:bb:cc` | MAC address |
| `vlan 100` | VLAN |
| `tcp[tcpflags] & tcp-syn != 0` | Packets with the SYN flag |
| `tcp[tcpflags] & tcp-rst != 0` | Resets |
| `'tcp[tcpflags] == tcp-syn'` | SYN only (new connection attempts) |
| `greater 1400`, `less 100` | By packet size |
| `A and B`, `A or B`, `not A` | Combine. Use parentheses (quote them) |

```console
# tcpdump -i enp3s0 -nn 'host 203.0.113.9 and (port 80 or port 443)'
# tcpdump -i enp3s0 -nn 'tcp[tcpflags] & tcp-syn != 0 and dst port 22'    # new SSH connection attempts
# tcpdump -i enp3s0 -nn 'tcp[tcpflags] & tcp-rst != 0'                    # who sends resets
# tcpdump -i enp3s0 -nn icmp                                              # ping and ICMP errors
# tcpdump -i enp3s0 -nn 'not port 22'                                     # hide your own SSH session
```

> [!TIP]
> When you capture over SSH, exclude your SSH session (`not port 22`), or you create a feedback loop with endless output.

### Reading tcpdump output

```
10:41:02.100100 IP 192.0.2.10.51234 > 203.0.113.80.443: Flags [S], seq 1000, win 64240, length 0
10:41:02.112300 IP 203.0.113.80.443 > 192.0.2.10.51234: Flags [S.], seq 5000, ack 1001, win 65535, length 0
10:41:02.112340 IP 192.0.2.10.51234 > 203.0.113.80.443: Flags [.], ack 1, win 502, length 0
```

| Flag | Meaning |
|------|---------|
| `[S]` | SYN |
| `[S.]` | SYN-ACK |
| `[.]` | ACK |
| `[P.]` | PUSH + ACK (data) |
| `[F.]` | FIN + ACK (closing) |
| `[R]`, `[R.]` | **RST** (reset) |

`.` means ACK. This three-line example is a normal TCP handshake with about 12 ms round-trip time.

### What patterns mean

| You see | Meaning |
|---------|---------|
| SYN repeated, **no SYN-ACK** | SYN dropped (firewall, routing) or host down. Capture on the server: does the SYN arrive? |
| SYN then **RST** | Port closed (nothing listening) or reject rule |
| SYN-ACK arrives at the client, client does not answer | Client firewall or local problem |
| SYN arrives at server, SYN-ACK sent, never seen by client | Return path dropped (routing, NAT, firewall) |
| Same data **retransmitted** (`seq` repeated) | Packet loss |
| Many duplicate ACKs | Packets lost or reordered |
| `win 0` (zero window) | Receiver is overloaded or slow to read |
| DNS query without response | DNS server unreachable, blocked, or overloaded |
| ICMP `unreachable - need to frag` | MTU problem |
| ICMP `host unreachable` / `port unreachable` | Router or host reports the destination problem |
| ARP requests without replies | Layer 2 problem, host down, wrong VLAN |
| Long gap between request and response | Server-side processing delay (application), not the network |

### Capture for a later analysis

```console
# tcpdump -i any -nn -s 0 -C 100 -W 5 -w /var/tmp/cap.pcap 'host 203.0.113.9'     # rotate 5 x 100 MB
# timeout 60 tcpdump -i enp3s0 -nn -w /tmp/cap.pcap port 8080                     # 60 seconds only
```

On very busy links, `tcpdump` can drop packets. The summary at the end reports "packets dropped by kernel". Use narrow filters.

Copy the file to your workstation to analyze it: `scp server:/tmp/cap.pcap .`

### tshark and Wireshark

**Wireshark** (GUI) and **tshark** (command line) use richer **display filters** (different syntax from BPF).

```console
$ tshark -r cap.pcap -q -z io,phs                        # protocol hierarchy
$ tshark -r cap.pcap -Y 'http.request' -T fields -e ip.src -e http.host -e http.request.uri
$ tshark -r cap.pcap -Y 'dns.flags.rcode != 0' -T fields -e dns.qry.name -e dns.flags.rcode
$ tshark -r cap.pcap -Y 'tcp.analysis.retransmission' | head
$ tshark -r cap.pcap -q -z conv,tcp                      # conversations
$ tshark -i enp3s0 -f 'port 443' -w /tmp/cap.pcapng       # capture with a capture (BPF) filter
```

Useful Wireshark **display filters**:

| Filter | Shows |
|--------|-------|
| `ip.addr == 203.0.113.9` | Traffic to or from the host |
| `tcp.port == 443` | TCP port |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | New connection attempts |
| `tcp.flags.reset == 1` | Resets |
| `tcp.analysis.retransmission` | Retransmissions |
| `tcp.analysis.flags` | Any TCP problem Wireshark detects |
| `tcp.analysis.zero_window` | Zero window events |
| `dns` / `dns.flags.rcode != 0` | DNS / DNS errors |
| `http.request` / `http.response.code >= 400` | HTTP requests / errors |
| `tls.handshake.type == 1` | TLS Client Hello (shows the SNI) |
| `frame.time_delta > 1` | Gaps of more than 1 second |
| `icmp` | ICMP |

Wireshark helpers: **Follow TCP Stream** (right click a packet), **Statistics → Conversations / I/O Graphs**, **Analyze → Expert Information**, **Statistics → Flow Graph**.

### Decrypting TLS (when you control the client)

Set `SSLKEYLOGFILE=/tmp/keys.log` before starting a browser or `curl` (supported by many clients), then load the file in Wireshark (**Preferences → Protocols → TLS → (Pre)-Master-Secret log**). Treat the key log as highly sensitive.

## Common pitfalls

- **Capturing on the wrong interface.** Check with `ip route get`. Use `-i any` if unsure (note: `any` uses a special header format).
- **No filter on a busy link.** Huge files and dropped packets. Always filter, and set a time or size limit.
- **Filling the disk** with a long capture. Use `-C`, `-W`, and `timeout`.
- **Forgetting `-nn`.** DNS lookups from `tcpdump` itself generate traffic and slow output.
- **Capturing only one side.** For loss and drops, capture **on both ends** and compare. A packet seen leaving the client but not arriving at the server was lost in between.
- **Checksum errors** shown for outgoing local packets are an offload artifact.
- **Mixing BPF and Wireshark display filter syntax.** `-f` / `tcpdump` use BPF. `-Y` / the filter bar use display filters.
- **Capturing sensitive data** without protection or authorization.
- **Time differences between two captures.** Hosts must have synchronized clocks (see [NTP and chrony](../../part-2-system-administration/10-logging-time-and-scheduling/04-time-zones-ntp-and-chrony.md)).
- **Reading the capture from a container host.** Containers' traffic is on veth/bridge interfaces. Capture on `docker0`, `cni0`, or inside the namespace with `nsenter -t PID -n tcpdump`.

## Further reading

- [`tcpdump(8)` and `pcap-filter(7)`](https://www.tcpdump.org/manpages/pcap-filter.7.html)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Wireshark display filter reference](https://www.wireshark.org/docs/dfref/)
- [`tshark(1)`](https://www.wireshark.org/docs/man-pages/tshark.html)
