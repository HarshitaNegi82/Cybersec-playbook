# TLS and HTTPS

TLS (Transport Layer Security) is the protocol that makes HTTPS secure. Every time a browser shows a padlock, TLS is running underneath. The padlock only tells you encryption is present — it says nothing about whether the configuration is actually safe. Misconfigured TLS is one of the most common cloud infrastructure findings.

---

## What TLS Does

Three things:

1. **Authentication** — proves you are talking to the real server, not an impostor
2. **Confidentiality** — encrypts all data so nobody between client and server can read it
3. **Integrity** — detects any tampering with data in transit

Without TLS, plain HTTP sends everything as readable text — passwords, session cookies, form data — visible to anyone between you and the server.

---

## Symmetric vs Asymmetric Encryption

Before the handshake makes sense, understand these two types.

**Symmetric encryption:** one shared key, both sides encrypt and decrypt with it. Fast. Used for bulk data. Problem: how do two strangers securely agree on a shared key without an eavesdropper intercepting it?

**Asymmetric encryption:** a mathematically linked key pair. Public key encrypts or verifies. Private key decrypts or signs. Slower. Solves the key agreement problem without sending secrets over the wire.

TLS uses asymmetric cryptography to agree on a shared symmetric key, then switches to symmetric encryption for all data. You get the security of asymmetric key exchange and the speed of symmetric bulk encryption.

---

## The TLS Handshake Step by Step

```
1. Client Hello
   Client → Server
   "I support TLS 1.2 and 1.3. Here are the cipher suites I accept.
    Here is a random number (client random)."

2. Server Hello + Certificate
   Server → Client
   "Let's use TLS 1.3 with TLS_AES_256_GCM_SHA384.
    Here is another random number (server random).
    Here is my certificate."

3. Client verifies the certificate
   Client checks:
   - Is this certificate signed by a trusted CA?
   - Does the domain name in the cert match what I connected to?
   - Is the certificate expired?
   If any check fails → handshake aborted, browser shows error

4. Key Exchange
   Client and server use asymmetric cryptography (ECDH in TLS 1.3)
   to derive a shared session key — without transmitting the key itself.
   Both sides compute the same key independently from the exchange.

5. Finished messages
   Both sides send a "Finished" message encrypted with the session key.
   If the other side decrypts it correctly, both know the key is shared.

6. Symmetric encryption begins
   All HTTP data is now encrypted with AES-GCM (or ChaCha20-Poly1305).
   Fast, secure, private.
```

In TLS 1.3, the handshake is faster — only 1 round trip instead of 2, because the key exchange parameters are sent in the Client Hello itself.

---

## Certificate Chain and CA Trust Model

A TLS certificate is not trusted on its own. It is trusted because it is signed by a Certificate Authority (CA) that your device already trusts.

**The chain:**

```
Root CA (e.g., ISRG Root X1 for Let's Encrypt)
    ↓ signs
Intermediate CA (e.g., Let's Encrypt R3)
    ↓ signs
Your server's certificate (example.com)
```

Your browser and OS come with a list of trusted Root CAs (called the trust store). When you connect to a site, the server presents its certificate plus any intermediate certificates. The browser verifies the chain: does this certificate chain up to a root CA it trusts?

**Why intermediate CAs exist:** Root CAs keep their private key offline and extremely protected — if compromised, all trust on the internet collapses. Intermediate CAs do the day-to-day signing, so if an intermediate is compromised, only it needs to be revoked, not the root.

**A common misconfiguration:** the server only sends its own certificate without the intermediate certificates. The browser fails to build the trust chain and shows a certificate error, even though the certificate itself is valid.

---

## TLS Versions

| Version | Status |
|---------|--------|
| SSL 2.0, 3.0 | Broken — never use |
| TLS 1.0 | Deprecated — disabled by PCI DSS since 2018 |
| TLS 1.1 | Deprecated |
| TLS 1.2 | Acceptable if configured correctly — ECDHE + AEAD ciphers only |
| TLS 1.3 | Current standard — use this where possible |

