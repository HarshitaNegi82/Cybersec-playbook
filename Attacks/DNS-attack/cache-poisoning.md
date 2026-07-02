# DNS Cache Poisoning

DNS cache poisoning (also called DNS spoofing) is an attack where a malicious actor inserts fake DNS records into a resolver's cache. Every user who queries that resolver for the targeted domain gets sent to the attacker's server instead of the real one — until the TTL expires or the cache is manually cleared.

---

## Why It Is Possible

Classic DNS has a fundamental design weakness: it uses UDP, which has no connection state and allows any source to send a packet claiming to be from any address. The resolver trusts responses based on a 16-bit transaction ID (values 0–65535) that must match the outgoing query.

An attacker who can guess or brute-force that transaction ID before the legitimate response arrives can poison the cache.

---

## The Attack Step by Step

```
Step 1 — Trigger a query
   Attacker sends a request for a subdomain that is not cached:
   rand8723.bank.com
   This forces the resolver to send a real DNS query.

Step 2 — Resolver sends UDP query to bank.com's nameserver
   Query contains transaction ID = 41829 (random 16-bit value)
   Sent via UDP — connectionless, no handshake

Step 3 — Attacker floods the resolver with fake UDP responses
   Forged source IP: appears to be bank.com's nameserver
   Transaction IDs: 0, 1, 2, 3 ... racing toward 41829
   Fake answer: bank.com → 198.51.100.99 (attacker's IP)

Step 4 — If the forged response with ID 41829 arrives before the real answer
   Resolver accepts it as legitimate
   Caches: bank.com → 198.51.100.99

Step 5 — Every user querying this resolver for bank.com
   Gets sent to 198.51.100.99
   Until the poisoned TTL expires
```

The attacker never gains access to the DNS server. They exploit the protocol from the outside by winning a race condition.

---

## Why TTL Affects Recovery

The poisoned record stays in cache until its TTL expires. If that TTL is 86400 seconds (24 hours), users are redirected to the attacker's server for up to a day even after the attack is detected and stopped.

With a TTL of 60 seconds, the poisoned entry self-expires in one minute. The resolver fetches the real answer on the next query. This is why low TTL values during active incidents or migrations significantly limit the damage window.

---

## Defenses

**Source port randomization:**  
Modern resolvers randomize both the transaction ID (16 bits) and the UDP source port (16 bits). An attacker must now guess both simultaneously — 65,536 × 65,536 ≈ 4.3 billion combinations. This makes brute-force poisoning practically infeasible in real time.

**DNSSEC:**  
The domain owner signs all DNS records with a private key. The corresponding public key is published in the DNS itself, creating a chain of trust from the root down. A DNSSEC-validating resolver verifies the signature on every response. A poisoned record with no valid signature is rejected regardless of whether the transaction ID matched.

Check DNSSEC status:
```bash
dig example.com +dnssec +short
dig example.com DNSKEY
```

**Low TTL during sensitive operations:**  
During a server migration or CDN switch, lower TTL before making the change. If something goes wrong and a bad record gets cached, it expires quickly.

**0x20 encoding:**  
Some resolvers randomize the capitalization of query names — e.g., `gOOgLe.CoM`. The authoritative server must preserve this exact casing in its response. An attacker spoofing a response cannot replicate this without having intercepted the original query.

---

## Impact

- Users silently redirected to attacker-controlled servers
- Credential theft if the fake site mimics a login page
- Malware distribution if the fake site serves downloads
- All users of the affected resolver are impacted simultaneously, not just one victim
- In 2008, Dan Kaminsky disclosed a variant of this attack that could poison caches for entire domains in seconds — this triggered mass patching of DNS software worldwide
