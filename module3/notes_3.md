# Module 3 — IP Connectivity: Routing
## Cisco G4 Interview Preparation · Print & Revision Notes

> **How to use:** Read once to learn. Cover answers and test yourself aloud. Every section ends with a ready interview answer. Tick each topic when you can answer it without looking.

---

# SECTION 1 — Anatomy of a Routing Table

---

## The Core Idea

Before a router moves a single bit of data, it needs a map. That map is the **IP Routing Table**, stored in RAM. Running `show ip route` shows this map as a dense wall of text. Every single entry breaks down into **6 critical components**.

### Sample Entry to Dissect

```
O   192.168.50.0/24   [110/20]   via 10.1.1.2,   00:04:25,   GigabitEthernet0/1
```

---

## The 6 Components

### 1. Protocol Code — "How did the router learn this?"

| Code | Meaning |
|---|---|
| **C** | Connected — directly attached to a router interface |
| **L** | Local — the exact IP assigned to the router's own interface (always /32 in IPv4) |
| **S** | Static — manually typed by an administrator |
| **O** | OSPF — learned dynamically via OSPF |
| **D** | EIGRP — learned dynamically via EIGRP |
| **B** | BGP — learned dynamically via BGP |

### 2. Destination Prefix & Mask — "Where is it going?"

```
192.168.50.0/24
```
The target network and its subnet mask — the size of the "room" the router is trying to reach.

### 3. Administrative Distance (AD) — "How trustworthy is this source?"

```
[110/...]
```
The **first number** inside the brackets.

> **Mental rule:** AD is a believability rating from 0–255. **Lower = more trusted.**

**AD Table (memorize this cold):**

| Route Source | AD |
|---|---|
| Connected Interface | **0** (most trusted) |
| Static Route | **1** |
| EIGRP (Internal) | **90** |
| OSPF | **110** |
| RIP | **120** |
| Unreachable / Poisoned | **255** (completely untrusted — drop) |

### 4. Metric — "How good is this specific path?"

```
[.../20]
```
The **second number** inside the brackets.

> **Mental rule:** Metric is only compared when routes come from the **exact same protocol**. You never compare an OSPF metric to an EIGRP metric — different protocols calculate metric completely differently.

### 5. Next-Hop IP Address — "What's the next stop?"

```
via 10.1.1.2
```
The IP address of the adjacent router's inbound interface — the next runner in the relay race.

### 6. Outgoing Interface — "Which door does the packet leave through?"

```
GigabitEthernet0/1
```
The physical/logical port on **this** router used to eject the packet toward the next hop.

---

## Gateway of Last Resort — Deep Dive

### Why It Exists

There are over **1 million public networks** on the global Internet. A standard office or home router does not have the RAM or CPU to store all of them. It only stores **local routes** (its own subnets, internal servers). For everything else, it relies on one single backup plan.

```
Gateway of last resort is 10.1.1.1 to network 0.0.0.0
```

### Step-by-Step: Packet to 8.8.8.8 Arrives

```
Step 1: Comprehensive Scan
        Router sweeps the entire routing table for a match to 8.8.8.8.
        Checks for 8.0.0.0, 8.8.0.0, 8.8.8.8 — anything specific.

Step 2: Blank Search Result
        No specific route exists. Router has no idea where 8.8.8.8 lives.

Step 3: Decision Point
```

| Scenario | What Happens |
|---|---|
| **No Gateway of Last Resort set** | Router drops the packet immediately. Sends ICMP "Destination Host Unreachable" back to sender. |
| **Gateway of Last Resort set** | Router uses the default route `0.0.0.0/0`. Forwards the packet toward the upstream provider router (e.g., `10.1.1.1`), trusting that device has a bigger map. |

`0.0.0.0/0` is a wildcard meaning: **"Any network in the universe that I don't explicitly know."**

> **Interview answer:**
> "The Gateway of Last Resort is the router's default exit path, mapped to the 0.0.0.0/0 default route. It exists because edge routers cannot store the entire global Internet routing table. Instead of dropping packets bound for unknown destinations, the router forwards them upstream to a provider device with a broader view of the network. Without it, external connectivity fails unless every destination is manually mapped."

---

> **Interview answer (full routing table entry):**
> "A routing table entry has six parts: the protocol code showing how the route was learned, the destination network and mask, the administrative distance showing how trustworthy the source is, the metric showing how good the path is within that protocol, the next-hop IP, and the outgoing interface. AD is checked between different protocols; metric is only checked within the same protocol."

---

# SECTION 2 — Longest Prefix Match Rule

---

## The Golden Rule of Routing

When a packet arrives, the router extracts the destination IP and scans the routing table. If **multiple entries match**, the router does NOT default to the lowest AD first.

> **The Golden Rule:** The router always selects the route with the **longest prefix match** (the most specific subnet mask / highest CIDR value) — **first**. AD is only used as a tiebreaker if prefix length is equal.

```
ORDER OF OPERATIONS:
1. Longest Prefix Match  ← checked FIRST, always wins
2. Administrative Distance ← only used if prefix lengths TIE
3. Metric ← only used if AD also ties (same protocol)
```

---

## Worked Example

**Routing table contains:**

