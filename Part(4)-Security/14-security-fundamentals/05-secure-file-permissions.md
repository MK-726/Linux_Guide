# Secure File Permissions

## Overview

Wrong permissions are among the most common and most easily fixed security problems: world-readable secrets, world-writable scripts that run as root, and forgotten SUID programs. This page shows what is expected for important files and **how to audit** the whole system. The mechanics of `chmod`, `chown`, ACLs, SUID, and `umask` are in Chapter 4. For the concepts see [File Ownership and Permissions](../../part-1-foundations/04-users-groups-and-permissions/04-ownership-and-permissions.md) and [Special Permissions and umask](../../part-1-foundations/04-users-groups-and-permissions/05-special-permissions-and-umask.md).

## Key concepts

### Rules of thumb

1. **No world-writable files** (and no world-writable directories without the sticky bit).
2. **Secrets are readable only by the owner and the service group** (`600` or `640`).
3. **Anything run by root must not be writable by anyone else**, including the directories above it.
4. **Minimize SUID/SGID** programs.
5. **No unowned files.** A file with no valid owner or group can be claimed later by a new account with that UID.
6. **Default `umask` of `027`** (or `077`) for service accounts and administrators.
7. **Check the whole path.** A secure file in a writable directory can be replaced.

### Expected permissions of important files

| Path | Typical mode and owner | Notes |
|------|------------------------|-------|
| `/etc/passwd` | `644 root:root` | World-readable by design |
| `/etc/group` | `644 root:root` | |
| `/etc/shadow` | `640 root:shadow` (Debian-based), `000 root:root` (RHEL-based) | Password hashes. Never world-readable |
| `/etc/gshadow` | `640 root:shadow` / `000` | |
| `/etc/sudoers`, `/etc/sudoers.d/*` | `440 root:root` | Edit with `visudo` |
| `/etc/ssh/sshd_config` | `600` or `644 root:root` | |
| `/etc/ssh/ssh_host_*_key` | `600 root:root` | Host private keys |
| `~/.ssh` | `700` | |
| `~/.ssh/authorized_keys` | `600` (or `644`) | |
| `~/.ssh/id_*` (private keys) | `600` | SSH refuses wider modes |
| `/root` | `700` | |
| `/boot/grub/grub.cfg`, `/boot/grub2/grub.cfg` | `600` (RHEL-based `700` dir) | May contain a GRUB password hash |
| `/etc/crontab`, `/etc/cron.d/*` | `644 root:root` (not writable by others) | |
| `/var/spool/cron/*` | `600` | |
| `/var/log` | `755 root:root`; individual logs `640` | Logs can contain sensitive data |
| `/etc/ssl/private/*`, `*.key` | `600` or `640 root:ssl-cert` | TLS private keys |
| Application config with passwords | `640 root:appgroup` | |
| `/tmp`, `/var/tmp` | `1777` | Sticky bit |
| Home directories | `750` or `700` | Debian's default has been `755` in older releases, so check `DIR_MODE` in `/etc/adduser.conf` and `HOME_MODE` in `/etc/login.defs` |

## Commands

### Audit the system

```console
# find / -xdev -type f -perm -0002 -ls 2>/dev/null              # world-writable files
# find / -xdev -type d -perm -0002 ! -perm -1000 -ls 2>/dev/null   # world-writable directories WITHOUT sticky bit
# find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null          # unowned files
# find / -xdev -type f -perm -4000 -ls 2>/dev/null                # SUID files
# find / -xdev -type f -perm -2000 -ls 2>/dev/null                # SGID files
# getcap -r / 2>/dev/null                                         # file capabilities
# find /home -maxdepth 2 -name '.*' -type f \( -name '*history' -o -name '.netrc' -o -name '.pgpass' \) -ls   # sensitive dotfiles
# find / -xdev -type f \( -name '*.pem' -o -name '*.key' -o -name 'id_rsa' -o -name 'id_ed25519' \) -perm /044 -ls 2>/dev/null   # key files readable by others
# find /etc -xdev -type f -perm /022 -ls                          # config files writable by group or others
```

`-xdev` keeps `find` on one filesystem. Run it for each mounted filesystem, or use `findmnt` to list them. Add the results to a **baseline** file and compare them over time (`diff`).

### Compare with the package database

```console
# rpm -Va 2>/dev/null | grep -E '^.M|^..5'              # RHEL-based / SUSE-based: mode or checksum changed
# dpkg -V 2>/dev/null                                    # Debian-based: verify installed files
# debsums -s ; debsums -c                                # (package debsums)
```

