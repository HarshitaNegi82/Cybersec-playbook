# AWS Global Accelerator & Edge Networking

---

## The Problem Global Accelerator Solves

When a user in India connects to your application hosted in `us-east-1` (N. Virginia):

```
User (Mumbai)
    │
    │  Travels over public internet
    │  Multiple hops through various ISP networks
    │  Variable latency, packet loss, route changes
    ▼
Application (us-east-1, N. Virginia)
```

The public internet is unpredictable. Routing between ISPs is optimised for cost, not performance. A packet from Mumbai to Virginia might travel through 15–20 routers across multiple ISPs, each adding latency and introducing potential for packet loss.

AWS Global Accelerator solves this by getting traffic onto AWS's own private global network as quickly as possible — reducing the distance traffic travels over the unpredictable public internet.

---

## How Global Accelerator Works

### Step 1 — Anycast IP Addresses

Global Accelerator assigns your application **two static anycast IPv4 addresses** (and optionally IPv6). These IPs are announced simultaneously from all AWS edge locations worldwide using anycast routing.

```
Your application gets: 75.2.26.83 and 35.71.157.12
These two IPs are announced from 90+ edge locations globally.
```

When a user queries your application's IP:
- Their BGP routing automatically sends them to the **nearest AWS edge location**
- Not because AWS chose it — because the internet's routing protocol chose the shortest path to that IP

### Step 2 — AWS Private Network

From the edge location, traffic travels over **AWS's private global backbone** — dedicated fibre infrastructure, not the public internet.

```
User (Mumbai)
    │
    │ ~5ms over public internet
    ▼
AWS Edge Location (Mumbai)
    │
    │ AWS private backbone (optimised routing, SLAs)
    ▼
Your Application (us-east-1)
```

Instead of 20 hops across multiple ISPs, traffic hops twice — user to nearest edge, edge to your application over AWS's own network.

---

## Anycast vs Unicast

**Unicast:** one IP address → one specific server. Standard routing. Every packet to `52.66.1.1` goes to that one server.

**Anycast:** one IP address → multiple servers (nearest one responds). The same IP is advertised from many locations. BGP routing sends each query to the nearest location.

```
Anycast IP: 75.2.26.83

User in Mumbai  → routed to AWS Mumbai edge
User in London  → routed to AWS London edge
User in New York → routed to AWS Virginia edge

All using the same IP: 75.2.26.83
```

This is the same mechanism Cloudflare uses for `1.1.1.1` and how AWS Route 53 provides low-latency DNS globally.

**Why two static IPs matter:** Your users, firewalls, and partner systems can whitelist these two IPs permanently. Even if your application moves between AWS regions, the anycast IPs stay the same — Global Accelerator handles the routing change. With standard DNS, a region change requires DNS propagation (minutes to hours depending on TTL). With Global Accelerator's static IPs, the change is near-instant.

---

## DNS vs Global Accelerator

Both can route users to the nearest or healthiest endpoint. Different mechanisms.

| | Route 53 DNS | Global Accelerator |
|--|-------------|-------------------|
| Mechanism | DNS response returns nearest endpoint IP | Anycast IP + AWS private network |
| User experience | Client resolves DNS to nearest server | Client connects to nearest edge, backend is transparent |
| Latency reduction | Reduces DNS lookup time | Reduces actual data transfer latency |
| IP stability | IP changes with DNS record | Two static IPs always |
| Protocol | DNS (UDP/TCP port 53) | TCP, UDP — works at network layer |
| Failover speed | DNS TTL (seconds to minutes) | Sub-minute (no DNS propagation) |

Global Accelerator works below DNS — it is a network-layer service. This means it works for **any TCP or UDP application**, not just HTTP/HTTPS. UDP game servers, real-time communications, custom protocols — all benefit from Global Accelerator.

---

## AWS Edge Locations and Global Network

AWS has three tiers of infrastructure:

**Regions** — full AWS data centres with all services. `ap-south-1` (Mumbai), `us-east-1` (N. Virginia), etc.

**Availability Zones** — isolated data centre buildings within a region. Each region has 2–6 AZs.

**Edge Locations** — lightweight nodes used for CloudFront CDN and Global Accelerator. 90+ globally. Do not run full AWS services — only caching and network acceleration. Much more numerous than regions.

```
AWS Regions: ~30
AWS Availability Zones: ~90
AWS Edge Locations: 400+
```

Global Accelerator and CloudFront use edge locations to get close to users globally without needing a full region in every city.

---

## Failover and High Availability

Global Accelerator continuously monitors your application endpoints using health checks. If an endpoint becomes unhealthy, traffic is automatically rerouted to the next healthy endpoint — without any DNS change, without TTL wait, without user-visible interruption.

```
Normal state:
  75.2.26.83 → us-east-1 (primary, healthy)

us-east-1 becomes unhealthy:
  Global Accelerator detects within seconds
  75.2.26.83 → eu-west-1 (secondary, healthy)
  Users automatically routed to secondary — same IP, same connection
```

