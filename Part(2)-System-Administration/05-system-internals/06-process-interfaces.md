# Process Interfaces and File Descriptors

## Overview

For every running process, the kernel creates a directory `/proc/<PID>/`. It lets you see what the process is doing without special tools: its command line, environment, memory, open files, limits, and more. Tools such as `ps`, `top`, and `lsof` read these same files.

## Key concepts

### Contents of /proc/PID

| Path | Content |
|------|---------|
| `cmdline` | The full command line (arguments separated by null characters) |
| `exe` | Link to the executable file |
| `cwd` | Link to the current working directory |
| `root` | Link to the process's root directory (differs in a chroot or container) |
| `environ` | Environment variables (readable by the owner and root) |
| `fd/` | One link per open file descriptor |
| `fdinfo/` | Position and flags for each descriptor |
| `status` | Readable summary: state, UID, memory, threads, capabilities |
| `stat` | Raw status used by `ps` |
| `limits` | Resource limits (open files, processes, memory) |
| `maps`, `smaps` | Memory mappings of the process |
| `io` | Bytes read and written |
| `mountinfo` | Mounts visible to the process |
| `ns/` | Namespaces the process belongs to |
| `cgroup` | Control group of the process |
| `task/` | One directory per thread |

`$$` is the PID of your shell, and `self` always points to the process reading it: `/proc/self`.

## Commands

```console
$ sleep 600 &
[1] 4821

$ ls -l /proc/4821/exe /proc/4821/cwd
lrwxrwxrwx 1 alice alice 0 Mar  3 12:00 /proc/4821/cwd -> /home/alice
lrwxrwxrwx 1 alice alice 0 Mar  3 12:00 /proc/4821/exe -> /usr/bin/sleep

$ tr '\0' ' ' < /proc/4821/cmdline; echo
sleep 600

$ grep -E 'Name|State|Pid|PPid|Uid|VmRSS|Threads' /proc/4821/status
Name:   sleep
State:  S (sleeping)
Pid:    4821
PPid:   1930
Uid:    1000    1000    1000    1000
VmRSS:      1024 kB
Threads:        1

$ grep "open files" /proc/4821/limits
Max open files            1024                 524288               files

$ cat /proc/4821/io
rchar: 2123
wchar: 0
```

### Open file descriptors

```console
$ ls -l /proc/4821/fd
lrwx------ 1 alice alice 64 Mar  3 12:00 0 -> /dev/pts/0
lrwx------ 1 alice alice 64 Mar  3 12:00 1 -> /dev/pts/0
lrwx------ 1 alice alice 64 Mar  3 12:00 2 -> /dev/pts/0
```

For the basics of file descriptors and `lsof`, see [File Descriptors and Open Files](../../part-1-foundations/03-filesystem-and-file-management/07-file-descriptors-and-open-files.md).

### Find which process owns a file, port, or directory

```console
# ls -l /proc/*/cwd 2>/dev/null | grep /var/www
# ss -tulpn | grep :443            # listening sockets and their processes
# readlink /proc/$(pgrep -o nginx)/exe
```

### Useful system-wide files

| File | Content |
|------|---------|
| `/proc/loadavg` | Load and the last PID created |
| `/proc/sys/kernel/pid_max` | Highest PID value |
| `/proc/sys/kernel/threads-max` | Maximum number of threads |
| `/proc/sys/fs/file-nr` | Open file handle counts |

## Security notes

- `/proc/PID/environ` can contain **secrets** (passwords in environment variables). Only the owner and root can read it, but any process of the same user can.
- Mount `/proc` with `hidepid=2` to hide other users' processes from non-root users (use with care; it can affect some tools):

```
proc  /proc  proc  defaults,hidepid=2  0 0
```

## Common pitfalls

- **Process disappears while you read.** `/proc/PID` exists only while the process runs. Scripts must handle "No such file or directory".
- **Reading `cmdline` shows arguments joined.** They are separated by null bytes. Use `tr '\0' ' '`.
- **Confusing threads and processes.** Threads appear under `task/` and have thread IDs. Tools like `ps -L` show them.
- **Deleted executables.** If `exe` shows `(deleted)`, the program was replaced while running. Restart the service to use the new version.

## Further reading

- [`proc_pid_status(5)`](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html)
- [`proc(5)`](https://man7.org/linux/man-pages/man5/proc.5.html)
