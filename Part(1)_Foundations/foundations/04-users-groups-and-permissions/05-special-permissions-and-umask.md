# Special Permissions (SUID, SGID, Sticky Bit) and umask

## Overview

Besides `rwx`, Linux has three **special permission bits** and a mechanism (`umask`) that sets the default permissions for new files. Special bits are powerful and are a common source of security problems when used carelessly.

## Key concepts

### The special bits

| Bit | Octal | Symbol | On a **file** | On a **directory** |
|-----|-------|--------|---------------|---------------------|
| **SUID** (set user ID) | 4000 | `s` in the owner `x` position | Runs with the **file owner's** privileges | Ignored on Linux |
| **SGID** (set group ID) | 2000 | `s` in the group `x` position | Runs with the **file group's** privileges | New files inherit the **directory's group** |
| **Sticky bit** | 1000 | `t` in the others `x` position | Ignored | Only the file owner, directory owner, or root can delete or rename entries |

When the underlying `x` is missing, the symbol is uppercase (`S` or `T`). This usually signals a mistake.

### SUID example

`passwd` must write to `/etc/shadow`, which only root can modify. It is owned by root and has SUID, so any user running it gets root's rights **for that program only**.

```console
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 59976 Mar  3 09:00 /usr/bin/passwd
```

### SGID on directories: shared folders

With SGID on a directory, files created inside get the directory's group, not the creator's primary group. This is ideal for team folders.

```console
$ ls -ld /srv/project
drwxrwsr-x 2 root developers 4096 Mar  3 11:00 /srv/project
```

### Sticky bit example

`/tmp` is writable by everyone, but users must not delete each other's files. The sticky bit enforces that.

```console
$ ls -ld /tmp
drwxrwxrwt 12 root root 4096 Mar  3 11:05 /tmp
```

### Setting the bits

```console
# chmod u+s /usr/local/bin/tool       # SUID
# chmod g+s /srv/project              # SGID
# chmod +t /srv/shared                # sticky
# chmod 4755 file                     # SUID + rwxr-xr-x
# chmod 2775 dir                      # SGID + rwxrwxr-x
# chmod 1777 dir                      # sticky + rwxrwxrwx
```

### Finding SUID and SGID files (security audit)

```console
# find / -xdev -perm -4000 -type f 2>/dev/null     # SUID files
# find / -xdev -perm -2000 -type f 2>/dev/null     # SGID files
# find / -xdev -type d -perm -0002 ! -perm -1000   # world-writable directories without sticky bit
```

Compare the result with a known good list. Unexpected SUID programs deserve investigation.

> [!WARNING]
> SUID and SGID **scripts** are ignored by Linux for security reasons. SUID on programs owned by root increases risk: a bug in such a program can give an attacker root. Do not set SUID on editors, shells, `find`, `cp`, or interpreters. These give easy privilege escalation.

Mount options like `nosuid` disable SUID/SGID on a filesystem. Use it on `/tmp`, `/home`, and removable media where possible.

## umask: default permissions for new files

The **umask** removes permissions from the defaults.

| New item | Starting permissions |
|----------|----------------------|
| File | `666` (`rw-rw-rw-`) |
| Directory | `777` (`rwxrwxrwx`) |

**Result = default minus umask bits.**

| umask | New files | New directories | Notes |
|-------|-----------|-----------------|-------|
| `022` | `644` | `755` | Common default (for example, Debian and root on most systems) |
| `002` | `664` | `775` | Group-friendly; used with user private groups (normal users on RHEL-based systems and Ubuntu) |
| `027` | `640` | `750` | More restrictive |
| `077` | `600` | `700` | Private; only the owner |

```console
$ umask
0022
$ umask 077
$ touch private.txt && mkdir private-dir
$ ls -ld private.txt private-dir
-rw------- 1 alice alice    0 Mar  3 11:10 private.txt
drwx------ 2 alice alice 4096 Mar  3 11:10 private-dir
$ umask -S
u=rwx,g=,o=
```

New files are never created executable by default, because the starting point for files is `666`.

### Where umask is set

| Scope | Location |
|-------|----------|
| One shell session | `umask 027` |
| One user | `~/.bashrc` or `~/.profile` |
| All users (login shells) | `/etc/profile`, `/etc/bash.bashrc` (Debian), `/etc/bashrc` (RHEL), `/etc/login.defs` (`UMASK`) |
| A systemd service | `UMask=0027` in the unit file |
| A cron job | Set in the script or crontab |

## Common pitfalls

- **Uppercase `S` or `T`.** The special bit is set but `x` is not. Fix the underlying permission.
- **SGID does not change permissions**, only the group of new files. Add group write (`g+w`) as well.
- **A permissive umask in services** (for example `000`) creates world-writable files. Set `UMask=` in the unit.
- **umask only applies to new files.** It does not change existing ones.
- **Sticky bit missing on shared writable directories** lets users delete each other's files.

## Further reading

- [`chmod(1)`](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [`umask(2)`](https://man7.org/linux/man-pages/man2/umask.2.html)
- [`bash` builtin `umask`](https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html)