| # | Route | Source | AD |
|---|---|---|---|
| 1 | `172.16.0.0/16` | Static | 1 |
| 2 | `172.16.10.0/24` | OSPF | 110 |
| 3 | `172.16.10.0/26` | EIGRP | 90 |
| 4 | `0.0.0.0/0` | Default | 1 |

**Packet arrives for: `172.16.10.22`**

```
Check Route 4 (0.0.0.0/0):     Matches everything.        → /0  bits matched
Check Route 1 (172.16.0.0/16): First 16 bits match.        → /16 bits matched
Check Route 2 (172.16.10.0/24): First 24 bits match.       → /24 bits matched
Check Route 3 (172.16.10.0/26): /26 = blocks of 64.
                                 Range: .0 to .63
                                 Does .22 fall inside?  YES → /26 bits matched
```

**Result:** Router picks Route 3 — `/26` — even though the Static Route (Route 1) has a far better AD of 1 vs EIGRP's AD of 90.

> **The router completely ignores AD at this stage.** Longest prefix match wins regardless of trust level.

---

## /24 Subnet Math — Why .5 Belongs Inside It

A `/24` freezes the **first 24 bits** as the Network ID. The remaining 8 bits = Host ID.

```
2^8 = 256 total addresses in a /24
```

**Boundaries of `10.0.0.0/24`:**

| Address | Meaning |
|---|---|
| `10.0.0.0` | Network Address — cannot assign to a device |
| `10.0.0.1` | First usable host (often the gateway) |
| `10.0.0.2` – `10.0.0.4` | Usable hosts |
| **`10.0.0.5`** | **5th usable host — falls inside this /24** |
| ... | ... continues sequentially ... |
| `10.0.0.254` | Last usable host |
| `10.0.0.255` | Broadcast Address — cannot assign to a device |

### Why This Matters for Longest Prefix Match

```
Route A: 10.0.0.0/16  → Range: 10.0.0.0 – 10.0.255.255  (65,536 addresses — huge area)
Route B: 10.0.0.0/24  → Range: 10.0.0.0 – 10.0.0.255    (256 addresses — small street)

Packet for 10.0.0.5 matches BOTH.
Route B is more specific → Router picks Route B.
```

---

## Packet-Level Anatomy: What Happens at the Wire

When the router selects the winning route, it performs these physical actions before pushing the packet onto the cable:

```
Step 1: Decapsulation
        Strip off the incoming Layer 2 Ethernet header.

Step 2: Inspection & TTL Decrement
        Inspect destination IP.
        Subtract 1 from TTL (prevents infinite loops).
        Recalculate IP header checksum.

Step 3: Encapsulation
        Look at the Next-Hop IP of the winning route.
        ARP for the MAC address of that next-hop (if not cached).
        Build a brand-new Layer 2 Ethernet header.
        Fire the packet out the outgoing physical interface.
```

> **Interview golden nugget:**
> "Layer 3 IP addresses stay identical from source to destination end-to-end, but Layer 2 MAC addresses are completely rewritten at every single router hop along the path."

---

## Interview Trap — The Classic Question

**Q:** "A router has an OSPF route for `10.0.0.0/24` and a Static route for `10.0.0.0/16`. A packet arrives for `10.0.0.5`. Which path wins?"

> **Interview answer:**
> "The router forwards via the OSPF route. Even though static routes have a better AD of 1 compared to OSPF's 110, the router's forwarding logic checks Longest Prefix Match before Administrative Distance. The packet matches /24 on 24 bits versus /16 on only 16 bits. Because prefix length is evaluated first, the more specific /24 OSPF route wins."

---

# SECTION 3 — Configuring Static and Default Routes (IPv4 & IPv6)

---

## Command Architecture

```cisco
! IPv4
Router(config)# ip route [destination_network] [subnet_mask] [next_hop_address OR exit_interface]

! IPv6
Router(config)# ipv6 route [destination_prefix/mask] [next_hop_address OR exit_interface]
```

---

## The 3 Flavors of Static Routes

### 1. Standard Network Route

Points traffic toward a specific remote subnet.

```cisco
! IPv4 — reach 192.168.20.0/24 via neighbor 10.1.1.2
Router(config)# ip route 192.168.20.0 255.255.255.0 10.1.1.2

! IPv6 — same concept
Router(config)# ipv6 route 2001:db8:acad:20::/64 2001:db8:acad:10::2
```

### 2. Default Route — "Gateway of Last Resort" Configuration

Matches **every** packet with no specific entry in the table. All-zero mask = "any matching bits."

```cisco
! IPv4
Router(config)# ip route 0.0.0.0 0.0.0.0 10.1.1.2

! IPv6 (all-zero compressed)
Router(config)# ipv6 route ::/0 2001:db8:acad:10::2
```

### 3. Host Route

Points to a **single specific IP** instead of a network block.

> **Trivia:** Host route mask = `/32` in IPv4 (`255.255.255.255`), `/128` in IPv6. It locks down every bit — the ultimate Longest Prefix Match.

**Use case:** Send traffic for one critical server (e.g., a core database) over a specialized link while the rest of the subnet takes a different path.

