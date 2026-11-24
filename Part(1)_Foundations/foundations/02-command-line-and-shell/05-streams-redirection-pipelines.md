# Input/Output Streams, Redirection, Pipelines, and Command Substitution

## Overview

Every Linux command reads from an input stream and writes to output streams. You can redirect those streams to files and connect commands together. This is the core of command-line power.

## Key concepts

### The three standard streams

| Name | Number (file descriptor) | Default | Purpose |
|------|--------------------------|---------|---------|
| **stdin** | 0 | Keyboard | Input |
| **stdout** | 1 | Terminal screen | Normal output |
| **stderr** | 2 | Terminal screen | Error messages |

stdout and stderr both show on screen, but they are **separate streams**. You can redirect them independently.

### Redirection operators

| Operator | Meaning |
|----------|---------|
| `> file` | Send stdout to `file` (**overwrite**) |
| `>> file` | Send stdout to `file` (**append**) |
| `< file` | Read stdin from `file` |
| `2> file` | Send stderr to `file` |
| `2>> file` | Append stderr to `file` |
| `&> file` | Send stdout and stderr to `file` (Bash) |
| `> file 2>&1` | Send stdout to `file`, and stderr to the same place (portable) |
| `2>&1` | Send stderr to where stdout goes |
| `/dev/null` | A special file that discards everything written to it |
| `<<EOF` | Here-document: feed several lines as input |

```console
$ ls /etc > list.txt              # stdout to file
$ ls /nonexistent 2> errors.txt   # stderr to file
$ ls /etc /nonexistent > out.txt 2> err.txt
$ command > all.log 2>&1          # both to one file
$ command > /dev/null 2>&1        # discard everything
$ echo "new line" >> list.txt     # append
```

> [!WARNING]
> `>` **overwrites** the target file without asking. Run `set -o noclobber` to make Bash refuse to overwrite existing files with `>` (use `>|` to force it).

Order matters: `command > file 2>&1` works. `command 2>&1 > file` does **not** send errors to the file (stderr goes to the old stdout, the terminal).

### Here-documents

```console
$ cat > hello.txt <<EOF
Hello
This is a file
EOF
```

### Pipelines

A **pipe** (`|`) sends the stdout of one command to the stdin of the next. Pipes let you build tools from small parts.

```console
$ ls /etc | wc -l
231

$ ps aux | grep ssh | grep -v grep
$ cat access.log | sort | uniq -c | sort -rn | head -5
```

By default, a pipe passes only stdout. To include stderr: `command1 2>&1 | command2` (or `|&` in Bash).

Useful companion commands:

| Command | Purpose |
|---------|---------|
| `tee FILE` | Copy stdin to both the screen and a file |
| `tee -a FILE` | Same, but append |
| `xargs` | Build command lines from stdin |

```console
$ ls | tee files.txt | wc -l
$ echo "line" | sudo tee /etc/example.conf > /dev/null
```

The last example is the correct way to write to a root-owned file: `sudo echo "x" > /etc/file` fails because the **redirection is done by your shell, not by sudo**.

### Command substitution

`$(command)` runs the command and replaces itself with its output.

```console
$ echo "Kernel: $(uname -r)"
Kernel: 6.8.0-45-generic

$ today=$(date +%F)
$ mkdir "backup-$today"

$ ls -l $(which bash)
```

Prefer `$(...)` over the old backtick form. It is clearer and can be nested.

### Using `xargs`

```console
$ find . -name '*.tmp' -print0 | xargs -0 rm
```

`-print0` and `-0` use a null character as separator, which handles spaces in file names safely.

## Common pitfalls

- **`sudo command > /root/file` fails.** The redirect runs as you. Use `command | sudo tee /root/file`.
- **Reading and writing the same file.** `sort data.txt > data.txt` empties the file before `sort` reads it. Write to a temporary file, then move it, or use `sort -o data.txt data.txt`.
- **Wrong order of `2>&1`.** Put it **after** the `> file`.
- **Large outputs filling a disk.** Be careful with `>` and `tee` on big streams.

## Further reading

- [Bash manual: Redirections](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)
- [`pipe(7)`](https://man7.org/linux/man-pages/man7/pipe.7.html)
