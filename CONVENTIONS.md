# Conventions Used in This Guide

Read this page once. It explains how every file in the guide is written.

## Prompts

| Prompt | Meaning |
|--------|---------|
| `$` | A command you run as a **normal user** |
| `#` | A command you run as **root** (or with `sudo`) |

Do not type the `$` or `#`. Type only the command after it.

## Callouts

> [!NOTE]
> Extra information that is useful but not required.

> [!IMPORTANT]
> Something you must know to avoid mistakes.

> [!TIP]
> A shortcut or good habit.

> [!WARNING]
> A command or action that can **destroy data** or **break the system**. Read the whole section before you run it.

## Root privileges

Commands that need root are marked with **Requires root** and use the `#` prompt (or `sudo`).

## Distribution-specific behavior

When a command or file differs between distribution families, the guide uses these labels:

| Label | Distributions |
|-------|---------------|
| **Debian-based** | Debian, Ubuntu, Linux Mint |
| **RHEL-based** | RHEL, Fedora, CentOS Stream, Rocky Linux, AlmaLinux |
| **SUSE-based** | SLES, openSUSE |

If there is no label, the command works the same on all of them.

## Example output

Output shown in this guide is **example output**. Your output will differ (names, dates, sizes, versions). The *shape* of the output is what matters.

## File structure

Every topic file follows the same layout:

1. **Overview**: what the topic is and why it matters
2. **Key concepts**: the ideas, in plain language
3. **Commands**: practical examples with example output
4. **Configuration files**: where they live and what they do (when relevant)
5. **Common pitfalls**: mistakes and how to fix them
6. **Distribution notes**: differences between families (when relevant)
7. **Further reading**: official documentation

## Safe practice

- Practice in a **virtual machine** or a disposable cloud instance, not on a production server.
- Take a **snapshot or backup** before changing system files.
- Read the manual page (`man command`) before you run an unfamiliar command with `sudo`.