```cisco
! IPv4
Router(config)# ip route 192.168.20.5 255.255.255.255 10.1.1.2

! IPv6
Router(config)# ipv6 route 2001:db8:acad:20::5/128 2001:db8:acad:10::2
```

---

## Interview Trap — Next-Hop IP vs. Exit Interface

### Scenario A: Next-Hop IP Only

```cisco
Router(config)# ip route 192.168.20.0 255.255.255.0 10.1.1.2
```

- Router says: "To reach 192.168.20.0, go to 10.1.1.2."
- Router must do a **second search** to find which port reaches 10.1.1.2.
- **Term: Recursive Lookup** — doubles the CPU work per packet.

### Scenario B: Exit Interface Only

```cisco
Router(config)# ip route 192.168.20.0 255.255.255.0 Gig0/0
```

- Router immediately knows which door to use. No recursive lookup.
- **The Danger on Ethernet (multi-access):** Router assumes it's directly connected to every destination on that route. It will ARP for **every single destination IP** traversing the route. This can exhaust the CAM table and spike CPU — a process called **Proxy ARP**.

### The Expert Solution: Fully Specified Static Route

```cisco
Router(config)# ip route 192.168.20.0 255.255.255.0 Gig0/0 10.1.1.2
```

Specifies both the exit interface **and** the next-hop Layer 2 destination — avoids recursive lookup **and** the Proxy ARP trap.

| Method | Recursive Lookup? | Proxy ARP Risk? | Best Practice |
|---|---|---|---|
| Next-hop IP only | Yes | No | OK on point-to-point, costs CPU |
| Exit interface only | No | Yes (on Ethernet) | Risky on multi-access links |
| **Fully specified (both)** | **No** | **No** | **Recommended on Ethernet** |

---

## Verification Commands

```cisco
show ip route static          ! Filter to manually configured routes only
show ipv6 route static
```

> **Interview answer:**
> "A static route can be configured with a next-hop IP, an exit interface, or both. Next-hop IP alone forces a recursive lookup to find the exit interface. Exit interface alone on an Ethernet segment risks Proxy ARP, where the router ARPs for every destination on that route, spiking CPU. The best practice on multi-access networks is a fully specified static route — both the exit interface and the next-hop IP — to avoid both problems."

---

# SECTION 4 — Floating Static Routes

---

## What It Is

A **Floating Static Route** is a backup static route configured with a **higher AD** than the primary route. Because it's less trusted, it stays invisible in the configuration ("floats") until the primary route fails.

```
Normal state:  Primary route (low AD) is active. Floating route (high AD) is hidden.
On failure:    Primary route disappears. Floating route (higher AD) takes over.
On recovery:   Primary route reappears. Floating route goes back to hidden.
```

---

## Operational Scenario

```
Office Router
  ├── Primary: Fiber link  → GigabitEthernet0/0
  └── Backup:  4G/5G link  → GigabitEthernet0/1

Goal: All Internet traffic (0.0.0.0/0) uses Fiber normally.
      If Fiber is cut, instantly shift to 4G — no human intervention.
```

### Configuration

```cisco
! Primary route — default AD = 1 (not specified)
Router(config)# ip route 0.0.0.0 0.0.0.0 10.1.1.2

! Backup route — custom AD = 5, appended at the end, makes it "float"
Router(config)# ip route 0.0.0.0 0.0.0.0 192.168.1.2 5
```

---

## Behind the Scenes — How the Router Thinks

```
Step 1: Normal Operations
        Route 1 (Fiber): AD = 1
        Route 2 (4G):    AD = 5
        Router installs Route 1 (lower AD). Route 2 stays hidden in running-config.

Step 2: Link Failure
        Fiber is cut. GigabitEthernet0/0 → down/down.

Step 3: Failover
        Router detects dead interface. Purges Route 1 from the table.
        Scans config, finds backup route (AD = 5). Installs it.
        Traffic now flows through 4G.

Step 4: Restoration
        Fiber comes back up.
        Router sees AD = 1 route is available again.
        1 < 5 → removes 4G route, reinstalls Fiber route.
```

---

## Interview Trap — Floating Static vs. OSPF Backup

**Q:** "You want a floating static route to back up a route learned via OSPF. What AD should you assign, and why?"

> **Interview answer:**
> "To float behind an OSPF route, the static route's AD must be set higher than OSPF's default of 110 — for example, 120 or 130. A standard static route defaults to AD 1 and would override OSPF completely. By raising the AD above 110, the static route stays out of the table and only activates if OSPF loses its adjacency and removes the dynamic path."

---

## Verification

```cisco
show ip route                              ! Under normal conditions, only primary route shown
show running-config | include ip route     ! Shows BOTH commands sitting in config
```

---

# SECTION 5 — OSPFv2 Fundamentals and Neighbor Adjacency

---

## What is OSPF?

OSPFv2 is a **Link-State** Interior Gateway Protocol (IGP) used to route IPv4 inside an enterprise network.

- Routers share the status of their own links (up/down, speed, connected networks).
- This info collects into a shared database: the **Link-State Database (LSDB)**.
- Every router runs **Dijkstra's Shortest Path First (SPF)** algorithm on that database independently to compute its own loop-free best path to every destination.

### Why OSPF Over Static Routes at Scale

