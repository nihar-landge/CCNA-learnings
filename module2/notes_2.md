# Module 2 — Network Access: Switching and VLANs
## Cisco G4 Interview Preparation · Print & Revision Notes

> **How to use:** Read once to learn. Cover answers and test yourself aloud. Every section ends with a ready interview answer. Tick each topic when you can answer it without looking.

---

# SECTION 1 — Network Devices at a Glance

---

## Device-to-Layer Reference

| Device | Layer | Core Function | What it reads |
|---|---|---|---|
| **Hub** | L1 | Repeats signal to all ports | Electrical signal only |
| **Switch** | L2 | Forwards frames using MAC addresses | Source + Destination MAC |
| **Router** | L3 | Routes packets between IP networks | Source + Destination IP |
| **Layer 3 Switch** | L2 + L3 | Does both switching and routing internally | MAC + IP |
| **Firewall** | L3–L7 | Stateful inspection and policy enforcement | IP, ports, application data |

### Key Rules (these come up constantly)

- A switch does NOT look at IP addresses when making forwarding decisions.
- A router strips the old L2 frame at every hop and builds a brand-new one for the next segment.
- A Layer 3 switch routes internally using hardware (ASICs) — much faster than sending traffic to an external router.

---

## Collision Domain vs Broadcast Domain (Module 2 Relevance)

| Domain | Definition | Separated by |
|---|---|---|
| **Collision domain** | Segment where simultaneous transmissions collide | Each switch port (1 per port) |
| **Broadcast domain** | Segment where all devices receive a broadcast | Routers + VLANs |

| Device | Collision Domains | Broadcast Domains |
|---|---|---|
| Hub (4 ports) | 1 (all ports share) | 1 |
| Switch (4 ports, no VLANs) | 4 | 1 |
| Switch (4 ports, 2 VLANs) | 4 | 2 |
| Router (2 interfaces) | 2 | 2 |

---

# SECTION 2 — How a Switch Works: CAM Table and Forwarding Logic

---

## The Problem a Switch Solves

A **hub** blindly copies every frame to every port — everyone on the network gets everything. A **switch** is intelligent. It builds a map of which device is on which port and forwards traffic only where it needs to go.

---

## Step-by-Step: How a Switch Handles a Frame

```
Step 1: Frame arrives on a switch port.

Step 2: Switch reads the SOURCE MAC address.
        → Records: [Source MAC] → [Port] → [VLAN] in CAM table.
        → This is how the switch "learns" who is where.

Step 3: Switch reads the DESTINATION MAC address.
        → Looks it up in the CAM table.
        → Three possible outcomes:
```

---

## The Three Forwarding Decisions

| Situation | Decision | Why |
|---|---|---|
| **Known Unicast** — destination MAC is in the CAM table | **Forward** out of that one specific port | Switch knows exactly where device lives |
| **Unknown Unicast** — destination MAC is NOT in the CAM table | **Flood** out all ports except the incoming port | Switch must find the device |
| **Broadcast** — destination MAC is `FF:FF:FF:FF:FF:FF` | **Flood** out all ports in the same VLAN (except incoming) | Frame is intended for all devices |

```
Known Unicast:
PC1 → Switch → Port 5 only (Switch knows PC3 is on Port 5)

Unknown Unicast (Flooding):
PC1 → Switch → Port 2, Port 3, Port 4, Port 5 (all except Port 1)
When PC3 replies, Switch learns PC3 is on Port 5 → records it.

Broadcast (ARP):
PC1 → Switch → all ports (same VLAN)
"Who has 192.168.1.1? Tell 192.168.1.10"
Every device hears it.
```

---

## CAM Table (MAC Address Table)

The CAM table stores the mapping the switch builds from watching traffic.

| Column | Description |
|---|---|
| MAC Address | The source MAC the switch saw |
| Port | Which port the frame arrived on |
| VLAN | Which VLAN the port belongs to |
| Age Timer | How long since this MAC was last seen |

- **Default aging time:** 300 seconds (5 minutes)
- **What happens when a MAC ages out?** The entry is deleted. If traffic arrives again, the switch re-learns it. Until then, frames to that MAC will be flooded.
- **Why aging matters:** Prevents the CAM table from filling with stale entries from devices that have moved or disconnected.

---

## Hub vs Switch — Side by Side

```
Hub (L1):                          Switch (L2):
  PC1 ─┐                             PC1 ─── Port 1 (collision domain 1)
  PC2 ─┼── Hub ──── all ports         PC2 ─── Port 2 (collision domain 2)
  PC3 ─┘                              PC3 ─── Port 3 (collision domain 3)

If PC1 and PC3 transmit at once:    If PC1 and PC3 transmit at once:
  COLLISION (shared medium)           No collision (separate port buffers)

One collision domain.               One collision domain per port.
One broadcast domain.               One broadcast domain (unless VLANs).
Everyone sees all traffic.          Only destination port gets unicast.
```

> **Interview answer:**
> "A switch learns MAC addresses by reading the source MAC of incoming frames and recording them in its CAM table. When forwarding, if the destination MAC is known, it sends only to that port. If unknown, it floods all ports except the incoming one. Broadcasts are also flooded within the same VLAN. Each switch port is a separate collision domain, but all ports share one broadcast domain unless VLANs are used."

---

# SECTION 3 — VLANs (Virtual Local Area Networks)

---

## What is a VLAN?

A VLAN is a **logical broadcast domain** created on a switch. It divides one physical switch into multiple isolated virtual switches.

```
Without VLANs (one big room):         With VLANs (separate rooms):
    Switch                                Switch
A B C D E F                          VLAN10 | VLAN20
all in one broadcast domain           A B C | D E F
                                      isolated | isolated
```

