# Hybrid Networking — VPN, Direct Connect & Transit Gateway

Hybrid networking connects your AWS VPC to an on-premises data centre or office network. Every large company has servers they cannot immediately move to the cloud — databases, legacy systems, compliance-restricted systems. Hybrid networking lets cloud and on-premises resources communicate over private, encrypted channels instead of routing across the public internet.

---

## Virtual Private Networks (VPN)

A VPN creates an encrypted tunnel over the public internet. Traffic inside the tunnel is encrypted so eavesdroppers see only ciphertext — not the actual data.

### Site-to-Site VPN

Connects an entire on-premises network to a VPC. Both sides have routers/firewalls that handle encryption/decryption transparently. Individual devices on either side do not need VPN client software — they just send traffic normally and their gateway handles the tunneling.

```
On-premises network (192.168.0.0/16)
    │
    │ Encrypted IPsec tunnel over public internet
    │
AWS VPC (10.0.0.0/16)
```

Use case: company has a data centre in Mumbai and wants EC2 instances in AWS to access on-premises databases without exposing them to the internet.

### Remote Access VPN

Connects individual users to a network. Each user runs a VPN client on their laptop. The client encrypts all traffic from that device and sends it to a VPN endpoint.

Use case: employee working from home connects to corporate network or AWS resources as if physically in the office.

---

## AWS Site-to-Site VPN Components

### Customer Gateway (CGW)

A resource you create in AWS that represents your on-premises VPN device. It contains the public IP address of your physical router or firewall sitting in your data centre.

```bash
# Create a Customer Gateway
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.5 \        # your on-premises router's public IP
  --bgp-asn 65000 \                 # BGP Autonomous System Number
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=OnPrem-Router}]'
```

### Virtual Private Gateway (VGW)

The AWS side of the VPN connection. Attached to your VPC. It terminates the IPsec tunnels coming from your Customer Gateway.

```bash
# Create and attach Virtual Private Gateway
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512

aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-xxxxxxxxxx \
  --vpc-id vpc-xxxxxxxxxx
```

### VPN Connection

Links the Customer Gateway to the Virtual Private Gateway. AWS automatically creates **two IPsec tunnels** for redundancy — if one tunnel fails, traffic automatically fails over to the second.

```bash
# Create VPN connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxxxxxxxxx \
  --vpn-gateway-id vgw-xxxxxxxxxx \
  --options StaticRoutesOnly=false  # false = use BGP

# Download VPN configuration for your specific router vendor
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-xxxxxxxxxx
```

### Route Propagation

After creating the VPN connection, enable route propagation on the VPC route table. This allows the VGW to automatically add routes to on-premises networks into the route table.

```bash
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-xxxxxxxxxx \
  --gateway-id vgw-xxxxxxxxxx
```

Without this, even with the tunnel up, VPC instances have no route to reach on-premises IP ranges.

---

## Border Gateway Protocol (BGP)

BGP is the routing protocol of the internet. It dynamically shares routing information between autonomous systems (large networks with their own AS number).

In AWS hybrid networking, BGP is used to:
- Automatically advertise which IP ranges exist on each side of a VPN or Direct Connect
- Allow AWS and your on-premises router to exchange routes dynamically without manual static route configuration
- Handle failover — if one path fails, BGP automatically finds an alternative

```
On-premises router (AS 65000) ←→ AWS VGW (AS 64512)

On-premises advertises: 192.168.0.0/16
AWS advertises:         10.0.0.0/16

Both sides learn these routes automatically via BGP
No manual route table entries needed
```

**ASN (Autonomous System Number):** a unique identifier for each network participating in BGP routing. AWS uses ASN 7224 for its backbone. You can specify a private ASN (64512–65534) for your Customer Gateway.

Without BGP (static routing), you manually add each on-premises CIDR to the AWS route table. With BGP, routes are exchanged automatically and routes that become unreachable are withdrawn.

---

## AWS Direct Connect

Site-to-Site VPN tunnels run over the public internet — encrypted, but still sharing infrastructure with everyone else. This introduces variable latency, bandwidth limitations, and dependency on internet conditions.

AWS Direct Connect is a dedicated physical network connection between your data centre and AWS. A Direct Connect provider (telco) runs a physical fibre from your data centre to an AWS Direct Connect Location.

```
Your data centre
    │
    │ Dedicated physical fibre (1 Gbps or 10 Gbps)
    │ (not the internet)
    ▼
AWS Direct Connect Location (colocation facility)
    │
    ▼
AWS Region
```

### Benefits over VPN

| | Site-to-Site VPN | Direct Connect |
|--|-----------------|---------------|
| Path | Public internet | Dedicated private fibre |
| Latency | Variable, higher | Consistent, lower |
| Bandwidth | Up to 1.25 Gbps | 1 Gbps or 10 Gbps |
| Cost | Cheaper | More expensive (fibre + port charges) |
| Setup time | Minutes | Weeks to months |
| Use case | Backup, small workloads | Production, high-throughput, compliance |

### Direct Connect Gateway

