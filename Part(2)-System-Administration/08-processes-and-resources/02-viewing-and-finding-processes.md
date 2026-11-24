# Viewing and Finding Processes: ps, top, htop, pgrep, pkill

## Overview

These tools answer: *What is running? Who started it? What is using the CPU and memory? Which PID do I need?*

## Commands

### ps: a snapshot

`ps` has two option styles. The BSD style has no dash (`ps aux`). The standard style has a dash (`ps -ef`). Both are common.

| Command | Purpose |
|---------|---------|
| `ps aux` | All processes, user-oriented format |
| `ps -ef` | All processes, full format (shows PPID) |
| `ps -u alice` | Processes of one user |
| `ps -C nginx` | Processes with this command name |
| `ps -p 812,813` | Specific PIDs |
| `ps -eo pid,ppid,user,%cpu,%mem,stat,etime,cmd` | Choose columns |
| `ps -eo pid,%mem,cmd --sort=-%mem \| head` | Top memory users |
| `ps -eo pid,%cpu,cmd --sort=-%cpu \| head` | Top CPU users |
| `ps -ef --forest` | ASCII process tree |
| `ps -eLf` | Threads |

```console
$ ps aux | head -3
USER   PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root     1  0.0  0.1 168000 13000 ?     Ss   Mar01   0:09 /sbin/init
root   812  0.0  0.0  15400  8200 ?     Ss   Mar01   0:00 sshd: /usr/sbin/sshd -D
```

| Column | Meaning |
|--------|---------|
| `%CPU` | CPU time divided by the **lifetime** of the process (not the current load) |
| `%MEM` | Share of physical memory |
| `VSZ` | Virtual memory size (includes mapped but unused memory) |
| `RSS` | Resident set size: memory actually in RAM |
| `TTY` | Terminal (`?` = none) |
| `STAT` | State (see previous page) |
| `TIME` | Total CPU time used |
| `ETIME` | Elapsed time since start |

### top: live view

```console
$ top
top - 11:30:02 up 12 days,  3:11,  2 users,  load average: 0.45, 0.30, 0.25
Tasks: 182 total,   1 running, 181 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.1 us,  1.0 sy,  0.0 ni, 95.5 id,  0.3 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :  15990.2 total,   1250.4 free,   4180.1 used,  10559.7 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.  10810.1 avail Mem

  PID USER   PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 2210 app    20   0 4812344 812344  31200 S   2.3   5.0   1:22.10 java
```

Useful keys inside `top`:

| Key | Action |
|-----|--------|
| `q` | Quit |
| `h` | Help |
| `1` | Show each CPU separately |
| `P` / `M` / `T` | Sort by CPU / memory / time |
| `c` | Show full command line |
| `H` | Show threads |
| `u` | Filter by user |
| `k` | Kill a process (asks for PID and signal) |
| `r` | Renice a process |
| `f` | Choose fields and sort column |
| `E` / `e` | Change memory unit (summary / list) |
| `W` | Save the current settings |

Batch mode for scripts and logs:

```console
$ top -b -n 1 | head -15
$ top -b -n 1 -o %MEM | head -12
```

### htop: friendlier

`htop` (install the package `htop`) has colors, mouse support, a tree view (`F5`), search (`F3`), filter (`F4`), and kill (`F9`). Same information as `top`.

### Find processes by name: pgrep and pkill

| Command | Purpose |
|---------|---------|
| `pgrep nginx` | PIDs of processes whose name matches |
| `pgrep -a nginx` | PIDs and full command lines |
| `pgrep -u alice` | Processes of a user |
| `pgrep -f "python app.py"` | Match the **whole command line** |
| `pgrep -x sshd` | Exact name match |
| `pgrep -o` / `-n` | Oldest / newest match |
| `pidof sshd` | PIDs of a program by exact name |
| `pkill nginx` | Send a signal (SIGTERM by default) to matches |
| `pkill -u alice` | Signal all processes of a user |
| `pkill -f pattern` | Match the full command line |

```console
$ pgrep -a sshd
812 sshd: /usr/sbin/sshd -D
2100 sshd: alice [priv]

$ pgrep -af "backup.sh"
```

Test a pattern with `pgrep -a` **before** you run `pkill` with the same pattern.

### Other useful views

| Command | Purpose |
|---------|---------|
| `pstree -p` | Tree |
| `uptime` | Load averages |
| `w` | Logged-in users and their commands |
| `lsof -p PID` | Open files of a process |
| `systemd-cgtop` | CPU, memory, and I/O by service |
| `systemctl status SERVICE` | Processes of a service |

## Common pitfalls

- **`ps aux | grep name` lists the `grep` itself.** Use `pgrep -a name`, or `grep '[n]ame'`.
- **Reading `%CPU` in `ps` as current usage.** It is an average over the process lifetime. Use `top` or `pidstat 1` for current usage.
- **`VIRT` and `VSZ` look huge.** Virtual size includes memory that is mapped but not used. Look at `RES` and `RSS`.
- **`pkill -f` matching too much.** It matches the full command line, including your own shell command. Check with `pgrep -af` first.
- **Missing processes in containers.** Processes of containers are visible on the host, but with different PIDs. Use `nsenter` or `docker top`.

## Further reading

- [`ps(1)`](https://man7.org/linux/man-pages/man1/ps.1.html)
- [`top(1)`](https://man7.org/linux/man-pages/man1/top.1.html)
- [`pgrep(1)`](https://man7.org/linux/man-pages/man1/pgrep.1.html)
- [htop](https://htop.dev)