Reset RPM-managed permissions to the package defaults (**review first**):

```console
# rpm --setperms PACKAGE
# rpm --setugids PACKAGE
```

### Check the path, not only the file

```console
$ namei -l /srv/app/config/secret.env
f: /srv/app/config/secret.env
dr-xr-xr-x root root /
drwxr-xr-x root root srv
drwxrwxr-x alice app  app          ← group-writable directory: anyone in "app" can replace files below
drwxr-x--- root app   config
-rw-r----- root app   secret.env
```

A writable directory lets users **rename or delete** the files inside it, even without write permission on the files.

### Fix common problems

```console
# chmod o-w FILE                                   # remove world-write
# chmod -R go-rwx /root/secrets
# chown root:appgroup /etc/app/app.conf && chmod 640 /etc/app/app.conf
# chmod u-s /usr/bin/unneeded-suid-tool            # remove SUID where not needed
# chown -R root:root /opt/myapp/bin                # binaries run by root: owned by root, not writable by service users
# chmod g+s,o+t /srv/shared                        # shared directory: inherit group, restrict deletion
```

### Immutable and append-only attributes (ext4, XFS, Btrfs)

```console
# chattr +i /etc/resolv.conf             # immutable: cannot be changed or deleted, even by root
# chattr +a /var/log/app/audit.log       # append-only: can only grow
$ lsattr /etc/resolv.conf
----i---------e------- /etc/resolv.conf
# chattr -i /etc/resolv.conf             # remove before legitimate changes
```

Use `+i` on small, critical files to stop accidental or malicious changes. Remember that updates and package managers will fail on immutable files.

### Make permissions persistent: systemd-tmpfiles

```
# /etc/tmpfiles.d/myapp.conf
d /var/lib/myapp   0750 myapp myapp -
z /etc/myapp/app.conf 0640 root myapp -
```

```console
# systemd-tmpfiles --create /etc/tmpfiles.d/myapp.conf
```

`d` creates and fixes a directory, and `z` sets the mode and owner of an existing path. Configuration management (Ansible `file` module) achieves the same.

### Safe temporary files in scripts

```bash
tmp=$(mktemp)                 # unique, mode 600
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmp" "$tmpdir"' EXIT
```

Never create predictable names such as `/tmp/myscript.tmp`. They enable symlink attacks.

### Restrict `su` and limit who can use `sudo`

```
# /etc/pam.d/su  (uncomment or add)
auth required pam_wheel.so use_uid group=wheel
```

### Umask for services

```ini
# in the unit file
[Service]
UMask=0027
```

For users: set `umask 027` in `/etc/profile.d/umask.sh` or `UMASK 027` in `/etc/login.defs` (test, because it affects shared directories and software installations).

## Common pitfalls

- **`chmod -R 777`** as a "fix". It exposes data and makes scripts executable by anyone. Use the `namei -l` method to find the real problem.
- **Group-writable directories** above root-run scripts. A user in the group can replace the script and gain root.
- **World-readable configuration files with passwords** (database credentials in `/etc/app.conf`, `.env` files in web roots).
- **Files in web roots writable by the web server user.** An application bug becomes code execution. Use read-only code and writable data directories only where needed.
- **Backups and dumps** with default permissions (database dumps in `/tmp` or `/var/backups` readable by others).
- **Ignoring editors' backup and swap files** (`~`, `.swp`, `.bak`) that keep copies of secrets with default permissions.
- **Leaving SUID bits on interpreters or editors** (`vim`, `python`, `find`, `cp`): they give a root shell.
- **Unowned files after deleting users.** Reassign or remove them (`find / -nouser`).
- **Using `chown -R` on a symlink or on `/`** by mistake. Check the path. Use `-h` for links.
- **Not checking mounted filesystems** (`-xdev` skips them). Audit each mount that holds data.
- **Not handling ACLs.** Files with a `+` in `ls -l` have extra access entries. List them with `getfacl -R -s -p /path`.

## Further reading

- [`chmod(1)`](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [`chattr(1)`](https://man7.org/linux/man-pages/man1/chattr.1.html)
- [`systemd-tmpfiles(8)`](https://man7.org/linux/man-pages/man8/systemd-tmpfiles.8.html)
- [CIS Benchmarks: file permissions sections](https://www.cisecurity.org/cis-benchmarks)
