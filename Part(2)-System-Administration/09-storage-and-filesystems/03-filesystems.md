# Filesystems: ext4, XFS, Btrfs, and Others

## Overview

A **filesystem** organizes data on a block device into files and directories. Before you can use a partition or volume, you must create a filesystem on it (**format** it). The filesystem decides performance, features, and how you can resize and repair it.

## Key concepts

### Terms

| Term | Meaning |
|------|---------|
| **Journaling** | The filesystem logs changes before it applies them, so after a crash it recovers quickly and stays consistent |
| **Extent** | A range of blocks. Extents describe large files efficiently |
| **Inode** | Metadata of a file (see [File Types, Inodes, and Links](../../part-1-foundations/03-filesystem-and-file-management/02-file-types-inodes-and-links.md)) |
| **Copy-on-write (CoW)** | Data is never overwritten in place. New blocks are written. Enables snapshots and checksums |
| **Block size** | Smallest allocation unit (usually 4 KiB) |

### Common filesystems

| Filesystem | Strengths | Limits and notes |
|------------|-----------|------------------|
| **ext4** | Mature, stable, simple, good general purpose. Default on Debian and Ubuntu | Grow online; **shrink offline only** |
| **XFS** | Excellent with large files and parallel I/O, scalable. **Default on RHEL-based** systems | **Cannot be shrunk.** Grow online |
| **Btrfs** | Snapshots, subvolumes, checksums, compression, built-in RAID. Default for root on SUSE | More complex; RAID 5/6 modes are not recommended for production |
| **ZFS** (OpenZFS) | Very strong integrity, snapshots, pools | Not in the mainline kernel (license). Installed separately. Not covered in depth here |
| **vfat (FAT32)** | Compatible everywhere; used for the EFI partition | No permissions, 4 GiB file limit |
| **exFAT** | Removable media, large files | No permissions |
| **NTFS** | Windows disks (`ntfs3` kernel driver or `ntfs-3g`) | Permission model differs |
| **tmpfs** | Memory-backed. `/run`, `/dev/shm`, `/tmp` on some systems | Lost at reboot |
| **NFS / SMB** | Network filesystems | See Chapter 13 |

### Defaults by family

| Family | Root filesystem default | Notes |
|--------|------------------------|-------|
| Debian-based | ext4 | |
| RHEL-based | XFS | Installs on LVM by default |
| SUSE-based | Btrfs (root) | XFS for `/home` on SLES by default; snapshots with snapper |

## Commands

### Create a filesystem (destroys existing data on the device)

```console
# mkfs.ext4 /dev/sdb1                        # ext4
# mkfs.ext4 -L data /dev/sdb1                # with a label
# mkfs.xfs -L data /dev/sdb1                 # XFS (-f to overwrite an old filesystem)
# mkfs.btrfs -L data /dev/sdb1               # Btrfs
# mkfs.vfat -F 32 -n EFI /dev/sdb1           # FAT32
# mkfs -t ext4 /dev/sdb1                     # generic form
```

```console
# mkfs.ext4 -L data /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 52428544 4k blocks and 13107200 inodes
Filesystem UUID: 5b0c3e9a-3d6a-4c5b-8a77-3f2f6e0a9d11
...
Writing superblocks and filesystem accounting information: done
```

> [!WARNING]
> `mkfs` erases the filesystem on that device. Check the name with `lsblk -f` first. If the device already contains a filesystem, `mkfs` may ask for confirmation. Do not use `-f` or `-F` unless you are sure.

### Inspect

| Command | Purpose |
|---------|---------|
| `lsblk -f` | Filesystem type, label, UUID |
| `blkid /dev/sdb1` | UUID, label, type |
| `df -hT` | Used and free space, type |
| `df -i` | Inode usage |
| `dumpe2fs -h /dev/sdb1` | ext4 superblock information |
| `tune2fs -l /dev/sdb1` | ext4 settings |
| `xfs_info /mountpoint` | XFS geometry (the filesystem must be mounted) |
| `btrfs filesystem show` / `btrfs filesystem usage /mnt` | Btrfs overview |
| `stat -f /mountpoint` | Filesystem statistics |

### Maintain

| Task | ext4 | XFS | Btrfs |
|------|------|-----|-------|
| Change label | `e2label /dev/sdb1 NAME` | `xfs_admin -L NAME /dev/sdb1` (unmounted) | `btrfs filesystem label /mnt NAME` |
| Change UUID | `tune2fs -U random /dev/sdb1` | `xfs_admin -U generate /dev/sdb1` | `btrfstune -u /dev/sdb1` |
| Check / repair (**unmounted**) | `e2fsck -f /dev/sdb1` | `xfs_repair /dev/sdb1` | `btrfs check` (read-only by default; read the manual before `--repair`) |
| Grow | `resize2fs` | `xfs_growfs` | `btrfs filesystem resize` |
| Shrink | `resize2fs` (offline) | Not possible | `btrfs filesystem resize` |
| Integrity scan | — | — | `btrfs scrub start /mnt` |

```console
# tune2fs -m 1 /dev/sdb1             # reserved blocks 1% instead of 5% (large data disks)
# e2fsck -f /dev/sdb1                # force a full check (unmounted)
# xfs_repair -n /dev/sdb1            # dry run: report problems only
```

### Btrfs highlights

```console
# btrfs subvolume create /mnt/data/@projects
# btrfs subvolume list /mnt/data
# btrfs subvolume snapshot -r /mnt/data/@projects /mnt/data/@projects-2025-03-03
# btrfs filesystem usage /mnt/data
# btrfs scrub start /mnt/data
```

Mount option for compression: `compress=zstd`.

### Discard (TRIM) on SSDs

Tell an SSD which blocks are free:

```console
# fstrim -av                          # trim all mounted filesystems that support it
# systemctl enable --now fstrim.timer # weekly automatic trim
```

This is better than the `discard` mount option for most servers, which trims continuously and can slow writes.

### Other useful commands

| Command | Purpose |
|---------|---------|
| `mount -t TYPE` | Mount with a specific type |
| `cat /proc/filesystems` | Filesystems the kernel supports |
| `findmnt -t ext4,xfs,btrfs` | Mounted filesystems of these types |
| `ls /sbin/mkfs.*` | Available formatters |

## Common pitfalls

- **Choosing XFS and then needing to shrink.** XFS cannot shrink. Plan sizes, or use ext4 or LVM with a different layout.
- **Running `fsck` on a mounted filesystem.** It can corrupt data. Unmount first or use a rescue environment.
- **Using `xfs_repair` options from the internet.** `-L` zeroes the log and can lose recent changes. Use only after a failed normal repair and with a backup.
- **Running out of inodes** on ext4 with many small files. Set inode density at creation: `mkfs.ext4 -i 8192` or `-N`.
- **Duplicate UUIDs** after cloning. Mounting two filesystems with the same UUID fails (XFS) or is confusing. Generate new UUIDs.
- **Reserved blocks** on large ext4 data volumes waste space (5% by default). Reduce with `tune2fs -m`.
- **Btrfs full disk behavior.** "No space left" can appear with free space in data chunks. Use `btrfs filesystem usage` and `btrfs balance` carefully.
- **Using `discard` and `fstrim` together** is unnecessary.

## Further reading

- [`ext4` kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/ext4.html)
- [XFS documentation](https://xfs.wiki.kernel.org)
- [Btrfs documentation](https://btrfs.readthedocs.io)
- [`mkfs.ext4(8)`](https://man7.org/linux/man-pages/man8/mke2fs.8.html)
- [`mkfs.xfs(8)`](https://man7.org/linux/man-pages/man8/mkfs.xfs.8.html)