TLS 1.3 eliminates weak cipher suites by design — only AEAD (Authenticated Encryption with Associated Data) ciphers are allowed. It removes the complexity and attack surface of TLS 1.2 configuration.

---

## Cipher Suites

A cipher suite specifies the exact set of cryptographic algorithms used for a TLS connection:

```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
     │      │          │          │
     │      │          │          └─ Hash for MAC
     │      │          └──────────── Symmetric cipher (AES-256 in GCM mode)
     │      └─────────────────────── Authentication (RSA key in the certificate)
     └────────────────────────────── Key exchange (ECDHE = Elliptic Curve Diffie-Hellman Ephemeral)
```

**ECDHE** provides forward secrecy — each session uses a new ephemeral key pair for key exchange. Even if the server's private key is compromised later, past sessions cannot be decrypted because the ephemeral keys are discarded after the session.

**Safe TLS 1.2 cipher suites:**
```
ECDHE-ECDSA-AES128-GCM-SHA256
ECDHE-RSA-AES128-GCM-SHA256
ECDHE-ECDSA-AES256-GCM-SHA384
ECDHE-RSA-AES256-GCM-SHA384
ECDHE-ECDSA-CHACHA20-POLY1305
ECDHE-RSA-CHACHA20-POLY1305
```

**Cipher suites to disable:**
- RC4 — statistically breakable
- 3DES — vulnerable to SWEET32 attack
- NULL — no encryption at all
- EXPORT ciphers — intentionally weakened by US export law, now exploitable
- CBC mode in TLS 1.0/1.1 — vulnerable to BEAST and POODLE

---

## Common TLS Misconfigurations

These are what you will actually find and fix in cloud environments:

