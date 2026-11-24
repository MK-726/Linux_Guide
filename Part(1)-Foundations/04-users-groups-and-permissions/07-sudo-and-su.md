# sudo and su

## Overview

Administrators should not work as root all day. **`sudo`** runs a **single command** with elevated privileges and logs it. **`su`** switches to another user's account. Both let you do administrative work with accountability.

## Key concepts

| Tool | What it does | Authentication | Logging |
|------|--------------|----------------|---------|
| `sudo` | Runs one command as another user (root by default) | **Your own** password | Each command is logged |
| `su` | Starts a shell as another user | The **target user's** password | Only the login is logged |

Prefer `sudo`: it can limit which commands a person may run, does not need a shared root password, and leaves an audit trail.

### Who may use sudo

Membership in an administrative group, or an entry in the sudoers configuration:

| Distribution family | Administrator group |
|--------------------|---------------------|
| Debian-based | `sudo` |
| RHEL-based | `wheel` |
| SUSE-based | `wheel` or explicit entries (SUSE often asks for the **root** password by default unless configured) |

```console
# usermod -aG sudo alice        # Debian-based
# usermod -aG wheel alice       # RHEL-based
```

## Commands

### sudo

| Command | Purpose |
|---------|---------|
| `sudo COMMAND` | Run a command as root |
| `sudo -u USER COMMAND` | Run as another user |
| `sudo -i` | Root login shell (reads root's profile) |
| `sudo -s` | Root shell, keeping your environment |
| `sudo -l` | List what you are allowed to run |
| `sudo -k` | Forget cached credentials |
| `sudo -v` | Refresh the timestamp |
| `sudoedit FILE` | Safely edit a file as root |
| `sudo !!` | Repeat the previous command with `sudo` |

```console
$ sudo systemctl restart nginx
[sudo] password for alice:

$ sudo -u postgres psql -c '\l'

$ sudo -l
User alice may run the following commands on server01:
    (ALL : ALL) ALL
```

By default, `sudo` remembers your authentication for a few minutes (commonly 5 or 15 minutes, depending on the distribution).

Use `sudo` for redirection carefully:

```console
$ sudo echo "x" > /etc/example        # FAILS: redirection runs as you
$ echo "x" | sudo tee /etc/example    # works
$ sudo sh -c 'echo x > /etc/example'  # works
```

### su

```console
$ su - bob            # login shell as bob (loads bob's environment)
$ su bob              # shell as bob, keeping most of your environment
$ su -                # become root with root's environment
$ su -c 'whoami' bob  # run one command as bob
$ exit                # return to the previous user
```

The `-` (or `-l`) matters: it gives a clean login environment, including `PATH` and `HOME`.

### Configuring sudo

Always edit with **`visudo`**, which checks the syntax before saving. A broken sudoers file can lock you out of administrative access.

```console
# visudo                         # edits /etc/sudoers
# visudo -f /etc/sudoers.d/ops   # edits a drop-in file
# visudo -c                      # check syntax of all files
```

**Rule format:**

```
WHO   WHERE=(RUNAS)   WHAT
```

| Part | Example | Meaning |
|------|---------|---------|
| WHO | `alice`, `%developers` | A user, or a group (prefix `%`) |
| WHERE | `ALL` | Hosts the rule applies to |
| RUNAS | `(ALL)`, `(postgres)` | Users the command may run as |
| WHAT | `ALL`, `/usr/bin/systemctl restart nginx` | Allowed commands |

Examples (put in `/etc/sudoers.d/` files):

```
# Full administrator
alice ALL=(ALL:ALL) ALL

# Group of operators; can restart nginx only
%ops ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx

# Run a backup script without a password
backup ALL=(root) NOPASSWD: /usr/local/bin/run-backup.sh
```

Rules for files in `/etc/sudoers.d/`:

- Permissions must be `0440` and owned by root.
- File names must not contain a `.` or end with `~` (those files are skipped).
- The main file must contain `@includedir /etc/sudoers.d` (or `#includedir`); it does by default.

### Logs

| System | Location |
|--------|----------|
| Debian-based | `/var/log/auth.log` |
| RHEL-based | `/var/log/secure` |
| Any systemd system | `journalctl _COMM=sudo` |

```console
# journalctl _COMM=sudo --since today
```

## Common pitfalls

- **Editing `/etc/sudoers` with a normal editor.** A syntax error stops sudo from working. Use `visudo`.
- **`NOPASSWD: ALL`** removes a safety barrier. Use it only for narrow, specific commands.
- **Allowing editors, shells, or interpreters** in sudo rules (`vim`, `less`, `find`, `python`). They let the user escape to a root shell. Prefer `sudoedit`.
- **Wildcards in allowed commands** can be abused. Be specific.
- **Relative command paths in sudoers.** Always use full paths.
- **Environment confusion.** `sudo` resets `PATH` and most variables (`secure_path`). A command that works as root through `su -` may not be found through `sudo`.
- **Sharing the root password** instead of using `sudo`. You lose accountability.

> [!TIP]
> Before logging out after changing sudo or group settings, open a **second session** and test `sudo -v`. Keep the first session open as a safety net.

Chapter 15 covers `sudoers` best practices in depth.

## Further reading

- [`sudo(8)`](https://man7.org/linux/man-pages/man8/sudo.8.html) and [sudoers manual](https://www.sudo.ws/docs/man/sudoers.man/)
- [`su(1)`](https://man7.org/linux/man-pages/man1/su.1.html)
