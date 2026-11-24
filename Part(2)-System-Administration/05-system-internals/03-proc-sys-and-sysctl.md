# /proc and /sys Virtual Filesystems and sysctl

## Overview

`/proc` and `/sys` look like directories full of files, but nothing is stored on disk. The kernel creates their content when you read them. They are the main window into the running kernel: reading shows status, and writing can change settings.

## Key concepts

| Filesystem | Mounted at | Purpose |
|------------|-----------|---------|
| `procfs` | `/proc` | Processes and general kernel information |
| `sysfs` | `/sys` | Devices, drivers, and kernel objects in a structured tree |
| `devtmpfs` | `/dev` | Device nodes |
| `tmpfs` | `/run`, `/dev/shm` | Memory-backed storage |

### Useful files in /proc

| File | Content |
|------|---------|
| `/proc/cpuinfo` | CPU details |
| `/proc/meminfo` | Memory statistics |
| `/proc/loadavg` | Load averages and process counts |
| `/proc/uptime` | Seconds since boot |
| `/proc/version` | Kernel version string |
| `/proc/cmdline` | Kernel boot parameters |
| `/proc/mounts` | Mounted filesystems |
| `/proc/filesystems` | Filesystems the kernel supports |
| `/proc/partitions` | Block devices and sizes |
| `/proc/interrupts` | Interrupt counts per CPU |
| `/proc/PID/` | One directory per process (see [Process Interfaces](06-process-interfaces.md)) |
| `/proc/sys/` | Tunable kernel parameters |

```console
$ cat /proc/loadavg
0.15 0.20 0.18 1/412 18345

$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-6.8.0-45-generic root=UUID=3f1c... ro quiet splash
```

### Useful paths in /sys

| Path | Content |
|------|---------|
| `/sys/class/net/` | Network interfaces |
| `/sys/block/` | Block devices |
| `/sys/class/power_supply/` | Batteries and power |
| `/sys/devices/` | The real device tree |
| `/sys/module/` | Loaded modules and parameters |
| `/sys/fs/cgroup/` | Control groups |

```console
$ cat /sys/class/net/eth0/operstate
up
$ cat /sys/class/net/eth0/mtu
1500
$ cat /sys/block/sda/queue/rotational     # 1 = spinning disk, 0 = SSD
0
```

### sysctl: kernel parameters

Parameters under `/proc/sys/` are called **sysctl** settings. The path `/proc/sys/net/ipv4/ip_forward` is the setting `net.ipv4.ip_forward`.

| Command | Purpose |
|---------|---------|
| `sysctl -a` | List all parameters |
| `sysctl NAME` | Read one |
| `sysctl -w NAME=VALUE` | Change until reboot (**root**) |
| `sysctl --system` | Reload all configuration files (**root**) |
| `sysctl -p FILE` | Load one file |

```console
$ sysctl vm.swappiness
vm.swappiness = 60

# sysctl -w vm.swappiness=10
vm.swappiness = 10
```

Make a change permanent with a file in `/etc/sysctl.d/`:

```console
# cat > /etc/sysctl.d/99-custom.conf <<'EOF'
vm.swappiness = 10
net.ipv4.ip_forward = 1
EOF
# sysctl --system
```

Files are read in alphabetical order across `/usr/lib/sysctl.d/`, `/etc/sysctl.d/`, and `/etc/sysctl.conf`. Later files win, so use a high number such as `99-`.

### Commonly tuned parameters

| Parameter | Meaning |
|-----------|---------|
| `vm.swappiness` | Tendency to use swap (0–100; lower = prefer RAM) |
| `net.ipv4.ip_forward` | Act as a router |
| `fs.file-max` | System-wide limit of open files |
| `kernel.pid_max` | Maximum PID |
| `net.core.somaxconn` | Maximum queued connections per socket |
| `fs.inotify.max_user_watches` | Number of file watches per user |
| `kernel.panic` | Seconds before reboot after a panic |

## Common pitfalls

- **`sysctl -w` is lost at reboot.** Put it in `/etc/sysctl.d/`.
- **Copying tuning values from the internet.** Understand the effect first. Wrong values can reduce performance or stability.
- **Writing to `/proc` and `/sys` with `sudo echo > file`.** The redirection runs as you. Use `echo VALUE | sudo tee /proc/sys/...` or `sysctl -w`.
- **Treating the files as normal files.** `ls -l` sizes are often `0`. Copying them or editing them with an editor does not work.
- **Container confusion.** Some sysctls are per-namespace and some are read-only inside containers.

## Further reading

- [`proc(5)`](https://man7.org/linux/man-pages/man5/proc.5.html)
- [`sysctl(8)`](https://man7.org/linux/man-pages/man8/sysctl.8.html)
- [`sysctl.d(5)`](https://man7.org/linux/man-pages/man5/sysctl.d.5.html)
- [Kernel sysctl documentation](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/index.html)
