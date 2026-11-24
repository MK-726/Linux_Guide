# Process Lifecycle and States

## Overview

A **process** is a running instance of a program. The kernel tracks every process with an ID and a state. Knowing how processes are created, how they relate to each other, and what their states mean is the base for all troubleshooting.

## Key concepts

### Program vs. process

- A **program** is a file on disk (`/usr/bin/nginx`).
- A **process** is that program running in memory, with its own PID, memory, open files, and environment.
- One program can run as many processes.
- A **thread** is a unit of execution inside a process. Threads of a process share memory.

### PID and PPID

| ID | Meaning |
|----|---------|
| **PID** | Process ID: unique number of a process |
| **PPID** | Parent process ID: the process that created it |
| **PGID** | Process group ID (for job control and signals) |
| **SID** | Session ID (a login session) |

`PID 1` is the first process (systemd). Every other process descends from it.

```console
$ pstree -p -s $$
systemd(1)───sshd(812)───sshd(2100)───sshd(2160)───bash(2161)───pstree(3012)
```

### How processes are created

```
parent (bash)
   │ fork()  →  creates a copy (child) with a new PID
   ▼
child (bash copy)
   │ exec()  →  replaces itself with the new program (ls)
   ▼
child runs "ls", then exits
   │
   ▼
parent calls wait() and collects the exit code
```

Modern programs use `clone()`, which is the system call behind both `fork` and thread creation.

### Process states

Shown in the `STAT` column of `ps` and the `S` column of `top`:

| Code | State | Meaning |
|------|-------|---------|
| `R` | Running / runnable | Using the CPU or waiting for it |
| `S` | Interruptible sleep | Waiting for an event (most processes) |
| `D` | **Uninterruptible sleep** | Waiting for I/O (disk, NFS). **Cannot be killed** until the I/O finishes |
| `T` | Stopped | Paused by a signal (Ctrl+Z, SIGSTOP) or by a debugger (`t`) |
| `Z` | **Zombie** | Finished, but the parent has not read its exit code |
| `I` | Idle kernel thread | Kernel thread with nothing to do |

Extra letters in `ps`:

| Letter | Meaning |
|--------|---------|
| `s` | Session leader |
| `l` | Multi-threaded |
| `+` | In the foreground process group |
| `<` | High priority |
| `N` | Low priority (nice) |

```console
$ ps -eo pid,ppid,stat,comm | head -5
  PID  PPID STAT COMMAND
    1     0 Ss   systemd
    2     0 S    kthreadd
  812     1 Ss   sshd
 2161  2160 Ss+  bash
```

### Special process types

| Type | Description |
|------|-------------|
| **Daemon** | Background service with no terminal (for example `sshd`) |
| **Zombie** | Exited, but still has a process table entry because the parent has not called `wait()`. It uses no CPU or memory. Many zombies point to a buggy parent |
| **Orphan** | A child whose parent exited. It is adopted by PID 1 (or a subreaper) |
| **Kernel thread** | Created by the kernel, shown in brackets: `[kworker/0:1]`, `[kthreadd]` |

### Exit codes

A process ends with an exit code. 0 means success. Codes above 128 mean the process was killed by signal (`128 + signal number`). See [Exit Codes and Chaining](../../part-1-foundations/02-command-line-and-shell/06-exit-codes-and-chaining.md).

## Commands

| Command | Purpose |
|---------|---------|
| `ps -o pid,ppid,stat,cmd -p PID` | State and parent of one process |
| `pstree -p` | Process tree with PIDs |
| `ps -eLf` | Show threads |
| `cat /proc/PID/status` | Detailed process status |
| `ps -eo stat,pid,ppid,cmd \| awk '$1 ~ /Z/'` | Find zombies |
| `echo $$` | PID of the current shell |

```console
$ ps -eo stat,pid,ppid,comm | awk '$1 ~ /^Z/'
Z     4120  4001 worker <defunct>
```

## Common pitfalls

- **Trying to kill a zombie.** It is already dead. Fix or restart its **parent** (PPID 4001 in the example). Orphaned zombies are cleaned up by PID 1.
- **Trying to kill a `D` state process.** It waits for hardware or network storage. Solve the I/O problem (NFS server, failing disk).
- **Confusing PID reuse.** PIDs are recycled. A saved PID may refer to a different process later. Check the command name before you kill it.
- **Assuming a process uses its full virtual size.** See [Memory and CPU Utilization](06-memory-and-cpu-utilization.md).

## Further reading

- [`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html)
- [`execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html)
- [`ps(1)`: process state codes](https://man7.org/linux/man-pages/man1/ps.1.html)
- [`credentials(7)`](https://man7.org/linux/man-pages/man7/credentials.7.html)
