# SSH & Encryption

---

## Symmetric vs Asymmetric Encryption

**Symmetric:** one shared key encrypts and decrypts. Fast. Used for bulk data transfer.

```
Alice encrypt("hello", key=K) → ciphertext
Bob   decrypt(ciphertext, key=K) → "hello"
```

Problem: how do Alice and Bob agree on K without an eavesdropper intercepting it?

**Asymmetric:** mathematically linked key pair. What one key encrypts, only the other can decrypt.

```
Public key  → share freely, anyone can have it
Private key → never leaves your machine, ever
```

Two operations, opposite key directions:

| Operation | Key used | What it proves |
|-----------|----------|----------------|
| Encrypt for confidentiality | Recipient's **public** key | Only recipient (private key holder) can read it |
| Sign for authenticity | Sender's **private** key | Anyone with sender's **public** key can verify it came from them |

**Hybrid encryption:** asymmetric is slow, symmetric is fast. Real protocols (TLS, SSH) use asymmetric only to agree on a shared symmetric key, then switch to symmetric for all data. Best of both.

---

## Hashing

A hash function takes arbitrary input and produces a fixed-size output (digest). SHA-256 always produces 256 bits regardless of input size.

Properties:
- Same input always produces same output
- Different input produces completely different output (avalanche effect)
- One-way — cannot reverse a hash to get the input
- Collision-resistant — practically impossible to find two inputs with the same hash

Used in:
- Digital signatures — sign the hash of a message (faster than signing the whole message)
- Password storage — store hash, not plaintext
- File integrity — hash a file before and after transfer, compare

```bash
echo "hello" | sha256sum
# 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03

echo "hello " | sha256sum   # one extra space
# 72af2b975cf958dc4b2a4e2e1b75f8e2f0ff7cbbf0547b5ee23fc5a50400c1c
# completely different hash
```

---

## SSH — Secure Shell

Port **22 (TCP)**. Used for remote terminal access, file transfer (SFTP), and tunneling.

### Password Auth vs Key Auth

| | Password | SSH Key |
|--|----------|---------|
| What crosses the network | Password hash (encrypted over SSH tunnel) | Nothing secret — just a challenge/response |
| Brute-force risk | Yes | Practically zero |
| Automation-friendly | No | Yes |
| Recommended | No | Yes |

Servers on the internet get thousands of automated login attempts per day targeting password auth.

### How Key Authentication Works

```
Setup (done once):
  Generate keypair on your machine:
    Private key: ~/.ssh/id_ed25519      ← never leaves your machine
    Public key:  ~/.ssh/id_ed25519.pub  ← copy this to the server

  On the server:
    ~/.ssh/authorized_keys contains your public key

Connection:
  1. Your client: "I want to authenticate with key X"
  2. Server: generates a random challenge, encrypts it with your public key
             "Decrypt this if you have the private key"
  3. Your client: decrypts challenge using private key, sends proof
  4. Server: verifies proof matches the expected challenge result
  5. Login granted — no password, no secret ever crossed the wire
```

### Generating Keys

```bash
# Ed25519 — modern, recommended (smaller key, stronger than RSA-2048)
ssh-keygen -t ed25519 -C "label@example.com"

# RSA — older, still widely supported
ssh-keygen -t rsa -b 4096 -C "label@example.com"

# Keys saved to:
~/.ssh/id_ed25519       (private)
~/.ssh/id_ed25519.pub   (public)

# Copy public key to a remote server
ssh-copy-id user@server-ip

# Connect with specific key
ssh -i ~/.ssh/id_ed25519 user@server-ip

# AWS EC2 — connect with downloaded .pem file
chmod 400 mykey.pem
ssh -i mykey.pem ec2-user@ec2-public-ip
```

### Passphrase

A passphrase encrypts the private key file on disk. If someone steals the file, they cannot use it without the passphrase. The passphrase never crosses the network — it only unlocks the local key file.

