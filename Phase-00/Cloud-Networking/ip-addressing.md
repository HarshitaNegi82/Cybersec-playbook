# IP Addressing — Binary, Subnet Masks & CIDR

---

## Binary & Dotted Decimal Notation

Every IPv4 address is 32 bits. Humans read it as four decimal numbers separated by dots. Computers work in binary. You need to be able to convert between both.

An IPv4 address has 4 octets (8 bits each). Each octet ranges from 0 to 255.

```
192      .    168     .     1      .    10
11000000    10101000   00000001   00001010
```

### Converting Decimal to Binary

Each bit position in an octet has a value:

```
Bit position:  128  64  32  16   8   4   2   1

Convert 192:
  128 fits → 1
  64 fits  → 1     (192 - 128 = 64)
  32 no    → 0     (64 - 64 = 0, stop)
  16 no    → 0
  8  no    → 0
  4  no    → 0
  2  no    → 0
  1  no    → 0
Result: 11000000 ✓
```

```
Convert 168:
  128 fits → 1    (168 - 128 = 40)
  64  no   → 0    (40 < 64)
  32  fits → 1    (40 - 32 = 8)
  16  no   → 0
  8   fits → 1    (8 - 8 = 0)
  4   no   → 0
  2   no   → 0
  1   no   → 0
Result: 10101000 ✓
```

Practice this until it is automatic. Subnetting depends entirely on binary.

### Converting Binary to Decimal

Multiply each bit by its position value and sum:

```
10101000
= 1×128 + 0×64 + 1×32 + 0×16 + 1×8 + 0×4 + 0×2 + 0×1
= 128 + 32 + 8
= 168 ✓
```

---

## Network ID, Host ID, and Subnet Masks

Every IP address has two parts: the **network portion** (identifies which network) and the **host portion** (identifies which device on that network).

The subnet mask defines the boundary.

```
IP address:   192.168.1.10
Subnet mask:  255.255.255.0

In binary:
IP:   11000000.10101000.00000001.00001010
Mask: 11111111.11111111.11111111.00000000

Network ID:   192.168.1    (first 24 bits — where the 1s are in the mask)
Host ID:      .10          (last 8 bits — where the 0s are in the mask)
```

The subnet mask in binary is always a continuous block of 1s followed by a continuous block of 0s. Never mixed.

### The AND Operation

To find the Network ID, AND the IP address with the subnet mask bit by bit:

```
1 AND 1 = 1
1 AND 0 = 0
0 AND 0 = 0

192.168.1.10  AND  255.255.255.0
= 192.168.1.0  ← Network address
```

Any IP that ANDs to the same network address is on the same network.

### CIDR Prefix Notation

`/24` means the first 24 bits are the network portion. Same as subnet mask `255.255.255.0`.

```
/8  = 11111111.00000000.00000000.00000000 = 255.0.0.0
/16 = 11111111.11111111.00000000.00000000 = 255.255.0.0
/24 = 11111111.11111111.11111111.00000000 = 255.255.255.0
/25 = 11111111.11111111.11111111.10000000 = 255.255.255.128
/26 = 11111111.11111111.11111111.11000000 = 255.255.255.192
/27 = 11111111.11111111.11111111.11100000 = 255.255.255.224
/28 = 11111111.11111111.11111111.11110000 = 255.255.255.240
```

### Hosts Per Subnet

Host bits = 32 - prefix length.
Total addresses = 2^(host bits).
Usable = 2^(host bits) - 2 (subtract network address and broadcast).
In AWS, subtract 5 total (AWS reserves 3 additional addresses).

```
/24 → 8 host bits → 256 total → 254 usable → 251 in AWS
/25 → 7 host bits → 128 total → 126 usable → 123 in AWS
/26 → 6 host bits → 64 total  → 62 usable  → 59 in AWS
/27 → 5 host bits → 32 total  → 30 usable  → 27 in AWS
/28 → 4 host bits → 16 total  → 14 usable  → 11 in AWS
```

---

## Private IP Ranges (RFC 1918)

Three ranges reserved for private networks. Not routable on the public internet — ISPs drop packets with these source addresses at their borders.

| Range | CIDR | Total IPs | Typical use |
|-------|------|-----------|-------------|
| `10.0.0.0` | `/8` | 16,777,216 | Large orgs, AWS VPCs |
| `172.16.0.0` | `/12` | 1,048,576 | Mid-size orgs |
| `192.168.0.0` | `/16` | 65,536 | Home networks |

AWS VPC CIDR blocks must be from these ranges. The most common choice is `10.0.0.0/16` for a VPC — gives you 65,536 addresses to carve into subnets.

### Why These Ranges Exist

IPv4 has 4,294,967,296 total addresses. The internet has billions of devices. RFC 1918 allows the same private IP to be reused in millions of private networks simultaneously — your home uses `192.168.1.1`, your office uses `192.168.1.1`, your neighbour uses `192.168.1.1`. None of them conflict because they are all private and never routable on the public internet. NAT handles translation when private devices need to reach the internet.

---

## Subnetting in Practice — Working Through It

Given a VPC CIDR of `10.0.0.0/16`, design subnets for a three-tier application across two Availability Zones.

```
VPC: 10.0.0.0/16 (65,536 IPs total)

Allocation:
  Public AZ-a:     10.0.1.0/24    (251 usable)
  Public AZ-b:     10.0.2.0/24    (251 usable)
  Private App AZ-a: 10.0.10.0/24  (251 usable)
  Private App AZ-b: 10.0.11.0/24  (251 usable)
  Private DB AZ-a:  10.0.20.0/24  (251 usable)
  Private DB AZ-b:  10.0.21.0/24  (251 usable)
```