Devices in the same VLAN communicate normally. Devices in different VLANs cannot talk directly — they need a Layer 3 device (router or Layer 3 switch).

---

## Why VLANs Exist

| Problem Without VLANs | How VLANs Solve It |
|---|---|
| 500 devices share one broadcast domain → slow | Broadcasts stay inside the VLAN — smaller domain |
| Guest PC can see HR/Finance traffic | VLANs isolate traffic between departments |
| Physical rewiring needed to change departments | Change a port's VLAN in config — no rewiring |
| One large flat network is hard to manage | Separate VLANs for HR, Engineering, Finance, Guests |

---

## The Golden Rule of VLANs

> **Devices in different VLANs cannot communicate directly at Layer 2, even if they are plugged into the exact same physical switch.**

If HR (VLAN 10) needs to send a file to Engineering (VLAN 20), that traffic must go through a Layer 3 device. This is called **inter-VLAN routing**.

---

## VLAN ID

- VLANs are identified by numbers (VLAN IDs): range **1 to 4094**.
- VLAN 1 is the default VLAN on all Cisco switches — all ports start in VLAN 1.
- Best practice: keep VLAN 1 unused for user traffic. Use it only for management or leave it empty.

---

## VLAN and Subnet Relationship

Every VLAN gets its own subnet in standard enterprise design:

| VLAN | Purpose | Subnet |
|---|---|---|
| VLAN 10 | HR | 192.168.10.0/24 |
| VLAN 20 | Finance | 192.168.20.0/24 |
| VLAN 30 | IT | 192.168.30.0/24 |

**Rule:** 1 VLAN = 1 Subnet. VLAN is Layer 2. Subnet is Layer 3. They are different things but designed to match.

> **Interview answer:**
> "A VLAN is a logical broadcast domain on a switch. It segments one physical switch into multiple isolated networks. Devices in different VLANs cannot communicate directly — they need a Layer 3 device. VLANs improve security, reduce broadcast traffic, and allow logical segmentation without physical rewiring."

---

# SECTION 4 — Access Ports and Trunk Ports

---

## Access Port

An **access port** carries traffic for **one VLAN only**. End devices connect here.

```
PC ─────────── Switch Port Fa0/1 (Access, VLAN 10)

PC sends normal Ethernet frame (no VLAN tag).
Switch receives it on Port Fa0/1.
Switch says: "This port is in VLAN 10. I'll treat this as VLAN 10 traffic."
```

**Key properties:**
- Carries one VLAN.
- Frames are sent and received **untagged** (no 802.1Q tag).
- The end device has no knowledge of VLANs.
- Used for: PCs, printers, IP phones, cameras, servers.

---

## Trunk Port

A **trunk port** carries traffic for **multiple VLANs** over a single link.

```
Switch A ══════════ Trunk ══════════ Switch B
          VLAN10 + VLAN20 + VLAN30
          all on one physical cable
```

**Key properties:**
- Carries multiple VLANs.
- Uses **802.1Q tagging** to identify which VLAN each frame belongs to.
- Tag is stripped before delivery to end devices.
- Used for: switch-to-switch links, switch-to-router links, switch-to-access point links.

---

## 802.1Q Tagging — How It Works

When a frame enters a trunk port, the switch inserts a **4-byte 802.1Q tag** into the Ethernet frame header:

```
Normal Ethernet Frame:
[ Destination MAC | Source MAC | EtherType | Payload | FCS ]

802.1Q Tagged Frame:
[ Destination MAC | Source MAC | 802.1Q Tag | EtherType | Payload | FCS ]
                                    ↑
                             4-byte insert:
                             - Tag Protocol ID (0x8100)
                             - Priority bits (CoS)
                             - VLAN ID (12 bits → 4094 VLANs)
```

**Flow on a trunk:**

```
PC1 (VLAN 10) sends frame → Switch A
Switch A sees: Port Fa0/1 is VLAN 10
Switch A inserts 802.1Q tag (VLAN ID = 10)
Tagged frame travels across trunk link to Switch B
Switch B reads tag: "VLAN 10"
Switch B strips tag
Switch B delivers frame to the correct VLAN 10 port
End device (PC2) receives normal untagged frame
```

> **Key point:** The end device never sees the 802.1Q tag. The tag is added by the sending switch and removed by the receiving switch. It is purely internal to the network infrastructure.

---

## Access vs Trunk — Full Comparison

| Feature | Access Port | Trunk Port |
|---|---|---|
| VLANs carried | One only | Multiple |
| 802.1Q tagging | None (untagged) | Yes, on all VLANs except native |
| Typical connection | PC, printer, server, phone | Switch↔Switch, Switch↔Router |
| End device VLAN awareness? | No | N/A |
| When to use | Edge of network | Between infrastructure devices |

> **Memory trick:**
> Access = single-lane road (one VLAN, one destination)
> Trunk = multi-lane highway (many VLANs, shared cable)
> 802.1Q tag = the lane marker label on each vehicle

---

# SECTION 5 — Native VLAN (In Depth)

---

## What is the Native VLAN?

The **native VLAN** is the one special VLAN on a trunk link whose frames are sent **untagged** — without a 802.1Q tag. All other VLANs on the trunk are tagged.

```
Trunk link with Native VLAN = 1:

VLAN 10 frame → [802.1Q Tag: VLAN 10 + data]   → Tagged
VLAN 20 frame → [802.1Q Tag: VLAN 20 + data]   → Tagged
VLAN 1 frame  → [data]                          → UNTAGGED (native)
```

