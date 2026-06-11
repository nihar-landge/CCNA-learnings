# Module 1 — Network Fundamentals
## Cisco G4 Interview Preparation · Print & Revision Notes

> **How to use:** Read once to learn. Cover the right column and test yourself. Speak every interview answer aloud. Tick each section when you can answer it without looking.

---

# SECTION 1 — OSI & TCP/IP Models

---

## 1.1 OSI Model — 7 Layers

> **Memory trick (bottom to top):** **"Please Do Not Throw Sausage Pizza Away"**
> Physical → Data Link → Network → Transport → Session → Presentation → Application

| # | Layer | PDU | Core Job | Protocols / Examples |
|---|---|---|---|---|
| 7 | Application | Data | User interface to the network | HTTP, HTTPS, DNS, FTP, SMTP |
| 6 | Presentation | Data | Encoding, encryption, compression | SSL/TLS, JPEG, ASCII |
| 5 | Session | Data | Establish, manage, end sessions | NetBIOS, RPC |
| 4 | Transport | **Segment** | End-to-end delivery, ports, reliability | TCP, UDP |
| 3 | Network | **Packet** | Logical IP addressing, routing | IP, ICMP, ARP |
| 2 | Data Link | **Frame** | MAC addressing, frame delivery | Ethernet, 802.1Q |
| 1 | Physical | **Bits** | Electrical / optical signals on wire | Copper, fiber, wireless |

### PDU Memory Chain

```
Application creates → DATA
Transport adds header → SEGMENT
Network adds IP header → PACKET
Data Link adds MAC header + trailer → FRAME
Physical puts on wire → BITS
```

### Device-to-Layer Mapping

| Device | Layer | What it reads | What it ignores |
|---|---|---|---|
| Hub | L1 | Electrical signal only | Everything above |
| Switch | L2 | Source + destination MAC | IP addresses |
| Router | L3 | Source + destination IP | MAC addresses (builds new L2 per hop) |
| Firewall | L3–L7 | IP, ports, application data | Nothing |
| Host / Server | All layers | Everything | Nothing |

> **Trap:** A router does NOT use MAC addresses for its forwarding decision. It strips the old L2 frame completely, reads the destination IP, looks up the routing table, and builds a brand-new L2 frame for the next hop.

### Why OSI Matters in Troubleshooting

```
Layer 1 broken → fix the cable first
Layer 2 broken → fix VLAN / MAC / trunk next
Layer 3 broken → fix IP / routing / subnet next
Layer 7 broken → fix application / DNS / config last
```

**Rule:** Always start from Layer 1 and work up. A broken Layer 1 makes every layer above irrelevant.

---

## 1.2 TCP/IP Model — 4 Layers

| TCP/IP Layer | Maps to OSI | Protocols |
|---|---|---|
| Application | Layers 5, 6, 7 | HTTP, HTTPS, DNS, DHCP, FTP, SMTP |
| Transport | Layer 4 | TCP, UDP |
| Internet | Layer 3 | IP, ICMP, ARP |
| Network Access | Layers 1, 2 | Ethernet, Wi-Fi |

### OSI vs TCP/IP — One-Line Comparison

| | OSI | TCP/IP |
|---|---|---|
| Purpose | Conceptual reference model | Practical real-world implementation |
| Layers | 7 | 4 |
| Used for | Troubleshooting, learning | Actual Internet communication |

> **Interview answer:**
> "OSI is a seven-layer conceptual model used for learning and troubleshooting. TCP/IP is the practical four-layer model used on real networks. OSI helps you think about the problem; TCP/IP is how the Internet actually works."

---

## 1.3 Encapsulation and Decapsulation

### Encapsulation — Sending Side (data moves DOWN)

```
Browser        → creates HTTP request  (DATA)
Transport      → adds TCP header       (SEGMENT)
Network        → adds IP header        (PACKET)
Data Link      → adds MAC hdr+trailer  (FRAME)
Physical       → converts to bits      (BITS on wire)
```

### Decapsulation — Receiving Side (data moves UP)

```
Physical       → receives BITS → reassembles FRAME
Data Link      → removes MAC header   → passes PACKET up
Network        → removes IP header    → passes SEGMENT up
Transport      → removes TCP header   → passes DATA up
Application    → receives original data
```

### Real Packet Flow — PC to Web Server

```
PC (192.168.1.10)  →  Switch  →  Router  →  Internet  →  Server (203.0.113.5)
```

| Step | Device | What happens |
|---|---|---|
| 1 | PC | Encapsulates: HTTP data → Segment → Packet → Frame → Bits |
| 2 | Switch | Reads L2 only. Checks CAM table. Forwards frame to router port. |
| 3 | Router | Strips old L2 frame. Reads IP. Builds NEW L2 frame for next hop. Forwards. |
| 4 | Each router hop | Same as step 3. IP never changes. MAC changes every hop. |
| 5 | Server | Decapsulates: Bits → Frame → Packet → Segment → Data |

> **Key rule:** IP address stays the same end-to-end. MAC address changes at every router hop.

### Where Devices Stop Decapsulating

- Switch → Stops at Layer 2 (reads MAC only)
- Router → Stops at Layer 3 (reads IP, strips L2, builds new L2)
- Host → Goes all the way to Layer 7

---

# SECTION 2 — IP Addressing

---

## 2.1 IPv4 Address Structure

- 32-bit address written in dotted decimal: `192.168.1.10`
- Each octet = 8 bits → range 0–255
- Every IP has two parts: **Network portion** + **Host portion**
- The subnet mask determines where the split is

### Example

```
IP:   192  .  168  .   1  .  10
Bits:  8        8       8      8  = 32 total bits
Mask: 255  .  255  .  255  .   0   (/24)
      Network portion        Hosts
```

---

## 2.2 Public vs Private IPv4 Ranges

| Range | Class | Usage |
|---|---|---|
| `10.0.0.0 – 10.255.255.255` | A | Private (largest block) |
| `172.16.0.0 – 172.31.255.255` | B | Private |
| `192.168.0.0 – 192.168.255.255` | C | Private (most common at home/office) |
| All other ranges | — | Public (Internet-routable) |

