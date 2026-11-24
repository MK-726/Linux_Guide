# Filesystem Hierarchy Standard (FHS) and Paths

## Overview

Linux uses **one single directory tree** that starts at `/` (called "root"). There are no drive letters. Other disks and devices are attached (mounted) somewhere inside this tree. The **Filesystem Hierarchy Standard (FHS)** describes what each top-level directory is for, so software and administrators know where to find things.

> [!NOTE]
> The **root directory** (`/`) is not the same as the **root user's home directory** (`/root`).

## Key concepts

### Top-level directories

| Directory | Purpose |
|-----------|---------|
| `/` | The top of the tree |
| `/bin` | Essential user commands (on modern systems usually a link to `/usr/bin`) |
| `/sbin` | Essential system administration commands (often a link to `/usr/sbin`) |
| `/boot` | Kernel, initramfs, and bootloader files |
| `/dev` | Device files (disks, terminals, `/dev/null`) |
| `/etc` | System-wide **configuration files** (text) |
| `/home` | Home directories of normal users |
| `/lib`, `/lib64` | Shared libraries (often links to `/usr/lib`) |
| `/media` | Mount points for removable media (automatic) |
| `/mnt` | Temporary manual mount points |
| `/opt` | Optional, add-on software packages |
| `/proc` | Virtual files showing process and kernel information |
| `/root` | Home directory of the root user |
| `/run` | Runtime data since boot (PID files, sockets); cleared at reboot |
| `/srv` | Data served by this system (web or FTP content) |
| `/sys` | Virtual files showing devices and kernel settings |
| `/tmp` | Temporary files; may be cleared at reboot |
| `/usr` | Most installed software, libraries, and documentation |
| `/var` | **Variable** data: logs, caches, spools, databases |

### Important subdirectories

| Path | Purpose |
|------|---------|
| `/usr/bin`, `/usr/sbin` | Programs |
| `/usr/lib` | Libraries |
| `/usr/local` | Software installed manually by the administrator (kept apart from the package manager) |
| `/usr/share` | Architecture-independent data (documentation, man pages) |
| `/var/log` | Log files |
| `/var/lib` | Persistent application data (databases, package state) |
| `/var/cache` | Cache data that can be recreated |
| `/var/spool` | Queued data (print, mail, cron) |
| `/var/tmp` | Temporary files that survive reboot |
| `/etc/systemd/system` | Administrator-created systemd units |

> [!NOTE]
> Modern distributions have **merged `/usr`**: `/bin`, `/sbin`, and `/lib` are symbolic links to the same directories under `/usr`. Check with `ls -ld /bin`.

### Absolute and relative paths

| Type | Starts with | Meaning | Example |
|------|-------------|---------|---------|
| **Absolute** | `/` | Complete path from the root | `/etc/ssh/sshd_config` |
| **Relative** | anything else | Path from the **current directory** | `ssh/sshd_config` (when in `/etc`) |

Special names:

| Name | Meaning |
|------|---------|
| `.` | Current directory |
| `..` | Parent directory |
| `~` | Your home directory |
| `~user` | Another user's home directory |

```console
$ pwd
/var/log
$ ls ../lib             # same as /var/lib
$ cd ~/projects         # /home/alice/projects
```

Use absolute paths in scripts and cron jobs, where the current directory is not predictable.

### Path helper commands

| Command | Purpose |
|---------|---------|
| `realpath PATH` | Show the full, resolved absolute path |
| `basename PATH` | Show only the file name |
| `dirname PATH` | Show only the directory part |
| `readlink -f PATH` | Resolve links to the final target |

```console
$ realpath ../lib
/var/lib
$ basename /etc/ssh/sshd_config
sshd_config
$ dirname /etc/ssh/sshd_config
/etc/ssh
```

## Commands

Explore the tree:

```console
$ ls /
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

$ ls -ld /bin
lrwxrwxrwx 1 root root 7 Mar  3 09:00 /bin -> usr/bin

$ tree -L 1 /etc | head      # needs the 'tree' package
$ df -h                      # mounted filesystems and free space
$ findmnt                    # tree view of mounts
```

## Common pitfalls

- **Confusing `/` and `/root`.** Deleting inside `/root` affects one user's files. Deleting inside `/` affects the whole system.
- **Putting your own software in `/usr/bin`.** Use `/usr/local/bin` or `/opt` so the package manager does not conflict with it.
- **Relying on `/tmp` for important data.** It can be cleared at reboot or by a cleaning job.
- **Filling `/var`.** Logs, caches, and databases live here. When `/var` is full, services fail. Monitor it.

## Further reading

- [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/fhs.shtml)
- [`hier(7)`](https://man7.org/linux/man-pages/man7/hier.7.html)
