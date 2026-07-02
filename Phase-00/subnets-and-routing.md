# Subnets, CIDR & Routing

---

## Why Subnets Exist

A flat network where every device can reach every other device directly has two problems:

- **Performance:** every ARP broadcast goes to all devices. With 500 devices on one network, broadcasts become noise.
- **Security:** a compromised device can attempt connections to every other device. Lateral movement is unrestricted.

Subnetting divides a large IP block into smaller, isolated segments. Communication between subnets requires going through a router, where access rules can be enforced. This is the foundational principle of network segmentation as a security control.

---

## CIDR Notation

CIDR (Classless Inter-Domain Routing) is how IP ranges are written.

```
10.0.0.0/16
    │       │
  Network  Prefix length
  address  (how many bits are fixed/network bits)
```

IPv4 addresses are 32 bits total. If `/16` means 16 bits are network bits, then 32 - 16 = 16 bits are host bits.

2^16 = 65,536 total addresses.

### Quick Reference

| CIDR | Host bits | Total IPs | AWS usable | Typical use |
|------|-----------|-----------|------------|-------------|
| `/8` | 24 | 16,777,216 | 16,777,211 | Huge org or RFC 1918 range |
| `/16` | 16 | 65,536 | 65,531 | VPC CIDR block |
| `/24` | 8 | 256 | 251 | Single subnet |
| `/26` | 6 | 64 | 59 | Small subnet |
| `/28` | 4 | 16 | 11 | Minimal subnet |
| `/32` | 0 | 1 | — | Single specific host |

AWS reserves 5 IPs per subnet: network address, VPC router, DNS server, future use, broadcast.

### Calculating a Block

For `10.0.2.0/24`:
- Network address: `10.0.2.0` (not usable)
- First host: `10.0.2.1` (AWS uses for VPC router)
- Last host: `10.0.2.254`
- Broadcast: `10.0.2.255` (not usable)
- Usable range: `10.0.2.4` → `10.0.2.254` (AWS reserves .0-.3)

### Does an IP Belong to a CIDR Block?

Check if `10.0.2.50` is in `10.0.0.0/16`:
- /16 means the first 16 bits are fixed: `10.0`
- `10.0.2.50` starts with `10.0` → yes, it is in this block

Check if `10.1.0.1` is in `10.0.0.0/16`:
- First 16 bits of `10.1.0.1` are `10.1`, not `10.0` → no

---

## RFC 1918 Private Ranges

These three ranges are reserved for private use — not routable on the public internet:

| Range | CIDR | Total Addresses |
|-------|------|----------------|
| `10.0.0.0` | `/8` | 16.7 million |
| `172.16.0.0` | `/12` | ~1 million |
| `192.168.0.0` | `/16` | 65,536 |

AWS VPC CIDR blocks must be from these ranges. Traffic from these addresses is dropped at internet boundaries by default — a packet with source `10.0.5.2` will never appear on the public internet.

---

## Route Tables — How Traffic Decisions Are Made

A route table is a list of rules. Each rule has two parts:
- **Destination** — a CIDR range this rule applies to
- **Target** — where to send matching traffic

When traffic leaves a subnet, the router checks the destination IP against the route table and uses the first matching rule (longest prefix match — most specific rule wins).

### Longest Prefix Match

If a route table has both `10.0.0.0/16 → local` and `10.0.1.0/24 → vpn-gateway`, traffic to `10.0.1.50` matches both rules. The `/24` rule is more specific (longer prefix), so it wins. Traffic goes to the VPN gateway, not locally.

### Route Table Examples

```
Public subnet (hosts: load balancer, NAT gateway, bastion)
────────────────────────────────────────────────────────
Destination       Target         Notes
10.0.0.0/16      local          all VPC traffic stays inside VPC
0.0.0.0/0        igw-abc123     everything else → Internet Gateway
```

```
Private app subnet (hosts: application servers)
────────────────────────────────────────────────────────
Destination       Target         Notes
10.0.0.0/16      local
0.0.0.0/0        nat-xyz789     outbound internet via NAT Gateway
```

```
Private database subnet
────────────────────────────────────────────────────────
Destination       Target         Notes
10.0.0.0/16      local          only talks to other VPC resources
(no default route)              no internet path exists
```