> **Why two companies can use the same private IP range:**
> Private IPs are not globally routed on the Internet. NAT translates them at the edge router, so 192.168.1.x can exist inside thousands of organizations without conflict.

> **Interview answer:**
> "Private IPs are used internally and are not routable on the Internet. NAT translates them to public IPs at the edge. Because private IPs never appear on the Internet directly, multiple organizations can use the same private ranges without conflict."

---

## 2.3 Subnet Masks, CIDR, and Quick Reference

### CIDR Notation

- `/24` means 24 bits are the network portion.
- `255.255.255.0` = `/24` (24 bits set to 1).
- Block size = `2^(32 - prefix)`.
- Usable hosts = `2^(host bits) - 2`.

### Quick Reference Table

| Prefix | Subnet Mask | Block Size | Usable Hosts |
|---|---|---|---|
| /24 | 255.255.255.0 | 256 | **254** |
| /25 | 255.255.255.128 | 128 | **126** |
| /26 | 255.255.255.192 | 64 | **62** |
| /27 | 255.255.255.224 | 32 | **30** |
| /28 | 255.255.255.240 | 16 | **14** |
| /29 | 255.255.255.248 | 8 | **6** |
| /30 | 255.255.255.252 | 4 | **2** |

### Finding Network and Broadcast Quickly

```
Given:  192.168.1.77/26

Block size = 64
Subnets: .0, .64, .128, .192

77 falls in the .64 block:
Network address:    192.168.1.64
Broadcast address:  192.168.1.127
Usable hosts:       192.168.1.65 – 192.168.1.126
```

**Formula:**
- Network = first address in the block (host bits all 0)
- Broadcast = last address in the block (host bits all 1)
- Usable = everything between them

---

## 2.4 VLSM — Variable Length Subnet Masking

**What it is:** Dividing a larger network into subnets of different sizes from the same address block. Allocate from largest requirement to smallest to avoid waste.

**Rule:** Always start with the largest subnet needed.

### Example: Allocate from 192.168.1.0/24

| Need | Size needed | Prefix | Subnet |
|---|---|---|---|
| 50 hosts | Next power of 2 above 52 = 64 | /26 | 192.168.1.0/26 |
| 25 hosts | Next power of 2 above 27 = 32 | /27 | 192.168.1.64/27 |
| 10 hosts | Next power of 2 above 12 = 16 | /28 | 192.168.1.96/28 |
| P2P link | 2 hosts needed | /30 | 192.168.1.112/30 |

> **Interview answer:**
> "VLSM allows us to use different subnet sizes within the same address block. We allocate from the largest requirement to the smallest to avoid wasting IPs. For example, a department needing 50 hosts gets a /26 while a point-to-point link only needs a /30."

---

## 2.5 IPv6 Basics

### Why IPv6?

- IPv4 is 32-bit → ~4.3 billion addresses → exhausted.
- IPv6 is 128-bit → 340 undecillion addresses → practically unlimited.
- IPv6 eliminates the need for NAT in pure IPv6 environments.
- IPv6 has **no broadcast** — uses multicast instead.

### IPv6 Address Format

```
2001:0db8:0000:0000:0000:0000:0000:0001

Compression rules:
1. Remove leading zeros in each group:    2001:db8:0:0:0:0:0:1
2. Replace one longest run of all-zeros with ::    2001:db8::1
```

> **Rule:** `::` can only be used once per address.

### IPv6 Address Types

| Type | Description | Example |
|---|---|---|
| **Unicast** | One-to-one delivery | `2001:db8::1` |
| **Multicast** | One-to-many (group) | `FF02::1` |
| **Anycast** | One-to-nearest (same address on multiple devices) | Looks like unicast |

### IPv6 Unicast Sub-types

| Sub-type | Prefix | Description |
|---|---|---|
| **Global Unicast** | `2000::/3` | Internet-routable (like public IPv4) |
| **Link-Local** | `FE80::/10` | Local segment only. Auto-assigned on every IPv6 interface. Never forwarded by routers. |
| **Unique Local** | `FC00::/7` | Like private IPv4 (RFC 4193). Not Internet-routable. |

### Special IPv6 Addresses

| Address | Meaning | IPv4 Equivalent |
|---|---|---|
| `::` | Unspecified (no address yet) | `0.0.0.0` |
| `::1` | Loopback (this device itself) | `127.0.0.1` |
| `FF02::1` | All nodes on local link | — |
| `FF02::2` | All routers on local link | — |

> **Key interview fact:** Every IPv6 interface automatically gets a link-local address (FE80::/10), even without any configuration.

### SLAAC — Stateless Address Autoconfiguration

- Hosts can self-configure a global unicast IPv6 address using Router Advertisement messages.
- No DHCP server needed.
- The host derives its interface ID from its MAC address (EUI-64) or randomly.

### Quick Pattern Recognition

```
::        → Unspecified (no IP yet)
::1       → Loopback (myself)
FE80::    → Link-local (local only)
FF02::    → Multicast
2001::    → Global unicast (Internet)
```

> **Interview answer:**
> "IPv6 uses 128-bit addresses compared to IPv4's 32-bit. It was introduced to solve IPv4 exhaustion. IPv6 has no broadcast; it uses multicast instead. Every interface automatically gets a link-local address. IPv6 also supports SLAAC, allowing hosts to configure their own addresses without a DHCP server."

---

# SECTION 3 — Core Protocols

---

## 3.1 ARP — Address Resolution Protocol

**Purpose:** Resolve an IPv4 address to a MAC address on the local network segment.

### Why ARP is Needed

Ethernet forwarding requires a destination MAC address. When a host knows the destination IP but not the MAC, it uses ARP to find it.

### ARP Process

