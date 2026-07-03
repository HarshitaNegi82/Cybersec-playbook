# DNS — Domain Name System

Computers talk in IP addresses. Humans talk in domain names. DNS is the system that translates one to the other — every single time you open a browser tab, DNS runs before any connection is made.

---

## What DNS Is

DNS (Domain Name System) is a **distributed, hierarchical database**. It is not a single server somewhere. Thousands of servers worldwide each hold a piece of the namespace, and together they can answer any domain lookup in milliseconds.

**Distributed** — no single point of failure or control.  
**Hierarchical** — structured like a tree, starting from a root and branching out.

The hierarchy is visible in every domain name:

```
blog  .  example  .  com
  |          |          |
3rd level  2nd level  TLD (1st level)
```

DNS reads this right to left — `.com` first, then `example`, then `blog`.

---

## The Four Server Types

Every DNS lookup without caching involves exactly four types of servers. Understanding each role is the core of understanding DNS.

### 1. Recursive Resolver

The first server your device contacts. Also called a DNS recursor. It acts as an intermediary — it receives your question, does all the work of finding the answer by querying other servers, and returns the final result to you. It also caches answers so it does not repeat the same work for future queries.

Most users get a recursive resolver assigned automatically via DHCP from their ISP. Public alternatives: Cloudflare `1.1.1.1`, Google `8.8.8.8`, Quad9 `9.9.9.9`.

Cloudflare's `1.1.1.1` is operated in partnership with APNIC and is built with privacy as a design goal — it does not log user IP addresses and does not sell query data to advertisers.

### 2. Root Nameserver

The root server is the entry point of the DNS hierarchy. There are 13 sets of root nameservers (labeled A through M), operated by different organizations. Each set has hundreds of physical servers worldwide using anycast routing.

Root nameservers do not know the IP address of any specific domain. They know which TLD nameserver to consult based on the domain extension (`.com`, `.net`, `.in`, etc.).

Cloudflare operates part of the F-root server network — one of the 13 sets.

### 3. TLD Nameserver

The top-level domain nameserver handles one TLD (`.com`, `.net`, `.org`, `.in`, etc.). It knows which authoritative nameserver is responsible for each domain under that TLD.

The `.com` TLD nameserver is managed by Verisign. `.in` is managed by NIXI. All TLD nameservers are managed under oversight of IANA (Internet Assigned Numbers Authority), a branch of ICANN.

### 4. Authoritative Nameserver

The final stop. The authoritative nameserver holds the actual DNS records for a specific domain. It is the source of truth. When it returns an answer, that answer is definitive — it does not need to ask anyone else, and it does not cache.

When you register a domain and configure Cloudflare DNS, Cloudflare's nameservers become the authoritative nameservers for your domain.

---

## The Full Resolution Chain

This is the path a query takes when no cached answer exists anywhere.

```
Your Device (browser)
    │
    │  "What is the IP for blog.example.com?"
    ▼
Recursive Resolver (e.g., 1.1.1.1)
    │
    │  Resolver checks its cache — nothing found
    │  Queries a root nameserver
    ▼
Root Nameserver
    │
    │  "I don't know blog.example.com, but .com is handled here:"
    │  Returns address of the .com TLD nameserver
    ▼
TLD Nameserver (.com)
    │
    │  "I don't know blog.example.com, but example.com's
    │   authoritative nameserver is ns1.example.com"
    ▼
Authoritative Nameserver (ns1.example.com)
    │
    │  "blog.example.com → 93.184.216.34, TTL 300"
    ▼
Recursive Resolver
    │
    │  Caches the answer for 300 seconds
    │  Returns the IP to your device
    ▼
Your Device
    │
    │  Opens a TCP connection to 93.184.216.34
    ▼
Web page loads
```

In practice, this full chain runs only when no caching layer has a valid answer. The resolver caches everything it finds, so subsequent queries for popular domains skip most of these steps.

**Shortcut behavior (from Cloudflare docs):** If the resolver already has NS records for the authoritative nameserver but not the A record, it skips the root and TLD servers entirely and queries the authoritative server directly. The chain is only as long as it needs to be.

---

## Query Types

**Recursive query** — The client asks the resolver: "Find me the final answer." The resolver is obligated to return an answer or an error. This is what your device sends to the resolver.

