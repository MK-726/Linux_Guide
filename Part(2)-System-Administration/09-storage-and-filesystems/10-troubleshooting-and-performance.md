# Filesystem Troubleshooting and Storage Performance Basics

## Overview

Most storage emergencies are one of a few types: the disk is **full**, the filesystem became **read-only**, a mount **fails**, or the system is **slow** because of I/O. This page gives a systematic approach to each. Chapter 25 goes deeper into performance analysis.

## Commands and scenarios

### Scenario 1: "No space left on device"

Work through these checks in order.

**1. Is space really used? Or inodes?**

```console
$ df -h                          # block usage per filesystem
$ df -i                          # inode usage per filesystem
Filesystem      Inodes   IUsed IFree IUse% Mounted on
/dev/sda1      3276800 3276800     0  100% /
```

Inode use at 100% means too many small files, even with free space.

**2. Find what is large**

```console
# du -xh --max-depth=1 / 2>/dev/null | sort -h | tail
# du -xh --max-depth=1 /var 2>/dev/null | sort -h | tail
# ncdu -x /                       # interactive (package ncdu)
# find / -xdev -type f -size +500M -exec ls -lh {} + 2>/dev/null
# find /var -xdev -type f -mtime -1 -size +50M            # recent big files (a growing log?)
```

`-x` / `-xdev` stays on one filesystem so network and other mounts are not counted.

**3. Common space users**

| Location | Cause | Action |
|----------|-------|--------|
| `/var/log`, `journalctl --disk-usage` | Logs not rotated | Fix `logrotate`, `journalctl --vacuum-size=200M` |
| `/var/cache`, package caches | Old downloads | `apt clean`, `dnf clean all` |
| `/var/lib/docker`, `/var/lib/containers` | Images, layers, volumes | `docker system df`, `docker system prune` (read first) |
| `/tmp`, `/var/tmp` | Leftover files | Clean, set `systemd-tmpfiles` rules |
| Old kernels in `/boot` | Updates | `apt autoremove --purge`, `dnf remove --oldinstallonly` |
| Core dumps (`/var/lib/systemd/coredump`) | Crashes | Delete, limit size |
| Databases, backups, user data | Normal growth | Capacity planning |

**4. Deleted files still held open**

`du` shows less than `df`. A process still has a deleted file open.

```console
# lsof +L1                                   # files with link count 0 (deleted but open)
# lsof -nP | grep '(deleted)' | sort -k7 -n -r | head
```

Restart the process holding the file. See [File Descriptors and Open Files](../../part-1-foundations/03-filesystem-and-file-management/07-file-descriptors-and-open-files.md).

**5. Reserved blocks (ext4)**

By default ext4 reserves 5% for root. On a big data disk this is a lot: `tune2fs -m 1 /dev/sdb1`.

**6. Snapshots and Btrfs**

Old snapshots hold deleted data. `btrfs filesystem usage /`, `snapper list`, `lvs` (check `Data%`).

### Scenario 2: a filesystem became read-only

Linux remounts a filesystem read-only when it detects errors (`errors=remount-ro`).

```console
$ touch /data/test
touch: cannot touch '/data/test': Read-only file system
$ dmesg -T | grep -i -E 'error|remount|I/O|read-only' | tail
EXT4-fs error (device sdb1): ext4_find_entry:1455: inode #2: comm ls: reading directory lblock 0
EXT4-fs (sdb1): Remounting filesystem read-only
blk_update_request: I/O error, dev sdb, sector 123456
```

Steps:

1. Read `dmesg` for the **cause**. Hardware I/O errors point to a failing disk, cable, or storage path. Check SMART ([Disk Health](09-disk-health-and-smart.md)).
2. **Protect the data**: copy or back up anything important now.
3. Stop services that use the filesystem, unmount, and repair.
4. Remount read-write only after repair and a clean result.

> [!WARNING]
> If hardware is failing, running `fsck` may cause more damage. Copy data first (`ddrescue`), then repair the copy.

### Scenario 3: repair a filesystem

Always **unmount** first (or boot rescue mode).

```console
# umount /data
# e2fsck -f /dev/sdb1                    # ext4: full check, ask on problems
# e2fsck -f -n /dev/sdb1                 # dry run: report only
# e2fsck -f -y /dev/sdb1                 # automatically answer yes (after you have backups)
# xfs_repair -n /dev/sdb1                # XFS: check only
# xfs_repair /dev/sdb1                   # XFS: repair
# btrfs check /dev/sdb1                  # Btrfs: read-only check
```

| Situation | Action |
|-----------|--------|
| ext4 superblock damaged | `e2fsck -b 32768 /dev/sdb1` (use a backup superblock; list with `mke2fs -n /dev/sdb1`) |
| XFS reports "log needs replaying" | Mount it once and unmount to replay, then `xfs_repair` |
| `xfs_repair` refuses because of a dirty log | `xfs_repair -L` **discards the log** and can lose recent changes. Last resort, after backup |
| Files in `lost+found` | Recovered fragments after `fsck`. Inspect or delete |
| Root filesystem | Boot to rescue media, or run `fsck` from the initramfs (`touch /forcefsck` is obsolete on systemd; use the `fsck.mode=force` kernel parameter) |

