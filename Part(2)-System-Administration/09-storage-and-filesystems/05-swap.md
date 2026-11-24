# Swap Management

## Overview

**Swap** is disk space the kernel uses when RAM is full. It moves rarely used memory pages to disk so RAM can serve active data. Swap also gives the kernel more room to cache files. It is slower than RAM, so heavy swapping hurts performance, but a little swap makes the system more stable.

## Key concepts

### Types of swap

| Type | Description |
|------|-------------|
| **Swap partition** | A dedicated partition or LVM volume |
| **Swap file** | A normal file used as swap. Flexible and easy to resize |
| **zram** | Compressed swap in **RAM**. Fast, no disk. Used by default on Fedora and some other distributions |
| **zswap** | A compressed cache in front of disk swap |

### Sizing (guideline)

| RAM | Suggested swap |
|-----|----------------|
| Under 2 GiB | 2 × RAM |
| 2–8 GiB | Equal to RAM |
| 8–64 GiB | About 4–8 GiB (or as vendor guidance requires) |
| Hibernation | At least the size of RAM (plus more) |

Databases and special applications (for example SAP, Oracle, Kubernetes nodes) have their own vendor requirements. Follow them.

### swappiness

`vm.swappiness` (0–100, default 60) sets how strongly the kernel prefers swapping out program memory over dropping file cache. Lower values keep applications in RAM longer. Many database servers use 1–10.

## Commands

### View swap

```console
$ swapon --show
NAME      TYPE      SIZE USED PRIO
/dev/dm-1 partition 8.5G   0B   -2
/swapfile file        2G 128M   -3

$ free -h
$ cat /proc/swaps
$ vmstat 1 5                # si / so columns show swapping activity
```

### Create a swap file (root)

```console
# fallocate -l 4G /swapfile                  # or: dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress
# chmod 600 /swapfile                        # swap must not be readable by others
# mkswap /swapfile
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
# swapon /swapfile
# swapon --show
```

Make it permanent in `/etc/fstab`:

```
/swapfile  none  swap  sw  0  0
```

> [!NOTE]
> On **XFS and ext4**, `fallocate` works. On **Btrfs**, use `btrfs filesystem mkswapfile --size 4g /swapfile` (newer versions) or the documented procedure with `chattr +C`, since swap files cannot be copy-on-write or compressed. Swap files also cannot span multiple devices on Btrfs. Check the current Btrfs documentation.

### Create a swap partition or volume

```console
# mkswap /dev/sdb2                           # partition
# mkswap /dev/vg0/swap                       # LVM volume
# swapon /dev/vg0/swap
```

In `/etc/fstab`, use `UUID=...` (from `blkid`):

```
UUID=0e1d6b5c-1a41-4c8b-9a9a-5a7b1f0c7e11  none  swap  sw  0  0
```

### Disable or remove

```console
# swapoff /swapfile            # moves pages back to RAM (needs enough free RAM)
# rm /swapfile
# swapoff -a                   # all swap (temporary)
```

Remove the line from `/etc/fstab`, or the swap returns after reboot.

### Tune

```console
$ sysctl vm.swappiness
vm.swappiness = 60
# sysctl -w vm.swappiness=10
# echo "vm.swappiness = 10" > /etc/sysctl.d/99-swappiness.conf
```

Swap priority (higher is used first):

```
/dev/nvme0n1p3  none  swap  sw,pri=10  0  0
/swapfile       none  swap  sw,pri=1   0  0
```

### zram

```console
# apt install systemd-zram-generator             # Debian-based (or zram-tools)
# dnf install zram-generator                     # RHEL-based / Fedora (installed by default on Fedora)
```

Example `/etc/systemd/zram-generator.conf`:

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
```

```console
# systemctl daemon-reload
# systemctl start systemd-zram-setup@zram0.service
# zramctl
```

## Common pitfalls

- **Swap file with open permissions.** `swapon` warns "insecure permissions". Use `chmod 600`.
- **`swapoff` on a busy system** with little free RAM can trigger the OOM killer. Check `free -h` first.
- **Using a sparse file** (`truncate`) as a swap file. It does not work. Use `fallocate` or `dd`.
- **Swap on Btrfs, or on a file in an LVM thin volume,** without the special steps fails or causes errors.
- **"Swap is used, so something is wrong."** Some usage is normal. Active swapping (`so` constantly above 0) is the issue. See [Memory and CPU Utilization](../08-processes-and-resources/06-memory-and-cpu-utilization.md).
- **Hibernate not working.** Swap must be large enough and the `resume=` kernel parameter must point to it.
- **Swap on cloud disks** with limited IOPS can be very slow. Monitor.
- **Encrypted data in swap.** Swap may contain sensitive data from memory. Use encrypted swap on laptops and sensitive systems.

## Further reading

- [`swapon(8)`](https://man7.org/linux/man-pages/man8/swapon.8.html)
- [`mkswap(8)`](https://man7.org/linux/man-pages/man8/mkswap.8.html)
- [Kernel: zswap](https://www.kernel.org/doc/html/latest/admin-guide/mm/zswap.html)
- [zram-generator](https://github.com/systemd/zram-generator)
