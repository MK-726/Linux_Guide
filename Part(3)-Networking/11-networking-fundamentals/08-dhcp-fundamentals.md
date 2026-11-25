# DHCP Fundamentals

## Overview

**DHCP** (Dynamic Host Configuration Protocol) lets hosts get their network settings automatically: IP address, subnet mask, default gateway, DNS servers, and more. Almost every client network uses it. Servers often use static addresses or DHCP **reservations**.

## Key concepts

### The DORA process

A client without an address talks to a DHCP server using **broadcasts** on UDP ports **67 (server)** and **68 (client)**:

```
Client                                      Server
  │ ── DHCPDISCOVER (broadcast) ──────────► │  "Is there a DHCP server?"
  │ ◄──────────────────── DHCPOFFER ─────── │  "You can have 192.0.2.50"
  │ ── DHCPREQUEST (broadcast) ───────────► │  "I accept 192.0.2.50"
  │ ◄──────────────────────── DHCPACK ───── │  "Confirmed. Lease: 12 hours"
```

| Step | Meaning |
|------|---------|
| **D**iscover | Client looks for servers |
| **O**ffer | Server proposes an address and settings |
| **R**equest | Client selects one offer |
| **A**cknowledge | Server confirms the lease |

### Leases

An address is **leased** for a fixed time. The client renews at **50%** of the lease time (T1) with the same server, and tries other servers at **87.5%** (T2). If it cannot renew, it loses the address when the lease expires.

### DHCP options

Along with the address, the server sends **options**:

| Option | Meaning |
|--------|---------|
| 1 | Subnet mask |
| 3 | Router (default gateway) |
| 6 | DNS servers |
| 15 | Domain name |
| 119 | DNS search list |
| 42 | NTP servers |
| 12 | Hostname |
| 51 | Lease time |
| 66, 67 | TFTP server and boot file (PXE boot) |
| 121 | Classless static routes |

### Relay agents

Broadcasts do not cross routers. A **DHCP relay** (`ip helper-address` on a router) forwards requests to a central DHCP server on another subnet.

### Static, reserved, and dynamic

| Type | Description |
|------|-------------|
| **Static (manual)** | Configured on the host. The DHCP server must not hand out the same address (exclude it from the pool) |
| **DHCP reservation** | The server always gives the same address to a given MAC (or client ID). Central control with a stable address |
| **Dynamic** | Address from a pool |

### IPv6

IPv6 hosts normally use **SLAAC** (router advertisements). **DHCPv6** provides extra information (DNS, domain) or full addressing ("stateful") and uses UDP ports 546 and 547.

### DHCP clients on Linux

| Client | Where used |
|--------|-----------|
| **NetworkManager** (built-in client, or `dhclient`/`dhcpcd` helpers) | RHEL, Fedora, SUSE desktops, Ubuntu desktop |
| **systemd-networkd** (built-in) | Ubuntu Server (through netplan), many minimal systems |
| `dhclient` (ISC) | Legacy. The ISC DHCP project is end-of-life |
| `dhcpcd` | Some distributions and embedded systems |
| **wicked** | SLES 15 and openSUSE (traditional) |

## Commands

### See DHCP-assigned settings

```console
$ ip -br addr show enp3s0
enp3s0  UP  192.0.2.50/24
$ ip addr show enp3s0 | grep -E 'dynamic|valid_lft'
    inet 192.0.2.50/24 brd 192.0.2.255 scope global dynamic noprefixroute enp3s0
       valid_lft 42901sec preferred_lft 42901sec

$ nmcli -f DHCP4 device show enp3s0         # NetworkManager: options received
DHCP4.OPTION[1]:  dhcp_lease_time = 86400
DHCP4.OPTION[2]:  domain_name_servers = 192.0.2.53
DHCP4.OPTION[3]:  ip_address = 192.0.2.50
DHCP4.OPTION[4]:  routers = 192.0.2.1

$ networkctl status enp3s0                  # systemd-networkd: lease details
```

### Lease files

| Client | Location |
|--------|----------|
| NetworkManager (internal client) | `/var/lib/NetworkManager/*.lease` (or `internal-*.lease`) |
| systemd-networkd | `/run/systemd/netif/leases/` |
| `dhclient` | `/var/lib/dhcp/dhclient.leases` |

### Renew or release a lease

```console
# nmcli device reapply enp3s0               # NetworkManager: re-run configuration
# nmcli connection up "Wired connection 1"  # bring the profile up again (renews)
# networkctl renew enp3s0                   # systemd-networkd
# dhclient -r enp3s0 && dhclient enp3s0     # legacy dhclient: release and request
```

### Watch DHCP traffic

```console
# tcpdump -i enp3s0 -nn -v 'udp port 67 or udp port 68'
10:15:01 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request, length 300
10:15:01 IP 192.0.2.5.67 > 192.0.2.50.68: BOOTP/DHCP, Reply, length 300
```

You will see Discover, Offer, Request, and ACK. No Offer means no server reachable.

### Logs

```console
$ journalctl -u NetworkManager | grep -i dhcp
Mar 03 10:15:01 server01 NetworkManager[812]: <info>  dhcp4 (enp3s0): state changed new lease, address=192.0.2.50
$ journalctl -u systemd-networkd | grep -i dhcp
```

## Common pitfalls

- **`169.254.x.x` address.** The client got **no DHCP answer** and assigned itself a link-local address. Check the cable, VLAN, DHCP server, and relay.
- **Rogue DHCP server** (a home router plugged into the corporate network) gives wrong gateways and DNS. Use DHCP snooping on switches.
- **Pool exhausted.** No free addresses. Shorten lease times or enlarge the pool.
- **Address conflicts** between static hosts and the DHCP pool. Exclude static ranges.
- **Servers on DHCP without a reservation.** Their address changes. Use static addresses or reservations.
- **Firewall blocks UDP 67/68** (on the client or on the server).
- **Lease-time too long or too short.** Long leases keep old addresses for devices that left. Very short leases create traffic.
- **DHCP hostnames vs. DNS.** DHCP does not update DNS by itself. Dynamic DNS updates need configuration (for example, Kea with DDNS or dnsmasq).
- **DHCP and cloud-init.** Cloud images get network settings from the cloud platform's DHCP. Do not remove the client.
- **PXE and TFTP options** misconfigured produce boot failures.

## Further reading

- [RFC 2131: DHCP](https://datatracker.ietf.org/doc/html/rfc2131)
- [RFC 8415: DHCPv6](https://datatracker.ietf.org/doc/html/rfc8415)
- [Kea DHCP server (ISC)](https://kea.readthedocs.io)
- [IANA DHCP options](https://www.iana.org/assignments/bootp-dhcp-parameters/)
