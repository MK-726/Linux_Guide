# NSS: Name Service Switch

## Overview

**NSS** (Name Service Switch) is the C library mechanism that tells programs **where to look up** information such as users, groups, host names, and services. It answers *"Who is user X? What is their UID, home directory, and shell? Which groups exist?"*

NSS and PAM work together but do different jobs:

| | **NSS** | **PAM** |
|---|---------|---------|
| Question | **Who/what exists?** (identity and attribute lookup) | **May this user log in? Is the password correct?** (authentication and session rules) |
| Examples | `id`, `ls -l` (owner names), `getent`, `chown alice` | `login`, `sshd`, `sudo` password prompts |
| Configuration | `/etc/nsswitch.conf` | `/etc/pam.d/*` |

Central identity needs **both**: NSS to find the user in LDAP/AD, and PAM to authenticate.

## Key concepts

### /etc/nsswitch.conf

Each line names a **database** and the ordered list of **sources**:

```
passwd:     files systemd sss
group:      files systemd sss
shadow:     files sss
hosts:      files mdns4_minimal [NOTFOUND=return] dns myhostname
services:   files sss
netgroup:   files sss
automount:  files sss
sudoers:    files sss
```

### Databases

| Database | Content | Typical source files |
|----------|---------|----------------------|
| `passwd` | Users | `/etc/passwd` |
| `group` | Groups | `/etc/group` |
| `shadow` | Password hashes | `/etc/shadow` |
| `hosts` | Host names and IP addresses | `/etc/hosts`, DNS |
| `services` | Port names | `/etc/services` |
| `protocols` | Protocol names | `/etc/protocols` |
| `networks` | Network names | `/etc/networks` |
| `netgroup` | Groups of hosts/users | `/etc/netgroup` |
| `automount` | Automounter maps | `/etc/auto.master` |
| `sudoers` | sudo rules (when stored centrally) | `/etc/sudoers` |

### Sources

| Source | Meaning |
|--------|---------|
| `files` | Local files in `/etc` |
| `systemd` | Dynamic users and groups from systemd (`DynamicUser=`, `systemd-userdbd`) |
| `sss` | **SSSD**: LDAP, AD, FreeIPA (see [SSSD](10-sssd-and-centralized-identity-management.md)) |
| `ldap` | Direct LDAP via `nss_ldap`/`nslcd` (legacy) |
| `winbind` | Samba winbind (AD) |
| `dns` | DNS (for `hosts`) |
| `resolve` | systemd-resolved |
| `myhostname` | The local host name and `localhost` |
| `mdns4_minimal` | Multicast DNS (`.local` names) |
| `nis` | NIS (obsolete) |

### Lookup order and actions

Sources are tried **left to right**. By default, if a source does **not find** an entry, the next one is tried. You can change this with an **action** in brackets:

```
hosts: files dns                              # try files, then DNS
hosts: files [NOTFOUND=return] dns            # if files says "not found", stop (do not use DNS)
passwd: files sss [UNAVAIL=continue]
```

| Status | Meaning |
|--------|---------|
| `SUCCESS` | Entry found |
| `NOTFOUND` | Source works but has no entry |
| `UNAVAIL` | Source is not available (service down) |
| `TRYAGAIN` | Temporary failure |

| Action | Meaning |
|--------|---------|
| `return` | Stop and return the result |
| `continue` | Try the next source |
| `merge` | Combine results from sources (for `group`) |

**Order is a security decision.** With `passwd: files sss`, local accounts (including root) win over central ones, so a central directory cannot override local `root`.

### Caching

Lookups happen very often, so caches are used:

| Cache | Notes |
|-------|-------|
| **SSSD** | Its own cache (`/var/lib/sss/db/`). Handles offline use |
| **nscd** | Old name service cache daemon (can conflict with SSSD. Do not run both for the same databases) |
| **nsncd** | A newer compatible cache (some distributions) |
| **systemd-resolved** | Caches DNS |

## Commands

```console
$ grep -v '^#' /etc/nsswitch.conf | grep -v '^$'
$ getent passwd alice                    # look up through ALL configured sources
alice:x:1000:1000:Alice Smith:/home/alice:/bin/bash
$ getent passwd | wc -l                  # enumerates only sources that allow it (SSSD often does not)
$ getent group developers
$ getent hosts www.example.com
$ getent services https
$ getent passwd alice@example.com        # a domain user (with AD / SSSD)
$ getent -s files passwd alice           # ask ONE source only
$ getent -s sss passwd alice
$ id alice                               # uses NSS
$ ls /usr/lib/x86_64-linux-gnu/libnss_* /usr/lib64/libnss_* 2>/dev/null      # installed NSS modules
$ ldd $(which ls) | grep libc            # programs use NSS through glibc
```

### Why `/etc/passwd` is not the whole picture

```console
$ grep alice /etc/passwd            # no output
$ getent passwd alice               # shows alice → she comes from LDAP/AD/systemd
```

Always use **`getent`** to check what the system really sees. Direct `grep` only reads the local file.

### Trace a lookup problem

```console
$ strace -f -e trace=openat,connect getent passwd alice 2>&1 | grep -E 'nss|sss|ldap|passwd'
$ systemctl status sssd nscd 2>/dev/null
$ sssctl user-checks alice          # SSSD: NSS and PAM view of a user (see the SSSD page)
$ sss_cache -E                      # clear the SSSD cache for all entries
```

## Common pitfalls

- **Editing `nsswitch.conf` and breaking logins.** A wrong line (for example, removing `files` from `passwd`) can stop root lookups. Keep a root shell open and `cp /etc/nsswitch.conf /etc/nsswitch.conf.bak`.
- **Using `grep` on `/etc/passwd`** instead of `getent`.
- **Running `nscd` and SSSD caches together** causes stale and inconsistent results. Disable `nscd` for `passwd`, `group`, `netgroup`, and `services` when you use SSSD.
- **Directory outage and slow logins.** If the LDAP server is unreachable, lookups wait for timeouts. Use SSSD (caches and handles offline mode), short timeouts, and `[UNAVAIL=return]` thoughtfully.
- **UID/GID collisions** between local and central accounts. Reserve ranges (for example local UIDs below 1000 and central UIDs from 100000+).
- **`getent passwd` shows only local users** when enumeration is disabled in SSSD (`enumerate = false`, the default and a good setting). Look up specific names.
- **Expecting `hosts:` changes to affect `dig`.** `dig` talks to DNS directly. Applications use NSS. See [DNS Resolution](../../part-3-networking/11-networking-fundamentals/07-dns-resolution-and-hostname.md).
- **`sudoers: files sss`** needs SSSD to provide the `sudo` service as well.

## Further reading

- [`nsswitch.conf(5)`](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)
- [`getent(1)`](https://man7.org/linux/man-pages/man1/getent.1.html)
- [GNU C Library: Name Service Switch](https://www.gnu.org/software/libc/manual/html_node/Name-Service-Switch.html)
- [`nss-systemd(8)`](https://man7.org/linux/man-pages/man8/nss-systemd.8.html)
