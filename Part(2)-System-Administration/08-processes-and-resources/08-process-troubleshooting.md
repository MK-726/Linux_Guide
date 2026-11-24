# Process Troubleshooting

## Overview

Most process problems fall into a few categories: using too much CPU or memory, not responding, not starting, or disappearing. This page gives a method and the commands for each case. Chapter 26 covers troubleshooting and incident response as a whole.

## Key concepts

### A simple method

1. **Define the symptom.** What is slow, failing, or missing? Since when? What changed?
2. **Check the basics.** Is the process running? Is the service healthy? Is the system itself overloaded (load, memory, disk)?
3. **Look at the logs.** `journalctl -u SERVICE`, application logs.
4. **Look at what the process is doing.** CPU, memory, I/O, open files, system calls.
5. **Change one thing at a time** and observe.
6. **Write down** what you found and did.

## Commands and scenarios

### Is it running?

```console
$ systemctl status myapp
$ pgrep -a myapp
$ ss -ltnp | grep :8080            # is it listening?
$ journalctl -u myapp -n 50 --no-pager
```

### Scenario 1: a process uses too much CPU

```console
$ top -o %CPU                       # find the process (press H for threads)
$ pidstat -u -p 2210 1 5            # CPU over time
$ top -H -p 2210                    # which thread
# strace -c -p 2210                 # which syscalls (stop with Ctrl+C)
# perf top -p 2210                  # which functions (needs perf)
```

Questions to ask: Is it a busy loop (bug)? Real workload (more traffic)? A runaway job? Compare `us` vs `sy` vs `wa`.

Mitigate: `renice`, limit with `CPUQuota=`, restart the service, then investigate the cause.

### Scenario 2: a process uses too much memory

```console
$ ps -eo pid,rss,cmd --sort=-rss | head -5
$ pidstat -r -p 2210 5              # memory growth
$ cat /proc/2210/status | grep -E 'VmRSS|VmSwap'
$ dmesg -T | grep -i oom
```

Steady growth means a **leak**. Spikes may be normal workload. Apply a `MemoryMax=` limit as a safety net and `Restart=on-failure` while you fix the root cause.

### Scenario 3: a process is hung (not responding)

```console
$ ps -o pid,stat,wchan:32,cmd -p 2210
  PID STAT WCHAN                            CMD
 2210 D    nfs_wait_bit_killable            /usr/bin/app
# cat /proc/2210/stack                       # kernel stack (root)
# strace -p 2210                             # if stopped in one syscall, shows which
$ ls -l /proc/2210/fd                        # what is it waiting on?
```

| What you see | Meaning |
|--------------|---------|
| State `D`, `wchan` mentions NFS or I/O | Waiting for storage. Check the NFS server, disk health, network |
| State `S`, stuck in `futex` | Waiting for a lock; likely an application deadlock |
| State `S`, in `poll` or `epoll_wait` | Waiting for network or events. This may be normal |
| State `R` at 100% CPU | Busy loop |

Application-level hints: `kill -QUIT` makes a JVM print a thread dump to its output. Python and others have their own tools. Check application logs and metrics.

### Scenario 4: a process keeps crashing or restarting

```console
$ systemctl status myapp                       # exit code, signal, restart counter
$ journalctl -u myapp -b --no-pager | tail -50
$ coredumpctl list                             # crash dumps
$ dmesg -T | grep -i -E 'segfault|oom|killed'
```

| Exit status | Likely cause |
|-------------|--------------|
| `status=1/FAILURE` or `2` | Application error. Read the log |
| `status=127` | Command not found (bad path or missing file) |
| `status=203/EXEC` | systemd could not execute `ExecStart` (path, permissions, missing interpreter) |
| `status=137` (signal 9) | Killed. The OOM killer or a manual kill |
| `status=139` (signal 11) | Segmentation fault. Crash (bug, bad library) |
| `status=143` (signal 15) | Stopped normally by SIGTERM |
| `start-limit-hit` | Restarted too often. Fix the cause, then `systemctl reset-failed` |

### Scenario 5: a service will not start

```console
$ systemctl status myapp -l
$ journalctl -xeu myapp
$ systemd-analyze verify /etc/systemd/system/myapp.service
$ sudo -u myapp /opt/myapp/bin/myapp --config /etc/myapp/app.conf     # run it by hand as the service user
```

Check: permissions of files and directories for the service user, SELinux or AppArmor denials, port already in use (`ss -ltnp`), missing environment variables, wrong `WorkingDirectory`.

### Scenario 6: zombies or too many processes

```console
$ ps -eo stat,pid,ppid,cmd | awk '$1 ~ /^Z/'      # find zombies and their parents
$ ps -eo ppid | sort | uniq -c | sort -rn | head  # which parent has the most children
$ ps -eLf | wc -l                                  # number of threads
$ cat /proc/sys/kernel/pid_max
```

Fix the parent. "Resource temporarily unavailable" or "fork: retry" often indicates hitting `nproc`, `TasksMax`, or `pid_max`.

### Scenario 7: find what a process is touching

```console
# lsof -p 2210                      # open files and sockets
# strace -f -e trace=file -p 2210   # file access attempts
# strace -f -e trace=network -p 2210
# ss -tnp | grep 2210               # connections
```

### Collect evidence before restarting

Restarting clears the evidence. If you can, first capture:

```console
$ date; uptime
$ top -b -n 1 > /tmp/top.txt
$ ps -eLf > /tmp/ps.txt
$ cat /proc/2210/status > /tmp/status.txt
# lsof -p 2210 > /tmp/lsof.txt
# gcore 2210                         # core dump of a running process (needs gdb)
$ journalctl -u myapp --since "1 hour ago" > /tmp/journal.txt
```

## Common pitfalls

- **Restarting first, investigating later.** You lose the evidence. Capture state, then restart.
- **Trusting one tool.** Cross-check: `top`, `pidstat`, `vmstat`, logs, and metrics.
- **Tracing production at full speed.** `strace -p` slows the process. Use short captures.
- **Fixing symptoms.** Increasing limits or adding RAM hides a leak. Find the cause.
- **Not checking the environment the service runs in.** Running a command by hand as your user differs from the service (user, environment, working directory, limits, sandboxing).
- **Ignoring recent changes.** Packages, configuration, kernel, traffic. Check `journalctl`, package logs, and change tickets.

## Further reading

- [`strace(1)`](https://man7.org/linux/man-pages/man1/strace.1.html)
- [`pidstat(1)`](https://man7.org/linux/man-pages/man1/pidstat.1.html)
- [Brendan Gregg: Linux performance](https://www.brendangregg.com/linuxperf.html)
- [`systemd.exec(5)`: exit status](https://man7.org/linux/man-pages/man5/systemd.exec.5.html)