- **Default native VLAN on all Cisco switches:** VLAN 1
- The receiving switch assumes any **untagged** frame belongs to the native VLAN.

---

## Why the Native VLAN Exists

**Reason 1 — Backward compatibility:**
Old legacy devices (unmanaged switches, old hubs) don't understand 802.1Q tags. They drop tagged frames as corrupted. The native VLAN allows them to still pass basic traffic.

**Reason 2 — Control plane traffic:**
Important Cisco protocols (STP BPDUs, CDP, DTP) send their management frames as untagged traffic by default. The native VLAN is where these travel.

---

## Native VLAN Mismatch — The Danger

```
Switch A: Native VLAN = 1
Switch B: Native VLAN = 99

What happens:
1. PC in VLAN 1 on Switch A sends broadcast.
2. Switch A sends it untagged across the trunk.
3. Switch B receives untagged frame.
4. Switch B says: "No tag? Must be my native VLAN = 99"
5. Switch B places it into VLAN 99 — WRONG VLAN!

Result:
- Traffic leaks into the wrong VLAN.
- Devices in VLAN 99 receive traffic not meant for them.
- Security risk + broken communication.
```

**Cisco will warn you:**
`%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on GigabitEthernet0/1`

> **Rule: Both ends of a trunk MUST have the same native VLAN. Always verify.**

---

## Native VLAN Security Best Practices

| Practice | Reason |
|---|---|
| Change native VLAN from VLAN 1 to an unused VLAN (e.g., VLAN 999) | VLAN 1 carries control traffic by default; isolate it from user access |
| Never assign users, servers, or end devices to the native VLAN | Keeps the native VLAN empty and reduces attack surface |
| Use `vlan dot1q tag native` to force-tag the native VLAN | Eliminates untagged frames entirely, removes the double-tagging hopping vector |

**Why change from VLAN 1?**
VLAN 1 is the default on all Cisco switches. A malicious user who knows the defaults can attempt a **VLAN hopping attack** by crafting double-tagged frames using VLAN 1 as the outer tag.

---

## VLAN Double-Tagging Hopping Attack (Interview Awareness)

A VLAN hopping attack exploits the native VLAN:

```
Attacker in VLAN 1 (native) sends a frame with TWO 802.1Q tags:
Outer tag: VLAN 1 (native, stripped by first switch)
Inner tag: VLAN 20 (target VLAN)

Switch A strips outer tag (native VLAN, as expected).
Sends the remaining frame (inner VLAN 20 tag) across trunk.
Switch B reads inner tag: VLAN 20.
Frame lands in VLAN 20 without going through a router.
```

**Fix:** Never use VLAN 1 as native VLAN. Change to an unused VLAN ID everywhere.

> **Interview answer:**
> "The native VLAN is the one untagged VLAN on a trunk link. Frames belonging to it cross the trunk without an 802.1Q tag. Both ends must have the same native VLAN — a mismatch causes frames to land in the wrong VLAN, creating connectivity problems and a security vulnerability. Best practice is to change the native VLAN from the default VLAN 1 to an unused VLAN ID."

---

# SECTION 6 — Inter-VLAN Routing

---

## The Problem

VLANs are isolated by design. A device in VLAN 10 cannot communicate with a device in VLAN 20 at Layer 2 — even on the same physical switch. A Layer 3 device must route the traffic between them.

```
PC1 (VLAN 10, 192.168.10.10) → wants to reach → PC2 (VLAN 20, 192.168.20.20)

Layer 2 says: "You're in different VLANs. I can't help you."
Layer 3 says: "I can route between these subnets."
```

---

## Method 1 — Router-on-a-Stick (RoAS)

Uses a **single physical link** between a Layer 2 switch and an external router, configured as a trunk. The router's physical interface is divided into logical **sub-interfaces** — one per VLAN.

```
         Layer 2 Switch
         /     |     \
     VLAN10  VLAN20  Trunk Port
                        |
                      Router
                   Gig0/0.10 (VLAN 10 gateway: 192.168.10.1)
                   Gig0/0.20 (VLAN 20 gateway: 192.168.20.1)
```

**Traffic flow (PC1 in VLAN 10 → PC2 in VLAN 20):**

```
1. PC1 (192.168.10.10) sends packet to default GW 192.168.10.1
2. Frame travels up the trunk (tagged VLAN 10)
3. Router receives on sub-interface Gig0/0.10
4. Router reads destination IP (192.168.20.20)
5. Router routes to Gig0/0.20
6. Router sends frame back DOWN the same physical trunk (tagged VLAN 20)
7. Switch receives, strips tag, delivers to PC2 on VLAN 20 port
```

**Router configuration concept:**

```cisco
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```

**Limitation of RoAS:**

All inter-VLAN traffic must travel up and down the same single physical cable. High traffic between VLANs creates a **bottleneck** on that one link — hence the name "router-on-a-stick" (all traffic balanced on one stick).

**When to use:** Small networks, lab environments, budget setups where only a Layer 2 switch and router are available.

---

## Method 2 — Layer 3 Switch with SVIs

A **Layer 3 switch** (multilayer switch) performs routing internally in hardware. No external router needed for inter-VLAN traffic. Uses **SVIs (Switched Virtual Interfaces)** — one logical Layer 3 interface per VLAN.

```
         Layer 3 Switch
         /     |     \
     VLAN10  VLAN20  VLAN30

Internal routing:
SVI for VLAN 10: ip address 192.168.10.1 → gateway for VLAN 10
SVI for VLAN 20: ip address 192.168.20.1 → gateway for VLAN 20
SVI for VLAN 30: ip address 192.168.30.1 → gateway for VLAN 30
```

