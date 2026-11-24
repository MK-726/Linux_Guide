# Archives and Compression: tar, gzip, xz, zip

## Overview

- An **archive** combines many files and directories into one file.
- **Compression** makes data smaller.

On Linux these are separate steps. `tar` creates archives. `gzip`, `bzip2`, `xz`, and `zstd` compress. `tar` can call a compressor for you, producing files like `backup.tar.gz`.

## Key concepts

| Tool | Extension | Speed | Compression | Notes |
|------|-----------|-------|-------------|-------|
| `gzip` | `.gz` | Fast | Good | Most common and widely supported |
| `bzip2` | `.bz2` | Slower | Better than gzip | Less used today |
| `xz` | `.xz` | Slow | Best of these | Good for distribution and long-term storage |
| `zstd` | `.zst` | Very fast | Very good | Modern; adjustable levels |
| `zip` | `.zip` | Medium | Good | Archive **and** compression in one; common for exchange with Windows |

`tar` stores permissions, owners, timestamps, and symbolic links. `zip` stores fewer Linux-specific details. Use `tar` for Linux backups.

## Commands

### tar

Syntax: `tar OPERATION[OPTIONS] -f ARCHIVE [FILES]`

| Option | Meaning |
|--------|---------|
| `-c` | **C**reate an archive |
| `-x` | E**x**tract |
| `-t` | **T**est or list contents |
| `-f FILE` | Archive file name (put it last in a group of options) |
| `-v` | **V**erbose: show files |
| `-z` | Use gzip |
| `-J` | Use xz |
| `-j` | Use bzip2 |
| `--zstd` | Use zstd |
| `-C DIR` | Change to DIR before working |
| `-p` | Preserve permissions (default for root when extracting) |
| `--exclude=PATTERN` | Skip files |

```console
$ tar -czvf backup.tar.gz /home/alice/projects          # create, gzip
$ tar -cJf backup.tar.xz /home/alice/projects           # create, xz
$ tar -tzf backup.tar.gz | head                         # list contents
$ tar -xzf backup.tar.gz                                # extract here
$ tar -xzf backup.tar.gz -C /tmp/restore                # extract elsewhere
$ tar -xzf backup.tar.gz home/alice/projects/notes.txt  # extract one file
$ tar -czf logs.tar.gz --exclude='*.tmp' /var/log/app
```

Modern `tar` detects the compression when extracting, so `tar -xf archive.tar.xz` works for any of them.

> [!NOTE]
> `tar` removes the leading `/` from absolute paths when creating an archive, so extraction cannot overwrite system files by accident. Always **list the contents** (`-t`) of an archive from an unknown source before extracting it.

### gzip, xz, and bzip2

These replace the original file by default.

```console
$ gzip access.log            # creates access.log.gz, removes access.log
$ gzip -k access.log         # keep the original
$ gunzip access.log.gz       # decompress
$ zcat access.log.gz | grep ERROR     # read without extracting
$ xz -k -9 bigfile           # best compression, keep original
$ unxz bigfile.xz
$ zstd -T0 data.img          # use all CPU threads
```

Helper tools that work on compressed files: `zcat`, `zless`, `zgrep`, `xzcat`.

### zip and unzip

```console
$ zip -r project.zip project/        # create (recursive)
$ unzip -l project.zip               # list
$ unzip project.zip -d /tmp/out      # extract to a directory
```

If missing, install the `zip` and `unzip` packages.

### Compare sizes

```console
$ ls -lh backup.tar*
-rw-r--r-- 1 alice alice 120M Mar  3 11:00 backup.tar.gz
-rw-r--r-- 1 alice alice  98M Mar  3 11:02 backup.tar.xz
```

### Copy a directory tree over SSH with tar

```console
$ tar -cf - /data | ssh user@host "tar -xf - -C /backup"
```

## Common pitfalls

- **Wrong option order.** `tar -cvf archive.tar.gz files` does not compress. Add `-z` or use a modern `tar` with `-a`, which picks the compression from the extension: `tar -caf backup.tar.gz files`.
- **Putting the archive name wrong.** In `tar -cf files archive.tar`, the first name after `-f` is the archive. Writing it in the wrong place **overwrites** your files. Always check the command before pressing Enter.
- **"Tar bomb".** Archives that extract many files into the current directory. List first with `tar -tf`, or extract into a new directory with `-C`.
- **Running out of space** while creating an archive. Check with `df -h` and estimate the size with `du -sh`.
- **Compressing already-compressed data** (images, video, archives) gains little.
- **Backups that skip special details.** Use `tar --xattrs --acls --selinux` when you need extended attributes, ACLs, and SELinux contexts preserved.

## Further reading

- [GNU tar manual](https://www.gnu.org/software/tar/manual/tar.html)
- [`gzip(1)`](https://man7.org/linux/man-pages/man1/gzip.1.html)
- [XZ Utils](https://tukaani.org/xz/)
- [Zstandard](https://facebook.github.io/zstd/)
