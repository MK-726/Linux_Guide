# iptables Overview

## Overview

**iptables** was the standard Linux firewall tool for two decades. You will still meet it in older documentation, scripts, Docker, Kubernetes, and legacy servers. On current distributions the `iptables` command is usually a **compatibility layer** that translates its syntax into **nftables** rules (`iptables-nft`). You should be able to read and maintain iptables rules, and know how they map to nftables.

## Key concepts

### Tables and chains

| Table | Purpose | Built-in chains |
|-------|---------|-----------------|
| `filter` (default) | Allow or block packets | `INPUT`, `FORWARD`, `OUTPUT` |
| `nat` | Address translation | `PREROUTING`, `OUTPUT`, `POSTROUTING` |
| `mangle` | Modify packet fields (TOS, TTL, marks, MSS) | all five |
| `raw` | Bypass connection tracking (`NOTRACK`) | `PREROUTING`, `OUTPUT` |
| `security` | SELinux-related marking | `INPUT`, `FORWARD`, `OUTPUT` |

`iptables` handles IPv4. **`ip6tables`** handles IPv6. Both must be configured.

### Targets

`ACCEPT`, `DROP`, `REJECT`, `LOG`, `RETURN`, `MASQUERADE`, `SNAT`, `DNAT`, `REDIRECT`, `TCPMSS`, or a user-defined chain.

### Rule order and policy

Rules in a chain are checked **top to bottom**. The first match with a terminal target ends the evaluation. If none match, the chain **policy** (`ACCEPT` or `DROP`) applies.

### Check which backend you use

```console
$ iptables --version
iptables v1.8.10 (nf_tables)      # compatibility layer over nftables
iptables v1.8.7 (legacy)          # old backend
$ update-alternatives --display iptables      # Debian-based: choose legacy or nft
```

Do **not** mix `legacy` and `nf_tables` backends on one system. Rules in one are invisible to the other.

## Commands

### List and inspect

```console
# iptables -L -n -v --line-numbers          # filter table with counters and rule numbers
# iptables -L INPUT -n -v
# iptables -t nat -L -n -v                  # nat table
# iptables -S                               # rules as commands (like iptables-save, readable)
# iptables-save                             # complete dump (for backups)
```

`-n` avoids slow DNS lookups. `-v` shows packet and byte counters, which tell you if a rule is being hit.

### Add, insert, delete

| Command | Purpose |
|---------|---------|
| `iptables -A CHAIN RULE` | **Append** to the end |
| `iptables -I CHAIN [N] RULE` | **Insert** at position N (default top) |
| `iptables -D CHAIN N` or `-D CHAIN RULE` | **Delete** by number or by specification |
| `iptables -R CHAIN N RULE` | Replace rule N |
| `iptables -P CHAIN TARGET` | Set the default **policy** |
| `iptables -F [CHAIN]` | **Flush** (delete all rules) |
| `iptables -N NAME` / `-X NAME` | Create / delete a user chain |
| `iptables -Z` | Zero counters |

Common match options:

| Option | Meaning |
|--------|---------|
| `-p tcp`, `-p udp`, `-p icmp` | Protocol |
| `-s ADDR[/MASK]` / `-d ADDR[/MASK]` | Source / destination |
| `--dport PORT` / `--sport PORT` | Destination / source port (need `-p tcp` or `-p udp`) |
| `-m multiport --dports 80,443` | Several ports |
| `-i IFACE` / `-o IFACE` | Input / output interface |
| `-m conntrack --ctstate ESTABLISHED,RELATED` | Connection state |
| `-m limit --limit 5/min` | Rate limit |
| `-m recent`, `-m hashlimit` | Rate-based blocking |
| `-j TARGET` | What to do |

### A basic host firewall

```console
# iptables -F                                   # start clean
# iptables -A INPUT -i lo -j ACCEPT
# iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
# iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 10/second -j ACCEPT
# iptables -A INPUT -p tcp -s 192.0.2.0/24 --dport 22 -j ACCEPT
# iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT
# iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "ipt-drop: "
# iptables -P INPUT DROP                        # policy LAST, after the allow rules
```

