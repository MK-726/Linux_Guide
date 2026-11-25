# Port Management and Service Exposure

## Overview

Every open port is a possible entry point. **Port management** means knowing exactly which ports are open, why, to whom, and verifying it from both the inside and the outside. This page ties together `ss`, firewalls, service configuration, and scanning into a repeatable audit.

## Key concepts

### Three views of "open"

A port is **reachable** only if all of these are true:

| Layer | Question | Tool |
|-------|----------|------|
| **Service** | Is a process listening, and on which address? | `ss -tulpn` |
| **Host firewall** | Does the firewall allow the traffic from that source? | `nft list ruleset`, `firewall-cmd --list-all`, `ufw status verbose` |
| **Network path** | Do network firewalls, security groups, NAT, and routing allow it? | Test from outside: `nc -zv`, `nmap` |

A service can be **listening but blocked** (safe), **allowed but not listening** (a stale rule), or **listening and exposed** (the thing to review).

### Reading listening addresses

| Local address in `ss` | Exposure |
|-----------------------|----------|
| `127.0.0.1:5432`, `[::1]:5432` | Local only |
| `192.0.2.10:8080` | Only on that address (could be a private or public interface) |
| `0.0.0.0:22`, `[::]:22`, `*:22` | **All interfaces**. Exposure depends on the firewall |
| `unix` sockets | Local only (not network) |

### Privileged ports

Ports **below 1024** normally need root to bind. Safer ways to serve them without running as root:

| Method | Notes |
|--------|-------|
| `AmbientCapabilities=CAP_NET_BIND_SERVICE` in the systemd unit | Preferred. See [Linux Capabilities](../../part-1-foundations/04-users-groups-and-permissions/08-capabilities-introduction.md) |
| `setcap cap_net_bind_service=+ep /path/to/binary` | Per file; lost on update |
| `net.ipv4.ip_unprivileged_port_start=80` (sysctl) | System-wide change. Affects all users |
| Reverse proxy or firewall redirect | App listens on a high port, proxy/DNAT serves 80/443 |

## Commands

### 1. What is listening (inside)

```console
# ss -tulpn
Netid State  Local Address:Port  Process
tcp   LISTEN 0.0.0.0:22          users:(("sshd",pid=812,fd=3))
tcp   LISTEN 0.0.0.0:80          users:(("nginx",pid=1020,fd=6))
tcp   LISTEN 127.0.0.1:5432      users:(("postgres",pid=1130,fd=5))
tcp   LISTEN 0.0.0.0:6379        users:(("redis-server",pid=1501,fd=6))      ← unexpected exposure!
udp   UNCONN 0.0.0.0:68          users:(("NetworkManager",pid=700,fd=25))
```

| Question | Command |
|----------|---------|
| Which process owns port 6379? | `ss -tlnp 'sport = :6379'`, `lsof -i :6379`, `fuser 6379/tcp` |
| Which package and unit is it? | `systemctl status PID`, `readlink /proc/PID/exe`, `dpkg -S` / `rpm -qf` |
| Count listeners by address | `ss -tuln -H` (then group the Local Address column with `sort` and `uniq -c`) |
| Sockets via systemd | `systemctl list-sockets` |
| Containers | `docker ps --format '{{.Names}} {{.Ports}}'`, `nsenter -t PID -n ss -tlnp` |

### 2. What does the firewall allow

```console
# nft list ruleset | grep -E 'dport|saddr|policy'
# firewall-cmd --list-all-zones | grep -E '^[a-z]|services|ports|sources'
# ufw status numbered
# iptables -S | grep -E 'dport|policy'
```

### 3. What is reachable (outside)

From another host (and from outside the network for internet-facing services):

```console
$ nc -zv -w 3 192.0.2.30 22 80 443 6379
$ nmap -Pn -p 1-1024 192.0.2.30                          # most common ports
$ nmap -sT -p- --min-rate 1000 192.0.2.30                # all 65535 TCP ports (slow, noisy)
$ nmap -sV -p 22,80,443 192.0.2.30                       # service and version detection
$ nmap -sU -p 53,123,161 192.0.2.30                      # selected UDP ports (slow and less reliable)
$ nmap -6 -Pn -p 22,80,443 2001:db8::30                  # do not forget IPv6
```

nmap port states:

