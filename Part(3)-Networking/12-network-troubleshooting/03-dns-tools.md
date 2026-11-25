# DNS Tools: dig, nslookup, and host

## Overview

DNS problems are among the most common causes of "the network is down". These tools let you ask DNS servers questions **directly**, see the full answer, and compare servers. Install them with:

| Family | Package |
|--------|---------|
| Debian-based | `dnsutils` (or `bind9-dnsutils`) |
| RHEL-based | `bind-utils` |
| SUSE-based | `bind-utils` |

> [!IMPORTANT]
> `dig`, `host`, and `nslookup` query DNS servers **directly**. They do not read `/etc/hosts` or use `nsswitch.conf`. Applications do. To see what an application sees, use `getent hosts NAME`.

## Commands

### dig

`dig [@SERVER] NAME [TYPE] [OPTIONS]`

```console
$ dig www.example.com

; <<>> DiG 9.18.24 <<>> www.example.com
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41520
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;www.example.com.               IN      A

;; ANSWER SECTION:
www.example.com.        300     IN      A       203.0.113.80

;; Query time: 23 msec
;; SERVER: 192.0.2.53#53(192.0.2.53) (UDP)
;; WHEN: Mon Mar 03 10:40:00 UTC 2025
;; MSG SIZE  rcvd: 60
```

| Part | Meaning |
|------|---------|
| `status: NOERROR` | The query worked (even if the answer is empty) |
| `status: NXDOMAIN` | The name **does not exist** |
| `status: SERVFAIL` | The server could not answer (upstream failure, DNSSEC failure, misconfigured zone) |
| `status: REFUSED` | The server refuses to answer you (access control) |
| `flags: qr rd ra` | `qr` response, `rd` recursion desired, `ra` recursion available, `aa` authoritative answer, `ad` DNSSEC validated |
| `300 IN A` | TTL (seconds the answer may be cached), class, record type |
| `Query time` | How long the server took |
| `SERVER` | Which server answered |

Common usage:

| Command | Purpose |
|---------|---------|
| `dig +short www.example.com` | Only the answer |
| `dig www.example.com AAAA` | IPv6 address |
| `dig example.com MX` | Mail servers |
| `dig example.com NS` | Name servers |
| `dig example.com TXT` | Text records (SPF, DKIM, verification) |
| `dig example.com SOA` | Zone information and serial number |
| `dig -x 203.0.113.80` | **Reverse** lookup (PTR) |
| `dig @198.51.100.53 www.example.com` | Ask a **specific** server |
| `dig @8.8.8.8 www.example.com` | Compare with a public resolver |
| `dig +trace www.example.com` | Follow the delegation from the root servers |
| `dig +noall +answer www.example.com` | Clean output with only the answer |
| `dig +tcp www.example.com` | Use TCP instead of UDP |
| `dig +dnssec example.com` | Request DNSSEC records |
| `dig +norecurse @server name` | Non-recursive query (is the answer cached?) |
| `dig +nocmd +noall +answer +ttlid ...` | Compact scripting form |
| `dig AXFR example.com @ns1` | Zone transfer (usually refused to outsiders) |

```console
$ dig +short example.com NS
a.iana-servers.net.
b.iana-servers.net.

$ dig +short -x 203.0.113.80
www.example.com.

$ dig +trace +nodnssec www.example.com | tail -5
```

### Checking consistency

Query **each authoritative server** and compare the answers and serials:

```console
$ for ns in $(dig +short example.com NS); do echo "== $ns"; dig +short @$ns example.com SOA; done
```

Different serial numbers show that zone transfers (or updates) are not in sync.

### host and nslookup

Simpler commands for quick checks.

```console
$ host www.example.com
www.example.com has address 203.0.113.80
www.example.com has IPv6 address 2001:db8::80

$ host -t MX example.com
$ host 203.0.113.80
80.113.0.203.in-addr.arpa domain name pointer www.example.com.

$ nslookup www.example.com
$ nslookup www.example.com 198.51.100.53         # use a specific server
$ nslookup -type=MX example.com
```

`nslookup` also has an interactive mode (`nslookup` with no arguments). `dig` is preferred for scripts and detail.

### What the application sees

```console
$ getent hosts www.example.com
203.0.113.80    www.example.com
$ getent ahostsv4 www.example.com
$ resolvectl query www.example.com               # via systemd-resolved
```

If `dig` works but `getent` does not (or gives a different answer), look at `/etc/hosts`, `/etc/nsswitch.conf`, search domains, a local cache, or VPN/split-DNS configuration.

## Troubleshooting patterns

| Symptom | Test | Likely cause |
|---------|------|--------------|
| Name does not resolve at all | `dig @<your DNS> name` and `dig @8.8.8.8 name` | If only the first fails: your resolver is broken or unreachable. If both fail: the name is wrong or the domain is broken |
| `NXDOMAIN` | Check spelling, zone, trailing dot, search domain | Name missing in DNS, wrong zone, or stale negative cache |
| `SERVFAIL` | `dig +cd name` (checking disabled), `dig +trace` | DNSSEC failure, broken delegation, unreachable authoritative servers |
| `REFUSED` | Try another resolver | ACL on the DNS server |
| Slow lookups (seconds) | `dig` query time; `strace -T -e trace=connect`; check `resolv.conf` | First nameserver down (timeout then second server), IPv6/AAAA timeouts, search domain chain, DNS server overloaded |
| Old address returned | `dig +norecurse`, check TTL | Cached record. Wait for TTL or flush cache (`resolvectl flush-caches`) |
| Different answers on different servers | Compare `SOA` serial | Zone transfer problems |
| Internal name fails from outside | | Split-horizon: internal zone not visible externally |
| Reverse lookup fails | `dig -x` | Missing PTR record (affects mail servers, logs, some auth) |
| Works for `host` but not for `ssh` or `curl` | `getent hosts` | Local config differs (hosts file, `nsswitch.conf`) |

### Flush caches

```console
# resolvectl flush-caches                  # systemd-resolved
# systemctl restart nscd                   # nscd (if used)
# sss_cache -E                             # SSSD
# rndc flush                               # BIND resolver
```

## Common pitfalls

- **Testing only with `dig`** and ignoring `/etc/hosts` or search domains.
- **Missing trailing dot.** `www.example.com.` is the absolute name. Without the dot, search domains may be appended.
- **Forgetting that answers are cached** at several levels (application, OS, resolver, upstream). Respect the TTL.
- **Comparing resolvers with different views** (split-horizon). A public resolver cannot see internal names.
- **`ANY` queries** are deprecated and often answered minimally. Ask for specific types.
- **UDP truncation.** Large answers (DNSSEC) need TCP. If TCP/53 is blocked, some lookups fail. Test `dig +tcp`.
- **DNS depends on a working firewall path** (UDP and TCP 53) between clients, resolvers, and authoritative servers.
- **Changing a record and expecting instant results.** TTLs delay changes. Lower the TTL **before** a planned change.
- **Zone transfer enabled for everyone** is an information leak. Restrict it.

## Further reading

- [`dig(1)`](https://man7.org/linux/man-pages/man1/dig.1.html)
- [BIND 9 documentation](https://bind9.readthedocs.io)
- [DNSViz](https://dnsviz.net) (DNSSEC and delegation visualization)
- [RFC 1035: DNS](https://datatracker.ietf.org/doc/html/rfc1035)
