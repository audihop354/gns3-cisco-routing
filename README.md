# GNS3 Multi-AS Network Simulation

> Enterprise network simulation with OSPF, RIPv2, BGP, and route redistribution

---

## Overview

This project simulates a **multi-site enterprise network** using GNS3, implementing three autonomous systems (AS100, AS200, AS300) interconnected via **BGP**. Each region runs a different Interior Gateway Protocol (IGP):

| Region | Protocol | Routers | Type |
|--------|----------|---------|------|
| **AS100** | OSPF | R1, R2, R3 | Core / Head Office |
| **AS200** | RIPv2 | R4, R7 | Branch Office A |
| **AS300** | RIPv2 | R5, R6 | Branch Office B |

**Goal:** Full end-to-end connectivity between all PCs across all three autonomous systems.

---

## Network Topology

```
                          ┌────────── AS100 (OSPF) ──────────┐
                          │                                    │
  PC1 ── Switch1 ── R1 ──┤── R2 ──────────────────┐          │
     200.200.21.0/28     │    │                    │          │
                          │    R3 ──────────────────┤          │
                          │    │                    │          │
                          └────┼────────────────────┘          │
                               │                               │
              ┌────────────────┼────────────────┐              │
              │                │                │              │
        ┌─────▼─────┐   ┌─────▼─────┐          │              │
        │    R4     │   │    R5     │          │              │
        │  AS200    │   │  AS300    │          │              │
        │  RIPv2    │   │  RIPv2    │          │              │
        └─────┬─────┘   └─────┬─────┘          │              │
              │               │                │              │
              R7              R6               │              │
              │               │                │              │
          Switch3          Switch2             │              │
              │               │                │              │
             PC3             PC2               │              │
      220.220.21.0/28  210.210.21.0/28        │              │
                                               │              │
        BGP Peering:  AS100 ◄──► AS200 ◄──► AS300            │
        eBGP: R2 ↔ R5 (AS100↔AS300), R3 ↔ R4 (AS100↔AS200)   │
        iBGP: R2 ↔ R3 (within AS100)                          │
```

---

## Hardware & Software

| Component | Specification |
|-----------|--------------|
| **Router** | Cisco 7200 (c7200) — 7 units |
| **Switch** | GNS3 Ethernet Switch — 3 units |
| **PC** | GNS3 VPCS — 3 units |
| **Router OS** | Cisco IOS 12.4 (c7200-adventerprisek9-mz.124-24.T5) |
| **Simulator** | GNS3 v2.2.51 |
| **Emulator** | Dynamips |

---

## IP Addressing Plan

### AS100 — OSPF Area 100

| Device | Interface | IP Address | Subnet | Mask |
|--------|-----------|------------|--------|------|
| PC1 | e0 | 200.200.21.2 | /28 | 255.255.255.240 |
| R1 | f0/0 | 200.200.21.1 | /28 | 255.255.255.240 |
| R1 | f1/0 | 100.100.21.1 | /30 | 255.255.255.252 |
| R1 | f1/1 | 110.110.21.1 | /30 | 255.255.255.252 |
| R2 | f0/0 | 100.100.21.2 | /30 | 255.255.255.252 |
| R2 | f1/0 | 120.120.21.1 | /30 | 255.255.255.252 |
| R3 | f0/0 | 110.110.21.2 | /30 | 255.255.255.252 |
| R3 | f1/0 | 160.160.21.1 | /30 | 255.255.255.252 |

### AS300 — RIPv2

| Device | Interface | IP Address | Subnet | Mask |
|--------|-----------|------------|--------|------|
| PC2 | e0 | 210.210.21.2 | /28 | 255.255.255.240 |
| R6 | f0/0 | 210.210.21.1 | /28 | 255.255.255.240 |
| R6 | f1/0 | 130.130.21.2 | /30 | 255.255.255.252 |
| R5 | f0/0 | 120.120.21.2 | /30 | 255.255.255.252 |
| R5 | f1/0 | 130.130.21.1 | /30 | 255.255.255.252 |

### AS200 — RIPv2

| Device | Interface | IP Address | Subnet | Mask |
|--------|-----------|------------|--------|------|
| PC3 | e0 | 220.220.21.2 | /28 | 255.255.255.240 |
| R7 | f0/0 | 220.220.21.1 | /28 | 255.255.255.240 |
| R7 | f1/0 | 140.140.21.2 | /30 | 255.255.255.252 |
| R4 | f0/0 | 160.160.21.2 | /30 | 255.255.255.252 |
| R4 | f1/0 | 140.140.21.1 | /30 | 255.255.255.252 |

---

## Implementation Details

### Phase 1 — IP Configuration (Static)

