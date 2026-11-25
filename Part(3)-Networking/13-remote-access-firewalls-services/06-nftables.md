# nftables

## Overview

**nftables** is the current Linux packet-filtering framework. It replaces iptables, ip6tables, arptables, and ebtables with **one** tool (`nft`) and one syntax. Its features include sets, maps, atomic rule updates, one table for IPv4 and IPv6 (`inet`), and better performance with many rules. It is the backend for firewalld and the compatibility iptables on current distributions.

## Key concepts

### Structure

```
ruleset
 └── table (family + name)            e.g.  inet filter
      ├── chain (base or regular)     e.g.  input   (hook input, policy drop)
      │     └── rules                  e.g.  tcp dport 22 accept
      ├── set / map                    e.g.  a list of allowed addresses
      └── ...
```

| Element | Meaning |
|---------|---------|
| **Family** | `ip` (IPv4), `ip6` (IPv6), **`inet`** (both), `arp`, `bridge`, `netdev` |
| **Table** | A container for chains and sets. You choose the name |
| **Base chain** | Attached to a kernel hook with a type, hook, priority, and policy. Receives packets |
| **Regular chain** | No hook. Used with `jump` or `goto` |
| **Rule** | Matches and a verdict (and optional statements such as `counter`, `log`, `limit`) |
| **Set** | A collection of addresses, ports, and so on for fast matching |

Base chain declaration:

```
chain input {
    type filter hook input priority 0; policy drop;
}
```

| Part | Values |
|------|--------|
| `type` | `filter`, `nat`, `route` |
| `hook` | `prerouting`, `input`, `forward`, `output`, `postrouting` |
| `priority` | Number; lower runs first. `0` for filter, `-100` for dnat, `100` for snat |
| `policy` | `accept` or `drop` |

## Commands

### Inspect

```console
# nft list ruleset                         # everything
# nft list tables
# nft list table inet filter
# nft -a list chain inet filter input      # -a shows rule handles (needed to delete a single rule)
# nft list ruleset -n                      # numeric (no name resolution)
# nft monitor trace                        # trace packets (after enabling nftrace in a rule)
```

### A complete host firewall (file)

`/etc/nftables.conf` (Debian-based) or `/etc/sysconfig/nftables.conf` (RHEL-based; check `systemctl cat nftables`):

```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {

    set admin_nets {
        type ipv4_addr
        flags interval
        elements = { 192.0.2.0/24, 198.51.100.0/28 }
    }

    chain input {
        type filter hook input priority 0; policy drop;

        ct state invalid drop
        ct state established,related accept

        iif "lo" accept

        # ICMP / ICMPv6 (needed for diagnostics, path MTU, IPv6 neighbor discovery)
        ip protocol icmp icmp type { echo-request, destination-unreachable, time-exceeded } limit rate 10/second accept
        ip6 nexthdr icmpv6 limit rate 10/second accept

        # SSH only from admin networks
        tcp dport 22 ip saddr @admin_nets accept

        # Web services
        tcp dport { 80, 443 } accept

        # Log what is about to be dropped (rate limited)
        limit rate 5/minute log prefix "nft-input-drop: " level info
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

```console
# nft -c -f /etc/nftables.conf             # CHECK syntax only (does not apply)
# nft -f /etc/nftables.conf                # apply (atomic: all or nothing)
# systemctl enable --now nftables          # load the file at boot
```

Notes on this example:

- `flush ruleset` at the top makes loading **replace** the whole configuration. It also removes rules created by other tools (firewalld, Docker). Do not combine it with those.
- IPv6 SSH is not allowed in this example because `admin_nets` is IPv4 only. Add an `ip6` set for IPv6 admin networks.
- `ip6 nexthdr icmpv6` accepts all ICMPv6 types (rate limited). Blocking neighbor discovery breaks IPv6.

### Add and remove rules interactively (runtime only)

```console
# nft add rule inet filter input tcp dport 8080 accept
# nft insert rule inet filter input tcp dport 8443 accept        # at the top
# nft -a list chain inet filter input
    tcp dport 8080 accept # handle 17
