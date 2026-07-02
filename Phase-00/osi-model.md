# OSI Model

The OSI (Open Systems Interconnection) model is a 7-layer framework that describes how data moves across a network. Every network protocol, every attack, every defence mechanism operates at a specific layer. Knowing where things belong tells you why they behave the way they do, and why certain defences fail against certain attacks.

---

## The 7 Layers

```
7 — Application     HTTP, DNS, FTP, SMTP — user-facing protocols
6 — Presentation    Encryption, compression, encoding (TLS lives here)
5 — Session         Connection management, authentication tokens
4 — Transport       TCP, UDP — ports, reliability, flow control
3 — Network         IP addressing, routing — how packets find their destination
2 — Data Link       MAC addressing, frames — delivery within a local network
1 — Physical        Cables, radio signals, raw bits
```

---

## Each Layer in Detail

### Layer 7 — Application

Where user-facing protocols live. HTTP, HTTPS, DNS, FTP, SMTP, SSH at the application level.

This layer does not handle delivery — it relies on all layers below it. A web browser making an HTTP request is operating at Layer 7.

**Attacks at this layer:** SQL injection, XSS, CSRF, API abuse, directory traversal.

See:
- [attacks/application-attacks/sql-injection.md](../../attacks/application-attacks/sql-injection.md)
- [attacks/application-attacks/xss.md](../../attacks/application-attacks/xss.md)

**Defence:** WAF (Web Application Firewall), input validation, output encoding, parameterized queries.

---

### Layer 6 — Presentation

Handles data encoding, compression, and encryption/decryption. TLS conceptually belongs here — it encrypts data before it is handed to lower layers.

**Attacks at this layer:** TLS downgrade attacks (forcing older, weaker protocol versions), HSTS bypass.

**Defence:** Enforce TLS 1.2 minimum (TLS 1.3 preferred), disable deprecated cipher suites, HSTS headers.

---

### Layer 5 — Session

Manages the establishment, maintenance, and termination of sessions between two applications. Session tokens, authentication state, and connection multiplexing operate here.

**Attacks at this layer:** Session hijacking (stealing a valid session token), session fixation.

**Defence:** Secure, HttpOnly, SameSite cookies. Session token rotation after login. Short session lifetimes.

---

### Layer 4 — Transport

TCP and UDP. This layer adds ports, handles reliable delivery (TCP), and manages flow control. Every connection is identified by the 4-tuple: source IP, source port, destination IP, destination port.

**Attacks at this layer:** SYN flood (exhausts TCP connection queue), UDP flood, port scanning.

See: [attacks/network-attacks/syn-flood.md](../../attacks/network-attacks/syn-flood.md)

**Defence:** SYN cookies, rate limiting, stateful firewalls, Security Groups (AWS).

**Commands:**
```bash
ss -tulpn          # all listening sockets with process info
ss -s              # socket statistics summary
netstat -an        # connection states (older tool)
```

---

### Layer 3 — Network

IP addressing and routing. Routers operate here, reading destination IP addresses and forwarding packets along the best path using routing tables.

**Attacks at this layer:** IP spoofing, ICMP flood, routing attacks (BGP hijacking).

**Defence:** Network ACLs (AWS), ingress filtering (BCP38), DDoS scrubbing at network edge.

**Commands:**
```bash
ip route show                  # view routing table
traceroute 8.8.8.8             # trace path to destination
ping -c 4 8.8.8.8              # test Layer 3 reachability
```

---

### Layer 2 — Data Link

MAC addressing and frame delivery within a local network. Switches operate here — they build MAC address tables and forward frames only to the correct port.

**Attacks at this layer:** ARP spoofing, MAC flooding, VLAN hopping.

See: [attacks/network-attacks/arp-spoofing.md](../../attacks/network-attacks/arp-spoofing.md)

**Defence:** Dynamic ARP Inspection (DAI) on managed switches, VLANs, 802.1X port authentication.

**Commands:**
```bash
arp -a             # view ARP cache (IP → MAC mappings)
ip neigh           # Linux equivalent, more detail
```

---

### Layer 1 — Physical

Raw bit transmission over cables, fiber, or radio waves. No addressing, no logic — just signal.

**Attacks at this layer:** Physical wiretapping, signal jamming, hardware implants.

**Defence:** Physical security controls, encrypted signals (WPA3 for WiFi).

---

## Device Classification by Highest Layer

The layer at which a device makes meaningful decisions:

| Device | Layer | Reason |
|--------|-------|--------|
| Hub | L1 Physical | Repeats electrical signals only — no awareness of addresses |
| Switch | L2 Data Link | Reads MAC addresses, forwards frames to the correct port |
| Router | L3 Network | Reads IP addresses, consults routing table, forwards packets |
| Basic firewall | L3–L4 | Reads IP + port, applies allow/deny rules |
| WAF | L7 Application | Reads full HTTP request content, inspects for attacks |
| L4 Load balancer | L4 Transport | Routes by IP + port, no HTTP awareness |
| L7 Load balancer | L7 Application | Routes by URL path, headers, cookies |

---

## Encapsulation

When data is sent across a network, each layer adds its own header before passing it down. This is encapsulation.

```
Application data (HTTP GET request)
    ↓  Layer 4 adds:
TCP segment (source port, destination port, sequence numbers)
    ↓  Layer 3 adds:
IP packet (source IP, destination IP, TTL)
    ↓  Layer 2 adds:
Ethernet frame (source MAC, destination MAC)
    ↓  Layer 1:
Bits on the wire / radio signal
```

At the destination, each layer strips its header and passes the inner content upward. This is decapsulation.

Each layer only understands its own header. The IP layer does not know or care what is inside the TCP segment. TCP does not know or care what HTTP method is being used. This is why a network firewall (Layer 3–4) is blind to what is inside an HTTP request — it only reads IP and TCP headers.

---

## What a Network Firewall Cannot See

A traditional network firewall at Layers 3–4 sees:

```
Source IP:       203.0.113.45
Destination IP:  10.0.1.50
Protocol:        TCP
Port:            443
Payload:         [encrypted or just bytes — firewall cannot interpret]
```

The firewall's decision: is this IP allowed? Is this port open? Yes to both → allow through.

Inside the HTTP request body might be:
```
username=' OR '1'='1' --&password=x
```

The firewall has no visibility into this. It passed the packet based on IP and port. The SQL injection reaches the application. This is why Layer 7 defences (WAF, parameterized queries) are required for application attacks — network firewalls are irrelevant to them.

---

## Quick Reference

| Attack | Layer | What Stops It |
|--------|-------|--------------|
| SQL Injection | L7 | Parameterized queries, WAF |
| XSS | L7 | Output encoding, CSP, WAF |
| SYN Flood | L4 | SYN cookies, CDN, edge rate limiting |
| ARP Spoofing | L2 | Dynamic ARP Inspection, VLANs |
| IP Spoofing | L3 | Ingress filtering (BCP38) |
| DNS Attacks | L7 (app) + L3 (transport) | DNSSEC, source port randomization |
