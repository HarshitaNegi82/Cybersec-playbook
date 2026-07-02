# AWS VPC — Virtual Private Cloud

A VPC is a logically isolated section of the AWS Cloud where you define and control the entire network environment — IP ranges, subnets, route tables, gateways, and access rules. Resources inside a VPC are isolated from other AWS customers by default.

Every AWS account gets a default VPC created automatically in each region. Production systems use custom VPCs with deliberately designed subnets and security boundaries.

---

## How a VPC Differs from a Physical Subnet

A physical subnet is a segment of a real LAN — cables, switches, actual hardware. A VPC subnet is software-defined: AWS creates the logical isolation at the hypervisor level, and the underlying physical infrastructure is shared but completely opaque to you.

What stays the same: CIDR notation, routing logic, IP address assignment. What changes: no physical switch to configure, no cables to run, everything is API-controlled. The security model is also software-defined — Security Groups and NACLs replace physical firewall appliances.

---

## Core Components

### CIDR Block — The IP Range

When creating a VPC, you assign it an IP range in CIDR notation. This defines the pool of private IP addresses available for all subnets in that VPC.

```
10.0.0.0/16     → 65,536 total addresses (most common for VPCs)
10.0.0.0/24     → 256 addresses
10.0.0.0/28     → 16 addresses
```

AWS requires VPC CIDR blocks to be from the RFC 1918 private ranges:
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

These are not internet-routable. Traffic from these addresses is dropped at the internet boundary by default — which is intentional.

AWS reserves 5 IP addresses in every subnet: the first 4 and the last 1. A `/24` subnet gives 256 - 5 = 251 usable addresses.

### Subnets

A subnet is a segment of the VPC's IP range, associated with exactly one Availability Zone. A subnet cannot span AZs.

Subnets are categorized by their routing, not by any inherent property:

- **Public subnet** — has a route in its route table pointing to an Internet Gateway for `0.0.0.0/0`. Resources here can receive inbound connections from the internet if they have a public IP.
- **Private subnet** — no route to an Internet Gateway. Resources here cannot receive inbound connections from the internet. Outbound traffic can still flow through a NAT Gateway.

```
VPC: 10.0.0.0/16
├── Public Subnet AZ-a:  10.0.1.0/24   (load balancer, NAT gateway, bastion)
├── Public Subnet AZ-b:  10.0.2.0/24   (load balancer redundancy)
├── Private App AZ-a:    10.0.10.0/24  (application servers)
├── Private App AZ-b:    10.0.11.0/24  (application servers)
├── Private DB AZ-a:     10.0.20.0/24  (database primary)
└── Private DB AZ-b:     10.0.21.0/24  (database standby)
```

### Route Tables

Per AWS documentation: your VPC has an implicit router, and route tables control where network traffic is directed. Each subnet must be associated with exactly one route table. A subnet not explicitly associated with a custom route table uses the main route table.

Every route table contains a **local route** by default — `10.0.0.0/16 → local` — which enables resources in the VPC to communicate with each other without going through a gateway. This route cannot be deleted.

Additional routes are added to control outbound traffic:

```
Public subnet route table:
Destination       Target
10.0.0.0/16      local          ← all VPC traffic stays local
0.0.0.0/0        igw-abc123     ← everything else → Internet Gateway
```

```
Private app subnet route table:
Destination       Target
10.0.0.0/16      local
0.0.0.0/0        nat-xyz789     ← outbound goes through NAT Gateway
```

```
Private DB subnet route table:
Destination       Target
10.0.0.0/16      local          ← only talks to other VPC resources
(no default route)              ← no internet path at all
```

**Longest prefix match:** when multiple routes match a destination IP, the most specific one wins. `10.0.1.0/24` beats `10.0.0.0/16` for traffic to `10.0.1.50`.

### Internet Gateway (IGW)

Per AWS documentation: an internet gateway enables resources in public subnets to connect to the internet if the resource has a public IPv4 address or IPv6 address. Similarly, resources on the internet can initiate a connection to resources in your subnet using the public IP.

The IGW also performs one-to-one NAT for IPv4 — it translates the instance's private IP to its public IP for outbound traffic and back for inbound traffic. The EC2 instance itself only knows its private IP; the IGW handles the translation transparently.

There is no charge for the IGW itself. Data transfer charges apply for traffic flowing through it.

### NAT Gateway

Per AWS documentation: a NAT gateway is a Network Address Translation service. Instances in private subnets can connect to services outside the VPC, but external services cannot initiate a connection with those instances.

A public NAT Gateway:
- Lives in a public subnet
- Has an Elastic IP (static public IP) assigned at creation
- Private subnet routes `0.0.0.0/0` to the NAT Gateway
- NAT Gateway routes outbound traffic to the Internet Gateway
- Return traffic flows back to the NAT Gateway, which forwards it to the originating private instance

```
Private EC2 (10.0.10.25) wants to reach 1.2.3.4

10.0.10.25 → NAT Gateway (10.0.1.30, Elastic IP: 52.66.1.1)
             NAT Gateway → IGW → 1.2.3.4
             1.2.3.4 responds to 52.66.1.1
             NAT Gateway translates back → 10.0.10.25
```

The internet sees only `52.66.1.1`. The private instance's address never appears outside the VPC.

