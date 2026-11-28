# SSSD and Centralized Identity Management

## Overview

**SSSD** (System Security Services Daemon) is the standard Linux client for central identity. It connects to **LDAP, Active Directory, FreeIPA, or Kerberos**, **caches** users and credentials (so logins work when the server is unreachable), and gives NSS, PAM, `sudo`, SSH, and automounter one consistent service. **Centralized identity management** means one authoritative source for users, groups, and access rules instead of separate accounts on every server.

## Key concepts

### Why centralize

| Without central identity | With central identity |
|--------------------------|----------------------|
| Accounts and keys on every server | One account, available everywhere permitted |
| Departures need edits on N servers | **Disable once** |
| Inconsistent UIDs, groups, sudo rules | Consistent |
| Weak audit trail | Clear identity and policy in one place |
| Passwords per server | SSO (Kerberos), MFA enforced centrally |

### SSSD architecture

```
 NSS (id, getent, ls)  PAM (login, ssh, sudo)  sudo  ssh keys  autofs
         │                     │                 │      │        │
         └──── responders (nss, pam, sudo, ssh, autofs) ─────────┘
                                │
                           sssd (monitor) ── cache (/var/lib/sss/db)
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
        domain "example.com"                  other domains
   providers: id, auth, access, chpass, sudo, autofs, selinux, hostid
   backends:  ldap, krb5, ad, ipa, proxy, files
              │
              ▼
      LDAP / AD / FreeIPA servers
```

| Piece | Role |
|-------|------|
| **Responders** | Serve a client interface: `nss`, `pam`, `sudo`, `ssh`, `autofs`, `pac` |
| **Domain** | A connection to one identity source |
| **Providers** | `id_provider` (users and groups), `auth_provider` (passwords), `access_provider` (who may log in), `chpass_provider` (password change), `sudo_provider`, `autofs_provider`, `selinux_provider`, `hostid_provider` |
| **Cache** | Local database. Provides speed and **offline** authentication |

Common provider choices:

| Environment | Domain `id_provider` |
|-------------|----------------------|
| Active Directory | `ad` |
| FreeIPA / Red Hat IdM | `ipa` |
| Generic LDAP | `ldap` (+ `auth_provider = ldap` or `krb5`) |
| Local files (container use cases) | `files` |

## Commands and configuration

### Install

**RHEL-based:** `dnf install sssd sssd-tools sssd-ldap` (and `sssd-ad`, `sssd-ipa` as needed)
**Debian-based:** `apt install sssd sssd-tools libnss-sss libpam-sss` (+ `sssd-ldap`, `sssd-ad`, `sssd-ipa`)

### Example: generic LDAP domain with TLS

`/etc/sssd/sssd.conf` (must be `root:root`, **mode 0600**, or SSSD refuses to start):

```ini
[sssd]
services = nss, pam, ssh, sudo
domains = example.com
config_file_version = 2

[domain/example.com]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
sudo_provider = ldap
ldap_uri = ldaps://ldap1.example.com, ldaps://ldap2.example.com
ldap_search_base = dc=example,dc=com
ldap_default_bind_dn = uid=svc-lookup,ou=services,dc=example,dc=com
ldap_default_authtok = <password or obfuscated>
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/pki/ca-trust/source/anchors/corp-ca.pem
ldap_id_use_start_tls = False
cache_credentials = True
enumerate = False
access_provider = simple
simple_allow_groups = linux-users
ldap_schema = rfc2307bis
fallback_homedir = /home/%u
default_shell = /bin/bash
```

```console
# chown root:root /etc/sssd/sssd.conf && chmod 600 /etc/sssd/sssd.conf
# sssctl config-check                                   # validate the configuration
# systemctl enable --now sssd
```

Key settings:

| Setting | Meaning |
|---------|---------|
| `domains`, `services` | Which domains and responders are active |
| `ldap_uri` | Servers in failover order |
| `ldap_search_base` | Base DN |
| `ldap_default_bind_dn`, `ldap_default_authtok` | Read-only lookup account |
| `ldap_tls_reqcert = demand` | Require and verify the server certificate |
| `cache_credentials = True` | Offline logins (stores hashed credentials). Set `offline_credentials_expiration` for a limit |
| `enumerate = False` | Do not list all users on `getent passwd` (default, better performance and safer) |
| `access_provider` | `permit` (all), `deny`, `simple` (lists), `ldap` (filter), `ad`, `ipa` (HBAC) |
| `use_fully_qualified_names` | `user@domain` names (needed for multiple domains) |
| `ldap_schema` | `rfc2307` or `rfc2307bis` (how group members are stored) |
| `debug_level` | 0–9. Set to 6 temporarily for troubleshooting |

### Connect NSS and PAM

```
# /etc/nsswitch.conf
passwd:   files sss
group:    files sss
shadow:   files sss
sudoers:  files sss
```

**RHEL-based:** `authselect select sssd with-mkhomedir with-sudo --force`
**Debian-based:** `pam-auth-update --enable sss mkhomedir`

### SSH keys and sudo from the directory

```
# sshd_config.d/60-sss-keys.conf
AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys
AuthorizedKeysCommandUser nobody
```

`sudo` rules stored in LDAP (`sudoRole` objects) or FreeIPA are read through the `sudo` responder. See [sudoers Best Practices](05-sudoers-best-practices.md) and [SSH Authentication and Key Management](06-ssh-authentication-and-key-management.md).

### Operate and troubleshoot

