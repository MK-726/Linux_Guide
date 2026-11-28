# Active Directory Integration

## Overview

Many organizations manage identities in **Microsoft Active Directory (AD)**. Linux servers can **join the AD domain**, so AD users and groups can log in, administrators manage access centrally, and single sign-on with **Kerberos** works. The recommended method on current distributions is **`realmd` + SSSD**. This page shows the prerequisites, the join, and troubleshooting. For SSSD itself see [SSSD and Centralized Identity Management](10-sssd-and-centralized-identity-management.md).

## Key concepts

### What AD provides

| Component | Protocol | Role |
|-----------|----------|------|
| **Directory** | LDAP / LDAPS (389, 636, GC 3268/3269) | Users, groups, computers |
| **Authentication** | **Kerberos** (88/tcp+udp, 464 for password change) | Tickets and single sign-on |
| **DNS** | 53 | Locates domain controllers through **SRV records** |
| **File and RPC services** | SMB (445), RPC (135 + dynamic) | Samba/Winbind joins, group policy |
| **Time** | NTP (123/udp) | Kerberos requires clocks within **5 minutes** by default |

### Vocabulary

| Term | Meaning |
|------|---------|
| **Domain** | `example.com` (DNS name), **realm** `EXAMPLE.COM` (Kerberos name, uppercase) |
| **Domain controller (DC)** | AD server |
| **sAMAccountName / UPN** | Login names: `alice` / `alice@example.com` |
| **SID** | Security identifier. Unique ID of users and groups |
| **Computer account** | The Linux host's object in AD (created at join) |
| **Keytab** | `/etc/krb5.keytab`: the host's Kerberos secret |
| **GPO** | Group Policy Object (SSSD can honor logon rights from GPO) |

### Two ways to integrate

| | **SSSD (via `realmd`/`adcli`)** | **Winbind (Samba)** |
|---|--------------------------------|---------------------|
| Recommended for | **Logins, sudo, SSH on servers and workstations** | Servers that also act as **Samba file servers** (shares with AD ACLs) |
| Join tool | `realm join` (uses `adcli`) | `net ads join` |
| Notes | Lighter, caches credentials, offline logins | Needed for some Samba features |

### How user and group IDs are created

Linux needs numeric UIDs and GIDs. Two options:

| Method | How | Notes |
|--------|-----|-------|
| **Algorithmic ID mapping** (`ldap_id_mapping = True`, default) | SSSD computes UID/GID from the user's **SID** | Easy. Consistent across all SSSD clients with the same configuration. UIDs are large (for example 1,000,000+) |
| **POSIX attributes in AD** (`ldap_id_mapping = False`) | Uses `uidNumber`, `gidNumber`, `unixHomeDirectory`, `loginShell` stored in AD | Needed for **NFS** consistency or existing UIDs. Requires the attributes to be populated |

## Commands

### 1. Prerequisites (do these first)

```console
# hostnamectl set-hostname server01.example.com        # FQDN
$ chronyc tracking | head -3                             # time within a few seconds of the DC
$ resolvectl status | grep -i 'DNS Servers'              # DNS must point to AD DNS servers (or forward the domain to them)
$ dig +short -t SRV _ldap._tcp.dc._msdcs.example.com     # SRV records visible?
0 100 389 dc1.example.com.
$ dig +short -t SRV _kerberos._tcp.example.com
$ nc -zv dc1.example.com 88 389 636 445 464              # ports open?
```

Open on the firewall between the host and the DC: 53, 88, 123 (UDP), 389, 445, 464, 636, 3268/3269, and RPC if needed. **DNS and time are the top causes of join failures.**

### 2. Install packages

**RHEL-based**

```console
# dnf install realmd sssd sssd-tools adcli samba-common-tools oddjob oddjob-mkhomedir krb5-workstation
```

**Debian-based**

```console
# apt install realmd sssd sssd-tools adcli libnss-sss libpam-sss packagekit samba-common-bin krb5-user oddjob oddjob-mkhomedir
```

### 3. Discover and join

