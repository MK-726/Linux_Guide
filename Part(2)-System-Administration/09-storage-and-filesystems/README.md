# Chapter 9: Storage, Disks, Partitions & Filesystems

| # | Topic | Summary |
|---|-------|---------|
| 1 | [Block Devices and Disk Inspection](01-block-devices-and-disk-inspection.md) | Device names, `lsblk`, `blkid`, `fdisk -l` |
| 2 | [Partitioning: GPT and MBR](02-partitioning-gpt-and-mbr.md) | Partition tables, `fdisk`, `gdisk`, `parted` |
| 3 | [Filesystems: ext4, XFS, Btrfs](03-filesystems.md) | Concepts, creation, maintenance tools |
| 4 | [Mounting, /etc/fstab, UUIDs, and Labels](04-mounting-fstab-uuids-labels.md) | Attach filesystems and make it permanent |
| 5 | [Swap Management](05-swap.md) | Swap partitions, swap files, zram |
| 6 | [LVM: PV, VG, and LV](06-lvm.md) | Flexible volumes |
| 7 | [Software RAID](07-software-raid.md) | RAID levels and `mdadm` |
| 8 | [Disk Expansion and Filesystem Resizing](08-resizing-and-expansion.md) | Grow and shrink safely |
| 9 | [Disk Health and SMART Monitoring](09-disk-health-and-smart.md) | `smartctl`, early warnings |
| 10 | [Filesystem Troubleshooting and Storage Performance](10-troubleshooting-and-performance.md) | Disk full, read-only, repair, `iostat` |

> [!WARNING]
> This chapter contains commands that **destroy data** (`mkfs`, `wipefs`, `fdisk`, `dd`, `pvcreate`). Practice in a virtual machine with spare virtual disks. Always check the device name with `lsblk` before you run a destructive command.

**Previous:** [Chapter 8](../08-processes-and-resources/README.md) | **Next:** [Chapter 10](../10-logging-time-and-scheduling/README.md)
