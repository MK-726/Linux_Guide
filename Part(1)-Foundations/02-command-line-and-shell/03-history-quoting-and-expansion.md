# Command History, Tab Completion, Quoting, and Expansion

## Overview

The shell does a lot of work on your input before it runs a command. Understanding **history**, **completion**, **quoting**, and **expansion** prevents many mistakes and makes you faster.

## Key concepts

### Command history

Bash remembers commands in memory and saves them to `~/.bash_history` when the shell exits.

| Command / keys | Purpose |
|----------------|---------|
| `history` | Show numbered history |
| `!!` | Repeat the last command |
| `!N` | Repeat command number N |
| `!string` | Repeat the last command that started with `string` |
| `sudo !!` | Repeat the last command with `sudo` |
| `Ctrl+R` | Search history as you type |
| `Up` / `Down` | Move through history |

```console
$ history | tail -3
  498  cd /etc
  499  ls
  500  history | tail -3
```

> [!WARNING]
> Never type passwords on the command line. They are saved in history. Use prompts, environment files with correct permissions, or secret managers.

### Tab completion

Press **Tab** to complete command names, file names, and (with the `bash-completion` package) options and arguments. Press **Tab twice** to list possible matches.

### Quoting

Quotes control which special characters the shell interprets.

| Form | Behavior | Example |
|------|----------|---------|
| No quotes | The shell expands everything and splits on spaces | `echo $HOME  *.txt` |
| `"double quotes"` | Variables (`$VAR`), command substitution, and `\` still work. Spaces are kept. Wildcards are **not** expanded. | `echo "Home: $HOME"` |
| `'single quotes'` | Everything is literal | `echo 'Home: $HOME'` |
| `\` (backslash) | Escapes the next character | `echo \$HOME` |

```console
$ name="Ada Lovelace"
$ echo "Hello, $name"
Hello, Ada Lovelace
$ echo 'Hello, $name'
Hello, $name
$ echo Hello,\ $name
Hello, Ada Lovelace
```

> [!IMPORTANT]
> **Always quote variables** that may contain spaces: `rm "$file"`, not `rm $file`.

### Expansion

The shell expands these forms, in this order: brace expansion, tilde expansion, parameter (variable) expansion, command substitution, arithmetic expansion, word splitting, then pathname (wildcard) expansion.

| Expansion | Example | Result |
|-----------|---------|--------|
| Brace | `echo {a,b,c}` | `a b c` |
| Brace range | `echo {1..5}` | `1 2 3 4 5` |
| Tilde | `echo ~` | `/home/alice` |
| Variable | `echo $USER` | `alice` |
| Default value | `echo ${NAME:-guest}` | `guest` if `NAME` is unset or empty |
| Command substitution | `echo "Today is $(date +%F)"` | `Today is 2025-03-03` |
| Arithmetic | `echo $((5 * 4))` | `20` |
| Wildcard | `echo *.txt` | `notes.txt todo.txt` |

Command substitution is covered further in [Streams, Redirection, and Pipelines](05-streams-redirection-pipelines.md).

## Commands

Test what the shell will do **before** running something dangerous. Put `echo` in front:

```console
$ echo rm *.log
rm app.log error.log
```

Add and see an alias:

```console
$ alias ll='ls -alF'
$ alias
alias ll='ls -alF'
$ unalias ll
```

Add aliases permanently to `~/.bashrc`.

## Configuration files

| Setting (in `~/.bashrc`) | Effect |
|--------------------------|--------|
| `HISTSIZE=10000` | Commands kept in memory |
| `HISTFILESIZE=20000` | Lines kept in `~/.bash_history` |
| `HISTCONTROL=ignoredups:ignorespace` | Skip duplicates and commands starting with a space |
| `HISTTIMEFORMAT="%F %T "` | Add timestamps to history |

## Common pitfalls

- **Unquoted variables.** If `file="my report.txt"`, then `rm $file` tries to delete `my` and `report.txt`.
- **Wrong quotes.** Single quotes block variable expansion. Use double quotes when you need `$VAR`.
- **`!` in double quotes.** In an interactive shell, `echo "Hello!"` can trigger history expansion. Use single quotes or turn it off with `set +H`.

## Further reading

- [Bash manual: Shell Expansions](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html)
- [Bash manual: Quoting](https://www.gnu.org/software/bash/manual/html_node/Quoting.html)
