# DHCP — Dynamic Host Configuration Protocol

Port: **UDP 67 (server), UDP 68 (client)**

---

## What It Does

Automatically assigns IP configuration to a device when it joins a network. Without DHCP, every device needs manual IP configuration — not scalable beyond a handful of machines.

DHCP hands out four things at once:

| Item | Example | Purpose |
|------|---------|---------|
| IP address | `192.168.1.42` | Your address on this network |
| Subnet mask | `255.255.255.0` | Defines which IPs are local vs external |
| Default gateway | `192.168.1.1` | The router — exit door to other networks |
| DNS server | `8.8.8.8` | Who to ask for name resolution |

This is the direct link between DHCP and DNS — DHCP is what tells your device which DNS resolver to use. Without it, the device has no idea who to ask for `google.com`.

---

## DORA — The 4-Step Process

```
Step 1 — DISCOVER
Device joins network, has no IP yet.
Broadcasts to 255.255.255.255:
"Is there a DHCP server? I need an address."
Source IP: 0.0.0.0 (device has nothing yet)

Step 2 — OFFER
DHCP server receives the broadcast.
Replies: "I can offer you 192.168.1.42 for 24 hours."
Includes: IP, subnet mask, gateway, DNS server, lease time.

Step 3 — REQUEST
Device broadcasts: "I accept 192.168.1.42 from server 192.168.1.1."
Broadcast not unicast — other DHCP servers (if any) also see the rejection.

Step 4 — ACKNOWLEDGE
DHCP server confirms: "192.168.1.42 is yours for 24 hours."
Device configures itself. Communication begins.
```

DORA uses **broadcast** because the device has no IP yet — it cannot make a unicast request to anyone.

---

## Lease and Renewal

The IP is not permanent. It is a **lease** with a time limit (default: 8 hours to 24 hours depending on router).

```
Lease granted: 192.168.1.42 for 24 hours

At 50% of lease time (12 hours):
  Device sends unicast DHCPREQUEST to the same server
  Server sends DHCPACK — lease extended

At 87.5% of lease time (21 hours), if first renewal failed:
  Device broadcasts DHCPREQUEST to any available server
  If no response by lease expiry — device loses its IP
```

The same device usually gets the same IP on renewal because DHCP servers track MAC → IP mappings.

---

## DHCP and MAC Addresses

DHCP uses MAC addresses to identify clients and track leases. The DISCOVER and REQUEST messages include the client's MAC address. The server stores `MAC → IP → lease expiry` in its lease table.

This is also why DHCP reservation works — you configure a specific MAC to always receive a specific IP. The device still goes through DORA, but the server always offers the same pre-assigned address.

---

## Static IP vs DHCP — The Real Difference

Setting a static IP on a device opts that one device out of receiving a DHCP assignment. The DHCP server keeps running for all other devices.

A static IP is only valid on the specific network it was configured for:

```
Home network:  192.168.1.0/24
Static IP set: 192.168.1.50

Take same device to office: 10.0.5.0/24
  Static IP 192.168.1.50 is outside 10.0.5.0/24
  Device cannot communicate — wrong range entirely
```

Why use static IPs at all:
- Servers and databases that other devices need to find reliably
- Port forwarding rules that target a specific internal IP
- Network printers, security cameras
- Anything that must always be at the same address

---

## Rogue DHCP Attack

Since DHCP uses broadcast and has no authentication, any device on the network can respond to a DISCOVER with an OFFER. An attacker runs a rogue DHCP server that hands out:
- Legitimate IP address
- Attacker's machine as the default gateway → all traffic routed through attacker
- Attacker's DNS server → all name resolution controlled by attacker

This is a man-in-the-middle attack at the network provisioning layer.

**Defence:** DHCP snooping on managed switches — only designated trusted ports are allowed to send DHCP OFFER and ACK messages. Untrusted ports (client ports) have DHCP server messages dropped.

---

## DHCP in AWS VPC

AWS VPCs do not run a traditional broadcast-based DHCP server. Each VPC has a DHCP option set that defines:
- Domain name
- DNS servers (by default, AWS provides a resolver at VPC base IP + 2, e.g., `10.0.0.2`)
- NTP servers
- NetBIOS settings

When an EC2 instance launches, it gets its private IP assigned from the subnet's CIDR range. This happens through an AWS-managed process that behaves like DHCP from the instance's perspective (the instance runs a DHCP client) but does not involve traditional Layer 2 broadcasts within AWS's infrastructure.

The VPC's DNS resolver at `VPC_CIDR_BASE + 2` is what handles DNS for all EC2 instances by default. You can override this by modifying the DHCP option set.

---

## Commands

```bash
# View current IP configuration including DHCP-assigned values
ip addr show
ip route show

# Linux — view DHCP lease information
cat /var/lib/dhcp/dhclient.leases

# Release current DHCP lease
sudo dhclient -r eth0

# Request new DHCP lease
sudo dhclient eth0

# On systemd-networkd systems
networkctl status

# Capture DHCP traffic (all 4 DORA steps visible)
sudo tcpdump -i eth0 port 67 or port 68 -v

# Windows
ipconfig /release
ipconfig /renew
ipconfig /all   # shows DHCP server IP, lease obtained/expires times
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| DORA | Discover, Offer, Request, Acknowledge — the 4 DHCP steps |
| Lease | Temporary IP assignment with an expiry time |
| DHCP reservation | Static assignment by MAC address via DHCP server |
| Rogue DHCP | Attacker's unauthorized DHCP server handing out malicious config |
| DHCP snooping | Switch feature blocking DHCP offers from untrusted ports |
| UDP 67/68 | DHCP server/client ports |
