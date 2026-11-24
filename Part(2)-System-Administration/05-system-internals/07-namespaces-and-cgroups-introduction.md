# Namespaces and cgroups (Introduction)

## Overview

**Namespaces** and **cgroups** are two kernel features that make containers possible.

- **Namespaces** control **what a process can see** (its own process list, network, mounts, hostname).
- **cgroups** (control groups) control **how much a group of processes can use** (CPU, memory, I/O, number of processes).

A container is an ordinary set of processes placed in its own namespaces and limited by cgroups. There is no separate kernel. This page is an introduction. Chapter 8 covers cgroups for resource control and Chapter 22 covers containers.

## Key concepts

### Namespace types

| Namespace | Isolates | Effect |
|-----------|----------|--------|
| `mnt` | Mount points | Process sees its own filesystem tree |
| `pid` | Process IDs | Process sees itself as PID 1 |
| `net` | Network stack | Own interfaces, routes, firewall rules, ports |
| `uts` | Hostname and domain name | Own hostname |
| `ipc` | Inter-process communication | Own shared memory and queues |
| `user` | User and group IDs | Process can be root inside, unprivileged outside |
| `cgroup` | cgroup view | Own view of the cgroup tree |
| `time` | Clocks (boot and monotonic) | Own offsets |

Every process belongs to one namespace of each type.

```console
$ ls -l /proc/$$/ns
lrwxrwxrwx 1 alice alice 0 Mar  3 12:10 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 alice alice 0 Mar  3 12:10 net -> 'net:[4026531840]'
lrwxrwxrwx 1 alice alice 0 Mar  3 12:10 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 alice alice 0 Mar  3 12:10 uts -> 'uts:[4026531838]'
```

Two processes with the same number are in the same namespace.

### cgroups

cgroups organize processes in a **tree** and apply limits and accounting. Modern distributions use **cgroup v2** (a single unified tree). systemd creates a cgroup for every service.

| Controller | Controls |
|------------|----------|
| `cpu` | CPU time share and limits |
| `memory` | Memory limit and accounting (kills a process when exceeded: OOM) |
| `io` | Disk bandwidth and IOPS |
| `pids` | Maximum number of processes |
| `cpuset` | Which CPUs and memory nodes are used |

## Commands

### Namespaces

| Command | Purpose |
|---------|---------|
| `lsns` | List namespaces |
| `unshare` | Run a program in new namespaces |
| `nsenter` | Enter the namespaces of another process (**root**) |
| `ip netns` | Manage named network namespaces |

```console
$ lsns -t net
        NS TYPE NPROCS PID USER COMMAND
4026531840 net     182   1 root /sbin/init

# unshare --uts hostname demo-host        # change hostname in a private namespace only
# unshare --pid --fork --mount-proc bash  # new PID namespace
root@server01:/# ps
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
    8 pts/0    00:00:00 ps
```

Inside the new PID namespace the shell is PID 1 and sees only its own processes. Leaving the shell returns to the host.

```console
# nsenter -t 4821 -n ip addr        # run "ip addr" in the network namespace of PID 4821
```

`nsenter -t PID -n` is very useful for debugging containers from the host.

### cgroups

```console
$ stat -fc %T /sys/fs/cgroup           # filesystem type: cgroup2fs = v2
cgroup2fs

$ cat /proc/self/cgroup
0::/user.slice/user-1000.slice/session-3.scope

$ systemd-cgls | head                  # tree view
$ systemd-cgtop                        # live usage by cgroup
$ cat /sys/fs/cgroup/system.slice/nginx.service/memory.current
```

Set a limit through systemd (**root**):

```console
# systemd-run --scope -p MemoryMax=200M -p CPUQuota=50% stress --vm 1 --vm-bytes 500M
# systemctl set-property nginx.service MemoryMax=1G
```

## Common pitfalls

- **Thinking a container is a VM.** It shares the host kernel. A kernel problem affects all containers.
- **Memory limit kills.** When a cgroup reaches `memory.max`, the OOM killer ends a process inside it. Check `dmesg` and `journalctl -k`.
- **cgroup v1 versus v2.** Older documents describe v1. Check which version your system uses before you copy examples.
- **Tools inside containers show host values** (`free`, `nproc`) because they read `/proc` data that is not namespaced by cgroup.
- **User namespaces.** "root" in a container is not necessarily root on the host when user namespaces (rootless) are in use.

## Further reading

- [`namespaces(7)`](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [`cgroups(7)`](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- [Control Group v2 (kernel docs)](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
