# DNS Amplification Attack

DNS amplification is a type of DDoS attack that uses open DNS resolvers as amplifiers. A small outgoing query triggers a much larger response that gets sent to the victim, effectively multiplying the attacker's bandwidth.

---

## The Core Mechanism

DNS uses UDP, which allows source IP spoofing — sending a packet with a forged source address. This is the property the attack depends on.

```
Attacker sends:
  Small DNS query (50–100 bytes)
  Source IP: forged to be victim's IP (e.g., 5.6.7.8)
  Destination: open DNS resolver

Resolver responds:
  Large DNS response (1000–4000 bytes)
  Sent to the source IP = victim's IP (5.6.7.8)

Result:
  Victim receives flood of large DNS responses they never requested
  Amplification factor: 20x to 70x depending on query type
```

The attacker's own bandwidth usage is minimal. The resolver multiplies the traffic and delivers it to the victim.

---

## Why a Spoofed Source IP Is Required

Without IP spoofing, the DNS responses come back to the attacker, not the victim. The attacker would flood themselves. The entire attack mechanism depends on responses being delivered to a different IP than the real sender.

This is also why the attack requires **open resolvers** — DNS servers that answer queries from anyone on the internet, not just their own customers. Legitimate recursive resolvers (like those run by ISPs) should only answer queries from their own users.

---

## Why DNS Creates Such Large Amplification

Certain DNS query types return very large responses:
- `ANY` queries — request all record types for a domain
- `DNSSEC`-enabled queries — add cryptographic signatures to responses
- Domains with many TXT records or MX records

A 64-byte query asking for `ANY` records on a domain with DNSSEC can return a 4000-byte response — a 60x amplification factor. With a botnet sending millions of these per second, the resulting traffic to the victim is devastating.

---

## Defenses

**BCP38 (ingress filtering) at the ISP level:**  
Network operators drop outbound packets whose source IP does not belong to their address space. This prevents IP spoofing at the source. If BCP38 were universally deployed, DNS amplification attacks would be impossible. Deployment remains incomplete across the internet.

**Close open resolvers:**  
Configure DNS resolvers to only answer queries from authorized clients — not from arbitrary internet addresses. This removes the amplification infrastructure.

**Rate limiting DNS responses per source IP:**  
Even if queries arrive, rate limiting prevents the resolver from sending unlimited large responses to any single source.

**Response Rate Limiting (RRL):**  
A specific DNS server feature that limits how many identical or similar responses are sent to the same IP address in a time window.