The database subnet has no `0.0.0.0/0` route. There is literally no path for internet-bound traffic to leave. Even if someone misconfigured a Security Group to allow outbound, the routing table would drop the packet.

---

## Traffic Flow — A Full Request

```
User browser → ALB (public subnet, 10.0.1.50)
  Route table: 0.0.0.0/0 → IGW
  IGW performs 1:1 NAT: user's packet destination (52.66.1.10 ALB EIP)
                        translated to internal 10.0.1.50

ALB → App server (private subnet, 10.0.10.25)
  Route table: 10.0.0.0/16 → local
  No gateway needed — same VPC, local route

App server → Database (private DB subnet, 10.0.20.10)
  Route table: 10.0.0.0/16 → local
  Same VPC, direct

App server → External API (e.g., payment gateway on internet)
  Route table: 0.0.0.0/0 → NAT Gateway (10.0.1.30)
  NAT Gateway translates 10.0.10.25 → NAT EIP (52.66.2.1)
  NAT Gateway → IGW → internet
  Return traffic: internet → IGW → NAT Gateway → 10.0.10.25

Database: no internet connectivity possible
  No 0.0.0.0/0 route exists in DB subnet's route table
```

---

## Subnet Design Patterns

**Three-tier architecture (standard for most applications):**

```
Public tier      → load balancers, NAT gateways
Private app tier → application servers, containerized services
Private data tier → databases, caches, file systems
```

Each tier communicates only downward — load balancer to app, app to database. The database never initiates connections to the app tier. Security Groups enforce this directional restriction.

**Why separate app and DB subnets:**  
You can configure the DB Security Group to only accept connections from the app subnet's CIDR (`10.0.10.0/24`). Nothing else — not the public subnet, not the internet — can reach the database, even if it tried.

**Multi-AZ:**  
Each tier has subnets in at least two Availability Zones. If one AZ's infrastructure fails, the other AZ handles traffic. The same CIDR cannot be in two AZs — each subnet is tied to exactly one AZ.

```
AZ ap-south-1a           AZ ap-south-1b
Public:  10.0.1.0/24     Public:  10.0.2.0/24
App:     10.0.10.0/24    App:     10.0.11.0/24
DB:      10.0.20.0/24    DB:      10.0.21.0/24
```

---

## Security Angle — Why Segmentation Limits Attacks

If an attacker compromises an app server:
- They are inside the VPC — they can reach things the compromised server can reach
- App Security Group: allows 443 outbound (to external APIs), allows 5432 to DB subnet only
- DB Security Group: allows 5432 from app subnet CIDR only

The attacker can try to connect to the database on port 5432 from the compromised app server — this is allowed. But they cannot connect to the database on port 22 (not in DB Security Group rules), cannot connect to the public subnet on sensitive ports, and cannot reach other services outside the VPC unless those services are in the allowed outbound rules.

Proper segmentation does not prevent initial compromise. It contains the blast radius after compromise.

---

## Commands

```bash
# Linux — view network interfaces and IPs
ip addr show

# View routing table
ip route show

# Test if an IP is in a CIDR block (Python)
python3 -c "import ipaddress; print('10.0.2.50' in ipaddress.ip_network('10.0.0.0/16'))"

# Scan what hosts are alive in a subnet (only on networks you own)
nmap -sn 10.0.1.0/24

# AWS CLI — list subnets with CIDR and AZ
aws ec2 describe-subnets \
  --query "Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]" \
  --output table

# Check route tables
aws ec2 describe-route-tables \
  --query "RouteTables[*].Routes" \
  --output table
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| CIDR | Notation for IP ranges — `/24` means 24 fixed bits, 8 host bits |
| Subnet | A segment of a VPC's IP range, bound to one AZ |
| Route table | Rules controlling where traffic from a subnet gets forwarded |
| Local route | Default VPC-internal route — always present |
| Longest prefix match | Most specific route wins when multiple routes match |
| Three-tier architecture | Public (LB), private app, private data — standard design pattern |
| Segmentation | Dividing a network into isolated subnets to limit lateral movement |
| RFC 1918 | Standard defining private IP ranges not routable on the internet |
