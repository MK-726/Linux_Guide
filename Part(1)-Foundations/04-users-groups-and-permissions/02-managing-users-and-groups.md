# User and Group Management

## Overview

Administrators create, change, lock, and remove accounts with a small set of standard tools. All commands on this page **require root** (or `sudo`).

## Commands

### Create a user

```console
# useradd -m -s /bin/bash -c "Alice Smith" alice
# passwd alice
New password:
Retype new password:
passwd: password updated successfully
```

| Option | Meaning |
|--------|---------|
| `-m` | Create the home directory (copies files from `/etc/skel`) |
| `-s SHELL` | Login shell |
| `-c TEXT` | Comment (full name) |
| `-d DIR` | Home directory path |
| `-u UID` | Choose the UID |
| `-g GROUP` | Primary group |
| `-G G1,G2` | Supplementary groups |
| `-r` | Create a **system** account |
| `-e DATE` | Account expiration date (`YYYY-MM-DD`) |

> [!NOTE]
> On **RHEL-based** systems, `useradd` creates the home directory by default. On **Debian-based** systems you need `-m`. Debian also provides the friendlier `adduser` command, which asks questions and does the setup for you.

Create a service account with no login and no home:

```console
# useradd -r -s /usr/sbin/nologin -d /var/lib/myapp myapp
```

On Debian, `/usr/sbin/nologin` is the path. On RHEL it is also `/sbin/nologin` (same file through the merged `/usr`).

### Modify a user

```console
# usermod -aG sudo alice         # add to the sudo group (Debian-based)
# usermod -aG wheel alice        # add to the wheel group (RHEL-based)
# usermod -s /bin/zsh alice      # change the shell
# usermod -L alice               # lock the password
# usermod -U alice               # unlock the password
# usermod -e 2025-12-31 alice    # set an expiration date
```

> [!WARNING]
> Always use `-a` with `-G`. `usermod -G sudo alice` **replaces** all supplementary groups with only `sudo`. `usermod -aG sudo alice` **adds** it.

Group membership changes apply at the **next login**. Check with `id alice`. The current session needs `newgrp` or a new login.

### Remove a user

```console
# userdel alice          # remove the account, keep the home directory
# userdel -r alice       # remove the account and the home directory and mail spool
```

> [!WARNING]
> `userdel -r` deletes the user's files permanently. Back up first. Files outside the home directory that belonged to the user stay and become owned by a bare UID number. Find them with `find / -nouser -xdev`.

Before removal, make sure the user has no running processes: `pgrep -u alice` or `loginctl terminate-user alice`.

### Passwords and aging

```console
# passwd alice               # set or change the password
# passwd -l alice            # lock (prefix hash with "!")
# passwd -u alice            # unlock
# passwd -e alice            # force a change at next login
# passwd -S alice            # status
alice P 03/03/2025 0 99999 7 -1
```

`chage` controls password aging:

| Option | Meaning |
|--------|---------|
| `chage -l alice` | Show the aging information |
| `chage -M 90 alice` | Maximum password age: 90 days |
| `chage -m 1 alice` | Minimum days between changes |
| `chage -W 14 alice` | Warn 14 days before expiry |
| `chage -E 2025-12-31 alice` | Account expiration date |
| `chage -d 0 alice` | Force a password change at next login |

```console
# chage -l alice
Last password change                  : Mar 03, 2025
Password expires                      : Jun 01, 2025
Password inactive                     : never
Account expires                       : never
Minimum number of days between changes: 1
Maximum number of days between changes: 90
Number of days of warning before expiry: 14
```

Locking a password does **not** stop SSH key logins. To block all access, also set the shell to `nologin` or expire the account (`usermod -e 1 alice`).

### Groups

```console
# groupadd developers
# groupadd -g 2000 ops              # with a specific GID
# groupmod -n engineers developers  # rename
# groupdel engineers                # delete (fails if it is a user's primary group)
# gpasswd -a alice developers       # add a user to a group
# gpasswd -d alice developers       # remove a user from a group
$ id alice
```

`newgrp GROUP` starts a shell with a different primary group for the current session.

### Debian helper commands

```console
# adduser alice                    # interactive user creation
# adduser alice developers         # add an existing user to a group
# deluser --remove-home alice
```

### Default values

New-user defaults come from:

| File | Controls |
|------|----------|
| `/etc/default/useradd` | Default shell, home base, and group |
| `/etc/login.defs` | UID/GID ranges, password aging, umask, home creation |
| `/etc/skel/` | Files copied to every new home directory |
| `/etc/adduser.conf` | Defaults for `adduser` (Debian-based) |

## Common pitfalls

- **Forgetting `-a` with `usermod -G`.** The user loses other group memberships, possibly including `sudo`.
- **Deleting the only administrator account.** Keep at least one working administrator and test `sudo` **before** logging out.
- **Orphaned files.** Files owned by a deleted UID may be inherited by a new account that reuses that UID. Find and reassign them.
- **Locked accounts that still log in.** Key-based SSH ignores the password lock.
- **Changing a user's UID without fixing files.** After `usermod -u`, run `find / -uid OLDUID -exec chown -h NEWUID {} +` on the affected filesystems (with care).

## Further reading

- [`useradd(8)`](https://man7.org/linux/man-pages/man8/useradd.8.html)
- [`usermod(8)`](https://man7.org/linux/man-pages/man8/usermod.8.html)
- [`chage(1)`](https://man7.org/linux/man-pages/man1/chage.1.html)