```console
# realm discover example.com
example.com
  type: kerberos
  realm-name: EXAMPLE.COM
  domain-name: example.com
  configured: no
  server-software: active-directory
  client-software: sssd
  required-package: sssd-tools
  required-package: adcli

# realm join --user=svc-join example.com                        # prompts for the password
# realm join --user=svc-join --computer-ou="OU=Linux,OU=Servers,DC=example,DC=com" example.com    # choose the OU
# realm list
example.com
  configured: kerberos-member
  login-formats: %U@example.com
  login-policy: allow-realm-logins
```

Use a **dedicated account with delegated rights** to create computer objects in a specific OU, not a Domain Admin.

### 4. Enable home directory creation

```console
# authselect select sssd with-mkhomedir --force            # RHEL-based
# systemctl enable --now oddjobd
# pam-auth-update --enable mkhomedir                       # Debian-based
```

### 5. Test

```console
$ id alice@example.com
uid=1259401103(alice@example.com) gid=1259400513(domain users@example.com) groups=1259400513(domain users@example.com),1259403201(linux-admins@example.com)
$ getent passwd alice@example.com
$ getent group linux-admins@example.com
$ ssh alice@example.com@server01.example.com            # login name is alice@example.com, host is server01.example.com
$ su - alice@example.com
# kinit alice@EXAMPLE.COM && klist
Ticket cache: KCM:1259401103
Default principal: alice@EXAMPLE.COM
Valid starting     Expires            Service principal
03/03/25 10:00:00  03/03/25 20:00:00  krbtgt/EXAMPLE.COM@EXAMPLE.COM
# klist -k                                               # host keytab entries
# sssctl domain-status example.com
```

### 6. Control who can log in

By default `realm join` permits **all** domain users. Restrict it:

```console
# realm deny --all
# realm permit --groups 'linux-admins@example.com' 'linux-users@example.com'
# realm permit alice@example.com
# realm permit --all                  # (back to all) or realm deny --all
```

This writes `simple_allow_groups` or `simple_allow_users` into `sssd.conf`. For centralized control, use AD **GPO-based access control** (`ad_gpo_access_control`) or FreeIPA HBAC rules.

Grant `sudo` to an AD group:

```
# /etc/sudoers.d/50-ad-admins
%linux-admins@example.com ALL=(ALL) ALL
```

Escape spaces in group names with a backslash (`%domain\ admins@example.com`). Or use central sudo rules from LDAP.

### 7. The generated configuration

`/etc/sssd/sssd.conf` (must be `root:root`, mode `0600`):

```ini
[sssd]
domains = example.com
config_file_version = 2
services = nss, pam, ssh

[domain/example.com]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = EXAMPLE.COM
realmd_tags = manages-system joined-with-adcli
id_provider = ad
fallback_homedir = /home/%u@%d
ad_domain = example.com
use_fully_qualified_names = True
ldap_id_mapping = True
access_provider = simple
simple_allow_groups = linux-admins@example.com, linux-users@example.com
```

| Setting | Meaning |
|---------|---------|
| `id_provider = ad` | Look up users and groups in AD |
| `use_fully_qualified_names = True` | Login names are `user@domain` (avoids clashes with local accounts). `False` allows plain `alice` for a single domain |
| `fallback_homedir = /home/%u@%d` | Home directory pattern. Use `/home/%u` with FQN off |
| `ldap_id_mapping = True` | UIDs derived from SIDs |
| `access_provider = simple`, `ad` | Local allow/deny lists, or AD-based (`ad_gpo_access_control = enforcing`) |
| `cache_credentials = True` | Allow offline logins |
| `ad_gpo_access_control` | `enforcing`, `permissive` (log only), `disabled` for GPO logon rights |

After changes: `systemctl restart sssd` (and `sss_cache -E` to clear cached data).

### 8. SSH single sign-on with Kerberos (optional)

```
# /etc/ssh/sshd_config.d/50-gssapi.conf
GSSAPIAuthentication yes
```

```console
$ kinit alice@EXAMPLE.COM
$ ssh -o GSSAPIAuthentication=yes server01.example.com            # no password prompt
```

### 9. Samba/Winbind (file server) alternative

Use `net ads join -U svc-join` with `security = ads`, `realm = EXAMPLE.COM`, `workgroup = EXAMPLE`, and `idmap config` lines in `smb.conf`, with `winbind`. This is needed for AD-aware Samba shares. See Samba and Red Hat documentation, and [NFS and SMB](../../part-3-networking/13-remote-access-firewalls-services/11-nfs-and-smb.md).