### Scenario 4: mount problems

| Message | Likely cause and action |
|---------|--------------------------|
| `wrong fs type, bad option, bad superblock` | Wrong type, damaged filesystem, missing driver. Check `blkid`, `dmesg`, try `mount -t TYPE` |
| `special device ... does not exist` | Wrong device name or UUID. Check `lsblk -f` and `blkid` |
| `mount point does not exist` | `mkdir -p` the directory |
| `target is busy` (on umount) | Open files or current directory. `fuser -vm /mnt`, `lsof +f -- /mnt` |
| `can't find in /etc/fstab` | Add an entry or mount with the full command |
| `Structure needs cleaning` (XFS) | Run `xfs_repair` |
| `Mounting read-only` | Errors, or the device is write-protected |
| XFS: `Filesystem has duplicate UUID` | `mount -o nouuid` (temporary), then `xfs_admin -U generate` |
| NFS: `access denied` | Export or firewall (Chapter 13) |

```console
$ findmnt --verify
$ journalctl -b -u 'systemd-fsck@*' -u '*.mount'
# mount -a -v
```

### Scenario 5: the system is slow because of storage

**Find out if it is I/O.**

```console
$ top                    # high 'wa' (I/O wait)
$ vmstat 1 5             # 'b' blocked processes, 'wa', 'bi/bo'
$ iostat -xz 1 5         # per-device statistics (package sysstat)
```

```console
$ iostat -xz 1 2
Device   r/s   w/s  rkB/s   wkB/s  r_await w_await aqu-sz %util
sda    120.0 450.0 4800.0 36000.0     2.1    28.5   12.4  98.7
```

| Column | Meaning |
|--------|---------|
| `r/s`, `w/s` | Reads and writes per second (IOPS) |
| `rkB/s`, `wkB/s` | Throughput |
| `r_await`, `w_await` | Average time (ms) a request takes, including queue time. **Latency** |
| `aqu-sz` | Average queue length |
| `%util` | Time the device was busy. Near 100% on a single HDD = saturated. On SSD/NVMe and arrays, 100% does not always mean saturated |

Rough guide: HDD latency above about 20–30 ms and SSD above a few ms under normal load deserve investigation.

**Find who generates the I/O.**

```console
# iotop -oPa                 # processes doing I/O (package iotop)
$ pidstat -d 1 5             # per-process disk I/O
$ lsof +D /var/lib/mysql | head
```

**Quick checks and tests**

```console
$ cat /sys/block/sda/queue/scheduler           # I/O scheduler: [none] mq-deadline kyber bfq
$ cat /sys/block/sda/queue/rotational          # 1 = HDD, 0 = SSD
$ mount | grep -E 'noatime|relatime'
# fio --name=test --filename=/data/fio.test --size=1G --rw=randrw --bs=4k --iodepth=16 --runtime=30 --time_based --direct=1
# rm /data/fio.test
```

> [!NOTE]
> Test on a non-production system or off-peak. `dd` is a poor benchmark because caching skews results. `fio` with `--direct=1` is better. Never run write tests against raw devices that hold data.

### Common performance improvements

| Problem | Options |
|---------|---------|
| Heavy metadata writes | Mount with `noatime` (or keep `relatime`) |
| Small random I/O on HDD | Move to SSD, add RAID 10, add RAM for cache |
| Writes stall during backups | `nice`/`ionice`, snapshots, schedule off-peak |
| Many small files in one directory | Spread over subdirectories, ext4 `dir_index` (default), XFS handles well |
| SSD slow after long use | `fstrim` weekly |
| Wrong I/O scheduler | NVMe: `none`. SATA SSD: `none` or `mq-deadline`. HDD: `mq-deadline` or `bfq` |
| Cloud disk limits | Check provisioned IOPS/throughput. Throttling shows as high `await` with low `%util` |
| Database and journaling filesystem tuning | Follow the database vendor's filesystem and mount guidance |

## Common pitfalls

- **Deleting random files to free space** without finding the cause. The problem returns.
- **Running `fsck` on a mounted filesystem.**
- **Trusting `%util`** alone. Look at latency (`await`) and queue size.
- **Forgetting inodes.** "No space left" with free blocks.
- **Repairing a failing disk** instead of copying it first.
- **Using `-L` or `--force` options** of repair tools casually.
- **Forgetting that cloud volumes have IOPS and throughput limits**, shared across all processes on the volume.
- **Changing the I/O scheduler or mount options on production** without testing.
- **Fixing the symptom.** Log growth needs `logrotate` or fixing noisy logging, not just deleting logs.

## Further reading

- [`e2fsck(8)`](https://man7.org/linux/man-pages/man8/e2fsck.8.html)
- [`xfs_repair(8)`](https://man7.org/linux/man-pages/man8/xfs_repair.8.html)
- [`iostat(1)`](https://man7.org/linux/man-pages/man1/iostat.1.html)
- [`fio`](https://fio.readthedocs.io)
- [Brendan Gregg: Linux disk I/O](https://www.brendangregg.com/linuxperf.html)
