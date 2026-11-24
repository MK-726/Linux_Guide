# Block Devices, Device Naming, and Disk Inspection

## Overview

Disks, partitions, LVM volumes, and RAID arrays all appear as **block devices**: devices that read and write data in fixed-size blocks. Before you change anything in storage, you must know exactly **which device is which**.

## Key concepts

### Device naming

| Name pattern | Device |
|--------------|--------|
| `/dev/sda`, `/dev/sdb` | SATA, SAS, and USB disks (SCSI layer). Partitions: `sda1`, `sda2` |
| `/dev/vda`, `/dev/vdb` | Virtio disks in KVM virtual machines |
| `/dev/xvda` | Xen virtual disks |
| `/dev/nvme0n1` | NVMe disk 0, namespace 1. Partitions: `nvme0n1p1`, `nvme0n1p2` |
| `/dev/mmcblk0` | SD/eMMC cards. Partitions: `mmcblk0p1` |
| `/dev/sr0` | Optical drive |
| `/dev/loop0` | Loop device (a file used as a disk) |
| `/dev/md0` | Software RAID array |
| `/dev/dm-0`, `/dev/mapper/NAME` | Device mapper: LVM volumes, encrypted volumes |
| `/dev/vgname/lvname` | LVM logical volume (link) |

> [!IMPORTANT]
> `/dev/sdX` letters are assigned in detection order and **can change after a reboot** or when you add disks. Use **UUIDs, labels, or `/dev/disk/by-id/`** in `/etc/fstab` and scripts.

### Stable names

```console
$ ls -l /dev/disk/by-id/ | head -4
lrwxrwxrwx 1 root root  9 Mar  3 09:00 ata-Samsung_SSD_870_S5Y... -> ../../sda
lrwxrwxrwx 1 root root 10 Mar  3 09:00 ata-Samsung_SSD_870_S5Y...-part1 -> ../../sda1
$ ls /dev/disk/
by-diskseq  by-id  by-label  by-partlabel  by-partuuid  by-path  by-uuid
```

| Directory | Based on |
|-----------|----------|
| `by-id` | Model and serial number (stable for the physical disk) |
| `by-uuid` | Filesystem UUID |
| `by-label` | Filesystem label |
| `by-partuuid` | Partition UUID (stored in the GPT) |
| `by-path` | Hardware location (controller port, PCI address) |

### Sector size

Disks use **512-byte** or **4096-byte** (4K) sectors. Partitions should be **aligned** (start at multiples of 1 MiB). Modern tools do this by default.

## Commands

### lsblk: the tree of devices

```console
$ lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   50G  0 disk
├─sda1          8:1    0  512M  0 part /boot/efi
├─sda2          8:2    0    1G  0 part /boot
└─sda3          8:3    0 48.5G  0 part
  ├─vg0-root  253:0    0   40G  0 lvm  /
  └─vg0-swap  253:1    0  8.5G  0 lvm  [SWAP]
sdb             8:16   0  200G  0 disk
```

| Command | Purpose |
|---------|---------|
| `lsblk -f` | Add filesystem type, label, UUID, mount point |
| `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,ROTA,MODEL` | Choose columns |
| `lsblk -d` | Only disks, no partitions |
| `lsblk -e7` | Hide loop devices |
| `lsblk -b` | Sizes in bytes |
| `lsblk -t` | Topology (sector sizes, I/O limits) |
| `lsblk -J` | JSON output for scripts |

The `ROTA` column: `1` = spinning disk (HDD), `0` = SSD or NVMe.

### More tools

| Command | Purpose |
|---------|---------|
| `blkid` | UUID, label, and filesystem type (**root** sees all devices) |
| `fdisk -l` | Partition tables of all disks (**root**) |
| `fdisk -l /dev/sdb` | One disk |
| `parted -l` | Partition tables (shows GPT/MBR) |
| `df -hT` | Mounted filesystems: size, used, type |
| `findmnt` | Mount tree |
| `findmnt -S UUID=...` | Where a filesystem is mounted |
| `cat /proc/partitions` | Kernel's list of block devices |
| `udevadm info /dev/sdb` | udev's information about a device |
| `smartctl -i /dev/sdb` | Model, serial, firmware (`smartmontools`) |
| `lsscsi` | SCSI devices (if installed) |
| `nvme list` | NVMe devices (`nvme-cli`) |

```console
# blkid
/dev/sda2: UUID="3f1c9a1e-8d5e-4a3b-9f11-0d8d6d5b1c22" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="8b1a..."
/dev/sdb: UUID="a1b2..." TYPE="LVM2_member"

# fdisk -l /dev/sdb
Disk /dev/sdb: 200 GiB, 214748364800 bytes, 419430400 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
Disklabel type: gpt
```

### Identify a new disk safely

When you attach a new disk (or after a cloud volume attach):

```console
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL,SERIAL    # compare with expectations
$ dmesg | tail                                               # kernel detection message
# echo "- - -" > /sys/class/scsi_host/host0/scan             # ask the kernel to rescan (SCSI hot-add, root)
```

A **disk with no `FSTYPE` and no partitions** is probably new. A disk with a filesystem or partitions is **in use**. Check twice.

### Loop devices (practice without real disks)

```console
$ truncate -s 1G /tmp/disk.img
# losetup --find --show /tmp/disk.img
/dev/loop0
# losetup -d /dev/loop0              # detach
```

Use loop devices for safe practice of partitioning, formatting, LVM, and RAID.

## Common pitfalls

- **Working on the wrong device.** Confirm with `lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINT` and compare size and serial.
- **Using `/dev/sdX` in `/etc/fstab`.** Names can shift. Use UUID or label.
- **Confusing the disk and its partitions.** `/dev/sdb` is the disk, `/dev/sdb1` is a partition. NVMe uses the `p` suffix: `nvme0n1p1`.
- **Cached `blkid` output.** After changes run `blkid -p /dev/sdb1` (probe directly) or `udevadm settle`.
- **Virtual disks.** In clouds and VMs, the "disk" size may be changed by the platform. See [Disk Expansion](08-resizing-and-expansion.md).

## Further reading

- [`lsblk(8)`](https://man7.org/linux/man-pages/man8/lsblk.8.html)
- [`blkid(8)`](https://man7.org/linux/man-pages/man8/blkid.8.html)
- [Kernel: devices and names](https://www.kernel.org/doc/html/latest/admin-guide/devices.html)