### 10. Leave the domain

```console
# realm leave example.com                  # removes SSSD configuration; add --remove to also delete the AD computer object (needs rights)
```

## Troubleshooting

| Symptom | Checks |
|---------|--------|
| `realm discover` finds nothing | DNS: `dig SRV _ldap._tcp.dc._msdcs.DOMAIN`. Point resolvers at AD DNS or add a forwarder |
| `Couldn't authenticate to active directory: ... Clock skew too great` | Fix time (`chronyc`, ensure the DC and client sync to a good source) |
| `Insufficient permissions to join the domain` | The join account lacks "create computer objects" on the OU |
| `Cannot find KDC for realm` | `/etc/krb5.conf` realm name wrong (must be uppercase), DNS SRV for `_kerberos`, firewall 88 |
| Join works but `id user@domain` fails | `systemctl status sssd`, `journalctl -u sssd`, check `sssd.conf` permissions (`0600`), domain name case, `sss_cache -E` |
| User found but login denied | `access_provider`, `simple_allow_groups`, GPO, expired or locked AD account, logon hours; check `sssctl user-checks alice@example.com` |
| Login works but no home directory | `mkhomedir`/`oddjobd` not enabled, SELinux context, `fallback_homedir` path |
| Group membership missing | Nested groups, cache. Wait or `sss_cache -E`. Large groups may need `ldap_group_nesting_level` or `ad_enable_gc` |
| Slow logins | DNS or unreachable DC, trusted-domain lookups (`subdomains_provider = none`), too many groups. See SSSD logs |
| Only some DCs reachable | Use `ad_server = dc1.example.com, dc2.example.com`, `ad_site` |
| After DC password rotation, host loses trust | Machine password renewal. `adcli` and `sssd` renew automatically. Test with `adcli testjoin` |
| `kinit` OK but `ssh` still asks for a password | `GSSAPIAuthentication` off, client uses the wrong host name, no `host/` principal in the keytab (`klist -k`) |

```console
# adcli testjoin                                 # is the machine account valid?
# adcli info example.com
# sssctl config-check
# sssctl logs-fetch /tmp/sssd-logs.tar           # package debug logs (after setting debug_level)
# sss_debuglevel 6                               # raise logging for all sssd components
# less /var/log/sssd/sssd_example.com.log
```

## Common pitfalls

- **DNS and time not prepared** before joining. They cause most failures.
- **Join with a Domain Admin account** on routine servers. Delegate rights on an OU.
- **Everything in the default `Computers` container.** Use OUs so GPOs and permissions can be applied.
- **`realm permit --all`** left in place, so every domain user can log in to every server.
- **Mixed UID mappings** across servers (some algorithmic, some POSIX attributes). NFS and file ownership become inconsistent. Choose one method for the environment.
- **Mixed-case realm names.** Kerberos realms are uppercase. DNS domains are lowercase.
- **Using NTLM-era assumptions.** Prefer Kerberos and LDAPS. Disable weak settings per your AD policy.
- **`sssd.conf` permission errors** keep the service from starting.
- **Not planning offline access.** Servers should still allow local administrator access if AD is unreachable (break-glass account).
- **Treating the cache as truth** during troubleshooting. Clear it after changing group memberships.
- **Cloned virtual machines** share the same computer account and keytab. Rejoin each clone (sysprep-like process with a unique host name).
- **Time zone vs. time sync confusion.** Kerberos needs correct **UTC time**, not a matching time zone.

## Further reading

- [Red Hat: integrating RHEL with Active Directory](https://docs.redhat.com)
- [`realm(8)`](https://man7.org/linux/man-pages/man8/realm.8.html)
- [`sssd-ad(5)`](https://man7.org/linux/man-pages/man5/sssd-ad.5.html)
- [`adcli`](https://www.freedesktop.org/software/realmd/adcli/adcli.html)
- [Microsoft: AD ports and protocols](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/config-firewall-for-ad-domains-and-trusts)
- [Samba wiki: Setting up Samba as a domain member](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Domain_Member)
