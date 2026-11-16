# CCNA Learnings

A structured collection of Cisco networking study notes and hands-on lab checklists. The material is organized into four modules, progressing from networking fundamentals to switching, routing, and IP services. Notes are written for revision and Cisco networking interview preparation.

## Contents

| Module | Topics | Notes |
|---|---|---|
| 1 — Network Fundamentals | OSI and TCP/IP models, IPv4/IPv6 addressing, subnetting, core protocols, DNS, DHCP, common ports, NAT/PAT, and network domains | [Open Module 1 notes](module1/notes_1.md) |
| 2 — Network Access: Switching and VLANs | Switch forwarding and CAM tables, VLANs, access and trunk ports, 802.1Q, inter-VLAN routing, STP/RSTP, PortFast, BPDU Guard, and EtherChannel/LACP | [Open Module 2 notes](module2/notes_2.md) |
| 3 — IP Connectivity: Routing | Routing tables, longest-prefix match, static and default routes, floating static routes, OSPFv2, DR/BDR elections, and HSRP | [Open Module 3 notes](module3/notes_3.md) |
| 4 — IP Services | VRRP, GLBP, NTP, and Syslog | [Open Module 4 notes](module4/notes_4.md) |

## How to study

1. Read a module once to understand the concepts and examples.
2. Use the rapid-fire questions and trap questions to check your recall; try answering before revealing the notes.
3. Work through the module checklist and mark topics you can explain without help.
4. Build the suggested topologies in a Cisco-compatible lab environment, such as Packet Tracer, CML, or GNS3, where available.
5. Use the verification commands in each module to inspect device state and confirm your configuration.
6. Revisit the one-page revision summary before a review session or interview.

## Repository structure

```text
.
├── module1/
│   └── notes_1.md
├── module2/
│   └── notes_2.md
├── module3/
│   └── notes_3.md
└── module4/
    └── notes_4.md
```

## Requirements

The notes are Markdown files and can be read directly on GitHub or with any Markdown viewer. No software installation is required to read them. Cisco device access or a network simulator is useful for completing the optional labs.

## Contributions

Corrections, clarifications, and study improvements are welcome. When contributing, keep explanations accurate, include relevant verification steps where useful, and preserve the module structure.

## Disclaimer

This repository is an independent set of study notes. It is not affiliated with or endorsed by Cisco. Use vendor documentation and current exam objectives to verify details and prepare for certification exams.
