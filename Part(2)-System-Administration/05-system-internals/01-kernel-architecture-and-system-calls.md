# Kernel Architecture and System Calls

## Overview

The kernel is the core of the operating system. It controls the hardware and decides which program gets the CPU, memory, disk, and network. Programs never touch hardware directly. They ask the kernel through **system calls**.

## Key concepts

### Kernel design

Linux is a **monolithic kernel with loadable modules**: the main parts run together in one privileged address space, and extra drivers can be loaded and unloaded while the system runs.

| Subsystem | Job |
|-----------|-----|
| Process scheduler | Decides which process runs on which CPU, and for how long |
| Memory manager | Virtual memory, paging, page cache, swap |
| Virtual File System (VFS) | One common interface above ext4, XFS, NFS, and others |
| Network stack | TCP/IP, sockets, firewall hooks (netfilter) |
| Device drivers | Talk to disks, network cards, GPUs, and so on |
| Security modules | SELinux, AppArmor, capabilities |
| IPC | Pipes, sockets, shared memory, signals |

### User space and kernel space

| | User space | Kernel space |
|---|-----------|--------------|
| Runs | Applications, shells, services | The kernel and drivers |
| Privilege | Limited. Cannot access hardware or other processes' memory directly | Full |
| If it crashes | Only that process dies | Can crash the whole system ("kernel panic") |

The CPU enforces this split with privilege levels. A program in user space cannot simply read another program's memory or write to a disk controller.

### System calls

A **system call** (syscall) is the controlled doorway from user space into the kernel. Common ones:

| Syscall | Purpose |
|---------|---------|
| `openat`, `read`, `write`, `close` | Work with files |
| `fork`, `clone`, `execve` | Create and start processes |
| `mmap`, `brk` | Get memory |
| `socket`, `connect`, `accept` | Networking |
| `kill` | Send a signal |
| `ioctl` | Device-specific control |

```
Application → library function (glibc: fopen) → syscall (openat) → kernel → hardware
```

The C library (glibc, or musl on Alpine) wraps syscalls into friendly functions. Each syscall switches the CPU from user mode to kernel mode and back.

## Commands

### strace: watch the syscalls of a program

`strace` shows what a program asks the kernel to do. It is one of the best troubleshooting tools: it reveals which files a program opens, which fail, and where it hangs.

```console
$ strace -e trace=openat cat /etc/hostname
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
server01
+++ exited with 0 +++
```

| Command | Purpose |
|---------|---------|
| `strace COMMAND` | Trace a new command |
| `strace -p PID` | Attach to a running process (**needs root** for other users' processes) |
| `strace -f` | Follow child processes |
| `strace -e trace=file` | Only file-related syscalls |
| `strace -e trace=network` | Only network syscalls |
| `strace -c COMMAND` | Summary table of syscalls and time |
| `strace -tt -T` | Timestamps and time spent in each call |
| `strace -o out.txt` | Write the trace to a file |

Find why a program cannot read its configuration:

```console
$ strace -f -e trace=openat myapp 2>&1 | grep ENOENT
openat(AT_FDCWD, "/etc/myapp/app.conf", O_RDONLY) = -1 ENOENT (No such file or directory)
```

Common error names in traces: `ENOENT` (no such file), `EACCES` (permission denied), `EADDRINUSE` (port in use), `ECONNREFUSED` (nothing listening).

> [!NOTE]
> `strace` slows the program noticeably. Use it with care on busy production processes.

### Other useful commands

| Command | Purpose |
|---------|---------|
| `uname -r` | Kernel version |
| `ltrace COMMAND` | Trace library calls (if installed) |
| `man 2 syscalls` | List of Linux syscalls |
| `man 2 open` | Documentation of one syscall |

## Common pitfalls

- **Blaming "Linux" for a user-space crash.** A segmentation fault in one application is not a kernel failure. Check `dmesg` and the logs.
- **Running strace on a production service without a plan.** It can add large delays.
- **Confusing kernel version and distribution version.** Distributions backport fixes, so a "5.14" kernel on RHEL contains much newer code.

## Further reading

- [Kernel documentation](https://www.kernel.org/doc/html/latest/)
- [`syscalls(2)`](https://man7.org/linux/man-pages/man2/syscalls.2.html)
- [`strace(1)`](https://man7.org/linux/man-pages/man1/strace.1.html)
