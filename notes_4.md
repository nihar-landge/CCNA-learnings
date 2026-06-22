# Module 4 — IP Services: VRRP, GLBP, NTP, and Syslog
## Cisco G4 Interview Preparation · Print & Revision Notes

> **How to use:** Read once to learn. Cover answers and test yourself aloud. Every section ends with a ready interview answer. Tick each topic when you can answer it without looking.
>
> **Continuity note:** This module continues directly from Module 3's HSRP. VRRP and GLBP solve the exact same problem as HSRP — eliminating the default gateway as a single point of failure — so compare everything here back to HSRP as you read.

---

# SECTION 1 — VRRP (Virtual Router Redundancy Protocol)

---

## 1.1 What is VRRP?

VRRP is an **open standard, IEEE protocol (RFC 5798)** — not Cisco-proprietary. It lets multiple physical routers combine into a single logical "virtual router" on a LAN.

Just like HSRP: end devices are configured with a single **Virtual IP (VIP)** as their default gateway. If the router actively handling traffic drops offline, the backup router instantly intercepts traffic without hosts ever noticing.

> **Why interviewers test this specifically:** VRRP solves the identical problem as HSRP, but uses entirely different terminology, timers, and defaults. Using HSRP terms while explaining VRRP signals shallow knowledge.

---

## 1.2 The Terminology Shift (Critical for Interviews)

| Architecture Role | HSRP (Cisco Proprietary) | VRRP (Open Standard) |
|---|---|---|
| The Active Forwarder | Active Router | **Master Router** |
| The Silent Listener(s) | Standby Router | **Backup Router(s)** |
| Virtual MAC Structure | `0000.0c07.acXX` | **`0000.5e00.01XX`** |
| Multicast Hello Address | `224.0.0.2` | **`224.0.0.18`** |
| Preemption Status | **Disabled** by default | **Enabled** by default |

> Memorize this table as a pair — for every HSRP term, know its VRRP equivalent instantly.

---

## 1.3 How VRRP Operates Mechanically

### Virtual MAC Address

VRRP's dedicated virtual MAC format: **`0000.5e00.01XX`** — where `XX` is the hex representation of the VRRP Group ID. When a PC ARPs for its gateway, the **Master** replies with this virtual MAC.

### Timers and Advertisements

```
Master sends VRRP advertisements every 1 second
  → to multicast 224.0.0.18

(Compare: HSRP sends Hellos every 3 seconds to 224.0.0.2)
```

If the Master goes silent, Backup routers calculate the **Master Down Timer**:

```
Master Down Timer = (3 × Advertisement Interval) + Skew Time

With default 1-second advertisements:
Master Down Timer ≈ 3.2 to 3.5 seconds
```

