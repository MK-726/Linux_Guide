# firewalld and UFW

## Overview

**firewalld** (default on RHEL-based and SUSE-based systems) and **UFW** (the Ubuntu helper) manage the firewall with simple commands instead of low-level rules. Both create nftables (or iptables) rules for you. Use the one your distribution provides, and do not mix it with hand-written rules.

## Key concepts: firewalld

| Concept | Meaning |
|---------|---------|
| **Zone** | A trust level with its own allowed services. Interfaces and source networks are assigned to zones |
| **Service** | A named group of ports and protocols (`ssh`, `http`, `https`) |
| **Runtime vs. permanent** | **Runtime** changes apply now and are lost at reload or reboot. **Permanent** changes are stored and apply after `--reload` or reboot |
| **Rich rule** | A more detailed rule (source address, logging, rate limits) |
| **Default zone** | Zone for interfaces without a specific assignment (usually `public`) |

### Standard zones (least to most trusted)

| Zone | Meaning |
|------|---------|
| `drop` | Drop all incoming traffic silently |
| `block` | Reject all incoming traffic (with an ICMP message) |
| `public` | Untrusted networks. Only selected services (default: `ssh`, `dhcpv6-client`, `cockpit` on some systems) |
| `external` | External router interface with masquerading |
| `dmz` | Limited access servers |
| `work`, `home`, `internal` | More trusted networks |
| `trusted` | Everything allowed |

## Commands: firewalld

Check state:

```console
# systemctl enable --now firewalld
# firewall-cmd --state
running
# firewall-cmd --get-default-zone
public
# firewall-cmd --get-active-zones
public
  interfaces: enp3s0
# firewall-cmd --list-all                       # settings of the default (or --zone=NAME)
public (active)
  target: default
  interfaces: enp3s0
  services: cockpit dhcpv6-client ssh
  ports:
  rich rules:
# firewall-cmd --get-services | tr ' ' '\n' | head
# firewall-cmd --info-service=https
```

### Allow and remove services and ports

```console
# firewall-cmd --add-service=https                         # runtime only (test it)
# firewall-cmd --add-service=https --permanent             # permanent (needs --reload)
# firewall-cmd --reload                                    # apply permanent config (runtime changes are discarded)
# firewall-cmd --runtime-to-permanent                      # save the current runtime state as permanent

# firewall-cmd --permanent --add-port=8080/tcp
# firewall-cmd --permanent --add-port=60000-60100/udp
# firewall-cmd --permanent --remove-service=cockpit
# firewall-cmd --permanent --remove-port=8080/tcp
# firewall-cmd --query-service=https                       # yes/no
# firewall-cmd --add-service=http --timeout=10m            # temporary rule that expires
```

> [!TIP]
> **Safe workflow:** add the rule at **runtime**, test it, then run `firewall-cmd --runtime-to-permanent`. If you lock yourself out, a reload or reboot restores the saved configuration.

### Zones and sources

```console
# firewall-cmd --permanent --zone=internal --add-source=192.0.2.0/24       # whole network to a zone
# firewall-cmd --permanent --zone=internal --add-service=ssh
# firewall-cmd --permanent --zone=public --remove-service=ssh              # SSH only from internal
# firewall-cmd --permanent --zone=trusted --add-interface=wg0
# firewall-cmd --permanent --zone=public --change-interface=enp3s0
# firewall-cmd --set-default-zone=internal
# firewall-cmd --reload
```

### Rich rules

```console
# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.0.2.0/24" service name="ssh" accept'
# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.9" reject'
# firewall-cmd --permanent --add-rich-rule='rule service name="ssh" limit value="5/m" accept'
# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="198.51.100.0/24" port port="5432" protocol="tcp" accept'
# firewall-cmd --list-rich-rules
```

### NAT and port forwarding

```console
# firewall-cmd --permanent --zone=external --add-masquerade
# firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=10.0.0.5
```

### Custom service

Create `/etc/firewalld/services/myapp.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>MyApp</short>
  <description>My application</description>
  <port protocol="tcp" port="8443"/>
</service>
```

```console
# firewall-cmd --reload
# firewall-cmd --permanent --add-service=myapp
```

### Emergency and debugging

```console
# firewall-cmd --panic-on          # drops ALL traffic (will cut your SSH session!)
# firewall-cmd --panic-off
# firewall-cmd --set-log-denied=all        # log dropped packets (off, all, unicast, ...)
# journalctl -k | grep -i REJECT
# nft list ruleset | less                  # what firewalld created (nftables backend)
```

