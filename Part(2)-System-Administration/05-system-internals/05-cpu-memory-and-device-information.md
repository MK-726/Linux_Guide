# CPU, Memory, and Device Information

## Overview

The first step in any performance or capacity question is to know what the system has: how many CPUs, how much memory, and which disks. This page shows the commands and how to read them.

## Commands

### CPU

```console
$ lscpu
Architecture:             x86_64
CPU(s):                   8
  On-line CPU(s) list:    0-7
Model name:               Intel(R) Xeon(R) Silver 4210 CPU @ 2.20GHz
Thread(s) per core:       2
Core(s) per socket:       4
Socket(s):                1
Virtualization:           VT-x
L3 cache:                 13.8 MiB

$ nproc
8
```

| Item | Meaning |
|------|---------|
| Socket | A physical CPU package |
| Core | An independent processing unit in the socket |
| Thread | Hardware threads per core (Hyper-Threading / SMT). One core with 2 threads appears as 2 logical CPUs |
| CPU(s) | Number of **logical** CPUs (sockets × cores × threads) |

CPU feature flags are in `/proc/cpuinfo`:

```console
$ grep -m1 flags /proc/cpuinfo | tr ' ' '\n' | grep -E '^(vmx|svm|aes|avx2)$'
vmx
aes
avx2
```

`vmx` (Intel) or `svm` (AMD) means hardware virtualization is available (needed for KVM).

### Memory

```console
$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.1Gi       1.2Gi       312Mi        10Gi        10Gi
Swap:          2.0Gi          0B       2.0Gi
```

| Column | Meaning |
|--------|---------|
| **total** | Installed memory the kernel can use |
| **used** | Memory used by programs |
| **free** | Completely unused memory |
| **shared** | Memory used by `tmpfs` and shared segments |
| **buff/cache** | Memory used for file caching (the kernel gives this back when programs need it) |
| **available** | **Best estimate of memory available to start new programs without swapping** |

> [!IMPORTANT]
> Low **free** memory is normal and healthy. Linux uses spare memory as a disk cache. Look at **available**, not free.

```console
$ grep -E 'MemTotal|MemAvailable|SwapTotal|SwapFree|Dirty' /proc/meminfo
MemTotal:       16384000 kB
MemAvailable:   10485760 kB
SwapTotal:       2097152 kB
SwapFree:        2097152 kB
Dirty:                 48 kB

$ vmstat 1 3          # memory, swap, I/O, CPU every second
$ lsmem               # memory blocks
# dmidecode -t memory | grep -E 'Size|Speed|Type:'    # physical DIMMs (root)
```

### Storage and other devices

```console
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   50G  0 disk
├─sda1   8:1    0  512M  0 part /boot/efi
└─sda2   8:2    0 49.5G  0 part /
sdb      8:16   0  200G  0 disk

$ df -hT              # filesystem usage and types
$ lsblk -f            # filesystems, labels, and UUIDs
$ ip -br link         # network interfaces
```

### System summary

```console
$ hostnamectl
 Static hostname: server01
       Chassis: server
 Operating System: Ubuntu 24.04 LTS
          Kernel: Linux 6.8.0-45-generic
    Architecture: x86-64

$ uptime
 11:30:02 up 12 days,  3:11,  2 users,  load average: 0.15, 0.20, 0.18
```

**Load average** shows the average number of runnable (and uninterruptible) tasks over the last 1, 5, and 15 minutes. Compare it with the number of CPUs: on an 8-CPU system, a load of 8 means the CPUs are fully busy. (See Chapter 25 for details.)

## Quick inventory script

```bash
#!/bin/bash
echo "Host:   $(hostname)"
echo "OS:     $(. /etc/os-release && echo "$PRETTY_NAME")"
echo "Kernel: $(uname -r)"
echo "CPUs:   $(nproc)"
echo "RAM:    $(free -h | awk '/^Mem:/ {print $2}')"
lsblk -d -o NAME,SIZE,TYPE,ROTA | grep disk
```

## Common pitfalls

- **Reading "free" as the amount of memory left.** Use **available**.
- **Counting hyper-threads as full cores** for licensing or capacity. Check `lscpu`.
- **Container and VM limits.** Inside a container, `free` and `nproc` may show the **host** values (not the cgroup limits). Check `/sys/fs/cgroup/`.
- **Virtual machines** may report CPU model but share physical CPUs. Steal time (`st` in `top`) shows contention.
- **`lshw` and `dmidecode` need root** for full information.

## Further reading

- [`lscpu(1)`](https://man7.org/linux/man-pages/man1/lscpu.1.html)
- [`free(1)`](https://man7.org/linux/man-pages/man1/free.1.html)
- [`proc_meminfo(5)`](https://man7.org/linux/man-pages/man5/proc_meminfo.5.html)