| Command | Purpose |
|---------|---------|
| `sssctl domain-list` | Configured domains |
| `sssctl domain-status example.com` | Online/offline, active server |
| `sssctl user-checks alice` | **NSS and PAM view of a user**, group membership, access decision |
| `sssctl config-check` | Check `sssd.conf` |
| `sssctl cache-expire -u alice` / `-E` | Mark cache entries expired (refresh on next request) |
| `sss_cache -E` | Invalidate all cached entries |
| `sssctl cache-remove` | Remove the cache (stops SSSD) |
| `sssctl logs-fetch FILE` | Collect logs |
| `sssctl analyze request list` | Analyze lookups in the logs (with `debug_level`) |
| `sss_override` | Override attributes of a user locally |
| `getent passwd alice`, `id alice` | Lookup results |
| `journalctl -u sssd` and `/var/log/sssd/*.log` | Logs |

```console
$ sssctl domain-status example.com
Online status: Online
Active servers:
LDAP: ldap1.example.com
$ sssctl user-checks alice
user: alice
action: acct
service: system-auth
SSSD nss user lookup result:
 - user name: alice
 - user id: 10001
 - group id: 10001
 - home directory: /home/alice
SSSD InfoPipe user lookup result:
...
pam_acct_mgmt: Success
```

Clean restart after changes (also clears stale cache):

```console
# systemctl stop sssd && sss_cache -E && rm -f /var/lib/sss/db/* /var/lib/sss/mc/* && systemctl start sssd
```

Debugging: set `debug_level = 6` in `[domain/...]` (and `[nss]`, `[pam]` as needed), restart, reproduce the problem, read `/var/log/sssd/sssd_DOMAIN.log`, `sssd_nss.log`, `sssd_pam.log`. Reset the level afterwards.

### FreeIPA / Red Hat IdM in brief

**FreeIPA** bundles LDAP (389-DS), Kerberos (KDC), a CA, DNS, **HBAC** rules (which users may log in to which hosts and services), centrally managed **sudo rules**, SSH keys, **OTP**, and certificates. Clients join with `ipa-client-install` and use SSSD with `id_provider = ipa`.

```console
# dnf install ipa-client                             # or: apt install freeipa-client
# ipa-client-install --domain=example.com --realm=EXAMPLE.COM --mkhomedir
$ kinit admin
$ ipa user-add bob --first=Bob --last=Jones
$ ipa group-add-member developers --users=bob
$ ipa hbacrule-add allow_devs_ssh --hostcat=all --servicecat=all     # example; use tight rules in practice
$ ipa sudorule-add ...
```

### Design guidance for centralized identity

| Topic | Recommendation |
|-------|----------------|
| **Source of truth** | One authoritative IdP (AD, FreeIPA, or a cloud IdP with directory sync). Avoid parallel account databases |
| **Naming and IDs** | Unique user names. Plan **UID/GID ranges** (for example, local 1–999 system, 1000–9999 local users, 100000+ central) |
| **Authorization** | Groups per role. Map groups to **access (HBAC/allow groups)** and **sudo**. Avoid per-user exceptions |
| **Redundancy** | Two or more directory servers per site, `ldap_uri` failover lists, reliable DNS |
| **Offline resilience** | `cache_credentials = True`. A local **break-glass admin** account |
| **MFA and SSO** | Kerberos, OTP, or smart cards enforced centrally |
| **TLS everywhere** | LDAPS/StartTLS with verified certificates. Manage certificate renewal |
| **Lifecycle** | Joiner-mover-leaver automation from HR. Disable accounts centrally, then remove keys and tokens |
| **Service accounts** | Managed separately (no interactive login), reviewed, with expiry or rotation |
| **Auditing** | Log authentication centrally, review privileged group changes |
| **Cloud and SaaS** | Federation with SAML/OIDC (Keycloak, Entra ID, Okta). SCIM for provisioning |
| **Testing** | Test with a canary host before changing PAM/SSSD fleet-wide |

## Common pitfalls

- **`sssd.conf` permissions or syntax errors.** SSSD will not start. Check `sssctl config-check` and `journalctl -u sssd`.
- **Stale cache.** Group changes appear late. Use `sss_cache -E` or `sssctl cache-expire`. Tune `entry_cache_timeout`.
- **`enumerate = True`** on large directories slows everything and loads the server.
- **Local and central UID collisions.** Reserve distinct ranges.
- **Skipping TLS verification** (`ldap_tls_reqcert = never`) to make it work. Fix the CA and host names.
- **Wrong `ldap_schema`** causing missing group members (`rfc2307` vs. `rfc2307bis`/AD).
- **Running `nscd` alongside SSSD** for the same maps. Disable the overlapping caches.
- **Case sensitivity and name formats** (`alice` vs. `alice@example.com`). Check `use_fully_qualified_names` and `case_sensitive`.
- **Directory outage and nothing local.** Always keep a local administrator account and console access.
- **Large nested groups** and slow logins. Reduce nesting, use `ldap_group_nesting_level`, or restrict with `access_provider` filters.
- **Assuming SSSD handles everything.** `sudo`, SSH keys, and automount need their responders and configuration (`services = ... sudo, ssh`).
- **Changing PAM or `nsswitch.conf` by hand** on managed systems. Use `authselect` or `pam-auth-update`.
- **Time and DNS issues** with Kerberos-based setups (AD, FreeIPA).

## Further reading

- [`sssd.conf(5)`](https://man7.org/linux/man-pages/man5/sssd.conf.5.html)
- [SSSD documentation](https://sssd.io/docs/)
- [`sssctl(8)`](https://man7.org/linux/man-pages/man8/sssctl.8.html)
- [FreeIPA documentation](https://freeipa.readthedocs.io)
- [Red Hat: configuring authentication and authorization](https://docs.redhat.com)
- [NIST SP 800-63C: Federation and Assertions](https://pages.nist.gov/800-63-3/sp800-63c.html)
