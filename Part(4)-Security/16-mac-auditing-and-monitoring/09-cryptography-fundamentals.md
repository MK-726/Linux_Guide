# Cryptography Fundamentals

## Overview

Cryptography protects **confidentiality** (only the right people read data), **integrity** (data is not altered), and **authenticity** (data comes from who it claims). Linux administrators use it every day: SSH, TLS, disk encryption, package signatures, password hashes, and VPNs. You do not need the mathematics, but you need to know **which tool does what**, **which algorithms are safe**, and **where keys go wrong**.

## Key concepts

### Building blocks

| Primitive | Purpose | Keys | Examples (current recommended) |
|-----------|---------|------|-------------------------------|
| **Hash function** | A fixed-size fingerprint of data. One-way. Any change gives a different hash | None | **SHA-256, SHA-512, SHA-3, BLAKE2/3**. Avoid MD5 and SHA-1 |
| **MAC / HMAC** | A hash with a secret key: proves integrity and origin | One shared secret | HMAC-SHA-256 |
| **Symmetric encryption** | Encrypts and decrypts with the **same** key. Fast | Shared key | **AES-256-GCM, ChaCha20-Poly1305** |
| **Asymmetric (public-key) encryption** | **Public key** encrypts, **private key** decrypts | Key pair | RSA 3072+, ECC (X25519, P-256) |
| **Digital signature** | **Private key** signs, **public key** verifies. Proves who signed and that data is unchanged | Key pair | Ed25519, ECDSA P-256, RSA-PSS 3072+ |
| **Key exchange** | Two parties agree on a shared key over an open channel | Ephemeral key pairs | ECDH (X25519), (EC)DHE |
| **Key derivation / password hashing** | Slow, salted hashing for passwords and for turning passwords into keys | Password + salt | **Argon2id, scrypt, yescrypt, bcrypt, PBKDF2** |
| **Random numbers** | Unpredictable values for keys, nonces, salts | — | Kernel RNG: `/dev/urandom`, `getrandom()` |

### How they combine (hybrid encryption)

Public-key operations are slow, so real systems use both:

```
1. Key exchange or public-key encryption establishes a random session key.
2. Symmetric encryption (AES-GCM / ChaCha20) protects the actual data with that key.
3. A signature or MAC proves authenticity and integrity.
```

TLS, SSH, GPG, and IPsec all work like this.

### Authenticated encryption

Encryption alone does not detect tampering. Use **authenticated encryption (AEAD)** modes such as **GCM** and **ChaCha20-Poly1305**, which encrypt **and** verify integrity. Avoid **ECB** mode (reveals patterns), and treat plain **CBC** without a MAC as unsafe for new designs.

> [!IMPORTANT]
> **Never reuse a nonce/IV with the same key** in GCM or ChaCha20. It can destroy confidentiality and authenticity.

### Where encryption is needed

| State | Meaning | Linux tools |
|-------|---------|-------------|
| **In transit** | Data moving across a network | TLS, SSH, WireGuard, IPsec |
| **At rest** | Data stored on disk or backups | LUKS/dm-crypt, `fscrypt`, GPG/`age`, database encryption, encrypted backups |
| **In use** | Data being processed | Confidential computing (advanced). Usually not available |

### Trust and keys

The hardest part is not the algorithm. It is **keys and trust**:

- **Generating** keys with good randomness.
- **Storing** private keys safely (permissions, HSM, TPM, secrets manager).
- **Distributing** public keys so you know **whose** key it is (certificates, fingerprints, web of trust).
- **Rotating and revoking** keys.
- **Backing up** keys, so encrypted data is not lost.

### Algorithm status (guidance, check current recommendations)

