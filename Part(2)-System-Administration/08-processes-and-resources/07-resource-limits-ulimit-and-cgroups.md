# Resource Limits: ulimit and cgroups

## Overview

Limits stop one process, user, or service from consuming all system resources. Linux has two main mechanisms:

- **Per-process limits** (`ulimit`, called *rlimits*): limits for a process and its children.
- **cgroups**: limits for a **group** of processes (a service, a container, a user). This is what systemd uses.

## Key concepts

### Soft and hard limits (rlimits)

| Type | Meaning |
|------|---------|
| **Soft limit** | The limit currently enforced. A process or user may raise it up to the hard limit |
| **Hard limit** | The maximum the soft limit may be raised to. Only root can raise it |

### Common limits

| Option | rlimit name | Meaning |
|--------|-------------|---------|
| `-n` | `nofile` | Open file descriptors |
| `-u` | `nproc` | Processes (threads count) per user |
| `-c` | `core` | Core file size |
| `-v` | `as` | Address space (virtual memory) |
| `-s` | `stack` | Stack size |
| `-t` | `cpu` | CPU time in seconds |
| `-l` | `memlock` | Locked memory |
| `-f` | `fsize` | Maximum file size |

## Commands

### ulimit

```console
$ ulimit -a                      # all soft limits
core file size          (blocks, -c) 0
open files                      (-n) 1024
max user processes              (-u) 63000

$ ulimit -n                      # soft limit for open files
1024
$ ulimit -Hn                     # hard limit
524288
$ ulimit -n 4096                 # raise soft limit for this shell and its children
$ ulimit -c unlimited            # allow core dumps in this shell
```

`ulimit` is a shell builtin. It affects only the current shell and its children.

### Inspect a running process

```console
$ cat /proc/2210/limits
Limit                     Soft Limit           Hard Limit           Units
Max open files            1024                 524288               files
Max processes             63000                63000                processes

$ prlimit --pid 2210 --nofile
RESOURCE DESCRIPTION              SOFT   HARD UNITS
NOFILE   max number of open files 1024 524288 files

# prlimit --pid 2210 --nofile=65536:65536         # change a running process (root)
```

### Persistent limits for logins: limits.conf

Applies to sessions that use **PAM** (`pam_limits`), such as SSH and console logins. Edit `/etc/security/limits.conf` or add files in `/etc/security/limits.d/`:

```
# <domain>  <type>  <item>   <value>
alice       soft    nofile   4096
alice       hard    nofile   8192
@developers hard    nproc    2048
*           soft    core     0
```

| Field | Meaning |
|-------|---------|
| domain | User, `@group`, or `*` for all |
| type | `soft`, `hard`, or `-` for both |
| item | `nofile`, `nproc`, `core`, `memlock`, and so on |

> [!IMPORTANT]
> `limits.conf` does **not** apply to **systemd services**. A service does not go through a PAM login. Use `LimitNOFILE=` and similar settings in the unit.

### Limits for systemd services

```ini
[Service]
LimitNOFILE=65536
LimitNPROC=4096
LimitCORE=infinity
```

The system default for services is in `/etc/systemd/system.conf` (`DefaultLimitNOFILE=`).

### cgroup limits with systemd

systemd puts each service in its own cgroup, so you can limit **all processes of the service together**.

| Directive | Meaning |
|-----------|---------|
| `CPUQuota=50%` | At most 50% of **one** CPU (200% = two CPUs) |
| `CPUWeight=100` | Relative CPU share when there is competition (1–10000, default 100) |
| `AllowedCPUs=0-1` | Restrict to CPUs |
| `MemoryHigh=800M` | Soft limit: throttles and reclaims above this |
| `MemoryMax=1G` | **Hard limit**: the OOM killer acts inside the cgroup if exceeded |
| `MemorySwapMax=0` | No swap for this service |
| `TasksMax=512` | Maximum number of processes and threads |
| `IOWeight=100` | Relative disk I/O share |
| `IOReadBandwidthMax=/dev/sda 50M` | Limit disk read speed |

Set permanently with a drop-in:

```console
# systemctl edit myapp
```

```ini
[Service]
MemoryHigh=800M
MemoryMax=1G
CPUQuota=150%
TasksMax=512
```

Set temporarily (lasts until reboot with `--runtime`):

```console
# systemctl set-property --runtime myapp.service MemoryMax=512M
# systemd-run --scope -p MemoryMax=200M -p CPUQuota=50% ./heavy-job     # one-off
```

Check the result:

```console
$ systemctl show myapp -p MemoryMax,CPUQuotaPerSecUSec,TasksMax
$ systemd-cgtop
$ cat /sys/fs/cgroup/system.slice/myapp.service/memory.max
1073741824
$ cat /sys/fs/cgroup/system.slice/myapp.service/memory.events
```

`memory.events` counts how often the limits were hit (`high`, `max`, `oom`, `oom_kill`).

### Limits for users

systemd also has user slices:

```console
# systemctl set-property user-1000.slice MemoryMax=4G CPUQuota=200%
```

Or set defaults in `/etc/systemd/system/user-.slice.d/50-limits.conf`.

### Disk quotas (different topic)

User and group **disk space** limits use filesystem quotas (`quota`, `xfs_quota`). They are separate from `ulimit` and cgroups.

## Common pitfalls

- **Editing `limits.conf` for a service and seeing no change.** Use `LimitNOFILE=` in the unit file and restart.
- **Raising a soft limit above the hard limit** fails: `ulimit: open files: cannot modify limit: Operation not permitted`.
- **`MemoryMax` too low** causes OOM kills inside the service. Start with `MemoryHigh` and watch `memory.events`.
- **`CPUQuota=100%` means one CPU**, not the whole machine.
- **Setting `nproc` too low** prevents a user from starting new processes or threads (`fork: Resource temporarily unavailable`).
- **cgroup v1 vs. v2 paths.** Examples here use v2 (`/sys/fs/cgroup/<unit>/`). Check `stat -fc %T /sys/fs/cgroup`.
- **Limits are lost after restart** when set with `ulimit` or `prlimit`. Make them permanent in unit files or PAM configuration.

## Further reading

- [`getrlimit(2)`](https://man7.org/linux/man-pages/man2/getrlimit.2.html)
- [`limits.conf(5)`](https://man7.org/linux/man-pages/man5/limits.conf.5.html)
- [`systemd.resource-control(5)`](https://man7.org/linux/man-pages/man5/systemd.resource-control.5.html)
- [Kernel cgroup v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