**SVI configuration concept:**

```cisco
interface Vlan10
 ip address 192.168.10.1 255.255.255.0

interface Vlan20
 ip address 192.168.20.1 255.255.255.0

ip routing
```

**Traffic flow (PC1 in VLAN 10 → PC2 in VLAN 20):**

```
1. PC1 sends packet to default gateway 192.168.10.1 (SVI VLAN 10)
2. Layer 3 switch receives it internally (never leaves the box)
3. Switch routes packet to the VLAN 20 subnet via SVI VLAN 20
4. Switch delivers frame to PC2 on VLAN 20 port
No trunk cable bottleneck. All routing done in hardware.
```

---

## RoAS vs Layer 3 Switch — Full Comparison

| Feature | Router-on-a-Stick | Layer 3 Switch (SVI) |
|---|---|---|
| Hardware needed | L2 Switch + External Router | One multilayer switch |
| Link efficiency | Shared single link (bottleneck risk) | Internal backplane (no bottleneck) |
| Speed | Limited by physical link speed | Hardware ASICs — very fast |
| Configuration | Router sub-interfaces | SVIs on the switch |
| Best for | Small/budget networks, labs | Enterprise, heavy inter-VLAN traffic |
| External router still needed? | Yes (for Internet) | Only for external/Internet traffic |

> **Interview answer:**
> "Inter-VLAN routing allows devices in different VLANs to communicate through a Layer 3 device. Router-on-a-stick uses one physical trunk link and sub-interfaces on a router — simple but can bottleneck under heavy traffic. A Layer 3 switch uses SVIs and routes internally in hardware — much faster and preferred in enterprise networks."

---

# SECTION 7 — Spanning Tree Protocol (STP and RSTP)

---

## Why STP Exists — The Layer 2 Loop Problem

Redundant switch links are added to networks for fault tolerance — if one link fails, another takes over. But redundancy creates **physical loops**. Ethernet frames have no TTL field (unlike IP packets). Without a loop-prevention mechanism:

```
Switch A ─────── Switch B
      \           /
       Switch C ─

A broadcast sent by Switch A:
→ B forwards it to C
→ C forwards it back to A
→ A forwards it to B again
→ Loops forever at wire speed

Results:
1. Broadcast storm   → all bandwidth consumed, network crashes
2. MAC table thrashing → switch sees same MAC on multiple ports, can't learn
3. Duplicate frames → hosts receive multiple copies of the same data
```

**STP solves this by keeping redundant links physically present but logically blocking specific ports, so only one forwarding path exists between any two points.**

---

## How STP Works — 3-Step Convergence

### Step 1: Root Bridge Election

One switch becomes the **Root Bridge** — the central reference point for the entire topology. All other switches calculate their best path to the Root Bridge.

**Election rule:** The switch with the **lowest Bridge ID (BID)** wins.

**Bridge ID structure:**

```
Bridge ID = [Priority (2 bytes)] + [MAC Address (6 bytes)]

Default priority on all Cisco switches: 32768
If two switches have the same priority: lower MAC address wins.

Example:
Switch A: Priority 32768, MAC 00:1A:2B:3C:4D:01
Switch B: Priority 32768, MAC 00:1A:2B:3C:4D:02

Switch A wins (lower MAC) → Switch A is Root Bridge
```

**How to manually force a switch to be Root Bridge:**

```cisco
spanning-tree vlan 1 priority 4096
```

Lower priority = more likely to be elected Root Bridge.

> **Design Rule:** Never leave the Root Bridge to be determined by MAC address. In production, always manually set the core/distribution switch as the Root Bridge by lowering its priority.

---

### Step 2: Root Port (RP) Election

Every **non-root switch** selects exactly **one Root Port** — the port with the best (lowest cost) path back to the Root Bridge.

**STP Port Cost by bandwidth:**

| Bandwidth | STP Cost |
|---|---|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

Lower cost = better path = preferred as Root Port.

**Tiebreakers (in order):**
1. Lowest **path cost** to root.
2. Lowest **neighbor Bridge ID** (sender BID in BPDU).
3. Lowest **neighbor port priority**.
4. Lowest **neighbor port number**.

---

### Step 3: Designated Port (DP) Election

Every **network segment** (every link between switches) must have exactly **one Designated Port**. This is the port responsible for forwarding traffic on that segment toward the Root Bridge.

**Rules:**
- All ports on the **Root Bridge** are automatically Designated Ports.
- On segments between two non-root switches: the switch with the lower path cost to the root has the Designated Port.
- Any port that is neither a Root Port nor a Designated Port is placed into **Blocking** state.

---

## STP Port Roles Summary

| Role | Description | State |
|---|---|---|
| **Root Port (RP)** | Best path to root on a non-root switch | Forwarding |
| **Designated Port (DP)** | Best port on a segment for reaching root | Forwarding |
| **Blocked Port (BP)** | Redundant port — breaks the loop | Blocking |

---

## STP Port States (802.1D)

```
Blocking  → Listening  → Learning  → Forwarding
(15 sec)     (15 sec)    (stable)
```

| State | Receives BPDUs? | Learns MACs? | Forwards Data? |
|---|---|---|---|
| **Blocking** | Yes | No | No |
| **Listening** | Yes | No | No |
| **Learning** | Yes | Yes | No |
| **Forwarding** | Yes | Yes | Yes |
| **Disabled** | No | No | No |

**Total STP convergence time (802.1D):** Up to **30–50 seconds**
(Listening = 15 sec + Learning = 15 sec, controlled by Forward Delay timer)

---

## RSTP — Rapid Spanning Tree Protocol (802.1w)