**Iterative query** — The resolver asks each nameserver in turn. If a server does not know the answer, it returns a referral — "I don't know, but try this server." The resolver follows these referrals one by one. This is what the resolver sends internally.

**Non-recursive query** — The resolver already has the answer cached. Returns immediately without making any network request.

---

## Caching and TTL

Caching is what allows DNS to scale to billions of daily queries without root servers collapsing.

**Where caching happens (closest to farthest):**

1. **Browser cache** — Chrome, Firefox, Edge all cache DNS internally. In Chrome: `chrome://net-internals/#dns` to inspect it.
2. **OS stub resolver** — The OS checks this before any network request. Also checks `/etc/hosts` first on Linux.
3. **Recursive resolver cache** — The resolver caches results shared across all users. Cloudflare's `1.1.1.1` has very high cache hit rates due to query volume from millions of users.
4. **Authoritative nameserver** — Only reached when all caches miss.

**TTL (Time To Live)** — set by the domain owner in each DNS record. Controls how long any caching layer may keep that answer.

```
blog.example.com.    300    IN    A    93.184.216.34
                      ↑
                 TTL = 300 seconds
```

After 300 seconds, the caching server discards this entry and must query again.

**Choosing TTL values:**

| TTL | Tradeoff |
|-----|---------|
| 60–300s (low) | Propagation of changes is fast. Useful when migrating servers or moving to a new CDN. Also limits the damage window of cache poisoning. |
| 3600–86400s (high) | Reduces query load on authoritative servers. Changes propagate slowly — bad if you need to respond quickly to an incident. |

**Cloud context:** When a company moves their domain to Cloudflare, they temporarily lower TTL to 60–120 seconds before switching nameservers, so the change propagates worldwide quickly.

---

## DNS Record Types

| Record | What it answers | Example |
|--------|----------------|---------|
| **A** | IPv4 address for a domain | `example.com → 93.184.216.34` |
| **AAAA** | IPv6 address | `example.com → 2606:2800:220:1:248:1893:25c8:1946` |
| **CNAME** | Alias — maps one name to another name (resolver then looks up that name) | `www.example.com → example.com` |
| **MX** | Where to deliver email | `example.com → mail.example.com` (priority 10) |
| **NS** | Which servers are authoritative for this domain | `example.com → ns1.cloudflare.com` |
| **TXT** | Arbitrary text — SPF records, DKIM, domain ownership proof | `v=spf1 include:_spf.google.com ~all` |
| **PTR** | Reverse lookup: IP → hostname | Used in email server verification |
| **SOA** | Zone metadata — serial number, refresh intervals, contact info | One per zone, automatically maintained |
| **SRV** | Service locator — hostname + port for a specific protocol | Used by SIP, XMPP, game servers |

**CNAME note:** A CNAME points to a name, not an IP. The resolver follows the chain — if `www.example.com` is a CNAME for `example.com`, and `example.com` is a CNAME for `lb.example.net`, the resolver keeps following until it finds an A or AAAA record. This adds an extra lookup per CNAME hop.

---

## UDP vs TCP

DNS uses **UDP port 53** by default because:
- Most queries and responses are small (under 512 bytes for basic A record lookups)
- No handshake overhead — lower latency
- Connectionless — resolver handles thousands of queries per second without per-connection state

DNS switches to **TCP port 53** when:
- **Zone transfers (AXFR/IXFR)** — copying an entire zone's records between nameservers, can be megabytes of data
- **Response truncated** — if a UDP response exceeds the size limit, the server sets the TC (truncated) bit; the client retries over TCP
- **DNSSEC** — cryptographic signatures make records significantly larger, often exceeding UDP limits

Modern encrypted DNS:
- **DNS over TLS (DoT)** — port 853
- **DNS over HTTPS (DoH)** — port 443, indistinguishable from HTTPS traffic

---

## Cloudflare DNS Architecture

Cloudflare operates both a recursive resolver (`1.1.1.1`) and authoritative nameservers for millions of domains. Two things make this different from a typical DNS setup:

