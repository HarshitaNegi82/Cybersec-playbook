# EC2, Firewalls & Network Security Controls

---

## Firewall Fundamentals

A firewall inspects network traffic and allows or blocks it based on rules.

### Host-based vs Network-based Firewalls

| | Host-based | Network-based |
|--|------------|---------------|
| Where it runs | On the individual machine | At the network perimeter or subnet boundary |
| What it protects | That one machine | All machines behind it |
| Examples | `iptables`, `ufw`, Windows Firewall | AWS Security Groups, NACLs, Cloudflare, physical firewall appliance |
| Visibility | Sees all traffic to/from that host | Sees all traffic crossing that network boundary |

In AWS, both layers exist simultaneously:
- Security Group = host-based (attached to each EC2 instance's network interface)
- NACL = network-based (applied at the subnet boundary)

You always want both. A Security Group misconfiguration on one instance does not affect others. A NACL misconfiguration affects every instance in that subnet.

### Stateful vs Stateless Firewalls

**Stateful:** tracks the state of each connection. If an inbound connection is allowed, return traffic is automatically permitted — the firewall remembers the connection was established.

```
Client → Server : TCP SYN on port 443   ← rule allows this
Server → Client : TCP SYN-ACK           ← automatically allowed (stateful)
Client → Server : TCP ACK               ← automatically allowed
[data flows both ways without extra rules]
```

**Stateless:** evaluates every packet independently with no memory of previous packets. You must explicitly allow both directions.

```
Inbound rule:  allow TCP 443 from anywhere  ← allows the request
Outbound rule: allow TCP 1024-65535 to anywhere  ← must exist or response is blocked
(no return rule = response packet is dropped even though you allowed the request)
```

AWS Security Groups = **stateful**. AWS NACLs = **stateless**.

---

## AWS Security Groups

A Security Group is a stateful, virtual firewall attached to an EC2 instance's Elastic Network Interface (ENI). It controls what traffic can reach and leave that specific instance.

### Key Properties

- **Allow rules only** — you cannot create a deny rule
- **Default inbound** — deny all (nothing can reach the instance unless explicitly allowed)
- **Default outbound** — allow all (instance can initiate any outbound connection)
- **Stateful** — return traffic for allowed connections is automatically permitted
- **Attached to ENI** — not to the subnet, not to the instance ID directly
- **Multiple SGs** — one instance can have multiple Security Groups; all rules are evaluated together

### Rule Structure

Each rule specifies:
- Protocol (TCP, UDP, ICMP, All)
- Port range (single port, range, or all)
- Source/destination (CIDR block, another Security Group ID, or prefix list)

```
Web server Security Group:
Inbound:
  TCP 80   0.0.0.0/0        ← HTTP from anywhere
  TCP 443  0.0.0.0/0        ← HTTPS from anywhere
  TCP 22   10.0.1.5/32      ← SSH only from bastion host IP

Outbound:
  All traffic  0.0.0.0/0    ← default, allow all outbound
```

### Security Group Referencing

Instead of specifying a CIDR as source, you can reference another Security Group. This is more robust than IP-based rules.

```
Database Security Group inbound rule:
  TCP 5432  source: sg-0abc123 (the App Security Group ID)
```

This means: only instances that have the App Security Group attached can connect to this database on port 5432. Even if an attacker knows the database IP, their instance doesn't have the App SG attached, so the rule doesn't match. This survives IP changes, auto-scaling, and instance replacements — the rule stays valid as long as the SG exists.

### Common Security Audit Findings

```bash
# Find Security Groups with SSH open to the world (critical finding)
aws ec2 describe-security-groups \
  --query "SecurityGroups[?IpPermissions[?FromPort==\`22\` && IpRanges[?CidrIp=='0.0.0.0/0']]].[GroupId,GroupName]" \
  --output table

# Find SGs with any port open to 0.0.0.0/0
aws ec2 describe-security-groups \
  --query "SecurityGroups[?IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0']]].[GroupId,GroupName,IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0']].FromPort]" \
  --output table

# Find instances with their Security Groups
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].[InstanceId,SecurityGroups[*].GroupId]" \
  --output table
```

The three ports that, if open to `0.0.0.0/0`, are always a critical finding in any cloud security audit:
- **22 (SSH)** — direct remote access
- **3389 (RDP)** — Windows remote desktop
- **5432/3306/1433** — database ports

---

## Network ACLs (NACLs)

A NACL is a stateless, subnet-level firewall. It applies to all traffic entering or leaving a subnet, regardless of which instance it is going to.

### Key Properties

- **Allow and deny rules** — unlike Security Groups, NACLs can explicitly deny
- **Stateless** — must explicitly allow both inbound and outbound
- **Evaluated in order** — rules have numbers; lowest number evaluated first; first match wins
- **Explicit deny at end** — rule `*` denies everything not matched above
- **One per subnet** — a subnet can only be associated with one NACL
- **Default NACL** — allows all inbound and outbound (created with every VPC)

### Ephemeral Ports Problem (Stateless Trap)

When a client connects to your server on port 443, the response traffic comes from the server on port 443 back to the client's **ephemeral port** (random port 1024–65535). In a stateful Security Group, this is handled automatically. In a NACL, you must explicitly allow it:

```
NACL for web server subnet:
Inbound:
  Rule 100: ALLOW TCP 443  0.0.0.0/0   ← HTTPS requests in
  Rule 200: ALLOW TCP 1024-65535 0.0.0.0/0 ← return traffic from internet (responses to outbound requests)

Outbound:
  Rule 100: ALLOW TCP 443  0.0.0.0/0   ← HTTPS requests out
  Rule 200: ALLOW TCP 1024-65535 0.0.0.0/0 ← responses going out to clients
```

If you forget the ephemeral port range in either direction, connections mysteriously fail even though the Security Group is correctly configured.

### Security Groups vs NACLs

| | Security Group | NACL |
|--|----------------|------|
| Level | Instance (ENI) | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules together | In order, first match |
| Default | Deny all inbound, allow all outbound | Allow all |
| Use case | Primary traffic control | Block IP ranges, extra subnet layer |

Use Security Groups as your primary control. Use NACLs to block specific known-bad IP ranges or to add a subnet-wide deny that Security Groups cannot do.

```bash
# Describe NACLs and their rules
aws ec2 describe-network-acls \
  --query "NetworkAcls[*].[NetworkAclId,IsDefault,Entries[*].[RuleNumber,Protocol,RuleAction,CidrBlock,PortRange]]" \
  --output json

# Associate a NACL with a subnet
aws ec2 replace-network-acl-association \
  --association-id aclassoc-xxxxxxxxxx \
  --network-acl-id acl-xxxxxxxxxx
```

---

## EC2 Instance Launching

When you launch an EC2 instance, the network configuration decisions you make at launch time determine whether it is reachable:

**1. Subnet choice** — determines which AZ, whether it has internet access (route table), and what IP range it gets from.

**2. Auto-assign public IP** — if enabled, instance gets a public IP automatically. If disabled, instance is only reachable by private IP within the VPC. For instances in private subnets, this should always be disabled.

**3. Security Group assignment** — at minimum, restrict inbound to only what is needed.

**4. Key pair** — for SSH access. If no key pair is assigned, you cannot SSH unless you use EC2 Instance Connect or Systems Manager.

```bash
# Launch an EC2 instance via CLI
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --subnet-id subnet-xxxxxxxxxx \
  --security-group-ids sg-xxxxxxxxxx \
  --key-name my-keypair \
  --no-associate-public-ip-address \  # private subnet — no public IP
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=AppServer}]'

# Describe running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,PublicIpAddress,SubnetId,State.Name]" \
  --output table
```

---

## EC2 Instance Connect Endpoint

EC2 Instance Connect Endpoint (EIC Endpoint) lets you SSH into a private EC2 instance without:
- A bastion host
- A public IP on the instance
- Opening port 22 in a Security Group to the internet

### How It Works

```
Your laptop
    │
    │  AWS creates a tunnel through EIC Endpoint
    ▼
EIC Endpoint (lives inside your VPC)
    │
    │  Connects to instance's private IP on port 22
    ▼
Private EC2 instance (10.0.10.25) — no public IP
```

The EIC Endpoint is a VPC resource. AWS handles the secure tunnel from your browser or CLI through IAM authentication to the endpoint, which then connects to the instance's private IP on port 22.

### Security Group Rule Required

The Security Group on the target EC2 instance must allow inbound TCP 22 from the EIC Endpoint's IP or Security Group. You do not open port 22 to `0.0.0.0/0`.

```bash
# Create an EIC Endpoint
aws ec2 create-instance-connect-endpoint \
  --subnet-id subnet-xxxxxxxxxx \
  --security-group-ids sg-xxxxxxxxxx

# Connect to a private instance using EIC Endpoint
aws ec2-instance-connect ssh \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --os-user ec2-user

# Alternative — open tunnel then SSH manually
aws ec2-instance-connect open-tunnel \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --local-port 2222
# Then in another terminal:
ssh -p 2222 ec2-user@localhost -i my-keypair.pem
```

### IAM Policy Required

The user or role calling `ec2-instance-connect:OpenTunnel` must have permission:

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2-instance-connect:OpenTunnel",
    "ec2-instance-connect:SendSSHPublicKey",
    "ec2:DescribeInstances"
  ],
  "Resource": "*"
}
```

---

## Testing NAT Gateway Connectivity

After setting up a NAT Gateway, verify private instances can reach the internet:

```bash
# SSH into private instance (via bastion or EIC Endpoint)
# Then from the private instance:

