# File Comparison and Inspection

## Overview

Administrators often need to answer: *Are these two files the same? What changed? What is this file? Did it get corrupted?* These tools help you compare and inspect files without opening them in an editor.

## Commands

### Compare files

| Command | Purpose |
|---------|---------|
| `diff A B` | Show line-by-line differences |
| `diff -u A B` | Unified format (used by patches and Git) |
| `diff -r DIR1 DIR2` | Compare directories recursively |
| `diff -q DIR1 DIR2 -r` | Only list which files differ |
| `cmp A B` | Find the first byte that differs (works for binary files) |
| `comm A B` | Compare two **sorted** files in columns |
| `sdiff A B` | Side-by-side comparison |
| `vimdiff A B` | Visual comparison in Vim |

```console
$ diff -u sshd_config.orig sshd_config
--- sshd_config.orig   2025-02-20 09:00:00.000000000 +0000
+++ sshd_config        2025-03-03 10:30:00.000000000 +0000
@@ -32,7 +32,7 @@
 #LoginGraceTime 2m
-PermitRootLogin yes
+PermitRootLogin prohibit-password
 #StrictModes yes
```

In unified format, lines starting with `-` were removed and lines starting with `+` were added.

> [!TIP]
> Before editing an important configuration file, copy it: `cp -a file file.orig`. Afterwards `diff -u file.orig file` shows exactly what you changed.

### Inspect files

| Command | Purpose |
|---------|---------|
| `file FILE` | Detect the file type from its content |
| `stat FILE` | Size, permissions, owner, inode, timestamps |
| `ls -lh` | Size and permissions |
| `du -sh PATH` | Disk space used by a file or directory |
| `wc -l FILE` | Number of lines |
| `head`, `tail`, `less` | View parts of a file |
| `strings FILE` | Show readable text in a binary |
| `xxd FILE \| head` or `hexdump -C FILE \| head` | Show bytes in hexadecimal |
| `lsattr`, `chattr` | Show and set file attributes (ext filesystems) |

```console
$ file /bin/ls /etc/passwd photo.jpg archive.tar.gz
/bin/ls:        ELF 64-bit LSB pie executable, x86-64, dynamically linked
/etc/passwd:    ASCII text
photo.jpg:      JPEG image data, JFIF standard 1.01
archive.tar.gz: gzip compressed data, from Unix

$ du -sh /var/log
412M    /var/log

$ du -h --max-depth=1 /var | sort -h
```

### Checksums

A **checksum** (hash) is a short fingerprint of a file's content. If a single byte changes, the hash changes. Use hashes to verify downloads and detect changes.

| Command | Algorithm | Use today? |
|---------|-----------|-----------|
| `sha256sum` | SHA-256 | Yes (recommended) |
| `sha512sum` | SHA-512 | Yes |
| `md5sum` | MD5 | Only for non-security integrity checks |

```console
$ sha256sum ubuntu.iso
3c7d...e91f  ubuntu.iso

$ sha256sum ubuntu.iso > ubuntu.iso.sha256     # create a checksum file
$ sha256sum -c ubuntu.iso.sha256               # verify
ubuntu.iso: OK
```

> [!IMPORTANT]
> A checksum proves the file was not corrupted. It does not prove the file is **trusted** unless the checksum itself comes from a trusted source, ideally a signed file.

### Quick "are these the same?" checks

```console
$ cmp -s a.bin b.bin && echo same || echo different
$ diff -rq /etc /backup/etc
$ sha256sum a.bin b.bin
```

## Common pitfalls

- **Comparing by size or date only.** Files with the same size can differ. Use `cmp` or a checksum.
- **`diff` on binary files.** It reports only "Binary files differ". Use `cmp` or `xxd`.
- **`du` vs. `df` mismatch.** `df` counts what the filesystem reports used (including deleted files still held open by processes). `du` adds up files you can see. See [File Descriptors and Open Files](07-file-descriptors-and-open-files.md).
- **Trusting `file` blindly.** It guesses from content and can be wrong on unusual data.

## Further reading

- [GNU diffutils manual](https://www.gnu.org/software/diffutils/manual/diffutils.html)
- [`sha256sum(1)`](https://man7.org/linux/man-pages/man1/sha256sum.1.html)
- [`file(1)`](https://man7.org/linux/man-pages/man1/file.1.html)
