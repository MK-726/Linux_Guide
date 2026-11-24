# systemd Architecture, Targets, and Unit Types

## Overview

**systemd** is the init system and service manager on nearly all major distributions. It is the first process started by the kernel (**PID 1**). It starts and supervises services, mounts filesystems, manages logging, timers, and much more.

## Key concepts

### What systemd manages: units

Everything systemd manages is a **unit**, described by a text file.

| Unit type | Extension | Purpose |
|-----------|-----------|---------|
| Service | `.service` | A daemon or one-time task |
| Socket | `.socket` | Starts a service when a connection arrives |
| Target | `.target` | A group of units, a "state" of the system |
| Timer | `.timer` | Runs a service on a schedule |
| Mount | `.mount` | A mounted filesystem |
| Automount | `.automount` | Mounts on first access |
| Path | `.path` | Starts a service when a file or directory changes |
| Swap | `.swap` | A swap area |
| Device | `.device` | A device known to udev |
| Slice | `.slice` | A cgroup group for resource control |
| Scope | `.scope` | Externally created processes (for example, a login session) |

### Targets and runlevels

A **target** is a synchronization point: a set of units that make up a system state. Targets replace the old SysV **runlevels**.

| Target | Old runlevel | Meaning |
|--------|--------------|---------|
| `poweroff.target` | 0 | Shut down |
| `rescue.target` | 1 / S | Single-user mode, basic services, root shell |
| `multi-user.target` | 2, 3, 4 | Multi-user, **no graphical interface** (typical server) |
| `graphical.target` | 5 | Multi-user with graphical login |
| `reboot.target` | 6 | Reboot |
| `emergency.target` | — | Minimal shell, root filesystem read-only, almost nothing started |
| `default.target` | — | Link to the target booted by default |

```console
$ systemctl get-default
multi-user.target
# systemctl set-default multi-user.target        # change the default (root)
# systemctl isolate rescue.target                # switch now (stops other services!)
```

### Dependencies and ordering

Two separate ideas, which are often confused:

| Kind | Directive | Meaning |
|------|-----------|---------|
| **Requirement** (what must be started) | `Wants=` | Start the other unit too, but continue if it fails (**preferred**) |
| | `Requires=` | Start the other unit too; if it fails, this one fails |
| | `BindsTo=` | Like `Requires`, and stops when the other stops |
| | `Conflicts=` | Cannot run together; starting one stops the other |
| **Ordering** (when) | `After=` | Start this unit **after** the other |
| | `Before=` | Start this unit **before** the other |

`Requires=` does not set an order. To start B after A, you normally need both `Wants=A` (or `Requires=A`) **and** `After=A`.

Units start in **parallel** unless ordering rules say otherwise. That is why boot is fast.

### How a unit becomes active at boot: enable

`systemctl enable` creates a symbolic link from the unit's `[Install]` target into `/etc/systemd/system/<target>.wants/`. The target pulls the unit in.

```console
# systemctl enable nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.
```

### Where unit files are

Searched in this order. The first match wins:

| Directory | Purpose |
|-----------|---------|
| `/etc/systemd/system/` | **Administrator** units and overrides (highest priority) |
| `/run/systemd/system/` | Runtime units |
| `/usr/lib/systemd/system/` (also `/lib/systemd/system/` on Debian-based) | Units installed by packages |

Never edit files in `/usr/lib/systemd/system/`. Use overrides (see [Unit Files, Overrides, and Drop-ins](05-unit-files-overrides-dropins.md)).

## Commands

```console
$ systemctl list-units --type=target
$ systemctl list-dependencies multi-user.target | head
$ systemctl list-dependencies --reverse nginx.service     # what depends on it
$ systemd-analyze critical-chain
$ pstree -p 1 | head -5
systemd(1)─┬─ModemManager(701)
           ├─cron(654)
           ├─sshd(812)───sshd(2100)───sshd(2160)───bash(2161)
```

| Tool | Purpose |
|------|---------|
| `systemctl` | Control units |
| `journalctl` | Read logs |
| `systemd-analyze` | Analyze boot and verify units |
| `loginctl` | Sessions and users |
| `hostnamectl`, `timedatectl`, `localectl` | System settings |
| `systemd-cgls`, `systemd-cgtop` | Control groups |
| `busctl` | D-Bus inspection |

## Common pitfalls

- **Assuming `After=` pulls in a unit.** It only orders. Add `Wants=` or `Requires=`.
- **Using `Requires=` for everything.** One failure can stop many services. Prefer `Wants=`.
- **Switching targets with `isolate` on a remote system** can cut off your SSH session.
- **Editing vendor unit files** under `/usr/lib/systemd/system/` and losing changes after upgrades.
- **Thinking systemd is "just init".** It also provides login sessions, timers, logging, DNS resolver, and more. Many of these are optional.

## Further reading

- [`systemd(1)`](https://man7.org/linux/man-pages/man1/systemd.1.html)
- [`systemd.unit(5)`](https://man7.org/linux/man-pages/man5/systemd.unit.5.html)
- [`systemd.special(7)`](https://man7.org/linux/man-pages/man7/systemd.special.7.html)
- [systemd project documentation](https://systemd.io)
