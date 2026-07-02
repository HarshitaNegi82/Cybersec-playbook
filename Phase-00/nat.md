# NAT — Network Address Translation

Operates at: **Layer 3 (Network) + Layer 4 (Transport)**

---

## The Problem NAT Solves

IPv4 has ~4.3 billion addresses. The internet has far more devices. NAT allows an entire network of devices to share a single public IP address — each device has a private IP internally, but the outside world sees only the router's public IP.

Private IP ranges (RFC 1918) — not routable on the public internet:

| Range | CIDR | Common use |
|-------|------|-----------|
| `10.0.0.0` | `/8` | Large orgs, AWS VPCs |
| `172.16.0.0` | `/12` | Mid-size orgs |
| `192.168.0.0` | `/16` | Home networks |

---

## How NAT Works — The Translation Table

When a private device initiates a connection, the router creates a NAT table entry mapping the private IP:port to the public IP:port.

```
Device 192.168.1.5 opens connection to google.com:443
  Private source:  192.168.1.5:54321
  NAT translates → Public source: 45.25.65.142:61001

Device 192.168.1.9 opens connection to google.com:443
  Private source:  192.168.1.9:54321 (same ephemeral port, different device)
  NAT translates → Public source: 45.25.65.142:61002 (different public port)

NAT table:
  45.25.65.142:61001 ↔ 192.168.1.5:54321
  45.25.65.142:61002 ↔ 192.168.1.9:54321
```

Google sees both requests coming from `45.25.65.142` — one IP, two different ports. When Google responds, the router reads the destination port, looks up the NAT table, and forwards the packet to the correct internal device.

The port number is what makes NAT work for multiple simultaneous connections from the same public IP. This is why this specific form is called **PAT (Port Address Translation)** or **NAT overload** — many-to-one, using ports to distinguish sessions.

---

## Types of NAT

**Static NAT (1:1)**
One private IP permanently maps to one public IP. Used when an internal server must be reachable from the internet at a fixed public address.

```
203.0.113.10 (public) ↔ 10.0.0.50 (internal web server)
Always. Every connection to 203.0.113.10 goes to 10.0.0.50.
```

**Dynamic NAT**
A pool of public IPs is assigned to private devices on demand. Less common — requires one public IP per active device.

**PAT / NAT Overload (most common)**
Many private IPs share one public IP, differentiated by port. This is what your home router does. What AWS NAT Gateway does.

---

## What Changes at Each Router Hop

This is a frequent source of confusion. As a packet travels across the internet through multiple routers:

| Field | Changes at each hop | Stays the same |
|-------|--------------------|----|
| Source MAC | Yes — replaced by each router's MAC | |
| Destination MAC | Yes — replaced with next-hop MAC | |
| TTL | Yes — decremented by 1 | |
| Source IP | Only at NAT boundary | ✓ (otherwise) |
| Destination IP | Only at NAT boundary | ✓ (otherwise) |
| TCP data | Never | ✓ |
| HTTP data | Never | ✓ |

MAC addresses are local — they only have meaning within a single network segment. Every router strips the incoming frame and builds a new one with updated MACs for the next hop.

---

## NAT and Security

**Incidental protection:**
NAT provides a form of protection by default — inbound connections from the internet cannot reach internal devices unless the router has a NAT table entry for that connection (which only exists if the internal device initiated the connection first). This is not a firewall — it is a side effect of how NAT works. It should not be relied on as a security control.

**Port forwarding breaks this:**
Port forwarding creates a permanent NAT table entry: "all inbound traffic on public port 8080 → forward to 192.168.1.20:80." Now the internal device is directly reachable from the internet on that port. If port forwarding rules are misconfigured, internal services are exposed.

---

## AWS NAT Gateway

Per AWS documentation: a NAT gateway enables instances in private subnets to connect to services outside the VPC, but external services cannot initiate a connection with those instances.

```
EC2 in private subnet (10.0.10.25)
    │
    │ needs to reach api.github.com
    ▼
NAT Gateway (in public subnet, Elastic IP: 52.66.1.1)
    │
    │ translates 10.0.10.25 → 52.66.1.1
    ▼
Internet Gateway → github.com

Return traffic:
github.com → 52.66.1.1
NAT Gateway → 10.0.10.25
```

The public internet only ever sees `52.66.1.1`. The private instance IP never appears outside the VPC.

**NAT Gateway vs Internet Gateway:**

| | NAT Gateway | Internet Gateway |
|--|-------------|-----------------|
| Direction | Outbound only (initiated from inside) | Both directions |
| Where it lives | Public subnet | Attached to VPC |
| Used for | Private subnet internet access | Public subnet internet access |
| Cost | Per-hour + data transfer charges | No charge (data transfer charges apply) |

**NAT Gateway in each AZ:**
AWS recommends a NAT Gateway per AZ. If AZ-a fails, private instances in AZ-a route through NAT Gateway in AZ-a. If there is only one NAT Gateway in AZ-a, private instances in AZ-b lose outbound internet access when AZ-a fails.

---

## IPv6 Eliminates NAT

IPv6 uses 128-bit addresses — 3.4 × 10^38 possible addresses. Every device can have a globally unique address. Address exhaustion does not exist. NAT is not needed for IPv6.

AWS VPCs use an **Egress-Only Internet Gateway** for IPv6 to achieve the same result as a NAT Gateway — outbound-only internet access for private instances over IPv6, without NAT.

---

## Commands

```bash
# View your public IP (what the internet sees after NAT)
curl ifconfig.me
curl api.ipify.org

# View your private IP
ip addr show
hostname -I

# View NAT conntrack table on Linux (router/firewall)
sudo conntrack -L

# On iptables-based NAT (Linux router)
sudo iptables -t nat -L -n -v

# AWS — describe NAT Gateways
aws ec2 describe-nat-gateways \
  --query "NatGateways[*].[NatGatewayId,State,SubnetId,NatGatewayAddresses[0].PublicIp]" \
  --output table
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| PAT / NAT overload | Many private IPs → one public IP using port differentiation |
| Static NAT | One-to-one permanent mapping |
| NAT table | Router's record of active translations |
| Port forwarding | Static NAT entry for inbound connections to a specific internal host |
| AWS NAT Gateway | Managed NAT for private subnet outbound access |
| Egress-only IGW | IPv6 equivalent of NAT Gateway |
