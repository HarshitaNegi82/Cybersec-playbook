# SYN Flood Attack

A SYN flood is a denial-of-service attack that exploits the TCP three-way handshake. The attacker sends a large number of SYN packets but never completes the handshake, filling the server's connection queue and making it unable to accept legitimate connections.

---

## The TCP Handshake (Normal Behavior)

```
Client → Server : SYN        (I want to connect, seq = x)
Server → Client : SYN-ACK    (OK, my seq = y, I acknowledge yours)
Client → Server : ACK        (Connection established)
```

When the server receives a SYN, it must remember it — the OS stores the half-open connection in a data structure called the **SYN backlog queue** (also called the half-open queue or TCP backlog). It waits there until either:
- The client sends the final ACK → connection moves to established state
- A timeout expires → connection is dropped

---

## The Attack

The attacker sends SYN packets continuously but never sends the final ACK. Each SYN fills one slot in the SYN backlog queue.

```
Attacker → Server : SYN
Attacker → Server : SYN
Attacker → Server : SYN
... (thousands per second, no ACKs ever follow)
```

The attacker typically spoofs source IPs so the server's SYN-ACKs go nowhere. The half-open connections sit in the queue until timeout.

When the queue is full:
- The OS cannot accept new TCP connections
- Legitimate users trying to connect get dropped or time out
- The server appears completely offline even if CPU and bandwidth are fine

**This is a memory/queue exhaustion problem, not a CPU problem.** CPU usage may remain low while the server is functionally unreachable.

---

## Defenses

**SYN cookies:**  
When the SYN backlog fills, the server stops allocating queue slots. Instead, it encodes the connection state into the SYN-ACK sequence number itself (a cryptographic cookie). If a legitimate client sends the final ACK, the server reconstructs the connection state from the cookie. Attackers who never send ACKs never consume persistent resources.

**CDN / reverse proxy:**  
A CDN (Cloudflare, AWS Shield) terminates TCP connections at its edge. The SYN flood fills the CDN's backlog — which is massively distributed across hundreds of data centers — instead of the origin server's small backlog. The origin never sees the malicious SYNs. This protection collapses if the origin IP is exposed (DNS misconfiguration, as covered in dns.md).

**Rate limiting at network edge:**  
Upstream routers or firewalls can rate-limit SYN packets per source IP. Combined with IP reputation, this filters obvious flood traffic before it reaches the server.

---

## Identifying a SYN Flood

```bash
# Watch TCP connection states in real time
ss -s

# See half-open connections accumulating
netstat -an | grep SYN_RECV | wc -l

# Packet capture to see the flood
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'
```

A high and rising count of `SYN_RECV` connections with no corresponding `ESTABLISHED` connections is the signature of an active SYN flood.
