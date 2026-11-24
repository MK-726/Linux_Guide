# File Descriptors and Open Files

## Overview

When a program opens a file, the kernel gives it a small number called a **file descriptor** (FD). The program uses this number for reading and writing. File descriptors also represent sockets, pipes, and devices. Knowing how they work helps you debug "too many open files", disk space that will not free, and ports in use.

## Key concepts

### What a file descriptor is

- Each **process** has its own table of file descriptors.
- Numbers start at 0 and go up.
- The first three are standard streams:

| FD | Name | Purpose |
|----|------|---------|
| 0 | stdin | Standard input |
| 1 | stdout | Standard output |
| 2 | stderr | Standard error |
| 3+ | Other | Files, sockets, pipes, devices the process opened |

(See [Streams, Redirection, and Pipelines](../02-command-line-and-shell/05-streams-redirection-pipelines.md) for redirection.)

### Where to see them

Every process has a directory `/proc/PID/fd` with one entry per open descriptor.

```console
$ ls -l /proc/$$/fd           # $$ is the PID of the current shell
lrwx------ 1 alice alice 64 Mar  3 11:10 0 -> /dev/pts/0
lrwx------ 1 alice alice 64 Mar  3 11:10 1 -> /dev/pts/0
lrwx------ 1 alice alice 64 Mar  3 11:10 2 -> /dev/pts/0
```

### Deleted but still open files

If a process has a file open and you delete the file, the **name** disappears but the **data stays** on disk until the process closes it. This is a common reason why `df` shows a full disk while `du` finds nothing large.

### Limits

Each process, and the whole system, has a maximum number of open file descriptors.

| Limit | Where | Command |
|-------|-------|---------|
| Per-process (soft / hard) | `ulimit`, `/etc/security/limits.conf`, systemd `LimitNOFILE=` | `ulimit -n` (soft), `ulimit -Hn` (hard) |
| System-wide | Kernel setting `fs.file-max` | `cat /proc/sys/fs/file-max` |

```console
$ ulimit -n
1024
$ cat /proc/sys/fs/file-max
9223372036854775807
$ cat /proc/sys/fs/file-nr          # allocated, unused, maximum
2368    0       9223372036854775807
```

## Commands

### lsof: list open files

`lsof` is not always installed. Install `lsof` first (**requires root**).

| Command | Purpose |
|---------|---------|
| `lsof` | All open files (large output) |
| `lsof -p PID` | Files opened by one process |
| `lsof -u USER` | Files opened by a user |
| `lsof FILE` | Which processes have this file open |
| `lsof +D /dir` | Open files under a directory |
| `lsof -i` | Network connections |
| `lsof -i :80` | What uses port 80 |
| `lsof -i TCP -sTCP:LISTEN` | Listening TCP ports |
| `lsof +L1` | Files that are **deleted** but still open |

```console
# lsof -i :22
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd     812 root    3u  IPv4  18321      0t0  TCP *:ssh (LISTEN)

# lsof +L1
COMMAND  PID USER   FD   TYPE DEVICE   SIZE/OFF NLINK   NODE NAME
java    2210 app    12w   REG  253,0 8589934592     0 131074 /var/log/app/big.log (deleted)
```

The `(deleted)` line shows 8 GB held by a deleted log.

### Other tools

| Command | Purpose |
|---------|---------|
| `fuser -v /path` | Show which processes use a file or mount point |
| `fuser -vm /mnt/data` | Processes using a mounted filesystem (useful before unmounting) |
| `ss -tulpn` | Listening sockets with the owning process |
| `ls -l /proc/PID/fd` | Open descriptors without `lsof` |
| `ls /proc/PID/fd \| wc -l` | Count a process's descriptors |

### Free space held by a deleted file

Best fix: restart or reload the service so it closes the file, then fix log rotation.

If a restart is not possible, you can empty the file through `/proc`:

```console
# : > /proc/2210/fd/12
```

> [!WARNING]
> Truncating a file through `/proc` changes data held by a running process. Do this only when you understand the application, and prefer a normal service restart.

### Typical fixes for "Too many open files"

1. Find the process: `lsof -p PID | wc -l`
2. Check its limit: `cat /proc/PID/limits | grep "open files"`
3. Raise it for a systemd service:

```ini
# /etc/systemd/system/myapp.service.d/override.conf
[Service]
LimitNOFILE=65536
```

```console
# systemctl daemon-reload
# systemctl restart myapp
```

Then find out whether the application leaks descriptors.

## Common pitfalls

- **`/etc/security/limits.conf` has no effect on systemd services.** Set `LimitNOFILE=` in the unit instead.
- **Cannot unmount a filesystem: "target is busy".** A process has files open or its current directory is there. Use `fuser -vm /mountpoint` or `lsof +f -- /mountpoint`.
- **Space not freed after deleting logs.** Check `lsof +L1`.
- **Running `lsof` as a normal user** shows only your own processes.

## Further reading

- [`lsof(8)`](https://man7.org/linux/man-pages/man8/lsof.8.html)
- [`proc(5)`](https://man7.org/linux/man-pages/man5/proc.5.html)
- [`open(2)`](https://man7.org/linux/man-pages/man2/open.2.html)