RSTP achieves **1–2 second convergence** instead of 30–50 seconds by replacing passive timer waiting with an active **Proposal-Agreement handshake** between adjacent switches.

### RSTP Port States (Simplified)

| RSTP State | What it does |
|---|---|
| **Discarding** | Combines Blocking + Listening from STP. No data, just BPDUs. |
| **Learning** | Builds MAC table. No data forwarding yet. |
| **Forwarding** | Normal operation. Full data + BPDU. |

### RSTP Port Roles (Added role)

- **Alternate Port** — Backup to the Root Port. Instantly takes over if the Root Port fails.
- **Backup Port** — Backup Designated Port on the same segment.

### STP vs RSTP Comparison

| Feature | STP (802.1D) | RSTP (802.1w) |
|---|---|---|
| Convergence time | 30–50 seconds | 1–2 seconds |
| Port states | 5 | 3 |
| Port roles | 3 (RP, DP, Blocked) | 5 (RP, DP, Alternate, Backup, Discarding) |
| Mechanism | Timer-based | Proposal-Agreement handshake |
| Standard | Older (1998) | Modern, preferred |

---

## STP Example Topology

```
        Switch A (Root Bridge — lowest BID)
       /                        \
   Link 1                      Link 2
     /                            \
Switch B ─────── Link 3 ───── Switch C

STP Result:
- Switch A: all ports = Designated Ports (Root Bridge)
- Switch B: port toward A = Root Port (RP)
- Switch C: port toward A = Root Port (RP)
- Link 3 between B and C:
  - Switch B's port = Designated Port (lower path cost to root)
  - Switch C's port = BLOCKED (higher path cost, redundant path)

One forwarding path: B → A → C
Redundant link (B-C) is blocked, but physically available for failover.
```

---

## BPDU (Bridge Protocol Data Unit)

BPDUs are the management frames STP uses to communicate between switches. They carry:
- Bridge ID (to identify the sender)
- Root Bridge ID (current root belief)
- Path cost to root
- Port ID

Switches send Hello BPDUs every **2 seconds** by default. If a switch stops receiving BPDUs, it assumes the neighbor failed and begins reconvergence.

> **Interview answer:**
> "STP prevents Layer 2 loops by electing a Root Bridge — the switch with the lowest Bridge ID — and then blocking redundant ports. Every non-root switch selects a Root Port (best path to root) and every segment has a Designated Port (forwarding port). Any remaining ports are blocked. RSTP improves this with a Proposal-Agreement handshake, reducing convergence from 30–50 seconds to 1–2 seconds."

---

# SECTION 8 — PortFast and BPDU Guard

---

## The Problem PortFast Solves

When you plug a PC into a switch access port, normal STP makes the port cycle through:

```
Blocking (15 sec) → Listening (15 sec) → Learning → Forwarding
Total wait: ~30 seconds before PC can send data or get DHCP
```

For a PC or printer, this delay is unnecessary — they are not switches and will never cause a loop.

---

## PortFast

**What it does:** Skips the Listening and Learning states entirely. The port moves **immediately to Forwarding** when the cable is plugged in.

```
With PortFast:
Cable plugged in → Forwarding (instant)
PC can get DHCP immediately.
```

**Where to use:** Access ports connected to end devices only (PCs, printers, IP phones, servers).

**Configuration:**

```cisco
interface FastEthernet0/1
 spanning-tree portfast
```

Or enable on all access ports by default:

```cisco
spanning-tree portfast default
```

### The Danger of PortFast on the Wrong Port

```
⚠️ NEVER use PortFast on switch-to-switch links.

What happens if you do:
Switch B connects to a PortFast port on Switch A.
Port immediately goes to Forwarding.
Loop forms before STP has time to detect and block it.
Broadcast storm begins within milliseconds.
```

---

## BPDU Guard

**What it does:** If a PortFast-enabled port receives a **BPDU** (which only switches send), BPDU Guard **immediately shuts the port down** by putting it into **err-disabled** state.

```
PortFast port (access port)
  ↓
Someone plugs in an unauthorized switch
  ↓
Switch starts sending BPDUs
  ↓
BPDU Guard detects BPDU
  ↓
Port → err-disabled (shut down immediately)
  ↓
Network protected from loop
```

**Why BPDU Guard is critical:** Without it, an employee in a cubicle could plug a small unmanaged switch into a wall jack and accidentally create a loop that takes down the entire building's network.

**Configuration:**

```cisco
interface FastEthernet0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
```

Or enable globally for all PortFast ports:

```cisco
spanning-tree portfast bpduguard default
```

**How to recover an err-disabled port:**

```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
```

Or configure automatic recovery:

```cisco
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

---

## BPDU Guard vs BPDU Filter — The Critical Interview Trap

This is one of the most tested distinctions in networking interviews.

| Feature | BPDU Guard | BPDU Filter |
|---|---|---|
| What it does | **Shuts port down** (err-disabled) if BPDU received | **Stops BPDUs** from being sent or received on the port |
| Safe to use? | Yes — protective action | Dangerous if misused on uplinks |
| Effect on STP? | Port is disabled — STP cannot use it | Port is invisible to STP |
| Risk | None — by design | If used on an uplink, STP loop can form undetected |
| Where to use | PortFast access ports | Very limited use cases only |

> **Why BPDU Filter is dangerous:** If you apply BPDU Filter on a switch-to-switch link, that port stops sending and receiving BPDUs. STP cannot see or manage that link. If a loop exists there, STP will never detect it, and a broadcast storm will run silently until the network crashes.

> **Interview answer for BPDU Guard:**
> "BPDU Guard is a safety mechanism for PortFast ports. If a PortFast-configured access port receives a BPDU — which only switches send — BPDU Guard immediately puts the port into err-disabled state. This prevents an unauthorized switch from being connected to an access port and accidentally creating a Layer 2 loop."

> **Interview answer for the BPDU Guard vs Filter trap:**
> "BPDU Guard shuts down the port if a BPDU is received — it is a safe protective response. BPDU Filter prevents BPDUs from being sent or received at all. If BPDU Filter is applied to an uplink, STP becomes blind to that port, loops can form undetected, and the result is a broadcast storm. BPDU Guard is safe; BPDU Filter requires very careful use."

---

# SECTION 9 — EtherChannel / LACP (Awareness Level)

---

## What is EtherChannel?

EtherChannel bundles multiple physical links between two switches into a **single logical link**. This provides both **bandwidth aggregation** and **redundancy**.

```
Without EtherChannel:
Switch A ─── 1 Gbps ─── Switch B
(only one link active; STP blocks the other)

