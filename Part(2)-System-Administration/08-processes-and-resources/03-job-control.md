# Foreground and Background Processes, and Job Control

## Overview

A shell can run several commands at once. **Job control** lets you move commands between the foreground and the background, pause them, and resume them. It works only in an interactive shell. For long-running tasks on servers, you also need to keep them alive after you log out.

## Key concepts

- A **foreground** job holds your terminal. It receives your keyboard input.
- A **background** job runs without taking the terminal. You get the prompt back.
- A **job** is a pipeline that the shell started. Jobs have numbers (`%1`, `%2`), different from PIDs.
- When you close a terminal or SSH session, the shell sends **SIGHUP** to its jobs. Most programs then exit.

## Commands

### Basic job control

| Action | How |
|--------|-----|
| Start in the background | `command &` |
| Pause the foreground job | `Ctrl+Z` (sends SIGTSTP) |
| List jobs | `jobs` (`jobs -l` includes PIDs) |
| Resume in the background | `bg %1` |
| Bring to the foreground | `fg %1` |
| Stop (kill) a job | `kill %1` |
| Wait for background jobs | `wait` or `wait PID` |

```console
$ sleep 300 &
[1] 4821

$ sleep 600
^Z
[2]+  Stopped                 sleep 600

$ jobs -l
[1]-  4821 Running                 sleep 300 &
[2]+  4830 Stopped                 sleep 600

$ bg %2
[2]+ sleep 600 &
$ fg %1
sleep 300
^C
```

`%+` (or `%%`) is the current job and `%-` the previous one. `$!` holds the PID of the most recent background command.

### Keep a job running after logout

| Method | Command | Notes |
|--------|---------|-------|
| `nohup` | `nohup ./script.sh > out.log 2>&1 &` | Ignores SIGHUP. Without redirects, output goes to `nohup.out` |
| `disown` | `./script.sh &` then `disown -h %1` | Detach an **already running** job from the shell |
| `setsid` | `setsid ./script.sh > out.log 2>&1 &` | New session; no terminal |
| **`tmux` / `screen`** | `tmux new -s work` | A **persistent terminal session**. Detach with `Ctrl+b d`, reattach with `tmux attach -t work`. **Best choice for interactive long tasks** |
| **systemd** | `systemd-run --unit=mytask ./script.sh` | Runs as a managed service with logging and limits |

```console
$ nohup ./backup.sh > backup.log 2>&1 &
[1] 5120
$ disown %1
```

A job that is already running and was not started with `nohup`:

1. Press `Ctrl+Z` to pause it.
2. `bg` to continue in the background.
3. `disown -h %1` so SIGHUP is not sent.

### tmux in brief

| Command / keys | Action |
|----------------|--------|
| `tmux new -s NAME` | New session |
| `Ctrl+b d` | Detach |
| `tmux ls` | List sessions |
| `tmux attach -t NAME` | Reattach |
| `Ctrl+b c` / `Ctrl+b n` | New window / next window |
| `Ctrl+b %` / `Ctrl+b "` | Split vertically / horizontally |
| `tmux kill-session -t NAME` | End a session |

### Run a task as a transient service

```console
# systemd-run --unit=reindex --description="Reindex job" /usr/local/bin/reindex.sh
Running as unit: reindex.service
# systemctl status reindex
# journalctl -u reindex -f
```

This survives logouts, logs output to the journal, and can have limits (`-p MemoryMax=2G`).

### Other useful commands

| Command | Purpose |
|---------|---------|
| `watch -n 2 'command'` | Re-run a command every 2 seconds |
| `timeout 30 command` | Stop after 30 seconds |
| `wait` | Wait in scripts for background jobs |
| `xargs -P 4` / `parallel` | Run jobs in parallel |

## Common pitfalls

- **Background job dies when the SSH session closes.** Use `tmux`, `nohup`, or `systemd-run`.
- **A background job waits for input and stops** (`Stopped (tty input)`). Redirect stdin: `command < /dev/null &`.
- **Output from background jobs clutters the terminal.** Redirect it to a file.
- **`nohup` and `&` do not make a service.** For permanent services use a systemd unit.
- **Job numbers vs. PIDs.** `kill %1` uses a job number. `kill 4821` uses a PID.
- **Jobs belong to one shell.** Another terminal cannot see your `jobs`. Use `ps` there.

## Further reading

- [Bash manual: Job Control](https://www.gnu.org/software/bash/manual/html_node/Job-Control.html)
- [`nohup(1)`](https://man7.org/linux/man-pages/man1/nohup.1.html)
- [tmux wiki](https://github.com/tmux/tmux/wiki)
- [`systemd-run(1)`](https://man7.org/linux/man-pages/man1/systemd-run.1.html)
