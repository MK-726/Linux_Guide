# NFS and SMB/CIFS

## Overview

Network file sharing lets many hosts use the same files.

- **NFS** (Network File System) is the native Unix/Linux protocol. Use it between Linux and Unix systems.
- **SMB/CIFS** is the Windows file-sharing protocol. On Linux, **Samba** serves SMB, and the `cifs` client mounts SMB shares. Use it for Windows clients and mixed environments.

Both have security and performance details that matter. Plan permissions and network access carefully.

## Key concepts

| | **NFS** | **SMB (Samba)** |
|---|---------|-----------------|
| Typical clients | Linux, Unix, macOS | Windows, macOS, Linux |
| Port | **2049/tcp** (NFSv4). NFSv3 also uses rpcbind (111) and extra ports | **445/tcp** (also legacy 139/tcp) |
| Authentication | By host and UID/GID (`sec=sys`), or **Kerberos** (`sec=krb5`, `krb5i`, `krb5p`) | User name and password (local Samba users or Active Directory) |
| Permissions model | POSIX (UID/GID, mode bits, NFSv4 ACLs) | Windows ACLs mapped onto POSIX |
| Encryption | Kerberos `krb5p`, or a VPN/TLS (newer kernels support TLS with `xprtsec`) | SMB 3 encryption (`server smb encrypt`) |
| Use cases | Shared home directories, application data, VM storage | Office file shares, Windows profiles |

---

## NFS

### Server setup

**Debian-based:** `apt install nfs-kernel-server`
**RHEL-based:** `dnf install nfs-utils`

Define exports in `/etc/exports`:

```
# /etc/exports
/srv/share        192.0.2.0/24(rw,sync,no_subtree_check)
/srv/readonly     192.0.2.0/24(ro,sync,no_subtree_check) 198.51.100.15(ro)
/srv/home         192.0.2.0/24(rw,sync,no_subtree_check,root_squash)
```

Format: `directory  client(options) client(options)`. **No space between the client and the opening parenthesis.** A space changes the meaning: `host (rw)` gives `rw` to everyone.

| Option | Meaning |
|--------|---------|
| `rw` / `ro` | Read-write / read-only |
| `sync` | Reply only after data is safely written (default, safe) |
| `async` | Reply before writing (faster, **risk of data loss** on a crash) |
| `root_squash` | Map the client's root to an unprivileged user (**default, keep it**) |
| `no_root_squash` | Client root keeps root rights on the share (dangerous) |
| `all_squash` | Map all users to `anonuid`/`anongid` |
| `anonuid=`, `anongid=` | IDs used for squashed users |
| `no_subtree_check` | Avoids problems when files are renamed (recommended) |
| `sec=sys` / `krb5` / `krb5i` / `krb5p` | Security flavor |
| `fsid=0` | Mark the NFSv4 pseudo-root (older style) |

```console
# exportfs -ra                           # re-read /etc/exports and apply
# exportfs -v                            # what is exported, with effective options
# systemctl enable --now nfs-server      # RHEL-based; Debian-based: nfs-kernel-server
# showmount -e localhost                 # exports as seen by clients (NFSv3 mount protocol)
```

Firewall for NFSv4: open **2049/tcp**.

```console
# firewall-cmd --permanent --add-service=nfs && firewall-cmd --reload      # RHEL-based
# ufw allow from 192.0.2.0/24 to any port 2049 proto tcp                   # Ubuntu
```

NFSv3 needs `rpcbind` and fixed ports for `mountd`, `statd`, and `lockd`, which makes firewalls harder. **Prefer NFSv4**.

### Client setup

**Debian-based:** `apt install nfs-common`
**RHEL-based:** `dnf install nfs-utils`

```console
# mkdir -p /mnt/share
# mount -t nfs4 192.0.2.30:/srv/share /mnt/share
# mount -t nfs4 -o ro,noexec,nosuid 192.0.2.30:/srv/readonly /mnt/ro
$ df -hT /mnt/share
$ nfsstat -m                              # mounted NFS options as used by the kernel
$ mount | grep nfs
```

Permanent mount in `/etc/fstab`:

```
192.0.2.30:/srv/share  /mnt/share  nfs4  _netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=600,noatime  0  0
```