| Path | Purpose |
|------|---------|
| `/etc/firewalld/firewalld.conf` | Main settings (default zone, backend) |
| `/etc/firewalld/zones/*.xml` | Your zone configuration |
| `/etc/firewalld/services/*.xml` | Your services |
| `/usr/lib/firewalld/` | Vendor defaults (do not edit) |

---

## Key concepts: UFW

**UFW** (Uncomplicated Firewall) is a simple front end, mostly used on Ubuntu. It is **installed but disabled** by default on Ubuntu. Rules are processed top to bottom, and UFW writes IPv4 and IPv6 rules together (when `IPV6=yes` in `/etc/default/ufw`).

## Commands: UFW

> [!IMPORTANT]
> **Allow SSH before you enable UFW.** Enabling it with default deny and no SSH rule cuts off remote access.

```console
# ufw allow OpenSSH                         # or: ufw allow 22/tcp
# ufw default deny incoming
# ufw default allow outgoing
# ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup

# ufw status verbose
Status: active
Default: deny (incoming), allow (outgoing), disabled (routed)
To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
80,443/tcp                 ALLOW IN    Anywhere
```

| Command | Purpose |
|---------|---------|
| `ufw allow 80/tcp` | Allow a port and protocol |
| `ufw allow 8000:8100/tcp` | Port range |
| `ufw allow http` / `ufw allow "Nginx Full"` | Service name or **application profile** |
| `ufw allow from 192.0.2.0/24 to any port 22 proto tcp` | Only from a network |
| `ufw allow from 192.0.2.10` | Everything from one host |
| `ufw deny from 203.0.113.9` | Block a host |
| `ufw limit ssh` | Allow, but rate-limit repeated connections (6 in 30 seconds) |
| `ufw status numbered` | Rules with numbers |
| `ufw delete 3` / `ufw delete allow 80/tcp` | Delete a rule |
| `ufw insert 1 deny from 203.0.113.9` | Insert at a position |
| `ufw route allow in on eth0 out on eth1` | Forwarded traffic |
| `ufw logging on` | Enable logging (`low`, `medium`, `high`) |
| `ufw reload` | Reload |
| `ufw disable` | Turn off |
| `ufw reset` | Delete **all** rules and disable |
| `ufw app list` / `ufw app info "Nginx Full"` | Application profiles (`/etc/ufw/applications.d/`) |

| Path | Purpose |
|------|---------|
| `/etc/default/ufw` | Defaults (IPv6, default policies) |
| `/etc/ufw/user.rules`, `user6.rules` | Your rules (written by UFW) |
| `/etc/ufw/before.rules`, `after.rules` | Rules applied before and after yours (advanced) |
| `/var/log/ufw.log` | Log (if logging is on) |

## Common pitfalls

- **Lockout when enabling UFW or `firewall-cmd --panic-on`.** Allow SSH first. Keep a session open.
- **Forgetting `--permanent`** in firewalld. The rule disappears at the next reload. Or the reverse: adding permanent rules and forgetting `--reload` so they are not active.
- **Reloading discards runtime-only changes** in firewalld.
- **Wrong zone.** An interface sits in `public` while you added the service to `internal`. Check `firewall-cmd --get-active-zones`.
- **A service is allowed but the application does not listen** (or listens on `127.0.0.1`). The firewall is not the problem then. Check `ss -tlnp`.
- **UFW rule order.** The first match wins. Use `ufw status numbered` and `ufw insert`.
- **Docker bypasses UFW and firewalld.** Published container ports (`-p 8080:80`) are added in the `nat` table and can be reachable even if UFW "denies" them. Bind to loopback (`-p 127.0.0.1:8080:80`), or manage rules in the `DOCKER-USER` chain, or use a front proxy.
- **Using UFW or firewalld together** with custom iptables/nft rules or each other. Use one.
- **IPv6 disabled in UFW** (`IPV6=no`) while the host has IPv6 connectivity.
- **Using `ufw reset` and `ufw disable` in production** casually. Both remove protection.
- **SELinux and firewalld** are separate. A port can be open in the firewall but blocked by SELinux policy (`semanage port`).

## Further reading

- [firewalld documentation](https://firewalld.org/documentation/)
- [`firewall-cmd(1)`](https://man7.org/linux/man-pages/man1/firewall-cmd.1.html)
- [`firewalld.richlanguage(5)`](https://firewalld.org/documentation/man-pages/firewalld.richlanguage.html)
- [Ubuntu: UFW](https://documentation.ubuntu.com/server/how-to/security/firewalls/)
- [`ufw(8)`](https://manpages.ubuntu.com/manpages/noble/man8/ufw.8.html)
