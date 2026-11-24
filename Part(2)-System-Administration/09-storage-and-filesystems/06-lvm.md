# LVM: Physical Volumes, Volume Groups, and Logical Volumes

## Overview

**LVM** (Logical Volume Manager) adds a flexible layer between disks and filesystems. With LVM you can combine several disks into one pool, create volumes of any size, **grow them while mounted**, move data between disks, and take snapshots. It is the default layout on RHEL-based installations and common on servers.

## Key concepts

```
Disks / partitions        /dev/sdb   /dev/sdc            ← physical storage
        │                     │          │
        ▼                     ▼          ▼
Physical Volumes (PV)       PV         PV                ← pvcreate
        └──────────┬─────────┘
                   ▼
Volume Group (VG)        vg_data (a pool of space)      ← vgcreate
        ┌──────────┼───────────┐
        ▼          ▼           ▼
Logical Volumes   lv_db      lv_logs     free space      ← lvcreate
        │          │
        ▼          ▼
Filesystems      ext4/XFS  ext4/XFS                       ← mkfs, mount
```

| Term | Meaning |
|------|---------|
| **PV** (physical volume) | A disk or partition prepared for LVM |
| **VG** (volume group) | A pool made of one or more PVs |
| **LV** (logical volume) | A "virtual partition" carved from the VG. You put a filesystem on it |
| **PE** (physical extent) | Allocation unit of a VG (default 4 MiB) |
| **Snapshot** | A point-in-time copy of an LV (uses copy-on-write) |
| **Thin pool** | Space allocated on demand (thin provisioning) |

LVs appear as `/dev/VGNAME/LVNAME` and `/dev/mapper/VGNAME-LVNAME`.

## Commands

### Overview commands

| Level | Short list | Detailed |
|-------|-----------|----------|
| Physical volumes | `pvs` | `pvdisplay` |
| Volume groups | `vgs` | `vgdisplay` |
| Logical volumes | `lvs` | `lvdisplay` |
| Everything | `lsblk`, `lvs -a -o +devices` | |

```console
# pvs
  PV         VG      Fmt  Attr PSize    PFree
  /dev/sdb   vg_data lvm2 a--  <200.00g 100.00g
# vgs
  VG      #PV #LV #SN Attr   VSize    VFree
  vg_data   1   1   0 wz--n- <200.00g 100.00g
# lvs
  LV    VG      Attr       LSize
  lv_db vg_data -wi-ao---- 100.00g
```

### Create storage from scratch

```console
# pvcreate /dev/sdb /dev/sdc               # 1. prepare the disks (destroys existing data)
# vgcreate vg_data /dev/sdb /dev/sdc       # 2. create the pool
# lvcreate -L 100G -n lv_db vg_data        # 3. a volume of 100 GiB
# lvcreate -l 100%FREE -n lv_logs vg_data  #    a volume with all remaining space
# mkfs.xfs /dev/vg_data/lv_db              # 4. filesystem
# mkdir -p /srv/db
# mount /dev/vg_data/lv_db /srv/db         # 5. mount
```

Add to `/etc/fstab` using the **mapper path or UUID**:

```
/dev/mapper/vg_data-lv_db  /srv/db  xfs  defaults  0 0
```

| `lvcreate` option | Meaning |
|-------------------|---------|
| `-L 10G` | Size in bytes (K, M, G, T) |
| `-l 100%FREE` or `-l 50%VG` | Size in extents or percentage |
| `-n NAME` | Volume name |
| `-s` | Snapshot |
| `--type thin` / `-T` | Thin volume |

### Grow storage (online)

Add space to a VG from a new disk:

```console
# pvcreate /dev/sdd
# vgextend vg_data /dev/sdd
```

Extend an LV **and its filesystem** in one command:

```console
# lvextend -r -L +50G /dev/vg_data/lv_db        # add 50 GiB
# lvextend -r -l +100%FREE /dev/vg_data/lv_db   # use all free space
```

`-r` (`--resizefs`) resizes the filesystem (ext4, XFS) too. Without `-r`, grow the filesystem yourself:

