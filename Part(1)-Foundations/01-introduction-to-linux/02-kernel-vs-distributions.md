# Kernel vs. GNU/Linux Distributions

## Overview

Strictly, **Linux** is only the **kernel**. A complete operating system needs much more. A **distribution** (distro) bundles the kernel with other software into something you can install and use.

## Key concepts

### The kernel

The Linux kernel is one program that manages hardware and gives other programs a safe way to use it. Linus Torvalds and thousands of contributors develop it. It is distributed at [kernel.org](https://www.kernel.org).

### GNU/Linux

Most of the tools you use daily (the shell, `ls`, `cp`, `grep`, the C library, the compiler) come from the **GNU Project**, not from the kernel. Some people say **GNU/Linux** to credit both parts. In everyday speech, "Linux" usually means the whole system. This guide uses "Linux" for the whole system and says "the kernel" when it means only the kernel.

### What a distribution adds

| Component | Examples |
|-----------|----------|
| Kernel | A tested Linux kernel build with patches |
| Core tools | GNU coreutils, Bash, util-linux |
| Init system | systemd (most distributions) |
| Package manager | APT, DNF, Zypper |
| Software repositories | Servers hosting thousands of packages |
| Installer | The program that sets up the system |
| Defaults and policies | Security settings, default filesystem, firewall tools |
| Updates and support | Security patches and a support schedule |
| Documentation and community | Wikis, forums, vendor support |

### Why many distributions exist

Different groups want different things: stability, newest software, small size, security, commercial support, or ease of use. Because the software is open source, anyone can build a distribution.

## Commands

Identify the kernel and distribution separately:

```console
$ uname -sr
Linux 6.8.0-45-generic

$ grep -E '^(NAME|VERSION)=' /etc/os-release
NAME="Rocky Linux"
VERSION="9.4 (Blue Onyx)"
```

List installed kernels (Debian-based example):

```console
$ dpkg --list 'linux-image-*' | grep '^ii'
```

List installed kernels (RHEL-based example):

```console
$ rpm -q kernel
kernel-5.14.0-427.13.1.el9_4.x86_64
```

## Common pitfalls

- **Assuming the kernel is the same everywhere.** Distributions patch their kernels and may keep an older version number while backporting fixes. Check the distribution's security tracker, not only the version number.
- **Mixing up "Linux" and "Unix".** Linux is *Unix-like*. It is not derived from Unix source code.

## Further reading

- [kernel.org](https://www.kernel.org)
- [DistroWatch](https://distrowatch.com) (list and comparison of distributions)
