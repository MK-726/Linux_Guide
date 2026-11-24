# Process Priorities: nice and renice

## Overview

When more processes want the CPU than there are CPUs, the kernel scheduler decides who runs. You can influence this with the **nice value**. A "nicer" process gives way to others. Priority settings are useful for background work (backups, compression) that must not slow down important services.

## Key concepts

### Nice values

| Nice value | Meaning |
|-----------|---------|
| `-20` | Highest priority ("least nice") |
| `0` | Default |
| `19` | Lowest priority ("very nice") |

- Any user can **raise** the nice value (lower priority) of their own processes.
- Only **root** can **lower** the nice value (raise priority) or set negative values.
- Nice affects **CPU time share** under competition. It does not limit usage when the CPU is idle.
- Child processes inherit the nice value.

In `top`: the `NI` column is the nice value and `PR` is the kernel priority (for normal processes, `PR = 20 + NI`).

### Other priority controls

| Mechanism | Controls | Tool |
|-----------|----------|------|
| **nice value** | CPU share for normal (`SCHED_OTHER`) processes | `nice`, `renice` |
| **I/O priority** | Disk access order (depends on the I/O scheduler, for example BFQ) | `ionice` |
| **Real-time policy** | Strict priority (`SCHED_FIFO`, `SCHED_RR`) | `chrt` |
| **CPU affinity** | Which CPUs a process may use | `taskset` |
| **cgroup weights and limits** | CPU and I/O shares per service | systemd (`CPUWeight=`, `CPUQuota=`) |

## Commands

### Start with a different priority

```console
$ nice -n 10 tar -czf backup.tar.gz /data           # lower priority
# nice -n -5 /usr/local/bin/critical-job            # higher priority (root)
```

### Change a running process

```console
$ renice -n 15 -p 4821               # one process
$ renice -n 10 -u alice              # all processes of a user (affects only future-visible ones now)
# renice -n -5 -p 4821               # raise priority (root)
4821 (process ID) old priority 0, new priority -5
```

`top` also lets you press `r`, enter the PID and the new nice value.

### I/O priority

```console
$ ionice -c 3 tar -czf backup.tar.gz /data     # class 3 = idle: only when no one else needs the disk
$ ionice -c 2 -n 7 command                      # best-effort, lowest level (0 = highest, 7 = lowest)
$ ionice -p 4821                                # show
# ionice -c 3 -p 4821                           # change a running process
```

| Class | Name | Meaning |
|-------|------|---------|
| 1 | Realtime | First access to disk (root only) |
| 2 | Best-effort | Default; levels 0–7 |
| 3 | Idle | Only when the disk is otherwise idle |

> [!NOTE]
> `ionice` only works with I/O schedulers that support priorities (for example **BFQ**). Many modern systems use `none` or `mq-deadline` on SSDs and NVMe, where `ionice` has little effect. For reliable I/O control, use cgroup `IOWeight=` or `IOReadBandwidthMax=` in systemd.

### Combined, typical for backups

```console
$ nice -n 19 ionice -c 3 rsync -a /data/ /backup/data/
```

### Real-time scheduling (rare)

```console
# chrt -p 4821                          # show the policy and priority
# chrt -f -p 50 4821                    # SCHED_FIFO priority 50
```

> [!WARNING]
> A real-time process that never yields can **freeze the system**, because normal processes cannot preempt it. Use only for special cases (audio, industrial control) and with care.

### CPU affinity

```console
$ taskset -p 4821                       # show mask
$ taskset -c 0,1 ./compute-job          # run on CPUs 0 and 1
$ taskset -cp 2-3 4821                  # move a running process to CPUs 2 and 3
```

### systemd services

```ini
[Service]
Nice=10
IOSchedulingClass=idle
CPUWeight=50
CPUSchedulingPolicy=other
```

Set it temporarily on a running service:

```console
# systemctl set-property backup.service CPUWeight=20
```

## Common pitfalls

- **Expecting `nice` to cap usage.** A nice process still uses all idle CPU. To **limit** use, set `CPUQuota=` (cgroup).
- **Negative nice values for convenience.** It can starve other work. Use for specific, tested cases.
- **`ionice` seems to do nothing.** Check the I/O scheduler: `cat /sys/block/sda/queue/scheduler`.
- **Priority inversion.** A low-priority process holding a lock that a high-priority one needs can slow everything.
- **`renice -u` affects only existing processes**, not those started later.
- **Real-time processes** can lock up a machine. Avoid unless required.

## Further reading

- [`nice(1)`](https://man7.org/linux/man-pages/man1/nice.1.html)
- [`renice(1)`](https://man7.org/linux/man-pages/man1/renice.1.html)
- [`ionice(1)`](https://man7.org/linux/man-pages/man1/ionice.1.html)
- [`sched(7)`](https://man7.org/linux/man-pages/man7/sched.7.html)