```
Step 1: Host checks ARP cache first.
Step 2: If MAC unknown → sends ARP REQUEST as broadcast:
        "Who has 192.168.1.1? Tell 192.168.1.10"
        Dest MAC = FF:FF:FF:FF:FF:FF (broadcast)
Step 3: Every device on the LAN receives it.
Step 4: Only the device with that IP sends ARP REPLY:
        "192.168.1.1 is at AA:BB:CC:DD:EE:FF"
        Sent as unicast.
Step 5: Sender stores IP→MAC in ARP cache.
```

### Critical Rules

- ARP operates at **Layer 2** only.
- ARP does **NOT cross routers** — it stays within the local network segment.
- For remote traffic, a host ARPs for the **default gateway's MAC**, not the server's MAC.
- Stale or wrong ARP cache entries cause Layer 2 failures even when IP is correct.

### ARP and the Default Gateway

```
PC (192.168.1.10) wants to reach 8.8.8.8

Step 1: PC checks: Is 8.8.8.8 in my subnet 192.168.1.0/24? → NO
Step 2: PC sends to default gateway 192.168.1.1
Step 3: PC ARPs → "Who has 192.168.1.1?"
Step 4: Router replies with its MAC
Step 5: PC sends frame to router's MAC with destination IP = 8.8.8.8
Step 6: Router forwards it toward 8.8.8.8
```

### Subnet Mask + Default Gateway + ARP Together

| Component | Role |
|---|---|
| Subnet mask | Tells PC whether destination is local or remote |
| Default gateway | Next hop for all remote traffic |
| ARP | Finds the MAC of the gateway (or local destination) |
| NAT | Translates private→public at the edge (separate from ARP) |

> **Interview answer:**
> "ARP resolves an IPv4 address to a MAC address on the local LAN. The host broadcasts an ARP request; the device with the matching IP replies with its MAC. For remote traffic, the host does not ARP for the remote server — it ARPs for the default gateway's MAC instead. ARP does not cross routers."

> **Trap:** "If the ARP table is wrong, can IP still work?" → No. Incorrect ARP = wrong MAC = frame goes to the wrong device or gets dropped at Layer 2, even if the IP configuration is correct.

---

## 3.2 ICMP — Internet Control Message Protocol

**Purpose:** Diagnostic messages and error reporting at Layer 3. ICMP itself carries no application data.

### Key Uses

| Tool | ICMP Type | What it does |
|---|---|---|
| `ping` | Echo Request / Echo Reply | Tests reachability |
| `traceroute` | TTL exceeded messages | Maps path to destination |

### How Traceroute Works (TTL Mechanism)

```
Every router decrements TTL by 1.
When TTL reaches 0, the router drops the packet
and sends an ICMP "Time Exceeded" message back to the source.
Traceroute exploits this:
  Sends packet with TTL=1 → first router drops it, reveals its IP
  Sends packet with TTL=2 → second router drops it, reveals its IP
  ... continues until destination is reached
```

### Ping Failure Scenarios

| Symptom | Likely cause |
|---|---|
| Ping fails to same subnet IP | ARP failure, wrong VLAN, Layer 1 issue, host firewall |
| Ping by IP works, by hostname fails | DNS issue (not a network problem) |
| Ping works, HTTP fails | Port 80 blocked by ACL or firewall |
| Increasing TTL in ping | Routing loop |

> **Interview answer:**
> "ICMP is used for network diagnostics and error reporting. Ping uses ICMP Echo Request and Reply to test reachability. Traceroute uses TTL — each router decrements TTL by 1, and when TTL reaches zero, it sends an ICMP Time Exceeded message back, allowing traceroute to map each hop in the path."

---

## 3.3 TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed — uses ACKs and retransmission | No guarantee |
| Ordering | Ordered delivery | No ordering |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Use cases | HTTP/S, FTP, SSH, SMTP, Telnet | DNS, DHCP, VoIP, video streaming, SNMP |

### Memory Trick

```
TCP = Trusted, Careful, Predictable (reliable, ordered, slower)
UDP = Ultrafast, Doesn't-wait, Partial-delivery-ok (fast, unreliable, streaming)
```

> **Why is TCP slower?**
> TCP requires a 3-way handshake before data flows. Every segment must be acknowledged. If a segment is lost, it is retransmitted. This overhead adds latency but guarantees delivery.

---

## 3.4 TCP Three-Way Handshake

### Steps

```
Client ──── SYN ────────────────→ Server
            "I want to connect"

Client ←─── SYN-ACK ─────────── Server
            "I hear you, I'm ready"

Client ──── ACK ────────────────→ Server
            "Connection established"
```

### What Sequence Numbers Do

- Each byte of data has a sequence number.
- The receiver sends ACK = (last sequence received + 1) to confirm.
- If a segment is lost, the receiver's ACK does not advance → sender retransmits.

### What If SYN-ACK Never Returns?

- Client retransmits SYN several times.
- After the timeout period, the connection fails.
- Cause: firewall blocking the SYN, routing issue, or server unavailable.

### Where to See This in Wireshark

Filter: `tcp.flags.syn == 1`

You will see:
- Packet 1: `SYN` (from client)
- Packet 2: `SYN, ACK` (from server)
- Packet 3: `ACK` (from client)

> **Interview answer:**
> "The TCP three-way handshake starts with the client sending SYN, the server responding with SYN-ACK, and the client completing it with ACK. After that, the TCP connection is established and data can flow. If SYN-ACK is never received, the client retransmits before eventually timing out — usually caused by a firewall or unreachable server."

---

## 3.5 DNS — Domain Name System

**Purpose:** Translate human-readable hostnames (like `www.cisco.com`) to IP addresses.

### DNS Resolution Flow (Full)

```
Step 1: Client types www.cisco.com in browser
Step 2: OS checks local DNS cache
Step 3: Not cached → asks configured DNS resolver (e.g. 8.8.8.8)
Step 4: Resolver checks its cache
Step 5: Not cached → resolver asks Root DNS server
Step 6: Root replies: "Ask the .com TLD server"
Step 7: Resolver asks .com TLD server
Step 8: TLD replies: "Ask cisco.com's authoritative DNS"
Step 9: Resolver asks cisco.com's authoritative DNS
Step 10: Gets back: "www.cisco.com = 72.163.4.185"
Step 11: Resolver caches the record and returns IP to client
Step 12: Browser connects to 72.163.4.185
```

### DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| **A** | Hostname → IPv4 address | `cisco.com → 72.163.4.185` |
| **AAAA** | Hostname → IPv6 address | `cisco.com → 2001:db8::1` |
| **CNAME** | Alias to another name | `www → cisco.com` |
| **MX** | Mail server for a domain | `mail.cisco.com` |
| **PTR** | Reverse lookup: IP → hostname | `72.163.4.185 → cisco.com` |
| **NS** | Name server for a domain | — |

### TTL in DNS

- **TTL** = Time To Live = how long a cached DNS record is valid.
- Low TTL → frequent lookups → always fresh, more DNS traffic.
- High TTL → records stay cached longer → stale risk if IP changes.

### DNS Failure Symptoms

```
Ping by IP works     → Network is fine
Ping by hostname fails → DNS is broken

Website won't load   → Likely DNS (not routing)
"Server not found"   → DNS failure
```

### DNS Troubleshooting Commands

```bash
nslookup cisco.com          # Test resolution (Windows / Linux)
dig cisco.com               # Detailed resolution trace (Linux/Mac)
ipconfig /flushdns          # Clear DNS cache (Windows)
resolvectl flush-caches     # Clear DNS cache (Linux systemd)
```

> **Interview answer:**
> "DNS translates hostnames to IP addresses. A client checks its local cache, then queries a resolver, which works through root, TLD, and authoritative servers to get the answer. If ping works by IP but not by hostname, DNS is the problem, not the network path."

---

## 3.6 DHCP — Dynamic Host Configuration Protocol

**Purpose:** Automatically assign IP address, subnet mask, default gateway, and DNS server to hosts.

### DHCP DORA Process

| Step | Direction | What happens |
|---|---|---|
| **D**iscover | Client → **Broadcast** | Client looks for any DHCP server. No IP yet. Source IP = 0.0.0.0 |
| **O**ffer | Server → Client | Server offers available IP, subnet mask, gateway, DNS, lease time |
| **R**equest | Client → **Broadcast** | Client requests the offered IP (still broadcast to notify others) |
| **A**cknowledge | Server → Client | Server confirms lease. Client now has a valid IP. |

### What DHCP Provides

- IP address
- Subnet mask
- Default gateway
- DNS server address
- Lease duration
- (Optional) Domain name, NTP server

### DHCP Lease Renewal

- At **50%** of lease time: client tries to renew with the original server.
- At **87.5%**: if renewal failed, client broadcasts to any available DHCP server.
- At **100%**: if still unrenewed, client loses IP and restarts the DORA process.

---

## 3.7 DHCP Relay — `ip helper-address`

### The Problem

DHCP Discover is a **Layer 2 broadcast** (destination `255.255.255.255`).
Routers do **not forward broadcasts** by default.
So if the DHCP server is on a different subnet, clients cannot reach it.

### The Solution — ip helper-address

```
Router interface configured with: ip helper-address 10.0.0.5

Flow:
Client (VLAN 10)
  → DHCP Discover broadcast
  → Router receives it on VLAN 10 interface
  → Router converts broadcast to UNICAST
  → Forwards to DHCP server at 10.0.0.5
  → DHCP server sees GIADDR = router's interface IP
  → DHCP server selects correct scope for that subnet
  → Offer sent back through router to client
```

### GIADDR Field

- GIADDR = **Gateway IP Address** field in the DHCP packet.
- The relay agent (router) inserts **its own interface IP** into GIADDR.
- The DHCP server uses GIADDR to know which subnet the client is in.
- This lets one DHCP server serve multiple VLANs with the right IP scopes.

### Configuration — Where to Apply It

```cisco
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 10.0.0.5
```

- Applied on the **client-facing** interface (the SVI or subinterface facing the clients).
- The IP is the **DHCP server's IP**, not the client's IP.
- Configured by the **network administrator** — not automatic.

> **Does ip helper-address forward only DHCP?**
> No. By default it forwards several UDP services: DHCP (67/68), DNS (53), TFTP (69), NTP (37), and others. DHCP is the most common use case.

### Full Interview Answer

> "DHCP Relay allows DHCP clients in one VLAN to get IPs from a DHCP server in another VLAN. Since DHCP Discover is a broadcast and routers don't forward broadcasts, we configure `ip helper-address` on the client-facing gateway interface. The router relays the request as a unicast to the DHCP server and inserts its interface IP into the GIADDR field so the server knows which subnet the client belongs to and can assign an IP from the correct pool."

> **Trap — Who configures ip helper-address?**
> The network administrator manually configures it on the router or Layer 3 switch, on the interface that faces the client VLAN. It is not automatic.

---

# SECTION 4 — Common Port Numbers

---

## Must-Know Ports

| Port | Protocol | Service | Memory hint |
|---|---|---|---|
| 20, 21 | TCP | FTP (data / control) | File Transfer Protocol |
| **22** | TCP | SSH | Secure Shell |
| 23 | TCP | Telnet | Unencrypted shell |
| 25 | TCP | SMTP | Email sending |
| **53** | TCP/UDP | DNS | Name resolution |
| **67/68** | UDP | DHCP | 67 = server, 68 = client |
| **80** | TCP | HTTP | Web (unencrypted) |
| 110 | TCP | POP3 | Email retrieval |
| 143 | TCP | IMAP | Email (stays on server) |
| **443** | TCP | HTTPS | Web (encrypted/TLS) |
| 161/162 | UDP | SNMP | Network monitoring |
| 514 | UDP | Syslog | Log collection |

> **Must-memorize 6:** 22 (SSH), 23 (Telnet), 53 (DNS), 67/68 (DHCP), 80 (HTTP), 443 (HTTPS).
> These appear in almost every ACL and troubleshooting question.

### Why Ports Matter in Troubleshooting

```
Ping works (ICMP, no port)
But HTTP fails (TCP port 80)
→ ACL or firewall is blocking port 80, not the IP

Telnet fails but SSH works
→ Port 23 blocked (good security practice)

DNS resolution fails
→ Check if UDP port 53 is reachable to the DNS server
```

