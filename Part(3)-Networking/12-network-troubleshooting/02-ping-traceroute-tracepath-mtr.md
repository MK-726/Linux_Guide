# ping, traceroute, tracepath, and mtr

## Overview

These tools test **reachability**, **latency**, **packet loss**, and the **path** packets take. They use ICMP (and sometimes UDP or TCP) probes. Results can be affected by firewalls and rate limits, so interpret them with care.

## Key concepts

- **ICMP echo request/reply** is what `ping` uses.
- **TTL (time to live)**: every router decreases it by 1. When it reaches 0, the router discards the packet and sends back "time exceeded". `traceroute` uses this to discover each hop by increasing the TTL one by one.
- **RTT** (round-trip time) is reported in milliseconds.
- Many networks **filter or rate-limit ICMP**. A missing reply does not always mean a failure.

## Commands

### ping

```console
$ ping -c 4 203.0.113.9
PING 203.0.113.9 (203.0.113.9) 56(84) bytes of data.
64 bytes from 203.0.113.9: icmp_seq=1 ttl=57 time=12.4 ms
64 bytes from 203.0.113.9: icmp_seq=2 ttl=57 time=12.1 ms
64 bytes from 203.0.113.9: icmp_seq=3 ttl=57 time=40.8 ms
64 bytes from 203.0.113.9: icmp_seq=4 ttl=57 time=12.3 ms

--- 203.0.113.9 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 12.1/19.4/40.8/12.4 ms
```

| Option | Purpose |
|--------|---------|
| `-c N` | Send N packets (otherwise it runs until `Ctrl+C`) |
| `-i SEC` | Interval between packets |
| `-W SEC` | Wait time for each reply |
| `-w SEC` | Total deadline |
| `-s BYTES` | Payload size |
| `-I IFACE` or `-I ADDRESS` | Source interface or address |
| `-4` / `-6` | Force IPv4 / IPv6 (`ping6` also exists) |
| `-M do` | Set "Don't Fragment". Used for **MTU testing** |
| `-n` | No name lookup |
| `-f` | Flood (**root**, can overload networks; use with care) |
| `-t N` | Set TTL |

Reading the output:

| Item | Meaning |
|------|---------|
| `ttl=57` | Remaining TTL. Starts at 64, 128, or 255, so 57 suggests about 7 hops (if the start was 64) |
| `time=12.4 ms` | Round-trip time |
| `packet loss` | Percentage with no reply. **Any sustained loss is a problem** |
| `mdev` / jitter | Variation of latency. High values = unstable path |

Error messages:

| Message | Meaning |
|---------|---------|
| `Destination Host Unreachable` (from your own IP) | ARP failed: host not on the local network, or down |
| `Destination Host Unreachable` (from a router IP) | The router has no route or ARP failed on its side |
| `Network is unreachable` | No route in your table |
| `ping: connect: Network is unreachable` | Same |
| `Request timeout` / no output | Dropped (firewall, host down) |
| `Time to live exceeded` | Routing loop or TTL too small |
| `Frag needed and DF set` | MTU problem |

### Test MTU with ping

Find the largest packet that passes without fragmenting (1472 + 28 header bytes = 1500):

```console
$ ping -c 2 -M do -s 1472 203.0.113.9
PING 203.0.113.9 (203.0.113.9) 1472(1500) bytes of data.
1480 bytes from 203.0.113.9: icmp_seq=1 ttl=57 time=13.0 ms

$ ping -c 2 -M do -s 1473 203.0.113.9
ping: local error: message too long, mtu=1500
```

Reduce `-s` until it passes. Path MTU = largest working payload + 28 (IPv4). A tunnel with MTU 1420 would pass 1392 but not 1393.

### traceroute

Shows each router on the path.

```console
$ traceroute -n 203.0.113.9
traceroute to 203.0.113.9 (203.0.113.9), 30 hops max, 60 byte packets
 1  192.0.2.1      0.4 ms   0.3 ms   0.3 ms
 2  198.51.100.1   2.1 ms   2.0 ms   2.2 ms
 3  * * *
 4  203.0.113.1   11.8 ms  11.9 ms  11.7 ms
 5  203.0.113.9   12.3 ms  12.2 ms  12.4 ms
```

