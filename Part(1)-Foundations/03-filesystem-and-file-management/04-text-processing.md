# Text Processing: grep, sed, awk, cut, sort, uniq

## Overview

Logs, configuration files, and command output are all text. These tools search, filter, transform, and summarize text. Combined with pipes, they answer most questions about a system quickly.

## Key concepts

- Each tool reads **lines** from files or from stdin and writes to stdout.
- Most use **regular expressions** (patterns) to match text.
- Chain them with pipes: `command | grep ... | sort | uniq -c`.

### Basic regular expression symbols

| Symbol | Meaning | Example |
|--------|---------|---------|
| `.` | Any one character | `a.c` matches `abc` |
| `*` | Zero or more of the previous item | `ab*` matches `a`, `ab`, `abb` |
| `^` | Start of line | `^root` |
| `$` | End of line | `bash$` |
| `[abc]` | One of these characters | `[0-9]` is a digit |
| `[^abc]` | Not one of these | `[^0-9]` |
| `\|` (ERE) | Or | `error\|fail` |
| `+`, `?` (ERE) | One or more, zero or one | `[0-9]+` |
| `{n,m}` (ERE) | Repeat n to m times | `[0-9]{1,3}` |

**ERE** means *extended* regular expressions. Use `grep -E` (or `egrep`), `sed -E`, and `awk` (default).

## Commands

### grep: search for lines that match

| Option | Meaning |
|--------|---------|
| `-i` | Ignore case |
| `-v` | Invert: show lines that do **not** match |
| `-n` | Show line numbers |
| `-c` | Count matching lines |
| `-r` | Search directories recursively |
| `-l` | Show only file names with matches |
| `-w` | Match whole words |
| `-E` | Extended regular expressions |
| `-F` | Fixed text, no regex (fast and safe) |
| `-A N` / `-B N` / `-C N` | Show N lines after / before / around a match |
| `-o` | Show only the matching part |

```console
$ grep "Failed password" /var/log/auth.log | tail -3
$ grep -rn "PermitRootLogin" /etc/ssh/
/etc/ssh/sshd_config:34:PermitRootLogin prohibit-password
$ grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'     # remove comments and blank lines
$ grep -E "error|warning" app.log
$ grep -c "404" access.log
```

### cut: extract columns

```console
$ cut -d: -f1 /etc/passwd | head -3         # field 1, ":" separator
root
daemon
bin
$ cut -d: -f1,7 /etc/passwd | head -2
root:/bin/bash
daemon:/usr/sbin/nologin
$ cut -c1-10 file.txt                       # characters 1 to 10
```

### sort and uniq: order and count

`uniq` only works on **adjacent** lines, so sort first.

| Command | Meaning |
|---------|---------|
| `sort` | Alphabetical |
| `sort -n` | Numeric |
| `sort -r` | Reverse |
| `sort -h` | Human-readable sizes (`10K`, `2G`) |
| `sort -k2` | Sort by field 2 |
| `sort -t: -k3 -n` | Field separator `:`, field 3, numeric |
| `sort -u` | Remove duplicates |
| `uniq -c` | Count repeats |
| `uniq -d` | Show only duplicated lines |

```console
$ cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -3
   1520 203.0.113.9
    860 198.51.100.4
    214 192.0.2.77
```

This shows the three most active client IP addresses.

### sed: stream editor

`sed` edits text as it flows through. Most-used form is **substitute**: `s/old/new/flags`.

```console
$ echo "hello world" | sed 's/world/Linux/'
hello Linux
$ sed 's/foo/bar/g' file.txt              # replace all on each line (print result)
$ sed -i 's/foo/bar/g' file.txt           # edit the file in place
$ sed -i.bak 's/foo/bar/g' file.txt       # in place, with backup file.txt.bak
$ sed -n '5,10p' file.txt                 # print lines 5-10 only
$ sed '/^#/d' file.txt                    # delete comment lines
$ sed -E 's/([0-9]+)-([0-9]+)/\2-\1/' f   # capture groups
```

> [!WARNING]
> `sed -i` changes the file immediately. Test without `-i` first, or use `-i.bak`.

### awk: process columns and records

`awk` splits each line into **fields** (`$1`, `$2`, ... `$NF` = last, `$0` = whole line). Default separator is whitespace.

Syntax: `awk 'CONDITION { ACTION }' file`

```console
$ awk '{print $1}' access.log | head -2
$ awk -F: '{print $1, $3}' /etc/passwd | head -2          # -F sets the separator
root 0
daemon 1
$ awk -F: '$3 >= 1000 {print $1}' /etc/passwd             # users with UID >= 1000
$ awk '{sum += $5} END {print sum}' data.txt              # total of column 5
$ awk 'NR==1 || /error/' app.log                          # header line and error lines
$ ps aux | awk '$3 > 50 {print $2, $3, $11}'             # processes using more than 50% CPU
```

### Other handy tools

| Tool | Purpose | Example |
|------|---------|---------|
| `wc` | Count lines, words, bytes | `wc -l file` |
| `tr` | Translate or delete characters | `tr 'a-z' 'A-Z'` |
| `head`, `tail` | First or last lines | `tail -n 20 file` |
| `paste` | Merge lines side by side | `paste a.txt b.txt` |
| `column -t` | Align columns | `mount \| column -t` |
| `tee` | Write to a file and to stdout | `cmd \| tee out.txt` |
| `xargs` | Turn input into arguments | `find . -name "*.log" \| xargs grep ERROR` |

## Common pitfalls

- **`uniq` without `sort`** misses duplicates that are not next to each other.
- **Numbers sorted as text.** `sort` puts `10` before `2` unless you use `-n`.
- **Regex vs. fixed text.** A dot in `grep "1.2"` matches any character. Use `grep -F "1.2"` for exact text.
- **Special characters in `sed` patterns.** Use a different delimiter for paths: `sed 's|/usr/bin|/opt/bin|'`.
- **GNU vs. BSD tools.** On Linux these are GNU versions; some options differ on macOS.
- **`grep` matching itself in `ps | grep`.** Add `grep -v grep`, or use `pgrep`.

## Further reading

- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html)
- [GNU sed manual](https://www.gnu.org/software/sed/manual/sed.html)
- [GNU awk user's guide](https://www.gnu.org/software/gawk/manual/gawk.html)
- [`regex(7)`](https://man7.org/linux/man-pages/man7/regex.7.html)
