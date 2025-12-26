# RD L2 Business Accounting Test - Beginner's Guide

## What is this test?

This test checks if **L2VPN (Layer 2 Virtual Private Network)** traffic is being handled correctly between two leaf switches. Think of it like testing a private tunnel between two buildings.

---

## Simple Analogy 🏢➡️🏢

Imagine two office buildings (BLR1LEAF01 and BLR2LEAF01) connected by a private underground tunnel (pseudowire). 

- Different types of messages travel through this tunnel:
  - **Voice calls** (VO) - High priority, must never be dropped
  - **Video** (LD - Low Delay) - Important, should not be dropped  
  - **Important files** (LL - Low Loss) - Should not be dropped
  - **Critical Traffic** (CT) - Must never be dropped
  - **Regular emails** (BE - Best Effort) - Can be dropped if tunnel is busy

The test checks: **"Are important messages getting through, and only regular emails being dropped when busy?"**

---

## What does the test do? (Step by Step)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Step 1: Check if tunnel exists                                      │
│  ─────────────────────────────                                       │
│  Does the pseudowire (tunnel) exist between the two leaf switches?   │
│  If NO → Create it automatically                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 2: Reset counters                                              │
│  ──────────────────────                                              │
│  Clear all previous traffic statistics (start fresh)                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 3: Send test traffic                                           │
│  ─────────────────────────                                           │
│  BNG-Blaster tool sends 5 types of traffic through the tunnel:       │
│    • BE (Best Effort)  - 15 Mbps - Low priority                      │
│    • CT (Critical)     - 100 Kbps - Highest priority                 │
│    • VO (Voice)        - 500 Kbps - High priority                    │
│    • LL (Low Loss)     - 1 Mbps - Medium-high priority               │
│    • LD (Low Delay)    - 1 Mbps - Medium-high priority               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 4: Wait 60 seconds                                             │
│  ───────────────────────                                             │
│  Let traffic flow through the tunnel                                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 5: Check results                                               │
│  ─────────────────────                                               │
│  Verify:                                                             │
│    ✅ CT, VO, LL, LD traffic → 0 packets lost                        │
│    ✅ BE traffic → Some packets lost (expected, it's low priority)   │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Step 6: Cleanup                                                     │
│  ──────────────                                                      │
│  Stop traffic, save reports, delete test configuration               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Understanding the File Structure

### The Runbook File (`TC_18_004_001__RD_L2_Business_Accounting__BLR1LEAF01.yml`)

```yaml
config:
  macroFiles:                    # Files that provide variables/settings
    - .../macros/BLR1LEAF01.yml           # Device-specific settings
    - .../macros/18_RD/BLR1LEAF01_ETHP2P.yml  # EVPL config (tunnel settings)
    - .../macros/18_RD/RD_L2_Business_Accounting.yml  # Test parameters
    - .../macros/04_PPPOE/ACCESS_generic_BLR1LEAF01.yml  # File paths for reports
    - .../macros/Leaf_generic.yml          # Generic leaf functions
```

### Why `04_PPPOE` folder? 🤔

You asked about line 32: `04_PPPOE/ACCESS_generic_BLR1LEAF01.yml`

**This is NOT a PPPoE test!** The file is in the `04_PPPOE` folder because it contains **shared variables** used by multiple tests:

| Variable | Purpose |
|----------|---------|
| `PCAP_PATH` | Where to save capture files |
| `BBL_RUN_REPORT` | Filename for the test report |
| `PCAP_BBL` | Filename for packet capture |
| `BBL_LOG_FILE` | Filename for log file |

It's like a shared toolbox - just because the toolbox is in the "PPPoE room" doesn't mean you're doing PPPoE work!

---

## Key Files Explained

### 1. Main Runbook (What you run)
```
TC_18_004_001__RD_L2_Business_Accounting__BLR1LEAF01.yml
```
- This is the test script
- Contains all the steps to execute

### 2. EVPL Configuration (Tunnel settings)
```
macros/18_RD/BLR1LEAF01_ETHP2P.yml
```
Contains:
- `RT: "41383"` - Route Target (tunnel ID)
- `IFP: "ifp-0/1/11"` - Physical interface
- `IFL: "ifl-0/1/11/3070"` - Logical interface
- `T_TAG: "3070"` - VLAN tag

### 3. Test Parameters (What to check)
```
macros/18_RD/RD_L2_Business_Accounting.yml
```
Contains:
- `BBL_INSTANCE` - Name for the traffic generator
- `BLASTER_STREAM_RESULTS` - Expected results for each traffic type
- `COUNTER_TYPE_RESULTS` - What counters to check

---

## Network Diagram

```
                    ┌─────────────────────────────────────┐
                    │         EVPN-VPWS Pseudowire        │
                    │        (Private L2 Tunnel)          │
                    └─────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┴─────────────────────────────┐
        │                                                           │
        ▼                                                           ▼
┌───────────────┐                                         ┌───────────────┐
│  BLR1LEAF01   │                                         │  BLR2LEAF01   │
│               │◄───────── Fabric Network ──────────────►│               │
│  ifp-0/1/11   │                                         │  ifp-0/1/11   │
│  ifl-0/1/11/  │                                         │  ifl-0/1/11/  │
│     3070      │                                         │     3070      │
└───────┬───────┘                                         └───────┬───────┘
        │                                                         │
        │  SRIOV Interface                         SRIOV Interface│
        │                                                         │
        ▼                                                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         BNG-Blaster (Traffic Generator)                  │
│                                                                          │
│   Sends traffic with different priorities:                               │
│   • BE  (vlan-priority: 0) - Best Effort    - 15 Mbps                   │
│   • CT  (vlan-priority: 6) - Critical       - 100 Kbps                  │
│   • VO  (vlan-priority: 5) - Voice          - 500 Kbps                  │
│   • LL  (vlan-priority: 3) - Low Loss       - 1 Mbps                    │
│   • LD  (vlan-priority: 4) - Low Delay      - 1 Mbps                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Expected Results

| Traffic Type | Priority | Expected Packet Loss |
|-------------|----------|---------------------|
| CT (Critical) | 6 (Highest) | ❌ 0 (No loss) |
| VO (Voice) | 5 | ❌ 0 (No loss) |
| LD (Low Delay) | 4 | ❌ 0 (No loss) |
| LL (Low Loss) | 3 | ❌ 0 (No loss) |
| BE (Best Effort) | 0 (Lowest) | ✅ > 0 (Some loss expected) |

**Why is BE expected to have loss?**  
Because we're sending more traffic than the tunnel can handle. The QoS (Quality of Service) system drops low-priority traffic first to protect important traffic.

---

## How to Run This Test

```bash
te4-worker te-runbooks/18_RD/TC_18_004_001__RD_L2_Business_Accounting__BLR1LEAF01.yml
```

---

## Glossary

| Term | Meaning |
|------|---------|
| **L2VPN** | Layer 2 Virtual Private Network - A private network tunnel |
| **EVPL** | Ethernet Virtual Private Line - A type of L2VPN service |
| **Pseudowire** | The virtual tunnel between two switches |
| **IFP** | Physical interface (port on the switch) |
| **IFL** | Logical interface (virtual interface with VLAN) |
| **QoS** | Quality of Service - Rules for prioritizing traffic |
| **BNG-Blaster** | Traffic generator tool |
| **Prometheus** | Monitoring system that collects metrics |
| **SRIOV** | Hardware feature for direct network access |

---

## File Comparison: FAB3 vs BLR1

Both files are **identical in structure**, only the device names change:

| FAB3LEAF01 Version | BLR1LEAF01 Version |
|-------------------|-------------------|
| FAB3LEAF01 | BLR1LEAF01 |
| FAB3LEAF02 | BLR2LEAF01 |

This is because the test logic is the same - just different leaf switches in different locations!