IPv6 needs the equivalent with `ip6tables` (and ICMPv6 must be allowed):

```console
# ip6tables -A INPUT -i lo -j ACCEPT
# ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# ip6tables -A INPUT -p ipv6-icmp -j ACCEPT
# ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT
# ip6tables -P INPUT DROP
```

### NAT examples

```console
# iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE                     # share a public IP
# iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.5:80
# iptables -A FORWARD -i eth0 -o eth1 -p tcp -d 10.0.0.5 --dport 80 -j ACCEPT
# iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Make rules persistent

| Family | Method |
|--------|--------|
| Debian-based | `apt install iptables-persistent` (or `netfilter-persistent`). Save: `netfilter-persistent save` → `/etc/iptables/rules.v4`, `rules.v6` |
| RHEL-based | `dnf install iptables-services`; `systemctl enable --now iptables ip6tables`; save with `service iptables save` → `/etc/sysconfig/iptables` (note: conflicts with firewalld, so disable that) |
| Manual | `iptables-save > /root/rules.v4` and restore with `iptables-restore < /root/rules.v4` at boot (systemd unit or network hook) |

```console
# iptables-save > /etc/iptables/rules.v4
# iptables-restore < /etc/iptables/rules.v4
# iptables-restore -t < /etc/iptables/rules.v4       # test the file (syntax) without applying
```

### Mapping to nftables

| iptables | nftables |
|----------|----------|
| `iptables -A INPUT -p tcp --dport 22 -j ACCEPT` | `nft add rule inet filter input tcp dport 22 accept` |
| `-m conntrack --ctstate ESTABLISHED,RELATED` | `ct state established,related` |
| `-j DROP` / `-j REJECT` | `drop` / `reject` |
| `-j MASQUERADE` | `masquerade` |
| `-j DNAT --to-destination X` | `dnat to X` |
| `-m multiport --dports 80,443` | `tcp dport { 80, 443 }` |
| `iptables-save` / `-restore` | `nft list ruleset` / `nft -f file` |

Translate: `iptables-translate`, `iptables-restore-translate`.

## Common pitfalls

- **Flushing with a `DROP` policy** (`iptables -F` after `-P INPUT DROP`) leaves a host with a default deny and no allow rules. Set the policy to `ACCEPT` **before** flushing, and set `DROP` **after** the allow rules.
- **Lockout over SSH.** Insert the SSH rule first. Test with a rollback timer: `echo 'iptables-restore < /root/good.rules' | at now + 5 minutes`.
- **Appending (`-A`) after a catch-all `DROP` or `REJECT`** rule. The new rule never matches. Use `-I`.
- **Rules not persistent.** Reboot loses them unless saved and restored.
- **Forgetting IPv6** (`ip6tables`).
- **Mixed backends** (`legacy` and `nf_tables`). Pick one (`update-alternatives`).
- **Docker, Kubernetes, and fail2ban** add chains. Do not flush blindly (`iptables -F` removes them and breaks containers). Use their documented hooks (the `DOCKER-USER` chain for custom Docker rules).
- **Using `-p tcp` implicit modules** and mistyping options. A missing `-p` with `--dport` gives an error.
- **No rule counters checked.** Use `iptables -L -n -v` to find unused rules and to verify traffic hits.
- **Slow lists with DNS lookups.** Always use `-n`.
- **`firewalld` and `iptables-services` together.** They conflict.

## Further reading

- [`iptables(8)`](https://man7.org/linux/man-pages/man8/iptables.8.html)
- [`iptables-extensions(8)`](https://man7.org/linux/man-pages/man8/iptables-extensions.8.html)
- [Netfilter: iptables](https://www.netfilter.org/projects/iptables/)
- [Red Hat: migrating from iptables to nftables](https://docs.redhat.com)
