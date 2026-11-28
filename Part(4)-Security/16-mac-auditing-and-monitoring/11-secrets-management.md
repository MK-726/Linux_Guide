# Secrets Management

## Overview

A **secret** is any credential that grants access: passwords, API keys, tokens, SSH private keys, TLS private keys, database credentials, and encryption keys. Leaked secrets are one of the most common causes of breaches. **Secrets management** is the practice of storing, delivering, rotating, and auditing them so they are not exposed in code, logs, images, or shell history.

## Key concepts

### Where secrets leak

| Location | How it leaks |
|----------|--------------|
| **Source code and Git history** | Committed passwords, `.env` files, keys. Remains in history after deletion |
| **Command-line arguments** | `mysql -pSECRET` is visible to every user in `ps` and in shell history |
| **Environment variables** | Readable in `/proc/PID/environ` by the same user and root, shown by `docker inspect`, `systemctl show`, crash dumps, debug pages |
| **Configuration files** | World-readable files, backups, copies in tickets and chats |
| **Container images** | Secrets baked into layers (`ENV`, `COPY .env`, build arguments) stay in the image history |
| **Logs and error messages** | Debug output with tokens or connection strings |
| **CI/CD systems** | Unmasked variables printed in build logs |
| **Backups and snapshots** | Unencrypted copies |
| **People** | Passwords in chat, wikis, spreadsheets |

### Principles

1. **Never store secrets in code or images.** Inject them at run time.
2. **Least privilege:** each application gets only the secrets it needs, with the smallest permissions.
3. **Short-lived and rotated:** prefer dynamic or time-limited credentials. Rotate regularly and after any suspicion.
4. **Encrypt at rest and in transit**, and control who can read.
5. **Audit access** to secrets (who read what, when).
6. **Separate environments:** different secrets for development, test, and production.
7. **One owner per secret** and a documented rotation process.
8. **Plan for the "secret zero" problem:** how does the first credential reach the application securely? Use platform identity (cloud instance roles, Kubernetes service accounts, TPM, AppRole) rather than another static password.

### Options compared

| Approach | Good for | Notes |
|----------|----------|-------|
| **Files with strict permissions** | Simple hosts | `600`/`640`, owned by the service user or root. Better than environment variables for sensitive data |
| **systemd credentials** | Services on a host | `LoadCredential=`, encrypted credentials, available only to the service in a private directory |
| **Environment variables** | Convenient, widely supported | Easy to leak. Acceptable for low-risk values. Prefer files or credentials for high-value secrets |
| **Secrets manager** (HashiCorp Vault, OpenBao, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) | Production at scale | Central storage, access control, auditing, rotation, **dynamic secrets** |
| **Encrypted config in Git** (SOPS, git-crypt, sealed-secrets, Ansible Vault) | GitOps and infrastructure code | Secrets are encrypted in the repository. Decryption keys are managed separately |
| **Kubernetes Secrets** | Kubernetes | Base64 is **not encryption**. Enable encryption at rest, restrict RBAC, consider External Secrets or CSI drivers |
| **HSM / KMS / TPM** | Master keys, signing keys | Keys never leave the device or service |
| **Password managers** (Bitwarden, 1Password, KeePassXC, `pass`) | **People** | Not a substitute for application secret delivery |

## Commands

### 1. Keep secrets off the command line and out of history

```console
$ read -rs -p "Password: " DBPASS; echo            # read without echo, not saved in history
$ mysql --defaults-extra-file=/etc/myapp/mysql.cnf -e 'SELECT 1'      # credentials in a protected file, not in arguments
$ psql "host=db user=app dbname=app passfile=/etc/myapp/.pgpass"
$ chmod 600 ~/.pgpass ~/.netrc ~/.my.cnf
$ curl --netrc-file /etc/myapp/netrc https://api.example.com/        # not -u user:password
$ export HISTCONTROL=ignorespace                                      # commands starting with a space are not saved
$  some-command --token "$TOKEN"                                      # (leading space) not recorded
```

Verify who can see what:

```console
$ ps -eo pid,user,args | grep -i -E 'password|token|secret'            # secrets in process arguments?
# cat /proc/1234/environ | tr '\0' '\n' | head                         # environment of a process (root or same user)
$ systemctl show myapp -p Environment                                  # variables set in the unit
```

### 2. Files and permissions

```console
# install -d -m 0750 -o root -g myapp /etc/myapp
# install -m 0640 -o root -g myapp /dev/null /etc/myapp/db.secret
# printf '%s' "$DBPASS" > /etc/myapp/db.secret
```

The file is readable by root and the `myapp` group only. Never place secrets in world-readable paths (`/tmp`, web roots, shared NFS directories).

### 3. systemd credentials

```ini
# /etc/systemd/system/myapp.service
[Service]
User=myapp
LoadCredential=dbpassword:/etc/myapp/db.secret
ExecStart=/opt/myapp/bin/myapp --db-password-file=%d/dbpassword
```

`%d` (or `$CREDENTIALS_DIRECTORY`) is a private, memory-backed directory visible only to the service. Encrypt credentials at rest with a host key or the TPM:

```console
# systemd-creds encrypt --name=dbpassword - /etc/credstore.encrypted/dbpassword.cred <<< 'S3cret'
# systemd-creds list
```

```ini
LoadCredentialEncrypted=dbpassword:/etc/credstore.encrypted/dbpassword.cred
```

`SetCredential=name:value` stores a value in the unit file itself (visible to anyone who can read the unit). Use it only for non-sensitive data.

