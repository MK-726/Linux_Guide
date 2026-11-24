# Navigation and File Operations

## Overview

These commands are the ones you will use every day: see where you are, move around, and create, copy, move, and delete files.

## Key concepts

- Your **current working directory** is the folder commands use by default.
- `~` means your **home directory**. `.` means the **current directory**. `..` means the **parent directory**.
- Files whose names start with a dot (`.bashrc`) are **hidden**. They are normal files, just not listed by default.
- Linux has **no recycle bin** on the command line. A deleted file is gone.

## Commands

### Navigation

| Command | Purpose |
|---------|---------|
| `pwd` | Print the current directory |
| `ls` | List files |
| `cd DIR` | Change directory |
| `cd` or `cd ~` | Go to your home directory |
| `cd -` | Go back to the previous directory |
| `cd ..` | Go up one level |

```console
$ pwd
/home/alice

$ cd /var/log
$ pwd
/var/log

$ cd -
/home/alice
```

### Listing files

| Command | Meaning |
|---------|---------|
| `ls -l` | Long format (permissions, owner, size, date) |
| `ls -a` | Include hidden files |
| `ls -lh` | Human-readable sizes |
| `ls -lt` | Sort by modification time, newest first |
| `ls -ld DIR` | Show the directory itself, not its contents |
| `ls -R` | List recursively |

```console
$ ls -lah
total 28K
drwxr-x--- 4 alice alice 4.0K Mar  3 10:12 .
drwxr-xr-x 5 root  root  4.0K Feb 20 09:00 ..
-rw-r--r-- 1 alice alice  220 Feb 20 09:00 .bash_logout
-rw-r--r-- 1 alice alice 3.7K Feb 20 09:00 .bashrc
drwxr-xr-x 2 alice alice 4.0K Mar  3 10:12 projects
-rw-r--r-- 1 alice alice   42 Mar  3 10:10 notes.txt
```

### Creating

```console
$ mkdir projects                  # create a directory
$ mkdir -p projects/app/src       # create parents too
$ touch notes.txt                 # create an empty file (or update its timestamp)
```

### Copying

```console
$ cp notes.txt notes.bak          # copy a file
$ cp -r projects projects.bak     # copy a directory (recursive)
$ cp -a projects projects.bak2    # archive mode: keeps permissions, ownership, timestamps
$ cp -i a.txt b.txt               # ask before overwriting
```

### Moving and renaming

```console
$ mv notes.txt todo.txt           # rename
$ mv todo.txt projects/           # move into a directory
```

### Deleting

```console
$ rm todo.txt                     # delete a file
$ rm -i todo.txt                  # ask first
$ rmdir emptydir                  # delete an empty directory
$ rm -r projects.bak              # delete a directory and everything in it
```

> [!WARNING]
> `rm -r` and `rm -rf` delete without confirmation and **cannot be undone**. Never run `rm -rf` with a path you have not checked. A stray space can change the meaning: `rm -rf /tmp/build /` is not the same as `rm -rf /tmp/build/`. Use `ls` with the same path first to see what you will delete.

### Viewing file contents

| Command | Purpose |
|---------|---------|
| `cat FILE` | Print the whole file |
| `less FILE` | Page through a file (`q` quits, `/text` searches) |
| `head -n 5 FILE` | First 5 lines |
| `tail -n 5 FILE` | Last 5 lines |
| `tail -f FILE` | Follow a file as it grows (good for logs) |
| `file FILE` | Show the file type |
| `stat FILE` | Show detailed file information |
| `wc -l FILE` | Count lines |

### Wildcards (globbing)

The shell expands these patterns **before** the command runs:

| Pattern | Matches |
|---------|---------|
| `*` | Any number of characters |
| `?` | Exactly one character |
| `[abc]` | One of `a`, `b`, or `c` |
| `[0-9]` | One digit |
| `{a,b,c}` | Brace expansion: `a`, `b`, and `c` |

```console
$ ls *.txt
notes.txt  todo.txt
$ cp file{1,2,3}.txt backup/
```

## Common pitfalls

- **Spaces in file names.** `rm my file.txt` tries to delete `my` and `file.txt`. Quote the name: `rm "my file.txt"`.
- **Trailing slash on `cp`/`mv` targets.** `cp a.txt dir` creates a file named `dir` if `dir` does not exist. Use `dir/` to make sure you mean a directory.
- **`cp` overwrites silently.** Use `cp -i` or `cp -n` (do not overwrite) when unsure.
- **`rm` on a symbolic link** removes the link, not the target. See [File Types, Inodes, and Links](../03-filesystem-and-file-management/02-file-types-inodes-and-links.md).

## Further reading

- [GNU coreutils manual](https://www.gnu.org/software/coreutils/manual/coreutils.html)