Use `ssh-agent` to cache the decrypted key in memory so you do not type the passphrase on every connection:

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
```

### AWS and SSH Keys

When you launch an EC2 instance, you select a key pair. AWS stores the public key on the instance. You download the private key (`.pem`) once at creation — it cannot be downloaded again.

If you lose the `.pem` file:
- No password reset exists — there is no password
- Must either attach the instance's EBS volume to another instance and inject a new public key
- Or use AWS Systems Manager Session Manager if it was configured on the instance

Never commit `.pem` files to version control. Add `*.pem` to `.gitignore`.

---

## Signing vs Encrypting

Frequently confused — completely different operations with different key directions.

**Encrypting (for confidentiality):**
```
Alice wants to send a secret to Bob.
Alice encrypts with Bob's PUBLIC key.
Only Bob's PRIVATE key can decrypt.
Bob reads the message. Nobody else can.
```

**Signing (for authenticity + integrity):**
```
Alice wants to prove a message came from her and was not altered.
Alice hashes the message → SHA-256 digest.
Alice signs the hash with her PRIVATE key → signature.
Anyone with Alice's PUBLIC key can verify the signature.
Verification proves: (1) only Alice could have produced this signature,
                     (2) the message content was not changed after signing.
```

```bash
# Generate keypair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# Encrypt with public key (confidentiality)
openssl pkeyutl -encrypt -pubin -inkey public.pem -in message.txt -out encrypted.bin

# Decrypt with private key
openssl pkeyutl -decrypt -inkey private.pem -in encrypted.bin -out decrypted.txt

# Sign with private key (authenticity)
openssl dgst -sha256 -sign private.pem -out signature.bin message.txt

# Verify with public key
openssl dgst -sha256 -verify public.pem -signature signature.bin message.txt
# Output: Verified OK
```

---

## SSH Tunneling

SSH can forward network traffic through an encrypted tunnel. Useful for accessing services that are not directly reachable.

```bash
# Local port forwarding
# Access remote database (private subnet) through a bastion host
ssh -L 5432:db.internal:5432 user@bastion-ip
# Now: psql -h localhost -p 5432  → connects through tunnel to db.internal:5432

# Dynamic port forwarding (SOCKS proxy)
ssh -D 1080 user@server
# Configure browser to use SOCKS proxy localhost:1080
# All browser traffic now goes through the SSH server

# Remote port forwarding
ssh -R 8080:localhost:3000 user@server
# Anyone connecting to server:8080 gets forwarded to localhost:3000
```

SSH tunneling is used legitimately for database access through bastion hosts. Attackers use it for C2 communication and bypassing firewalls — same mechanism, different intent.

---

## Commands Reference

```bash
# Connect to a server
ssh user@ip
ssh -p 2222 user@ip          # non-standard port
ssh -i key.pem user@ip       # specific key file

# Verbose mode (see handshake details)
ssh -v user@ip
ssh -vvv user@ip             # maximum verbosity

# Test SSH connection without opening a shell
ssh -T git@github.com

# Copy files securely
scp file.txt user@ip:/remote/path/
scp -r directory/ user@ip:/remote/path/

# Mount remote filesystem
sshfs user@ip:/remote/path /local/mount

# View authorized keys on a server
cat ~/.ssh/authorized_keys

# View SSH host keys (fingerprints)
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Symmetric | One key encrypts and decrypts — fast, for bulk data |
| Asymmetric | Public/private key pair — slow, for key exchange and signatures |
| Hybrid | Asymmetric to agree on key, symmetric for data (TLS, SSH) |
| Encrypt | Use recipient's public key — only they can decrypt |
| Sign | Use your own private key — anyone with your public key verifies |
| Hash | Fixed-size fingerprint of data — one-way, collision-resistant |
| SSH | Secure shell — remote terminal access on port 22 |
| Authorized keys | File on server containing permitted public keys |
| Passphrase | Encrypts private key on disk — never sent over network |
| SSH tunnel | Forwarding network traffic through an encrypted SSH connection |
| `.pem` | AWS key file containing an RSA private key |