A Direct Connect Gateway allows you to connect a single Direct Connect connection to multiple VPCs across different AWS regions — without needing a separate Direct Connect connection per region.

```
Your data centre
    │
    │ One Direct Connect connection
    ▼
Direct Connect Gateway
    ├── VPC in ap-south-1 (Mumbai)
    ├── VPC in us-east-1 (N. Virginia)
    └── VPC in eu-west-1 (Ireland)
```

Without a Direct Connect Gateway, you would need a separate physical connection to each region.

---

## AWS Transit Gateway

As you connect more VPCs, direct peering becomes unmanageable.

VPC peering is point-to-point. 5 VPCs need 10 peering connections. 10 VPCs need 45. VPC peering is also non-transitive — VPC-A peers with VPC-B, VPC-B peers with VPC-C, but VPC-A cannot reach VPC-C through that chain.

**Transit Gateway** solves this. It is a central hub that connects VPCs, VPNs, and Direct Connect connections. Any attached network can reach any other attached network through the Transit Gateway.

```
                    Transit Gateway
                    /    |    \    \
                VPC-A  VPC-B  VPC-C  On-premises VPN
```

Add a new VPC — just attach it to the Transit Gateway. No new peering connections to every existing VPC.

### Hub-and-Spoke Architecture

Transit Gateway naturally creates a hub-and-spoke model:
- **Hub:** Transit Gateway
- **Spokes:** VPCs, VPN connections, Direct Connect connections

Traffic from any spoke can reach any other spoke through the hub. This is also where you insert central security appliances — a network firewall or intrusion detection system attached to the Transit Gateway inspects all inter-VPC traffic in one place.

```
Spoke VPCs    →  Transit Gateway  →  Security Inspection VPC  →  Internet
```

```bash
# Create Transit Gateway
aws ec2 create-transit-gateway \
  --description "Central hub for all VPCs" \
  --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=disable

# Attach a VPC to Transit Gateway
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-xxxxxxxxxx \
  --vpc-id vpc-xxxxxxxxxx \
  --subnet-ids subnet-xxxxxxxx subnet-yyyyyyyy

# View Transit Gateway route table
aws ec2 describe-transit-gateway-route-tables \
  --filters "Name=transit-gateway-id,Values=tgw-xxxxxxxxxx"
```

---

## Multi-Region Connectivity

Connecting VPCs across AWS regions requires either:

**Transit Gateway peering** — two Transit Gateways in different regions peer with each other. Traffic between regions stays on AWS's global backbone network.

**VPC peering across regions** — direct peering between two VPCs in different regions. No Transit Gateway needed for simple two-VPC cases. Non-transitive.

```bash
# Create inter-region Transit Gateway peering
aws ec2 create-transit-gateway-peering-attachment \
  --transit-gateway-id tgw-xxxxxxxxxx \          # local region TGW
  --peer-transit-gateway-id tgw-yyyyyyyyyy \     # remote region TGW
  --peer-account-id 123456789012 \
  --peer-region us-east-1
```

---

## Choosing the Right Connectivity Option

| Scenario | Right choice |
|----------|-------------|
| Connect one office to one VPC, budget-limited | Site-to-Site VPN |
| Connect one office to one VPC, production-critical | Direct Connect |
| Connect multiple VPCs across regions | Transit Gateway |
| Developers connecting from home | Remote Access VPN / AWS Client VPN |
| Multiple on-premises sites to multiple VPCs | Direct Connect + Direct Connect Gateway + Transit Gateway |

---

## Commands Reference

```bash
# VPN
aws ec2 describe-customer-gateways
aws ec2 describe-vpn-gateways
aws ec2 describe-vpn-connections --query "VpnConnections[*].[VpnConnectionId,State,CustomerGatewayId]" --output table

# Direct Connect
aws directconnect describe-connections
aws directconnect describe-virtual-interfaces

# Transit Gateway
aws ec2 describe-transit-gateways
aws ec2 describe-transit-gateway-attachments
aws ec2 describe-transit-gateway-route-tables

# Check BGP session status on VPN
aws ec2 describe-vpn-connections \
  --query "VpnConnections[*].VgwTelemetry" \
  --output json
# Look for "Status": "UP" on both tunnels
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Site-to-Site VPN | Encrypted tunnel over internet connecting two networks |
| Remote Access VPN | Individual device connects to a network via VPN client |
| Customer Gateway | AWS resource representing your on-premises VPN device |
| Virtual Private Gateway | AWS-side VPN termination point, attached to a VPC |
| BGP | Routing protocol that dynamically exchanges network routes |
| ASN | Autonomous System Number — BGP identifier for a network |
| Direct Connect | Dedicated physical network connection to AWS (not internet) |
| Direct Connect Gateway | Connects one Direct Connect to multiple VPCs across regions |
| Transit Gateway | Central hub connecting multiple VPCs and on-premises networks |
| Hub-and-spoke | Architecture where all networks connect through a central hub |
| Route propagation | VGW automatically adds on-premises routes to VPC route table |