---

# SECTION 5 — Collision Domain vs Broadcast Domain

---

## The Core Concept

| Domain | Question it answers | Separated by |
|---|---|---|
| **Collision domain** | Who can interfere with my transmission? | Switch ports (each port = 1 collision domain) |
| **Broadcast domain** | Who hears my broadcast? | Routers + VLANs |

---

## Collision Domain

A collision domain is a segment where two simultaneous transmissions cause a collision.

### Hub (Old Technology — 1 Collision Domain)

```
       Hub
    /   |   \
   A    B    C

If A and C transmit at the same time → COLLISION
All devices share one collision domain.
```

### Switch (Modern — 1 Collision Domain Per Port)

```
       Switch
    /   |   \
   A    B    C

A sends → Switch port 1 buffer
C sends → Switch port 3 buffer
No collision. Each port = isolated collision domain.

3 ports = 3 collision domains
```

> **Why collisions rarely matter today:** Modern Ethernet uses full-duplex — a device can send and receive simultaneously. No CSMA/CD needed. But interviewers still test this concept.

---

## Broadcast Domain

A broadcast domain is the area where a broadcast frame (destination MAC `FF:FF:FF:FF:FF:FF`) reaches all devices.

### Switch — One Broadcast Domain (default)

```
       Switch
    /   |   \
   A    B    C

A sends broadcast (ARP: "Who has 192.168.1.1?")
Switch floods it → B receives it ✓, C receives it ✓

All devices in the same broadcast domain.
```

### Router — Separates Broadcast Domains

```
PC1 — Switch — Router — Switch — PC2

PC1 sends broadcast → Router receives it → Router STOPS it
PC2 never hears it.

Each router interface = separate broadcast domain.
```

### VLANs — Create Logical Broadcast Domains on One Switch

```
24-port switch without VLANs:
→ 1 broadcast domain (all 24 ports)

Add VLANs:
VLAN 10: Ports 1–12
VLAN 20: Ports 13–24

Broadcast from Port 1 → only reaches Ports 2–12 (VLAN 10)
Does NOT reach Ports 13–24 (VLAN 20)

→ 2 broadcast domains, still 1 physical switch
```

---

## Device Comparison Summary

| Device | Collision Domains | Broadcast Domains |
|---|---|---|
| Hub (4 ports) | 1 | 1 |
| Switch (4 ports, no VLANs) | 4 | 1 |
| Switch (4 ports, 2 VLANs) | 4 | 2 |
| Router (2 interfaces) | 2 | 2 |

> **Interview answer:**
> "A collision domain is a segment where simultaneous transmissions can collide. Each switch port is a separate collision domain. A broadcast domain is where broadcasts are received. Switches create one broadcast domain by default; routers and VLANs separate them. A hub is one collision domain and one broadcast domain for all ports."

---

# SECTION 6 — VLANs and Subnets

---

## What is a VLAN?

A VLAN (Virtual Local Area Network) is a logical way to divide a physical switch into multiple separate broadcast domains. Devices in the same VLAN behave as if they are on the same physical switch, even if they are not. Devices in different VLANs cannot communicate directly.

```
Without VLANs:           With VLANs:
    Switch                    Switch
A B C D E F          VLAN10     |    VLAN20
one network          A  B  C    |    D  E  F
one broadcast        one domain | separate domain
```

### Why VLANs Were Created

| Problem | VLAN Solution |
|---|---|
| All departments share one network | Separate HR, Finance, IT logically |
| Broadcasts reach everyone | Broadcasts stay within VLAN |
| Security isolation needed | VLANs isolate traffic |
| Performance issues | Smaller broadcast domains = less wasted bandwidth |

---

## VLAN vs Subnet

| | VLAN | Subnet |
|---|---|---|
| OSI Layer | Layer 2 | Layer 3 |
| What it separates | Broadcast domains | IP networks |
| Feature of | Switch | IP addressing |
| Uses | VLAN IDs (1–4094) | Network addresses (CIDR) |

### How They Relate

In enterprise design: **1 VLAN = 1 Subnet** (standard practice)

```
VLAN 10  ↔  192.168.10.0/24  (HR)
VLAN 20  ↔  192.168.20.0/24  (Finance)
VLAN 30  ↔  192.168.30.0/24  (IT)
```

### Why VLANs Must Have Different Subnets

```
If VLAN 10 and VLAN 20 both use 192.168.1.0/24:
→ Host thinks destination is in its subnet
→ Tries to ARP for it (Layer 2 broadcast)
→ ARP cannot cross VLANs
→ Communication breaks

Therefore: Different VLANs = Different Subnets
```

### Can Devices in Different VLANs Talk?

Not directly. They need a **Layer 3 device** (router or Layer 3 switch with SVI).

```
PC1 (VLAN 10, 192.168.10.10)
  → Sends packet to router (inter-VLAN routing)
  → Router routes to VLAN 20 subnet
  → PC2 (VLAN 20, 192.168.20.20) receives it
```

> **Interview answer:**
> "A VLAN is a Layer 2 technology that divides a switch into separate broadcast domains. A subnet is a Layer 3 IP network. In enterprise networks, each VLAN is mapped to a unique subnet. Devices in different VLANs cannot communicate directly — they need inter-VLAN routing through a router or Layer 3 switch."

> **Memory trick:**
> VLAN = Broadcast Domain (L2)
> Subnet = IP Network (L3)
> 1 VLAN ↔ 1 Subnet (standard design)

---

# SECTION 7 — Access Ports, Trunk Ports, and 802.1Q

---

## Access Port

- Belongs to **one VLAN only**.
- Used for end devices: PCs, printers, IP phones, cameras.
- Frames enter and leave **without VLAN tags**.
- The device connected to an access port does NOT need to know about VLANs.

```
PC ───── Switch Port Fa0/1 (Access, VLAN 10)
Traffic automatically treated as VLAN 10
```

---

## Trunk Port

