# File Discovery: find, locate, which, whereis

## Overview

Linux provides several tools to find files and programs. They differ in speed and in what they search:

| Tool | Searches | Speed | Up to date? |
|------|----------|-------|-------------|
| `find` | The live filesystem | Slower | Always |
| `locate` | A prebuilt database | Very fast | Only after the database updates |
| `which` | Directories in `PATH` | Instant | Yes |
| `whereis` | Standard locations for binaries, sources, and manuals | Fast | Yes |
| `type` | The shell (aliases, builtins, programs) | Instant | Yes |

## Commands

### find

Syntax: `find WHERE CONDITIONS ACTION`

```console
$ find /etc -name "sshd_config"
/etc/ssh/sshd_config

$ find /var/log -name "*.log"          # quote patterns so the shell does not expand them
$ find . -iname "*.TXT"                # case-insensitive
```

Common conditions:

| Condition | Meaning |
|-----------|---------|
| `-name PATTERN` | Match the file name |
| `-iname PATTERN` | Same, ignoring case |
| `-type f` / `-type d` / `-type l` | File / directory / symlink |
| `-size +100M` | Larger than 100 MB (`-size -1k` smaller than 1 KB) |
| `-mtime -1` | Modified in the last day (`-mtime +30`: more than 30 days ago) |
| `-mmin -10` | Modified in the last 10 minutes |
| `-user alice` / `-group staff` | By owner / group |
| `-perm 644` | Exact permissions |
| `-perm -4000` | Has the SUID bit set (at least) |
| `-empty` | Empty files or directories |
| `-maxdepth N` | Do not go deeper than N levels |
| `-xdev` | Stay on one filesystem |

Combine with `-and` (default), `-or`, `-not` and use `\( ... \)` to group:

```console
$ find /home -type f \( -name "*.jpg" -o -name "*.png" \)
```

Actions:

| Action | Meaning |
|--------|---------|
| `-print` | Print the path (default) |
| `-ls` | Detailed listing |
| `-delete` | Delete matches |
| `-exec CMD {} \;` | Run a command for each match |
| `-exec CMD {} +` | Run a command with many matches at once (faster) |

```console
$ find /var/log -name "*.gz" -mtime +90 -ls
$ find . -type f -name "*.tmp" -exec rm {} +
$ find / -xdev -type f -size +500M 2>/dev/null
```

> [!WARNING]
> `-delete` and `-exec rm` remove files immediately. **First run the same `find` without the action** and check the list. Put `-delete` last, because `find` processes expressions in order.

> [!TIP]
> Searching `/` creates "Permission denied" noise for normal users. Hide it with `2>/dev/null`, or run with `sudo`.

### locate

`locate` is fast because it reads a database that is updated daily by a scheduled job.

```console
$ locate sshd_config
/etc/ssh/sshd_config
/usr/share/man/man5/sshd_config.5.gz
$ locate -i "*.conf" | head
$ locate -c nginx                    # count matches
```

Install and update (**requires root**):

**Debian-based**

```console
# apt install plocate
# updatedb
```

**RHEL-based**

```console
# dnf install plocate
# updatedb
```

Files created after the last `updatedb` run will not appear. Deleted files may still show.

### which, whereis, and type

```console
$ which python3
/usr/bin/python3

$ whereis nginx
nginx: /usr/sbin/nginx /etc/nginx /usr/share/nginx /usr/share/man/man8/nginx.8.gz

$ type -a ls
ls is aliased to `ls --color=auto'
ls is /usr/bin/ls
```

| Tool | Best for |
|------|----------|
| `which` | Which program will run for this name? |
| `whereis` | Where are the binary, config, and manual? |
| `type -a` | Is it an alias, function, builtin, or program? (most accurate in Bash) |

### Finding the package that owns a file

**Debian-based:** `dpkg -S /usr/bin/ssh` (installed) or `apt-file search` (any)
**RHEL-based:** `rpm -qf /usr/bin/ssh` or `dnf provides /usr/bin/ssh`
**SUSE-based:** `rpm -qf /usr/bin/ssh` or `zypper search --provides`

## Common pitfalls

- **Unquoted wildcards.** `find . -name *.txt` can fail if the current directory has matching files. Quote it: `-name "*.txt"`.
- **Using `locate` for fresh files.** Run `updatedb` or use `find`.
- **`which` does not see aliases or builtins.** Use `type`.
- **Expensive searches on busy servers.** `find /` generates heavy I/O. Limit the path, use `-xdev`, and run it off-peak or with `ionice -c3 nice`.

## Further reading

- [`find(1)`](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU findutils manual](https://www.gnu.org/software/findutils/manual/html_mono/find.html)
