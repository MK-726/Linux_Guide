# Boot Troubleshooting and Recovery

## Overview

When a system does not boot, work **from the start of the boot chain forward**. Find which stage fails, then use the correct recovery tool. Practice these steps in a virtual machine before you need them.

> [!WARNING]
> Recovery work happens on an unbootable system, often with direct access to disks. Take a **snapshot or backup** first if you can. Mistakes with `fsck`, `grub-install`, or partitioning tools can destroy data.

## Key concepts

### Which stage failed?

| Symptom | Likely stage | First things to try |
|---------|--------------|---------------------|
| No output, no boot device | Firmware | Boot order, disk detection, cables, hardware |
| `grub>` or `grub rescue>` prompt | Bootloader | Find the root, reinstall GRUB |
| "error: file not found" / "no such partition" | Bootloader | Disk UUID changed, GRUB config outdated |
| Kernel panic: "VFS: Unable to mount root fs" | Kernel / initramfs | Missing driver, wrong `root=`, rebuild initramfs |
| Dropped to `dracut:/#` or `(initramfs)` | initramfs | Root device not found, LVM/RAID not assembled |
| "You are in emergency mode" | systemd | Failed mount in `/etc/fstab`, damaged filesystem |
| Boots but services missing | systemd / services | `systemctl --failed`, `journalctl -b` |
| Hangs with "A start job is running..." | systemd | Wait for timeout, find the unit (often a mount or network) |

### Targets used for recovery

| Target | What you get |
|--------|--------------|
| `rescue.target` | Basic system: local filesystems mounted, few services, root shell |
| `emergency.target` | Minimal: root filesystem mounted **read-only**, root shell, almost nothing else |

Both ask for the **root password** (unless the account has none). If you do not know it, use the recovery methods below.

## Commands and procedures

### 1. Boot into rescue or emergency mode

At the GRUB menu press `e`, add one of these to the `linux` line, and press `Ctrl+X`:

```
systemd.unit=rescue.target
systemd.unit=emergency.target
```

In emergency mode, remount the root read-write to make changes:

```console
# mount -o remount,rw /
```

### 2. Fix an /etc/fstab error (very common)

A wrong entry in `/etc/fstab` (missing disk, bad UUID) drops the system to emergency mode.

```console
# journalctl -xb | grep -i -E 'mount|fstab'
# blkid                                  # check real UUIDs
# mount -o remount,rw /
# vi /etc/fstab                          # fix or comment out the bad line
# mount -a                               # test all entries
# systemctl daemon-reload
# reboot
```

For non-critical disks, add `nofail` (and optionally `x-systemd.device-timeout=10s`) in the options so a missing disk does not stop boot.

### 3. Check and repair a filesystem

Run `fsck` only on **unmounted** (or read-only) filesystems.

```console
# umount /dev/sdb1
# fsck -f /dev/sdb1                  # ext2/3/4
# xfs_repair /dev/sdb1               # XFS (use xfs_repair, not fsck)
```

> [!WARNING]
> Never run `fsck` on a mounted filesystem. If the disk shows hardware errors, copy data off first (`ddrescue`) and check SMART (see Chapter 9).

### 4. Reset a forgotten root password

**RHEL-based (with SELinux):**

1. At the GRUB menu press `e`, add `rd.break` at the end of the `linux` line, and press `Ctrl+X`.
2. In the initramfs shell:

```console
switch_root:/# mount -o remount,rw /sysroot
switch_root:/# chroot /sysroot
sh-5.1# passwd root
sh-5.1# touch /.autorelabel        # needed so SELinux labels are fixed
sh-5.1# exit
switch_root:/# exit
```

The system reboots (the relabel takes a while).

**Debian-based:**

1. At GRUB press `e`, change `ro` to `rw` and add `init=/bin/bash` at the end of the `linux` line, then `Ctrl+X`.
2. In the shell:

```console
root@(none):/# passwd root
root@(none):/# exec /sbin/init       # or: sync; reboot -f
```

> [!IMPORTANT]
> These methods work because anyone with console access can do them. This is why **physical security**, GRUB passwords, and **full-disk encryption** matter. On cloud VMs use the provider's recovery or "rescue instance" feature instead.

### 5. Repair GRUB from a live system (chroot)

Boot from a live ISO or rescue image, then:

```console
# lsblk -f                                    # find root and EFI partitions
# mount /dev/sda2 /mnt                        # root partition
# mount /dev/sda1 /mnt/boot/efi               # ESP (UEFI only)
# mount --bind /dev  /mnt/dev
# mount --bind /proc /mnt/proc
# mount --bind /sys  /mnt/sys
# mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars   # UEFI only
# chroot /mnt
```

Inside the chroot:

**Debian-based**

```console
# grub-install /dev/sda                       # BIOS
# grub-install --target=x86_64-efi --efi-directory=/boot/efi   # UEFI
# update-grub
```

**RHEL-based**

```console
# dnf reinstall grub2-efi-x64 shim-x64        # UEFI
# grub2-mkconfig -o /boot/grub2/grub.cfg
# grub2-install /dev/sda                      # BIOS only
```

Then `exit`, unmount everything, and reboot.

### 6. Rebuild the initramfs

Needed after missing-driver errors, storage changes, or a damaged initramfs. Run inside the booted system or a chroot.

| Family | Command |
|--------|---------|
| Debian-based | `update-initramfs -u -k all` |
| RHEL-based | `dracut -f --regenerate-all` |
| SUSE-based | `mkinitrd` or `dracut -f` |

### 7. Boot an older kernel

In the GRUB menu choose **Advanced options** and pick a previous kernel. If it works, the new kernel or its modules are the problem. Keep the old kernel and investigate.

### 8. Manual boot from the GRUB prompt

If you land at `grub>`:

```
grub> ls
(hd0) (hd0,gpt1) (hd0,gpt2)
grub> ls (hd0,gpt2)/
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz root=/dev/sda2 ro
grub> initrd /initrd.img
grub> boot
```

After you are in the system, run `update-grub` or `grub2-mkconfig` and reinstall GRUB.

### 9. Find out what went wrong after you are in

```console
# systemctl --failed
# systemctl is-system-running
# journalctl -b -p err
# journalctl -b -1 -p err              # previous boot (if the journal is persistent)
# dmesg --level=err,warn
# systemd-analyze blame
# systemd-analyze critical-chain
```

## Common pitfalls

- **No console access.** Servers without iDRAC, iLO, IPMI, or a cloud serial console are hard to recover. Set it up **before** you need it.
- **Forgetting `/.autorelabel`** on SELinux systems after a password reset leads to login failures.
- **Mounting the wrong partition** in a chroot, then reinstalling the bootloader to the wrong disk.
- **Disabling SELinux permanently** to fix a boot problem. Use `enforcing=0` temporarily, then fix labels.
- **Skipping `mount -a`** to test fstab changes before rebooting.
- **Ignoring hardware.** Repeated filesystem errors usually mean a failing disk. Check SMART and replace it.

## Further reading

- [`systemd.special(7)`: rescue and emergency](https://man7.org/linux/man-pages/man7/systemd.special.7.html)
- [`dracut.cmdline(7)`](https://man7.org/linux/man-pages/man7/dracut.cmdline.7.html)
- [GRUB manual: rescue mode](https://www.gnu.org/software/grub/manual/grub/html_node/GRUB-only-offers-a-rescue-shell.html)
- [`fstab(5)`](https://man7.org/linux/man-pages/man5/fstab.5.html)
