# Kernel Modules

## Overview

A **kernel module** is a piece of kernel code (usually a driver or a filesystem) that can be loaded or removed without rebooting. Modules keep the kernel small and let the system load only what the hardware needs.

## Key concepts

- Module files end in `.ko` (kernel object), often compressed (`.ko.xz`, `.ko.zst`).
- They live in `/lib/modules/<kernel-version>/`. Each kernel version has its own set.
- Modules can **depend** on other modules. `modprobe` handles this automatically. `insmod` does not.
- Udev loads most modules automatically when hardware appears.
- Some features are **built in** to the kernel and do not appear as modules.

## Commands

| Command | Purpose |
|---------|---------|
| `lsmod` | List loaded modules |
| `modinfo NAME` | Show information about a module |
| `modprobe NAME` | Load a module and its dependencies (**root**) |
| `modprobe -r NAME` | Unload a module and unused dependencies (**root**) |
| `modprobe -n -v NAME` | Dry run: show what would happen |
| `insmod FILE.ko` | Load one file only (no dependency handling) |
| `rmmod NAME` | Unload one module |
| `depmod -a` | Rebuild the dependency list |
| `dmesg \| tail` | See kernel messages after loading |

```console
$ lsmod | head -4
Module                  Size  Used by
br_netfilter           32768  0
bridge                307200  1 br_netfilter
overlay               163840  2

$ modinfo overlay
filename:       /lib/modules/6.8.0-45-generic/kernel/fs/overlayfs/overlay.ko.zst
description:    Overlay filesystem
license:        GPL
depends:
parm:           redirect_dir:Default to on or off for the redirect_dir feature (bool)

# modprobe overlay
$ lsmod | grep overlay
overlay               163840  0
```

`Used by` shows the number of users and which modules depend on this one. A module that is in use cannot be removed.

### Find out which module a device uses

```console
$ lspci -k | grep -A3 -i ethernet
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller
        Kernel driver in use: e1000
        Kernel modules: e1000
```

### Load modules at boot

Put one module name per line in a file under `/etc/modules-load.d/`:

```console
# echo overlay > /etc/modules-load.d/overlay.conf
# echo br_netfilter >> /etc/modules-load.d/overlay.conf
```

### Module options and blacklisting

Create files in `/etc/modprobe.d/`:

```
# /etc/modprobe.d/e1000.conf
options e1000 debug=1

# /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
install nouveau /bin/false
```

| Directive | Effect |
|-----------|--------|
| `options MODULE param=value` | Parameters used when the module loads |
| `blacklist MODULE` | Prevent **automatic** loading by alias |
| `install MODULE /bin/false` | Prevent loading completely, even by dependency or manually |

After changing blacklist rules for a module that is also in the initramfs, rebuild it:

**Debian-based:** `# update-initramfs -u`
**RHEL-based:** `# dracut -f`

### Check parameters of a loaded module

```console
$ ls /sys/module/e1000/parameters/
$ cat /sys/module/overlay/parameters/redirect_dir
```

## Configuration files

| Path | Purpose |
|------|---------|
| `/lib/modules/$(uname -r)/` | Module files and dependency data |
| `/etc/modules-load.d/*.conf` | Modules to load at boot |
| `/etc/modprobe.d/*.conf` | Options, aliases, blacklists |
| `/usr/lib/modprobe.d/` | Distribution defaults |

## Common pitfalls

- **"Module not found" after a kernel update.** The modules for the new kernel were not installed, or you booted a kernel whose `/lib/modules/<version>` is missing. Compare `uname -r` and `ls /lib/modules`.
- **"Operation not permitted" on Secure Boot systems.** The kernel only loads **signed** modules. Third-party drivers must be signed (for example with MOK enrollment).
- **Unloading a module in use** fails ("Module is in use"). Stop the service or unmount the filesystem first.
- **Using `rmmod -f`.** Forced removal can crash the system.
- **Blacklist ignored.** The module is loaded from the initramfs, or another module depends on it. Use `install MODULE /bin/false` and rebuild the initramfs.

> [!WARNING]
> Loading or removing storage, network, or filesystem modules on a production server can interrupt services. Test first.

## Further reading

- [`modprobe(8)`](https://man7.org/linux/man-pages/man8/modprobe.8.html)
- [`modprobe.d(5)`](https://man7.org/linux/man-pages/man5/modprobe.d.5.html)
- [Kernel module signing](https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html)