With EtherChannel (2 links bundled):
Switch A ══════════════ Switch B
        logical 2 Gbps
        (both links active; STP sees one link)
```

**Why STP matters with EtherChannel:**
Without EtherChannel, STP blocks one of the redundant links. With EtherChannel, STP sees the bundle as a single logical interface and all physical links can forward traffic simultaneously.

---

## LACP vs PAgP

| Protocol | Type | Description |
|---|---|---|
| **LACP** (802.3ad) | Open standard | IEEE standard — works across any vendor |
| **PAgP** | Cisco proprietary | Only works between Cisco devices |

**LACP modes:**
- `Active` — sends LACP negotiation packets.
- `Passive` — waits to receive LACP packets.
- At least one side must be `Active`.

---

# SECTION 10 — Module 2 Rapid-Fire Q&A

---

> Practice: Read question, cover answer, respond in one sentence. Uncover and check.

---

**Q: What is a switch?**
A: A Layer 2 device that forwards Ethernet frames based on MAC addresses using a CAM table.

**Q: How does a switch learn MAC addresses?**
A: It reads the source MAC address of incoming frames and records the MAC-to-port mapping in the CAM table.

**Q: What is the CAM table?**
A: The MAC address table that stores MAC address, port, VLAN, and aging timer for each known device.

**Q: What does a switch do when the destination MAC is unknown?**
A: It floods the frame out all ports except the incoming port.

**Q: What does a switch do with a broadcast frame?**
A: Floods it out all ports in the same VLAN except the incoming port.

**Q: What is the default CAM table aging time?**
A: 300 seconds (5 minutes). Entries not refreshed within this time are removed.

**Q: What is a VLAN?**
A: A logical broadcast domain created on a switch that segments traffic without requiring separate physical switches.

**Q: Why are VLANs used?**
A: Segmentation, security, reduced broadcast traffic, and flexibility to reorganize departments without rewiring.

**Q: Can devices in different VLANs talk directly?**
A: No. Different VLANs cannot communicate at Layer 2 — they need a Layer 3 device (router or Layer 3 switch).

**Q: What is an access port?**
A: A switch port that carries traffic for one VLAN only, used for end devices. Frames are untagged.

**Q: What is a trunk port?**
A: A switch port that carries multiple VLANs using 802.1Q tagging, used between network devices.

**Q: What is 802.1Q tagging?**
A: A 4-byte tag inserted into the Ethernet frame on trunk ports to identify which VLAN the frame belongs to.

**Q: What is the native VLAN?**
A: The one VLAN on a trunk link whose frames are sent untagged. Default is VLAN 1 on Cisco switches.

**Q: What is a native VLAN mismatch?**
A: When both ends of a trunk have different native VLANs, causing untagged frames to be placed in the wrong VLAN — a security and connectivity problem.

**Q: What is inter-VLAN routing?**
A: The process of routing traffic between different VLANs using a Layer 3 device.

**Q: What is Router-on-a-Stick?**
A: A method where one physical router interface is divided into sub-interfaces, each serving one VLAN over a trunk link.

**Q: What is an SVI?**
A: A Switched Virtual Interface — a logical Layer 3 interface on a Layer 3 switch, one per VLAN, acting as the gateway for that VLAN.

**Q: Why is Layer 3 switch routing faster than Router-on-a-Stick?**
A: Routing happens in hardware ASICs inside the switch — no single cable bottleneck, no external device round-trip.

**Q: Why is STP needed?**
A: To prevent Layer 2 loops caused by redundant switch links. Without it, broadcast storms and MAC table instability crash the network.

**Q: What is a broadcast storm?**
A: When broadcast frames loop endlessly through switches with no TTL mechanism, consuming all bandwidth and crashing the network.

**Q: How is the Root Bridge elected?**
A: The switch with the lowest Bridge ID (priority + MAC address) becomes the Root Bridge.

**Q: What is the default STP bridge priority?**
A: 32768 on all Cisco switches. If equal, the lower MAC address wins.

**Q: What is a Root Port?**
A: The port on a non-root switch with the best (lowest cost) path back to the Root Bridge.

**Q: What is a Designated Port?**
A: The forwarding port on a network segment responsible for sending traffic toward the Root Bridge.

**Q: What is a Blocked Port?**
A: A redundant port placed in blocking state by STP to prevent loops. It can take over if the active path fails.

**Q: How long does STP take to converge?**
A: 30–50 seconds for STP (802.1D).

**Q: How long does RSTP take to converge?**
A: 1–2 seconds using the Proposal-Agreement handshake mechanism.

**Q: What is PortFast?**
A: An STP feature that bypasses Listening and Learning states, allowing access ports to go directly to Forwarding for faster end-device connectivity.

**Q: Where should PortFast never be used?**
A: On switch-to-switch links — a loop forms before STP can detect and block it.

**Q: What is BPDU Guard?**
A: A feature that immediately puts a PortFast port into err-disabled state if it receives a BPDU, protecting against unauthorized switch connections.

**Q: What is the difference between BPDU Guard and BPDU Filter?**
A: BPDU Guard shuts the port down if a BPDU is received (safe). BPDU Filter prevents BPDUs from being sent or received (dangerous on uplinks — STP loop goes undetected).

**Q: What is EtherChannel?**
A: A method to bundle multiple physical links into one logical link for bandwidth aggregation and redundancy.

**Q: What is LACP?**
A: Link Aggregation Control Protocol — an open-standard protocol for negotiating EtherChannel between devices.

---

# SECTION 11 — Module 2 Trap Questions

---

> These are follow-up questions interviewers ask after your initial answer. Practice not freezing.

---

| Trap Question | Strong Answer |
|---|---|
| What happens when a MAC address ages out of the CAM table? | The entry is deleted. The next frame to that MAC will be flooded until the device sends traffic and is re-learned. |
| Can a switch forward a frame without a CAM table entry? | Yes — it floods. Flooding is the fallback for unknown unicast. The device is found on reply and then learned. |
| What is the difference between flooding and broadcasting? | Flooding is a switch action for unknown unicasts (destination MAC unknown). Broadcasting is when a frame is destined for FF:FF:FF:FF:FF:FF (all devices). Both result in forwarding out all ports, but for different reasons. |
| Why does a trunk port strip the 802.1Q tag before delivery to an end device? | End devices don't understand 802.1Q tags. If the tag isn't removed, the device would see a corrupted frame. The switch always removes it on delivery. |
| Can two different VLANs use the same subnet? | No. That would cause ARP to fail — hosts think the destination is local, send a broadcast, but the broadcast can't cross VLANs. Different VLANs must have different subnets. |
| What is the bottleneck of Router-on-a-Stick? | All inter-VLAN traffic must physically travel up and back down the single trunk link to the router. Under heavy traffic, this becomes a bandwidth bottleneck. |
| What happens if `ip routing` is not enabled on a Layer 3 switch? | The switch acts as a pure Layer 2 switch — SVIs will have IP addresses but the switch will not route between VLANs. `ip routing` must be enabled for inter-VLAN routing to work. |
| Why should you manually set the Root Bridge instead of letting STP elect it? | Default election is by MAC address, which is unpredictable. The Root Bridge should always be the most capable, central switch for optimal traffic flow. Manual priority setting ensures this. |
| What is the difference between STP port roles and port states? | Roles describe the port's function: Root Port, Designated Port, Blocked Port. States describe what the port is currently doing: Blocking, Listening, Learning, Forwarding. A port has both a role and a state simultaneously. |
| What is the difference between RSTP's Alternate Port and a Blocked Port? | In STP, a blocked port is just "blocked." In RSTP, an Alternate Port is a pre-calculated backup to the Root Port — it knows the path to the root and can transition to forwarding almost instantly when the Root Port fails, instead of recomputing from scratch. |
| Can a VLAN exist on a trunk but not be created on the switch? | Yes, traffic for that VLAN will travel the trunk, but the switch at the other end won't know what to do with it unless the VLAN is also created in its VLAN database. Always create VLANs on all switches they need to traverse. |
| What is a VLAN hopping attack? | An attacker in the native VLAN crafts a double-tagged frame. The first switch strips the outer native VLAN tag. The inner tag (targeting VLAN 20) is then read by the next switch, placing the frame in VLAN 20 without routing. Fix: change native VLAN to an unused ID. |
| Should PortFast be combined with BPDU Guard? | Yes, always. PortFast alone makes access ports faster but doesn't protect against someone connecting a switch. BPDU Guard is the safety net that shuts down the port if a switch is detected. They should always be deployed together. |

---

# SECTION 12 — Module 2 Completion Checklist

---

Tick each item once you can explain it in interview style without notes.

### Network Devices
- [ ] Hub vs Switch vs Router vs Layer 3 Switch — layer, function, what it reads
- [ ] Collision domain and broadcast domain — definition, how each device affects them

### Switch Forwarding
- [ ] How a switch learns MAC addresses — source MAC + CAM table
- [ ] Three forwarding decisions — known unicast, unknown unicast (flood), broadcast (flood)
- [ ] CAM table structure — MAC, port, VLAN, aging timer
- [ ] Default aging time and what happens when it expires
- [ ] Hub vs switch comparison — collision domains, broadcast domains, traffic handling

### VLANs
- [ ] What a VLAN is — logical broadcast domain, one switch = multiple virtual switches
- [ ] Why VLANs are used — segmentation, security, performance, flexibility
- [ ] VLAN golden rule — different VLANs need Layer 3 to communicate
- [ ] VLAN ID range — 1 to 4094; default VLAN is 1

### Access and Trunk Ports
- [ ] Access port — one VLAN, untagged, end devices
- [ ] Trunk port — multiple VLANs, 802.1Q tagged, infrastructure links
- [ ] 802.1Q tagging — 4-byte insert, VLAN ID, how it is added and stripped
- [ ] When the tag is removed — always before delivery to end device

### Native VLAN
- [ ] What native VLAN is — untagged VLAN on a trunk
- [ ] Why it exists — legacy compatibility, control traffic
- [ ] Native VLAN mismatch — what happens, how to detect, how to fix
- [ ] Security best practices — change from VLAN 1 to unused VLAN, never assign users
- [ ] VLAN hopping attack concept — double-tagging, prevention

### Inter-VLAN Routing
- [ ] Router-on-a-Stick — sub-interfaces, trunk link, bottleneck limitation
- [ ] Layer 3 Switch with SVIs — internal routing, hardware speed, `ip routing` command
- [ ] Full traffic flow for both methods — step by step from PC1 to PC2 across VLANs
- [ ] When to use each method — small/budget vs enterprise/performance

### STP and RSTP
- [ ] Why STP is needed — broadcast storm, MAC instability, duplicate frames
- [ ] Root Bridge election — lowest Bridge ID (priority + MAC)
- [ ] Three port roles — Root Port, Designated Port, Blocked Port
- [ ] STP port states — Blocking, Listening, Learning, Forwarding, Disabled
- [ ] STP convergence time — 30–50 seconds
- [ ] RSTP improvement — 1–2 seconds, Proposal-Agreement handshake
- [ ] RSTP port states — Discarding, Learning, Forwarding
- [ ] BPDU — what it is, what it carries, how often sent (every 2 seconds)
- [ ] How to manually set Root Bridge — lower the priority value

### PortFast and BPDU Guard
- [ ] PortFast — skips Listening/Learning, immediate Forwarding, access ports only
- [ ] Why PortFast must not be on switch-to-switch links — loop before STP reacts
- [ ] BPDU Guard — err-disables port if BPDU received, protects PortFast ports
- [ ] BPDU Guard vs BPDU Filter — shut vs suppress, safe vs dangerous
- [ ] How to recover an err-disabled port — shut / no shut

### EtherChannel
- [ ] What EtherChannel does — bundles physical links into one logical link
- [ ] LACP — open standard for EtherChannel negotiation

---

## Module 2 Labs to Complete

- [ ] Create VLANs on a switch and assign ports to each VLAN
- [ ] Verify traffic is isolated between VLANs (ping should fail between VLANs without routing)
- [ ] Configure a trunk link between two switches and allow specific VLANs
- [ ] Verify native VLAN matches on both trunk ends
- [ ] Configure Router-on-a-Stick and test inter-VLAN connectivity
- [ ] Configure SVI-based inter-VLAN routing on a Layer 3 switch
- [ ] Observe STP root bridge election using `show spanning-tree`
- [ ] Manually set a switch as Root Bridge by changing priority
- [ ] Enable PortFast and BPDU Guard on access ports
- [ ] Test BPDU Guard by plugging in another switch and observe err-disabled state

### Useful Show Commands for Module 2

```cisco
show mac address-table              # View CAM table
show vlan brief                     # List all VLANs and assigned ports
show interfaces trunk               # Trunk links and allowed VLANs
show spanning-tree vlan 10          # STP details for VLAN 10
show spanning-tree summary          # Overview of all STP instances
show interfaces status              # Port VLAN, speed, status
show ip interface brief             # SVI IP and status (Layer 3 switch)
show etherchannel summary           # EtherChannel bundle status
```

---

## One-Page Revision Summary

```
SWITCH: Learns source MAC → CAM table. Forwards to known MAC. Floods unknown + broadcast.
CAM TABLE: MAC | Port | VLAN | Aging (300sec). Delete on age = flood until relearned.

