# DNS and DHCP Server Concepts

## Overview

Almost every network needs **DNS** (names) and **DHCP** (addresses). This page explains how the servers work, which software to choose, and shows small working examples. For the client side, see [DNS Resolution and Hostname Configuration](../11-networking-fundamentals/07-dns-resolution-and-hostname.md) and [DHCP Fundamentals](../11-networking-fundamentals/08-dhcp-fundamentals.md).

## Key concepts: DNS servers

### Roles

A DNS server can act in different roles. **Do not combine them on one exposed server** without a reason.

| Role | What it does | Example software |
|------|--------------|------------------|
| **Authoritative server** | Holds the **zone data** for your domains and answers for them. Does not look up other domains | BIND, NSD, Knot DNS, PowerDNS |
| **Recursive resolver** (caching) | Answers clients by querying other servers and **caching** results | Unbound, BIND, Knot Resolver, systemd-resolved (client-side) |
| **Forwarder** | Passes queries to another resolver | dnsmasq, BIND, Unbound |
| **Small LAN DNS + DHCP** | One tool for a small network | **dnsmasq** |

### Zones and records

- A **zone** is a part of the DNS tree managed by one administration (for example `example.com`).
- A zone file contains **resource records**: `A`, `AAAA`, `CNAME`, `MX`, `NS`, `TXT`, `PTR`, `SRV`, and the **SOA** (start of authority) with a **serial number**.
- **Primary (master)** servers hold the editable zone. **Secondary (slave)** servers copy it by **zone transfer** (AXFR/IXFR) and answer for it. Increase the **serial** with every change, or secondaries will not update.
- **Delegation**: the parent zone (`.com`) lists your NS records, so the world finds your authoritative servers.
- **Reverse zones** (`2.0.192.in-addr.arpa`) hold PTR records (IP to name).
- **Split-horizon (split-view) DNS**: different answers for internal and external clients.
- **DNSSEC** signs zones so resolvers can verify answers.

### Security rules for DNS servers

| Rule | Why |
|------|-----|
| **No open recursion** for the internet | Open resolvers are abused in DDoS amplification attacks. Allow recursion only for your own networks |
| Restrict **zone transfers** to your secondaries | Prevents zone data leaks |
| Keep authoritative and recursive roles **separate** | Different risk profiles |
| Use **rate limiting** (RRL) | Reduces abuse |
| Keep software **updated** | DNS servers are a frequent target |
| Run **at least two** authoritative servers on different networks | Redundancy |

## Commands and examples: DNS

### Unbound: a small recursive/caching resolver for a LAN

```
# /etc/unbound/unbound.conf.d/lan.conf
server:
    interface: 192.0.2.53
    interface: 127.0.0.1
    access-control: 127.0.0.0/8 allow
    access-control: 192.0.2.0/24 allow
    access-control: 0.0.0.0/0 refuse
    hide-identity: yes
    hide-version: yes
    cache-min-ttl: 60
    prefetch: yes
    private-domain: "corp.example.com"

local-zone: "lab.example.com." static
local-data: "git.lab.example.com. A 192.0.2.40"

forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 9.9.9.9@853#dns.quad9.net
```

```console
# unbound-checkconf
# systemctl enable --now unbound
$ dig @192.0.2.53 www.example.com +short
$ dig @192.0.2.53 git.lab.example.com +short
```

### dnsmasq: DNS forwarder and DHCP for small networks

```
# /etc/dnsmasq.d/lan.conf
interface=enp3s0
bind-interfaces
domain=lab.example.com
local=/lab.example.com/
expand-hosts
no-resolv
server=192.0.2.1                       # upstream DNS

dhcp-range=192.0.2.100,192.0.2.200,255.255.255.0,12h
dhcp-option=option:router,192.0.2.1
dhcp-option=option:dns-server,192.0.2.53
dhcp-host=52:54:00:aa:bb:cc,printer,192.0.2.60         # reservation
address=/test.lab.example.com/192.0.2.70               # fixed DNS answer
```

```console
# dnsmasq --test                       # syntax check
# systemctl enable --now dnsmasq
$ cat /var/lib/misc/dnsmasq.leases     # active leases
```

### BIND 9: authoritative zone (minimal)

`/etc/bind/named.conf.local` (Debian-based) or `/etc/named.conf` (RHEL-based):

```
zone "example.com" {
    type primary;
    file "/var/lib/bind/db.example.com";
    allow-transfer { 198.51.100.53; };      # the secondary only
    notify yes;
};
```

`/var/lib/bind/db.example.com`:

```
$TTL 3600
@   IN SOA ns1.example.com. hostmaster.example.com. (
        2025030301 ; serial (YYYYMMDDnn)
        3600       ; refresh
        900        ; retry
        1209600    ; expire
        300 )      ; negative caching TTL
    IN NS   ns1.example.com.
    IN NS   ns2.example.com.
    IN MX   10 mail.example.com.
ns1  IN A    192.0.2.53
ns2  IN A    198.51.100.53
www  IN A    203.0.113.80
www  IN AAAA 2001:db8::80
mail IN A    203.0.113.25
ftp  IN CNAME www
```

