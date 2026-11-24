# Access Control Lists (ACLs)

## Overview

Standard permissions have only three classes: owner, group, and others. An **ACL** adds extra, more specific rules to a file or directory, for example: "give user `bob` read access and group `auditors` read-only access, without changing the owner or main group."

Use ACLs when normal permissions are too limited. For simple cases, regular groups are easier to understand and audit.

## Key concepts

- An ACL consists of **entries**, each naming a user or group and the permissions it gets.
- A **mask** entry sets the **maximum** permissions for all named users, named groups, and the owning group.
- A **default ACL** on a directory is copied to new files and directories created inside it.
- Files with an ACL show a `+` after the permission string in `ls -l`.
- Modern filesystems (ext4, XFS, Btrfs) support ACLs by default. NFS and some others may not.

```console
$ ls -l report.txt
-rw-rw-r--+ 1 alice alice 120 Mar  3 12:00 report.txt
```

### Entry types

| Entry | Example | Meaning |
|-------|---------|---------|
| `user::rw-` | | The file owner |
| `user:bob:r--` | | Named user `bob` |
| `group::r--` | | The owning group |
| `group:auditors:r--` | | Named group `auditors` |
| `mask::rw-` | | Maximum for named users/groups and the owning group |
| `other::---` | | Everyone else |
| `default:...` | `default:group:dev:rwx` | Inherited by new items in the directory |

## Commands

Tools: `getfacl` (view) and `setfacl` (change). Install the `acl` package if missing.

### View

```console
$ getfacl report.txt
# file: report.txt
# owner: alice
# group: alice
user::rw-
user:bob:r--
group::r--
group:auditors:r--
mask::r--
other::---
```

### Set

| Command | Meaning |
|---------|---------|
| `setfacl -m u:bob:rw file` | Modify: give `bob` read and write |
| `setfacl -m g:auditors:r file` | Give group `auditors` read |
| `setfacl -x u:bob file` | Remove the entry for `bob` |
| `setfacl -b file` | Remove **all** ACL entries |
| `setfacl -R -m u:bob:rX dir` | Apply recursively (`X` = execute only for directories) |
| `setfacl -d -m g:dev:rwx dir` | Set a **default** ACL on a directory |
| `setfacl -k dir` | Remove the default ACL |
| `setfacl -m m::r file` | Set the mask |

```console
$ setfacl -m u:bob:rw report.txt
$ setfacl -m g:auditors:r report.txt
$ getfacl --omit-header report.txt
user::rw-
user:bob:rw-
group::r--
group:auditors:r--
mask::rw-
other::---
```

### Shared directory with default ACL

```console
# mkdir /srv/project
# setfacl -m g:dev:rwx /srv/project            # access to the directory itself
# setfacl -d -m g:dev:rwx /srv/project         # inherited by new files and directories
# getfacl /srv/project
# file: srv/project
# owner: root
# group: root
user::rwx
group::r-x
group:dev:rwx
mask::rwx
other::r-x
default:user::rwx
default:group::r-x
default:group:dev:rwx
default:mask::rwx
default:other::r-x
```

Files that exist before you set the default ACL are not changed. Apply the normal ACL recursively with `-R`.

### Backup and restore ACLs

```console
$ getfacl -R /srv/project > project.acl
$ setfacl --restore=project.acl
```

### The mask explained

The mask limits what named users, named groups, and the owning group can actually get. This is the **effective** permission:

```console
$ getfacl report.txt
user:bob:rw-         #effective:r--
mask::r--
```

`bob` was granted `rw`, but the mask is `r--`, so his effective access is read-only.

`chmod` on a file with an ACL changes the **mask** when you change the group permissions. If an ACL entry seems to have no effect, check the mask and run `setfacl -m m::rwx file` to correct it.

## Common pitfalls

- **ACL entries have no effect.** The mask is too restrictive. Check `getfacl` for `#effective:`.
- **Missing `x` on the parent directory.** ACLs on a file do not help if the user cannot traverse the directory path. Use `namei -l` and give directory `x` permission.
- **Default ACLs affect only new items.** Existing files keep their previous ACL.
- **Copying and archiving.** `cp -a` and `rsync -A` preserve ACLs. `tar` needs `--acls`. Otherwise they are lost.
- **Hard-to-find access rules.** Files with `+` are easy to overlook. Search with `getfacl -R -s -p /path` (shows only items with non-trivial ACLs).
- **Overusing ACLs** makes systems difficult to audit. Prefer groups when they are enough.

## Further reading

- [`acl(5)`](https://man7.org/linux/man-pages/man5/acl.5.html)
- [`setfacl(1)`](https://man7.org/linux/man-pages/man1/setfacl.1.html)
- [`getfacl(1)`](https://man7.org/linux/man-pages/man1/getfacl.1.html)