| Use | Recommended | Avoid |
|-----|-------------|-------|
| Hash | SHA-256, SHA-384/512, SHA-3, BLAKE2 | MD5, SHA-1 (broken for signatures) |
| Symmetric | AES-128/256-GCM, ChaCha20-Poly1305 | DES, 3DES, RC4, ECB mode |
| Signatures | Ed25519, ECDSA P-256/P-384, RSA-PSS ≥ 3072 | RSA < 2048, DSA, SHA-1 signatures |
| Key exchange | X25519, ECDH P-256/P-384, DH ≥ 2048 | Static RSA key exchange, DH < 2048 |
| Password hashing | Argon2id, scrypt, yescrypt, bcrypt | Unsalted or fast hashes (MD5, SHA-1, plain SHA-256) |
| TLS | TLS 1.3, TLS 1.2 with AEAD ciphers | SSL 2/3, TLS 1.0/1.1 |

**Post-quantum cryptography:** large quantum computers could break RSA and elliptic curves. NIST has standardized post-quantum algorithms (ML-KEM for key exchange, ML-DSA and SLH-DSA for signatures). Recent OpenSSH and TLS libraries already offer **hybrid** key exchange (classical + post-quantum) by default or as an option. Keep software updated and watch vendor guidance. Data that must stay secret for decades is at risk of "harvest now, decrypt later".

### FIPS mode

Some environments (government, finance) require **FIPS 140-validated** cryptography. Distributions provide a FIPS mode that restricts algorithms (`fips-mode-setup --enable` on RHEL-based systems; Ubuntu Pro offers FIPS-certified modules). It can break software that uses non-approved algorithms. Enable only when required and test first.

```console
$ fips-mode-setup --check            # RHEL-based
$ cat /proc/sys/crypto/fips_enabled  # 1 = enabled
```

## Commands

### Random data and hashes

```console
$ openssl rand -hex 32                          # 256-bit random value in hex (for secrets, tokens)
$ openssl rand -base64 24
$ head -c 32 /dev/urandom | base64              # also fine
$ sha256sum file.iso                            # hash a file
$ sha512sum file.iso
$ echo -n "hello" | sha256sum
$ openssl dgst -sha256 file.iso                 # same through OpenSSL
$ openssl dgst -sha256 -hmac "secretkey" message.txt      # HMAC (do not use real secrets on the command line!)
$ b2sum file.iso                                # BLAKE2
```

Verify a download against a published checksum (see [File Comparison and Inspection](../../part-1-foundations/03-filesystem-and-file-management/05-file-comparison-and-inspection.md)):

```console
$ sha256sum -c SHA256SUMS --ignore-missing
```

### Key pairs with OpenSSL

```console
$ openssl genpkey -algorithm ED25519 -out ed25519.key           # private key
$ openssl pkey -in ed25519.key -pubout -out ed25519.pub         # public key
$ openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out rsa.key
$ openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out ec.key
$ chmod 600 *.key
$ openssl pkey -in rsa.key -noout -text | head -3            # inspect
```

Sign and verify a file with Ed25519:

```console
$ openssl pkeyutl -sign -rawin -inkey ed25519.key -in message.txt -out message.sig
$ openssl pkeyutl -verify -rawin -pubin -inkey ed25519.pub -in message.txt -sigfile message.sig
Signature Verified Successfully
```

### Encrypt files

| Tool | Best for |
|------|----------|
| **GnuPG (`gpg`)** | Encrypt/sign files and mail, verify software signatures, long-standing and widely supported |
| **`age`** | Simple modern file encryption with small keys (recommended for new scripts and secrets in repositories) |
| **`openssl enc`** | Quick tests only. It lacks authentication (use `-pbkdf2`) |
| **LUKS / `cryptsetup`** | Whole block devices ([Secure Boot Overview](../14-security-fundamentals/07-secure-boot-overview.md)) |
| **`ssh-keygen -Y sign`** | Sign files using SSH keys |

**age:**

```console
$ age-keygen -o key.txt                               # prints the public key: age1...
Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
$ age -r age1ql3z... -o secrets.txt.age secrets.txt   # encrypt for a recipient
$ age -d -i key.txt secrets.txt.age > secrets.txt     # decrypt
$ age -p -o file.age file                              # passphrase-based
```