Manually configuring static routes across thousands of routers is an administrative nightmare. If a link fails, a human must intervene. OSPF lets routers talk to each other and adapt automatically.

---

## The 3 Phases of OSPF Operation

```
[ Discover Neighbors ]  →  [ Synchronize Databases (LSDB) ]  →  [ Run SPF Calculation ]
```

| Phase | What Happens |
|---|---|
| **A. Discover Neighbors** | Routers send multicast **Hello packets** out OSPF-enabled interfaces to find other OSPF routers. |
| **B. Database Synchronization** | Routers swap Database Description (DBD) summaries until every router in the area has an **identical LSDB**. |
| **C. Route Calculation** | Each router runs SPF against the shared map, finds the shortest path to each network by bandwidth cost, installs winners into the routing table. |

---

## OSPF Neighbor Adjacency — The 7 States

Knowing exactly which state an adjacency is stuck on tells you exactly what's broken.

| # | State | What's Happening |
|---|---|---|
| 1 | **DOWN** | No OSPF packets received on the link yet. |
| 2 | **INIT** | Router A receives a Hello from Router B but doesn't see its own Router ID listed inside B's Hello yet. One-way only. |
| 3 | **2-WAY** | Router A sees its own name inside Router B's Hello. Two-way confirmed. **DR/BDR election happens here** on multi-access broadcast networks. |
| 4 | **EXSTART** | Routers decide Master vs Slave to manage sync order. Highest Router ID wins the tie. |
| 5 | **EXCHANGE** | Routers swap DBD packets — a "table of contents" of known networks. |
| 6 | **LOADING** | If Router A notices B has newer/more specific data, it sends a Link-State Request (LSR). B replies with a Link-State Update (LSU). |
| 7 | **FULL** | Databases 100% synchronized. The ideal, stable operational state. |

```
DOWN → INIT → 2-WAY → EXSTART → EXCHANGE → LOADING → FULL
                ↑
        DR/BDR election occurs HERE
```

---

## Interview Trap — The 5 OSPF Hello Requirements

**Q:** "Two routers are directly connected, but stuck in INIT or 2-WAY. What must match for a FULL adjacency?"

> **Interview answer:**
> "Five parameters inside the Hello packets must match identically:
> 1. **Area ID** — both routers must be in the same OSPF area (e.g., Area 0).
> 2. **Subnet & subnet mask** — interfaces must be on the exact same IP subnet.
> 3. **Hello and Dead timers** — default 10 seconds Hello / 40 seconds Dead on Ethernet, must match on both ends.
> 4. **Authentication** — if configured, strings/passwords must match.
> 5. **Stub area flags** — both routers must agree on the area type."

---

## Verification

```cisco
show ip ospf neighbor    ! Primary tool. Shows neighbor, interface, and state.
                          ! Healthy state: FULL/DR, FULL/BDR, or FULL/2WAY
```

> **Interview answer:**
> "OSPF is a link-state protocol where routers exchange Hello packets to discover neighbors, synchronize their link-state databases, and independently run SPF to compute the shortest path to every network. Adjacency progresses through seven states from Down to Full, and a stuck adjacency in Init or 2-Way usually points to a mismatch in area ID, subnet, timers, authentication, or stub flags."

---

# SECTION 6 — OSPF DR/BDR Elections (Broadcast Networks)

---

## The Problem: n(n-1)/2 Adjacency Chaos

On a **Broadcast Multi-Access Network** (e.g., 5 routers on the same switch), by default every router wants a FULL relationship with every other router.

```
Formula: n(n-1)/2

5 routers  → 5(4)/2  = 10 adjacencies
20 routers → 20(19)/2 = 190 adjacencies
```

If one link fails, a storm of Link-State Updates floods the switch, spiking CPU and choking real traffic.

---

## The Solution: Elect a DR and BDR

| Role | Function |
|---|---|
| **DR (Designated Router)** | The "boss." Every other router forms a FULL adjacency only with the DR. |
| **BDR (Backup Designated Router)** | The "deputy." Listens silently, ready to instantly take over if DR fails. |
| **DROTHER** | Every other router. Stays at **FULL/2-WAY** with other DROTHERs — knows who they are but doesn't sync maps directly with them. |

### Traffic Flow Architecture

```
DROTHER detects a change
   → sends LSU to DR + BDR via multicast 224.0.0.6  (All DR Routers)
DR receives it
   → redistributes to all DROTHERs via multicast 224.0.0.5  (All OSPF Routers)

Result: Adjacencies scale LINEARLY instead of exponentially.
```

---

## Election Tiebreakers (In Order)

### 1. OSPF Interface Priority (Primary Key)

- Range: 0–255. Default: **1**.
- **Highest priority wins DR.** Second-highest becomes BDR.
- **Priority = 0** → permanently disqualified from DR/BDR. Stays a DROTHER forever.

### 2. Router ID (Tiebreaker if priorities match)

Highest Router ID wins. Determined in this order:

```
1. Manually configured: router-id x.x.x.x   (always wins if set)
2. Highest IPv4 address on any Loopback interface
3. Highest IPv4 address on any physical interface
```

---

## Interview Trap — Non-Preemptive Behavior

