# Text Editors: Vim and Nano

## Overview

Almost all Linux configuration is plain text, so you must be able to edit files in a terminal. Two editors cover most needs:

- **Nano** is simple. Choose it when you want to edit quickly.
- **Vim** is powerful and installed almost everywhere (often as `vi`). Learn at least the basics, because it may be the only editor on a rescue system.

## Key concepts

### Nano

Nano shows its shortcuts at the bottom of the screen. `^` means the Ctrl key.

```console
$ nano /etc/hostname
```

| Keys | Action |
|------|--------|
| `Ctrl+O` then `Enter` | Save (write out) |
| `Ctrl+X` | Exit (asks to save if there are changes) |
| `Ctrl+W` | Search |
| `Ctrl+K` / `Ctrl+U` | Cut line / paste line |
| `Ctrl+G` | Help |
| `Alt+U` | Undo |
| `Ctrl+_` | Go to line number |

### Vim

Vim is **modal**: the keys do different things depending on the mode.

| Mode | How to enter | Purpose |
|------|--------------|---------|
| **Normal** | `Esc` | Move around, delete, copy, and run commands |
| **Insert** | `i` (before cursor), `a` (after), `o` (new line below) | Type text |
| **Visual** | `v` (characters), `V` (lines) | Select text |
| **Command-line** | `:` | Save, quit, search and replace |

> [!TIP]
> If you are lost, press `Esc` twice. You are then in Normal mode.

### Surviving Vim: the minimum

| Task | Keys (in Normal mode) |
|------|----------------------|
| Start editing | `i` |
| Stop editing | `Esc` |
| Save | `:w` then `Enter` |
| Save and quit | `:wq` or `ZZ` |
| Quit without saving | `:q!` |
| Quit (no changes) | `:q` |

### Moving and editing in Vim

| Keys | Action |
|------|--------|
| `h` `j` `k` `l` | Left, down, up, right (arrow keys also work) |
| `w` / `b` | Next / previous word |
| `0` / `$` | Start / end of line |
| `gg` / `G` | First / last line |
| `:42` | Go to line 42 |
| `x` | Delete character |
| `dd` | Delete (cut) line |
| `yy` | Copy (yank) line |
| `p` | Paste after cursor |
| `u` | Undo |
| `Ctrl+r` | Redo |
| `/text` then `n` | Search forward, next match |
| `:%s/old/new/g` | Replace all `old` with `new` in the file |
| `:set number` | Show line numbers |

Counts multiply commands: `3dd` deletes 3 lines, `5j` moves down 5 lines.

### Editing files you do not own

Use `sudoedit` or `sudo -e`. It edits a **copy** with your own editor and then writes it back, which is safer than running the editor as root.

```console
$ sudoedit /etc/hosts
```

### Setting the default editor

```console
$ export EDITOR=nano       # add to ~/.bashrc to keep it
```

**Debian-based:** `sudo update-alternatives --config editor`

## Configuration files

| File | Purpose |
|------|---------|
| `~/.vimrc` | Vim settings |
| `~/.nanorc` | Nano settings |
| `/etc/nanorc` | System-wide Nano settings |

Example `~/.vimrc`:

```vim
set number
set tabstop=4
set shiftwidth=4
set expandtab
syntax on
```

## Common pitfalls

- **Typing in Normal mode** in Vim triggers commands instead of inserting text. Press `i` first.
- **Cannot exit Vim.** Press `Esc`, then type `:q!` and press `Enter` to quit without saving.
- **Swap file warnings.** Vim creates `.filename.swp` files. If you see a warning, another session may have the file open, or a previous session crashed. Check before deleting.
- **Pasting into a terminal editor** can mangle indentation. In Vim use `:set paste` before pasting.
- **Editing as root by habit.** Prefer `sudoedit`.

## Further reading

- Built-in tutorial: run `vimtutor`
- [Vim documentation](https://vimhelp.org)
- [GNU nano manual](https://www.nano-editor.org/docs.php)
