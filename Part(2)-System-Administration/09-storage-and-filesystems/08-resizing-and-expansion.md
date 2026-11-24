# Disk Expansion and Filesystem Resizing

## Overview

Space runs out. In virtual and cloud environments you can often **grow a disk** without downtime. The key is to work in the right **order**, layer by layer, from the bottom (disk) to the top (filesystem).

> [!WARNING]
> Resizing is safe when done correctly, but a mistake can destroy data. Take a **snapshot or backup** first. **Growing is easy. Shrinking is dangerous.**

## Key concepts

### The layers

Growing goes **bottom up**:

```
1. Disk (virtual disk / cloud volume)       ← grow in the hypervisor or cloud console
2. Partition                                ← growpart / parted resizepart
3. (RAID)                                   ← mdadm --grow
4. LVM physical volume                      ← pvresize
5. LVM logical volume                       ← lvextend
6. Filesystem                               ← resize2fs / xfs_growfs / btrfs resize
```

Skip the layers you do not use. **Shrinking** goes the opposite way: filesystem first, then the volume, then the partition.

### What each filesystem allows

| Filesystem | Grow | Shrink |
|------------|------|--------|
| ext4 | Online (mounted) | **Offline only** (unmounted) |
| XFS | Online | **Not supported** |
| Btrfs | Online | Online |

## Commands

### Step 0: check the current state

```console
$ lsblk -f
$ df -hT /data
$ lsblk /dev/sdb             # does the kernel see the new size?
```

### Step 1: make the kernel see the larger disk

After you grow a virtual disk or cloud volume:

```console
# echo 1 > /sys/class/block/sdb/device/rescan        # SCSI/SATA/virtio-scsi
# partprobe /dev/sdb
$ lsblk /dev/sdb                                      # SIZE should show the new size
```

NVMe disks usually update automatically. A reboot always works as a fallback.

### Step 2: grow the partition (if the disk has partitions)

`growpart` (package `cloud-guest-utils` on Debian-based, `cloud-utils-growpart` on RHEL-based) is the simplest tool:

```console
# growpart /dev/sda 3                  # grow partition 3 of /dev/sda to fill the disk
CHANGED: partition=3 start=2099200 old: size=... new: size=...
```

Or with `parted`:

```console
# parted /dev/sda resizepart 3 100%
```

Online partition growth works on mounted partitions on modern kernels, as long as you only change the **end** and the start stays the same.

### Step 3a: filesystem directly on the partition

```console
# resize2fs /dev/sda3             # ext4: grow to fill the partition (online)
# xfs_growfs /                    # XFS: pass the MOUNT POINT
# btrfs filesystem resize max /   # Btrfs
```

### Step 3b: with LVM

```console
# pvresize /dev/sda3                                   # tell LVM the PV is larger
# vgs                                                  # VFree shows new space
# lvextend -r -l +100%FREE /dev/vg0/root               # grow LV and filesystem together
```

Or without `-r`, grow the filesystem afterwards (`resize2fs` or `xfs_growfs`).

### Complete example: cloud VM with root on LVM

```console
# lsblk
sda               8:0    0   100G  0 disk                    # was 50G
├─sda1            8:1    0   512M  0 part /boot/efi
└─sda2            8:2    0  49.5G  0 part
  └─vg0-root    253:0    0  49.5G  0 lvm  /
# growpart /dev/sda 2
# pvresize /dev/sda2
# lvextend -r -l +100%FREE /dev/vg0/root
# df -h /
```

### Add a second disk to an LVM volume instead

```console
# pvcreate /dev/sdc
# vgextend vg0 /dev/sdc
# lvextend -r -L +100G /dev/vg0/data
```

### Grow a RAID array

After replacing members with larger disks, one by one:

```console
# mdadm --grow /dev/md0 --size=max
# pvresize /dev/md0           # or resize2fs / xfs_growfs on the array directly
```

### Shrink an ext4 filesystem on LVM (offline)

```console
# umount /srv/data
# e2fsck -f /dev/vg0/data
# lvreduce -r -L 20G /dev/vg0/data       # shrinks filesystem to 20G first, then the LV
# mount /srv/data
```

Do **not** shrink the LV below the filesystem size. Using `lvreduce -r` (which calls `resize2fs` first) prevents this mistake. Without `-r`, shrink the filesystem to a **smaller** size, shrink the LV to the target, then grow the filesystem to fill it.

### Shrink XFS (alternative)

XFS cannot shrink. Instead:

1. Create a new smaller volume and filesystem.
2. Copy the data (`rsync -aHAX /old/ /new/` or `xfsdump | xfsrestore`).
3. Switch the mount, test, and remove the old volume.

## Common pitfalls

- **Wrong order.** Growing the filesystem before the partition or volume does nothing. Shrinking the volume before the filesystem destroys data.
- **Using the device name with `xfs_growfs`.** It needs the **mount point**.
- **The kernel does not see the new disk size.** Rescan or reboot. In clouds, confirm the volume modification finished (some need time).
- **Free space exists in the VG but `df` shows no change.** The LV or the filesystem was not extended.
- **Partition cannot grow because another partition is after it.** Move or delete the later partition, or use LVM and add a new disk.
- **Resizing mounted ext4 filesystems with `resize2fs` smaller** fails. Unmount first.
- **No backup before shrinking.** Always back up.
- **Cloud providers' limits.** Some volumes can be modified once per few hours. Check provider rules.
- **Swap or `/boot` on the same disk.** Partitions after the one you need to grow block it. Plan layouts so the growing partition is **last**.

## Further reading

- [`resize2fs(8)`](https://man7.org/linux/man-pages/man8/resize2fs.8.html)
- [`xfs_growfs(8)`](https://man7.org/linux/man-pages/man8/xfs_growfs.8.html)
- [`growpart` (cloud-utils)](https://github.com/canonical/cloud-utils)
- [`lvextend(8)`](https://man7.org/linux/man-pages/man8/lvextend.8.html)
