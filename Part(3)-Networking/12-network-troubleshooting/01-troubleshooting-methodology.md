# Network Troubleshooting Methodology

## Overview

"The network is broken" is a symptom, not a diagnosis. A good troubleshooter works **in a fixed order**, tests one thing at a time, and uses evidence instead of guesses. The most reliable approach follows the layers of the network stack from the bottom up (or splits the problem in half and goes toward the faulty part).

## Key concepts

### The method

1. **Define the problem precisely.** What fails? From where? To where? Since when? Always, or sometimes? What changed recently?
2. **Reproduce it** and note exact error messages.
3. **Scope it.** One host or many? One destination or all? One protocol or all? IPv4 or IPv6? Inside or outside the network?
4. **Test layer by layer** (below).
5. **Form a hypothesis** and test it with a command that can **prove it wrong**.
6. **Change one thing at a time.** Undo changes that did not help.
7. **Verify the fix** from the user's point of view, and monitor.
8. **Document** the cause, the fix, and how to detect it earlier.

### Layer-by-layer checklist

Start at the client and move outward. Stop at the first layer that fails.

| Step | Question | Test | Healthy result |
|------|----------|------|----------------|
| 1. Link | Is the interface up with a link? | `ip -br link`, `ethtool IFACE` | `UP`, `LOWER_UP`, link detected |
| 2. Address | Does it have the correct IP, mask? | `ip -br addr` | Expected address (not `169.254.x.x`) |
| 3. Local stack | Does the IP stack work? | `ping -c1 127.0.0.1` and `ping -c1 <own IP>` | Replies |
| 4. Gateway | Can I reach the default gateway? | `ip route`, `ping -c2 <gateway>`, `ip neigh` | Replies; neighbor `REACHABLE` |
| 5. Remote IP | Can I reach a remote **IP**? | `ping -c2 203.0.113.9`, `traceroute -n` | Replies; path looks right |
| 6. DNS | Does the **name** resolve? | `getent hosts name`, `dig name` | Correct address |
| 7. Port | Can I connect to the **port**? | `nc -zv host 443`, `curl -v` | "succeeded" / TLS handshake |
| 8. Service | Does the **application** answer correctly? | `curl`, protocol-specific client, server logs | Correct response |

```console
$ ip -br link
$ ip -br addr
$ ip route
$ ping -c 2 192.0.2.1              # gateway
$ ping -c 2 203.0.113.9            # remote IP (bypasses DNS)
$ getent hosts www.example.com     # DNS (as applications see it)
$ nc -zv www.example.com 443       # TCP port
$ curl -sv https://www.example.com -o /dev/null
```

### How to read the results

| Observation | Probable cause |
|-------------|----------------|
| Can ping IPs but not names | DNS problem |
| Can ping gateway but not beyond | Gateway/routing/NAT/upstream problem |
| Cannot ping gateway | Link, VLAN, subnet mask, ARP, switch port, gateway down |
| Ping works, TCP port fails with **refused** | Host reachable, service not listening (or reject rule) |
| Ping works, TCP port **times out** | Firewall dropping, wrong route for return traffic, security group |
| Ping fails but service works | ICMP is blocked. Not an outage |
| Works for small data, hangs on large | MTU / fragmentation problem |
| Works sometimes | Packet loss, load balancer with a bad backend, two default routes, DNS round robin with a dead address, duplicate IP |
| Works from A, fails from B | Source-specific firewall rule, routing, VLAN, ACL |
| Works by IP, fails by name only for one app | `/etc/hosts`, search domain, IPv6 vs IPv4 preference, proxy settings |

### Divide and conquer

For multi-hop problems (client → proxy → app → database), test from the **middle**: can the app reach the database? Can the client reach the proxy? Halve the search space with each test.

### Compare working and failing cases

A working host next to a failing one is the best clue. Compare: IP configuration, routes, DNS, firewall rules, software versions, time, and logs.

### Both directions

A connection needs a working **return path**. Test from the other side as well (`tcpdump` on both ends). Many "one-way" failures are routing or firewall rules on the return path.

### Evidence tools

| Question | Tool |
|----------|------|
| What did the system log at that time? | `journalctl --since`, `/var/log` |
| Does the packet leave? Arrive? | `tcpdump` on client and server |
| Is a firewall dropping? | `nft list ruleset`, counters, `iptables -L -v -n`, log rules |
| Which route is used? | `ip route get` |
| What does the app see? | `strace -e trace=network`, application logs |
| Is it slow or lossy? | `mtr`, `ping -c 100`, `ss -ti` |
| Which process owns the socket? | `ss -tulpn` |

### Common error messages

| Message | Meaning | Look at |
|---------|---------|---------|
| `Network is unreachable` | No route to that network | `ip route` |
| `No route to host` | ARP failed or ICMP "host unreachable" received | `ip neigh`, VLAN, firewall reject |
| `Connection refused` | Host answered with RST: nothing listening | `ss -tlnp` on the server, service status |
| `Connection timed out` | No response | Firewall drop, routing, host down |
| `Name or service not known` / `Temporary failure in name resolution` | DNS failure | `resolvectl status`, `dig` |
| `Connection reset by peer` | Remote closed abruptly | Application, proxy, firewall/IDS, TLS mismatch |
| `Destination Host Unreachable` | A router (or your host) reports no path | `ip route`, ARP |
| `SSL certificate problem` | TLS validation failed | Clock, CA bundle, hostname, expiry |

## Common pitfalls

- **Jumping to conclusions.** "It must be the firewall" without evidence wastes time.
- **Testing with the wrong tool.** `ping` working does not prove TCP works. `dig` working does not prove the application resolves the name.
- **Changing several things at once.** You will not know what fixed it.
- **Ignoring IPv6.** A name may resolve to an IPv6 address that does not work, while IPv4 works.
- **Forgetting the proxy.** Environment variables (`http_proxy`) and application proxy settings change behavior.
- **Not checking the server side.** Many problems are in the service, its bind address, or its firewall, not in the network.
- **Trusting cached results.** DNS caches, ARP caches, browser caches, and connection reuse hide changes.
- **Assuming the problem is network.** Check CPU, memory, disk, and application errors on both ends.
- **Not capturing evidence before restarting** services or interfaces.

## Further reading

- [Brendan Gregg: Linux performance tools](https://www.brendangregg.com/linuxperf.html)
- [`ip(8)`](https://man7.org/linux/man-pages/man8/ip.8.html)
- [Red Hat: troubleshooting network issues](https://docs.redhat.com)