All router interfaces and PCs configured with static IPs per the addressing plan.
- `/30` subnets used for point-to-point router links (conserves IP space)
- `/28` subnets used for LAN segments (supports up to 14 hosts each)

### Phase 2 — IGP Deployment

**OSPF (AS100):**
- Single-area OSPF (Area 100) on R1, R2, R3
- All directly connected networks advertised via `network` statements
- Wildcard mask `0.0.0.63` used for route advertisement

**RIPv2 (AS200 & AS300):**
- RIPv2 on R4, R7 (AS200) and R5, R6 (AS300)
- `no auto-summary` to prevent route summarization across major network boundaries
- Version 2 explicitly set for classless routing support

### Phase 3 — BGP & Inter-AS Routing

**eBGP Peering:**
- R2 (AS100) ↔ R5 (AS300) — connects head office to branch B
- R3 (AS100) ↔ R4 (AS200) — connects head office to branch A

**iBGP Peering:**
- R2 ↔ R3 — full mesh within AS100

**Network Advertisement:**
- Each border router advertises its local networks via BGP `network` statements
- Proper subnet masks specified for classless advertisement

### Phase 4 — Route Redistribution

Two-way redistribution between IGP and BGP on border routers:

| Router | Configuration | Purpose |
|--------|--------------|---------|
| R2 | `redistribute bgp 100 subnets` in OSPF | Inject AS300 routes into OSPF domain |
| R3 | `redistribute bgp 100 subnets` in OSPF | Inject AS200 routes into OSPF domain |
| R5 | `redistribute bgp 300 metric 3` in RIP | Inject AS100 routes into AS300 RIP domain |
| R4 | `redistribute bgp 200 metric 3` in RIP | Inject AS100 routes into AS200 RIP domain |

---

## Verification & Results

### Connectivity Test

| Source | Destination | Protocol | Result |
|--------|-------------|----------|--------|
| PC1 (AS100) | PC2 (AS300) | ICMP | ✅ Success |
| PC1 (AS100) | PC3 (AS200) | ICMP | ✅ Success |
| PC2 (AS300) | PC1 (AS100) | ICMP | ✅ Success |
| PC2 (AS300) | PC3 (AS200) | ICMP | ✅ Success |
| PC3 (AS200) | PC1 (AS100) | ICMP | ✅ Success |
| PC3 (AS200) | PC2 (AS300) | ICMP | ✅ Success |

### Routing Table Verification

All routers show complete routing tables with routes learned from all three autonomous systems:
- **R1, R2, R3:** AS300 and AS200 routes appear as OSPF external (E2) routes
- **R4, R7:** AS100 and AS300 routes appear as RIP routes (redistributed from BGP)
- **R5, R6:** AS100 and AS200 routes appear as RIP routes (redistributed from BGP)
- **R6 routing table confirms:** Route to `220.220.21.0/28` (AS200) learned via RIPv2 from R5

> Full routing table outputs available in the [detailed report](Report.pdf).

---

## Project Structure

```
.
├── README.md
├── Report.pdf                           # Full 15-page documentation
├── topology.gns3                        # GNS3 project file
├── .gitignore
└── configs/
    ├── AS100-OSPF/
    │   ├── R1_startup-config.cfg
    │   ├── R2_startup-config.cfg
    │   └── R3_startup-config.cfg
    ├── AS200-RIPv2/
    │   ├── R4_startup-config.cfg
    │   └── R7_startup-config.cfg
    ├── AS300-RIPv2/
    │   ├── R5_startup-config.cfg
    │   └── R6_startup-config.cfg
    └── PCs/
        ├── PC1.vpc
        ├── PC2.vpc
        └── PC3.vpc
```

---

## How to Reproduce

1. Install [GNS3](https://www.gns3.com/) with Dynamips support
2. Clone this repository and open `topology.gns3`
3. Provide Cisco IOS image: `c7200-adventerprisek9-mz.124-24.T5.image` (not included — licensing)
4. Start all devices — routers will auto-load startup configurations
5. Verify connectivity: `ping` from any PC to the other two PCs

> **Note:** The Cisco IOS image is proprietary software and is **not included** in this repository. You must supply your own legally obtained image.

---

## Skills Demonstrated

- **IP Subnetting** — VLSM design with `/28` and `/30` subnets
- **Cisco IOS CLI** — Router configuration, interface management, troubleshooting
- **OSPF** — Single-area deployment, network advertisement, wildcard masks
- **RIPv2** — Classless routing, auto-summary control, metric manipulation
- **BGP** — eBGP and iBGP peering, network advertisement, AS path management
- **Route Redistribution** — Two-way redistribution between OSPF/RIP and BGP
- **Network Troubleshooting** — Routing table inspection, ping verification, adjacency checks
- **GNS3 / Dynamips** — Network simulation and emulation

---