**AWS recommends** deploying a NAT Gateway in each AZ that contains resources requiring internet access, for high availability. A single NAT Gateway failure takes down outbound internet for any AZ routing through it.

### Security Groups

Stateful, instance-level firewalls. Attached to ENIs (Elastic Network Interfaces) of EC2 instances, RDS databases, load balancers, etc.

**Stateful** means connection state is tracked. If an inbound rule allows port 443, the response traffic is automatically permitted without a separate outbound rule. This mirrors how a real firewall tracks established connections.

Rules specify:
- Protocol (TCP, UDP, ICMP, or All)
- Port range
- Source (for inbound) or destination (for outbound) — can be CIDR, another Security Group ID, or prefix list

```
Security Group for web server:
Inbound:
  TCP 80    from 0.0.0.0/0           (HTTP from internet)
  TCP 443   from 0.0.0.0/0           (HTTPS from internet)
  TCP 22    from sg-bastion-id        (SSH only from bastion Security Group)

Outbound:
  All traffic → 0.0.0.0/0            (default — allow all outbound)
```

**Common audit finding:** Security Groups with TCP 22 (SSH), 3306 (MySQL), or 5432 (PostgreSQL) open to `0.0.0.0/0`. These allow the entire internet to attempt connections to sensitive services.

Security Groups use **allowlists only** — you cannot create deny rules. Default: all inbound denied, all outbound allowed.

### Network ACLs (NACLs)

Stateless subnet-level firewalls. Applied to all traffic entering or leaving a subnet, regardless of which instance it is going to or from.

**Stateless** means each packet is evaluated independently. If inbound TCP 443 is allowed, you must separately allow outbound traffic on the ephemeral port range (1024-65535) for the response packets to leave the subnet.

Rules are evaluated in numbered order (lowest first). First match wins. There is always an explicit deny-all at the end.

```
NACL (allow web traffic):
Rule 100  ALLOW  TCP 443  0.0.0.0/0  INBOUND
Rule 200  ALLOW  TCP 80   0.0.0.0/0  INBOUND
Rule 300  ALLOW  TCP 1024-65535  0.0.0.0/0  INBOUND  (return traffic)
Rule *    DENY   ALL                INBOUND
```

| | Security Group | NACL |
|--|---------------|------|
| Level | Instance/ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules in order, first match wins |

Use Security Groups as the primary control. NACLs are useful for blocking entire IP ranges at the subnet level — for example, blocking a known malicious IP range from reaching any resource in a subnet.

---

## Real Request Flow Through a VPC

```
User browser → Load balancer (public subnet, 10.0.1.50)
  Route: 0.0.0.0/0 → IGW (user comes from internet via IGW)

Load balancer → App server (private subnet, 10.0.10.25)
  Route: 10.0.0.0/16 → local (VPC-internal, no gateway)

App server → Database (private DB subnet, 10.0.20.10)
  Route: 10.0.0.0/16 → local

App server → External payment API (on internet)
  Route: 0.0.0.0/0 → NAT Gateway → IGW → internet

Database: no outbound internet path at all
```

The database is never reachable from the internet. It only receives inbound connections from the app subnet (enforced by Security Group: allow 5432 from app subnet CIDR only).

---

## Multi-AZ Design

An Availability Zone is a single data center building. Production systems span at least two AZs so a single building failure does not cause downtime.

Per AWS recommendations for production: deploy NAT Gateways in each AZ that contains resources requiring internet access. A NAT Gateway in AZ-a fails if AZ-a has an issue — private instances in AZ-b should have their own NAT Gateway in AZ-b.

RDS Multi-AZ runs a primary in one AZ and a synchronously replicated standby in another. Failover is automatic — takes 1–2 minutes for DNS to propagate to the new primary.

---

## AWS CLI

```bash
# List VPCs
aws ec2 describe-vpcs

# List subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxxxxxxxx"

# List route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-xxxxxxxxxx"

# Audit Security Groups for dangerous open ports
aws ec2 describe-security-groups \
  --query "SecurityGroups[?IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0'] && (FromPort==\`22\` || FromPort==\`3306\` || FromPort==\`5432\`)]].[GroupId,GroupName]" \
  --output table

# Create a VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create a subnet
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a

# Create and attach Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --internet-gateway-id igw-xxx --vpc-id vpc-xxx

# Add route to public subnet's route table
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| VPC | Logically isolated virtual network inside AWS |
| CIDR | Notation for IP ranges — `10.0.0.0/16` means 65,536 IPs |
| Subnet | Segment of VPC IP range, tied to one AZ |
| Public subnet | Has route to Internet Gateway — can receive inbound internet traffic |
| Private subnet | No route to IGW — cannot receive inbound from internet |
| Internet Gateway | Allows public subnets to reach and be reached from the internet |
| NAT Gateway | Allows private subnets to initiate outbound internet connections only |
| Security Group | Stateful, instance-level firewall (allow rules only) |
| NACL | Stateless, subnet-level firewall (allow and deny rules) |
| Route table | Rules that control where traffic from a subnet gets sent |
| Local route | Default VPC-internal route — always present, cannot be deleted |
| Elastic IP | Static public IP address assigned to a resource permanently |
| Multi-AZ | Deploying across multiple AZs for resilience against data center failure |