**Q:** "Router A (priority 255) is DR. It reboots. Router B (priority 1) becomes DR. When A comes back, does it reclaim DR?"

> **Interview answer:**
> "No. OSPF DR/BDR election is strictly non-preemptive. Once elected, a DR keeps the role until it physically fails, its OSPF process restarts, or its interface drops. This prevents constant recalculation storms from a flapping high-priority router. Router A will return as a DROTHER or BDR, not reclaim DR automatically."

---

## Verification

```cisco
show ip ospf interface [name]
! Shows: State: DR / State: BDR / State: DROTHER
! Also shows the current DR's ID and priority value
```

> **Interview answer:**
> "On broadcast multi-access networks, OSPF elects a DR and BDR to avoid the n(n-1)/2 full-mesh adjacency problem. DROTHERs only form FULL adjacencies with the DR and BDR, using multicast 224.0.0.6 to send updates and 224.0.0.5 for the DR to redistribute them. Election is based first on interface priority, then Router ID, and the process is non-preemptive — a returning higher-priority router does not reclaim the DR role."

---

# SECTION 7 — HSRP (First Hop Redundancy Protocol)

---

## The Problem: Static Default Gateway

End devices have one configured Default Gateway. If that router fails, the PC has no idea — it keeps sending traffic to a dead IP and loses connectivity.

```
[ Primary Router R1 ] ─── WAN/Internet
       (10.1.1.2)
           │
[ PC1 ] ── [ L2 Switch ]
Gateway: 10.1.1.2
           │
[ Backup Router R2 ] ─── WAN/Internet
       (10.1.1.3)
```

Even with a perfect backup router sitting on the same switch, PC1 cannot dynamically switch its gateway. A human would have to manually reconfigure every machine, or wait for DHCP leases to expire — that's downtime.

---

## The Solution: HSRP

**HSRP (Hot Standby Router Protocol)** — Cisco-proprietary. Multiple physical routers present themselves as **one Virtual Router** sharing a **Virtual IP (VIP)** and **Virtual MAC**.

```
R1 Physical IP:        10.1.1.2
R2 Physical IP:        10.1.1.3
HSRP Virtual IP (VIP): 10.1.1.1   ← THIS is what you configure on PCs!
```

### HSRP Roles

| Role | Behavior |
|---|---|
| **Active Router** | Owns the VIP and Virtual MAC. Physically processes and forwards all gateway traffic. |
| **Standby Router** | Silent. Listens to periodic Hello packets (multicast `224.0.0.2`, every 3 seconds by default). |

**Virtual MAC format:** `0000.0c07.acXX` — where `XX` is the hex representation of the HSRP group number.

---

## Failover Execution — Step by Step

```
Step 1: Normal Operations
        PC1 ARPs for the gateway MAC (10.1.1.1).
        Active Router (R1) replies with the Virtual MAC.
        R1 processes and forwards the frame.

Step 2: The Failure
        R1 loses power completely.

Step 3: The Detection
        Standby Router (R2) stops receiving HSRP Hellos.
        Waits for the Hold Timer to expire (default 10 seconds).

Step 4: The Takeover
        Hold timer expires. R2 declares R1 dead.
        R2 transitions Standby → Active.
        R2 assumes the Virtual IP AND the exact same Virtual MAC.

Step 5: The Result
        PC1 keeps sending to 10.1.1.1 / same virtual MAC — no reconfiguration needed.
        The switch updates its CAM table to point that MAC toward R2's port.
        Zero disruption to the client. Failover completes within seconds.
```

---

## Interview Trap — HSRP Preemption

**Q:** "R1 (priority 110) is Active. R2 (priority 100) is Standby. R1 reboots; R2 becomes Active. When R1 returns, does it reclaim Active?"

> **Interview answer:**
> "By default, no — HSRP preemption is disabled by default. Even though R1 has the higher priority, it stays in Standby to avoid unnecessary flapping, unless the command `standby [group] preempt` is explicitly configured. Enabling preemption forces R1 to immediately reclaim Active the moment its priority becomes the highest on the segment."

---

## Verification

```cisco
show standby          ! Detailed: state, priorities, timers, VIP, virtual MAC
show standby brief    ! Clean tabular view: active/standby router, VIP
```

---

## Quick Comparison — HSRP vs VRRP vs GLBP

| Protocol | Type | Active Routers | Notes |
|---|---|---|---|
| **HSRP** | Cisco proprietary | One Active, one Standby | Most common in Cisco-only shops |
| **VRRP** | Open standard (RFC 3768) | One Master, one Backup | Vendor-neutral, similar concept to HSRP |
| **GLBP** | Cisco proprietary | Active-Active load sharing | Multiple routers forward simultaneously |

> **Interview answer:**
> "HSRP lets multiple routers present a single virtual default gateway to end hosts. One router is Active and forwards all traffic; the Standby listens for Hellos and takes over within the hold timer if the Active fails — using the same Virtual IP and Virtual MAC so end devices need zero reconfiguration. Preemption is disabled by default, meaning a returning higher-priority router will not automatically reclaim the Active role."

---

# SECTION 8 — Module 3 Rapid-Fire Q&A

---

