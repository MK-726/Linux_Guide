# Account Files: /etc/passwd, /etc/shadow, /etc/group

## Overview

On a standalone Linux system, accounts are stored in plain text files in `/etc`. Learn their format and permissions: you will read them during troubleshooting, audits, and recovery.

> [!WARNING]
> Do not edit these files by hand unless you must. Use `vipw` and `vigr`, which lock the files while you edit. A syntax error can lock everyone out.

## Key concepts

### /etc/passwd

Readable by everyone. One line per account, seven fields separated by `:`.

```
alice:x:1000:1000:Alice Smith:/home/alice:/bin/bash
```

| # | Field | Example | Meaning |
|---|-------|---------|---------|
| 1 | Login name | `alice` | Account name |
| 2 | Password | `x` | `x` means the hash is in `/etc/shadow` |
| 3 | UID | `1000` | User ID |
| 4 | GID | `1000` | Primary group ID |
| 5 | GECOS | `Alice Smith` | Full name or comment |
| 6 | Home | `/home/alice` | Home directory |
| 7 | Shell | `/bin/bash` | Login shell (`nologin` or `false` blocks interactive login) |

### /etc/shadow

Readable only by root (and the `shadow` group on some systems). It stores password hashes and aging data. Nine fields:

```
alice:$y$j9T$abc...xyz:20150:1:90:14:::
```

| # | Field | Meaning |
|---|-------|---------|
| 1 | Login name | Matches `passwd` |
| 2 | Password hash | See below |
| 3 | Last change | Days since 1970-01-01 |
| 4 | Minimum age | Days before a change is allowed |
| 5 | Maximum age | Days until the password must change |
| 6 | Warning | Days of warning before expiry |
| 7 | Inactive | Days after expiry until the account locks |
| 8 | Expire | Account expiration (days since 1970-01-01) |
| 9 | Reserved | Not used |

Password field values:

| Value | Meaning |
|-------|---------|
| `$6$...` | SHA-512 hash |
| `$y$...` | yescrypt hash (default on newer Debian, Ubuntu, and Fedora) |
| `$5$...` | SHA-256 hash |
| `!` or `!!` | Password locked or not set. No password login is possible |
| `*` | No password login (typical for system accounts) |
| (empty) | **No password required.** Dangerous |

A `!` in front of a hash means the account is locked but the old hash is kept.

### /etc/group

Readable by everyone. Four fields:

```
developers:x:2001:alice,bob
```

| # | Field | Meaning |
|---|-------|---------|
| 1 | Group name | |
| 2 | Password | Almost always `x` (group passwords are rarely used) |
| 3 | GID | |
| 4 | Members | Comma-separated supplementary members |

A user's **primary** group is defined in `/etc/passwd` (field 4), so the user may not be listed in field 4 of `/etc/group` for that group.

`/etc/gshadow` stores group passwords and group administrators.

### Where else users may come from

The **Name Service Switch** (`/etc/nsswitch.conf`) decides which sources are used: local files, LDAP, SSSD, and others. Use `getent` to see the combined result.

```console
$ grep -E '^(passwd|group|shadow):' /etc/nsswitch.conf
passwd:         files systemd
group:          files systemd
shadow:         files
```

(See Chapter 15 for NSS, PAM, and centralized identity.)

## Commands

```console
$ cat /etc/passwd | wc -l                 # number of local accounts
$ awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd    # normal users
$ awk -F: '$3 == 0 {print $1}' /etc/passwd                      # accounts with UID 0 (should be only root)
$ grep -v -E 'nologin|false' /etc/passwd  # accounts that can log in

$ ls -l /etc/passwd /etc/shadow /etc/group
-rw-r--r-- 1 root root   2412 Mar  3 09:00 /etc/passwd
-rw-r----- 1 root shadow 1311 Mar  3 09:00 /etc/shadow
-rw-r--r-- 1 root root    998 Mar  3 09:00 /etc/group
```

Verify file consistency (**requires root**):

```console
# pwck           # check /etc/passwd and /etc/shadow
# grpck          # check /etc/group and /etc/gshadow
```

Edit safely:

```console
# vipw           # edit /etc/passwd
# vipw -s        # edit /etc/shadow
# vigr           # edit /etc/group
```

## Common pitfalls

- **More than one UID 0 account.** Any account with UID 0 is root. Check regularly.
- **Empty password field** in `/etc/shadow`. Lock it with `passwd -l` or set a password.
- **Editing with a normal editor** while another tool changes the file. Use `vipw`.
- **Wrong permissions.** `/etc/shadow` readable by everyone exposes password hashes to offline cracking. Fix with `chmod 640 /etc/shadow` (Debian: group `shadow`) or `chmod 000 /etc/shadow` (RHEL; root still reads it).

## Further reading

- [`passwd(5)`](https://man7.org/linux/man-pages/man5/passwd.5.html)
- [`shadow(5)`](https://man7.org/linux/man-pages/man5/shadow.5.html)
- [`group(5)`](https://man7.org/linux/man-pages/man5/group.5.html)
- [`nsswitch.conf(5)`](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)
