# BIOS, UEFI, and the Boot Sequence

## Overview

Booting is a chain of small programs. Each one starts the next, until the system is ready. Knowing the chain tells you where to look when a server does not start.

## Key concepts

### The boot sequence

```
Power on
   │
   ▼
Firmware (BIOS or UEFI) ── hardware test (POST), finds boot device
   │
   ▼
Bootloader (GRUB2) ── shows menu, loads kernel and initramfs
   │
   ▼
Kernel ── starts, detects hardware, mounts the temporary root
   │
   ▼
initramfs ── loads drivers, finds and mounts the real root filesystem
   │
   ▼
init (systemd, PID 1) ── starts services until the default target is reached
   │
   ▼
Login prompt / services running
```

### BIOS vs. UEFI

| | **BIOS** (legacy) | **UEFI** |
|---|------------------|----------|
| Partition table | MBR (limit 2 TB per disk) | GPT (also works with MBR) |
| Boot code location | First 440 bytes of the disk (MBR) | A file on the **EFI System Partition (ESP)** |
| Boot menu | Simple | Boot entries stored in NVRAM |
| Secure Boot | No | Yes |
| Status | Old, being retired | Standard on modern hardware |

The **ESP** is a small FAT32 partition (100–1000 MB), usually mounted at `/boot/efi`. It holds bootloader files such as `\EFI\ubuntu\grubx64.efi`.

Check how the system booted:

```console
$ [ -d /sys/firmware/efi ] && echo UEFI || echo BIOS
UEFI

$ bootctl status 2>/dev/null | head -5
$ efibootmgr -v           # UEFI boot entries (needs root)
BootCurrent: 0001
Boot0001* ubuntu  HD(1,GPT,...)/File(\EFI\ubuntu\shimx64.efi)
```

### Secure Boot

With Secure Boot, the firmware only runs **signed** bootloaders. Linux uses a signed **shim** that then verifies GRUB and the kernel. Unsigned kernel modules are blocked (see [Kernel Modules](../05-system-internals/02-kernel-modules.md)). Check the state:

```console
$ mokutil --sb-state
SecureBoot enabled
```

### The kernel and initramfs

| File | Purpose |
|------|---------|
| `/boot/vmlinuz-<version>` | Compressed kernel |
| `/boot/initrd.img-<version>` (Debian) or `/boot/initramfs-<version>.img` (RHEL) | **initramfs**: a small temporary root filesystem loaded into memory |
| `/boot/grub/` or `/boot/grub2/` | GRUB files |
| `/boot/efi/` | ESP mount point (UEFI) |

The **initramfs** exists because the kernel may need drivers (RAID, LVM, encrypted disks, special storage) **before** it can read the real root filesystem. The initramfs contains those drivers and tools, mounts the real root, then hands control to `/sbin/init` (systemd).

Look inside:

```console
$ lsinitramfs /boot/initrd.img-$(uname -r) | head        # Debian-based
$ lsinitrd /boot/initramfs-$(uname -r).img | head        # RHEL-based
```

### Stage 5: systemd

systemd (PID 1) reads the **default target**, works out the dependencies, and starts units in parallel. When the target is reached, the system is up. See [systemd Architecture](03-systemd-architecture-targets-units.md).

## Commands

| Command | Purpose |
|---------|---------|
| `systemd-analyze` | Total boot time (firmware, loader, kernel, userspace) |
| `systemd-analyze blame` | Services ordered by start time |
| `systemd-analyze critical-chain` | The chain that delayed boot |
| `journalctl -b` | Messages from this boot |
| `dmesg` | Kernel ring buffer |
| `lsblk -f` | Disks, partitions, filesystems |
| `last reboot` | Boot history |

```console
$ systemd-analyze
Startup finished in 3.2s (firmware) + 2.1s (loader) + 1.8s (kernel) + 9.4s (userspace) = 16.5s
graphical.target reached after 9.2s in userspace
```

## Common pitfalls

- **Blaming Linux when the firmware fails.** If the screen never reaches GRUB, the problem is hardware, firmware settings, or the boot order.
- **Mixed BIOS and UEFI installs.** A disk installed in UEFI mode will not boot in legacy mode, and the reverse.
- **Full or missing `/boot`.** Old kernels fill `/boot`, and updates fail. Remove old kernels with the package manager.
- **Changing storage drivers without rebuilding the initramfs.** The system may fail to find its root disk.
- **VM or cloud images** may boot through different mechanisms (for example, cloud-init and special console access).

## Further reading

- [`boot(7)`](https://man7.org/linux/man-pages/man7/boot.7.html)
- [`bootup(7)`](https://man7.org/linux/man-pages/man7/bootup.7.html)
- [UEFI specification overview](https://uefi.org/specifications)
