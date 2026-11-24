# Memory and CPU Utilization

## Overview

To find out whether a system is healthy, you must read CPU and memory numbers correctly. Linux uses memory aggressively for caching, and "high usage" is not always a problem. This page explains what the numbers mean and what to do when memory runs out.

## Key concepts

### CPU time breakdown

The `%Cpu(s)` line in `top` (and the columns in `vmstat` and `mpstat`):

| Field | Meaning | A problem when... |
|-------|---------|-------------------|
| `us` | User-space programs | High: busy applications |
| `sy` | Kernel (system calls, drivers) | High: heavy I/O, network, or context switching |
| `ni` | User programs with positive nice | Normal for background jobs |
| `id` | Idle | Low: CPU is saturated |
| `wa` | **Waiting for I/O** | High: slow disk or network storage |
| `hi` / `si` | Hardware / software interrupts | High: heavy network traffic |
| `st` | **Steal**: time taken by the hypervisor | High in a VM: noisy neighbors, overcommitted host |

**Load average** (from `uptime`) counts processes that are running, waiting for CPU, or in uninterruptible I/O wait. Compare it to the number of CPUs (`nproc`). On 4 CPUs, a load of 4 means full use, and a sustained load of 12 means a long queue.

### Memory types

| Term | Meaning |
|------|---------|
| **Virtual (VIRT, VSZ)** | Address space a process has reserved. Mostly not in use |
| **Resident (RES, RSS)** | Memory the process has in RAM now |
| **Shared (SHR)** | Part of RES shared with other processes (libraries) |
| **Page cache** | File data cached in RAM. Reclaimed when programs need memory |
| **Buffers** | Cache of block-device metadata |
| **Slab** | Kernel object caches |
| **Anonymous memory** | Program data (heap, stack). Can only go to swap |
| **Swap** | Disk space for anonymous memory pushed out of RAM |
| **Available** | Estimate of memory usable without swapping |

RSS counts shared memory in each process, so adding RSS of all processes overcounts. For a fairer figure use PSS (`smem`, or `/proc/PID/smaps_rollup`).

```console
$ grep -E 'Rss|Pss' /proc/2210/smaps_rollup
Rss:             812344 kB
Pss:             790120 kB
```

### What "memory pressure" looks like

| Sign | Command |
|------|---------|
| `available` is small and swap is heavily used | `free -h` |
| Constant swapping (`si`/`so` columns above 0) | `vmstat 1` |
| High `wa` and slow response | `top`, `iostat -x 1` |
| Processes killed by the OOM killer | `dmesg -T \| grep -i -E 'out of memory\|oom'` |
| Pressure stall information (kernel 4.20+) | `cat /proc/pressure/memory` |

```console
$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.12 avg300=0.05 total=2400123
full avg10=0.00 avg60=0.00 avg300=0.00 total=100231
```

`some` means at least one task was stalled for memory. `full` means all tasks were. Pressure above a few percent for long periods signals a problem. (Chapter 25 covers pressure files in detail.)

## Commands

| Command | Purpose |
|---------|---------|
| `top`, `htop` | Live CPU and memory by process |
| `free -h` | System memory summary |
| `vmstat 1 5` | CPU, memory, swap, I/O every second, 5 times |
| `mpstat -P ALL 1` | Per-CPU usage (`sysstat` package) |
| `pidstat -u -r 1` | Per-process CPU and memory over time (`sysstat`) |
| `ps -eo pid,%cpu,%mem,rss,cmd --sort=-rss \| head` | Biggest memory users |
| `smem -tk` | Proportional memory per process (`smem` package) |
| `cat /proc/meminfo` | Full memory statistics |
| `swapon --show` | Swap devices and use |
| `sar -r 1 5` | Historical memory (`sysstat`) |

```console
$ vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 1250400 210000 9800000    0    0     5    12  210  480  3  1 95  1  0
```

| Column | Meaning |
|--------|---------|
| `r` | Runnable processes (compare with CPU count) |
| `b` | Processes blocked on I/O |
| `si`, `so` | Memory swapped in / out per second. **Sustained non-zero `so` is a warning** |
| `bi`, `bo` | Blocks read / written |
| `cs` | Context switches per second |

### The OOM killer

When memory is exhausted and nothing can be reclaimed, the kernel's **Out-Of-Memory (OOM) killer** ends a process to save the system.

```console
$ dmesg -T | grep -i -E 'killed process|out of memory'
[Mon Mar  3 10:15:22 2025] Out of memory: Killed process 2210 (java) total-vm:8123456kB, anon-rss:7012344kB
```

| File | Meaning |
|------|---------|
| `/proc/PID/oom_score` | How likely the process is to be chosen (higher = more likely) |
| `/proc/PID/oom_score_adj` | Adjustment from `-1000` (never kill) to `1000` (kill first) |

Protect an important service in its unit file:

```ini
[Service]
OOMScoreAdjust=-900
```

A process in a container or cgroup with a memory limit can be killed when **its limit** is reached, even if the host has free memory. Look for "memory cgroup out of memory" in `dmesg`.

### Overcommit and swappiness

| Setting | Meaning |
|---------|---------|
| `vm.overcommit_memory` | `0` heuristic (default), `1` always allow, `2` strict limits |
| `vm.swappiness` | Preference for swapping (default 60; lower = prefer to keep applications in RAM) |

## Common pitfalls

- **Panicking about low "free" memory.** Look at `available`.
- **Swap in use but no activity.** Some swap use is normal. Active swapping (`si`/`so` constantly above 0) is the problem.
- **Adding up RSS** to find total use. Shared pages are counted several times. Use PSS or `free`.
- **Blaming the application only.** `st` (steal) in VMs and `wa` (I/O wait) often point to the platform or storage.
- **Disabling swap everywhere.** Without swap, the OOM killer acts sooner. Kubernetes has special requirements, but databases and general servers usually benefit from some swap.
- **Ignoring memory limits in containers.** The kernel kills processes at the limit. Set sensible limits and monitor.
- **Memory leaks.** Memory grows steadily over days until a restart. Track RSS over time (`pidstat -r`, monitoring) before you restart blindly.

## Further reading

- [`proc_meminfo(5)`](https://man7.org/linux/man-pages/man5/proc_meminfo.5.html)
- [`vmstat(8)`](https://man7.org/linux/man-pages/man8/vmstat.8.html)
- [Linux kernel: PSI](https://www.kernel.org/doc/html/latest/accounting/psi.html)
- [Kernel: OOM](https://www.kernel.org/doc/gorman/html/understand/understand016.html)
- [sysstat](https://github.com/sysstat/sysstat)
