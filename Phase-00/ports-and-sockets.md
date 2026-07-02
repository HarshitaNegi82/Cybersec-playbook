# Ports, Sockets & Ephemeral Ports

Layer: **Transport (Layer 4)**

---

## Ports

An IP address identifies a machine. A port identifies a specific service or connection on that machine.

Port numbers are 16-bit integers → values 0 to 65535 → 65,536 total ports.

**Port ranges:**

| Range | Name | Usage |
|-------|------|-------|
| 0–1023 | Well-known ports | Standard services — SSH (22), HTTP (80), HTTPS (443) |
| 1024–49151 | Registered ports | Application-specific, registered with IANA |
| 49152–65535 | Ephemeral ports | Temporary, assigned by OS for outgoing connections |

**Common ports to know:**

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 25 | TCP | SMTP |
| 53 | UDP/TCP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 445 | TCP | SMB |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 27017 | TCP | MongoDB |

---

## The 4-Tuple — How Connections Are Uniquely Identified

Every TCP/UDP connection is uniquely identified by four values together:

```
Source IP : Source Port → Destination IP : Destination Port
```

Two simultaneous connections to the same server on the same port are kept separate because the source ports differ:

```
192.168.1.5:55001 → 142.250.195.46:443   (browser tab 1 - google.com)
192.168.1.5:55002 → 142.250.195.46:443   (browser tab 2 - google.com)
```

Same source IP, same destination IP, same destination port — different source ports. The OS uses these 4-tuples to route incoming responses to the correct connection.

---

## Ephemeral Ports

When your machine initiates a connection, it does not use port 443 locally — that is the server's port. Your OS picks a random unused port from the ephemeral range (typically 49152–65535 on Linux, 1024–65535 on Windows) as the source port.

```
Your laptop:
  Tab 1 → google.com:443   using local port 55001
  Tab 2 → github.com:443   using local port 55002
  SSH session → server:22  using local port 55003
```

The server sees requests arriving from `your_ip:55001`, `your_ip:55002`, `your_ip:55003`. When it responds, it sends to those specific ports. Your OS maps each incoming packet to the right process using the 4-tuple.

---

## Sockets

A socket is the OS-level object that represents one endpoint of a network connection. It binds an IP address to a port and provides a file-like interface for a process to send and receive data.

**A listening server socket:**

```bash
python3 -m http.server 8080
```

What happens internally:
```
1. Python asks the OS kernel: "Create a socket on 0.0.0.0:8080"
2. Kernel creates the socket and marks it as LISTEN state
3. Browser connects → kernel receives SYN on port 8080
4. Kernel checks: is there a socket in LISTEN state on port 8080? Yes
5. Kernel completes the TCP handshake
6. Kernel creates a new socket for this specific connection
7. Hands the connection to Python's HTTP server
```

When you press CTRL+C:
```
1. Python process dies
2. Kernel closes the listening socket
3. Port 8080 is now "open" as a port number but nothing is listening
4. New browser request arrives → kernel checks for socket on 8080 → none
5. Kernel sends TCP RST back → browser shows "Connection Refused"
```

**Connection Refused vs Timeout:**
- **Connection Refused** — port exists but nothing is listening (active RST sent back)
- **Timeout** — firewall is silently dropping packets (no response at all)

---

## Reading `ss` Output

`ss` asks the kernel for a snapshot of all sockets currently existing.

```bash
ss -tulpn
```

Flags: `-t` TCP, `-u` UDP, `-l` listening, `-p` process name, `-n` numeric ports

```
Netid  State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
tcp    LISTEN  0       128     0.0.0.0:22           0.0.0.0:*          sshd
tcp    LISTEN  0       5       127.0.0.1:5432       0.0.0.0:*          postgres
tcp    ESTAB   0       0       10.0.0.5:22          203.0.113.10:55102 sshd
udp    UNCONN  0       0       0.0.0.0:68           0.0.0.0:*          dhclient
```

**Reading this:**
- `0.0.0.0:22` — SSH listening on all interfaces. Any IP can reach it.
- `127.0.0.1:5432` — PostgreSQL listening only on loopback. Only local processes can connect. Not reachable from network.
- `ESTAB` — established connection. Active SSH session from `203.0.113.10`.
- `UNCONN` UDP — UDP has no connection state. Just means it is bound and listening.

**Security relevance:** `0.0.0.0` on a sensitive port is dangerous if the Security Group or firewall allows it. `127.0.0.1` on a database is correct — it means the database is not network-accessible.

---

## Loopback Interface

`127.0.0.1` (IPv4) / `::1` (IPv6) — the loopback address. Traffic sent to this address never leaves the machine. It goes through a virtual interface (`lo`) entirely within the OS networking stack.

```bash
ping 127.0.0.1    # tests the OS networking stack itself
                  # does NOT test NIC, cable, router, or anything external
```

Use cases:
- Services that should only be accessible locally (database listening on 127.0.0.1)
- Testing locally before deploying
- Containers communicating with host services

Loopback traffic does not appear on physical interfaces (`eth0`, `wlan0`). `tcpdump -i lo` captures it. `tcpdump -i eth0` will never show it.

---

## Default Gateway and Routing Decision

When a device wants to send a packet, it first checks: is the destination IP in my local subnet?

```
My IP:      192.168.1.42
Subnet:     255.255.255.0 → local range is 192.168.1.0 to 192.168.1.255

Destination 192.168.1.100 → local → send directly (ARP to find MAC)
Destination 142.250.195.46 → not local → send to default gateway (router)
```

The default gateway is the router's IP on the local network. It is the exit point for all non-local traffic.

```bash
ip route show
# default via 192.168.1.1 dev wlan0   ← default gateway
# 192.168.1.0/24 dev wlan0 proto kernel ← local subnet
```

Without a default gateway configured, a device can only communicate with devices on its own subnet. Internet access is impossible.

---

## Commands

```bash
# All listening sockets with process names
ss -tulpn

# Active connections only
ss -tn state established

# Count connections per state
ss -s

# Watch connections in real time
watch -n 1 ss -tulpn

# Create a simple listening TCP server (for testing)
nc -lvnp 4444

# Connect to a specific IP:port (test if port is open)
nc -zv 192.168.1.1 443

# Scan open ports (only on your own machines)
nmap -sV 192.168.1.1

# View routing table
ip route show

# Add a static route
ip route add 10.0.5.0/24 via 192.168.1.1

# Check default gateway
ip route show default
```

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Port | 16-bit number identifying a specific service or connection endpoint |
| Well-known ports | 0–1023, assigned to standard services |
| Ephemeral port | Temporary port assigned by OS for an outgoing connection |
| 4-tuple | Source IP, source port, destination IP, destination port — uniquely identifies a connection |
| Socket | OS object binding an IP+port to a process |
| LISTEN | Socket waiting for incoming connections |
| ESTABLISHED | Active connection |
| Connection Refused | Port exists but nothing is listening (RST sent back) |
| Timeout | Firewall silently dropping packets (no response) |
| Loopback | 127.0.0.1 — traffic stays within the OS, never hits the network |
| Default gateway | Router's IP — where to send traffic that isn't destined for the local subnet |
