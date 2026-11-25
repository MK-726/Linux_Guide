# Routing Tables and Default Gateways

## Overview

**Routing** decides where to send each packet. Every Linux host has a **routing table**. For each outgoing packet the kernel looks up the destination in the table and picks the interface and next hop. A **default gateway** is the router that receives everything not matched by a more specific route.

## Key concepts

### How a lookup works

1. The kernel compares the destination IP with every route.
2. It chooses the **most specific** match (the **longest prefix**).
3. If no specific route matches, it uses the **default route** (`0.0.0.0/0` or `::/0`).
4. If there is no default route either: **"Network is unreachable"**.

```console
$ ip route
default via 192.0.2.1 dev enp3s0 proto dhcp src 192.0.2.10 metric 100
10.20.0.0/16 via 192.0.2.254 dev enp3s0
192.0.2.0/24 dev enp3s0 proto kernel scope link src 192.0.2.10
```

| Route | Meaning |
|-------|---------|
| `192.0.2.0/24 dev enp3s0 ... scope link` | **Connected** network. Reach hosts directly (ARP). Added automatically when you assign an address |
| `10.20.0.0/16 via 192.0.2.254` | Static route: send traffic for `10.20.0.0/16` to the router `192.0.2.254` |
| `default via 192.0.2.1` | Default gateway for everything else |

| Field | Meaning |
|-------|---------|
| `via ADDRESS` | Next hop (the gateway). It must be reachable on a connected network |
| `dev NAME` | Outgoing interface |
| `src ADDRESS` | Source address the host uses for this route |
| `proto kernel / dhcp / static / ra` | Who created the route |
| `metric N` | Preference when several routes match equally. **Lower wins** |
| `scope link` | Directly connected |

### Example decisions

| Destination | Matches | Sent to |
|-------------|---------|---------|
| `192.0.2.77` | `192.0.2.0/24` (connected) | Directly via `enp3s0` |
| `10.20.5.9` | `10.20.0.0/16` | Router `192.0.2.254` |
| `203.0.113.9` | `default` | Gateway `192.0.2.1` |

### Linux as a router

By default Linux **does not forward** packets between interfaces. To act as a router, enable forwarding:

```console
# sysctl -w net.ipv4.ip_forward=1
# sysctl -w net.ipv6.conf.all.forwarding=1
# echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-forward.conf
```

Then add routes and (for a private network behind it) NAT and firewall rules (Chapter 13).

### Policy routing (overview)

Linux supports **multiple routing tables** and rules that choose a table by source address, mark, or interface. Used for multi-homed hosts (two ISPs) and VPN split routing.

```console
$ ip rule
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
$ ip route show table all | head
```

### Reverse path filtering

`net.ipv4.conf.*.rp_filter` drops packets whose source address would **not** be routed back out the same interface. It prevents spoofing, but breaks **asymmetric routing**. Values: `0` off, `1` strict, `2` loose.

## Commands

### Show routes

| Command | Purpose |
|---------|---------|
| `ip route` | IPv4 routes (main table) |
| `ip -6 route` | IPv6 routes |
| `ip route show table all` | All tables |
| `ip route get 203.0.113.9` | **Which route and source address would be used for this destination** |
| `ip route show default` | Default routes only |
| `ip route show dev enp3s0` | Routes for one interface |

```console
$ ip route get 203.0.113.9
203.0.113.9 via 192.0.2.1 dev enp3s0 src 192.0.2.10 uid 1000
    cache
```

`ip route get` is one of the best troubleshooting commands: it shows what the kernel would do.

### Change routes (root, temporary)

```console
# ip route add 10.20.0.0/16 via 192.0.2.254                # static route
# ip route add 10.30.0.0/16 dev wg0                        # route through an interface
# ip route add default via 192.0.2.1                       # set default gateway
# ip route replace default via 192.0.2.2 metric 50         # add or change
# ip route del 10.20.0.0/16
# ip route del default via 192.0.2.1
# ip route add blackhole 198.51.100.0/24                   # silently discard
# ip route add unreachable 198.51.100.0/24                 # reject with ICMP unreachable
```

Test a gateway from the host:

```console
$ ping -c 2 192.0.2.1
$ ip neigh show 192.0.2.1           # is the gateway's MAC resolved?
$ traceroute -n 203.0.113.9
```

These changes disappear at reboot. See [Network Configuration Persistence](11-network-configuration-persistence.md) and [nmcli](09-networkmanager-and-nmcli.md).

### Two interfaces, one default route

When a host has two interfaces on different networks, normally only **one** default gateway should exist, or use different **metrics**. Two default routes with the same metric lead to unpredictable behavior and asymmetric routing.

```console
$ ip route
default via 192.0.2.1 dev enp3s0 metric 100
default via 198.51.100.1 dev enp4s0 metric 200      # backup
```

### Dynamic routing

For larger networks, routing daemons exchange routes automatically: **FRRouting (FRR)**, **BIRD**, and **Quagga** (older) support OSPF, BGP, and more. They are outside the scope of this page.

## Common pitfalls

- **"Network is unreachable".** No matching route and no default. Check `ip route`.
- **Gateway not in a connected subnet.** `ip route add default via 10.9.9.9` fails with "Nexthop has invalid gateway".
- **Two default gateways** on different interfaces causing intermittent failures. Use one, or set metrics deliberately and consider policy routing.
- **Return path differs from the outgoing path** (asymmetric routing). Stateful firewalls and `rp_filter` drop such traffic.
- **Routes lost after reboot or network restart.** Make them persistent in your network manager.
- **Wrong subnet mask** produces wrong connected routes.
- **Forgetting to enable forwarding** on a Linux router, or forgetting a return route on the other side. Traffic goes out but replies never return.
- **VPN clients replacing the default route** (full tunnel) and breaking access to local resources. Use split-tunnel routes if needed.
- **Containers and VMs add many routes and bridges.** `ip route` may show docker0/virbr0 networks that overlap yours. Check for conflicts.
- **Cloud environments** have their own routing (route tables, security groups). OS routes and cloud routes must agree.

## Further reading

- [`ip-route(8)`](https://man7.org/linux/man-pages/man8/ip-route.8.html)
- [`ip-rule(8)`](https://man7.org/linux/man-pages/man8/ip-rule.8.html)
- [Kernel: IP sysctl](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- [Linux Advanced Routing & Traffic Control](https://lartc.org)