```console
# named-checkconf
# named-checkzone example.com /var/lib/bind/db.example.com
zone example.com/IN: loaded serial 2025030301
OK
# systemctl reload named           # service name: named (RHEL-based) / bind9 (Debian-based)
$ dig @192.0.2.53 www.example.com +norecurse
```

Rules for zone files: names ending with a dot are absolute; names without a dot get the zone name appended; **increase the serial** with each edit.

## Key concepts: DHCP servers

### What a DHCP server needs

- A **static address** on the interface that serves clients.
- A **pool** (range) of addresses, excluding static hosts.
- **Options** to hand out: gateway, DNS servers, domain, NTP, search domains.
- **Reservations** for devices that need fixed addresses.
- A way to reach clients on other subnets: a **DHCP relay** (router helper) or a server interface in each subnet.
- Only **one** authoritative DHCP server per subnet (or a failover pair).

### Software

| Software | Notes |
|----------|-------|
| **Kea** (ISC) | Modern DHCPv4/v6 server with JSON configuration, REST API, hooks, and database back ends. The **successor** of ISC DHCP |
| **ISC DHCP (`dhcpd`)** | Legacy, end of life. Existing installations should migrate to Kea |
| **dnsmasq** | Lightweight DNS and DHCP for small networks and labs |
| **systemd-networkd** | Has a small built-in DHCP server (for simple point-to-point setups) |
| **Router/appliance DHCP** | Often handled by the router or network gear |

### Kea DHCPv4 example (JSON)

`/etc/kea/kea-dhcp4.conf`:

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": [ "enp3s0" ] },
    "lease-database": { "type": "memfile", "lfc-interval": 3600 },
    "valid-lifetime": 43200,
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.0.2.0/24",
        "pools": [ { "pool": "192.0.2.100 - 192.0.2.200" } ],
        "option-data": [
          { "name": "routers", "data": "192.0.2.1" },
          { "name": "domain-name-servers", "data": "192.0.2.53, 198.51.100.53" },
          { "name": "domain-name", "data": "example.com" }
        ],
        "reservations": [
          { "hw-address": "52:54:00:aa:bb:cc", "ip-address": "192.0.2.60", "hostname": "printer" }
        ]
      }
    ]
  }
}
```

```console
# kea-dhcp4 -t /etc/kea/kea-dhcp4.conf        # test the configuration
# systemctl enable --now kea-dhcp4-server     # unit name varies by distribution
$ journalctl -u kea-dhcp4-server -f
```

### Firewall

| Service | Open |
|---------|------|
| DNS server | UDP and TCP 53 from clients (and from secondaries for AXFR on TCP 53) |
| DHCPv4 server | UDP 67 (server), clients use 68 |
| DHCPv6 server | UDP 547 |

## Troubleshooting

| Symptom | Check |
|---------|-------|
| DNS server does not answer | `ss -ulnp` and look for port 53 (is it listening on the right address?), firewall, `journalctl -u named` / `unbound` |
| Answers for internal names fail from clients | ACLs (`allow-query`, `access-control`), forwarding, split-horizon views |
| Secondary has an old zone | Serial not increased, `notify` blocked, TCP 53 blocked, `allow-transfer` too strict. Compare `dig @ns1 SOA` and `dig @ns2 SOA` |
| `SERVFAIL` for a signed zone | DNSSEC problem (expired signatures, wrong DS record). Test with `dig +cd` |
| Clients get no DHCP lease | Server listening on the right interface? Pool exhausted? Relay configured? `tcpdump -ni IFACE port 67 or port 68` |
| Clients get wrong gateway or DNS | A **second DHCP server** (rogue) on the network. Check which server answered in the lease or capture |
| Hostnames not updating | DHCP and DNS are separate. Use dynamic DNS updates (Kea DDNS, dnsmasq local names) or static records |

## Common pitfalls

- **Open recursive resolver** on the public internet. Restrict recursion to your own networks.
- **Forgetting to increase the zone serial.** Secondaries keep the old version.
- **Missing trailing dots** in zone files (`ns1.example.com` becomes `ns1.example.com.example.com.`).
- **CNAME at the zone apex** or CNAME with other records at the same name. Not allowed.
- **No secondary authoritative server.** A single point of failure.
- **MX or NS records pointing to a CNAME.** Point them at A/AAAA names.
- **Running DHCP on a subnet where another DHCP server is active.** Use a single server, failover, or relay design.
- **DHCP pool overlapping static addresses.** Exclude static ranges.
- **Skipping `named-checkzone`/`named-checkconf`/`kea-dhcp4 -t`** before a reload.
- **Chroot or SELinux contexts for BIND zone files** not set correctly, so `named` cannot read or write them (`restorecon`, correct directory such as `/var/named`).
- **Low TTLs forever** create load. Raise TTLs after a migration is finished.
- **Reverse DNS neglected** (PTR missing). Mail servers and some services rely on it.

## Further reading

- [BIND 9 Administrator Reference Manual](https://bind9.readthedocs.io)
- [Unbound documentation](https://unbound.docs.nlnetlabs.nl)
- [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html)
- [Kea documentation](https://kea.readthedocs.io)
- [RFC 1034 / 1035: DNS](https://datatracker.ietf.org/doc/html/rfc1035)