# Test outbound connectivity
curl -s ifconfig.me
# Should return the NAT Gateway's Elastic IP, not the instance's private IP

# Trace the route — first hop should be the NAT Gateway's private IP
traceroute 8.8.8.8

# DNS resolution test
nslookup google.com

# Test HTTPS outbound
curl -I https://google.com

# If any of these fail:
# 1. Check the private subnet's route table has 0.0.0.0/0 → NAT Gateway
# 2. Check the NAT Gateway is in a public subnet
# 3. Check the public subnet has 0.0.0.0/0 → Internet Gateway
# 4. Check the Security Group allows outbound (default: yes)
# 5. Check NACL allows outbound and return traffic on ephemeral ports
```

---

## Testing NACL Rules

NACLs are the hardest AWS networking component to debug because they are stateless and because a misconfigured NACL rule can silently break traffic while the Security Group looks correct.

```bash
# On a Linux instance — test if a port is reachable
nc -zv 10.0.20.10 5432
# Connection succeeded = Security Group + NACL both allow it
# Connection refused  = Security Group blocking it (NACL let it through, nothing listening or SG blocked)
# Connection timed out = NACL blocking it (packet dropped, no response)

# The difference between refused and timed out is critical:
# Refused = something received the packet and rejected it
# Timed out = the packet never arrived (NACL dropped it silently)