- Carries **multiple VLANs** over a single link.
- Used between: switches, switch-to-router, switch-to-access point.
- Uses **802.1Q tagging** to identify which VLAN each frame belongs to.

```
Switch A ══════ Trunk ══════ Switch B
         VLAN10, VLAN20, VLAN30 all travel on one cable
```

---

## 802.1Q Tagging

When a frame enters a trunk port, the switch inserts a 4-byte **802.1Q tag** into the Ethernet header:

```
Original frame:  [ Dst MAC | Src MAC | Type | Payload ]
Tagged frame:    [ Dst MAC | Src MAC | 802.1Q Tag (VLAN ID) | Type | Payload ]
```

- The receiving switch reads the tag to know which VLAN the frame belongs to.
- The tag is **removed before delivery** to an access port (end device never sees the tag).
- 802.1Q VLAN IDs: 1–4094.

---

## Native VLAN

- The native VLAN is the **one VLAN on a trunk that is NOT tagged**.
- All other VLANs are tagged. The native VLAN traffic crosses untagged.
- Default native VLAN is VLAN 1.

### Native VLAN Mismatch — Critical Issue

```
Switch A: Native VLAN = 99
Switch B: Native VLAN = 1

An untagged frame arrives at Switch B.
Switch B thinks: "This is VLAN 1 traffic"
But it was VLAN 99 traffic.
→ Frame placed into wrong VLAN
→ Security risk + connectivity problems
```

**Rule: Both ends of a trunk MUST have the same native VLAN.**

### Best Practice

```
Change native VLAN away from VLAN 1:
Switch(config-if)# switchport trunk native vlan 999

VLAN 1 is also used by Cisco control protocols.
Using an unused VLAN (e.g. 999) for native reduces risk.
```

---

## Access Port vs Trunk Port Comparison

| Feature | Access Port | Trunk Port |
|---|---|---|
| VLANs carried | One | Many |
| Tagging | No tag | 802.1Q tags added |
| Used for | PCs, printers, phones | Switch↔Switch, Switch↔Router |
| End device aware of VLANs? | No | N/A |

> **Memory trick:**
> Access = single lane road (one VLAN)
> Trunk = multi-lane highway (many VLANs)
> 802.1Q tag = the label on each vehicle showing its neighborhood
> Native VLAN = the one neighborhood whose vehicles travel without labels

> **Interview answer:**
> "An access port carries traffic for one VLAN only and is used for end devices. A trunk port carries multiple VLANs using 802.1Q tagging and is used between network devices. The native VLAN is the untagged VLAN on a trunk — both ends must match to avoid a native VLAN mismatch."

---

# SECTION 8 — NAT and PAT

---

## Why NAT Exists

- Internal devices use private IPs (10.x, 172.16.x, 192.168.x).
- Private IPs are NOT routable on the Internet.
- NAT at the edge router translates private IPs to a public IP so traffic can reach the Internet.

```
Without NAT:
PC sends packet with Source IP = 192.168.1.10
Internet rejects it (private IP not routable)

With NAT:
PC sends packet with Source IP = 192.168.1.10
Router translates it to Source IP = 49.36.100.50 (public IP)
Internet accepts it ✓
```

---

## How the Reply Returns

```
Web server sees: packet from 49.36.100.50
Web server replies to: 49.36.100.50

Router receives reply → looks up NAT translation table
NAT table entry:
  49.36.100.50:50001  →  192.168.1.10:54321

Router rewrites destination IP back to 192.168.1.10
Forwards to the correct internal PC
```

---

## NAT Types

| Type | Description | Mapping | Use case |
|---|---|---|---|
| **Static NAT** | Fixed 1:1 mapping | 1 private IP → 1 public IP | Internal servers needing permanent public IP |
| **Dynamic NAT** | Pool-based mapping | Many private → pool of public IPs | Older enterprise edge routers |
| **PAT (NAT Overload)** | Many share one public IP using ports | Many private → 1 public IP + ports | Home Wi-Fi, most enterprise networks |

---

## PAT — Port Address Translation (Most Common)

Also called **NAT Overload**.

```
PC1  192.168.1.10:54321  →  49.36.100.50:50001
PC2  192.168.1.20:54321  →  49.36.100.50:50002
PC3  192.168.1.30:55000  →  49.36.100.50:50003

All share ONE public IP.
Router differentiates them by source port number in NAT table.
```

**NAT Table Example:**

| Inside IP:Port | Public IP:Port |
|---|---|
| 192.168.1.10:54321 | 49.36.100.50:50001 |
| 192.168.1.20:54321 | 49.36.100.50:50002 |
| 192.168.1.30:55000 | 49.36.100.50:50003 |

---

## NAT Terminology

| Term | Meaning |
|---|---|
| **Inside Local** | Private IP of the internal host |
| **Inside Global** | Public IP as seen by the Internet (translated) |
| **Outside Local** | Destination IP as seen from inside |
| **Outside Global** | Real destination IP on the Internet |

---

## Is NAT a Security Mechanism?

> Partially. NAT hides internal IPs from outside observers (obscurity). But it is NOT a proper firewall. Real security requires stateful packet inspection and access control lists. Do not rely on NAT alone for security.

---

## Full Packet Flow With NAT

```
PC (192.168.1.10)
→ Frame to default gateway (ARPs for router MAC)
→ Router receives: Src=192.168.1.10, Dst=203.0.113.5
→ NAT translation: Src becomes 49.36.100.50
→ Packet travels Internet to 203.0.113.5
→ Server replies to 49.36.100.50
→ Router checks NAT table
→ Translates back: Dst becomes 192.168.1.10
→ PC receives the reply
```

> **IP address rule:** In basic routing, IP addresses stay the same end-to-end. NAT is the major exception — the source IP is changed at the edge router.

> **Interview answer:**
> "NAT is a Layer 3 function on a router that translates private IP addresses to a public IP so internal devices can access the Internet. PAT (NAT overload) is the most common form — it lets many devices share one public IP by using different source port numbers. The router keeps a NAT translation table to reverse the translation for return traffic."

