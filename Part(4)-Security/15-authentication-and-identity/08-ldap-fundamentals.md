# LDAP Fundamentals

## Overview

**LDAP** (Lightweight Directory Access Protocol) is a protocol to **read and update a directory**: a hierarchical database optimized for many reads of small records, such as users, groups, hosts, and certificates. Organizations use directories as the central source of identity. Active Directory, FreeIPA, OpenLDAP, and 389 Directory Server all speak LDAP. Linux hosts query them (usually through [SSSD](10-sssd-and-centralized-identity-management.md)) to find users and verify passwords.

## Key concepts

### The directory tree (DIT)

Data is organized as a tree of **entries**. Each entry is identified by a **Distinguished Name (DN)**, read from the entry up to the root:

```
dc=example,dc=com                         ← base (suffix) of the directory
├── ou=people
│   ├── uid=alice,ou=people,dc=example,dc=com        ← DN of Alice's entry
│   └── uid=bob,ou=people,dc=example,dc=com
├── ou=groups
│   └── cn=developers,ou=groups,dc=example,dc=com
└── ou=hosts
```

| Term | Meaning | Example |
|------|---------|---------|
| **Entry** | One object | A user |
| **DN** | Unique full path of an entry | `uid=alice,ou=people,dc=example,dc=com` |
| **RDN** | The first part of the DN (relative name) | `uid=alice` |
| **Base DN / suffix** | The top of a subtree or the directory | `dc=example,dc=com` |
| **Attribute** | A named value of an entry (can have several values) | `mail: alice@example.com` |
| **objectClass** | Defines which attributes an entry must/may have | `inetOrgPerson`, `posixAccount` |
| **Schema** | The set of all objectClasses and attributes | RFC 2307 (POSIX), RFC 4519 |
| **LDIF** | Plain-text format for entries and changes | See below |

Common name components: `dc` (domain component), `ou` (organizational unit), `cn` (common name), `uid` (user ID), `o`, `c`.

### Typical user entry (LDIF)

```
dn: uid=alice,ou=people,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: alice
cn: Alice Smith
sn: Smith
givenName: Alice
mail: alice@example.com
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/alice
loginShell: /bin/bash
userPassword: {SSHA512}...
```

| Attribute | Used by Linux as |
|-----------|------------------|
| `uid` | Login name |
| `uidNumber`, `gidNumber` | UID and primary GID |
| `homeDirectory`, `loginShell` | Home and shell |
| `userPassword` | Password hash (never stored in plain text) |
| `memberUid` / `member` | Group membership |
| `sshPublicKey` | SSH key (with `ldapPublicKey` schema) |
| `sudoRole` entries | Central sudo rules |

> [!NOTE]
> **Active Directory** uses different attribute names (`sAMAccountName`, `userPrincipalName`, `objectSid`, `member`) and optional POSIX attributes (`uidNumber`, `gidNumber`, `unixHomeDirectory`, `loginShell`). See [Active Directory Integration](09-active-directory-integration.md).

### LDAP operations

| Operation | Meaning |
|-----------|---------|
| **Bind** | Authenticate to the server (anonymous, simple with DN+password, SASL such as GSSAPI/Kerberos, EXTERNAL with TLS certificates) |
| **Search** | Find entries by base, scope, and filter |
| **Add / Modify / Delete / ModDN** | Change entries |
| **Compare** | Test an attribute value |
| **StartTLS** | Upgrade an existing connection to TLS |
| **Unbind** | Close the connection |

**Password check by a client:** the client searches for the user's DN, then **binds as that DN** with the supplied password. A successful bind means the password is right.

### Search filters

A search has a **base DN**, a **scope** (`base`, `one` level, `sub` tree), and a **filter** in prefix notation:

| Filter | Meaning |
|--------|---------|
| `(uid=alice)` | Equality |
| `(uid=a*)` | Substring |
| `(&(objectClass=posixAccount)(uidNumber>=10000))` | AND |
| `(\|(uid=alice)(uid=bob))` | OR |
| `(!(loginShell=/sbin/nologin))` | NOT |
| `(memberOf=cn=developers,ou=groups,dc=example,dc=com)` | Group membership (when the server provides `memberOf`) |

### Ports and security

| Port | Protocol |
|------|----------|
| **389/tcp** | LDAP, with **StartTLS** to encrypt |
| **636/tcp** | **LDAPS** (TLS from the start) |
| 3268, 3269 | AD Global Catalog (plain / TLS) |

> [!WARNING]
> Plain LDAP on port 389 without TLS sends **passwords in clear text**. Always use LDAPS or StartTLS, validate the server certificate, and disable anonymous binds.

### Directory server software

| Software | Notes |
|----------|-------|
| **OpenLDAP** (`slapd`) | The classic open-source server. Configuration in `cn=config` (online) |
| **389 Directory Server** (Red Hat DS) | Rich features, multi-supplier replication, web console |
| **FreeIPA / Red Hat IdM** | 389-DS + Kerberos + CA + DNS + HBAC + sudo rules, with a Linux-focused management tool |
| **Active Directory** | Microsoft's directory (LDAP + Kerberos + DNS) |
| **Apache DS, OpenDJ/Ping, Authentik, Keycloak (with LDAP federation), lldap** | Others |

### How Linux uses LDAP

```
Application → NSS (getent/id)  ┐
            → PAM (login/ssh)  ┼─► SSSD ─► LDAP/AD/IPA server (TLS)
            → sudo, ssh keys   ┘
```

Older direct clients (`nss_ldap`, `nslcd` / `nss-pam-ldapd`) still exist. **SSSD is the recommended client.**

## Commands

Install the client tools: **Debian-based** `apt install ldap-utils`, **RHEL-based** `dnf install openldap-clients`.

