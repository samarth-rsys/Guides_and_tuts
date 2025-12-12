# 📡 What "Backbone" Means in te-runbooks

> A complete explanation of the "backbone" concept in network testing

---

## 🌐 Overview

In te-runbooks test files, **"backbone"** refers to the **network-side/internet-side** of the test setup, as opposed to the **access-side** (subscriber/customer side).

---

## 🏗️ Network Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                        NETWORK TOPOLOGY                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   🏠 SUBSCRIBER                                    🌐 INTERNET       │
│   (Access Side)                                   (Backbone Side)   │
│                                                                      │
│   ┌──────────┐      ┌──────────┐      ┌──────────────────────────┐ │
│   │ Customer │      │   LEAF   │      │     BACKBONE NETWORK     │ │
│   │ (CPE)    │ ───→ │  (BNG)   │ ───→ │ (Core/Internet/Upstream) │ │
│   └──────────┘      └──────────┘      └──────────────────────────┘ │
│                          │                        │                 │
│   ACCESS INTERFACE       │           NETWORK INTERFACE              │
│   (BBL_ACCESS_INF)       │           (BBL_NETWORK_INF)              │
│                          │                        │                 │
│                          │               BACKBONE IPs               │
│                          │           (BACKBONE_IP_1, BB1, BB2...)   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📖 Key Terms

| Term | What It Means | Example |
|------|---------------|---------|
| **Backbone** | The **upstream network** (core network, internet) | Deutsche Telekom's core network |
| **Backbone IP** | IP addresses representing **internet servers** | `80.156.102.83` (BB1) |
| **Backbone Device** | Simulated **internet endpoint** (traffic generator) | BNG Blaster network interface |
| **Backbone Capture** | Packet capture on the **network/internet side** | Traffic going to/from the internet |
| **Access** | The **subscriber/customer side** | Home router connecting via PPPoE |

---

## 🔧 In BNG Blaster Configuration

Looking at a typical test file configuration:

```yaml
interfaces:
  # SUBSCRIBER SIDE (Access)
  access:
    interface: #-MACRO- BBL_ACCESS_INF -#    # e.g., ens3v14
    type: pppoe
    
  # INTERNET SIDE (Backbone/Network)
  network:
    interface: #-MACRO- BBL_NETWORK_INF -#   # e.g., enp197s0v2
    address:   #-MACRO- BBL_NETWORK_IP -#    # e.g., 80.156.102.67
    gateway:   80.156.102.1                  # Backbone gateway

# Traffic streams use Backbone IPs as destination
streams:
  - name: "IPv4_TCP_stream_1"
    network-ipv4-address: #-MACRO- BACKBONE_IP_1 -#   # 80.156.102.85 = BB1
    
  - name: "IPv4_TCP_stream_2"
    network-ipv4-address: #-MACRO- BACKBONE_IP_2 -#   # 80.156.102.86 = BB2
```

---

## 🔄 Traffic Flow

```
                    UPSTREAM (to Internet)
                         ↑
┌────────────────────────┼────────────────────────┐
│  TEST SETUP            │                        │
│                        │                        │
│  ┌──────────────┐      │      ┌──────────────┐ │
│  │ Access       │      │      │ Network      │ │
│  │ Interface    │      │      │ Interface    │ │
│  │ (Subscriber) │      │      │ (BACKBONE)   │ │
│  └──────┬───────┘      │      └──────┬───────┘ │
│         │              │             │         │
│         │         ┌────┴────┐        │         │
│         └────────►│  LEAF   │◄───────┘         │
│                   │  (BNG)  │                   │
│                   └─────────┘                   │
│                                                 │
│  BNG Blaster simulates BOTH ends:              │
│  - Access side: PPPoE client (subscriber)      │
│  - Network side: BACKBONE endpoints (servers)  │
└─────────────────────────────────────────────────┘
                         │
                         ↓
                    DOWNSTREAM (to subscriber)
```

### Direction Definitions