| State | Meaning |
|-------|---------|
| `open` | A service accepts connections |
| `closed` | Host reachable, nothing listening (RST) |
| `filtered` | No answer: a firewall drops the probes |
| `open\|filtered` | Common for UDP: no answer could mean either |

> [!WARNING]
> Scan only systems you own or have **written permission** to test. Scanning others can violate law and policy, trigger intrusion detection, and disrupt fragile devices.

### 4. Reduce exposure

| Step | How |
|------|-----|
| Stop and disable services you do not need | `systemctl disable --now NAME`; remove the package |
| Bind to localhost or a private address | Service configuration: `bind 127.0.0.1` (Redis), `listen_addresses = 'localhost'` (PostgreSQL), `bind-address = 127.0.0.1` (MariaDB), `ListenAddress` (SSH) |
| Close the port in the firewall | Remove the rule or default deny |
| Limit sources | Firewall rules by source network; zones; security groups |
| Put it behind a reverse proxy or VPN | Expose only 443 (and VPN), reach admin interfaces through VPN or an SSH tunnel |
| Require TLS and authentication | Even on private networks |
| Rate-limit and ban abusers | `fail2ban`, nftables limits, proxy limits |
| Containers | Publish ports on loopback (`-p 127.0.0.1:8080:80`) when only the host needs them |
| Check IPv6 | Services listening on `::` are reachable over IPv6; firewall must cover it |

Example: stop Redis from listening on all interfaces (`/etc/redis/redis.conf`):

```
bind 127.0.0.1 -::1
protected-mode yes
requirepass <long random password>
```

### 5. Change a service's port

```console
# vi /etc/ssh/sshd_config                # Port 2222
# sshd -t
# semanage port -a -t ssh_port_t -p tcp 2222        # SELinux (RHEL-based)
# firewall-cmd --permanent --add-port=2222/tcp && firewall-cmd --reload
# systemctl restart sshd
# ss -tlnp | grep 2222
```

Order: **firewall and SELinux first**, then restart the service, then test from a second session.

### 6. Port conflicts

```console
$ systemctl start apache2
Job for apache2.service failed. ... (98)Address already in use: AH00072: make_sock: could not bind to address [::]:80
# ss -tlnp 'sport = :80'
```

Stop or reconfigure the existing owner, or change the port.

### 7. A repeatable exposure audit

1. **Inventory** listening sockets: `ss -tulpn` (save the output as a baseline).
2. **Explain each one**: purpose, owner, unit, intended audience.
3. **Check the firewall** rules for each exposed port, for IPv4 **and** IPv6.
4. **Scan from outside** (authorized) and **compare** with the expected list.
5. **Remove or restrict** anything unexpected.
6. **Document** the approved list (ports, sources, owner, justification).
7. **Monitor** for changes: periodic `ss` diffs, `auditd` rules for `bind`, configuration management drift detection, external scans.
8. **Repeat** regularly and after every change.

```console
# ss -tulpnH | sort > /var/lib/baseline/listening.txt
# diff <(ss -tulpnH | sort) /var/lib/baseline/listening.txt && echo "no change"
```

## Common pitfalls

- **Changing the SSH port and calling it "security".** It reduces bot noise only. Use keys, firewall limits, and updates.
- **Port knocking as the only protection.** It is obscurity, not authentication.
- **Fixing exposure only in the firewall** while the service stays on `0.0.0.0` (one firewall mistake away from exposure). Fix the **bind address** too.
- **Not checking IPv6.** A service on `[::]` may be open even if IPv4 is closed.
- **Docker and Kubernetes published ports** that bypass host firewall input rules.
- **Scanning from inside the network** and concluding the service is not exposed to the internet. Test from the outside.
- **Forgetting UDP.** DNS, NTP, SNMP, memcached, and others can be abused over UDP.
- **Stale firewall rules** for services that no longer run. They become holes when something else binds the port later.
- **Leaving management interfaces public** (databases, dashboards, `kubelet`, `etcd`, Elasticsearch, Docker API on 2375). These are attacked continuously.
- **Relying on default `0.0.0.0` bindings** of new software. Read the defaults.
- **Unreviewed `nmap` output.** `filtered` is not the same as `closed`.

## Further reading

- [`ss(8)`](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Nmap reference guide](https://nmap.org/book/man.html)
- [`nft(8)`](https://man7.org/linux/man-pages/man8/nft.8.html)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks) (hardening baselines)
- [Docker: packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