| Option | Purpose |
|--------|---------|
| `-n` | Numeric (no DNS). Faster |
| `-I` | Use ICMP echo (instead of UDP) |
| `-T -p 443` | Use TCP SYN to port 443 (**root**). Passes firewalls that block UDP and ICMP |
| `-m N` | Maximum hops |
| `-w SEC` | Wait per probe |
| `-q N` | Probes per hop |
| `-4` / `-6` | IP version |
| `-i IFACE` / `-s ADDR` | Interface / source address |

`* * *` on a hop means no answer from that router within the time. It may be filtering or rate-limiting ICMP. If later hops answer, the path is fine. If **all hops after a point** are `* * *`, traffic is blocked or lost there.

Install: **Debian-based** `apt install traceroute`; **RHEL-based** `dnf install traceroute`.

### tracepath

Similar to traceroute, needs **no root**, and also **discovers the path MTU**.

```console
$ tracepath -n 203.0.113.9
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.0.2.1                        0.431ms
 2:  198.51.100.1                     2.231ms
 3:  no reply
 4:  203.0.113.1                     11.923ms asymm  5
     Resume: pmtu 1500 hops 5 back 57
```

`pmtu 1500` is the path MTU. `asymm N` means the return path differs from the forward path.

### mtr

`mtr` combines `ping` and `traceroute` in a live view with statistics per hop. It is the best tool to find **where loss or delay starts**.

```console
$ mtr -rwzbc 100 203.0.113.9
Start: 2025-03-03T10:30:00+0000
HOST: server01                    Loss%   Snt   Last   Avg  Best  Wrst StDev
  1. AS64500  192.0.2.1            0.0%   100    0.4   0.4   0.3   1.2   0.1
  2. AS64500  198.51.100.1         0.0%   100    2.1   2.2   2.0   4.1   0.3
  3. ???                         100.0   100    0.0   0.0   0.0   0.0   0.0
  4. AS64501  203.0.113.1          0.0%   100   11.8  11.9  11.7  14.2   0.4
  5. AS64501  203.0.113.9          0.0%   100   12.3  12.2  12.1  13.0   0.2
```

| Option | Meaning |
|--------|---------|
| `-r` | Report mode (run, print, exit) |
| `-c N` | Number of probes |
| `-w` | Wide report (full host names) |
| `-n` | No DNS |
| `-z` | Show AS numbers |
| `-b` | Show both name and IP |
| `-T -P 443` | Use TCP to port 443 |
| `-u` | Use UDP |
| `-i SEC` | Interval |

### How to read mtr results (important)

- **Loss at one hop that does not continue to later hops is usually harmless.** Routers often rate-limit ICMP replies directed to themselves while forwarding traffic normally (like hop 3 above: 100% loss but the next hops are fine).
- **Real loss starts at a hop and continues to every hop after it**, including the destination. The first hop with that pattern is where the problem starts.
- **Latency jumps** that persist to the end are real (a long-distance link or congested hop). A high latency at one hop that disappears at the next is ICMP deprioritization.
- Packet loss **only at the final hop** points to the destination host or its firewall.
- Compare forward and reverse paths. Ask the remote side for an `mtr` back to you.

Install: `mtr` (full) or `mtr-tiny` (without GUI). Run as root for the best accuracy of some modes.

## Common pitfalls

- **Blocked ICMP treated as "down".** Verify with a TCP test (`nc -zv`, `curl`, `traceroute -T`).
- **Judging loss from the middle hops of `traceroute`/`mtr`.** See the rules above.
- **Running ping without `-c`** in scripts. It never ends.
- **Testing one packet.** Use at least 20–100 packets to see loss and jitter.
- **Using `ping` on names** and getting an IPv6 address by accident. Use `-4` or `-6` explicitly.
- **Testing from the wrong source address** (multi-homed hosts). Use `-I`.
- **Flood ping** on production or shared links.
- **Assuming traceroute shows the return path.** It shows only the forward path.
- **Misreading TTL as a distance.** Different systems start TTL at different values.

## Further reading

- [`ping(8)`](https://man7.org/linux/man-pages/man8/ping.8.html)
- [`traceroute(8)`](https://man7.org/linux/man-pages/man8/traceroute.8.html)
- [`tracepath(8)`](https://man7.org/linux/man-pages/man8/tracepath.8.html)
- [`mtr(8)`](https://man7.org/linux/man-pages/man8/mtr.8.html)
