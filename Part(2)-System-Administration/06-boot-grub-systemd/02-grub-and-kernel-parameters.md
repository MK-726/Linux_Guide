# GRUB Configuration and Kernel Parameters

## Overview

**GRUB 2** is the bootloader on most Linux systems. It shows the boot menu, loads the kernel and initramfs, and passes **kernel parameters** (boot options) to the kernel.

## Key concepts

### Where GRUB is configured

| File / directory | Purpose |
|------------------|---------|
| `/etc/default/grub` | Main settings (you edit this) |
| `/etc/grub.d/` | Scripts that generate menu entries (for example `40_custom`) |
| `/boot/grub/grub.cfg` (Debian-based) | **Generated** config. Do not edit by hand |
| `/boot/grub2/grub.cfg` (RHEL-based, SUSE-based) | **Generated** config. Do not edit by hand |

> [!IMPORTANT]
> After you edit `/etc/default/grub` or `/etc/grub.d/*`, you must **regenerate** `grub.cfg`. Editing `grub.cfg` directly is lost at the next kernel update.

### Common settings in /etc/default/grub

```
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"
GRUB_DISABLE_OS_PROBER=false
```

| Setting | Meaning |
|---------|---------|
| `GRUB_DEFAULT` | Menu entry to boot: number, `saved`, or an entry name |
| `GRUB_TIMEOUT` | Seconds the menu waits |
| `GRUB_CMDLINE_LINUX_DEFAULT` | Kernel parameters for **normal** boot entries only |
| `GRUB_CMDLINE_LINUX` | Kernel parameters for **all** entries, including recovery |
| `GRUB_DISABLE_OS_PROBER` | Controls detection of other operating systems |

## Commands

### Regenerate the configuration (requires root)

| Family | Command |
|--------|---------|
| Debian-based | `update-grub` (same as `grub-mkconfig -o /boot/grub/grub.cfg`) |
| RHEL-based | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| SUSE-based | `grub2-mkconfig -o /boot/grub2/grub.cfg` |

> [!NOTE]
> On some RHEL versions with UEFI, `grub.cfg` lives under `/boot/efi/EFI/<vendor>/`. Check the Red Hat documentation for your exact release before you regenerate it.

```console
# update-grub
Sourcing file `/etc/default/grub'
Found linux image: /boot/vmlinuz-6.8.0-45-generic
Found initrd image: /boot/initrd.img-6.8.0-45-generic
done
```

### Install GRUB to disk (requires root)

```console
# grub-install /dev/sda                       # BIOS, Debian-based
# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu   # UEFI
# grub2-install /dev/sda                      # BIOS, RHEL-based or SUSE-based
```

On RHEL with UEFI, do not use `grub2-install`. Reinstall the packages `grub2-efi-x64` and `shim-x64` instead.

### RHEL-based: grubby

`grubby` changes kernel parameters in the boot entries directly:

```console
# grubby --info=ALL | grep -E 'kernel|args'
# grubby --update-kernel=ALL --args="console=ttyS0,115200"
# grubby --update-kernel=ALL --remove-args="rhgb quiet"
# grubby --default-kernel
```

### Change the default kernel

```console
# grub2-set-default 1                  # RHEL-based; Debian uses GRUB_DEFAULT with "saved"
# grub2-editenv list
```

### Kernel parameters

View the parameters of the running kernel:

```console
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-6.8.0-45-generic root=UUID=3f1c... ro quiet splash
```

| Parameter | Meaning |
|-----------|---------|
| `root=UUID=...` | The root filesystem |
| `ro` / `rw` | Mount root read-only / read-write at first |
| `quiet` | Fewer boot messages |
| `nomodeset` | Do not load graphics drivers (fixes some video problems) |
| `console=ttyS0,115200` | Send console output to a serial port |
| `net.ifnames=0` | Use old `eth0` names instead of predictable names |
| `systemd.unit=rescue.target` | Boot into rescue mode |
| `systemd.unit=emergency.target` | Boot into emergency mode |
| `systemd.log_level=debug` | More systemd logging |
| `rd.break` | Stop in the initramfs shell (RHEL-based, dracut) |
| `init=/bin/bash` | Start a shell instead of systemd (last resort) |
| `selinux=0`, `enforcing=0` | Disable SELinux, or set permissive (troubleshooting only) |
| `mem=`, `maxcpus=` | Limit memory or CPUs |

### One-time change from the GRUB menu

1. At the boot menu, highlight the entry and press **`e`**.
2. Find the line that starts with `linux` (or `linuxefi`) and press **End**.
3. Add the parameters, for example ` systemd.unit=rescue.target`.
4. Press **Ctrl+X** (or **F10**) to boot.

This change is **not saved**.

### Protect GRUB with a password (optional)

Anyone with console access can edit boot parameters (and reset passwords). A GRUB password protects the menu editor. Use `grub-mkpasswd-pbkdf2` and add the superuser in `/etc/grub.d/40_custom`, then regenerate. Physical security and disk encryption are still needed.

## Common pitfalls

- **Editing `grub.cfg` directly.** The change disappears at the next update. Edit `/etc/default/grub`.
- **Forgetting to regenerate.** The new settings are not active.
- **Typos in kernel parameters** can make the system unbootable. Test once from the menu with `e` first.
- **Wrong disk for `grub-install`.** Install to the **disk** (`/dev/sda`), not a partition (`/dev/sda1`), for BIOS.
- **Dual boot:** Windows updates may replace the boot entry. Re-run `update-grub` and check `efibootmgr`.
- **Old kernels remain.** Keep at least one known-good kernel for fallback.

## Further reading

- [GNU GRUB manual](https://www.gnu.org/software/grub/manual/grub/grub.html)
- [Kernel command-line parameters](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html)
- [`kernel-command-line(7)`](https://man7.org/linux/man-pages/man7/kernel-command-line.7.html)
