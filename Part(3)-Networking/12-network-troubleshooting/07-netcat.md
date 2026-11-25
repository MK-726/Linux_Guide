# nc (netcat)

## Overview

**netcat** (`nc`) reads and writes data across network connections using TCP or UDP. It is called the "Swiss army knife" of networking: you can **test whether a port is open**, create a quick **listener**, send test data, and transfer files. It is invaluable for checking firewall rules and service reachability without installing a full client.

> [!NOTE]
> Several incompatible versions exist, and options differ. Check which one you have: `nc -h` or `ls -l $(which nc)`.

| Variant | Where | Notes |
|---------|-------|-------|
| **OpenBSD netcat** (`netcat-openbsd`) | Debian, Ubuntu | Listen with `nc -l PORT` |
| **Ncat** (from Nmap) | RHEL-based (`nmap-ncat` package) | Feature rich: TLS (`--ssl`), access control, `-e` options. Listen with `nc -l PORT` |
| **Traditional netcat** (`netcat-traditional`) | Some Debian systems | Listen with `nc -l -p PORT` |

## Commands

### Test whether a TCP port is open

```console
$ nc -zv www.example.com 443
Connection to www.example.com (203.0.113.80) 443 port [tcp/https] succeeded!

$ nc -zv 192.0.2.30 5432
nc: connect to 192.0.2.30 port 5432 (tcp) failed: Connection refused

$ nc -zv -w 3 192.0.2.30 5432
nc: connect to 192.0.2.30 port 5432 (tcp) timed out: Operation now in progress
```

| Option | Meaning |
|--------|---------|
| `-z` | Zero-I/O mode: just connect and close (scan) |
| `-v` | Verbose |
| `-w SEC` | Timeout |
| `-u` | UDP |
| `-l` | Listen |
| `-p PORT` | Source port (or listening port in the traditional variant) |
| `-k` | Keep listening after a client disconnects |
| `-4` / `-6` | IP version |
| `-N` | Shut down the network connection after end of input (OpenBSD, useful for file transfer) |
| `-s ADDR` | Source address |

Interpretation of the three results (same as in [TCP and the OSI model](../11-networking-fundamentals/01-tcp-ip-and-osi-model.md)):

| Result | Meaning |
|--------|---------|
| `succeeded` | The TCP handshake completed: the port is open and reachable |
| `Connection refused` | The host answered with RST: reachable, **nothing listening** (or a reject rule) |
| `timed out` | No answer: **filtered** (firewall drop), routing problem, or host down |

Scan a few ports:

```console
$ nc -zv 192.0.2.30 20-25 2>&1 | grep succeeded
$ for p in 22 80 443; do nc -zv -w 2 192.0.2.30 $p 2>&1; done
```

### Test a firewall path with a temporary listener

Is the firewall between A and B open for port 9000? Start a listener on the server, connect from the client.

```console
# On the server (B):
$ nc -l 9000                      # traditional variant: nc -l -p 9000
# On the client (A):
$ echo "hello" | nc 192.0.2.30 9000
```

If "hello" appears on the server, the path works. If not, check the firewalls on both hosts (and in the middle) and the **server's listening address**. Stop the listener afterwards.

UDP test:

```console
$ nc -u -l 9000                   # server
$ echo "ping" | nc -u -w1 192.0.2.30 9000      # client
```

UDP gives no "connection" feedback, so a test depends on the listener actually printing the data.

### Talk to a service by hand

```console
$ printf 'GET / HTTP/1.1\r\nHost: www.example.com\r\nConnection: close\r\n\r\n' | nc www.example.com 80
$ echo "QUIT" | nc -w 3 mail.example.com 25            # see the SMTP banner
$ echo | nc -w 3 192.0.2.30 22                          # SSH banner (SSH-2.0-OpenSSH_9.6)
```

Banner grabbing shows what software answers on a port.

### Transfer a file (on trusted networks only)

```console
# Receiver:
$ nc -l 9000 > received.tar.gz            # (traditional: nc -l -p 9000)
# Sender:
$ nc -N 192.0.2.30 9000 < file.tar.gz     # OpenBSD: -N closes after sending; Ncat: --send-only; others: -q 1
$ sha256sum file.tar.gz received.tar.gz    # verify
```

> [!WARNING]
> `nc` sends data **unencrypted and unauthenticated**. Use it only for tests on trusted networks. For real transfers use `scp`, `rsync` over SSH, or `ncat --ssl`.

### Bash alternative (when nc is not installed)

Bash can open TCP connections through `/dev/tcp`:

```console
$ timeout 3 bash -c '</dev/tcp/192.0.2.30/443' && echo open || echo closed
$ exec 3<>/dev/tcp/www.example.com/80; printf 'HEAD / HTTP/1.0\r\n\r\n' >&3; cat <&3; exec 3>&-
```

Other tools: `socat` (more powerful: TLS, Unix sockets, forwarding), `curl telnet://host:port`, `openssl s_client -connect host:443` (for TLS), and `nmap -p PORT host` (authorized scanning only).

### Ncat extras

```console
$ ncat --ssl -l 9443 --keep-open                     # TLS listener
$ ncat --ssl www.example.com 443                     # TLS client
$ ncat -l 9000 --allow 192.0.2.0/24                  # access control
```

## Common pitfalls

- **Wrong variant syntax.** `nc -l 9000` fails on the traditional version (needs `-p`). Check which `nc` you have.
- **Listener bound to the wrong address.** `nc -l 9000` listens on all addresses by default (OpenBSD). Use `-s 127.0.0.1` for local-only tests.
- **Interpreting UDP `-z` scans.** UDP "success" only means no ICMP unreachable was received. The port may still be filtered.
- **Test passes locally, fails remotely.** The firewall on the server or in between, or the service bound to `127.0.0.1`. Test from the real client.
- **Leaving listeners running.** An open `nc -l` is an unauthenticated service. Stop it.
- **Using `-e` (execute a program) remote shells.** Used by attackers; many versions removed it. Do not use it on production systems.
- **Scanning ranges without permission.** Even simple scans can be against policy or law.
- **Timeouts left at default** make scripts hang. Always add `-w`.

## Further reading

- [`nc(1)` (OpenBSD)](https://man.openbsd.org/nc.1)
- [Ncat reference guide](https://nmap.org/ncat/guide/)
- [socat](http://www.dest-unreach.org/socat/)
