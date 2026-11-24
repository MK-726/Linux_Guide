# Environment Variables and Shell Variables

## Overview

Variables store values the shell and programs can use. Some are private to the current shell. Others are passed to every program you start. The most important one for daily work is `PATH`.

## Key concepts

| Kind | Visible to | Created with |
|------|-----------|--------------|
| **Shell variable** | The current shell only | `NAME=value` |
| **Environment variable** | The shell **and** every program it starts | `export NAME=value` |

### Setting and reading variables

- No spaces around `=`.
- Read a variable with `$NAME` or `${NAME}`.

```console
$ city="Berlin"
$ echo "$city"
Berlin

$ bash -c 'echo "City is: $city"'      # child process does not see it
City is:

$ export city
$ bash -c 'echo "City is: $city"'      # now it does
City is: Berlin
```

Set a variable for one command only:

```console
$ LANG=C sort names.txt
```

### Important variables

| Variable | Meaning |
|----------|---------|
| `HOME` | Your home directory |
| `USER` | Your user name |
| `SHELL` | Your login shell |
| `PWD` | Current directory |
| `PATH` | Directories searched for commands |
| `EDITOR` / `VISUAL` | Your preferred text editor |
| `LANG`, `LC_*` | Language and locale settings |
| `PS1` | The shell prompt format |
| `?` | Exit code of the last command (`$?`) |
| `$$` | PID of the current shell |

### How PATH works

`PATH` is a colon-separated list of directories. The shell searches them from left to right and runs the first match.

```console
$ echo "$PATH"
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

$ which python3
/usr/bin/python3
```

Add a directory for the current session:

```console
$ export PATH="$HOME/bin:$PATH"
```

> [!WARNING]
> Never add `.` (the current directory) to `PATH`, especially for root. An attacker could place a malicious program with a common name (like `ls`) in a directory you visit.

## Commands

| Command | Purpose |
|---------|---------|
| `printenv` | Show all environment variables |
| `printenv HOME` | Show one variable |
| `env` | Show the environment, or run a command in a modified one |
| `set` | Show all shell variables and functions |
| `export NAME=value` | Create or set an environment variable |
| `unset NAME` | Remove a variable |
| `readonly NAME=value` | Make a variable constant |
| `declare -p NAME` | Show how a variable is defined |

```console
$ printenv | grep -E '^(HOME|USER|SHELL)='
HOME=/home/alice
USER=alice
SHELL=/bin/bash
```

### Making variables permanent

Variables you set in a terminal disappear when it closes. To keep them, put them in a startup file.

| Scope | File |
|-------|------|
| Your interactive shells | `~/.bashrc` |
| Your login shells | `~/.bash_profile` or `~/.profile` |
| All users, login shells | `/etc/profile` or a file in `/etc/profile.d/` |
| All users and programs, simple `KEY=value` lines (not shell code) | `/etc/environment` (Debian-based; also read by PAM) |

Example line in `~/.bashrc`:

```bash
export EDITOR=vim
export PATH="$HOME/bin:$PATH"
```

Apply it right away:

```console
$ source ~/.bashrc
```

## Common pitfalls

- **Spaces around `=`.** `NAME = value` fails with "command not found". Write `NAME=value`.
- **Forgetting `export`.** Programs you start will not see the variable.
- **`sudo` resets the environment.** `sudo` removes most variables for safety. Use `sudo -E` carefully, or set values on the command line: `sudo VAR=value command`.
- **Secrets in environment variables.** Other processes of the same user can read them via `/proc`. Prefer secret files with strict permissions or a secrets manager.

## Further reading

- [Bash manual: Shell Variables](https://www.gnu.org/software/bash/manual/html_node/Shell-Variables.html)
- [`environ(7)`](https://man7.org/linux/man-pages/man7/environ.7.html)