**GnuPG:**

```console
$ gpg --full-generate-key                              # create a key pair (choose ECC/Curve 25519, set expiry and a passphrase)
$ gpg --list-keys ; gpg --list-secret-keys
$ gpg --fingerprint alice@example.com                  # compare fingerprints out-of-band before trusting
$ gpg --armor --export alice@example.com > alice.pub.asc
$ gpg --import bob.pub.asc
$ gpg --encrypt --recipient bob@example.com --armor report.pdf        # creates report.pdf.asc
$ gpg --decrypt report.pdf.asc > report.pdf
$ gpg --symmetric --cipher-algo AES256 file.txt        # passphrase encryption: file.txt.gpg
$ gpg --detach-sign --armor release.tar.gz             # signature file: release.tar.gz.asc
$ gpg --verify release.tar.gz.asc release.tar.gz       # verify a download
gpg: Good signature from "Project Release Key <release@example.org>"
gpg: WARNING: This key is not certified with a trusted signature!   ← you must verify the fingerprint yourself
$ gpg --gen-revoke alice@example.com > revoke.asc      # make a revocation certificate NOW and store it safely
```

Keep a **backup** of private keys and revocation certificates in a secure, offline place.

**openssl enc** (quick test only):

```console
$ openssl enc -aes-256-cbc -pbkdf2 -salt -in secret.txt -out secret.enc      # asks for a passphrase
$ openssl enc -d -aes-256-cbc -pbkdf2 -in secret.enc -out secret.txt
```

Without `-pbkdf2` the key derivation is weak. `enc` does not authenticate data, so use `age` or `gpg` for real work.

### See what algorithms the system supports

```console
$ openssl version
$ openssl list -cipher-algorithms | head
$ openssl ciphers -v 'HIGH:!aNULL' | head           # TLS cipher suites
$ openssl speed -evp aes-256-gcm                    # benchmark (hardware acceleration check)
$ cryptsetup benchmark                              # disk encryption speed
$ grep -m1 -o -w aes /proc/cpuinfo                  # AES-NI present?
$ cat /proc/crypto | grep -E '^name' | sort -u | head
$ gpg --version | tail -5
```

## Common pitfalls

- **"Rolling your own" cryptography.** Use established libraries and tools. Do not invent algorithms or protocols.
- **Using broken or weak algorithms** (MD5, SHA-1, DES, RC4, SSLv3, RSA 1024).
- **Encryption without integrity.** Use AEAD modes. Use `age`/`gpg` instead of raw `openssl enc`.
- **Poor randomness.** Use the kernel RNG (`/dev/urandom`, `openssl rand`). Never `$RANDOM` or timestamps for secrets.
- **Weak passphrases protecting strong keys.** A key is only as strong as its passphrase and storage.
- **Losing keys.** Encrypted data without the key is gone. Back up keys, with recovery procedures.
- **Leaving private keys world-readable** or in repositories, images, and tickets.
- **Trusting a public key without verifying its fingerprint.** Check through a second channel.
- **Never rotating keys** or having no plan for revocation.
- **Password hashing with fast hashes.** Use Argon2id/yescrypt/bcrypt with salt for passwords.
- **Confusing encoding with encryption.** Base64 and hex are **not** encryption.
- **Passing secrets as command-line arguments** (visible to other users with `ps`).
- **Ignoring certificate validation** (`-k`, `--no-check-certificate`). It removes the authenticity protection from TLS.
- **Assuming "encrypted" means "safe" at every stage.** Data decrypted in memory, logs, and backups can still leak.

## Further reading

- [NIST SP 800-57: Key Management Recommendations](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final)
- [NIST Post-Quantum Cryptography](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [`openssl(1)`](https://docs.openssl.org/master/man1/openssl/)
- [GnuPG documentation](https://gnupg.org/documentation/)
- [age](https://age-encryption.org)
- [Mozilla: Server Side TLS guidelines](https://wiki.mozilla.org/Security/Server_Side_TLS)