VLAN: Logical broadcast domain. 1 switch = multiple virtual switches. VLAN ID 1–4094.
RULE: Different VLANs = different subnets. Can't communicate at L2 without routing.

ACCESS PORT: 1 VLAN. Untagged. End devices (PC, printer, server).
TRUNK PORT: Multiple VLANs. 802.1Q tagged. Switch↔Switch, Switch↔Router.
802.1Q TAG: 4-byte insert. VLAN ID (12 bits = 4094 VLANs). Stripped before delivery to end device.

NATIVE VLAN: Untagged VLAN on trunk. Default = VLAN 1. Both ends MUST match.
MISMATCH: Frame lands in wrong VLAN. Security risk. Fix: set same native VLAN both ends.
BEST PRACTICE: Change native VLAN to unused ID. Never put users in native VLAN.

INTER-VLAN ROUTING:
  RoAS: Router sub-interfaces + trunk. Simple. Single link bottleneck.
  L3 Switch SVI: Internal hardware routing. Fast. No bottleneck. Preferred in enterprise.

STP:
  Purpose: Prevent L2 loops from redundant links (broadcast storm, MAC instability).
  Root Bridge: Lowest Bridge ID (Priority + MAC). Lower priority = more likely to win.
  Roles: Root Port (best path to root) | Designated Port (forwarding) | Blocked (loop prevention)
  802.1D states: Blocking→Listening→Learning→Forwarding (30–50 sec total)
  RSTP (802.1w): Discarding→Learning→Forwarding (1–2 sec, Proposal-Agreement)

PORTFAST: Access ports only. Skip Listening/Learning → instant Forwarding.
  NEVER on switch-to-switch links (loop before STP reacts).
BPDU GUARD: Err-disables port if BPDU received. Always pair with PortFast.
BPDU FILTER: Suppresses BPDUs. Dangerous on uplinks. Avoid.

ETHERCHANNEL: Bundles links. Bandwidth + redundancy. STP sees as one logical link.
LACP: Open-standard EtherChannel negotiation. Active/Passive modes.
```

---

*End of Module 2 Notes | Cisco G4 Interview Preparation*
*Covers: Hub/Switch/Router, CAM Table, Switch Forwarding, VLANs, Access/Trunk Ports, 802.1Q, Native VLAN, Inter-VLAN Routing (RoAS + L3 SVI), STP, RSTP, PortFast, BPDU Guard, EtherChannel*
