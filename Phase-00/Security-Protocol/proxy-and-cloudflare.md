# Proxy, Reverse Proxy & Cloudflare Architecture

---

## Forward Proxy

A forward proxy sits in front of clients. It intercepts outgoing requests and sends them on behalf of the client.

```
Client → Forward Proxy → Internet
```

The destination server sees the proxy's IP, not the client's. The client is hidden.

**Who configures it:** The user or the network admin on the client side.

**Common uses:**
- Corporate networks routing all employee traffic through a proxy to filter or monitor it
- Bypassing geo-restrictions — the destination sees the proxy's country, not yours
- Privacy — your ISP sees only that you connected to the proxy, not what destinations you visited

---

## Reverse Proxy

A reverse proxy sits in front of servers. It intercepts incoming requests and forwards them to the real server.

```
Internet → Reverse Proxy → Origin Server
```

The client sees the proxy's IP, not the origin's. The server is hidden.

**Who configures it:** The server owner. Clients have no awareness of it — they believe they are connecting directly to the site.

**Common uses:**
- Load balancing — distributing requests across multiple backend servers
- TLS termination — proxy handles HTTPS, forwards plain HTTP internally
- WAF — inspects Layer 7 traffic for attacks before forwarding
- Caching — serves cached responses, reducing origin load
- DDoS protection — absorbs attacks at edge infrastructure

---

## The One-Sentence Distinction

> Forward proxy hides the client. Reverse proxy hides the server.

---

## Cloudflare as a Global Reverse Proxy

Cloudflare is the most widely deployed reverse proxy in the world. Its architecture is built on three foundations: DNS control, anycast routing, and edge processing.

### Step 1 — DNS Control Is the Entry Point

When you add a domain to Cloudflare, you update your domain registrar to use Cloudflare's nameservers:

```
ns1.cloudflare.com
ns2.cloudflare.com
```

Cloudflare now controls what IP address is returned when anyone looks up your domain. For proxied DNS records, Cloudflare returns one of its own edge IPs instead of your real server's IP. This is the mechanism that gets Cloudflare in the path of all your traffic — without it, nothing else works.

### Step 2 — Anycast Routing Gets Traffic to the Nearest Edge

Cloudflare announces its edge IPs from hundreds of data centers simultaneously using anycast. When a user's DNS resolver resolves your domain to a Cloudflare IP, the user's TCP connection reaches the nearest Cloudflare data center automatically, determined by BGP routing. No geographic load balancer needed — internet routing handles it.

This is also how Cloudflare's `1.1.1.1` resolver works: the same IP exists at every Cloudflare location, and queries are served by the nearest one.

### Step 3 — Edge Processing Before Forwarding

Once traffic arrives at Cloudflare's edge, before forwarding to your origin:

```
Incoming request
    │
    ▼
TLS termination — Cloudflare decrypts HTTPS
    │
    ▼
DDoS detection — volumetric L3/L4 floods dropped here
    │
    ▼
WAF rules — inspect HTTP content for SQL injection, XSS, etc.
    │
    ▼
Bot detection — distinguish real browsers from scrapers/bots
    │
    ▼
Cache check — if this resource is cached, serve it without hitting origin
    │
    ▼
Forward to origin server
```

### Step 4 — The Origin IP Protection Model

Because your real server's IP never appears in any public DNS record, the public internet cannot address it directly. An attacker trying to DDoS your server hits Cloudflare's edge — which is built to absorb it across a globally distributed infrastructure.

**This protection has one critical failure mode:**

If any DNS record points directly at the real origin IP (not proxied through Cloudflare), that record leaks the IP. An attacker runs `dig api.yourdomain.com +short`, gets `52.66.12.45`, and bypasses Cloudflare entirely to attack that IP directly. All of Cloudflare's protection becomes irrelevant for the traffic that hits the origin directly.

**Hardening the origin:** Even with Cloudflare in front, configure your origin server's firewall (Security Group in AWS) to only accept inbound TCP 443 from Cloudflare's published IP ranges. Reject all other sources. Cloudflare publishes its IP ranges at `cloudflare.com/ips`. This way, even if an attacker discovers the origin IP, connections from non-Cloudflare sources are dropped at the network level.

---

## Nginx as a Reverse Proxy — Practical Configuration

Cloudflare is not the only reverse proxy. Nginx is commonly deployed as a local reverse proxy in front of application servers.

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    location / {
        proxy_pass         http://127.0.0.1:3000;    # forward to app on port 3000
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;   # pass original client IP
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

`X-Real-IP` and `X-Forwarded-For` headers pass the original client's IP to the application — otherwise the application would only see `127.0.0.1` (the proxy's address) as the source of every request.

---

## Load Balancer — Reverse Proxy With Multiple Backends

A load balancer is a specialized reverse proxy that distributes traffic across multiple backend servers.

```
Users
    │
    ▼
Load Balancer
    ├── Server 1 (10.0.10.10)
    ├── Server 2 (10.0.10.11)
    └── Server 3 (10.0.10.12)
```

Load balancing algorithms: round-robin (default), least connections, IP hash (session affinity), weighted.

**AWS load balancers:**

| Type | Layer | Routes based on |
|------|-------|----------------|
| ALB (Application Load Balancer) | L7 | URL path, HTTP headers, query strings, hostname |
| NLB (Network Load Balancer) | L4 | IP address and TCP/UDP port |

ALB is the right choice for most web applications — it can route `/api/*` to one target group and `/*` to another, terminate TLS, and integrate with WAF. NLB handles extreme throughput and ultra-low latency requirements.

**Health checks:** Load balancers continuously check backend servers. If a server fails health checks, it is removed from rotation automatically. This is how zero-downtime deployments work — new server passes health checks before traffic is sent to it.

---

## Layer 3/4 DDoS vs Layer 7 Attack — Why They Need Different Defences

A WAF operates at Layer 7. It reads HTTP requests and responses. It can detect SQL injection in a query parameter, block an XSS payload in a form body, or rate-limit a specific API endpoint.

A volumetric Layer 3/4 flood sends millions of packets at the network and transport layers — SYN floods, UDP floods, ICMP floods. The goal is to exhaust network bandwidth or connection state before any traffic reaches the application.

By the time a L3/4 flood reaches a WAF, the network connection is already saturated. The WAF cannot fix bandwidth exhaustion at the transport layer.

```
L3/4 volumetric flood:
Attacker → [Cloudflare edge network] → traffic dropped at network layer
                                      → origin never reached

L7 application attack:
Attacker → Cloudflare → WAF inspects HTTP → blocks or forwards
```

The correct defence model is layered:
1. L3/4 DDoS protection at the network edge (Cloudflare Magic Transit, AWS Shield)
2. Reverse proxy / load balancer for connection distribution
3. WAF for L7 inspection
4. Application-level validation

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Forward proxy | Sits in front of clients, hides the client from destinations |
| Reverse proxy | Sits in front of servers, hides the server from clients |
| Anycast | Same IP announced from multiple locations; nearest handles traffic |
| TLS termination | Proxy decrypts HTTPS at the edge to inspect content |
| Origin server | The real backend server behind a reverse proxy |
| Origin IP leak | When a DNS record exposes the origin IP, bypassing CDN protection |
| WAF | Web Application Firewall — inspects Layer 7 HTTP traffic for attacks |
| ALB | AWS Application Load Balancer — L7 reverse proxy |
| Health check | Load balancer probe to verify backend servers are working |
