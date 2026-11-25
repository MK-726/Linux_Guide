# ss and netstat

## Overview

`ss` shows **sockets**: which ports are listening, which connections exist, and what state they are in. `netstat` is the older tool with the same purpose, part of the deprecated `net-tools` package. Use `ss` on modern systems. This page focuses on **troubleshooting** with these tools. For syntax and option details see [The ip and ss Commands](../11-networking-fundamentals/10-ip-and-ss.md).

## Key concepts: netstat to ss

| Old command | Modern equivalent |
|-------------|-------------------|
| `netstat -tulpn` | `ss -tulpn` |
| `netstat -an` | `ss -an` |
| `netstat -s` | `ss -s` (socket totals) and `nstat -az` (protocol counters) |
| `netstat -rn` | `ip route` |
| `netstat -i` | `ip -s link` |
| `netstat -g` | `ip maddr` |
| `netstat -anp \| grep :80` | `ss -tnp 'sport = :80 or dport = :80'` |

If `netstat` is not installed, you can install `net-tools`, but prefer learning `ss`.

## Commands and scenarios

### Scenario 1: is my service listening, and on which address?

```console
# ss -tlnp | grep -E ':80|:443'
LISTEN 0 511 0.0.0.0:80   0.0.0.0:* users:(("nginx",pid=1020,fd=6))
LISTEN 0 511 0.0.0.0:443  0.0.0.0:* users:(("nginx",pid=1020,fd=7))
```

| Result | Meaning |
|--------|---------|
| No line | Not listening. Service stopped or crashed. Check `systemctl status`, logs |
| `127.0.0.1:PORT` | Local only. Remote clients cannot connect |
| `0.0.0.0:PORT` / `[::]:PORT` | All addresses. Reachability then depends on the firewall |
| Wrong port | Check the service configuration |
| Process is not what you expect | Another program took the port |

### Scenario 2: "Address already in use"

```console
# ss -tlnp 'sport = :8080'
LISTEN 0 128 0.0.0.0:8080 0.0.0.0:* users:(("java",pid=2210,fd=45))
```

Find the owner, then stop it or change the port. A recently closed connection in `TIME-WAIT` does not usually block binding if the application sets `SO_REUSEADDR` (most servers do).

### Scenario 3: who is connected to my server?

```console
$ ss -tn state established '( sport = :443 )' | awk 'NR>1 {print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
    312 203.0.113.9
     41 198.51.100.4
$ ss -tn state established '( sport = :443 )' | wc -l
```

For IPv6 addresses (which contain colons) parse the last colon: `sed 's/:[0-9]*$//'`.

### Scenario 4: how many connections in which state?

```console
$ ss -s
Total: 1250
TCP:   2310 (estab 1100, closed 900, orphaned 4, timewait 880)

$ ss -tan | awk 'NR>1 {c[$1]++} END {for (s in c) print c[s], s}' | sort -rn
1100 ESTAB
 880 TIME-WAIT
  20 CLOSE-WAIT
   5 SYN-RECV
```

| State count high | Interpretation |
|------------------|----------------|
| Many `TIME-WAIT` | Normal for busy clients and proxies making many short connections. It can exhaust ephemeral ports (see below) |
| Many `CLOSE-WAIT` | **Application bug**: it is not closing sockets after the peer closed. Restart as a workaround, fix the code |
| Many `SYN-RECV` | SYN flood or slow clients. Check `nstat -az` for SYN-related counters, consider SYN cookies (`net.ipv4.tcp_syncookies=1`, default on) |
| Many `SYN-SENT` | This host cannot reach the destination (blocked, down) |
| Many `ESTAB` | Heavy load or leaked connections. Compare with normal levels |

### Scenario 5: connections queued or slow

```console
$ ss -ltn
State  Recv-Q Send-Q Local Address:Port
LISTEN 129    128    0.0.0.0:8080        # Recv-Q > Send-Q: accept queue full
```

On a `LISTEN` socket, `Recv-Q` is the number of connections waiting to be accepted and `Send-Q` is the backlog limit. If `Recv-Q` reaches the limit, new connections are dropped. Check the application and `net.core.somaxconn`.

```console
$ nstat -az TcpExtListenOverflows TcpExtListenDrops
TcpExtListenOverflows     12     0.0
$ ss -ti dst 203.0.113.9              # per-connection metrics
         cubic wscale:7,7 rto:204 rtt:12.3/4.1 cwnd:10 bytes_retrans:5840 retrans:0/4
```

`retrans` and `bytes_retrans` show packet loss on that connection.

### Scenario 6: ephemeral port exhaustion

A host making very many outgoing connections to the same destination can run out of source ports ("Cannot assign requested address").

```console
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999
$ ss -tan state time-wait | wc -l
```

Options: reuse connections (keep-alive, connection pools), widen the port range, add more destination or source addresses. Avoid `tcp_tw_recycle` (removed from the kernel).

### Scenario 7: protocol statistics

```console
$ nstat -az | grep -i -E 'retrans|drop|overflow|reset|abort'
$ cat /proc/net/snmp | head
$ ip -s -s link show enp3s0          # NIC errors and drops
```

Rising `TcpRetransSegs`, `TcpExtTCPTimeouts`, or interface `errors` and `dropped` indicate packet loss.

### netstat (legacy) quick reference

```console
$ netstat -tulpn                 # listening ports with PIDs
$ netstat -ant | awk '{print $6}' | sort | uniq -c       # count states
$ netstat -rn                    # routing table
$ netstat -s                     # protocol statistics
```

## Common pitfalls

- **Missing `-n`** makes the output slow and hides ports (names instead of numbers).
- **Missing root** with `-p`: the process column stays empty for processes of other users.
- **Reading `Recv-Q`/`Send-Q` the wrong way.** The meaning differs for listening and connected sockets.
- **Panic over `TIME-WAIT`.** It is a normal protective state. It is a problem only when it exhausts ports.
- **Treating `ss -s` "closed" as errors.** It counts sockets in closed state, not failed connections.
- **Seeing `LISTEN` and assuming reachability.** Firewalls, security groups, and routing still decide whether a client can connect.
- **Containers.** `ss` on the host shows published ports (via docker-proxy or NAT rules), while listeners inside the container are in another network namespace (`nsenter -t PID -n ss -tlnp`).
- **Unix sockets** (`-x`) are separate from TCP/UDP. Many local services (databases, PHP-FPM) use them.

## Further reading

- [`ss(8)`](https://man7.org/linux/man-pages/man8/ss.8.html)
- [`nstat(8)`](https://man7.org/linux/man-pages/man8/nstat.8.html)
- [`tcp(7)`](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [Kernel: IP sysctl](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