**Anycast routing on `1.1.1.1`:**  
The same IP address (`1.1.1.1`) is announced simultaneously from hundreds of Cloudflare data centers worldwide. When your query reaches the internet, BGP routing automatically sends it to the nearest Cloudflare location. There is no geographic lookup or load balancer directing you — the internet's own routing protocol handles it. This is why `1.1.1.1` feels fast from India, Japan, Brazil, and Europe alike.

**Cloudflare as authoritative nameserver:**  
When a domain uses Cloudflare DNS, Cloudflare's servers become the authoritative nameservers. For proxied DNS records, Cloudflare returns its own edge IP instead of the real server IP. This is what enables all of Cloudflare's protection features — Cloudflare controls what IP gets returned, so all traffic routes through their edge first.

---

## Commands

```bash
# Basic A record lookup
dig example.com

# Short output only
dig example.com +short

# Specific record types
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com AAAA

# Query a specific resolver instead of your default
dig @1.1.1.1 example.com
dig @8.8.8.8 example.com

# Trace the full resolution chain in real time
dig +trace example.com

# Reverse lookup: IP to hostname
dig -x 8.8.8.8

# Check TTL values
dig example.com | grep -A5 "ANSWER SECTION"

# Attempt zone transfer (should fail on legitimate domains)
dig @ns1.example.com example.com AXFR

# Check DNSSEC signatures
dig example.com +dnssec

# Check DNSKEY records
dig example.com DNSKEY

# nslookup alternative (works on Windows too)
nslookup example.com
nslookup example.com 1.1.1.1
```

Run `dig +trace google.com` and read the output carefully. You will see the resolver query root servers, get referred to `.com` TLD servers, get referred to Google's nameservers, and finally get the answer. That is the resolution chain running in real time.

---

## Attacks on DNS

DNS has several classes of attacks. Each one is covered in detail in its own file.

| Attack | What it does | Where to read |
|--------|-------------|---------------|
| Cache Poisoning | Injects fake records into a resolver's cache, redirecting users | [Attacks/DNS-attacks/cache-poisoning.md](../Attacks/DNS-attack/cache-poisoning.md) |
| DNS Amplification | Uses open resolvers to flood a victim with traffic | [Attacks/DNS-attacks/dns-amplification.md](../Attacks/DNS-attack/dns-amplification.md) |
| DNS Tunneling | Hides data or C2 traffic inside DNS queries | [Attacks/DNS-attacks/dns-tunneling.md](../Attacks/DNS-attack/dns-tunneling.md) |

**DNSSEC** is the primary defense against cache poisoning. It adds cryptographic signatures to DNS records so resolvers can verify authenticity. A poisoned record without a valid signature is rejected.

**Zone transfer restrictions** are critical. An open zone transfer lets anyone enumerate every DNS record for your domain at once — full infrastructure mapping without scanning. Zone transfers should be allowed only to authorized secondary nameservers.

---

## The Cloud Security Link

A domain fully proxied through Cloudflare returns Cloudflare's IP for all DNS lookups. The real origin server's IP is never exposed. DDoS hits Cloudflare's edge. WAF runs at Cloudflare's edge. Origin server never sees malicious traffic.

This entire protection collapses if even one subdomain has a DNS-only record pointing to the real server IP:

```bash
dig api.yourdomain.com +short
# Returns: 52.66.12.45   ← real AWS EC2 IP, not Cloudflare
```

An attacker finds this with a basic DNS lookup, bypasses Cloudflare entirely, and attacks the origin directly. The months of CDN configuration are irrelevant once the origin IP is known.

A DNS audit is always part of a cloud security review for exactly this reason.

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Recursive resolver | Does the lookup work on behalf of your device; caches results |
| Authoritative nameserver | Holds the actual records; final source of truth |
| Root nameserver | Top of hierarchy; points to TLD servers |
| TLD nameserver | Handles one TLD; points to domain's authoritative servers |
| TTL | How long a cached answer is valid |
| A record | Domain → IPv4 address |
| CNAME | Domain → another domain name |
| NS record | Domain → its authoritative nameservers |
| DNSSEC | Cryptographic signatures on DNS records; prevents cache poisoning |
| Anycast | Same IP announced from multiple locations; nearest one handles query |
| Zone transfer (AXFR) | Copies all records from primary to secondary nameserver |