# nft delete rule inet filter input handle 17
# nft add rule inet filter input ip saddr 203.0.113.9 drop
# nft add element inet filter admin_nets { 203.0.113.0/24 }      # add to a set
# nft delete element inet filter admin_nets { 203.0.113.0/24 }
```

To keep runtime changes, save them into the file: `nft list ruleset > /etc/nftables.conf` (check the result, and add `flush ruleset` at the top).

### Useful rule building blocks

| Goal | Rule |
|------|------|
| Allow a TCP port | `tcp dport 443 accept` |
| Several ports | `tcp dport { 80, 443, 8080 } accept` |
| Port range | `tcp dport 8000-8100 accept` |
| UDP | `udp dport 123 accept` |
| From a source address or network | `ip saddr 192.0.2.0/24 tcp dport 22 accept` |
| IPv6 source | `ip6 saddr 2001:db8::/32 tcp dport 22 accept` |
| By interface | `iifname "eth0" tcp dport 80 accept` |
| Reject politely | `tcp dport 23 reject with tcp reset` |
| Count packets | `tcp dport 22 counter accept` |
| Rate limit | `tcp dport 22 ct state new limit rate 6/minute accept` |
| Block an address | `ip saddr 203.0.113.9 drop` |
| Log | `log prefix "name: " level warn` |
| Established traffic | `ct state established,related accept` |

### NAT

```
table ip nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        iifname "eth0" tcp dport 8080 dnat to 10.0.0.5:80        # port forward
    }
    chain postrouting {
        type nat hook postrouting priority 100;
        oifname "eth0" masquerade                                 # share the public address
    }
}
```

Requires forwarding (`sysctl -w net.ipv4.ip_forward=1`) and matching `forward` chain rules (for example `ct state established,related accept` and `iifname "eth1" oifname "eth0" accept`).

### Counters and debugging

```console
# nft list ruleset | grep -B2 -A2 counter
# nft reset counters
# journalctl -k | grep nft-input-drop             # your log prefix
```

Trace one packet through the ruleset (advanced):

```console
# nft add rule inet filter input meta nftrace set 1
# nft monitor trace
```

### Migrating from iptables

```console
# iptables-save > rules.v4
# iptables-restore-translate -f rules.v4 > rules.nft       # translate (review the result!)
# iptables-translate -A INPUT -p tcp --dport 22 -j ACCEPT
nft add rule ip filter INPUT tcp dport 22 counter accept
```

## Common pitfalls

- **`flush ruleset` deleting rules from firewalld, Docker, or Kubernetes.** Decide who owns the ruleset.
- **Applying with a default `drop` policy** before allowing SSH. Keep a session open, or use a rollback timer.
- **Rules not persistent.** Enable `nftables.service` and keep the config file updated.
- **Forgetting IPv6.** Use `inet` tables, and include ICMPv6.
- **Semicolons and braces.** Inside chain definitions the `type ... policy drop;` line needs the semicolon. Test with `nft -c -f`.
- **Priority confusion.** Several base chains on the same hook all run, ordered by priority. A `drop` in any of them wins.
- **`ct state established,related accept` missing** makes return traffic fail.
- **Interface names in rules** (`iif`, `iifname`): `iif` binds to the interface index (the interface must exist when loading). `iifname` compares names (works for interfaces created later, such as VPN or container interfaces).
- **Large sets and performance.** Use sets and maps instead of hundreds of single rules.
- **Mixing `iptables` and `nft` commands** against different backends (`iptables-legacy` vs `nft`). Confirm `iptables --version`.

## Further reading

- [nftables wiki](https://wiki.nftables.org)
- [`nft(8)`](https://man7.org/linux/man-pages/man8/nft.8.html)
- [nftables quick reference](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)
- [Red Hat: getting started with nftables](https://docs.redhat.com)