| Direction | Meaning | Example |
|-----------|---------|---------|
| **Upstream** | Subscriber → Backbone (Internet) | User uploading a file |
| **Downstream** | Backbone (Internet) → Subscriber | User downloading a video |

---

## 📍 Backbone IP Addresses

Defined in `macros/Leaf_generic.yml`:

### ADF Test Backbone IPs

```yaml
# For Spirent TestCenter (STC) tests
STC_ADF_BB1_IPV4: 80.156.102.83   # Backbone 1
STC_ADF_BB2_IPV4: 80.156.102.84   # Backbone 2
STC_ADF_BB1_IPV6: 2003:4:f021:a500:80:156:102:83
STC_ADF_BB2_IPV6: 2003:4:f021:a500:80:156:102:84

# For BNG Blaster (BBL) tests
BBL_ADF_BB1_IPV4: 80.156.102.85   # Backbone 1
BBL_ADF_BB2_IPV4: 80.156.102.86   # Backbone 2
BBL_ADF_BB3_IPV4: 80.156.102.88   # Backbone 3
BBL_ADF_BB4_IPV4: 80.156.102.181  # Backbone 4
BBL_ADF_BB5_IPV4: 80.156.102.182  # Backbone 5
BBL_ADF_BB6_IPV4: 80.156.102.183  # Backbone 6
BBL_ADF_BB7_IPV4: 80.156.102.184  # Backbone 7
BBL_ADF_BB8_IPV4: 80.156.102.185  # Backbone 8
BBL_ADF_BB9_IPV4: 80.156.102.186  # Backbone 9
```

### Service-Specific Backbone IPs

```yaml
# Functional test backbone IPs (simulate different services)
FUNCTIONAL_VB_IPV4:       80.156.102.3   # Video Broadcast server
FUNCTIONAL_VC_IPV4:       80.156.102.4   # Voice server
FUNCTIONAL_IPTV_ICC_IPV4: 80.156.102.7   # IPTV Interactive Channel Change
FUNCTIONAL_IPTV_VOD_IPV4: 80.156.102.6   # IPTV Video-on-Demand
FUNCTIONAL_IPTV_OTT_IPV4: 80.156.102.9   # IPTV Over-the-Top
FUNCTIONAL_BE_IPV4:       80.156.102.8   # Best Effort (general internet)

# IPv6 versions
FUNCTIONAL_VB_IPV6:       2003:4:f021:a500:80:156:102:3
FUNCTIONAL_VC_IPV6:       2003:4:f021:a500:80:156:102:4
FUNCTIONAL_BE_IPV6:       2003:4:f021:a500:80:156:102:8
```

---

## 🎯 Why Multiple Backbone IPs?

Different backbone IPs simulate **different internet servers** for testing various scenarios:

| Backbone | Code | Purpose in Tests |
|----------|------|------------------|
| **BB1** | `BACKBONE_IP_1` | Primary test server |
| **BB2** | `BACKBONE_IP_2` | Secondary test server |
| **BB3-BB9** | `BBL_ADF_BB3` - `BBL_ADF_BB9` | Additional servers for ADF filter tests |
| **VB** | `FUNCTIONAL_VB_IPV4` | Video Broadcast streaming server |
| **VC** | `FUNCTIONAL_VC_IPV4` | VoIP/Voice server |
| **IPTV_VOD** | `FUNCTIONAL_IPTV_VOD_IPV4` | Video-on-Demand server |
| **IPTV_ICC** | `FUNCTIONAL_IPTV_ICC_IPV4` | Interactive channel change server |
| **IPTV_OTT** | `FUNCTIONAL_IPTV_OTT_IPV4` | Over-the-Top streaming (Netflix, etc.) |
| **BE** | `FUNCTIONAL_BE_IPV4` | Best Effort / General internet traffic |

---

## 📋 Use Cases in Tests

### 1. ADF (Access Data Filter) Tests

Backbone IPs are used as **filter targets**:

