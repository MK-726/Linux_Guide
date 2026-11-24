# Mounting, /etc/fstab, UUIDs, and Labels

## Overview

A filesystem becomes available only after it is **mounted** at a directory (the **mount point**). `/etc/fstab` lists the filesystems the system mounts automatically at boot. Mistakes here are a leading cause of boot problems, so the rules matter.

## Key concepts

- The mount point is an existing directory. Anything in it is **hidden** while something is mounted on it.
- A filesystem can be mounted by device name, **UUID**, or **label**.
- At boot, systemd generates `.mount` units from `/etc/fstab`.

### Identify by UUID or label

```console
$ lsblk -f /dev/sdb1
NAME FSTYPE FSVER LABEL UUID                                 MOUNTPOINTS
sdb1 ext4   1.0   data  5b0c3e9a-3d6a-4c5b-8a77-3f2f6e0a9d11

$ blkid /dev/sdb1
$ ls -l /dev/disk/by-uuid/
```

| Identifier | Use |
|------------|-----|
| **UUID** | Unique and stable. **Best for `/etc/fstab`** |
| **Label** | Human-friendly. Must be unique on the system |
| **Device name** (`/dev/sdb1`) | Can change. Avoid in `fstab` |
| **PARTUUID** | Identifies the partition, not the filesystem |

## Commands

### Mount and unmount (root)

```console
# mkdir -p /data
# mount /dev/sdb1 /data
# mount UUID=5b0c3e9a-3d6a-4c5b-8a77-3f2f6e0a9d11 /data
# mount -o ro,noexec /dev/sdb1 /data              # with options
# mount -o remount,rw /data                       # change options of a mounted filesystem
# mount --bind /srv/www /var/www                  # bind mount: show a directory at another path
# umount /data                                    # or: umount /dev/sdb1
```

| Command | Purpose |
|---------|---------|
| `mount` | Show all mounts |
| `findmnt` | Tree view of mounts |
| `findmnt /data` | Information about one mount point |
| `findmnt --verify` | Check `/etc/fstab` for errors |
| `mount -a` | Mount everything in `/etc/fstab` that is not mounted (test your changes) |
| `df -h /data` | Free space of the filesystem containing a path |

### Common mount options

| Option | Meaning |
|--------|---------|
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` |
| `ro` / `rw` | Read-only / read-write |
| `noexec` | Do not allow running programs from this filesystem |
| `nosuid` | Ignore SUID/SGID bits |
| `nodev` | Ignore device files |
| `noatime` | Do not update access times (less write load) |
| `relatime` | Default: update access time only if older than modification time |
| `nofail` | Do not fail boot if the device is missing |
| `x-systemd.device-timeout=10s` | How long to wait for the device |
| `_netdev` | Network filesystem (wait for the network) |
| `x-systemd.automount` | Mount on first access |
| `user`, `users` | Allow normal users to mount |
| `errors=remount-ro` | (ext4) Remount read-only on errors |
| `compress=zstd` | (Btrfs) Compression |
| `subvol=@data` | (Btrfs) Mount a subvolume |

Security hardening: use `nodev,nosuid,noexec` on `/tmp`, `/var/tmp`, and data volumes where possible.

### /etc/fstab

One line per filesystem, six fields separated by spaces or tabs:

```
# <device>                                  <mount point> <type> <options>        <dump> <pass>
UUID=3f1c9a1e-8d5e-4a3b-9f11-0d8d6d5b1c22   /             ext4   errors=remount-ro  0      1
UUID=7A2B-1C3D                              /boot/efi     vfat   umask=0077         0      1
UUID=5b0c3e9a-3d6a-4c5b-8a77-3f2f6e0a9d11   /data         ext4   defaults,nofail    0      2
LABEL=backup                                /backup       xfs    defaults,nofail    0      0
/dev/mapper/vg0-swap                        none          swap   sw                 0      0
nas01:/export/share                         /mnt/share    nfs    _netdev,nofail     0      0
tmpfs                                       /tmp          tmpfs  nodev,nosuid,size=2G 0    0
```

| Field | Meaning |
|-------|---------|
| 1. Device | `UUID=...`, `LABEL=...`, `/dev/...`, or a network path |
| 2. Mount point | Directory (`none` for swap) |
| 3. Type | `ext4`, `xfs`, `btrfs`, `vfat`, `nfs`, `swap`, `auto` |
| 4. Options | Comma-separated mount options |
| 5. Dump | Backup flag for the old `dump` tool. Use `0` |
| 6. Pass | `fsck` order at boot: `1` for the root filesystem, `2` for others, `0` = no check (use `0` for XFS, Btrfs, network and swap) |

### Safe workflow to add a filesystem

```console
# blkid /dev/sdb1                          # get the UUID
# mkdir -p /data
# cp /etc/fstab /etc/fstab.bak             # backup
# vi /etc/fstab                            # add the line
# findmnt --verify                         # check syntax
# systemctl daemon-reload                  # make systemd re-read fstab
# mount -a                                 # test; no errors = good
# findmnt /data
```

**Never reboot before `mount -a` succeeds.** A broken fstab leads to emergency mode (see [Boot Troubleshooting](../06-boot-grub-systemd/07-boot-troubleshooting-and-recovery.md)).

### Unmounting a busy filesystem

```console
# umount /data
umount: /data: target is busy.
# fuser -vm /data                          # who uses it?
# lsof +f -- /data
# systemctl stop myservice                 # stop the user, then:
# umount /data
```

`umount -l` (lazy) detaches the mount point now and cleans up later. It can hide problems and is a last resort. `umount -f` is mainly for unreachable NFS servers.

### Automount and systemd mount units

```
# fstab line, mounts on first access, unmounts after 5 minutes idle
nas01:/export/share /mnt/share nfs _netdev,x-systemd.automount,x-systemd.idle-timeout=300 0 0
```

```console
# systemctl list-units --type=mount
# systemctl status data.mount              # unit name = path with "/" as "-"
```

## Common pitfalls

- **Typos in `/etc/fstab`** that make boot fail. Test with `findmnt --verify` and `mount -a` before rebooting.
- **Missing `nofail`** for external, removable, or network disks. A missing disk stops boot.
- **Using device names** instead of UUIDs.
- **Mounting over a non-empty directory.** Existing files are hidden. They return after `umount`.
- **Forgetting `daemon-reload`.** systemd may use the old fstab generated units. Run `systemctl daemon-reload`.
- **Wrong `pass` for XFS.** XFS has no boot-time `fsck`. Use `0` (or `1` is harmless for root but unnecessary).
- **Network mounts without `_netdev`** can slow or break boot.
- **Mount point owned by root with wrong permissions** after mounting. The permissions come from the **root directory of the mounted filesystem**. Fix with `chown` and `chmod` on the mount point **while mounted**.
- **Same UUID on two disks** (clones). Change one.

## Further reading

- [`fstab(5)`](https://man7.org/linux/man-pages/man5/fstab.5.html)
- [`mount(8)`](https://man7.org/linux/man-pages/man8/mount.8.html)
- [`findmnt(8)`](https://man7.org/linux/man-pages/man8/findmnt.8.html)
- [`systemd.mount(5)`](https://man7.org/linux/man-pages/man5/systemd.mount.5.html)
