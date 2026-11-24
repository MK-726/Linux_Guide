# Signals and Process Termination

## Overview

A **signal** is a short message the kernel or another process sends to a process. Signals tell a process to stop, reload, pause, or continue. Most programs react in a defined way, and many can catch signals to shut down cleanly.

## Key concepts

### Important signals

| Number | Name | Default action | Typical use |
|--------|------|----------------|-------------|
| 1 | `SIGHUP` | Terminate | Terminal closed; many daemons **reload configuration** |
| 2 | `SIGINT` | Terminate | `Ctrl+C` |
| 3 | `SIGQUIT` | Core dump | `Ctrl+\`; JVM thread dump |
| 9 | `SIGKILL` | Kill | **Cannot be caught or ignored.** Last resort |
| 10 / 12 | `SIGUSR1` / `SIGUSR2` | Terminate | Application-defined (for example, reopen log files) |
| 11 | `SIGSEGV` | Core dump | Invalid memory access (a bug) |
| 15 | `SIGTERM` | Terminate | **Polite request to exit.** Default for `kill` |
| 17 | `SIGCHLD` | Ignore | Child process ended |
| 18 | `SIGCONT` | Continue | Resume a stopped process |
| 19 | `SIGSTOP` | Stop | Pause. **Cannot be caught** |
| 20 | `SIGTSTP` | Stop | `Ctrl+Z` |

Numbers can differ on other architectures. Use names in scripts. List all signals with `kill -l`.

### TERM first, KILL last

- `SIGTERM` lets the program clean up: flush data, close files, remove lock files, tell clients.
- `SIGKILL` ends the process immediately. It gets **no chance to clean up**: possible data loss, stale locks, and leftover child processes.

Recommended order:

1. `SIGTERM` (`kill PID`). Wait a few seconds.
2. If still running, `SIGKILL` (`kill -9 PID`).

## Commands

| Command | Purpose |
|---------|---------|
| `kill PID` | Send SIGTERM |
| `kill -SIGTERM PID` or `kill -15 PID` | Same |
| `kill -9 PID` / `kill -KILL PID` | SIGKILL |
| `kill -HUP PID` | Reload configuration (if supported) |
| `kill -STOP PID` / `kill -CONT PID` | Pause / resume |
| `kill -0 PID` | Test whether the process exists and you may signal it (no signal sent) |
| `kill -l` | List signal names |
| `pkill -SIGNAL pattern` | Signal by name |
| `killall name` | Signal all processes with that exact name |
| `kill -- -PGID` | Signal a whole **process group** (negative number) |
| `systemctl kill -s SIGNAL service` | Signal the processes of a service |

```console
$ kill 4821                     # polite request
$ sleep 5; kill -0 4821 2>/dev/null && kill -9 4821   # force if still there

$ pkill -HUP nginx              # ask nginx to reload
$ systemctl reload nginx        # preferred for services

$ kill -l | head -3
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL
```

You may only signal your own processes. **Root** can signal all.

### Stopping services: use systemd

For services managed by systemd, use `systemctl stop` or `restart`. systemd sends SIGTERM, waits (`TimeoutStopSec`, default 90 seconds), then sends SIGKILL to the whole control group. Killing the main PID by hand makes systemd treat it as a failure, and `Restart=` may start it again.

### Signals in scripts: trap

```bash
#!/bin/bash
cleanup() { rm -f /tmp/myjob.lock; echo "cleaned up"; }
trap cleanup EXIT                 # run at exit
trap 'echo interrupted; exit 130' INT TERM
touch /tmp/myjob.lock
sleep 60
```

(Chapter 17 covers `trap` in detail.)

### Core dumps

When a process crashes with SIGSEGV, SIGABRT, or SIGQUIT, the kernel can save its memory in a **core dump** for debugging. On systemd systems:

```console
$ coredumpctl list
$ coredumpctl info PID
$ coredumpctl debug PID          # opens in gdb
```

## Common pitfalls

- **Using `kill -9` first.** It skips cleanup. Use `SIGTERM` and wait.
- **Process does not exit after SIGKILL.** It is in `D` state (waiting for I/O) or a zombie. See [Process Lifecycle and States](01-process-lifecycle-and-states.md).
- **`pkill` pattern too broad.** `pkill python` ends every Python process, including important ones. Check with `pgrep -a`.
- **Killing a service's child process** and expecting no effect. Use systemd tools for services.
- **`killall` differences.** On Linux it matches by exact name. On some other Unix systems, `killall` without arguments kills **all** processes. Do not use it casually on unfamiliar systems.
- **Killing a parent kills children** (or leaves orphans). Think about the process tree first.
- **Sending SIGHUP to a program that does not handle it** terminates it. Check the documentation to see if reload is supported.

> [!WARNING]
> `kill -9 1` or killing critical system processes can crash the system. `kill -9 -1` sends SIGKILL to **every process you can signal**. Do not run it.

## Further reading

- [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html)
- [`kill(1)`](https://man7.org/linux/man-pages/man1/kill.1.html)
- [`coredumpctl(1)`](https://man7.org/linux/man-pages/man1/coredumpctl.1.html)
