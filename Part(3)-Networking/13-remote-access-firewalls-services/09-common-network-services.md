# Common Network Services

## Overview

A **network service** is a program that listens on a port and answers requests from other hosts. Each service you run is also **attack surface**. This page lists the services you will meet most often, what ports they use, how they are managed on Linux, and how to install and verify a simple one.

## Key concepts

### Service basics

| Term | Meaning |
|------|---------|
| **Daemon** | A background program providing a service (name often ends in `d`: `sshd`, `httpd`, `named`) |
| **Listening socket** | Address and port where the daemon waits for connections |
| **Bind address** | Which local addresses it listens on: `127.0.0.1` (local only), a specific IP, or `0.0.0.0` / `::` (all) |
| **systemd unit** | How the service is started, stopped, and supervised |
| **Socket activation** | systemd listens first and starts the service on demand (`.socket` units) |
| **`/etc/services`** | A text list that maps service names to well-known ports (informational only) |

### Common services and ports

| Service | Port(s) | Typical software | Notes |
|---------|---------|------------------|-------|
| SSH | 22/tcp | OpenSSH | Remote administration |
| HTTP / HTTPS | 80, 443/tcp (443/udp for HTTP/3) | Nginx, Apache (`httpd`/`apache2`), Caddy, HAProxy | Web, APIs, reverse proxy |
| DNS | 53/udp and 53/tcp | BIND, Unbound, dnsmasq, PowerDNS | Name resolution |
| DHCP | 67, 68/udp | Kea, dnsmasq, ISC dhcpd (legacy) | Address assignment |
| NTP | 123/udp | chrony, ntpd | Time |
| SMTP (mail transfer) | 25/tcp, 587 (submission), 465 | Postfix, Exim | Mail |
| IMAP / POP3 | 143, 993 / 110, 995 | Dovecot | Mailbox access |
| LDAP / LDAPS | 389, 636/tcp | OpenLDAP, 389-ds, FreeIPA | Directory |
| Kerberos | 88/tcp+udp | MIT Kerberos, FreeIPA | Authentication |
| NFS | 2049/tcp (v4) | nfs-server | File sharing (Unix) |
| SMB / CIFS | 445/tcp | Samba | File sharing (Windows) |
| Syslog | 514/udp, 514/tcp, 6514 (TLS) | rsyslog | Central logging |
| SNMP | 161/udp (agent), 162/udp (traps) | net-snmp | Monitoring (v3 recommended) |
| MySQL / MariaDB | 3306/tcp | MariaDB, MySQL | Database |
| PostgreSQL | 5432/tcp | PostgreSQL | Database |
| Redis | 6379/tcp | Redis, Valkey | Cache (do not expose without authentication) |
| MongoDB | 27017/tcp | MongoDB | Database |
| Prometheus / node_exporter | 9090, 9100/tcp | Prometheus | Monitoring |
| RDP / VNC | 3389, 5900/tcp | xrdp, VNC servers | Remote desktop |
| WireGuard | 51820/udp (typical) | wireguard | VPN |

### Package and unit names by family

| Service | Debian-based | RHEL-based |
|---------|--------------|------------|
| Apache | `apache2` | `httpd` |
| Nginx | `nginx` | `nginx` |
| SSH | `openssh-server` / `ssh` | `openssh-server` / `sshd` |
| DNS (BIND) | `bind9` / `named` or `bind9` | `bind` / `named` |
| NFS | `nfs-kernel-server` | `nfs-utils` / `nfs-server` |
| Samba | `samba` / `smbd` | `samba` / `smb` |
| Mail (Postfix) | `postfix` | `postfix` |
| Cron | `cron` | `cronie` / `crond` |

### Public, private, and local exposure

| Exposure | Bind to | Protected by |
|----------|---------|--------------|
| **Local only** | `127.0.0.1` / `::1` or a Unix socket | The host itself. Safest for databases used by local apps |
| **Private network** | The private interface address | Firewall, network segmentation, VPN |
| **Internet** | `0.0.0.0` / public IP | Firewall, TLS, authentication, patching, rate limiting, monitoring |

## Commands

### Install, start, and verify a simple web service (Nginx)

**Debian-based**

```console
# apt update && apt install nginx
# systemctl enable --now nginx
```

**RHEL-based**

```console
# dnf install nginx
# systemctl enable --now nginx
# firewall-cmd --permanent --add-service=http --add-service=https && firewall-cmd --reload
```

Verify it end to end:

