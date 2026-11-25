# Chapter 13: Remote Access, Firewalls & Network Services

| # | Topic | Summary |
|---|-------|---------|
| 1 | [SSH Fundamentals](01-ssh-fundamentals.md) | How SSH works, connecting, host keys, `scp` and `sftp` |
| 2 | [SSH Client and Server Configuration](02-ssh-client-and-server-configuration.md) | `~/.ssh/config`, `sshd_config`, hardening |
| 3 | [SSH Keys and Authentication](03-ssh-keys-and-authentication.md) | Key pairs, `ssh-copy-id`, agent, certificates |
| 4 | [SSH Tunneling and Port Forwarding](04-ssh-tunneling-and-port-forwarding.md) | `-L`, `-R`, `-D`, jump hosts |
| 5 | [Firewall Concepts](05-firewall-concepts.md) | Netfilter, stateful filtering, chains, NAT, zones |
| 6 | [nftables](06-nftables.md) | The modern Linux packet filter |
| 7 | [iptables Overview](07-iptables-overview.md) | The classic tool and how it relates to nftables |
| 8 | [firewalld and UFW](08-firewalld-and-ufw.md) | Friendly front ends for RHEL-based and Ubuntu |
| 9 | [Common Network Services](09-common-network-services.md) | Web, mail, DNS, directory, and database services |
| 10 | [DNS and DHCP Server Concepts](10-dns-and-dhcp-server-concepts.md) | Authoritative vs. recursive, dnsmasq, Unbound, BIND, Kea |
| 11 | [NFS and SMB/CIFS](11-nfs-and-smb.md) | Network file sharing for Unix and Windows clients |
| 12 | [Port Management and Service Exposure](12-port-management-and-service-exposure.md) | Know, limit, and verify what is reachable |

> [!WARNING]
> Firewall and SSH changes on a **remote** server can lock you out. Always keep a second session open, have console access, and use a rollback plan.

**Previous:** [Chapter 12](../12-network-troubleshooting/README.md) | **Next part:** Part 4: Linux Security & Hardening (planned)