**Why this is faster than DNS failover:**
DNS-based failover requires:
1. Health check detects failure
2. DNS record updated
3. Resolvers worldwide cache the old record until TTL expires
4. Users get the new IP when their TTL expires

TTL of 60 seconds means some users see the failure for up to 60 seconds. With a TTL of 300 seconds, up to 5 minutes. Global Accelerator has no TTL to wait for — it reroutes at the network layer in under a minute.

### Traffic Dials and Endpoint Weights

Global Accelerator allows fine-grained traffic control:

**Traffic dial:** set a percentage of traffic for each endpoint group. Set `us-east-1` to 100% and `eu-west-1` to 0% for normal operations. Shift to 50/50 for testing, or 0/100 to drain a region before maintenance.

**Endpoint weights:** within an endpoint group, assign weights to distribute traffic across multiple instances or load balancers.

---

## Key Features

**Static IP addresses** — two anycast IPs that never change, regardless of backend changes. Critical for applications where clients whitelist IPs.

**TCP and UDP support** — unlike CloudFront which is HTTP/HTTPS only, Global Accelerator works with any TCP or UDP traffic. Gaming, VoIP, custom protocols.

**DDoS protection** — Global Accelerator is integrated with AWS Shield Standard. Volumetric DDoS attacks are absorbed at edge locations. The origin never sees the full attack volume.

**Client IP preservation** — for TCP traffic, you can configure Global Accelerator to preserve the original client IP address so your application sees where requests are actually coming from (not just the edge location's IP).

---

## Common Use Cases

| Use case | Why Global Accelerator |
|----------|----------------------|
| Global web application | Reduce latency for users far from your region |
| Real-time gaming | UDP support, low latency, stable IPs |
| IP whitelisting required | Static anycast IPs never change |
| Multi-region failover | Sub-minute failover without DNS propagation |
| VoIP / real-time comms | Low latency UDP, consistent performance |
| API with SLA requirements | Private backbone, not internet-dependent |

---

## Global Accelerator vs CloudFront

Both use edge locations. Both reduce latency. Different use cases.

| | CloudFront | Global Accelerator |
|--|-----------|-------------------|
| What it caches | Static content (images, JS, HTML) | Nothing — pure routing |
| Protocols | HTTP/HTTPS only | TCP + UDP |
| Benefit | Cache hit = no backend request | Always reaches backend, just faster |
| Use for | Static assets, streaming | Dynamic apps, gaming, any TCP/UDP |

For a typical web app: use CloudFront for static content caching + Global Accelerator for API and dynamic traffic.

---

## Commands Reference

```bash
# Create a Global Accelerator
aws globalaccelerator create-accelerator \
  --name my-accelerator \
  --ip-address-type IPV4 \
  --enabled \
  --region us-west-2  # Global Accelerator API is in us-west-2 regardless of your app's region

# List accelerators and their static IPs
aws globalaccelerator list-accelerators \
  --region us-west-2 \
  --query "Accelerators[*].[Name,IpSets[0].IpAddresses,Status]" \
  --output table

# Create listener (which ports/protocols to accept)
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/xxxxxxxx \
  --port-ranges '[{"FromPort":80,"ToPort":80},{"FromPort":443,"ToPort":443}]' \
  --protocol TCP \
  --region us-west-2

# Create endpoint group (which region/endpoints to send traffic to)
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/xxxxxxxx/listener/xxxxxxxx \
  --endpoint-group-region ap-south-1 \
  --traffic-dial-percentage 100 \
  --endpoint-configurations "EndpointId=arn:aws:elasticloadbalancing:ap-south-1:123456789012:loadbalancer/app/my-alb/xxxx,Weight=100,ClientIPPreservationEnabled=true" \
  --region us-west-2

# Check health of endpoints
aws globalaccelerator describe-endpoint-group \
  --endpoint-group-arn arn:... \
  --region us-west-2 \
  --query "EndpointGroup.EndpointDescriptions[*].[EndpointId,HealthState,HealthReason]" \
  --output table
```

**Note:** All Global Accelerator API calls must go to `us-west-2` regardless of where your application is hosted. It is a global service with a single control plane endpoint.

---

## Quick Reference

| Term | Meaning |
|------|---------|
| Global Accelerator | AWS service routing traffic through private backbone via anycast IPs |
| Anycast | Same IP announced from multiple locations — nearest responds |
| Edge location | Lightweight AWS node for CDN and acceleration (400+ globally) |
| Static IP | Two permanent anycast IPs assigned to your accelerator — never change |
| Traffic dial | Percentage of traffic sent to each endpoint group (0–100%) |
| AWS backbone | AWS's private global fibre network — not the public internet |
| CloudFront | CDN for caching static content — HTTP/HTTPS only |
| Endpoint group | A set of endpoints in a specific AWS region |
| Client IP preservation | Passes original client IP to backend instead of edge location IP |
| Sub-minute failover | Global Accelerator re-routes in <60s vs DNS TTL-based failover |