Why the gaps (1, 2, then 10, 11, then 20, 21)? It leaves room to add subnets later without renumbering. A network designed without room to grow always needs to be redesigned.

### Are Two IPs in the Same Subnet?

To check if `10.0.1.50` and `10.0.1.200` are in `10.0.1.0/24`:
- AND `10.0.1.50` with `255.255.255.0` → `10.0.1.0` ✓
- AND `10.0.1.200` with `255.255.255.0` → `10.0.1.0` ✓
- Same network ID — same subnet. They can communicate directly.

To check if `10.0.1.50` and `10.0.2.50` are in the same subnet:
- AND `10.0.1.50` with `255.255.255.0` → `10.0.1.0`
- AND `10.0.2.50` with `255.255.255.0` → `10.0.2.0`
- Different — must go through a router.

---

## Subnet Calculator

Do not calculate subnets manually in production. Use tools:

**Online:**
- `https://www.subnet-calculator.com` — visual, shows network/broadcast/host range
- `https://cidr.xyz` — clean CIDR breakdown
- `https://www.ipaddressguide.com/cidr` — shows all IPs in a block

**Command line:**
```bash
# ipcalc — installed on most Linux systems
ipcalc 192.168.1.0/24
# Shows: Network, Broadcast, HostMin, HostMax, Hosts/Net

# Python — quick one-liners
python3 -c "import ipaddress; n = ipaddress.ip_network('10.0.1.0/24'); print(f'Network: {n.network_address}, Broadcast: {n.broadcast_address}, Hosts: {n.num_addresses - 2}')"

# Check if an IP is in a subnet
python3 -c "import ipaddress; print(ipaddress.ip_address('10.0.1.50') in ipaddress.ip_network('10.0.1.0/24'))"
# True

# List all subnets carved from a larger block
python3 -c "import ipaddress; [print(s) for s in ipaddress.ip_network('10.0.0.0/16').subnets(new_prefix=24)]"
```

**AWS context:**
```bash
# AWS CLI — describe subnets and see their CIDRs
aws ec2 describe-subnets \
  --query "Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,State]" \
  --output table

# Check if a given IP falls in any of your VPC subnets
aws ec2 describe-subnets \
  --query "Subnets[*].CidrBlock" \
  --output text
```

---

## IPv4 Exhaustion & IPv6

IANA (Internet Assigned Numbers Authority) allocated the last block of IPv4 addresses to regional registries in February 2011. IPv4 is exhausted at the allocation level. NAT extended IPv4's life by allowing private reuse, but this is a workaround not a solution.

### IPv6

128-bit addresses. Written as 8 groups of 4 hexadecimal digits separated by colons.

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334

Abbreviated (leading zeros dropped, consecutive zero groups replaced with ::):
2001:db8:85a3::8a2e:370:7334
```

Total IPv6 addresses: 3.4 × 10^38. Every device on Earth can have a globally unique address. NAT is not needed.

### IPv4 vs IPv6 comparison

| | IPv4 | IPv6 |
|--|------|------|
| Address size | 32 bits | 128 bits |
| Total addresses | ~4.3 billion | 3.4 × 10^38 |
| Notation | Dotted decimal | Hexadecimal with colons |
| NAT required | Yes (address exhaustion) | No |
| Header size | 20 bytes minimum | 40 bytes fixed |
| Broadcast | Yes | No (uses multicast) |

### AWS and IPv6

A VPC can have both an IPv4 CIDR block and an IPv6 CIDR block (dual-stack). AWS assigns IPv6 CIDR blocks from its own pool — you cannot bring your own IPv6 range by default.

For IPv6 in private subnets: use an **Egress-Only Internet Gateway** instead of a NAT Gateway. IPv6 addresses are globally routable, so a regular IGW would allow both inbound and outbound — the Egress-Only IGW blocks unsolicited inbound while allowing outbound.

Route table entries for IPv6 use `::/0` as the destination (equivalent of `0.0.0.0/0` for IPv4).

---

## Commands Reference

```bash
# View all interfaces and IP addresses
ip addr show

# View routing table
ip route show

# Subnet calculation
ipcalc 10.0.0.0/16
ipcalc 192.168.1.0/25

# Python subnet check
python3 -c "
import ipaddress
net = ipaddress.ip_network('10.0.0.0/16')
print('Network:', net.network_address)
print('Broadcast:', net.broadcast_address)
print('Total hosts:', net.num_addresses - 2)
print('First host:', list(net.hosts())[0])
print('Last host:', list(net.hosts())[-1])
"

# Check if two IPs are on the same subnet
python3 -c "
import ipaddress
ip1 = ipaddress.ip_address('10.0.1.50')
ip2 = ipaddress.ip_address('10.0.2.50')
net = ipaddress.ip_network('10.0.1.0/24')
print(ip1 in net, ip2 in net)
"

# AWS — list all VPC CIDRs
aws ec2 describe-vpcs --query "Vpcs[*].[VpcId,CidrBlock,IsDefault]" --output table

# AWS — describe subnets
aws ec2 describe-subnets \
  --query "Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]" \
  --output table
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Octet | 8-bit group in an IPv4 address (4 octets = 32 bits) |
| Network ID | Portion of IP identifying the network (determined by mask) |
| Host ID | Portion identifying a specific device on that network |
| Subnet mask | 32-bit value showing which bits are network vs host |
| CIDR | Classless Inter-Domain Routing — `/24` means 24 network bits |
| RFC 1918 | Standard defining private IP ranges |
| `/24` | 256 IPs, 254 usable, 251 in AWS |
| Broadcast address | Last IP in a subnet — cannot be assigned to a host |
| Network address | First IP in a subnet — cannot be assigned to a host |
| IPv6 | 128-bit addressing, globally unique, no NAT needed |
| Egress-Only IGW | IPv6 outbound-only gateway for private subnets |
