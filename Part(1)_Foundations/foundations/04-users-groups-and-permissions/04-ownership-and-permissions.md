# File Ownership and Permissions

## Overview

Every file and directory has an **owner**, a **group**, and a set of **permissions**. The kernel checks these every time a process accesses a file. This is called **Discretionary Access Control (DAC)**: the owner decides who gets access.

## Key concepts

### Reading `ls -l`

```console
$ ls -l script.sh
-rwxr-xr-- 1 alice developers 220 Mar  3 10:00 script.sh
```

| Part | Value | Meaning |
|------|-------|---------|
| `-` | File type | `-` regular file, `d` directory, `l` link |
| `rwx` | Owner (user) permissions | Owner can read, write, execute |
| `r-x` | Group permissions | Group can read and execute |
| `r--` | Others permissions | Everyone else can only read |
| `1` | Link count | |
| `alice` | Owner | |
| `developers` | Group | |

### What r, w, x mean

| Permission | On a **file** | On a **directory** |
|------------|---------------|--------------------|
| **r** (read) | View the contents | List the names inside |
| **w** (write) | Change the contents | Create, rename, and delete entries (needs `x` too) |
| **x** (execute) | Run the file as a program | Enter the directory (`cd`) and access items inside |

> [!IMPORTANT]
> To reach a file you need `x` on **every directory** in its path. To **delete** a file you need `w` and `x` on the **directory**, not on the file itself.

### Who is checked

The kernel picks **one** class, in this order, and uses only that class:

1. If you are the **owner** → owner permissions.
2. Else if you are in the file's **group** → group permissions.
3. Else → **others** permissions.

`root` bypasses these checks (with a few exceptions, such as when `x` is missing on a regular file).

### Octal (numeric) notation

Each permission has a value: **r = 4**, **w = 2**, **x = 1**. Add them for each class.

| Octal | Symbolic | Meaning |
|-------|----------|---------|
| 7 | `rwx` | Read, write, execute |
| 6 | `rw-` | Read, write |
| 5 | `r-x` | Read, execute |
| 4 | `r--` | Read only |
| 0 | `---` | None |

| Mode | Symbolic | Typical use |
|------|----------|-------------|
| `644` | `rw-r--r--` | Normal files |
| `600` | `rw-------` | Private files (SSH private keys) |
| `755` | `rwxr-xr-x` | Programs and directories |
| `700` | `rwx------` | Private directories |
| `664` | `rw-rw-r--` | Group-writable files |
| `775` | `rwxrwxr-x` | Group-writable directories |
| `640` | `rw-r-----` | Config files with secrets, group can read |

## Commands

### chmod: change permissions

Numeric form:

```console
$ chmod 644 notes.txt
$ chmod 755 script.sh
$ chmod -R 750 /srv/app            # recursive
```

Symbolic form: `chmod WHO OPERATOR PERMISSION`

| Part | Values |
|------|--------|
| Who | `u` user, `g` group, `o` others, `a` all |
| Operator | `+` add, `-` remove, `=` set exactly |
| Permission | `r`, `w`, `x`, `X`, `s`, `t` |

```console
$ chmod u+x script.sh              # add execute for the owner
$ chmod g-w,o-rwx file             # remove group write and all of others
$ chmod a+r file                   # everyone can read
$ chmod u=rw,g=r,o= file           # exactly 640
$ chmod -R u+rwX,g+rX,o-rwx dir    # X: execute only on directories (and files already executable)
```

The capital `X` is very useful with `-R`: it adds `x` to directories (so you can enter them) but not to ordinary files.

### chown: change owner and group

**Requires root** to change the owner. The owner can change the group to a group they belong to.

```console
# chown alice file                  # change owner
# chown alice:developers file       # change owner and group
# chown :developers file            # change group only
# chown -R alice:developers /srv/app
# chown -h alice link               # change the link itself, not its target
```

### chgrp: change group

```console
$ chgrp developers file
# chgrp -R www-data /var/www/site
```

### Show permissions

```console
$ ls -l file
$ ls -ld /srv/app                  # the directory itself
$ stat -c '%A %a %U:%G %n' file
-rw-r--r-- 644 alice:alice file
$ namei -l /srv/app/config/secret.txt     # permissions of every path component
```

`namei -l` is a very good tool for "Permission denied" problems: it shows each directory in the path.

### Typical examples

```console
# Private SSH key and directory
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/id_ed25519

# Web content: owned by deploy user, readable by the web server group
# chown -R deploy:www-data /var/www/site
# find /var/www/site -type d -exec chmod 750 {} +
# find /var/www/site -type f -exec chmod 640 {} +

# Shared project directory for a group
# mkdir /srv/project
# chown root:developers /srv/project
# chmod 2775 /srv/project            # 2 = SGID; new files inherit the group
```

## Common pitfalls

- **`chmod 777` as a "fix".** It lets everyone modify the file. Find the real cause with `namei -l` and `id`.
- **`chmod -R 755` on a tree** makes every file executable. Use the `find -type d` / `find -type f` pattern or `chmod -R u+rwX,go+rX`.
- **Missing `x` on a parent directory** gives "Permission denied" even when the file itself is readable.
- **Recursive `chown` or `chmod` on `/` or a system path** can break the system. Double-check the path.
- **Group changes not applied.** After adding a user to a group, they must log in again.
- **Scripts that will not run.** Check the `x` bit, the `#!` line, and whether the filesystem is mounted `noexec`.

> [!WARNING]
> `chown -R` and `chmod -R` change many files in one command and are hard to undo. Test on one directory first, and save the current state with `getfacl -R path > permissions.backup` or `ls -lR`.

## Further reading

- [`chmod(1)`](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [`chown(1)`](https://man7.org/linux/man-pages/man1/chown.1.html)
- [`path_resolution(7)`](https://man7.org/linux/man-pages/man7/path_resolution.7.html)
