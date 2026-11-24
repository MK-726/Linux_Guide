# Terminal, Shell, and Bash

## Overview

The command line lets you control Linux by typing commands. It is faster, scriptable, and works over remote connections. Three words are often confused: **terminal**, **shell**, and **command**.

## Key concepts

| Term | Meaning |
|------|---------|
| **Terminal** (terminal emulator) | The window that shows text and sends your keystrokes. Examples: GNOME Terminal, Konsole, the SSH client. |
| **Shell** | The program inside the terminal that reads your commands and runs them. Examples: Bash, Zsh, Dash. |
| **Command** | A program or shell feature that you ask the shell to run. |
| **Bash** | "Bourne Again SHell." The default shell on most distributions. This guide uses Bash. |

### Anatomy of a command

```console
$ ls -l /etc
```

| Part | Name | Meaning |
|------|------|---------|
| `ls` | Command | The program to run |
| `-l` | Option (flag) | Changes how the command behaves |
| `/etc` | Argument | What the command works on |

- **Short options** use one dash and one letter: `-l`. Short options can often be combined: `-la` means `-l -a`.
- **Long options** use two dashes and a word: `--all`.
- Linux is **case-sensitive**: `File.txt` and `file.txt` are different files.

### How the shell runs a command

1. You type a line and press **Enter**.
2. The shell splits the line into words and expands things like `*` and `$HOME`.
3. If the first word is a **built-in** (such as `cd`), the shell runs it itself.
4. Otherwise, the shell searches the directories in `PATH` for a program with that name and runs it.
5. When the program ends, the shell shows the prompt again.

### Types of commands

```console
$ type cd ls ll
cd is a shell builtin
ls is /usr/bin/ls
ll is aliased to `ls -alF'
```

| Type | Description |
|------|-------------|
| Builtin | Part of the shell (`cd`, `echo`, `export`) |
| External program | A file on disk (`/usr/bin/ls`) |
| Alias | A shortcut defined in the shell |
| Function | A named block of shell code |

### Login and non-login shells

| Shell kind | Reads these files (Bash) |
|-----------|--------------------------|
| Login shell (for example, SSH login) | `/etc/profile`, then the first found of `~/.bash_profile`, `~/.bash_login`, `~/.profile` |
| Interactive non-login shell (new terminal window) | `~/.bashrc` |

Most systems make `~/.bash_profile` or `~/.profile` load `~/.bashrc`, so your settings apply in both.

## Commands

```console
$ echo $SHELL            # your login shell
/bin/bash

$ echo $0                # the current shell
bash

$ cat /etc/shells        # shells allowed as login shells
/bin/sh
/bin/bash
/usr/bin/zsh

$ bash --version | head -1
GNU bash, version 5.2.21(1)-release (x86_64-pc-linux-gnu)
```

Useful keyboard shortcuts:

| Keys | Action |
|------|--------|
| `Ctrl+C` | Stop the running command |
| `Ctrl+D` | End of input; exit the shell if the line is empty |
| `Ctrl+L` | Clear the screen |
| `Ctrl+A` / `Ctrl+E` | Move to the start / end of the line |
| `Ctrl+W` | Delete the previous word |
| `Ctrl+U` | Delete from the cursor to the start of the line |
| `Ctrl+R` | Search command history |
| `Ctrl+Z` | Suspend the running command (see Chapter 8) |

## Configuration files

| File | Purpose |
|------|---------|
| `~/.bashrc` | Settings for interactive shells: aliases, prompt, functions |
| `~/.bash_profile` or `~/.profile` | Settings for login shells |
| `~/.bash_logout` | Runs when a login shell exits |
| `/etc/profile`, `/etc/bash.bashrc` (Debian) or `/etc/bashrc` (RHEL) | System-wide settings |

## Common pitfalls

- **Editing `~/.bashrc` and seeing no change.** Reload it with `source ~/.bashrc` or open a new terminal.
- **Confusing `sh` and `bash`.** On Debian-based systems `/bin/sh` is usually **Dash**, a smaller shell without all Bash features. Scripts that use Bash features need `#!/bin/bash`.
- **Forgetting case sensitivity.** `cd documents` fails if the directory is `Documents`.

## Further reading

- [Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html)
- [`bash(1)` manual page](https://man7.org/linux/man-pages/man1/bash.1.html)