### Client defaults

```
# /etc/ldap/ldap.conf (Debian-based) or /etc/openldap/ldap.conf (RHEL-based)
BASE    dc=example,dc=com
URI     ldaps://ldap1.example.com ldaps://ldap2.example.com
TLS_CACERT /etc/ssl/certs/ca-certificates.crt
TLS_REQCERT demand
```

### Search

```console
$ ldapsearch -x -H ldaps://ldap1.example.com -b "dc=example,dc=com" "(uid=alice)"
$ ldapsearch -x -H ldap://ldap1.example.com -ZZ -b "ou=people,dc=example,dc=com" "(uid=alice)" cn mail uidNumber    # StartTLS, selected attributes
$ ldapsearch -x -H ldaps://ldap1.example.com -D "uid=svc-lookup,ou=services,dc=example,dc=com" -W \
      -b "dc=example,dc=com" "(&(objectClass=posixAccount)(uid=alice))"
dn: uid=alice,ou=people,dc=example,dc=com
cn: Alice Smith
uidNumber: 10001
# numResponses: 2
# numEntries: 1
```

| Option | Meaning |
|--------|---------|
| `-x` | Simple authentication (not SASL) |
| `-H URI` | Server (`ldap://`, `ldaps://`, `ldapi://` for a Unix socket) |
| `-ZZ` | Require StartTLS |
| `-D DN` | Bind DN (the account to authenticate as) |
| `-W` | Prompt for the password (never use `-w password` on a shared system) |
| `-b BASE` | Search base |
| `-s base`, `-s one`, `-s sub` | Search scope |
| `-LLL` | Clean LDIF output |
| trailing names | Return only these attributes |

```console
$ ldapwhoami -x -H ldaps://ldap1.example.com -D "uid=alice,ou=people,dc=example,dc=com" -W      # check a login
dn:uid=alice,ou=people,dc=example,dc=com
$ ldapsearch -LLL -x -H ldaps://ldap1.example.com -b "ou=groups,dc=example,dc=com" "(cn=developers)" memberUid
```

### Modify (administrators)

```console
$ cat add-user.ldif
dn: uid=carol,ou=people,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
uid: carol
cn: Carol Jones
sn: Jones
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/carol
loginShell: /bin/bash

$ ldapadd -x -H ldaps://ldap1.example.com -D "cn=admin,dc=example,dc=com" -W -f add-user.ldif
$ ldappasswd -x -H ldaps://ldap1.example.com -D "cn=admin,dc=example,dc=com" -W -S "uid=carol,ou=people,dc=example,dc=com"
$ cat mod.ldif
dn: uid=carol,ou=people,dc=example,dc=com
changetype: modify
replace: loginShell
loginShell: /bin/zsh
$ ldapmodify -x -H ldaps://ldap1.example.com -D "cn=admin,dc=example,dc=com" -W -f mod.ldif
$ ldapdelete -x -H ldaps://ldap1.example.com -D "cn=admin,dc=example,dc=com" -W "uid=carol,ou=people,dc=example,dc=com"
```

### Test the TLS connection

```console
$ openssl s_client -connect ldap1.example.com:636 -showcerts </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
$ LDAPTLS_CACERT=/etc/pki/ca-trust/source/anchors/corp-ca.pem ldapsearch -x -H ldaps://ldap1.example.com -b "" -s base
```

### Server-side notes (OpenLDAP)

```console
# ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config "(olcDatabase=*)" olcSuffix olcAccess     # read configuration as root
# slapcat -n 1 > backup.ldif                                                                    # backup the main database
# slapadd -n 1 -l backup.ldif                                                                   # restore (stop slapd first)
```

Design guidelines:

- Use **access controls (ACLs)** so users can read only what they need and change only their own password.
- Hash passwords with a strong scheme (`{SSHA512}`, `{PBKDF2-SHA512}`, or Argon2 when available). Never plain text.
- Create a **read-only service account** for lookups (not the admin DN).
- **Replicate** across at least two servers (multi-provider/multi-master or provider-consumer).
- Index frequently searched attributes (`uid`, `uidNumber`, `memberUid`, `cn`).
- Back up regularly and test restores.
- Allocate **UID/GID ranges** centrally.

## Common pitfalls

- **Plain LDAP without TLS.** Passwords are exposed. Use LDAPS/StartTLS and verify certificates (`TLS_REQCERT demand`).
- **Anonymous binds** allowing enumeration of users. Disable them.
- **Using the admin DN** for application lookups. Use a limited service account.
- **`ldapsearch -w PASSWORD`** leaving the password in history and process lists. Use `-W` or `-y file`.
- **Wrong base DN or scope** returns "No such object" or nothing. Check with `ldapsearch -x -b "" -s base namingContexts`.
- **Size limits and paging.** Large directories return "Size limit exceeded". Use paged results or narrower filters.
- **Certificate name mismatch.** The server certificate must have the host name used in the URI as a Subject Alternative Name.
- **UID collisions** with local accounts or between systems.
- **No replication.** The directory is a critical single point of failure for logins.
- **Treating LDAP as a general database.** It is for read-heavy identity data.
- **Mixed-up attribute sets** when migrating between AD, OpenLDAP, and FreeIPA (different schemas and attribute names).

## Further reading

- [`ldapsearch(1)`](https://man7.org/linux/man-pages/man1/ldapsearch.1.html)
- [OpenLDAP Administrator's Guide](https://www.openldap.org/doc/admin26/)
- [RFC 4511: LDAP protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [RFC 2307: LDAP as a Network Information Service](https://datatracker.ietf.org/doc/html/rfc2307)
- [389 Directory Server](https://www.port389.org)
- [FreeIPA](https://www.freeipa.org)
