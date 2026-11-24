# Open Source and Licensing

## Overview

Linux software is distributed under licenses that let you use, study, change, and share it. Knowing the main license types helps you use software legally in your company and understand your obligations.

## Key concepts

### Free software vs. open source

Two names describe overlapping ideas:

| Term | Focus |
|------|-------|
| **Free software** (Free Software Foundation) | User freedoms: run, study, change, and share the software. "Free" means freedom, not price. |
| **Open source** (Open Source Initiative) | A development and licensing model based on open access to source code. |

Most Linux software meets both definitions.

### The four freedoms of free software

0. Run the program for any purpose.
1. Study how it works and change it (needs source code).
2. Redistribute copies.
3. Distribute your modified versions.

### Common license types

| Type | Idea | Examples |
|------|------|----------|
| **Copyleft (strong)** | If you distribute a modified version, you must release it under the same license | GPL v2, GPL v3 |
| **Copyleft (weak)** | Changes to the library itself must stay open; programs that only link to it need not | LGPL |
| **Network copyleft** | Copyleft also applies when users reach the software over a network | AGPL |
| **Permissive** | Few conditions; you may use the code in closed-source products | MIT, BSD, Apache 2.0 |

### Licenses you will meet

| Software | License |
|----------|---------|
| Linux kernel | GPL version 2 (with a syscall exception for user programs) |
| GNU tools (Bash, coreutils) | GPL |
| glibc | LGPL |
| Nginx | BSD-2-Clause |
| Apache HTTP Server | Apache 2.0 |
| PostgreSQL | PostgreSQL License (permissive) |

> [!NOTE]
> Using GPL software inside your company does not require you to publish your changes. The duty to share source applies when you **distribute** the software to others. Ask your legal team for company policy.

### Trademarks and commercial distributions

The source code of a distribution may be free, but its **name and logo are trademarks**. For example, Red Hat Enterprise Linux (RHEL) source is available under open-source licenses, but you pay for the subscription, support, and certified binaries. Rebuilds such as Rocky Linux and AlmaLinux must remove Red Hat branding.

## Commands

Find the license of an installed package:

**Debian-based**

```console
$ less /usr/share/doc/bash/copyright
```

**RHEL-based**

```console
$ rpm -q --qf '%{NAME}: %{LICENSE}\n' bash
bash: GPL-3.0-or-later
```

## Common pitfalls

- **Treating "free" as "no obligations".** Licenses have conditions, such as keeping copyright notices.
- **Mixing incompatible licenses** in software you distribute. Check compatibility before you combine code.

## Further reading

- [GNU licenses](https://www.gnu.org/licenses/)
- [Open Source Initiative: licenses](https://opensource.org/licenses)
- [SPDX license list](https://spdx.org/licenses/)
