# Exit Codes and Command Chaining

## Overview

Every command returns a number when it finishes. This **exit code** tells the shell whether the command worked. You can use exit codes to chain commands and make decisions.

## Key concepts

- **0** means **success**.
- **Any non-zero value** (1–255) means **failure** or a special condition.
- The exit code of the last command is stored in the variable `$?`.

### Common exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Misuse of a command (wrong options) |
| 126 | Command found but not executable |
| 127 | Command not found |
| 130 | Stopped by `Ctrl+C` |
| 137 | Killed by signal 9 (SIGKILL), often by the out-of-memory killer |
| 143 | Ended by signal 15 (SIGTERM) |

For `grep`: 0 means a match was found, 1 means no match, 2 means an error.

```console
$ ls /etc/hostname
/etc/hostname
$ echo $?
0

$ ls /nonexistent
ls: cannot access '/nonexistent': No such file or directory
$ echo $?
2

$ nosuchcommand
bash: nosuchcommand: command not found
$ echo $?
127
```

### Chaining operators

| Operator | Meaning | Runs the second command when... |
|----------|---------|---------------------------------|
| `;` | Sequence | Always |
| `&&` | AND | The first command **succeeds** (exit 0) |
| `\|\|` | OR | The first command **fails** (non-zero) |
| `&` | Background | Starts the first command in the background and continues |
| `\|` | Pipe | Connects output to input (not about exit codes) |

```console
$ mkdir build && cd build            # cd only if mkdir worked
$ cd /backup || echo "no backup dir" # message only if cd failed
$ make && make install || echo "build failed"
$ date; uptime; whoami               # run all three, in order
```

> [!NOTE]
> `a && b || c` is **not** a true if-else. If `a` succeeds and `b` fails, `c` also runs. Use a real `if` statement for that (see Chapter 17).

### Grouping

```console
$ (cd /tmp && ls)          # runs in a subshell; your current directory does not change
$ { echo one; echo two; } > out.txt   # runs in the current shell; needs spaces and a final ;
```

### Pipeline exit status

By default, the exit code of a pipeline is the exit code of the **last** command. In Bash, `set -o pipefail` makes a pipeline fail if **any** command fails. The array `${PIPESTATUS[@]}` holds each command's code.

```console
$ false | true; echo $?
0
$ set -o pipefail
$ false | true; echo $?
1
```

## Commands

| Command | Purpose |
|---------|---------|
| `true` | Does nothing, exit code 0 |
| `false` | Does nothing, exit code 1 |
| `test EXPR` or `[ EXPR ]` | Evaluate a condition; exit 0 if true |
| `[[ EXPR ]]` | Bash's improved test |
| `exit N` | Leave the shell or script with code N |

```console
$ [ -f /etc/passwd ] && echo "file exists"
file exists

$ [ -d /nonexistent ] || echo "directory missing"
directory missing
```

## Common pitfalls

- **Checking `$?` too late.** It changes after every command, including `echo`. Save it first: `rc=$?`.
- **Treating non-zero as always an error.** `grep` returning 1 only means "no match".
- **Using `;` when the second command depends on the first.** Use `&&`.

## Further reading

- [Bash manual: Exit Status](https://www.gnu.org/software/bash/manual/html_node/Exit-Status.html)
- [Bash manual: Lists of Commands](https://www.gnu.org/software/bash/manual/html_node/Lists.html)
