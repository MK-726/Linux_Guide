# File Types, Inodes, Hard Links, and Symbolic Links

## Overview

In Linux, "everything is a file", but there are different **kinds** of files. Each file is stored as an **inode** (its data and metadata), and a name in a directory points to that inode. Links let several names refer to the same data (hard links) or to another path (symbolic links).

## Key concepts

### File types

The first character in `ls -l` shows the type:

| Symbol | Type | Example |
|--------|------|---------|
| `-` | Regular file | Text, programs, images |
| `d` | Directory | `/etc` |
| `l` | Symbolic link | `/bin -> usr/bin` |
| `b` | Block device | `/dev/sda` |
| `c` | Character device | `/dev/tty`, `/dev/null` |
| `s` | Socket | `/run/systemd/private` |
| `p` | Named pipe (FIFO) | Created with `mkfifo` |

```console
$ ls -l /etc/hostname /etc /dev/sda /dev/null
-rw-r--r--  1 root root    7 Mar  3 09:00 /etc/hostname
brw-rw----  1 root disk 8, 0 Mar  3 09:00 /dev/sda
crw-rw-rw-  1 root root 1, 3 Mar  3 09:00 /dev/null
```

Linux does not use file extensions to decide the type. Use `file` to see the real content type.

### Inodes

An **inode** is a data structure that stores information about a file:

- Type, permissions, owner, group
- Size
- Timestamps (access, modification, change)
- Number of hard links
- Pointers to the data blocks on disk

An inode does **not** store the file name. The name is stored in the **directory entry**, which maps a name to an inode number.

```
Directory entry          Inode                     Data blocks
"report.txt" ──► inode 524301 ──► permissions, owner,
                                   size, times ──► [data]
```

```console
$ ls -i notes.txt
524301 notes.txt

$ stat notes.txt
  File: notes.txt
  Size: 42          Blocks: 8          IO Block: 4096   regular file
Device: 8,1         Inode: 524301      Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/   alice)   Gid: ( 1000/   alice)
Access: 2025-03-03 10:10:00.000000000 +0000
Modify: 2025-03-03 10:10:00.000000000 +0000
Change: 2025-03-03 10:10:00.000000000 +0000
```

| Timestamp | Changes when |
|-----------|-------------|
| **atime** | The file is read (often updated lazily, see `relatime`) |
| **mtime** | The file **content** changes |
| **ctime** | The inode (metadata) changes: permissions, owner, link count |

A filesystem can run out of **inodes** even if it has free space. Many tiny files cause this. Check with `df -i`.

```console
$ df -i /
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda1      3276800 215432 3061368    7% /
```

### Hard links

A **hard link** is another name for the same inode.

- Both names are equal. There is no "original".
- The data is deleted only when the **link count** reaches 0 and no process has the file open.
- A hard link cannot span different filesystems.
- You cannot hard-link directories.

```console
$ echo "data" > a.txt
$ ln a.txt b.txt
$ ls -li a.txt b.txt
524310 -rw-r--r-- 2 alice alice 5 Mar  3 10:20 a.txt
524310 -rw-r--r-- 2 alice alice 5 Mar  3 10:20 b.txt
$ rm a.txt
$ cat b.txt
data
```

Same inode number (524310) and link count 2.

### Symbolic (soft) links

A **symbolic link** is a small file that contains the **path** to another file or directory.

- It can point to directories and cross filesystems.
- It can be **relative** or **absolute**.
- If the target is removed, the link **breaks** (a "dangling" link).

```console
$ ln -s /etc/hostname myhost
$ ls -l myhost
lrwxrwxrwx 1 alice alice 13 Mar  3 10:25 myhost -> /etc/hostname
$ ln -s ../shared/config.yml config.yml     # relative link
```

### Comparison

| Feature | Hard link | Symbolic link |
|---------|-----------|---------------|
| Points to | An inode | A path |
| Works across filesystems | No | Yes |
| Can link directories | No | Yes |
| Survives target deletion | Yes (data stays) | No (breaks) |
| Own inode | No (shares) | Yes |

## Commands

| Command | Purpose |
|---------|---------|
| `ln TARGET LINKNAME` | Create a hard link |
| `ln -s TARGET LINKNAME` | Create a symbolic link |
| `ln -sf TARGET LINKNAME` | Replace an existing link |
| `readlink LINK` | Show where a link points |
| `readlink -f LINK` | Show the final resolved path |
| `ls -l` | Show links (`->`) |
| `find . -xtype l` | Find broken symbolic links |
| `find / -samefile FILE` | Find all names of the same inode |
| `unlink LINK` or `rm LINK` | Remove a link |
| `stat FILE` | Show inode information |
| `df -i` | Show inode usage |

## Common pitfalls

- **Relative symlink target is relative to the link's location**, not to your current directory. Test with `ls -l` and `readlink -f`.
- **Deleting a file with other hard links does not free the space.** The data remains until all names are gone.
- **Editors that "save by rename"** may break a hard link: they create a new file and replace the old one.
- **`rm -rf link/`** with a trailing slash can follow a symlink to a directory. Remove symlinks without a trailing slash.
- **Out of inodes** shows as "No space left on device" while `df -h` still shows free space. Use `df -i`.

## Further reading

- [`inode(7)`](https://man7.org/linux/man-pages/man7/inode.7.html)
- [`ln(1)`](https://man7.org/linux/man-pages/man1/ln.1.html)
- [`symlink(7)`](https://man7.org/linux/man-pages/man7/symlink.7.html)
