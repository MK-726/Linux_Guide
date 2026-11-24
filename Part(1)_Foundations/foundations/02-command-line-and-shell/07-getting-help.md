# Getting Help: man, info, and --help

## Overview

Linux ships with documentation for almost every command. Knowing how to find answers on your own system is a core administrator skill. The documentation is local, works offline, and matches the installed version.

## Key concepts

| Source | When to use it |
|--------|----------------|
| `command --help` | Quick summary of options |
| `man command` | Full reference (manual page) |
| `info command` | Long-form GNU manuals (not on every system) |
| `/usr/share/doc/` | Package documentation and examples |
| `tldr`, `cheat` | Short example-based helpers (installed separately) |
| Distribution and upstream websites | Guides and release notes |

### Manual page sections

| Section | Content |
|---------|---------|
| 1 | User commands |
| 2 | System calls |
| 3 | Library functions |
| 4 | Special files (devices) |
| 5 | File formats and configuration files |
| 6 | Games |
| 7 | Overviews and conventions |
| 8 | System administration commands |

The same name can exist in several sections. For example, `passwd(1)` is the command and `passwd(5)` is the file format.

## Commands

```console
$ ls --help | head -5
Usage: ls [OPTION]... [FILE]...
List information about the FILEs (the current directory by default).
...

$ man ls              # open the manual page
$ man 5 passwd        # the passwd file format (section 5)
$ man -k "copy file"  # search manual page descriptions (same as apropos)
$ apropos network     # search by keyword
$ whatis ls
ls (1)               - list directory contents
$ info coreutils
$ type -a echo        # shows builtin and external versions
$ help cd             # help for Bash builtins
```

### Reading a manual page

| Key | Action |
|-----|--------|
| `Space` / `b` | Next / previous page |
| `/word` then `n` | Search forward, repeat |
| `?word` | Search backward |
| `g` / `G` | Top / bottom |
| `q` | Quit |

Typical manual page parts: **NAME**, **SYNOPSIS**, **DESCRIPTION**, **OPTIONS**, **EXAMPLES**, **FILES**, **EXIT STATUS**, **SEE ALSO**.

In the SYNOPSIS, `[ ]` means optional, `...` means repeatable, and `|` means "or".

### Installing missing manual pages

Minimal server and container images often leave manual pages out.

**Debian-based**

```console
# apt install man-db manpages
```

**RHEL-based**

```console
# dnf install man-db man-pages
```

### Updating the search index

`man -k` and `apropos` use a database:

```console
# mandb
```

## Common pitfalls

- **"No manual entry for ..."** The manual page package is not installed, or the command is a Bash builtin (use `help`).
- **Wrong version of the answer from the web.** Online guides may describe other versions. `man` on your machine always matches your system.
- **Confusing builtins and programs.** `man cd` may show nothing useful because `cd` is a builtin. Use `help cd`.

## Further reading

- [man7.org Linux manual pages](https://man7.org/linux/man-pages/)
- [`man(1)`](https://man7.org/linux/man-pages/man1/man.1.html)