---

# SECTION 9 — Module 1 Rapid-Fire Interview Q&A

---

> Practice: Read each question, cover the answer, and answer in one sentence. Then uncover and check.

---

**Q: What is the OSI model?**
A: A seven-layer reference model that explains how data moves across a network, used for learning and troubleshooting.

**Q: What are the seven layers in order?**
A: Physical, Data Link, Network, Transport, Session, Presentation, Application.

**Q: What is the TCP/IP model?**
A: A practical four-layer model (Network Access, Internet, Transport, Application) used in real-world networks.

**Q: What is encapsulation?**
A: Each layer adds its own header as data moves down the stack, turning data into segment, packet, frame, then bits.

**Q: What PDU does the Transport layer produce?**
A: Segment.

**Q: What layer does a switch operate at?**
A: Layer 2 (Data Link). It forwards frames using MAC addresses.

**Q: What layer does a router operate at?**
A: Layer 3 (Network). It forwards packets using IP addresses and builds a new Layer 2 frame per hop.

**Q: What does IP address stay the same mean in a packet flow?**
A: Source and destination IPs remain unchanged end-to-end; only MAC addresses change at every router hop.

**Q: What are the three private IPv4 ranges?**
A: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16.

**Q: How many usable hosts does a /26 have?**
A: 62 (2^6 - 2 = 62).

**Q: What is VLSM?**
A: Using different subnet sizes from the same address block; always allocate largest first to minimize waste.

**Q: What is IPv6 and why is it needed?**
A: IPv6 uses 128-bit addresses to solve IPv4 exhaustion. It eliminates broadcast and supports SLAAC.

**Q: What is a link-local IPv6 address?**
A: Starts with FE80::/10. Auto-assigned to every IPv6 interface. Local segment only; never forwarded by routers.

**Q: What is `::1` in IPv6?**
A: The loopback address (equivalent to 127.0.0.1 in IPv4).

**Q: What is ARP?**
A: ARP resolves an IPv4 address to a MAC address on the local LAN.

**Q: Does ARP cross routers?**
A: No. ARP is a Layer 2 broadcast confined to the local network segment.

**Q: When a host wants to reach a remote server, what does it ARP for?**
A: The default gateway's MAC address, not the server's MAC.

**Q: What is ICMP used for?**
A: Network diagnostics and error reporting. Ping uses ICMP Echo Request/Reply. Traceroute uses TTL decrement.

**Q: What is the difference between TCP and UDP?**
A: TCP is connection-oriented and reliable; UDP is connectionless and faster but has no delivery guarantee.

**Q: What are the three steps of the TCP handshake?**
A: SYN, SYN-ACK, ACK.

**Q: What is DNS?**
A: DNS translates hostnames to IP addresses through a resolution chain: client → resolver → root → TLD → authoritative.

**Q: If ping by IP works but by hostname fails, what is the problem?**
A: DNS failure. IP connectivity is fine; name resolution is not.

**Q: What is DHCP DORA?**
A: Discover, Offer, Request, Acknowledge — the four-step process for a client to get an IP from a DHCP server.

**Q: What does `ip helper-address` do?**
A: Relays DHCP broadcasts from clients as unicast to a DHCP server in another subnet, inserting the GIADDR field so the server knows the client's subnet.

**Q: What is GIADDR?**
A: The Gateway IP Address field in DHCP — the relay agent inserts its own interface IP so the DHCP server knows which subnet to assign an IP from.

**Q: What port does SSH use?**
A: Port 22 (TCP).

**Q: What port does HTTPS use?**
A: Port 443 (TCP).

**Q: What ports does DHCP use?**
A: UDP 67 (server) and 68 (client).

**Q: What is a collision domain?**
A: A network segment where simultaneous transmissions can cause collisions. Each switch port is a separate collision domain.

**Q: What is a broadcast domain?**
A: A network segment where broadcast frames are received by all devices. Routers and VLANs separate broadcast domains.

**Q: How many broadcast domains does a switch create by default?**
A: One (for all ports in the same VLAN).

**Q: How many collision domains does a 4-port switch have?**
A: Four (one per port).

**Q: What is a VLAN?**
A: A logical broadcast domain created on a switch that segments traffic without requiring separate physical switches.

**Q: What is the relationship between a VLAN and a subnet?**
A: In standard enterprise design, each VLAN maps to one unique subnet. VLAN is Layer 2; subnet is Layer 3.

**Q: What is an access port?**
A: A switch port that carries traffic for one VLAN only, used for end devices. Frames are untagged.

**Q: What is a trunk port?**
A: A switch port that carries multiple VLANs using 802.1Q tagging. Used between network devices.

**Q: What is the native VLAN?**
A: The one untagged VLAN on a trunk. Both ends of the trunk must have the same native VLAN.

**Q: What is NAT?**
A: Translates private IP addresses to a public IP at the edge router, allowing internal devices to access the Internet.

**Q: What is PAT?**
A: NAT overload — many internal devices share one public IP by using different source port numbers.

**Q: What is the major exception to "IP addresses never change end-to-end"?**
A: NAT changes the source IP at the edge router when translating private to public.

---

# SECTION 10 — Module 1 Trap Questions

---

> These are follow-up questions interviewers ask after your first answer. Practice answering them without hesitation.

---