```console
$ systemctl status nginx --no-pager
$ ss -tlnp | grep nginx                   # is it listening, and on which address?
LISTEN 0 511 0.0.0.0:80 0.0.0.0:* users:(("nginx",pid=1020,fd=6))
$ curl -I http://127.0.0.1/               # works locally?
HTTP/1.1 200 OK
$ curl -I http://192.0.2.10/              # works from another host? (tests firewall and network)
$ journalctl -u nginx -n 20 --no-pager
$ nginx -t                                # validate configuration (before reload)
```

The same pattern works for every service: **install → configure → validate → enable/start → check listening → firewall → test from the client → check logs**.

### Find which services are running and exposed

```console
$ systemctl list-units --type=service --state=running
$ systemctl list-unit-files --state=enabled --type=service
$ systemctl list-sockets
# ss -tulpn                                # what listens, on which addresses
$ ss -tuln | awk 'NR>1 {print $5}' | sort -u
```

### Stop a service you do not need

```console
# systemctl disable --now cups.service          # example: a printing service
# systemctl mask bluetooth.service              # prevent it from starting at all
# apt purge cups     # or: dnf remove cups     # remove the software entirely if not needed
```

### Reverse proxy (concept)

Instead of exposing many applications, put **one** hardened front end (Nginx, HAProxy, Caddy, Traefik) on ports 80/443. It terminates TLS, applies authentication and rate limits, and forwards to applications listening on `127.0.0.1` or private addresses.

```
Internet ─► [Firewall] ─► Nginx :443 (TLS) ─► app on 127.0.0.1:8080
                                          └─► app2 on 127.0.0.1:9000
```

### TLS certificates

Public services should use TLS. Certificates can be issued automatically by an ACME service such as **Let's Encrypt** (`certbot`, `acme.sh`, or built into Caddy). Renewal is typically automated by a systemd timer. Check expiry in monitoring.

```console
$ echo | openssl s_client -connect www.example.com:443 -servername www.example.com 2>/dev/null | openssl x509 -noout -enddate
notAfter=Apr 10 23:59:59 2025 GMT
```

### Super-servers and socket activation

Old systems used `inetd`/`xinetd` to start small services on demand. Today, **systemd socket units** do this. List them with `systemctl list-sockets`. Services started this way appear in `ss` as owned by `systemd` until the service starts.

## Hardening principles for services

1. **Install only what you need.** Every package is risk.
2. **Bind to the narrowest address** that works.
3. **Run as a non-root service user** (`User=` in the unit), with systemd sandboxing options.
4. **Authenticate and encrypt** (TLS, SSH, Kerberos). Avoid clear-text protocols (`telnet`, `ftp`, plain `http` for credentials, SNMP v1/v2c).
5. **Patch** quickly and track vendor advisories.
6. **Firewall** to the required sources only.
7. **Log** and monitor access and errors.
8. **Change defaults**: default passwords, sample pages, and debug modes.
9. **Back up** the configuration and data.
10. **Review regularly** what is running.

## Common pitfalls

- **Service listens only on `127.0.0.1`** by default (databases, Redis, development servers), so remote clients cannot reach it. Check `ss -tlnp`.
- **Opening a database port to the internet.** Databases and caches (MySQL, PostgreSQL, Redis, MongoDB, Elasticsearch) must not be public without strict authentication, TLS, and network limits. They are targeted constantly.
- **Starting the service but not enabling it** (or the reverse). Use `systemctl enable --now`.
- **Configuration errors found only at restart.** Always validate first (`nginx -t`, `apachectl configtest`, `named-checkconf`, `postfix check`, `sshd -t`).
- **Package name vs. service name mismatch** (`apache2` vs `httpd`, `ssh` vs `sshd`).
- **SELinux/AppArmor denials** on custom ports or directories. Check `ausearch -m avc -ts recent` and `journalctl`.
- **Forgetting the firewall** after installing, then assuming the service is broken.
- **Running services as root** with no need.
- **Leaving default sites or test pages** online, which discloses software and versions.
- **Port conflicts.** Two services want the same port (for example Apache and Nginx on 80). `ss -tlnp` shows who has it.

## Further reading

- [`systemd.service(5)`](https://man7.org/linux/man-pages/man5/systemd.service.5.html)
- [`services(5)`](https://man7.org/linux/man-pages/man5/services.5.html)
- [Nginx documentation](https://nginx.org/en/docs/)
- [Apache HTTP Server documentation](https://httpd.apache.org/docs/)
- [Let's Encrypt](https://letsencrypt.org/docs/)
- [IANA service names and port numbers](https://www.iana.org/assignments/service-names-port-numbers/)
