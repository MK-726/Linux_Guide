# Device Management, udev, and Hardware Discovery

## Overview

In Linux, hardware appears as **device files** in `/dev`. The **udev** system creates these files when hardware is detected, gives them stable names, and runs rules (for example, set permissions or start a service). This page also lists the tools used to discover what hardware a system has.

## Key concepts

### Device files

| Type | Symbol in `ls -l` | Examples |
|------|-------------------|----------|
| **Block device** | `b` | Disks and partitions: `/dev/sda`, `/dev/nvme0n1p1` |
| **Character device** | `c` | Terminals, `/dev/null`, `/dev/random`, serial ports |

Each device file has a **major** number (the driver) and a **minor** number (the instance).

```console
$ ls -l /dev/sda /dev/null
brw-rw---- 1 root disk 8, 0 Mar  3 09:00 /dev/sda
crw-rw-rw- 1 root root 1, 3 Mar  3 09:00 /dev/null
```

Special devices:

| Device | Behavior |
|--------|----------|
| `/dev/null` | Discards everything written; reads return nothing |
| `/dev/zero` | Endless zero bytes |
| `/dev/random`, `/dev/urandom` | Random data |
| `/dev/tty` | The current terminal |
| `/dev/stdin`, `/dev/stdout`, `/dev/stderr` | Links to the process's descriptors |

### How udev works

1. The kernel detects a device and sends an event (a **uevent**).
2. `systemd-udevd` receives it and reads its **rules**.
3. It creates the device node in `/dev`, sets owner and permissions, creates symbolic links, and may load a module or run a program.

### Stable device names

Names such as `/dev/sdb` can change between boots. Use stable names:

```console
$ ls -l /dev/disk/
by-id  by-label  by-partuuid  by-path  by-uuid
$ ls -l /dev/disk/by-uuid/
lrwxrwxrwx 1 root root 10 Mar  3 09:00 3f1c9a1e-... -> ../../sda1
```

Use UUIDs or labels in `/etc/fstab` and scripts (see Chapter 9).

### udev rules

| Location | Purpose |
|----------|---------|
| `/usr/lib/udev/rules.d/` | Distribution and package rules (do not edit) |
| `/etc/udev/rules.d/` | Your rules; same file name overrides the system one |

Files are processed in order by file name (`10-`, `50-`, `99-`). Example rule that creates a fixed name for a USB serial adapter:

```
# /etc/udev/rules.d/99-usb-serial.rules
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", ATTRS{serial}=="A12345", SYMLINK+="ttyPRINTER", MODE="0660", GROUP="dialout"
```

`==` compares, `=` and `+=` assign.

## Commands

### udevadm

| Command | Purpose |
|---------|---------|
| `udevadm info -a /dev/sdb` | Show all attributes (use them to write rules) |
| `udevadm info /dev/sdb` | Show udev's database entry |
| `udevadm monitor` | Watch events live (plug in a device to see it) |
| `udevadm test /sys/class/net/eth0` | Simulate rule processing |
| `udevadm control --reload` | Reload rules (**root**) |
| `udevadm trigger` | Re-run rules for existing devices (**root**) |

```console
$ udevadm monitor --udev
UDEV  [4123.001] add      /devices/pci0000:00/.../usb1/1-1 (usb)
UDEV  [4123.050] add      /devices/.../block/sdb (block)
```

### Discover hardware

| Command | Shows | Package (if missing) |
|---------|-------|----------------------|
| `lspci` | PCI devices (network cards, GPUs, controllers) | `pciutils` |
| `lspci -k` | Plus the driver in use | |
| `lsusb` | USB devices | `usbutils` |
| `lsblk` | Block devices as a tree | `util-linux` |
| `lscpu` | CPU | `util-linux` |
| `lshw -short` | Full hardware inventory (**root** for full detail) | `lshw` |
| `dmidecode` | BIOS, system, memory slots (**root**) | `dmidecode` |
| `hwinfo --short` | Hardware list (SUSE-based) | `hwinfo` |
| `lsmod` | Loaded drivers | |
| `dmesg` | Kernel detection messages | |
| `inxi -F` | Summary (third-party tool) | `inxi` |

```console
$ lspci | head -3
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma]
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller
00:04.0 SCSI storage controller: Red Hat, Inc. Virtio block device

$ lsusb
Bus 001 Device 002: ID 0781:5581 SanDisk Corp. Ultra

# lshw -short -class network
H/W path        Device   Class      Description
/0/100/3        eth0     network    82540EM Gigabit Ethernet Controller

# dmidecode -s system-product-name
Standard PC (Q35 + ICH9, 2009)
```

### Hardware not working: checklist

1. Is the device detected? `lspci`, `lsusb`, `lsblk`
2. Did the kernel report errors? `dmesg --level=err,warn` or `journalctl -k -p warning`
3. Is a driver loaded? `lspci -k`, `lsmod`
4. Is firmware missing? Look for `firmware: failed to load` in `dmesg`. Install a package such as `linux-firmware`.
5. Does udev create the node? `udevadm monitor`

## Common pitfalls

- **Using `/dev/sdX` names in scripts and fstab.** They can change. Use UUIDs or `/dev/disk/by-id`.
- **Editing files in `/usr/lib/udev/rules.d/`.** Updates overwrite them. Copy to `/etc/udev/rules.d/`.
- **Rule changes not applied.** Run `udevadm control --reload` and `udevadm trigger`.
- **Rules that run long programs.** udev expects short tasks. Start services with systemd instead (`ENV{SYSTEMD_WANTS}="..."`).
- **Virtual machines and containers** show virtual hardware only.

## Further reading

- [`udev(7)`](https://man7.org/linux/man-pages/man7/udev.7.html)
- [`udevadm(8)`](https://man7.org/linux/man-pages/man8/udevadm.8.html)
- [`lspci(8)`](https://man7.org/linux/man-pages/man8/lspci.8.html)
