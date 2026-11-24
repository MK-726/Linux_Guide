# Linux User and Group Concepts

## Overview

Linux is a **multi-user** system. Every process and every file belongs to a user and a group. The kernel uses these identities to decide who may read, write, or run what. Understanding users and groups is the base for all access control.

## Key concepts

### Users

A **user account** represents a person or a service. The kernel does not use names. It uses numbers:

- **UID** (user ID): the number that identifies a user.
- **GID** (group ID): the number that identifies a group.

Names are only labels that map to these numbers.

| Account type | Typical UID range | Purpose |
|--------------|-------------------|---------|
| **root** | `0` | The superuser. Can do anything. |
| **System accounts** | `1`–`999` | Services such as `www-data`, `sshd`, `postgres`. Usually no login shell. |
| **Normal users** | `1000` and up | People |
| **Nobody** | `65534` | Unprivileged placeholder account (`nobody`) |

> [!NOTE]
> The exact ranges come from `/etc/login.defs` (`UID_MIN`, `SYS_UID_MAX`). Very old RHEL versions started normal users at 500.

### The root user

`root` (UID 0) bypasses most permission checks. Because of this:

- Do not log in as root for daily work.
- Use a normal account and `sudo` for administrative commands.
- Many servers disable direct root login over SSH.

### Groups

A **group** is a set of users that share access to resources.

| Group type | Description |
|------------|-------------|
| **Primary group** | Exactly one per user. New files created by the user get this group (unless a directory SGID bit changes it). |
| **Supplementary (secondary) groups** | Zero or more extra groups that grant additional access. |

**User Private Groups (UPG):** On most distributions, creating the user `alice` also creates a group `alice` as her primary group. This makes it safe to use a shared umask of `002` (see [Special Permissions and umask](05-special-permissions-and-umask.md)).

### Important groups

| Group | Purpose (varies by distribution) |
|-------|----------------------------------|
| `sudo` | Members can use `sudo` (Debian-based) |
| `wheel` | Members can use `sudo` (RHEL-based) |
| `adm` | Can read many log files (Debian-based) |
| `docker` | Can use the Docker daemon (equals root-level access) |
| `disk`, `shadow` | Direct access to disks or the shadow file; sensitive |
| `systemd-journal` | Can read the system journal |

> [!WARNING]
> Membership in groups like `docker`, `disk`, or `shadow` gives powers equal to or close to root. Add users to these groups with care.

### Real, effective, and saved IDs (overview)

A running process has several IDs. The most important are:

- **Real UID**: who started the process.
- **Effective UID**: which user's permissions the kernel checks. Programs with the SUID bit run with the file owner's effective UID.

## Commands

| Command | Purpose |
|---------|---------|
| `whoami` | Show your user name |
| `id` | Show your UID, GID, and all groups |
| `id alice` | Show the same for another user |
| `groups` | List your groups |
| `who` | Show logged-in users |
| `w` | Logged-in users and what they run |
| `last` | Login history |
| `getent passwd alice` | Look up a user (works with LDAP, SSSD, and local files) |
| `getent group sudo` | Look up a group |

```console
$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo),100(users)

$ whoami
alice

$ getent passwd alice
alice:x:1000:1000:Alice Smith:/home/alice:/bin/bash
```

## Common pitfalls

- **Assuming names, not numbers, matter.** Two systems may use different UIDs for the same name. This causes wrong ownership on shared storage (NFS) or restored backups. Keep UIDs consistent, or use central identity management.
- **Working as root all the time.** One mistyped command can damage the system. Use `sudo` per command.
- **Using `ls` or `cat /etc/passwd` to look up users.** Users from LDAP or Active Directory are not listed there. Use `getent`.

## Further reading

- [`credentials(7)`](https://man7.org/linux/man-pages/man7/credentials.7.html)
- [`login.defs(5)`](https://man7.org/linux/man-pages/man5/login.defs.5.html)
