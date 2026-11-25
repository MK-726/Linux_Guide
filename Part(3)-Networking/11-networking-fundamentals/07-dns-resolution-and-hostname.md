# DNS Resolution and Hostname Configuration

## Overview

People and applications use **names** (`www.example.com`). The network needs **IP addresses**. **DNS** (Domain Name System) translates between them. On Linux, name resolution involves several files and services. Knowing the order helps you fix "the name does not resolve" problems.

## Key concepts

### How a Linux host resolves a name

Applications call the C library (`getaddrinfo`). The library reads `/etc/nsswitch.conf` to decide **which sources** to use and in which order.

```console
$ grep '^hosts:' /etc/nsswitch.conf
hosts:          files mdns4_minimal [NOTFOUND=return] dns
```

| Source | Meaning |
|--------|---------|
| `files` | `/etc/hosts` |
| `dns` | The DNS servers in `/etc/resolv.conf` |
| `resolve` | systemd-resolved |
| `myhostname` | The machine's own hostname |
| `mdns4_minimal` | Multicast DNS (`.local` names) |

Default order: **`/etc/hosts` first**, then DNS.

```
Application ──► nsswitch.conf ──► /etc/hosts ──► (no match) ──► DNS resolver ──► DNS server
```

### /etc/hosts

A local table for fixed names. Format: `IP  name  [aliases...]`.

```
127.0.0.1   localhost
127.0.1.1   server01.example.com server01
192.0.2.30  db01.example.com db01
::1         localhost ip6-localhost
```

Use it for a few critical names or when DNS is unavailable. It is not a scalable replacement for DNS.

### /etc/resolv.conf

Lists the DNS servers (**nameservers**) and the **search domains**.

```
nameserver 192.0.2.53
nameserver 198.51.100.53
search example.com corp.example.com
options timeout:2 attempts:2 rotate
```

| Line | Meaning |
|------|---------|
| `nameserver IP` | A DNS server. The first is tried first. Maximum 3 are used |
| `search d1 d2` | Domains appended to short names. `db01` is tried as `db01.example.com`, then `db01.corp.example.com` |
| `options ndots:N` | A name with fewer than N dots is tried with search domains first |
| `options timeout:N attempts:N rotate` | Retry behavior |

> [!IMPORTANT]
> On most systems `/etc/resolv.conf` is **generated** by NetworkManager, systemd-resolved, `resolvconf`, or cloud-init. Manual edits are overwritten. Set DNS in the network configuration instead. Check what it is: `ls -l /etc/resolv.conf`.

### systemd-resolved

Many systems (Ubuntu, Fedora, others) run **systemd-resolved**, a local caching resolver. `/etc/resolv.conf` points to a stub at `127.0.0.53`, and the real DNS servers are stored per interface.

```console
$ ls -l /etc/resolv.conf
lrwxrwxrwx 1 root root 39 Mar  3 09:00 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

$ resolvectl status
Global
       Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub
Link 2 (enp3s0)
    Current Scopes: DNS
         Protocols: +DefaultRoute
Current DNS Server: 192.0.2.53
       DNS Servers: 192.0.2.53 198.51.100.53
        DNS Domain: example.com
```

| Command | Purpose |
|---------|---------|
| `resolvectl status` | DNS servers per interface |
| `resolvectl query example.com` | Resolve through resolved (shows which server and protocol) |
| `resolvectl flush-caches` | Clear the cache |
| `resolvectl statistics` | Cache hits and misses |
| `resolvectl dns enp3s0 192.0.2.53` | Set DNS for one interface (temporary) |

Configuration: `/etc/systemd/resolved.conf` and `/etc/systemd/resolved.conf.d/*.conf`:

```ini
[Resolve]
DNS=192.0.2.53 198.51.100.53
FallbackDNS=1.1.1.1 9.9.9.9
Domains=example.com
DNSSEC=allow-downgrade
DNSOverTLS=opportunistic
```

### DNS record types

| Type | Purpose | Example |
|------|---------|---------|
| `A` | Name to IPv4 | `www  A  192.0.2.80` |
| `AAAA` | Name to IPv6 | `www  AAAA  2001:db8::80` |
| `CNAME` | Alias to another name | `ftp  CNAME  www` |
| `MX` | Mail server | `example.com MX 10 mail.example.com.` |
| `NS` | Name servers of a zone | |
| `PTR` | IP to name (reverse DNS) | `10.2.0.192.in-addr.arpa` |
| `TXT` | Text (SPF, DKIM, verification) | |
| `SRV` | Service location | |
| `SOA` | Zone authority and serial number | |

