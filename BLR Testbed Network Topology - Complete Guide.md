# BLR Testbed Network Topology - Complete Guide

## Table of Contents
1. [Overview - What is the BLR Testbed?](#1-overview)
2. [Physical Devices and Their Roles](#2-physical-devices)
3. [Network Diagram](#3-network-diagram)
4. [IP Address Reference](#4-ip-address-reference)
5. [How Devices Connect to Each Other](#5-device-connections)
6. [The Headnode (Traffic Generator Server)](#6-headnode)
7. [Testing Traffic Paths](#7-traffic-paths)
8. [Pseudowire (L2VPN) Testing](#8-pseudowire-testing)
9. [Credentials Reference](#9-credentials)
10. [Common Commands](#10-common-commands)

---

## 1. Overview - What is the BLR Testbed? <a name="1-overview"></a>

The **BLR Testbed** is a network testing environment located in Bangalore (BLR). It consists of:

- **Two separate PODs (network units)**: BLR1 and BLR2
- **Each POD has**: 1 Leaf switch, 1-2 Spine switches, 1 A10 switch
- **Shared resources**: Headnode (server), NCS-5501 aggregation switches, Spirent TestCenter

### What is a POD?
Think of a POD as a complete mini-network with:
- **Leaf switches**: Connect to customer devices (access layer)
- **Spine switches**: Connect leaves together and to the outside world (core layer)
- **A10 switches**: Special purpose aggregation switches

```
Simple POD Structure:

                    +------------------+
                    |  External World  |
                    +--------+---------+
                             |
            +----------------+----------------+
            |                                 |
      +-----------+                    +-----------+
      | SPINE 01  |--------------------| SPINE 02  |
      +-----------+                    +-----------+
            |                                 |
            +----------------+----------------+
                             |
                      +-----------+
                      |  LEAF 01  |
                      +-----------+
                             |
                     Customer Devices
```

---

## 2. Physical Devices and Their Roles <a name="2-physical-devices"></a>

### BLR1 POD Devices

| Device Name | Role | What it Does |
|-------------|------|--------------|
| **BLR1LEAF01** | Access Leaf | Connects to customer-facing ports. Handles subscriber traffic (PPPoE, L2VPN, etc.) |
| **BLR1SPINE01** | Spine/Core | Routes traffic between leaves and to external networks. Runs BGP, ISIS |
| **BLR1SPINE02** | Spine/Core | Second spine for redundancy |
| **BLR1A10SWITCH01** | Aggregation | Special aggregation switch, connects to spines |

### BLR2 POD Devices

| Device Name | Role | What it Does |
|-------------|------|--------------|
| **BLR2LEAF01** | Access Leaf | Connects to customer-facing ports |
| **BLR2LEAF02** | Access Leaf | Second leaf for redundancy |
| **BLR2SPINE01** | Spine/Core | Routes traffic |
| **BLR2SPINE02** | Spine/Core | Second spine for redundancy |

### Shared Infrastructure

| Device | Role | What it Does |
|--------|------|--------------|
| **Headnode** | Test Server | Runs BNG-Blaster (BBL) traffic generator, connects to leaves for testing |
| **NCS5501-1** | Aggregation Switch | Cisco switch that aggregates traffic between devices and test equipment |
| **NCS5501-2** | Aggregation Switch | Second Cisco switch |
| **STC Chassis** | Spirent TestCenter | Hardware traffic generator for high-precision testing |

---

## 3. Network Diagram <a name="3-network-diagram"></a>

```
                                    ╔═══════════════════════════════════════════════════╗
                                    ║               EXTERNAL NETWORK                    ║
                                    ║           (Internet / Other Labs)                 ║
                                    ╚════════════════════╤══════════════════════════════╝
                                                         │
                                    ┌────────────────────┴────────────────────┐
                                    │                                         │
                    ╭───────────────┴───────────────╮       ╭────────────────┴────────────────╮
                    │         BLR1 POD              │       │          BLR2 POD               │
                    │      (Bangalore Pod 1)        │       │       (Bangalore Pod 2)         │
                    ╰───────────────┬───────────────╯       ╰────────────────┬────────────────╯
                                    │                                        │
           ┌────────────────────────┼────────────────────────┬───────────────┼────────────────────────┐
           │                        │                        │               │                        │
   ┌───────┴───────┐        ┌───────┴───────┐        ┌───────┴───────┐       │                ┌───────┴───────┐
   │  BLR1SPINE01  │◄──────►│  BLR1SPINE02  │◄──────►│ BLR1A10SW01   │       │                │  BLR2SPINE01  │
   │  172.27.65.185│        │  172.27.65.186│        │ 172.27.65.184 │       │                │  172.27.65.195│
   │   (91_7KDA)   │        │   (91_7KDB)   │        │   (91_7KCA)   │       │                │   (91_7KDA)   │
   └───────┬───────┘        └───────┬───────┘        └───────────────┘       │                └───────┬───────┘
           │                        │                                         │                        │
           └────────────┬───────────┘                                         │                        │
                        │                                                     │                        │
                ┌───────┴───────┐                                     ┌───────┴───────┐        ┌───────┴───────┐
                │  BLR1LEAF01   │◄═══════════════════════════════════►│  BLR2LEAF01   │◄──────►│  BLR2LEAF02   │
                │  172.27.65.187│      (L2VPN Pseudowire Path)        │  172.27.65.197│        │  172.27.65.198│
                │   (91_7KE0)   │                                     │   (91_7KE0)   │        │   (91_7KE1)   │
                └───────┬───────┘                                     └───────┬───────┘        └───────────────┘
                        │                                                     │
                        │ ifp-0/1/11                               ifp-0/1/11 │
                        │                                                     │
                ════════╪═════════════════════════════════════════════════════╪══════════
                        │             NCS-5501 SWITCHES                       │
                        │          (Aggregation Layer)                        │
                        ▼                                                     ▼
                ┌───────────────┐                                     ┌───────────────┐
                │   NCS5501-1   │◄───────────────────────────────────►│   NCS5501-2   │
                │ 10.100.98.160 │                                     │ 10.100.98.161 │
                └───────┬───────┘                                     └───────────────┘
                        │
                        │ (Trunk with many VLANs)
                        │
                        ▼
                ┌───────────────────────────────────────────────────────────────────────┐
                │                         HEADNODE SERVER                               │
                │                        172.27.65.189                                  │
                │                                                                       │
                │   Interface Mapping:                                                  │
                │   ┌─────────────────────────────────────────────────────────────┐    │
                │   │  eno2 (trunk)     ───► To NCS5501-1                         │    │
                │   │    ├── eno2.1230  ───► VLAN 1230 ───► BLR1LEAF01 ifp-0/1/11 │    │
                │   │    └── eno2.2258  ───► VLAN 2258 ───► BLR2LEAF01 ifp-0/1/11 │    │
                │   └─────────────────────────────────────────────────────────────┘    │
                │                                                                       │
                │   BNG-Blaster (BBL) runs here to generate test traffic               │
                └───────────────────────────────────────────────────────────────────────┘

                ┌───────────────────────────────────────────────────────────────────────┐
                │                    SPIRENT TEST CENTER (STC)                          │
                │                         172.27.64.1                                   │
                │                                                                       │
                │   Hardware traffic generator connected to FAB3 devices               │
                │   (NOT directly connected to BLR2LEAF01)                             │
                └───────────────────────────────────────────────────────────────────────┘
```

---

## 4. IP Address Reference <a name="4-ip-address-reference"></a>

### Management IP Addresses (SSH/API Access)

| Device | Management IP | Element ID | Platform |
|--------|--------------|------------|----------|
| **BLR1LEAF01** | 172.27.65.187 | 91_080_101_7KE0 | Edgecore s9600-72xc |
| **BLR1SPINE01** | 172.27.65.185 | 91_080_101_7KDA | Edgecore s9600-72xc |
| **BLR1SPINE02** | 172.27.65.186 | 91_080_101_7KDB | Edgecore s9600-72xc |
| **BLR1A10SWITCH01** | 172.27.65.184 | 91_080_101_7KCA | Edgecore AS5916-54XKS |
| **BLR2LEAF01** | 172.27.65.197 | 91_080_102_7KE0 | Edgecore s9600-72xc |
| **BLR2LEAF02** | 172.27.65.198 | 91_080_102_7KE1 | Edgecore s9600-72xc |
| **BLR2SPINE01** | 172.27.65.195 | 91_080_102_7KDA | Edgecore s9600-72xc |
| **BLR2SPINE02** | 172.27.65.196 | 91_080_102_7KDB | Edgecore s9600-72xc |
| **Headnode** | 172.27.65.189 | - | Linux Server |
| **NCS5501-1** | 10.100.98.160 | - | Cisco NCS-5501 |
| **NCS5501-2** | 10.100.98.161 | - | Cisco NCS-5501 |
| **STC Chassis** | 172.27.64.1 | - | Spirent TestCenter |

### Loopback Addresses (Used for BGP/EVPN)

| Device | Loopback0 (Default) | Loopback1 (IP2) |
|--------|--------------------|-----------------| 
| BLR1LEAF01 | fd00:a4::21 | 62.225.21.146 |
| BLR1SPINE01 | fd00:a4::11 | 62.225.21.144 |
| BLR2LEAF01 | fd00:a4::21 | 62.225.21.130 |

---

## 5. How Devices Connect to Each Other <a name="5-device-connections"></a>

### BLR1LEAF01 Connections

```
BLR1LEAF01 Interface Mapping:

┌─────────────────────────────────────────────────────────────────┐
│                         BLR1LEAF01                              │
│                                                                 │
│   Uplink Ports (to Spines):                                    │
│   ├── ifp-0/1/68 ──────► BLR1SPINE01 (91_7KDA)                │
│   └── ifp-0/1/64 ──────► BLR1SPINE02 (91_7KDB)                │
│                                                                 │
│   Access Ports (Customer-facing / Test ports):                 │
│   ├── ifp-0/1/11 ──────► NCS5501 ──────► Headnode eno2.1230   │
│   ├── ifp-0/1/4  ──────► OLT / DPU (GROOT)                    │
│   └── ifp-0/0/64, ifp-0/0/68 ──► Reserved test ports          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### BLR1SPINE01 Connections

```
BLR1SPINE01 Interface Mapping:

┌─────────────────────────────────────────────────────────────────┐
│                         BLR1SPINE01                             │
│                                                                 │
│   Downlink to Leaf:                                            │
│   └── ifp-0/1/20 ──────► BLR1LEAF01 (91_7KE0)                 │
│                                                                 │
│   Lateral to Other Spine:                                      │
│   └── ifp-0/1/31 ──────► BLR1SPINE02 (91_7KDB)                │
│                                                                 │
│   Downlink to A10SWITCH:                                       │
│   └── ifp-0/1/30 ──────► BLR1A10SWITCH01 (91_7KCA)            │
│                                                                 │
│   External:                                                     │
│   └── ifp-0/1/24 ──────► DA1-SA2 (External router)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### BLR2LEAF01 Connections

```
BLR2LEAF01 Interface Mapping:

┌─────────────────────────────────────────────────────────────────┐
│                         BLR2LEAF01                              │
│                                                                 │
│   Uplink Ports (to Spines):                                    │
│   ├── ifp-0/1/68 ──────► BLR2SPINE01 (91_7KDA)                │
│   └── ifp-0/1/64 ──────► BLR2SPINE02 (91_7KDB)                │
│                                                                 │
│   Access Ports:                                                 │
│   ├── ifp-0/1/11 ──────► NCS5501 ──────► Headnode eno2.2258   │
│   └── ifp-0/1/10 ──────► OLT / DPU (GROOT)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. The Headnode (Traffic Generator Server) <a name="6-headnode"></a>

### What is the Headnode?

The **Headnode** is a Linux server that runs:
- **BNG-Blaster (BBL)**: Software traffic generator for PPPoE, L2TP, and L2VPN testing
- Acts as a "fake customer" to test the network

### Headnode Network Interfaces

```
Key Headnode Interfaces:

┌──────────────────────────────────────────────────────────────────────────┐
│  Interface     │  Type        │  Connects To                            │
├──────────────────────────────────────────────────────────────────────────┤
│  eno2          │  Trunk       │  NCS5501-1 (carries multiple VLANs)     │
│  eno2.1230     │  VLAN 1230   │  BLR1LEAF01 ifp-0/1/11                  │
│  eno2.2258     │  VLAN 2258   │  BLR2LEAF01 ifp-0/1/11                  │
│  eno3.1000     │  VLAN 1000   │  Management / other traffic             │
└──────────────────────────────────────────────────────────────────────────┘
```

### ⚠️ Important Limitation: No SRIOV on BLR

The BLR headnode **does NOT have SR-IOV (Single Root I/O Virtualization)** interfaces like the FAB3 headnode.

| Testbed | Traffic Generator Interface | Connection Type |
|---------|---------------------------|-----------------|
| **FAB3** | enp69s0v11, enp69s0v23 | Direct SRIOV (high performance) |
| **BLR** | eno2.1230, eno2.2258 | Through NCS-5501 switch (shared path) |

This means:
- On FAB3: Traffic goes directly from headnode → leaf
- On BLR: Traffic goes headnode → NCS-5501 → leaf (switched path)

---

## 7. Testing Traffic Paths <a name="7-traffic-paths"></a>

### Path 1: PPPoE/L2TP Testing (Single Leaf)

```
PPPoE Test Traffic Flow:

┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  Headnode   │────────►│   NCS5501    │────────►│  BLR1LEAF01 │
│   BBL       │ VLAN    │  (Switch)    │         │  (BNG)      │
│  eno2.1230  │  1230   │              │         │ ifp-0/1/11  │
└─────────────┘         └──────────────┘         └─────────────┘

1. BBL sends PPPoE Discovery packets
2. Traffic goes through NCS-5501 with VLAN 1230
3. BLR1LEAF01 receives on ifp-0/1/11
4. BLR1LEAF01 acts as BNG (Broadband Network Gateway)
5. PPP session established
```

### Path 2: L2VPN/Pseudowire Testing (Between Two Leaves)

```
L2VPN Pseudowire Test Traffic Flow:

   HEADNODE                                                    HEADNODE
   ┌───────┐                                                  ┌───────┐
   │ BBL   │                                                  │ BBL   │
   │ TX/RX │                                                  │ TX/RX │
   └───┬───┘                                                  └───┬───┘
       │ eno2.1230                                      eno2.2258 │
       ▼                                                          ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │                        NCS-5501 Switch                           │
   │                    (VLAN 1230)    (VLAN 2258)                    │
   └───────────┬──────────────────────────────────────────┬───────────┘
               │                                          │
               ▼                                          ▼
        ┌─────────────┐                            ┌─────────────┐
        │ BLR1LEAF01  │◄═══════════════════════════│ BLR2LEAF01  │
        │ ifp-0/1/11  │   EVPN-VPWS Pseudowire    │ ifp-0/1/11  │
        │             │   (Through Spines)         │             │
        │  SiteID:    │                           │  SiteID:    │
        │   2314      │                           │   2315      │
        └─────────────┘                            └─────────────┘
              │                                          │
              │ ifp-0/1/68                     ifp-0/1/68│
              ▼                                          ▼
        ┌─────────────┐                            ┌─────────────┐
        │ BLR1SPINE01 │◄══════════════════════════►│ BLR2SPINE01 │
        │             │    (MPLS/EVPN Backbone)    │             │
        └─────────────┘                            └─────────────┘


    ┌────────────────────────────────────────────────────────────────┐
    │                    PSEUDOWIRE EXPLANATION                      │
    ├────────────────────────────────────────────────────────────────┤
    │  A pseudowire is like a "virtual cable" connecting two        │
    │  locations through a network. Customer traffic enters at      │
    │  one leaf, gets encapsulated in MPLS labels, travels          │
    │  through the spine network, and exits at the other leaf       │
    │  looking exactly as it went in.                                │
    │                                                                │
    │  EVPN-VPWS = Ethernet VPN - Virtual Private Wire Service      │
    └────────────────────────────────────────────────────────────────┘
```

---

## 8. Pseudowire (L2VPN) Testing - Detailed <a name="8-pseudowire-testing"></a>

### What is a Pseudowire?

Think of a **pseudowire** as a "virtual wire" that connects two remote locations through a network:

```
Without Pseudowire (Physical Wire):
┌──────────┐                                      ┌──────────┐
│ Location │══════════ Physical Wire ════════════│ Location │
│    A     │         (Very Expensive!)           │    B     │
└──────────┘                                      └──────────┘

With Pseudowire (Virtual Wire):
┌──────────┐     ┌───────────────────────┐      ┌──────────┐
│ Location │────►│    IP/MPLS Network    │─────►│ Location │
│    A     │     │   (Shared Network)    │      │    B     │
└──────────┘     └───────────────────────┘      └──────────┘
                         │
                   Pseudowire
                (Virtual Tunnel)
```

### BLR Testbed Pseudowire Configuration

For L2VPN testing between BLR1LEAF01 and BLR2LEAF01:

| Parameter | BLR1LEAF01 | BLR2LEAF01 |
|-----------|------------|------------|
| **Route Target (RT)** | 41383 | 41383 |
| **Local Site ID** | 2314 | 2315 |
| **Remote Site ID** | 2315 | 2314 |
| **Interface** | ifp-0/1/11 | ifp-0/1/11 |
| **VLAN (T_TAG)** | 3070 | 3070 |
| **Headnode Interface** | eno2.1230 | eno2.2258 |

### Why Site IDs Must Be Swapped

```
Pseudowire Site ID Matching:

BLR1LEAF01                          BLR2LEAF01
┌─────────────────┐                 ┌─────────────────┐
│ LocalSiteID:    │                 │ LocalSiteID:    │
│     2314        │◄═══════════════►│     2315        │
│                 │                 │                 │
│ RemoteSiteID:   │  Must Match!    │ RemoteSiteID:   │
│     2315        │◄═══════════════►│     2314        │
└─────────────────┘                 └─────────────────┘

Local of A = Remote of B
Local of B = Remote of A
```

### ⚠️ BLR Testbed Limitation for Pseudowire Testing

On BLR, both ends of the pseudowire connect through the **same NCS-5501 switch**:

```
Problem: Switched Path (Not Ideal)

┌───────────┐
│  Headnode │
│           │
│ eno2.1230 ├──── VLAN 1230 ────┬───────────────────────────────┐
│           │                   │                               │
│ eno2.2258 ├──── VLAN 2258 ────┤      NCS-5501 Switch          │
│           │                   │      ┌───────────────┐        │
└───────────┘                   └──────│               │────────┘
                                       │  Same Switch! │
                                       └───────────────┘
                                              │
                         ┌────────────────────┴────────────────────┐
                         │                                         │
                         ▼                                         ▼
                   BLR1LEAF01                               BLR2LEAF01
```

**Why this is not ideal:**
- Traffic might "shortcut" through the switch instead of going through the full network path
- For proper testing, each endpoint should have a separate, independent path

**Proper Setup (like FAB3):**
```
Ideal: Direct SRIOV Connections

┌───────────┐
│  Headnode │
│           │
│ enp69s0v11├───── Direct ──────────────────────► FAB3LEAF01
│           │      (No switch in between)
│ enp69s0v23├───── Direct ──────────────────────► FAB3LEAF02
│           │
└───────────┘
```

---

## 9. Credentials Reference <a name="9-credentials"></a>

### Device Access

| Device Type | Username | Password | Access Method |
|-------------|----------|----------|---------------|
| All RBFS Switches | supervisor | supervisor | SSH |
| Headnode | spsingh | 1!2@3#QwE | SSH |
| NCS-5501 | admin | (check with admin) | SSH |
| STC | (API access) | - | REST API |

### Example SSH Commands

```bash
# Connect to BLR1LEAF01
ssh supervisor@172.27.65.187

# Connect to Headnode
ssh spsingh@172.27.65.189

# Once on RBFS device, access CLI:
/usr/local/bin/cli
```

---

## 10. Common Commands <a name="10-common-commands"></a>

### On RBFS Devices (BLR1LEAF01, etc.)

```bash
# Show device info
show version

# Show interfaces
show interface

# Show BGP summary
show bgp summary instance default

# Show L2VPN/EVPN instances
show evpn instance

# Show EVPL configuration
show l2vpn evpl
show l2vpn attachment-circuit summary

# Show pseudowire status
show l2vpn pseudowire
```

### On Headnode (BBL Commands)

```bash
# Check available interfaces
ip link show | grep eno2

# Start BBL for L2VPN testing
sudo bngblaster -C /path/to/config.json -I test-run

# Check BBL status
curl http://localhost:8001/api/v1/status
```

### Interface Naming Convention (RBFS)

```
ifp-X/Y/Z     = Interface Physical (port)
ifl-X/Y/Z/V   = Interface Logical (with VLAN V)

Examples:
ifp-0/1/11    = Physical port, slot 0, blade 1, port 11
ifl-0/1/11/3070 = Logical interface with VLAN 3070 on port 11
```

---

## Quick Reference Card

```
╔════════════════════════════════════════════════════════════════════════════╗
║                      BLR TESTBED QUICK REFERENCE                           ║
╠════════════════════════════════════════════════════════════════════════════╣
║                                                                            ║
║  DEVICE IPs:                                                               ║
║  ├── BLR1LEAF01:     172.27.65.187                                        ║
║  ├── BLR1SPINE01:    172.27.65.185                                        ║
║  ├── BLR2LEAF01:     172.27.65.197                                        ║
║  ├── Headnode:       172.27.65.189                                        ║
║  └── NCS5501-1:      10.100.98.160                                        ║
║                                                                            ║
║  HEADNODE VLAN INTERFACES:                                                 ║
║  ├── eno2.1230 ──► BLR1LEAF01 ifp-0/1/11                                  ║
║  └── eno2.2258 ──► BLR2LEAF01 ifp-0/1/11                                  ║
║                                                                            ║
║  PSEUDOWIRE CONFIG (L2VPN):                                                ║
║  ├── RT: 41383 (must match both sides)                                    ║
║  ├── BLR1LEAF01: LocalSite=2314, RemoteSite=2315                          ║
║  └── BLR2LEAF01: LocalSite=2315, RemoteSite=2314                          ║
║                                                                            ║
║  CREDENTIALS:                                                              ║
║  ├── RBFS devices: supervisor / supervisor                                ║
║  └── Headnode:     spsingh / 1!2@3#QwE                                    ║
║                                                                            ║
║  ⚠️ LIMITATION: BLR uses switched path (NCS-5501), not direct SRIOV       ║
║                                                                            ║
╚════════════════════════════════════════════════════════════════════════════╝
```

---

## Glossary

| Term | Full Name | Explanation |
|------|-----------|-------------|
| **BGP** | Border Gateway Protocol | Routing protocol for exchanging routes between networks |
| **EVPN** | Ethernet VPN | Technology for Layer 2 connectivity over IP/MPLS |
| **VPWS** | Virtual Private Wire Service | Point-to-point L2 connection (pseudowire) |
| **MPLS** | Multi-Protocol Label Switching | Fast packet forwarding using labels |
| **BBL** | BNG Blaster | Software traffic generator |
| **STC** | Spirent Test Center | Hardware traffic generator |
| **SRIOV** | Single Root I/O Virtualization | Direct hardware access for VMs/containers |
| **RT** | Route Target | Identifier for VPN routing |
| **RBFS** | RtBrick Full Stack | Switch operating system |
| **BNG** | Broadband Network Gateway | Device that handles subscriber sessions |
| **POD** | Point of Delivery | A complete network deployment unit |
| **NCS** | Network Convergence System | Cisco router/switch platform |

---

*Document created: 2025*  
*Last updated: Based on current BLR testbed configuration*