> Practice: Read question, cover answer, respond in one sentence. Uncover and check.

---

**Q: What are the 6 components of a routing table entry?**
A: Protocol code, destination prefix/mask, administrative distance, metric, next-hop IP, outgoing interface.

**Q: What does the protocol code "L" mean?**
A: Local — the exact IP address assigned to the router's own interface, always shown with a /32 mask.

**Q: What is Administrative Distance?**
A: A trust rating from 0–255 for a route source; lower is more trusted. Used to compare routes from different protocols.

**Q: What is the AD of a connected route, static route, and OSPF?**
A: Connected = 0, Static = 1, OSPF = 110.

**Q: When is Metric compared instead of AD?**
A: Only when comparing multiple routes learned from the exact same protocol.

**Q: What is the Gateway of Last Resort?**
A: The default route (0.0.0.0/0) used when no specific match exists in the routing table; forwards unmapped traffic upstream.

**Q: What happens if no Gateway of Last Resort is configured and no route matches?**
A: The router drops the packet and sends an ICMP "Destination Host Unreachable" message back to the sender.

**Q: What is the Longest Prefix Match rule?**
A: When multiple routes match a destination, the router always selects the most specific (highest prefix length) route, regardless of AD.

**Q: Is AD or Longest Prefix Match checked first?**
A: Longest Prefix Match is always checked first. AD is only used to break a tie when prefix lengths are equal.

**Q: How many usable hosts in a /24?**
A: 254 (2^8 = 256 total, minus network and broadcast addresses).

**Q: What happens to the IP address and MAC address as a packet crosses multiple router hops?**
A: The IP address (source and destination) stays the same end-to-end. The MAC address is rewritten at every hop.

**Q: What is a recursive lookup?**
A: When a static route specifies only a next-hop IP, the router must do a second table lookup to find which interface reaches that next-hop — doubling CPU work.

**Q: What is the Proxy ARP danger with static routes?**
A: If a static route specifies only an exit interface on a multi-access network, the router ARPs for every destination IP on that route, risking CAM table exhaustion and high CPU.

**Q: What is a fully specified static route?**
A: A static route that includes both the exit interface AND the next-hop IP, avoiding both recursive lookup and Proxy ARP.

**Q: What is the subnet mask of a host route?**
A: /32 in IPv4, /128 in IPv6.

**Q: What is a Floating Static Route?**
A: A backup static route configured with a higher AD than the primary route; it stays hidden until the primary route fails.

**Q: What AD should a floating static route have to back up OSPF?**
A: Higher than 110 — e.g., 120 or 130.

**Q: What type of routing protocol is OSPF?**
A: Link-state Interior Gateway Protocol (IGP).

**Q: What algorithm does OSPF use to calculate best paths?**
A: Dijkstra's Shortest Path First (SPF) algorithm.

**Q: What are the 3 phases of OSPF operation?**
A: Discover neighbors (Hello), synchronize databases (LSDB), run SPF calculation.

**Q: How many states does an OSPF adjacency pass through?**
A: Seven — Down, Init, 2-Way, ExStart, Exchange, Loading, Full.

**Q: At which OSPF state does DR/BDR election happen?**
A: At the end of the 2-Way state, on multi-access broadcast networks.

**Q: What is the ideal, stable OSPF adjacency state?**
A: FULL — databases are completely synchronized.

**Q: What are the 5 OSPF Hello requirements that must match for adjacency?**
A: Area ID, subnet/mask, Hello and Dead timers, authentication, stub area flags.

**Q: Why does OSPF elect a DR and BDR?**
A: To avoid the n(n-1)/2 full-mesh adjacency problem on broadcast multi-access networks, which would otherwise flood the network with redundant Link-State Updates.

**Q: What multicast address do DROTHERs use to send updates to the DR/BDR?**
A: 224.0.0.6 (All DR Routers).

**Q: What multicast address does the DR use to redistribute updates?**
A: 224.0.0.5 (All OSPF Routers).

**Q: What relationship state do DROTHERs maintain with each other?**
A: FULL/2-WAY — they know about each other but don't sync databases directly.

**Q: What is the default OSPF interface priority?**
A: 1. Range is 0–255.

**Q: What happens if OSPF interface priority is set to 0?**
A: That router can never become DR or BDR; it remains a permanent DROTHER.

**Q: How is a Router ID determined if not manually configured?**
A: Highest IPv4 address on any loopback interface; if none, highest IPv4 address on any physical interface.

**Q: Is OSPF DR/BDR election preemptive?**
A: No. Once a DR is elected, it keeps the role until it fails — even if a higher-priority router returns later.

**Q: What problem does HSRP solve?**
A: It eliminates the single point of failure of a static default gateway by letting multiple routers share one virtual gateway IP and MAC.

**Q: What are the two main HSRP roles?**
A: Active (owns the VIP/virtual MAC, forwards traffic) and Standby (listens, ready to take over).

**Q: What is the HSRP virtual MAC format?**
A: 0000.0c07.acXX, where XX is the HSRP group number in hex.

**Q: What multicast address does HSRP use for Hello packets, and how often?**
A: 224.0.0.2, every 3 seconds by default.

**Q: What is the default HSRP hold timer?**
A: 10 seconds.

