# Partitioning: GPT and MBR

## Overview

A **partition** is a section of a disk that you use as a separate block device. A **partition table** at the start of the disk stores where each partition begins and ends. Two formats exist: **MBR** (old) and **GPT** (modern).

> [!WARNING]
> Changing a partition table can make existing data unreachable. Back up first. Double-check the device name. Changes in `fdisk` and `gdisk` are held in memory and only written when you press **`w`**. Press **`q`** to quit without saving.

## Key concepts

### MBR vs. GPT

| | **MBR** (msdos) | **GPT** |
|---|-----------------|---------|
| Maximum disk size | 2 TiB | 9.4 ZB (practically unlimited) |
| Partitions | 4 primary (or 3 primary + 1 extended with logical partitions) | 128 by default (more possible) |
| Identifiers | 32-bit disk ID | UUIDs for disk and partitions, partition names |
| Redundancy | None | Backup table at the end of the disk, CRC checksums |
| Boot | BIOS | UEFI (also BIOS with a "BIOS boot" partition) |
| Use today | Old systems, small disks, compatibility | **Default for new systems** |

### Common partitions

| Partition | Typical size | Purpose |
|-----------|-------------|---------|
| **EFI System Partition (ESP)** | 256 MiB–1 GiB, FAT32 | UEFI bootloader files (`/boot/efi`) |
| **BIOS boot partition** | 1 MiB, no filesystem | GRUB core image on GPT disks with BIOS firmware |
| `/boot` | 512 MiB–1 GiB | Kernels and initramfs (optional if root is simple) |
| Root (`/`) | 20 GiB and up | System |
| Swap | See [Swap](05-swap.md) | Virtual memory |
| `/home`, `/var`, data | As needed | Separate partitions or volumes isolate growth and enable different options |

For servers, using **LVM** (see [LVM](06-lvm.md)) for everything except `/boot` and the ESP makes resizing easy.

### Tools

| Tool | Notes |
|------|-------|
| `fdisk` | Interactive. Supports MBR and GPT. The most common tool |
| `gdisk` / `sgdisk` | GPT-focused. `sgdisk` is scriptable |
| `parted` | Supports both. Applies changes **immediately**. Good for scripts (`parted -s`) |
| `sfdisk` | Scriptable; dump and restore tables |
| `cfdisk` | Text-menu interface |
| `partprobe` / `partx` | Tell the kernel to re-read the table |

## Commands

### Create a GPT partition with fdisk

Example: one partition on new disk `/dev/sdb`.

```console
# fdisk /dev/sdb

Command (m for help): g                      # create a new empty GPT table
Created a new GPT disklabel (GUID: 7C3F...).

Command (m for help): n                      # new partition
Partition number (1-128, default 1):         # Enter
First sector (2048-419430366, default 2048): # Enter
Last sector, +/-sectors or +/-size{K,M,G,T,P}: # Enter = use all the disk

Created a new partition 1 of type 'Linux filesystem' and of size 200 GiB.

Command (m for help): p                      # print and check
Command (m for help): w                      # WRITE changes and exit
The partition table has been altered.
Syncing disks.
```

Useful `fdisk` keys: `m` help, `p` print, `n` new, `d` delete, `t` change type, `g` new GPT, `o` new MBR, `w` write, `q` quit without saving.

For **LVM or RAID** partitions, set the type with `t` (for example "Linux LVM", "Linux RAID").

### Scriptable creation

```console
# parted -s /dev/sdb mklabel gpt mkpart data ext4 1MiB 100%
# sgdisk -n 1:0:0 -t 1:8300 -c 1:data /dev/sdb        # GPT, one partition, type Linux filesystem
# printf 'label: gpt\n,,L\n' | sfdisk /dev/sdb          # sfdisk script
```

### Tell the kernel about changes

```console
# partprobe /dev/sdb              # or: partx -u /dev/sdb
# udevadm settle                  # wait for device nodes
$ lsblk /dev/sdb
```

If the disk is in use (mounted partitions), the kernel may keep the old table until reboot. `partprobe` may say so.

### Inspect and backup the table

```console
# sgdisk -p /dev/sdb              # print GPT
# parted /dev/sdb print
# sfdisk -d /dev/sdb > sdb.parttable          # backup
# sfdisk /dev/sdb < sdb.parttable             # restore (identical layout)
# sgdisk --backup=sdb.gpt /dev/sdb            # GPT backup
# sgdisk -R=/dev/sdc /dev/sdb && sgdisk -G /dev/sdc    # clone layout to sdc and give new GUIDs
```

### Wipe old signatures

Old filesystem or RAID signatures can confuse the system. `wipefs` shows and removes them.

```console
# wipefs /dev/sdb                 # list only (no change)
# wipefs -a /dev/sdb              # ERASE all signatures (destructive)
```

> [!WARNING]
> `wipefs -a` and `dd if=/dev/zero of=/dev/sdX` destroy data at once. Verify the device twice.

### Convert MBR to GPT without data loss

```console
# gdisk /dev/sda            # reads MBR, converts in memory, 'w' to write
# sgdisk -g /dev/sda        # same, scripted
```

Then add a BIOS boot partition (type `ef02`) if the system uses BIOS, and reinstall GRUB. Back up first, and test on a copy. This is a risky operation on a boot disk.

### Next steps after partitioning

1. Create a filesystem: `mkfs.ext4 /dev/sdb1` ([Filesystems](03-filesystems.md)).
2. Mount it ([Mounting](04-mounting-fstab-uuids-labels.md)), or add it to LVM or RAID.

## Common pitfalls

- **Forgetting `w` in `fdisk`.** Nothing is saved until you write.
- **Using MBR on disks over 2 TiB.** The space beyond 2 TiB cannot be used. Use GPT.
- **Overlapping or misaligned partitions.** Accept the defaults (first sector 2048).
- **Kernel does not see the new partition.** Run `partprobe`. If the disk is busy, reboot.
- **Deleting the wrong partition.** `fdisk` shows the plan, and `q` abandons it. Read `p` output before `w`.
- **Cloned disks with the same UUIDs.** After `dd` clones or `sfdisk` restores, duplicate UUIDs can confuse mounts. Change them (`sgdisk -G`, `tune2fs -U`, `xfs_admin -U generate`).
- **No ESP or BIOS boot partition** on a new boot disk, so GRUB cannot be installed.

## Further reading

- [`fdisk(8)`](https://man7.org/linux/man-pages/man8/fdisk.8.html)
- [`parted` manual](https://www.gnu.org/software/parted/manual/parted.html)
- [`sgdisk(8)`](https://man7.org/linux/man-pages/man8/sgdisk.8.html)
- [UEFI GPT specification](https://uefi.org/specifications)