**Expired certificate:**  
The most common and most avoidable. The certificate has passed its expiry date. The browser shows a hard error — users cannot proceed. Fix: automated certificate renewal (Let's Encrypt + certbot, AWS ACM handles renewal automatically). Alert at 30 days, critical at 7.

**Missing intermediate certificate:**  
Server sends only its own certificate, not the intermediate CA certificates. Some clients (especially mobile apps) cannot build the trust chain and reject the connection. Fix: concatenate the intermediate certificates into the certificate file.

**TLS 1.0/1.1 still enabled:**  
Old protocol versions remain enabled on the server because nobody disabled them after upgrading. Fix: configure the server to reject TLS 1.0 and 1.1. Nginx: `ssl_protocols TLSv1.2 TLSv1.3;`

**Weak cipher suites:**  
RC4, 3DES, CBC-mode ciphers still in the server's cipher list. These have known vulnerabilities. Fix: explicit cipher suite allowlist with only ECDHE + AEAD combinations.

**Missing HSTS:**  
HTTP Strict Transport Security is a response header that tells browsers to always use HTTPS for this domain, never HTTP — even if the user types `http://`. Without HSTS, a downgrade attack can strip HTTPS. Fix: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`

**Self-signed certificate in production:**  
A certificate that was not signed by a trusted CA — valid for testing, completely inappropriate for production. Browsers show a hard warning. Automation tools often skip verification entirely for self-signed certs, creating false confidence.

**Hostname mismatch:**  
Certificate is issued for `example.com` but the server is accessed as `www.example.com` or `api.example.com`. The certificate does not cover the domain being accessed. Fix: issue certificates with all required Subject Alternative Names (SANs), or use wildcard certificates appropriately.

---

## testssl.sh — The Tool for Auditing TLS

testssl.sh is a free, open-source command-line tool that checks a server's TLS/SSL configuration — protocols, cipher suites, certificate chain, security headers, and known vulnerabilities. It works against any TCP endpoint, not just port 443.

```bash
# Install
git clone --depth 1 https://github.com/drwetter/testssl.sh.git
cd testssl.sh

# Run full scan on a domain
./testssl.sh example.com

# Run via Docker (no installation needed)
docker run --rm -ti drwetter/testssl.sh example.com

# Specific checks only
./testssl.sh -p example.com          # protocols only (which TLS versions)
./testssl.sh -s example.com          # standard cipher categories
./testssl.sh -f example.com          # forward secrecy check
./testssl.sh -h example.com          # HTTP headers (HSTS, CSP, etc.)
./testssl.sh -U example.com          # all vulnerability checks

# Check each cipher individually (comprehensive, slow)
./testssl.sh -e example.com

# Output to file
./testssl.sh --jsonfile result.json example.com
./testssl.sh --htmlfile result.html example.com

# Test a non-HTTPS service (e.g., SMTP with STARTTLS)
./testssl.sh -t smtp smtp.gmail.com:587

# STARTTLS test
./testssl.sh --starttls smtp example.com:587
```

testssl.sh color-codes output: green = good, yellow = warning, red = problem. A properly configured server should be almost entirely green.

**What testssl.sh checks:**
- Which TLS/SSL protocol versions the server accepts
- Which cipher suites are offered and in what order
- Certificate validity, chain, and SANs
- Forward secrecy (ECDHE or DHE key exchange)
- Known vulnerabilities: Heartbleed, POODLE, BEAST, CRIME, SWEET32, ROBOT, Lucky13
- HTTP security headers: HSTS, HPKP, Expect-CT, X-Frame-Options, CSP
- OCSP stapling status
- Client simulation — which browsers and OS versions can connect and with which cipher

**HSTS_MIN is preset to 179 days** in testssl.sh — if the HSTS max-age is shorter than 179 days, it will warn. Recommended: at least 1 year (31,536,000 seconds).

---

## Checking TLS with openssl Directly

```bash
# Connect and see the full handshake and certificate
openssl s_client -connect example.com:443

# Show the certificate chain
openssl s_client -connect example.com:443 -showcerts

# Check certificate dates and subject
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates -subject

# Check SANs (Subject Alternative Names)
openssl x509 -in cert.pem -noout -text | grep -A2 "Subject Alternative"

# Force a specific TLS version to test
openssl s_client -connect example.com:443 -tls1_2
openssl s_client -connect example.com:443 -tls1_3

# Check if TLS 1.0 is accepted (should fail on a correctly configured server)
openssl s_client -connect example.com:443 -tls1
```

---

## Cloud Context

**Cloudflare SSL Termination:**  
Cloudflare terminates TLS at its edge — the user's HTTPS connection ends at Cloudflare's data center. Cloudflare decrypts the traffic, applies WAF rules and DDoS inspection, then re-encrypts and forwards to the origin. This is why Cloudflare's WAF can inspect HTTPS traffic. Without termination, the WAF would see only encrypted bytes.

**AWS ACM (Certificate Manager):**  
AWS provides free TLS certificates for use with AWS services (ALB, CloudFront, API Gateway). ACM handles automatic renewal. Certificates provisioned through ACM cannot be exported — they can only be used with supported AWS services.

**HSTS and Cloudflare:**  
Cloudflare can inject HSTS headers at the edge even if your origin server does not set them. This means the HTTPS enforcement happens at Cloudflare's edge for every response, without touching the origin configuration.

---

## Quick Reference

| Term | Meaning |
|------|---------|
| TLS | Protocol for encrypted, authenticated connections |
| Certificate | Contains a public key + domain name + CA signature |
| CA | Certificate Authority — trusted entity that signs certificates |
| Root CA | Top of trust chain — pre-installed in browsers and OS |
| Intermediate CA | Issued by root CA, signs end-entity certificates |
| ECDHE | Key exchange algorithm providing forward secrecy |
| Forward secrecy | Past sessions stay secret even if private key is later compromised |
| AEAD | Authenticated Encryption with Associated Data — provides both encryption and integrity |
| HSTS | HTTP header forcing browsers to always use HTTPS for this domain |
| OCSP stapling | Server sends certificate revocation status in the TLS handshake |
| testssl.sh | CLI tool to audit TLS configuration, cipher suites, vulnerabilities |
| SNI | Server Name Indication — allows multiple TLS certificates on one IP |
