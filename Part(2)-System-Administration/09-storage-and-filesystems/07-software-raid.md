# RAID Concepts and Linux Software RAID

## Overview

**RAID** (Redundant Array of Independent Disks) combines several disks into one device to gain **redundancy** (survive a disk failure), **performance**, or both. On Linux, the kernel's **MD** driver and the **`mdadm`** tool create software RAID without a hardware controller.

> [!IMPORTANT]
> **RAID is not a backup.** It protects against disk failure only. It does not protect against deletion, corruption, ransomware, or fire. You still need backups (Chapter 27).

## Key concepts

### RAID levels

| Level | Min. disks | Capacity | Survives | Notes |
|-------|-----------|----------|----------|-------|
| **RAID 0** (stripe) | 2 | All disks | **Nothing**: one failure loses everything | Speed only. No redundancy |
| **RAID 1** (mirror) | 2 | 1 disk | n−1 disks | Simple and robust. Good for boot and system disks |
| **RAID 5** (stripe + parity) | 3 | n−1 disks | 1 disk | Good capacity. Rebuilds are slow and risky on large disks |
| **RAID 6** (double parity) | 4 | n−2 disks | 2 disks | Safer for large disks |
| **RAID 10** (mirror + stripe) | 4 | n/2 disks | 1 per mirror pair (at least) | Best mix of speed and safety. Higher cost |

Capacity example with four 4 TB disks: RAID 0 = 16 TB, RAID 1 (two pairs, as 10) = 8 TB, RAID 5 = 12 TB, RAID 6 = 8 TB, RAID 10 = 8 TB.

### Software vs. hardware RAID

| | Software (MD) | Hardware controller |
|---|---------------|---------------------|
| Cost | Free | Controller and battery/cache |
| Portability | Array moves to any Linux system | Tied to the controller model |
| Monitoring | `mdadm`, standard tools | Vendor tools |
| Performance | Excellent on modern CPUs | Cache and offload |

### Other redundancy options

Btrfs and ZFS have integrated RAID. LVM can create RAID volumes (`lvcreate --type raid1`) using the same MD code. Storage controllers in "HBA/IT mode" are best for software RAID.

## Commands

### Create an array (destroys existing data on the members)

```console
# mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
mdadm: array /dev/md0 started.

# mdadm --create /dev/md1 --level=5 --raid-devices=4 /dev/sd[b-e]            # RAID 5
# mdadm --create /dev/md2 --level=6 --raid-devices=4 --spare-devices=1 /dev/sd[b-f]
```

Use whole disks or partitions (type "Linux RAID" on GPT). Then create a filesystem and mount:

```console
# mkfs.ext4 /dev/md0
# mount /dev/md0 /data
```

### Check status

```console
$ cat /proc/mdstat
Personalities : [raid1]
md0 : active raid1 sdc[1] sdb[0]
      209584128 blocks super 1.2 [2/2] [UU]
      bitmap: 0/2 pages [0KB], 65536KB chunk

# mdadm --detail /dev/md0
/dev/md0:
     Raid Level : raid1
     Array Size : 209584128 (199.88 GiB)
    Raid Devices : 2
   Total Devices : 2
          State : clean
 Active Devices : 2
Working Devices : 2
 Failed Devices : 0
```

Reading `[2/2] [UU]`: 2 of 2 members are active, `U` = up. A failed member shows `_`: `[U_]`.

During a rebuild:

```console
md0 : active raid1 sdc[2] sdb[0]
      [=>...................]  recovery =  8.4% (17600000/209584128) finish=22.1min speed=144000K/sec
```

### Make the array persistent

Save the array definition and rebuild the initramfs.

```console
# mdadm --detail --scan >> /etc/mdadm/mdadm.conf        # Debian-based
# mdadm --detail --scan >> /etc/mdadm.conf              # RHEL-based and SUSE-based
# update-initramfs -u                                   # Debian-based
# dracut -f                                             # RHEL-based
```

In `/etc/fstab`, use the filesystem **UUID** (`blkid /dev/md0`). Array names like `/dev/md127` can change.

### Replace a failed disk

```console
# mdadm --detail /dev/md0                         # find the failed device
# mdadm /dev/md0 --fail /dev/sdb --remove /dev/sdb    # mark as failed and remove (if not auto-removed)
# ...physically replace the disk...
# sgdisk -R=/dev/sdb /dev/sdc && sgdisk -G /dev/sdb   # copy the partition layout (if partitions are used)
# mdadm /dev/md0 --add /dev/sdb                   # add the new disk; rebuild starts
# watch cat /proc/mdstat
```

Check SMART data of the failed and the surviving disks. A rebuild stresses the remaining disks, and a second failure during rebuild loses the array (RAID 1/5).

### Other operations

| Command | Purpose |
|---------|---------|
| `mdadm --assemble --scan` | Assemble known arrays (after moving disks) |
| `mdadm --examine /dev/sdb` | Show RAID metadata on a member |
| `mdadm --stop /dev/md0` | Stop an array (unmount first) |
| `mdadm --zero-superblock /dev/sdb` | Erase RAID metadata from a disk (to reuse it) |
| `mdadm --grow /dev/md0 --raid-devices=3` | Reshape (add a disk to RAID 5) |
| `mdadm --grow /dev/md0 --size=max` | Use extra space after replacing members with larger disks |
| `mdadm --monitor --scan --mail=root` | Monitor and mail (usually run by a service) |

### Monitoring and scrubbing

```console
# systemctl status mdmonitor              # RHEL-based; Debian uses "mdadm" / "mdmonitor" service
# echo check > /sys/block/md0/md/sync_action        # start a data scrub (consistency check)
$ cat /sys/block/md0/md/mismatch_cnt                # inconsistencies found
```

Many distributions run a monthly scrub (`/usr/share/mdadm/checkarray` on Debian, `raid-check` on RHEL). Set `MAILADDR root` in `mdadm.conf` and make sure mail reaches someone.

## Common pitfalls

- **Treating RAID as backup.** A deleted file is deleted on all members.
- **Using RAID 0 for important data.** One disk failure loses everything.
- **No monitoring.** A degraded array can run for months until the second disk fails. Alert on `[U_]` and `mdmonitor` mail.
- **Not updating `mdadm.conf` and the initramfs.** Arrays may come up as `/dev/md127` or not at boot.
- **Disks of the same age and batch** fail around the same time. Consider staggering purchases or hot spares.
- **Using different-size disks.** The array uses the smallest size.
- **RAID 5 with very large disks.** Rebuild times are long and read errors during rebuild can lose the array. Prefer RAID 6 or RAID 10.
- **Wiping the wrong member.** `--zero-superblock` on a member of a live array destroys it.
- **Boot loader on one disk only** in a RAID 1 root. Install GRUB / ESP on both disks.
- **Write-intent bitmaps** speed up recovery after a crash. Keep them on (default for large arrays).

## Further reading

- [`mdadm(8)`](https://man7.org/linux/man-pages/man8/mdadm.8.html)
- [Linux RAID wiki](https://raid.wiki.kernel.org)
- [Kernel: md](https://www.kernel.org/doc/html/latest/admin-guide/md.html)