| Trap Question | Strong Answer |
|---|---|
| Why do we still use OSI if TCP/IP is what's real? | OSI is a diagnostic framework. It helps isolate which layer is failing without getting confused by implementation details. |
| What is added at Layer 2? At Layer 3? | L2: MAC header + FCS trailer. L3: IP header (source IP, destination IP, TTL, protocol). |
| Does a router care about MAC addresses? | Not for forwarding. It strips the old L2 frame, reads the IP, and builds a brand-new L2 frame for the next hop. |
| Is /24 always Class C? | No. In classful networking it was, but CIDR is classless — any prefix can apply to any block. |
| Is ARP used across routers? | Never. ARP is Layer 2 and stays on the local segment. Each router segment has its own ARP. |
| What if the ARP table has a wrong entry? | Frames go to the wrong device or get dropped. Connectivity fails at L2 even if IP is correct. Fix: clear ARP cache. |
| Why is TCP slower than UDP? | TCP requires a handshake, acknowledgments, retransmissions, and ordered delivery — all add latency. |
| What if SYN-ACK never comes back? | Client retransmits SYN several times, then times out. Usually a firewall blocking the SYN or server unreachable. |
| What if DNS is down but routing is fine? | Ping by IP works. Ping by hostname fails. Websites won't load. "Server not found" errors appear. |
| Who configures ip helper-address? | The network administrator. On the router/L3 switch interface facing the client VLAN. Points to the DHCP server IP. |
| Does ip helper-address only forward DHCP? | No. It also forwards DNS (53), TFTP (69), NTP (37), and others by default. DHCP is the most common reason. |
| Why can ping succeed but HTTP fail? | Ping uses ICMP (no port). HTTP uses TCP port 80. A firewall or ACL blocking port 80 won't affect ICMP. |
| Can two companies use the same private IP range? | Yes. Private IPs are not globally routed. NAT translates them at each edge, so there is no conflict. |
| Should PortFast be used on switch-to-switch links? | Never. It bypasses STP delay. If another switch connects to a PortFast port, a loop forms before STP reacts. |
| What is the native VLAN mismatch problem? | Untagged frames from VLAN 99 on Switch A arrive at Switch B where native VLAN = 1, so they land in VLAN 1. Wrong VLAN placement, connectivity issues, potential security risk. |
| Is NAT a security mechanism? | Partially. It hides internal IPs. But it is NOT a firewall. Real security requires stateful inspection and ACLs. |

---

# SECTION 11 — Module 1 Completion Checklist

---

Tick each item once you can explain it in interview style without notes.

### 2.1 Models
- [ ] OSI 7 layers — name, PDU, protocols for each
- [ ] TCP/IP 4 layers — name and OSI mapping
- [ ] Encapsulation — full chain from data to bits
- [ ] Decapsulation — receiving side strip sequence
- [ ] Device-to-layer mapping — hub, switch, router, firewall, host
- [ ] Where devices stop processing — switch at L2, router at L3
- [ ] Real packet flow — PC to web server, what each device does

### 2.2 IP Addressing
- [ ] IPv4 structure — 32-bit, octets, network + host portion
- [ ] Public vs private ranges — all 3 ranges memorized
- [ ] Subnet masks and CIDR — explain prefix, block size, network, broadcast, usable hosts
- [ ] VLSM — allocate largest to smallest from one block
- [ ] IPv6 basics — 128-bit, no broadcast, compression rules
- [ ] IPv6 address types — unicast, multicast, anycast
- [ ] IPv6 special addresses — ::1, ::, FE80::, FF02::1

### 2.3 Core Protocols
- [ ] ARP — purpose, process, does not cross routers, ARP for gateway not server
- [ ] ICMP — ping, traceroute, TTL mechanism
- [ ] TCP vs UDP — comparison table from memory
- [ ] TCP three-way handshake — SYN, SYN-ACK, ACK and what follows
- [ ] DNS — resolution flow, record types, TTL, failure symptoms
- [ ] DHCP — DORA process, what it provides, lease renewal
- [ ] ip helper-address — why needed, GIADDR, where configured, who configures it

### Ports and Concepts
- [ ] Common ports — 22, 23, 53, 67/68, 80, 443 at minimum
- [ ] Collision domain vs broadcast domain — definition + device comparison table
- [ ] Hub vs switch — collision and broadcast domain count
- [ ] VLANs vs subnets — L2 vs L3, 1 VLAN = 1 subnet rule
- [ ] Access port vs trunk port — tagging, use cases
- [ ] Native VLAN — what it is, mismatch risk, best practice
- [ ] NAT and PAT — purpose, types, PAT detail, NAT table concept

---

## Wireshark Labs to Complete (Module 1)

- [ ] Capture ARP Request and Reply — identify source/dest IP and MAC in each
- [ ] Capture ICMP Echo Request and Reply — observe TTL
- [ ] Capture TCP three-way handshake — filter and read SYN / SYN-ACK / ACK
- [ ] Capture DHCP DORA — see broadcast in Discover and Request

---

## One-Page Revision Summary

```
OSI:   7 layers  |  PDU: Data→Segment→Packet→Frame→Bits
TCP/IP: 4 layers  |  Network Access / Internet / Transport / Application
Devices: Hub=L1, Switch=L2 (MAC), Router=L3 (IP, rebuilds L2 per hop)
IP:    IPv4=32bit | IPv6=128bit, no broadcast, FE80=link-local, ::1=loopback
Subnetting: /26=62hosts, /27=30hosts, /28=14hosts, /30=2hosts
Protocols:
  ARP   = IP→MAC on local LAN, broadcast, doesn't cross routers
  ICMP  = ping+traceroute, TTL decrement reveals hops
  TCP   = reliable, connection, handshake (SYN/SYN-ACK/ACK)
  UDP   = fast, connectionless, VoIP/DNS/DHCP
  DNS   = name→IP, A/AAAA/CNAME/MX records, TTL=cache time
  DHCP  = DORA, gives IP+mask+GW+DNS, ip helper-address crosses routers
Ports: SSH=22, Telnet=23, DNS=53, DHCP=67/68, HTTP=80, HTTPS=443
Domains: Collision=per switch port | Broadcast=per router interface / VLAN
VLANs:  L2 broadcast domain | Subnet=L3 IP network | 1 VLAN = 1 Subnet
Trunks: 802.1Q tags frames | Native VLAN = untagged | Match on both ends
NAT:    Private→Public at edge | PAT=many share 1 IP via ports | NAT table
```

---

*End of Module 1 Notes | Cisco G4 Interview Preparation*
*Version: compiled from all Module 1 study sessions including OSI, TCP/IP, IPv4, IPv6, ARP, ICMP, TCP, DNS, DHCP, Ports, Collision/Broadcast Domains, VLANs, Trunking, and NAT/PAT*