### Recursive and authoritative

| Role | Description |
|------|-------------|
| **Stub resolver** | The small client on your host. Asks a recursive resolver |
| **Recursive resolver** (caching) | Does the work of finding the answer, caches it. Your ISP, company, or public resolver |
| **Authoritative server** | Holds the real records for a zone |
| **TTL** | How long an answer may be cached |

### The hostname

| Name | Meaning |
|------|---------|
| **Static hostname** | Stored in `/etc/hostname`. Set at boot |
| **Transient hostname** | Assigned by DHCP or the kernel at runtime |
| **Pretty hostname** | Free-form description |
| **FQDN** | Fully qualified domain name: `server01.example.com` |

Rules: letters, digits, and hyphens; up to 63 characters per label; no underscores; case-insensitive.

## Commands

```console
$ hostname                       # short name
$ hostname -f                    # FQDN (needs correct /etc/hosts or DNS)
$ hostnamectl
 Static hostname: server01
       Icon name: computer-vm
         Chassis: vm
      Machine ID: 2f1d...
        Boot ID: 8c6a...
Operating System: Ubuntu 24.04 LTS
          Kernel: Linux 6.8.0-45-generic

# hostnamectl set-hostname server01.example.com       # permanent (writes /etc/hostname)
$ getent hosts www.example.com            # what applications see (uses nsswitch.conf)
203.0.113.80    www.example.com
$ getent ahosts www.example.com           # all addresses (IPv4 and IPv6)
$ host www.example.com
$ dig +short www.example.com
```

Use `getent hosts` to test what an application would get. `dig` and `nslookup` query the DNS server **directly** and ignore `/etc/hosts` (see [DNS tools](../12-network-troubleshooting/03-dns-tools.md)).

### Set DNS servers (where it belongs)

| Tool | Command or file |
|------|-----------------|
| NetworkManager | `nmcli con mod "Wired" ipv4.dns "192.0.2.53 198.51.100.53" ipv4.dns-search example.com` |
| Netplan | `nameservers: addresses: [192.0.2.53]` |
| systemd-networkd | `DNS=192.0.2.53` in the `.network` file |
| systemd-resolved (global) | `/etc/systemd/resolved.conf` |
| DHCP | Provided by the DHCP server (default for most hosts) |

### Check the cache and timing

```console
$ time getent hosts www.example.com          # first lookup
$ time getent hosts www.example.com          # cached lookup (faster)
$ resolvectl statistics | grep -i -E 'cache|transactions'
```

## Common pitfalls

- **Editing `/etc/resolv.conf`** and losing the change. Configure DNS in the network manager.
- **Testing with `dig` only.** `dig` works but the application fails: the problem may be `/etc/hosts`, `nsswitch.conf`, search domains, IPv6 preference, or a local cache. Test with `getent hosts`.
- **Hostname does not resolve to the machine's address.** Some software needs `hostname -f` to work. Add the hostname to `/etc/hosts`.
- **Wrong search domain** creates slow lookups, or unexpected matches.
- **Large `ndots` in Kubernetes** causes many extra lookups for external names.
- **Stale cache** after DNS changes. Flush the cache, and wait for TTLs on upstream resolvers.
- **Only one nameserver and it is down.** Add at least two.
- **Containers have their own `/etc/resolv.conf`**, inherited from the host or runtime configuration.
- **Public DNS for internal names** leaks information and does not resolve private names. Use split-horizon DNS.
- **Underscores in hostnames** break some software and mail systems.
- **Changing the hostname on a running system** can confuse services (certificates, monitoring, Kerberos). Restart services and update records.

## Further reading

- [`nsswitch.conf(5)`](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)
- [`resolv.conf(5)`](https://man7.org/linux/man-pages/man5/resolv.conf.5.html)
- [`resolvectl(1)`](https://www.freedesktop.org/software/systemd/man/latest/resolvectl.html)
- [`hostnamectl(1)`](https://man7.org/linux/man-pages/man1/hostnamectl.1.html)
- [RFC 1034 / 1035: DNS concepts and specification](https://datatracker.ietf.org/doc/html/rfc1034)