**Q: Is HSRP preemption enabled by default?**
A: No. It must be explicitly configured with `standby [group] preempt`.

**Q: What is the difference between HSRP, VRRP, and GLBP?**
A: HSRP and GLBP are Cisco-proprietary; VRRP is an open standard. HSRP and VRRP use one active router at a time; GLBP allows active-active load sharing across multiple routers.

---

# SECTION 9 — Module 3 Trap Questions

---

| Trap Question | Strong Answer |
|---|---|
| If a route has a great AD but a worse prefix match, which wins? | Prefix match always wins. AD is irrelevant unless prefix lengths tie. |
| Why is comparing OSPF metric to EIGRP metric meaningless? | Metric is protocol-specific — OSPF uses bandwidth-based cost, EIGRP uses a composite formula. They are not directly comparable; AD is used to choose between different protocols instead. |
| What's wrong with using only an exit interface on an Ethernet static route? | Proxy ARP — the router will ARP for every destination IP on that route, risking CAM table exhaustion and CPU spikes. |
| Why use a fully specified static route instead of just a next-hop IP? | A next-hop-only route forces a recursive lookup (extra CPU per packet). Specifying both interface and next-hop avoids that entirely. |
| Why does a floating static route need an AD higher than the protocol it's backing up? | If its AD were lower or equal, it would override the dynamic protocol permanently, defeating the purpose of a backup. |
| Two routers are directly connected but stuck in 2-WAY — what's likely wrong? | Probably a DR/BDR election still pending, or one of the 5 Hello parameters (area, subnet, timers, auth, stub flag) doesn't match — though 2-WAY itself means Hellos are working; full sync hasn't started yet (often expected on DROTHER-DROTHER pairs). |
| Why don't DROTHERs sync directly with each other? | To prevent the n(n-1)/2 adjacency explosion. They only form FULL adjacencies with the DR and BDR; with each other they stay at FULL/2-WAY. |
| If you set OSPF priority to 0 on a router, what happens during an election? | That router is permanently excluded from DR/BDR roles, even if it would otherwise win on Router ID. |
| Why is OSPF DR election non-preemptive? | To prevent constant recalculation storms if a high-priority router has an unstable, flapping interface. |
| Why does HSRP failover not require any reconfiguration on end devices? | The Standby router takes over the exact same Virtual IP and Virtual MAC the Active router used, so ARP entries and gateway configuration on clients remain valid. |
| If preemption isn't configured, what happens when the original Active router comes back online? | It returns as Standby, not Active — even if its priority is higher. Preemption must be explicitly enabled to reclaim the Active role automatically. |
| What's the practical difference between HSRP and GLBP for traffic forwarding? | HSRP only allows one router (Active) to forward traffic at a time. GLBP allows multiple routers to forward simultaneously, load-sharing across them. |

---

# SECTION 10 — Module 3 Completion Checklist

---

Tick each item once you can explain it in interview style without notes.

### Routing Table Fundamentals
- [ ] 6 components of a routing table entry — protocol code, prefix/mask, AD, metric, next-hop, exit interface
- [ ] Protocol codes — C, L, S, O, D, B
- [ ] Administrative Distance table — memorized cold (0, 1, 90, 110, 120, 255)
- [ ] Difference between AD and Metric — when each is used
- [ ] Gateway of Last Resort — what it is, why it exists, what happens without it

### Longest Prefix Match
- [ ] The golden rule — prefix match checked before AD
- [ ] Worked multi-route example — can solve a new one independently
- [ ] /24 subnet math — network, broadcast, usable host range
- [ ] Packet-level anatomy — decapsulation, TTL decrement, re-encapsulation
- [ ] IP stays same end-to-end; MAC changes every hop

### Static and Default Routes
- [ ] IPv4 and IPv6 static route command syntax
- [ ] 3 flavors — standard, default (0.0.0.0/0 or ::/0), host (/32 or /128)
- [ ] Recursive lookup — what it is, why it costs CPU
- [ ] Proxy ARP danger — exit-interface-only on Ethernet
- [ ] Fully specified static route — why it's best practice

### Floating Static Routes
- [ ] What makes a route "float" — higher AD than primary
- [ ] Full failover and restoration sequence
- [ ] Correct AD to use when backing up OSPF (>110)
- [ ] Verification commands — show ip route, show running-config | include ip route

### OSPF Fundamentals
- [ ] OSPF type — link-state IGP, uses SPF/Dijkstra
- [ ] 3 phases of operation — discover, synchronize, calculate
- [ ] All 7 neighbor states in order, with what happens at each
- [ ] Where DR/BDR election occurs (end of 2-Way)
- [ ] 5 Hello requirements for adjacency

### OSPF DR/BDR Elections
- [ ] The n(n-1)/2 problem and why it matters
- [ ] DR / BDR / DROTHER roles and relationships
- [ ] Multicast addresses — 224.0.0.5 and 224.0.0.6
- [ ] Election tiebreakers — priority first, then Router ID
- [ ] Priority 0 behavior
- [ ] Non-preemptive election behavior