```console
# resize2fs /dev/vg_data/lv_db                  # ext4
# xfs_growfs /srv/db                            # XFS (use the mount point)
```

### Shrink (ext4 only, offline)

```console
# umount /srv/db
# e2fsck -f /dev/vg_data/lv_db
# lvreduce -r -L 50G /dev/vg_data/lv_db         # -r shrinks the filesystem first
# mount /srv/db
```

> [!WARNING]
> Shrinking risks data loss if the order is wrong (filesystem first, then volume). Back up first. **XFS cannot be shrunk.** See [Disk Expansion and Filesystem Resizing](08-resizing-and-expansion.md).

### Move data off a disk and remove it

```console
# pvmove /dev/sdb                    # move all data from sdb to other PVs in the VG (online)
# vgreduce vg_data /dev/sdb          # remove from the VG
# pvremove /dev/sdb                  # remove LVM labels
```

### Snapshots

```console
# lvcreate -s -L 5G -n db_snap /dev/vg_data/lv_db      # classic snapshot with 5 GiB of change space
# mount -o ro,nouuid /dev/vg_data/db_snap /mnt/snap    # (nouuid needed for XFS)
# ...copy data / run backup...
# umount /mnt/snap
# lvremove /dev/vg_data/db_snap
```

> [!IMPORTANT]
> A classic snapshot is **not a backup**. It depends on the origin volume. If its change space fills (check with `lvs`, column `Data%`), the snapshot becomes invalid. Remove snapshots after use because they slow writes.

Merge (roll back to the snapshot): `lvconvert --merge /dev/vg_data/db_snap` (needs the origin to be unmounted or happens on next activation).

### Rename, remove, rescan

```console
# lvrename vg_data lv_db lv_database
# lvremove /dev/vg_data/lv_database              # destroys data
# vgremove vg_data
# vgscan; lvscan; pvscan                          # rescan
# vgchange -ay vg_data                            # activate all LVs in a VG
# vgchange -an vg_data                            # deactivate (unmount first)
```

### Thin provisioning (brief)

```console
# lvcreate -L 100G -T vg_data/thinpool                 # thin pool
# lvcreate -V 500G -T vg_data/thinpool -n lv_thin1     # a "500G" volume in a 100G pool
```

You can promise more space than exists. **Monitor pool usage** (`lvs`, column `Data%`). A full thin pool causes serious errors.

## Configuration and files

| Path | Purpose |
|------|---------|
| `/etc/lvm/lvm.conf` | LVM settings |
| `/etc/lvm/backup/`, `/etc/lvm/archive/` | Automatic metadata backups |
| `vgcfgbackup`, `vgcfgrestore` | Backup and restore VG metadata |
| `/dev/mapper/` | Device nodes |

## Common pitfalls

- **Wrong disk for `pvcreate`.** It overwrites the start of the device. Check with `lsblk -f`.
- **Forgetting the filesystem resize** after `lvextend`. Use `-r`, or run `resize2fs` / `xfs_growfs`.
- **Trying to shrink XFS.** Not supported. Copy the data to a new smaller volume instead.
- **VG with no free extents.** `lvextend` fails with "Insufficient free space". Add a PV (`vgextend`).
- **Losing one disk in a multi-disk VG** can lose all LVs spread over it. LVM is **not redundancy**. Use RAID underneath or `--type raid1` for LVs.
- **Snapshots left forever** fill up and slow writes.
- **Duplicate VG names** when attaching a disk from another system. Rename: `vgrename UUID newname`.
- **Boot problems after changing LVM** (root on LVM). Rebuild the initramfs (`dracut -f` / `update-initramfs -u`).
- **Thin pool full.** Monitor and extend the pool before it fills.

## Further reading

- [`lvm(8)`](https://man7.org/linux/man-pages/man8/lvm.8.html)
- [Red Hat: Configuring and managing LVM](https://docs.redhat.com)
- [Arch Wiki: LVM](https://wiki.archlinux.org/title/LVM) (distribution-neutral background)
