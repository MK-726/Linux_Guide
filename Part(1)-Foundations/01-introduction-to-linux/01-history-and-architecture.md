# History and Architecture

## Overview

Linux is a free, open-source, Unix-like operating system. It powers most web servers, cloud platforms, containers, supercomputers, and Android phones. To manage Linux well, you need to know where it came from and how its parts fit together.

## Key concepts

### A short history

| Year | Event |
|------|-------|
| 1969–1970s | Unix is created at Bell Labs. It sets the design ideas Linux follows later. |
| 1983 | Richard Stallman starts the **GNU Project** to build a free Unix-like system. |
| 1991 | Linus Torvalds announces and releases the first version of the **Linux kernel**. |
| 1993 | Debian is started. Slackware is also released. |
| 1990s | Red Hat, SUSE, and others package Linux for business use. |
| 2000s | Linux becomes the main server operating system. Ubuntu (2004) brings Linux to many desktop users. |
| 2000s–today | Android (which uses the Linux kernel), cloud platforms, and containers make Linux the base of modern infrastructure. |

The GNU Project had most of the system (compiler, shell, core tools) but lacked a working kernel. Linux supplied the kernel. Together they formed a complete free operating system.

### Unix design ideas that Linux keeps

- **Everything is a file.** Devices, sockets, and process information are exposed through file-like interfaces.
- **Small tools that do one job.** You combine them with pipes.
- **Plain text for configuration.** Most settings are readable text files.
- **Multi-user and multi-tasking.** Many users and programs share one system safely.

### The layers of a Linux system

```
┌─────────────────────────────────────────────┐
│  Applications (web server, database, editor)│
├─────────────────────────────────────────────┤
│  Shell and system utilities (bash, ls, grep)│
├─────────────────────────────────────────────┤
│  System libraries (glibc, musl)             │
├══════════════ system call interface ════════┤
│  Linux kernel                               │
│  (processes, memory, filesystems, drivers,  │
│   networking, security)                     │
├─────────────────────────────────────────────┤
│  Hardware (CPU, RAM, disks, network cards)  │
└─────────────────────────────────────────────┘
```

- **User space** is where applications run. They have limited privileges.
- **Kernel space** is where the kernel runs. It has full access to hardware.
- Applications ask the kernel to do work through **system calls**. Libraries such as **glibc** make this easier for programmers.

### What the kernel does

| Job | Description |
|-----|-------------|
| Process management | Starts, schedules, and stops programs |
| Memory management | Gives each process its own memory and manages RAM and swap |
| Filesystems | Reads and writes data on disks through a common interface |
| Device drivers | Talks to hardware |
| Networking | Implements TCP/IP and other protocols |
| Security | Enforces permissions and isolation |

## Commands

Check what kernel is running and what system you are on:

```console
$ uname -r
6.8.0-45-generic

$ uname -a
Linux server01 6.8.0-45-generic #45-Ubuntu SMP x86_64 GNU/Linux

$ cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
ID=ubuntu
ID_LIKE=debian
```

| Command | Purpose |
|---------|---------|
| `uname -r` | Show the kernel version |
| `uname -m` | Show the CPU architecture (for example `x86_64`, `aarch64`) |
| `cat /etc/os-release` | Show the distribution name and version |
| `hostnamectl` | Show host, OS, and kernel information (systemd systems) |

## Common pitfalls

- **Saying "Linux" when you mean the distribution.** "I run Linux" does not say which distribution. When you ask for help, give the distribution and version (`cat /etc/os-release`).
- **Confusing kernel version with distribution version.** Ubuntu 24.04 and kernel 6.8 are different numbers for different things.

## Further reading

- [The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/)
- [GNU Project](https://www.gnu.org/gnu/gnu-history.html)
- [os-release manual](https://man7.org/linux/man-pages/man5/os-release.5.html)