### HSRP
- [ ] The problem HSRP solves — static gateway single point of failure
- [ ] Virtual IP and Virtual MAC concept
- [ ] Active vs Standby roles
- [ ] Virtual MAC format — 0000.0c07.acXX
- [ ] Hello multicast (224.0.0.2, 3 sec) and Hold timer (10 sec default)
- [ ] Full failover sequence, step by step
- [ ] Preemption — disabled by default, command to enable
- [ ] HSRP vs VRRP vs GLBP comparison

---

## Module 3 Labs to Complete

- [ ] Build a 3-router topology with static routes; verify with `show ip route static`
- [ ] Configure a fully specified static route and confirm no recursive lookup using `show ip cef` or trace behavior
- [ ] Configure a floating static route as backup to a primary route; simulate link failure and observe failover
- [ ] Configure single-area OSPF between 3 routers; verify neighbor states reach FULL
- [ ] Deliberately mismatch OSPF Hello/Dead timers between two routers and observe the adjacency get stuck
- [ ] Build a 3-router broadcast segment and observe DR/BDR election using `show ip ospf interface`
- [ ] Set OSPF priority to 0 on one router and confirm it never becomes DR/BDR
- [ ] Configure HSRP between two routers sharing a LAN segment; verify with `show standby brief`
- [ ] Simulate Active router failure and time the failover to Standby
- [ ] Enable HSRP preemption and verify the original router reclaims Active on return

### Useful Show Commands for Module 3

```cisco
show ip route                          # Full routing table
show ip route static                   # Only static routes
show ip route ospf                     # Only OSPF-learned routes
show ip ospf neighbor                  # OSPF adjacency states
show ip ospf interface [name]          # DR/BDR/DROTHER state, priority
show ip protocols                      # Summary of running routing protocols
show standby                           # Detailed HSRP state
show standby brief                     # Quick HSRP table view
show running-config | include ip route # All static route commands in config
```

---

## One-Page Revision Summary

```
ROUTING TABLE ENTRY: Protocol Code | Prefix/Mask | [AD/Metric] | via Next-Hop | Exit Interface
AD TABLE: Connected=0 | Static=1 | EIGRP=90 | OSPF=110 | RIP=120 | Unreachable=255
RULE: AD compares DIFFERENT protocols. Metric compares SAME protocol only.
GATEWAY OF LAST RESORT: 0.0.0.0/0 default route. No match + no default = ICMP unreachable, packet dropped.

LONGEST PREFIX MATCH: Checked BEFORE AD, always. Most specific prefix wins regardless of trust.
/24 = 256 addrs (2^8). Network=.0, Broadcast=.255, Usable=.1-.254 (254 hosts).
PACKET FLOW: Strip L2 → decrement TTL → recompute checksum → ARP next-hop → new L2 → forward.
RULE: IP same end-to-end. MAC rewritten every hop.

STATIC ROUTES: ip route / ipv6 route [dest] [mask] [next-hop or exit-int]
3 TYPES: Standard (specific subnet) | Default (0.0.0.0/0 or ::/0) | Host (/32 or /128)
Next-hop only = Recursive Lookup (extra CPU). Exit-int only on Ethernet = Proxy ARP risk.
BEST PRACTICE: Fully specified route = exit interface + next-hop IP together.

FLOATING STATIC: Backup route, HIGHER AD than primary. Hidden until primary fails.
  To back up OSPF (AD 110): use AD > 110, e.g. 120-130.

OSPF: Link-state IGP. Dijkstra SPF algorithm. Builds shared LSDB.
3 PHASES: Discover (Hello) → Synchronize (DBD/LSDB) → Calculate (SPF)
7 STATES: Down → Init → 2-Way → ExStart → Exchange → Loading → Full
  DR/BDR election happens at end of 2-Way.
5 HELLO MATCH REQUIREMENTS: Area ID | Subnet/Mask | Hello/Dead Timers | Auth | Stub flag

DR/BDR: Solves n(n-1)/2 adjacency explosion on broadcast networks.
  DR=boss (FULL with everyone) | BDR=deputy | DROTHER=others (FULL/2-WAY with each other)
  224.0.0.6 = DROTHER→DR/BDR | 224.0.0.5 = DR→everyone
  Election: Priority (default 1, range 0-255, 0=never DR/BDR) → tiebreak Router ID
  NON-PREEMPTIVE: returning high-priority router does NOT reclaim DR.

HSRP: Cisco FHRP. Virtual IP + Virtual MAC (0000.0c07.acXX) shared by multiple routers.
  ACTIVE = forwards traffic | STANDBY = listens, ready to take over
  Hello: 224.0.0.2 every 3 sec | Hold timer: 10 sec default
  FAILOVER: Standby waits hold timer → takes VIP+vMAC → switch CAM updates → zero client impact
  PREEMPTION: disabled by default. Command: standby [group] preempt
  HSRP/GLBP = Cisco proprietary | VRRP = open standard | GLBP = active-active load sharing
```

---

*End of Module 3 Notes | Cisco G4 Interview Preparation*
*Covers: Routing Table Anatomy, Gateway of Last Resort, Longest Prefix Match, Static/Default/Host Routes, Floating Static Routes, OSPFv2 Fundamentals & Neighbor States, OSPF DR/BDR Elections, HSRP First-Hop Redundancy*