### 4. HashiCorp Vault / OpenBao

```console
$ vault kv put secret/myapp/db username=app password='S3cret'
$ vault kv get secret/myapp/db
$ vault kv get -field=password secret/myapp/db
$ vault policy write myapp - <<'EOF'
path "secret/data/myapp/*" { capabilities = ["read"] }
EOF
$ vault write auth/approle/role/myapp token_policies=myapp token_ttl=1h
```

Applications authenticate with **AppRole**, cloud IAM, or Kubernetes identity and receive **short-lived tokens**. **Dynamic secrets** (the database engine) create a unique username and password per client with a lease that expires automatically, so there is nothing long-lived to steal.

### 5. Cloud secret stores

```console
$ aws secretsmanager get-secret-value --secret-id prod/myapp/db --query SecretString --output text
$ az keyvault secret show --vault-name myvault --name dbpassword --query value -o tsv
$ gcloud secrets versions access latest --secret=dbpassword
```

Give the **instance role / managed identity** permission to read only the needed secrets. Do not store access keys on the instance.

### 6. Encrypted secrets in Git (SOPS with age)

```console
$ age-keygen -o ~/.config/sops/age/keys.txt                     # prints the public key
$ cat .sops.yaml
creation_rules:
  - path_regex: secrets/.*\.yaml$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
$ sops --encrypt --in-place secrets/prod.yaml                   # values are encrypted, keys stay readable
$ sops --decrypt secrets/prod.yaml                              # needs the private key
$ sops secrets/prod.yaml                                        # edit in your editor (decrypts, re-encrypts on save)
```

SOPS also supports AWS KMS, GCP KMS, Azure Key Vault, and PGP. **Ansible Vault** does similar for Ansible (`ansible-vault encrypt vars/secrets.yml`).

### 7. Containers

```console
$ podman secret create db_password - <<< 'S3cret'
$ podman run --secret db_password,type=mount,target=/run/secrets/db_password myimage     # file at /run/secrets/db_password
$ docker build --secret id=npmrc,src=$HOME/.npmrc .             # BuildKit secret mount: not stored in image layers
```

In a Dockerfile: `RUN --mount=type=secret,id=npmrc ...`. **Never** use `ENV PASSWORD=...`, `ARG` for secrets, or `COPY .env` in images. Scan images (`trivy image --scanners secret IMAGE`).

### 8. Find leaked secrets

```console
$ gitleaks detect --source . --verbose                          # scan a repository, including history
$ gitleaks protect --staged                                     # pre-commit: scan staged changes
$ trufflehog git file://. --only-verified
$ git secrets --scan
$ grep -r -I -n -E '(password|passwd|secret|token|api[_-]?key)\s*[:=]' /etc /opt /srv 2>/dev/null | grep -v -E '^Binary' | head
$ find / -xdev -name '.env' -o -name '*.pem' -o -name 'id_rsa' -o -name '.git-credentials' 2>/dev/null
```

Add `gitleaks` or similar to **pre-commit hooks and CI** to block secrets before they reach the repository.

### 9. Rotation and response to a leak

**Rotate regularly** (database passwords, API keys, tokens): automated where possible. **Prefer short-lived credentials.**

**If a secret leaks, treat it as compromised:**

1. **Revoke or rotate it immediately** (this is the real fix).
2. **Find where it was used** and check logs for misuse.
3. **Remove it from code and history** (`git filter-repo`, BFG). This is hygiene only. Assume it was already copied.
4. **Find the cause** (how did it leak?) and fix the process.
5. Notify the owners and, if required, security and compliance teams.

### Practical checklist for a service on a host

1. Run as a dedicated user.
2. Store secrets in `/etc/myapp/` (`root:myapp`, `0640`), or use `LoadCredential`.
3. No secrets in unit files, command lines, or world-readable paths.
4. Secrets are **not** in Git, images, or tickets.
5. Backups are encrypted.
6. Rotation procedure documented and tested.
7. Monitoring and alerts for unusual secret access.

## Common pitfalls

- **Committing `.env` files or keys**, and believing deleting the file fixes it. History keeps it. **Rotate.**
- **Passing secrets as arguments** (`-p`, `--password`, `--token`) visible in `ps`.
- **Treating environment variables as secure.** They leak through `/proc`, container inspection, crash reports, and child processes.
- **Base64 as "encryption"** (Kubernetes Secrets, config files).
- **One shared secret for all environments** and all services.
- **Never rotating**, or having no idea which systems use a secret (so rotation is feared).
- **Secrets in container images or build logs** (`ENV`, `ARG`, `--build-arg`).
- **Overly broad access** to the secrets manager (every service can read every secret).
- **Long-lived cloud access keys on servers** instead of instance roles or workload identity.
- **No audit logs** of secret reads.
- **Storing the decryption key next to the encrypted file** (for example, the SOPS age key in the same repository).
- **Printing secrets in logs** or error pages, or leaving debug mode on.
- **Ignoring humans.** Provide a password manager and a way to share secrets securely, or they use chat and email.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [`systemd.exec(5)`: credentials](https://man7.org/linux/man-pages/man5/systemd.exec.5.html)
- [`systemd-creds(1)`](https://man7.org/linux/man-pages/man1/systemd-creds.1.html)
- [OpenBao documentation](https://openbao.org/docs/)
- [SOPS](https://github.com/getsops/sops)
- [Gitleaks](https://github.com/gitleaks/gitleaks)
- [NIST SP 800-57](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final)