> **VRRP fails over faster than default HSRP** (≈3.2–3.5 sec vs HSRP's default 10-second hold timer).

### Preemption is ON by Default

Unlike HSRP (preemption disabled by default), **VRRP preempts automatically**. If a Master with priority 110 drops and later reboots, it will **aggressively reclaim** Master status from a lower-priority Backup the instant it returns — no manual command needed.

---

## 1.4 The Core VRRP Secret: IP Address Owner

In HSRP, the Virtual IP **must** be different from any router's physical interface IP.

**VRRP breaks this rule.** You can configure the VIP to be the **exact same IP** already assigned to the Master's physical interface.

```
If you do this:
  → That router is called the IP Address Owner
  → It automatically gets VRRP priority 255 (the maximum)
  → This guarantees it always wins the Master election
```

> **Interview answer:**
> "VRRP is an open-standard FHRP defined in RFC 5798. Unlike HSRP, it uses Master/Backup terminology, a virtual MAC of 0000.5e00.01XX, multicast 224.0.0.18, and has preemption enabled by default with roughly 3.2 to 3.5 second failover. VRRP also allows the Virtual IP to match a router's real physical interface IP — that router becomes the IP Address Owner and automatically receives priority 255."

---

## 1.5 Deep Dive: VRRP Split-Brain Failure Mechanics

This is the single most-asked VRRP troubleshooting scenario. Understanding it deeply separates a strong candidate from someone who memorized definitions.

### The Critical Distinction: Control Plane vs Data Plane

| Plane | What it carries | What breaks in this scenario |
|---|---|---|
| **Control Plane** | The protocol's own heartbeat messages (VRRP advertisements) | **Broken** — Master's advertisements are blocked |
| **Data Plane** | Actual user/host traffic (the real internet packets) | **Still working** — physical wires, switches, ports all function fine |

> **Key insight:** "The physical path for host data traffic remains open" means cables, switch ports, and frame forwarding all work perfectly. Something has broken only the protocol's ability to talk to itself.

---

### The Architecture of the Failure

```
           [ MASTER ROUTER ]              [ BACKUP ROUTER ]
          Priority: 110 (Active)         Priority: 100 (Listening)
                 │                               │
                 ▼                               ▼
      🚫 VRRP Advertisements Blocked     ✅ VRRP Advertisements NOT Received
      (Dropped by a new inbound ACL)    (Assumes Master is completely dead)
                 │                               │
                 └──────────► [ L2 SWITCH ] ◄────┘
                                  │
                                  ▼
                   [ USER HOSTS / DATA TRAFFIC ]
         Interface cables and unicast forwarding function 100% perfectly.
```

### Step-by-Step Breakdown

```
Step 1: THE BLOCKAGE
        An administrator applies an ACL on the Master's interface that
        blocks VRRP multicast destination 224.0.0.18.

Step 2: THE CONTROL PLANE FAILS
        Master keeps sending VRRP advertisements every 1 second,
        but the switch/ACL drops them.
        Backup stops receiving heartbeats.

Step 3: THE BACKUP PROMOTES ITSELF
        Backup waits the Master Down Timer (≈3.2 sec).
        Assumes Master lost power.
        Promotes itself to Master.
        Starts listening for traffic on the Virtual IP/MAC.

Step 4: SPLIT-BRAIN CHAOS
        Original Master is STILL ALIVE (its physical link is up/up).
        Both physical routers now simultaneously claim the
        same Virtual IP AND Virtual MAC.

Step 5: WHY THE DATA PATH MATTERS
        Cables/ports/MAC tables still work fine, so PCs keep sending
        traffic to the gateway normally.
        The SWITCH gets confused: two different physical ports are
        claiming the same Virtual MAC.
        Switch rapidly flip-flops its MAC table between the two
        router ports → MAC FLAPPING.
        Half the packets go to Router A, half to Router B.
        TCP sessions get torn apart. Network blackout for hosts.
```

### The Three Most Common Root Causes

| Cause | Mechanism |
|---|---|
| **ACL blocking the multicast** | A new or modified inbound ACL filters `224.0.0.18` or IP protocol **112** (VRRP's protocol number) |
| **Layer 2 multicast pruning** | Intermediate switches drop or fail to flood multicast traffic across the VLAN |
| **Authentication mismatch** | If authentication strings are configured and mismatch, the Backup discards the Master's advertisements, assuming it's dead |

> **Interview answer:**
> "It is vital to distinguish between a total physical link failure and a protocol-specific control plane failure. When the data path remains open but the control plane heartbeats are blocked — such as by a misconfigured multicast ACL or transit switch pruning — it triggers a split-brain scenario. Both nodes assume Master status because their link status indicators remain functionally up, leading to severe MAC address table instability and duplicate gateway representation on the local wire."

---

## 1.6 Verification

```cisco
show vrrp          ! Master/Backup states, timers, virtual MAC, VIP
show vrrp brief     ! Clean summary table of active groups per interface
```

---

# SECTION 2 — GLBP (Gateway Load Balancing Protocol)

---

## 2.1 The Limitation GLBP Solves

With both HSRP and VRRP, **only one physical router actively forwards traffic at a time**. Backup/Standby routers sit completely idle — wasted bandwidth and hardware capacity, even on expensive enterprise routers.

**GLBP (Cisco proprietary)** solves this: it provides gateway redundancy AND **actively load-balances traffic across up to 4 routers simultaneously** using a single virtual IP.

---

## 2.2 The Two-Tier Role Architecture

GLBP abandons the simple Active/Standby model entirely.

### AVG — Active Virtual Gateway

- Elected via **priority + highest IP** (one router only).
- **Job:** The "brain"/coordinator. Intercepts every ARP Request from local PCs asking for the gateway.

### AVF — Active Virtual Forwarder

- **Every** router in the GLBP group (up to 4 max) automatically becomes an AVF.
- **Job:** Actually forwards data packets.
- Each AVF gets a **unique Virtual MAC** assigned by the AVG.

```
                    GLBP GROUP
        ┌────────────────────────────────┐
        │  AVG (elected) = traffic cop    │
        │  Intercepts ARP requests        │
        └────────────────────────────────┘
              │           │           │
          AVF 1       AVF 2       AVF 3
       (own vMAC)   (own vMAC)  (own vMAC)
       forwards     forwards    forwards
       packets      packets     packets
```

---

## 2.3 How GLBP Works — Worked Example

**3 routers, shared VIP = `10.1.1.1`**

```
Step 1: Virtual MAC Allocation (by the AVG)
        Router 1 (AVF 1) → 0007.b400.0101
        Router 2 (AVF 2) → 0007.b400.0102
        Router 3 (AVF 3) → 0007.b400.0103

Step 2: PC1 boots, ARPs: "Who owns 10.1.1.1?"
        AVG replies: "Go to 0007.b400.0101 (Router 1)"
        PC1 caches this MAC, sends ALL traffic through Router 1.

Step 3: PC2 boots, ARPs the same question.
        AVG uses ROUND-ROBIN, replies: "Go to 0007.b400.0102 (Router 2)"

Step 4: PC3 boots, asks the same question.
        AVG replies: "Go to 0007.b400.0103 (Router 3)"
```

**Result:** All 3 PCs share the exact same gateway IP, but their traffic is automatically spread across 3 different physical routers — simultaneously.

---

## 2.4 Failure Handling

```
If Router 1 dies:
  AVG (or backup AVG) detects the failure.
  Router 2 steps up and ALSO takes ownership of
  Router 1's virtual MAC (0007.b400.0101), in addition to its own.

  PC1 keeps sending to 0007.b400.0101 — no ARP cache refresh needed.
  Switch simply redirects those frames to Router 2's physical port.
  → Instant, hitless failover.
```

---

## 2.5 HSRP vs VRRP vs GLBP — Master Comparison Table

| Feature | HSRP | VRRP | GLBP |
|---|---|---|---|
| Standard Type | Cisco Proprietary | Open Standard (IEEE) | Cisco Proprietary |
| Active Nodes | 1 Active (others Standby) | 1 Master (others Backup) | **Up to 4 Active Forwarders simultaneously** |
| Load Balancing | No (manual per-VLAN only) | No (manual per-VLAN only) | **True automated dynamic load balancing** |
| Primary Roles | Active / Standby | Master / Backup | AVG (gateway) / AVF (forwarder) |
| Multicast Address | `224.0.0.2` (v1), `224.0.0.102` (v2) | `224.0.0.18` | `224.0.0.102` |
| Preemption Default | Disabled | **Enabled** | Enabled |
| Virtual MAC Format | `0000.0c07.acXX` | `0000.5e00.01XX` | `0007.b400.XXYY` |

> **Memorize this table as a single block — this exact comparison is asked almost verbatim in interviews.**

---

## 2.6 Interview Trap — Asymmetric Routing

**Q:** "GLBP load-balances outbound traffic beautifully, but inbound return traffic from the Internet is hitting a bottleneck on one link. Why isn't GLBP balancing inbound traffic?"

> **Interview answer:**
> "This is a fundamental architectural characteristic of GLBP. GLBP is strictly an outbound default-gateway load-balancing protocol — it operates at the LAN layer by manipulating ARP replies to distribute client traffic across multiple local MAC addresses. GLBP has zero control over data-plane paths chosen by upstream ISPs or WAN routers. Inbound traffic is governed entirely by upstream routing protocols, AS path metrics, or NAT policies. To balance return traffic, you'd need WAN-side techniques like BGP multipathing, ECMP, or symmetric NAT — GLBP cannot manipulate external routing tables."

---

## 2.7 Verification

```cisco
show glbp           ! Full AVG/AVF state, timers, load-balancing metrics
show glbp brief      ! High-level table: interface to active/standby gateway status
```

> **Interview answer (GLBP overview):**
> "GLBP solves the wasted-capacity problem of HSRP and VRRP by allowing up to 4 routers to actively forward traffic simultaneously, instead of leaving backups idle. One router is elected AVG and handles ARP requests, round-robin assigning a unique virtual MAC to each AVF. All hosts share one virtual IP, but their traffic is automatically spread across multiple physical routers. If one AVF fails, a surviving router takes over its virtual MAC for a hitless failover."

---

# SECTION 3 — NTP (Network Time Protocol)

---

## 3.1 Why NTP Matters

**Mental model:** A security breach touches 5 devices — firewall, switch, router, server, database. Each logs the event with its own local clock. If the firewall says 10:00 AM, the switch says 10:05 AM, and the server says 9:55 AM, **reconstructing the attack timeline becomes impossible**.

**NTP synchronizes every device on the network to the same time, down to the millisecond.**

---

## 3.2 The Stratum Hierarchy

The Stratum number = distance from the absolute reference clock. **Lower number = more accurate/authoritative.**

| Stratum | Description |
|---|---|
| **0** | Ultimate source — Atomic clocks, Hydrogen masers, GPS satellite clocks. Not on the network; physically wired via serial cable to a server. |
| **1** | Primary servers — directly connected to a Stratum 0 device. Act as the Internet's primary time servers. |
| **2** | Secondary servers — sync with a Stratum 1 server over the network. Tiny added latency. |
| **3** | Your enterprise devices — routers, switches, local servers sync with Stratum 2. |

```
Stratum 0 (Atomic/GPS)
    │ (direct cable)
Stratum 1 (Primary servers)
    │ (network)
Stratum 2 (Secondary servers)
    │ (network)
Stratum 3 (Your enterprise routers/switches) ← you are usually here
```

> **Interview limit rule:** Maximum NTP stratum is **15**. A device showing **Stratum 16** is considered completely unsynchronized and untrusted — other devices reject its time advertisements entirely.

---

## 3.3 How NTP Operates Mechanically

NTP runs over **UDP, port 123**.

**The problem:** Network latency means a timestamp received is already "old" by the time it arrives. NTP solves this using **4 timestamps**:

```
[ Router (Client) ]                          [ NTP Server ]
      │                                            │
   ( T1 ) ──── Request sent ──────────────────►    │
      │                                         ( T2 ) received
      │                                            │
      │  ◄──── Response sent ────────────────── ( T3 )
   ( T4 ) received
```

| Timestamp | Meaning |
|---|---|
| **T1** | Time the client transmits the request |
| **T2** | Time the server receives the request |
| **T3** | Time the server sends the response |
| **T4** | Time the client receives the response |

**Formula:**

```
Round-Trip Delay = (T4 - T1) - (T3 - T2)
```

The router uses this to cancel out network delay and set its clock accurately.

---

## 3.4 Configuration

```cisco
! Pull time from an upstream NTP server
Router(config)# ntp server 8.8.8.8

! Act as the master clock source for the LAN at a declared stratum
! ("If I lose Internet, treat me as a reliable Stratum 3 reference for the LAN")
CoreRouter(config)# ntp master 3
```

---

## 3.5 Interview Trap — System Clock vs Hardware Clock

**Q:** "I configured `ntp server 10.1.1.1`. `show ntp status` shows perfect sync. But after a reboot, the clock reverts to a date from 5 years ago. Why?"

> **Interview answer:**
> "Cisco devices maintain two separate clock mechanisms: the Software/System Clock, managed by the IOS kernel in RAM, and the Hardware Clock/Calendar — a physical, battery-backed chip on the motherboard. NTP only synchronizes the software clock in volatile RAM. On reboot, RAM is purged and the device reloads its boot time from the un-synchronized hardware clock. To fix this permanently, configure `ntp update-calendar`, which periodically writes the accurate NTP-synced time into the hardware calendar chip, preserving sync across reboots."

```cisco
Router(config)# ntp update-calendar
```

---

## 3.6 Verification

```cisco
show clock              ! Current local software time
show ntp status          ! Sync status, current stratum, master reference IP
show ntp associations    ! Table of upstream servers — delay, offset, jitter
```

> **Interview answer (NTP overview):**
> "NTP synchronizes device clocks across the network using a hierarchical Stratum system — Stratum 0 atomic/GPS sources down through Stratum 1, 2, and enterprise devices typically at Stratum 3 or higher. It runs over UDP port 123 and uses four timestamps to mathematically cancel out network latency when calculating the correct time. This precision matters most for correlating logs across multiple devices during security investigations or troubleshooting."

---

# SECTION 4 — Syslog (System Logging)

---

## 4.1 Why Syslog Exists

A router, switch, or firewall generates thousands of small events per second — port up/down, user login, OSPF neighbor drop. No administrator can watch a console all day.

**Syslog is the standard protocol that lets devices generate and forward system event messages to a centralized server** for monitoring, auditing, and troubleshooting.

---

## 4.2 Local Logging vs Remote Logging

### Local Logging — The Buffer

By default, devices store logs in a small **Logging Buffer** in RAM.

> ⚠️ **The Volatility Trap:** Because the buffer lives in RAM, a reboot, power loss, or buffer overflow **erases those logs forever** — a major compliance and security risk.

### Remote Logging — The Syslog Server

The device packages each event as a **UDP packet on port 514** and sends it to a centralized Syslog server (hardened Linux/Windows box), where logs are written to disk for **permanent** storage.

| | Local Buffer (RAM) | Remote Syslog Server |
|---|---|---|
| Survives reboot? | No | Yes |
| Survives power loss? | No | Yes |
| Storage limit | Small, fixed buffer size | Disk-based, scalable |
| Protocol/Port | N/A (internal) | UDP 514 |
| Use for compliance/audit? | Risky | Required |

---

## 4.3 Anatomy of a Syslog Message

```
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0, changed state to up
```

| Component | Example | Meaning |
|---|---|---|
| **Facility** | `LINEPROTO` | The subsystem that generated the message (e.g., SEC=security, OSPF, IP) |
| **Severity** | `5` | A single digit 0–7. **Lower = more dangerous.** |
| **Description** | `UPDOWN: Line protocol...` | Human-readable explanation of the event |

---

## 4.4 The 8 Syslog Severity Levels (Memorize Cold)

> **Mnemonic:** "**E**very **A**lleged **C**riminal **E**lephant **W**ill **N**ested **I**n **D**irt" → Emergency, Alert, Critical, Error, Warning, Notification, Informational, Debugging

| Level | Name | Severity | Example |
|---|---|---|---|
| **0** | Emergency | Highest | System totally unstable / hardware failure |
| **1** | Alert | High | Immediate action needed — e.g., all WAN links down |
| **2** | Critical | High | Critical condition — e.g., core temperature exceeded |
| **3** | Error | Medium | Non-fatal error — e.g., interface failed to back up |
| **4** | Warning | Medium | Warning event — e.g., redundant power supply failed |
| **5** | Notification | Normal | Normal operational change — e.g., UPDOWN interface state |
| **6** | Informational | Normal | Standard info — e.g., user authenticated via SSH |
| **7** | Debugging | Lowest | Maximum verbosity — only for active troubleshooting |

```
LOWER NUMBER = MORE SEVERE
0 ────────────────────────────────────────► 7
Emergency                              Debugging
(catastrophic)                        (routine detail)
```

---

## 4.5 Essential Configuration Commands

```cisco
! Send all syslog packets to a remote server
Router(config)# logging host 192.168.10.5

! Only forward messages at severity 4 (Warning) or LOWER (more severe: 0,1,2,3)
! Keeps informational noise off the network
Router(config)# logging trap 4

! Timestamp every log message down to the millisecond
! Critical for aligning logs with your NTP master clock
Router(config)# service timestamps log datetime msec
```

> **Note the inverse relationship:** `logging trap 4` sends levels **0 through 4** (everything *more severe than or equal to* Warning) — not just level 4 alone.

---

## 4.6 Interview Trap — `logging synchronous`

**Q:** "I'm SSH'd into a router configuring an OSPF interface. A Syslog message keeps popping up mid-typing and tears my command in half. How do I fix this without disabling logging?"

> **Interview answer:**
> "This happens because Cisco IOS pushes log messages to the terminal asynchronously, interrupting whatever the engineer is typing. The fix is to enter line configuration mode — `line vty 0 4` or `line console 0` — and run `logging synchronous`. This doesn't suppress logs; it tells IOS to wait until the current log finishes printing, then cleanly reprint your partially-typed command on a new line beneath it."

```cisco
Router(config)# line vty 0 4
Router(config-line)# logging synchronous
```

---

## 4.7 Verification

```cisco
show logging    ! Buffer size, remote logging status, all historical buffered logs
```

> **Interview answer (Syslog overview):**
> "Syslog lets network devices forward system event messages to a centralized server instead of relying on local volatile RAM buffers that get wiped on reboot. Each message has a Facility showing the subsystem, a Severity from 0 to 7 where lower is more critical, and a human-readable description. Remote logging uses UDP port 514, and `logging trap` controls which severity threshold gets forwarded off the device."

---

# SECTION 5 — Module 4 Rapid-Fire Q&A

---

> Practice: Read question, cover answer, respond in one sentence. Uncover and check.

---

**Q: Is VRRP Cisco-proprietary or an open standard?**
A: Open standard — IEEE, defined in RFC 5798.

**Q: What is the VRRP equivalent of HSRP's "Active Router"?**
A: Master Router.

**Q: What is the VRRP equivalent of HSRP's "Standby Router"?**
A: Backup Router.

**Q: What is the VRRP virtual MAC format?**
A: 0000.5e00.01XX, where XX is the VRRP Group ID in hex.

**Q: What multicast address does VRRP use?**
A: 224.0.0.18.

**Q: How often does a VRRP Master send advertisements by default?**
A: Every 1 second.

**Q: What is the Master Down Timer formula?**
A: (3 × Advertisement Interval) + Skew Time — roughly 3.2–3.5 seconds by default.

**Q: Is VRRP preemption enabled or disabled by default?**
A: Enabled by default — the opposite of HSRP.

**Q: What is an IP Address Owner in VRRP?**
A: A router where the Virtual IP equals its own physical interface IP; it automatically gets priority 255 and always wins the Master election.

**Q: What causes a VRRP split-brain condition?**
A: The control plane (VRRP advertisements) is blocked while the data plane (physical links) stays up — caused by an ACL blocking 224.0.0.18 or protocol 112, Layer 2 multicast pruning, or an authentication mismatch.

**Q: What happens to the switch's MAC table during a VRRP split-brain?**
A: It flip-flops between the two router ports claiming the same virtual MAC — called MAC flapping — causing packet loss and torn TCP sessions.

**Q: What limitation do both HSRP and VRRP share?**
A: Only one router actively forwards traffic at a time; the backup sits idle.

**Q: What does GLBP stand for, and what problem does it solve?**
A: Gateway Load Balancing Protocol — it allows up to 4 routers to actively forward traffic simultaneously instead of leaving backups idle.

**Q: What is an AVG in GLBP?**
A: Active Virtual Gateway — the elected router that intercepts ARP requests and assigns virtual MACs to each AVF.

**Q: What is an AVF in GLBP?**
A: Active Virtual Forwarder — every router in the group (up to 4) that actually forwards data packets using its assigned virtual MAC.

**Q: How does GLBP distribute traffic across multiple routers?**
A: The AVG replies to ARP requests with different AVF virtual MACs using a round-robin algorithm, spreading host traffic across multiple physical routers.

**Q: What happens if one GLBP AVF fails?**
A: A surviving AVF takes over the failed router's virtual MAC in addition to its own, providing hitless failover.

**Q: Why can't GLBP load-balance inbound/return traffic from the Internet?**
A: GLBP only manipulates ARP replies on the LAN side; it has no control over upstream routing decisions made by ISPs or WAN routers.

**Q: What multicast address does GLBP use?**
A: 224.0.0.102.

**Q: Why does NTP matter for network security?**
A: It synchronizes clocks across all devices so logs from different devices can be correlated accurately during an incident investigation.

**Q: What is a Stratum 0 device?**
A: The ultimate time source — atomic clocks, GPS satellites — not directly on the network, physically wired via serial cable.

**Q: What stratum level is considered unsynchronized/untrusted?**
A: Stratum 16. Maximum valid stratum is 15.

**Q: What port and protocol does NTP use?**
A: UDP port 123.

**Q: What are the 4 NTP timestamps, and what is the Round-Trip Delay formula?**
A: T1 (client sends), T2 (server receives), T3 (server sends reply), T4 (client receives). Round-Trip Delay = (T4 − T1) − (T3 − T2).

**Q: Why does a router's clock revert after a reboot even with NTP configured?**
A: NTP only syncs the software/system clock in RAM. The hardware clock/calendar chip is separate and not auto-updated; use `ntp update-calendar` to persist it.

**Q: What does `ntp master 3` do?**
A: Tells the router to act as a Stratum 3 reference clock for the LAN if it loses upstream connectivity.

**Q: Why is Syslog needed instead of just watching the console?**
A: Devices generate too many events to monitor manually; Syslog centralizes and automates log collection for monitoring and auditing.

**Q: What's the risk of relying only on the local logging buffer?**
A: It lives in volatile RAM — a reboot, power loss, or buffer overflow erases the logs permanently.

**Q: What port and protocol does remote Syslog use?**
A: UDP port 514.

**Q: What are the 3 parts of a Syslog message?**
A: Facility (subsystem), Severity (0–7), and Description (human-readable text).

**Q: In Syslog severity, does a lower or higher number mean more severe?**
A: Lower number = more severe. 0 is Emergency (worst); 7 is Debugging (routine).

**Q: What does `logging trap 4` actually send?**
A: All messages at severity 4 (Warning) and lower — meaning levels 0, 1, 2, 3, and 4 (more severe and equal).

**Q: What does `service timestamps log datetime msec` do?**
A: Adds a millisecond-precision timestamp to every log message, useful for aligning logs with the NTP master clock.

**Q: What does `logging synchronous` fix?**
A: Prevents Syslog messages from interrupting and garbling commands being typed in the CLI; reprints the partial command cleanly after the log message.

**Q: Where do you configure `logging synchronous`?**
A: Under line configuration mode — `line vty 0 4` or `line console 0`.

---

# SECTION 6 — Module 4 Trap Questions

---

| Trap Question | Strong Answer |
|---|---|
| Both HSRP and VRRP only forward via one router. Why does that matter in real networks? | It wastes the backup router's bandwidth and hardware capacity, which sit idle except during failover. This is the exact gap GLBP fills with simultaneous multi-router forwarding. |
| Why is VRRP's default failover faster than HSRP's? | VRRP advertises every 1 second with a ~3.2 second Master Down Timer; HSRP defaults to a 10-second hold timer with 3-second Hellos. VRRP is roughly 3x faster out of the box. |
| Can two VRRP routers both legitimately have priority 255? | No — only the IP Address Owner can hold priority 255, since it's tied to owning the physical interface IP matching the VIP. Any other router manually set to 255 would create an ambiguous tie. |
| If VRRP preemption is undesirable in your design, what must you do? | Explicitly disable it, since VRRP has preemption ON by default — the opposite of HSRP, where you'd have to explicitly enable it. |
| A VRRP split-brain is happening but `show interface` shows both router links as up/up. What does that tell you? | The failure is in the control plane (VRRP advertisements being blocked), not the physical/data plane. Look at ACLs, multicast pruning, or authentication mismatches — not cabling. |
| In a GLBP split-brain-like scenario, can two AVFs both forward for the SAME virtual MAC? | No — each AVF has a uniquely assigned virtual MAC from the AVG. The architecture itself prevents MAC collision, unlike VRRP/HSRP where both routers could claim the SAME single virtual MAC during split-brain. |
| Why doesn't GLBP help with inbound traffic bottlenecks from the WAN? | GLBP only controls ARP replies on the local LAN segment. Inbound routing decisions are made entirely by upstream ISP/WAN devices, which GLBP has no visibility into or control over. |
| If a Stratum 1 server loses its connection to Stratum 0, does it just stop working? | It can continue serving time based on its last known accurate reading for a period, but its accuracy degrades over time without resync; if isolated long enough it could be flagged untrusted depending on configuration. |
| Why is a Stratum 16 device rejected by others, instead of just being "less accurate"? | Stratum 16 explicitly signals "unsynchronized" in the NTP protocol itself — it's not a measure of slightly worse accuracy, it's the defined signal that this source cannot be trusted at all. |
| If `logging trap 4` is set, will severity 6 (Informational) messages be sent to the Syslog server? | No. `logging trap 4` only sends severities 0 through 4. Level 6 is less severe than the threshold and is filtered out from remote forwarding (though it may still appear in the local buffer). |
| Does `logging synchronous` reduce how many logs you see? | No — it doesn't filter or suppress anything. It only changes how cleanly logs are displayed relative to what you're typing in the CLI. |
| Why must NTP timestamps and Syslog timestamps both be precise for security investigations? | Without millisecond-accurate, synchronized timestamps across every device, you cannot reliably reconstruct the order of events during an incident — gaps or mismatches make root cause analysis unreliable. |

---

# SECTION 7 — Module 4 Completion Checklist

---

Tick each item once you can explain it in interview style without notes.

### VRRP
- [ ] VRRP is an open standard (RFC 5798), not Cisco-proprietary
- [ ] Full HSRP-to-VRRP terminology mapping table — Master/Backup, virtual MAC, multicast, preemption default
- [ ] Master Down Timer formula and approximate failover time
- [ ] IP Address Owner concept and automatic priority 255
- [ ] Split-brain failure mechanics — control plane vs data plane distinction
- [ ] Full step-by-step split-brain breakdown and MAC flapping explanation
- [ ] 3 root causes of split-brain — ACL, L2 multicast pruning, authentication mismatch
- [ ] Verification commands — show vrrp, show vrrp brief

### GLBP
- [ ] The shared HSRP/VRRP limitation GLBP solves — idle backup routers
- [ ] AVG role — election, ARP interception
- [ ] AVF role — up to 4, unique virtual MAC each
- [ ] Round-robin ARP reply mechanism — worked example with 3 routers
- [ ] Failure handling — surviving AVF absorbs the failed router's virtual MAC
- [ ] Full HSRP vs VRRP vs GLBP comparison table — memorized
- [ ] Asymmetric routing limitation — why GLBP can't balance inbound WAN traffic
- [ ] Verification commands — show glbp, show glbp brief

### NTP
- [ ] Why NTP matters — log correlation during security investigations
- [ ] Stratum hierarchy — 0 through 3+ and what each level represents
- [ ] Stratum 15 max, Stratum 16 = untrusted/unsynchronized
- [ ] UDP port 123
- [ ] 4-timestamp mechanism — T1, T2, T3, T4 and the Round-Trip Delay formula
- [ ] Configuration — ntp server, ntp master
- [ ] System clock vs hardware clock trap — why time reverts after reboot
- [ ] ntp update-calendar — what it fixes
- [ ] Verification commands — show clock, show ntp status, show ntp associations

### Syslog
- [ ] Why centralized logging is needed over manual console watching
- [ ] Local logging buffer — volatile, lost on reboot/power loss/overflow
- [ ] Remote logging — UDP port 514, permanent disk storage
- [ ] 3 parts of a syslog message — Facility, Severity, Description
- [ ] All 8 severity levels in order, with the mnemonic
- [ ] Lower severity number = more critical (inverse of what feels intuitive)
- [ ] Configuration — logging host, logging trap, service timestamps log datetime msec
- [ ] logging trap threshold behavior — sends that level AND everything more severe
- [ ] logging synchronous trap — what it fixes, where it's configured
- [ ] Verification command — show logging

---

## Module 4 Labs to Complete

- [ ] Configure VRRP between two routers sharing a LAN segment; verify with `show vrrp brief`
- [ ] Configure one router as the IP Address Owner (VIP = its physical IP) and confirm priority 255 automatically
- [ ] Simulate a VRRP split-brain by applying an ACL that blocks 224.0.0.18 on the Master's interface; observe both routers claim Master
- [ ] Configure GLBP across 3 routers sharing one VIP; verify with `show glbp brief` that each gets a unique virtual MAC
- [ ] Generate ARP requests from multiple hosts and confirm round-robin AVF assignment
- [ ] Fail one GLBP router and confirm a surviving AVF absorbs its virtual MAC
- [ ] Configure `ntp server` on a router and verify sync with `show ntp status`
- [ ] Reboot a router without `ntp update-calendar` configured and observe the clock revert; then configure it and confirm persistence
- [ ] Configure `logging host` to a Syslog server and confirm messages arrive on UDP port 514
- [ ] Configure `logging trap 4` and confirm only severity 0–4 messages are forwarded remotely
- [ ] Trigger the CLI interruption issue, then fix it with `logging synchronous` under `line vty 0 4`

### Useful Show Commands for Module 4

```cisco
show vrrp                    # VRRP Master/Backup state, timers, VIP
show vrrp brief              # Quick VRRP summary table
show glbp                    # Full AVG/AVF state and metrics
show glbp brief              # Quick GLBP summary table
show clock                   # Current device software time
show ntp status               # NTP sync status and stratum
show ntp associations         # Upstream NTP server tracking table
show logging                 # Buffer contents, remote logging config
```

---

## One-Page Revision Summary

```
VRRP: Open standard (RFC 5798). Master/Backup (not Active/Standby).
  vMAC: 0000.5e00.01XX | Multicast: 224.0.0.18 | Advertise every 1 sec
  Master Down Timer = (3 × Advertise Interval) + Skew ≈ 3.2-3.5 sec
  PREEMPTION: ON by default (opposite of HSRP).
  IP ADDRESS OWNER: VIP = real interface IP → auto priority 255.
  SPLIT-BRAIN: Control plane (advertisements) blocked, data plane (links) still up.
    Causes: ACL blocking 224.0.0.18/protocol 112 | L2 multicast pruning | auth mismatch
    Result: Both routers claim Master → switch MAC FLAPPING → packet loss/TCP breaks.

GLBP: Cisco proprietary. Solves HSRP/VRRP's "idle backup" waste.
  AVG = elected boss, handles ARP requests, assigns vMACs (round-robin)
  AVF = up to 4 routers, each gets own vMAC, all actively forward traffic
  Failure: surviving AVF absorbs failed router's vMAC → hitless failover
  LIMITATION: only balances OUTBOUND/LAN traffic via ARP. Can't touch inbound WAN routing.
  Multicast: 224.0.0.102

HSRP vs VRRP vs GLBP:
  Standard:    Cisco | Open(IEEE) | Cisco
  Active:      1     | 1          | up to 4
  Load-balance: No   | No         | Yes (true dynamic)
  Roles: Active/Standby | Master/Backup | AVG/AVF
  Preempt: Off | On | On
  vMAC: 0000.0c07.acXX | 0000.5e00.01XX | 0007.b400.XXYY

NTP: Synchronizes clocks for log correlation. UDP port 123.
  STRATUM: 0=atomic/GPS (not networked) → 1=primary → 2=secondary → 3=enterprise
  Max valid stratum = 15. Stratum 16 = untrusted/unsynchronized.
  4 TIMESTAMPS: T1(client send) T2(server recv) T3(server send) T4(client recv)
  Round-Trip Delay = (T4-T1) - (T3-T2)
  Config: ntp server <ip> | ntp master <stratum>
  TRAP: NTP syncs SOFTWARE clock (RAM) only. Hardware clock chip needs
        `ntp update-calendar` to persist time across reboot.

SYSLOG: Centralized event logging. Local buffer = RAM (volatile, lost on reboot).
  Remote logging: UDP port 514, permanent disk storage on Syslog server.
  MESSAGE FORMAT: %FACILITY-SEVERITY-MNEMONIC: Description
  8 SEVERITY LEVELS (lower = worse): 
    0=Emergency 1=Alert 2=Critical 3=Error 4=Warning 5=Notification 6=Informational 7=Debugging
    Mnemonic: Every Alleged Criminal Elephant Will Nested In Dirt
  Config: logging host <ip> | logging trap <level> | service timestamps log datetime msec
  `logging trap 4` = sends levels 0-4 (that level AND more severe)
  TRAP: `logging synchronous` (under line vty/console) fixes CLI text being
        interrupted by log messages — does NOT suppress or filter logs.
```

---

*End of Module 4 Notes | Cisco G4 Interview Preparation*
*Covers: VRRP (incl. Split-Brain Deep Dive), GLBP, HSRP/VRRP/GLBP Comparison, NTP (Stratum & Timestamp Mechanics), Syslog (Severity Levels & Configuration)*