| Mount option | Meaning |
|--------------|---------|
| `_netdev` | Needs the network (mount after it is up) |
| `nofail` | Do not block boot if the server is down |
| `x-systemd.automount` | Mount on first access |
| `hard` (default) | Retry forever when the server is unreachable (**safe for data**, but processes can hang) |
| `soft` | Give up after retries and return errors (**can corrupt or lose writes**; avoid for writable data) |
| `timeo=`, `retrans=` | Timeout and retries |
| `vers=4.2` | NFS version |
| `rsize=`, `wsize=` | I/O sizes (defaults are normally good) |
| `ro`, `noexec`, `nosuid`, `nodev` | Hardening options |

### UID/GID mapping

With `sec=sys`, NFS sends **numeric UIDs and GIDs**. The same name must have the **same number** on server and clients, or files appear with wrong owners and permissions. Solutions: centralized identity (LDAP, FreeIPA, SSSD), or consistent UIDs. NFSv4 can map names to IDs (`idmapd`), but numeric consistency is still the simplest.

### Troubleshooting NFS

| Symptom | Check |
|---------|-------|
| `mount.nfs: access denied by server` | `exportfs -v`: client IP not in the export; typo in `/etc/exports`; need `exportfs -ra` |
| `mount.nfs: Connection timed out` | Firewall (2049), server down, wrong IP, `nfs-server` not running |
| `No such file or directory` on mount | The path is not exported (NFSv4 root/`fsid` layout), or the server directory does not exist |
| `Permission denied` after mounting | UID/GID mismatch, `root_squash`, directory permissions, SELinux on server (`nfs_export_all_rw` boolean) |
| `Stale file handle` | The export or file was removed/recreated on the server. Unmount and mount again |
| Processes stuck in `D` state | Server unreachable with `hard` mounts. Fix the network/server; as a last resort `umount -f -l` |
| Slow performance | Network, `sync` + slow disk, small `rsize`/`wsize`, many small files. Check `nfsstat`, `iostat` |
| Works on server, SELinux denies | `getsebool -a` (look for the nfs booleans), `ausearch -m avc` |

### Security notes for NFS

- Export only to **specific hosts or subnets**, never to `*` on untrusted networks.
- Keep `root_squash`. Avoid `no_root_squash`.
- `sec=sys` **trusts the client** to report the right user. Use Kerberos (`krb5p`) on untrusted networks, or place NFS on a private, firewalled network.
- Mount shares with `nosuid,nodev,noexec` where possible.

---

## SMB / CIFS (Samba)

### Server setup

**Debian-based:** `apt install samba`
**RHEL-based:** `dnf install samba samba-client`

`/etc/samba/smb.conf`:

```ini
[global]
    workgroup = WORKGROUP
    server string = File Server
    security = user
    map to guest = Never
    server min protocol = SMB3
    server smb encrypt = desired
    log file = /var/log/samba/log.%m
    max log size = 1000

[projects]
    path = /srv/samba/projects
    browseable = yes
    read only = no
    valid users = @projects
    create mask = 0660
    directory mask = 0770
    force group = projects
```

```console
# groupadd projects
# mkdir -p /srv/samba/projects
# chown root:projects /srv/samba/projects && chmod 2770 /srv/samba/projects
# useradd -M -s /usr/sbin/nologin -G projects bob          # Linux account without shell
# smbpasswd -a bob                                          # set Samba password (separate from the Linux password)
# testparm                                                  # validate smb.conf and show effective settings
# systemctl enable --now smb nmb                            # RHEL-based; Debian-based: smbd (and nmbd if needed)
# firewall-cmd --permanent --add-service=samba && firewall-cmd --reload     # RHEL-based
# ufw allow from 192.0.2.0/24 to any port 445 proto tcp                     # Ubuntu
```

| Option | Meaning |
|--------|---------|
| `path` | Directory to share |
| `valid users`, `write list`, `read list` | Who may access, write, or read |
| `read only` / `writable` | Access mode |
| `guest ok` | Allow anonymous access (**avoid**) |
| `create mask`, `directory mask`, `force group`, `force user` | Permissions for new files |
| `server min protocol = SMB3` | Disable old, insecure SMB1/SMB2 versions |
| `hosts allow` | Restrict by client address |
| `security = user` / `ads` | Local users / Active Directory member |

Samba can also join an **Active Directory** domain (`net ads join`, with `winbind` or `sssd`) so domain users get access. That is a larger project; see the Samba and Red Hat documentation.