```yaml
# Block traffic to/from specific backbone IP
ADF_FILTER: family=ipv4 direction=in action=discard dst=80.156.102.85

# Test verifies:
# - Traffic to BB1 (80.156.102.85) is BLOCKED
# - Traffic to BB2 (80.156.102.86) is ALLOWED
```

### 2. QoS Tests

Different backbone IPs represent different **service classes**:

```yaml
streams:
  - name: "Voice_stream"
    network-ipv4-address: #-MACRO- FUNCTIONAL_VC_IPV4 -#      # Voice server
    vlan-priority: 6  # High priority
    
  - name: "BestEffort_stream"
    network-ipv4-address: #-MACRO- FUNCTIONAL_BE_IPV4 -#      # BE server
    vlan-priority: 0  # Low priority
```

### 3. Bandwidth Tests

Backbone represents the **internet endpoint** for measuring throughput:

```yaml
# Traffic flows: Subscriber ↔ Backbone
# Measure bandwidth accuracy in both directions
```

### 4. Ping Tests

Backbone IPs are **ping targets** to verify connectivity:

```yaml
commands:
  - ping #-MACRO-BACKBONE_IP_1-# instance ip2 source-interface lo-0/0/1/0 count 5
  - ping #-MACRO-BACKBONE_IP_2-# instance ip2 source-interface lo-0/0/1/0 count 5
```

---

## 🖥️ Physical Setup

### Interface Mapping

| Variable | Interface | Side | Example Value |
|----------|-----------|------|---------------|
| `BBL_ACCESS_INF` | Access interface | Subscriber | `ens3v14` |
| `BBL_NETWORK_INF` | Network interface | Backbone | `enp197s0v2` |
| `BBL_NETWORK_IP` | Network IP | Backbone | `80.156.102.67/23` |
| `BACKBONE_IP_1` | Traffic destination | Backbone | `80.156.102.85` |

### BNG Blaster Server Configuration

```
BNG Blaster Server (NEBULA/LOKI)
├── Access Interface (ens3vXX)
│   └── Connected to LEAF access port
│   └── Simulates subscriber (PPPoE client)
│
└── Network Interface (enp197s0vXX)
    └── Connected to LEAF network port
    └── Simulates BACKBONE (internet servers)
    └── Has multiple IPs (BB1, BB2, BB3...)
```

---

## 🔍 Backbone vs Access Summary

| Aspect | Access Side | Backbone Side |
|--------|-------------|---------------|
| **Represents** | Customer/Subscriber | Internet/Core Network |
| **Interface** | `BBL_ACCESS_INF` | `BBL_NETWORK_INF` |
| **Protocol** | PPPoE, L2TP | IP routing |
| **Traffic Direction** | Upstream source | Upstream destination |
| **Example IPs** | Assigned by RADIUS | `80.156.102.x` |
| **In Real World** | Home router | Netflix, Google, etc. |

---

## 📊 Real-World Analogy

```
YOUR HOME                                    THE INTERNET
┌─────────────────┐                         ┌─────────────────┐
│  Your Router    │                         │   Netflix       │ ← Backbone IP 1
│  (Access Side)  │ ═══════════════════════ │   YouTube       │ ← Backbone IP 2
│                 │      ISP Network        │   Google        │ ← Backbone IP 3
│  Gets IP from   │      (LEAF/BNG)        │   Facebook      │ ← Backbone IP 4
│  RADIUS/DHCP    │                         │   ...           │
└─────────────────┘                         └─────────────────┘

In te-runbooks:
- "Access" = Your home router (simulated by BNG Blaster)
- "Backbone" = Internet servers (simulated by BNG Blaster network interface)
- "LEAF/BNG" = The router being tested (real RtBrick device)
```

---

## 🎯 Summary

**"Backbone"** in te-runbooks refers to:

✅ The **core network / internet side** (opposite of subscriber access)

✅ The **network interface** of the BNG Blaster traffic generator

✅ **IP addresses** that simulate internet servers/services

✅ The **upstream direction** of traffic flow

✅ The **destination** for subscriber traffic

It's essentially simulating the "internet" that subscribers connect to through the BNG router!

---