# Check NACL rules via CLI
aws ec2 describe-network-acls \
  --network-acl-ids acl-xxxxxxxxxx \
  --query "NetworkAcls[*].Entries[*].[RuleNumber,RuleAction,CidrBlock,Protocol,PortRange]" \
  --output table

# AWS Reachability Analyzer — tests network paths and identifies what is blocking traffic
aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-xxxxxxxxxx
```

**AWS Reachability Analyzer** is the most useful tool for NACL/Security Group debugging. You specify source and destination, and it tells you exactly which rule in which Security Group or NACL is blocking the traffic — without sending any actual packets.

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Security Group | Stateful, instance-level firewall — allow rules only |
| NACL | Stateless, subnet-level firewall — allow + deny rules |
| Stateful | Connection state tracked — return traffic auto-allowed |
| Stateless | Each packet evaluated independently — must allow both directions |
| Ephemeral ports | 1024–65535 — used for return traffic in stateless firewalls |
| SG referencing | Using another SG as source instead of a CIDR — more robust |
| EIC Endpoint | SSH to private instances without bastion or public IP |
| Reachability Analyzer | AWS tool to diagnose network connectivity issues |
| Host-based firewall | Runs on the machine (Security Group, iptables) |
| Network-based firewall | At the subnet or perimeter level (NACL, dedicated appliance) |