### Test from Linux

```console
$ smbclient -L //192.0.2.30 -U bob                 # list shares
$ smbclient //192.0.2.30/projects -U bob           # interactive session (ls, get, put)
# smbstatus                                        # connections and open files (on the server)
```

### Client: mount a share

**Debian-based:** `apt install cifs-utils`
**RHEL-based:** `dnf install cifs-utils`

```console
# mkdir -p /mnt/projects
# mount -t cifs //192.0.2.30/projects /mnt/projects -o username=bob,uid=1000,gid=1000,vers=3.1.1
Password for bob@//192.0.2.30/projects:
```

Permanent mount with a **credentials file** (never put the password in `/etc/fstab`):

```console
# cat > /root/.smbcred <<'EOF'
username=bob
password=S3cret!Pass
domain=WORKGROUP
EOF
# chmod 600 /root/.smbcred
```

```
//192.0.2.30/projects  /mnt/projects  cifs  credentials=/root/.smbcred,uid=1000,gid=1000,file_mode=0660,dir_mode=0770,vers=3.1.1,_netdev,nofail,x-systemd.automount  0  0
```

| Mount option | Meaning |
|--------------|---------|
| `credentials=FILE` | Read user name and password from a protected file |
| `uid=`, `gid=` | Local owner shown for all files |
| `file_mode=`, `dir_mode=` | Permissions shown locally |
| `vers=3.1.1` | SMB protocol version |
| `sec=krb5` | Kerberos authentication |
| `seal` | Encrypt traffic (SMB 3) |
| `nofail`, `_netdev` | Boot behavior |

### Troubleshooting SMB

| Symptom | Check |
|---------|-------|
| `mount error(13): Permission denied` | Wrong password; user not created with `smbpasswd -a`; `valid users`; wrong `domain` |
| `mount error(112/115)` / host is down | Firewall (445), server not running, wrong address |
| `mount error: cifs filesystem not supported` | `cifs-utils` not installed or the `cifs` module missing |
| `mount error(95): Operation not supported` | Protocol version mismatch (`vers=`), SMB1 disabled |
| Can connect but cannot write | Share `read only`, Linux directory permissions, `force group`, SELinux (`samba_export_all_rw`, `samba_enable_home_dirs` booleans) |
| Files show wrong owner | Use `uid=`/`gid=` mount options, or configure idmap |
| Users are authenticated but cannot see the share | `browseable`, `valid users`, path permissions |
| Check what Samba does | `tail -f /var/log/samba/log.*`, `smbstatus`, `testparm -s` |

**SELinux** (RHEL-based): `setsebool -P samba_export_all_rw 1` and label the directory `samba_share_t` (`semanage fcontext -a -t samba_share_t '/srv/samba(/.*)?'; restorecon -Rv /srv/samba`).

## Common pitfalls

- **NFS exports with a space** before the options, exposing the share to the world.
- **Forgetting `exportfs -ra`** after editing `/etc/exports`.
- **UID/GID mismatch** between hosts (NFS), causing wrong owners and access errors.
- **`soft` NFS mounts for writable data** risk silent data corruption. Use `hard`.
- **Hanging processes** when an NFS server is down. Use `x-systemd.automount`, timeouts, monitoring, and redundant servers.
- **Using `async` exports** on important data.
- **Samba passwords separate from Linux passwords.** Users need `smbpasswd -a`.
- **SMB1 enabled** (insecure). Require SMB3.
- **Passwords in `/etc/fstab`** or on command lines. Use a credentials file with mode 600.
- **Exposing NFS or SMB to the internet.** Never. Use VPN and private networks.
- **Permissions designed for Windows and Linux separately** produce conflicts. Decide on one model per share.
- **SELinux denials** treated as network errors.
- **Locking and databases on NFS/SMB** can cause corruption or poor performance. Check the application's guidance.
- **Backups of mounted shares** may skip files or follow mounts unexpectedly. Back up the source filesystem.

## Further reading

- [`exports(5)`](https://man7.org/linux/man-pages/man5/exports.5.html)
- [`nfs(5)`](https://man7.org/linux/man-pages/man5/nfs.5.html)
- [Samba documentation](https://www.samba.org/samba/docs/)
- [`smb.conf(5)`](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)
- [`mount.cifs(8)`](https://man7.org/linux/man-pages/man8/mount.cifs.8.html)
