# ARP Spoofing

ARP spoofing (also called ARP poisoning) is an attack where a malicious device sends fake ARP replies to associate its own MAC address with another device's IP address on a local network. Traffic intended for the victim's IP gets delivered to the attacker instead — enabling man-in-the-middle interception or denial of service.

---

## ARP in Normal Operation

ARP (Address Resolution Protocol) resolves an IP address to a MAC address within a local network. Layer 2 (Ethernet/WiFi) needs MAC addresses to deliver frames to the right physical device. Layer 3 (IP) knows where to send traffic by IP. ARP bridges this gap.

```
Device A wants to send to 192.168.1.20 on the same LAN

Step 1: Check ARP cache
  If 192.168.1.20 → MAC is already cached, use it directly

Step 2: If not cached — send ARP Request (broadcast)
  "Who has 192.168.1.20? Tell 192.168.1.5"
  Broadcast to FF:FF:FF:FF:FF:FF — every device on the LAN receives it

Step 3: Device with 192.168.1.20 replies (unicast)
  "192.168.1.20 is at AA:BB:CC:DD:EE:FF"

Step 4: Device A caches the mapping and uses the MAC
```

ARP has no authentication. Any device can send any ARP reply at any time, and the recipient will update its ARP cache.

---

## The Attack

```
Legitimate state:
  192.168.1.1 (gateway) → MAC: 11:22:33:44:55:66
  192.168.1.20 (victim) → MAC: AA:BB:CC:DD:EE:FF

Attacker sends gratuitous ARP replies to victim:
  "192.168.1.1 is at 66:77:88:99:AA:BB"  (attacker's MAC)

Victim updates ARP cache:
  192.168.1.1 (gateway) → MAC: 66:77:88:99:AA:BB  ← now points to attacker

Attacker also poisons the gateway:
  "192.168.1.20 is at 66:77:88:99:AA:BB"  (attacker's MAC)

Gateway updates ARP cache:
  192.168.1.20 → MAC: 66:77:88:99:AA:BB  ← now points to attacker

Result:
  All traffic between victim and gateway flows through the attacker
  Attacker reads, modifies, or drops it
```

---

## Why This Is Strictly Local

ARP broadcast frames are never forwarded by routers. A router creates a boundary for Layer 2 broadcasts — the ARP request stays within the local subnet. An attacker on the internet cannot ARP-spoof your devices. The attacker must be physically present on the same network segment (same LAN, same WiFi network).

This is why ARP spoofing is a major threat in:
- Corporate offices (internal attacker or compromised device)
- Public WiFi (café, airport, hotel)
- Any shared network environment

---

## Defenses

**Dynamic ARP Inspection (DAI):**  
A managed switch feature that validates ARP packets against a DHCP snooping binding table. The switch knows which IP-MAC mappings are legitimate (from DHCP assignments) and drops ARP replies that do not match.

**Static ARP entries:**  
For critical devices (gateway router), configure static ARP entries on client machines so they cannot be overwritten by ARP replies.

**VLANs:**  
Network segmentation using VLANs limits how many devices are on the same broadcast domain. Even if an attacker poisons ARP within their VLAN, other VLANs are unaffected.

**Monitoring:**  
Detect by watching for ARP replies that appear without a preceding request, or IP-MAC mapping changes for known critical devices.

---

## Tools (Lab Environment Only)

- `arpspoof` — part of the `dsniff` package
- `ettercap` — GUI and CLI tool for ARP poisoning and MITM
- `bettercap` — modern, extensible framework

View your current ARP cache:
```bash
arp -a
ip neigh
```
