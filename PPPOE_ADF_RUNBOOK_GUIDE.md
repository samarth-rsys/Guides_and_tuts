# 🔍 Deep Dive: PPPoE ADF (Access Data Filter) Runbooks - Category 04_017

> **Complete Guide to ADF Testing in te-runbooks**

---

## 📋 Table of Contents

1. [What is ADF?](#-what-is-adf)
2. [Test Structure Overview](#-test-structure-overview)
3. [Single-Stage ADF Tests (001-019)](#-single-stage-adf-tests-001-019)
4. [ADF Replace Tests (020-045)](#-adf-replace-tests-020-045)
5. [Multi-Stage ADF Tests (046-055)](#-multi-stage-adf-tests-046-055)
6. [Complete Test Flow Explained](#-complete-test-flow-explained)
7. [Real-World Use Cases](#-real-world-use-cases)
8. [Test Results Interpretation](#-test-results-interpretation)

---

## 🛡️ What is ADF?

**ADF (Access Data Filter)** is a **firewall-like feature** for individual subscriber sessions on the BNG router. Think of it as a **per-user security checkpoint** that filters traffic based on rules.

### Simple Analogy 🏠

```
Imagine each subscriber (home) has a personal security guard:

                 🏠 Your Home (Subscriber)
                      │
                      │ ADF = Personal Security Guard
                      │
    ┌─────────────────┼─────────────────┐
    │                 ▼                 │
    │   📦 Incoming Traffic             │
    │                                   │
    │   ADF Filter Checks:              │
    │   ✅ Source IP: 1.2.3.4? → ALLOW  │
    │   ❌ Source IP: 5.6.7.8? → BLOCK  │
    │   ✅ Port 80 (HTTP)? → ALLOW      │
    │   ❌ Port 22 (SSH)? → BLOCK       │
    │                                   │
    └───────────────────────────────────┘
```

### What Can ADF Filter?

| Filter Type | What It Does | Example |
|-------------|--------------|---------|
| **IP Address** | Block/allow specific source/destination IPs | Block traffic from `5.6.7.8` |
| **Protocol** | Filter by protocol type | Block TCP, allow UDP |
| **Port** | Filter by port number | Block port `22` (SSH) |
| **Direction** | Apply to upstream or downstream | Block only outgoing traffic |
| **Priority** | Multiple filters with priority | Apply filter with priority `2` |

### ADF Filter Format

```bash
# Basic syntax
family=ipv4 direction=in action=discard dst=10.20.30.40

# Breakdown:
family=ipv4           # IPv4 or IPv6
direction=in          # in=upstream (subscriber→internet), out=downstream (internet→subscriber)
action=discard        # discard=block, permit=allow
dst=10.20.30.40      # destination IP
```

### Common Filter Variations

```bash
# Block upstream IPv4 traffic to specific IP
family=ipv4 direction=in action=discard dst=10.20.30.40

# Block downstream IPv4 traffic from specific IP
family=ipv4 direction=out action=discard src=10.20.30.40

# Block upstream TCP traffic
family=ipv4 direction=in action=discard proto=6 dst=10.20.30.40

# Block upstream traffic on specific source port
family=ipv4 direction=in action=discard sport=11010 sportq=2

# Block downstream traffic on specific destination port
family=ipv4 direction=out action=discard dport=13030 dportq=2

# IPv6 filter
family=ipv6 direction=in action=discard dst=2003:4:f021::100
```

### Protocol Numbers

| Number | Protocol |
|--------|----------|
| `proto=6` | TCP |
| `proto=17` | UDP |

### Priority (Queue)

- `sportq=2` or `dportq=2` = Priority level 2

---

## 📁 Test Structure Overview

The **Category 04_017** contains **55 test cases** organized into three main groups:

### Test Naming Pattern

```
TC_04_017_XXX__pppoe_<test_type>__<device>.yml
         │           │
         │           └─ Test description
         └─ Sub-category number (001-055)
```

### Test Categories

| Range | Category | Count | Purpose |
|-------|----------|-------|---------|
| **001-019** | Single-Stage ADF | 19 tests | Test basic ADF filtering in one direction/protocol |
| **020-045** | ADF Replace | 26 tests | Test changing ADF filters via CoA mid-session |
| **046-055** | Multi-Stage ADF | 10 tests | Test multiple simultaneous ADF filters |

---

## 🎯 Single-Stage ADF Tests (001-019)

These tests verify that **one ADF filter** works correctly.

### Test Breakdown

| Test # | Name | Filter Applied | What's Blocked |
|--------|------|----------------|----------------|
| **001** | `ipv4_up_only` | `family=ipv4 direction=in action=discard dst=BB1_IP` | All upstream IPv4 to BB1 |
| **002** | `ipv4_up_tcp` | `family=ipv4 direction=in action=discard proto=6 dst=BB1_IP` | Upstream TCP to BB1 |
| **003** | `ipv4_up_udp` | `family=ipv4 direction=in action=discard proto=17 dst=BB1_IP` | Upstream UDP to BB1 |
| **004** | `ipv4_up_port` | `family=ipv4 direction=in action=discard sport=11010 sportq=2` | Upstream from source port 11010 |
| **005** | `ipv6_up_only` | `family=ipv6 direction=in action=discard dst=BB1_IP6` | All upstream IPv6 to BB1 |
| **006** | `ipv6_up_tcp` | `family=ipv6 direction=in action=discard proto=6 dst=BB1_IP6` | Upstream TCP IPv6 to BB1 |
| **007** | `ipv6_up_udp` | `family=ipv6 direction=in action=discard proto=17 dst=BB1_IP6` | Upstream UDP IPv6 to BB1 |
| **008** | `ipv6_up_port` | `family=ipv6 direction=in action=discard sport=11010 sportq=2` | Upstream IPv6 from port 11010 |
| **009** | `ipv4_down_only` | `family=ipv4 direction=out action=discard src=BB1_IP` | All downstream IPv4 from BB1 |
| **010** | `ipv4_down_port` | `family=ipv4 direction=out action=discard dport=11010 dportq=2` | Downstream to dest port 11010 |
| **011** | `ipv6_down_only` | `family=ipv6 direction=out action=discard src=BB1_IP6` | All downstream IPv6 from BB1 |
| **012** | `ipv6_down_port` | `family=ipv6 direction=out action=discard dport=13030 dportq=2` | Downstream IPv6 to port 13030 |
| **014** | `ipv4_up_udp_port` | `family=ipv4 direction=in action=discard proto=17 sport=13030 sportq=2` | Upstream UDP from port 13030 |
| **015** | `ipv6_up_udp_port` | `family=ipv6 direction=in action=discard proto=17 sport=11010 sportq=2` | Upstream UDP IPv6 from port 11010 |
| **016** | `ipv4_down_tcp_port` | `family=ipv4 direction=out action=discard proto=6 dport=13030 dportq=2` | Downstream TCP to port 13030 |
| **017** | `ipv4_down_udp_port` | `family=ipv4 direction=out action=discard proto=17 dport=13030 dportq=2` | Downstream UDP to port 13030 |
| **018** | `ipv6_down_tcp_port` | `family=ipv6 direction=out action=discard proto=6 dport=13030 dportq=2` | Downstream TCP IPv6 to port 13030 |
| **019** | `ipv6_down_udp_port` | `family=ipv6 direction=out action=discard proto=17 dport=13030 dportq=2` | Downstream UDP IPv6 to port 13030 |

### Test Flow Example: `TC_04_017_004__pppoe_Single_Stage_ADF_ipv4_up_port`

```
┌────────────────────────────────────────────────────────────────────┐
│                      TEST FLOW DIAGRAM                              │
├────────────────────────────────────────────────────────────────────┤
│  1. SETUP PHASE                                                     │
│     ├─→ Check LEAF interface is UP                                 │
│     ├─→ Check default route exists                                 │
│     ├─→ Set VLAN profile (VLAN 303)                               │
│     └─→ Configure PyRAD RADIUS server with ADF filter:            │
│         "family=ipv4 direction=in action=discard sport=11010       │
│          sportq=2"                                                 │
│                                                                     │
│  2. START SESSION                                                   │
│     ├─→ Start BNG Blaster instance                                 │
│     ├─→ Configure 8 traffic streams (4 IPv4 + 4 IPv6):            │
│     │   ├─ IPv4_TCP_stream_1: src-port 11010 (BLOCKED ❌)          │
│     │   ├─ IPv4_TCP_stream_2: src-port 13030 (ALLOWED ✅)          │
│     │   ├─ IPv4_UDP_stream_1: src-port 11010 (BLOCKED ❌)          │
│     │   ├─ IPv4_UDP_stream_2: src-port 13030 (ALLOWED ✅)          │
│     │   └─ IPv6 streams (all ALLOWED ✅, filter is IPv4 only)      │
│     └─→ Establish PPPoE session                                    │
│                                                                     │
│  3. VERIFY ADF ACTIVE                                               │
│     ├─→ Get subscriber ID from router                              │
│     └─→ Check ADF filter is applied on router:                     │
│         "show subscriber <ID> acl subscriber-filter-ppp-..."       │
│                                                                     │
│  4. TEST TRAFFIC (20 seconds)                                       │
│     └─→ Let traffic flow and monitor streams                       │
│                                                                     │
│  5. CHECK RESULTS                                                   │
│     ├─→ IPv4_TCP_stream_1 upstream:   rx-bps-l2 == 0  ❌ BLOCKED  │
│     ├─→ IPv4_TCP_stream_1 downstream: rx-bps-l2 > 0   ✅ ALLOWED  │
│     ├─→ IPv4_TCP_stream_2 upstream:   rx-bps-l2 > 0   ✅ ALLOWED  │
│     ├─→ IPv4_UDP_stream_1 upstream:   rx-bps-l2 == 0  ❌ BLOCKED  │
│     └─→ IPv6 streams: all allowed (filter doesn't apply)           │
│                                                                     │
│  6. CLEANUP                                                         │
│     ├─→ Stop BNG Blaster                                           │
│     ├─→ Wait for subscriber termination                            │
│     ├─→ Delete VLAN profile                                        │
│     ├─→ Save captures (pcap files)                                 │
│     └─→ Reset RADIUS reply (remove filter)                         │
└────────────────────────────────────────────────────────────────────┘
```

### Traffic Streams Configuration

Each test uses **8 traffic streams** (bidirectional = 16 flows):

```yaml
Streams:
  IPv4_TCP_stream_1:  ports 11000 ↔ 11010  (to/from BB1)
  IPv4_TCP_stream_2:  ports 13000 ↔ 13030  (to/from BB2)
  IPv4_UDP_stream_1:  ports 11000 ↔ 11010  (to/from BB1)
  IPv4_UDP_stream_2:  ports 13000 ↔ 13030  (to/from BB2)
  IPv6_TCP_stream_1:  ports 11000 ↔ 11010  (to/from BB1)
  IPv6_TCP_stream_2:  ports 13000 ↔ 13030  (to/from BB2)
  IPv6_UDP_stream_1:  ports 11000 ↔ 11010  (to/from BB1)
  IPv6_UDP_stream_2:  ports 13000 ↔ 13030  (to/from BB2)
```

**BB = Backbone IP addresses** (traffic generators)

---

## 🔄 ADF Replace Tests (020-045)

These tests verify that **ADF filters can be changed mid-session** using **CoA (Change of Authorization)**.

### What is CoA?

**CoA** is a RADIUS feature that allows the RADIUS server to **push changes to an active session** without disconnecting:

```
Session Active with Filter #1
         │
         │ ← CoA Request from RADIUS
         │   "Change filter to Filter #2"
         ▼
Session continues with Filter #2 (no disconnect!)
```

### Test Pattern: Replace Tests

| Test # | Name | Original Filter | Replaced Filter | Change |
|--------|------|----------------|-----------------|--------|
| **020** | `ip_to_ip_downstream` | Block src=BB3 | Block src=BB4 | Change blocked IP |
| **021** | `ip_to_ip_port_downstream` | Block src=BB3 | Block dport=11010 | IP → Port |
| **022** | `ip_port_to_ip_downstream` | Block dport=11010 | Block src=BB4 | Port → IP |
| **023** | `ip6_to_ip6_downstream` | Block src=BB3_v6 | Block src=BB4_v6 | Change IPv6 IP |
| **026** | `ip_tcp_port_to_ip_downstream` | Block TCP dport=11010 | Block src=BB4 | TCP Port → IP |
| **027** | `ip_to_ip_udp_port_downstream` | Block src=BB3 | Block UDP dport=11010 | IP → UDP Port |
| **030** | `ip_to_ip_upstream` | Block dst=BB3 | Block dst=BB4 | Upstream IP change |
| **037** | `ip_udp_port_to_ip_upstream` | Block UDP sport=11010 | Block dst=BB5 | UDP Port → IP |
| **041** | `ip6_to_ip_upstream` | Block IPv6 dst=BB6 | Block IPv4 dst=BB5 | IPv6 → IPv4 |
| **042** | `ip_down_to_ip_upstream` | Block downstream src=BB6 | Block upstream dst=BB5 | Direction change |

### Test Flow Example: ADF Replace Test (with CoA)

```
┌────────────────────────────────────────────────────────────────────┐
│              ADF REPLACE TEST FLOW (with CoA)                       │
├────────────────────────────────────────────────────────────────────┤
│  1. SETUP (same as single-stage)                                    │
│     └─→ Set initial ADF filter in RADIUS:                          │
│         "family=ipv4 direction=out action=discard src=BB3"          │
│                                                                     │
│  2. START SESSION & INITIAL TRAFFIC                                 │
│     ├─→ Establish PPPoE session                                    │
│     ├─→ Start 12 traffic streams (IPv4 + IPv6, TCP + UDP)         │
│     └─→ Wait 10 seconds                                            │
│                                                                     │
│  3. CHECK INITIAL FILTER                                            │
│     ├─→ IPv4_TCP_stream_1 downstream: BLOCKED (src=BB3) ❌         │
│     ├─→ IPv4_UDP_stream_1 downstream: BLOCKED (src=BB3) ❌         │
│     └─→ All other streams: ALLOWED ✅                              │
│                                                                     │
│  4. SEND CoA REQUEST (Change Filter!)                               │
│     ├─→ Get Acct-Session-Id from subscriber                        │
│     ├─→ Send CoA to router with NEW filter:                        │
│     │   "family=ipv4 direction=out action=discard dport=11010      │
│     │    dportq=2"                                                 │
│     └─→ Router applies new filter immediately                      │
│                                                                     │
│  5. VERIFY NEW FILTER                                               │
│     └─→ Check router ACL changed to new filter                     │
│                                                                     │
│  6. RESET COUNTERS & TEST AGAIN                                     │
│     ├─→ Reset BNG Blaster stream counters                          │
│     └─→ Wait 20 seconds for new traffic                            │
│                                                                     │
│  7. CHECK NEW FILTER RESULTS                                        │
│     ├─→ IPv4_TCP_stream_1 downstream: BLOCKED (dport=11010) ❌     │
│     ├─→ IPv4_UDP_stream_1 downstream: BLOCKED (dport=11010) ❌     │
│     ├─→ Stream from BB3 now ALLOWED (filter changed!) ✅           │
│     └─→ Verify filter changed successfully                         │
│                                                                     │
│  8. CLEANUP (same as single-stage)                                  │
└────────────────────────────────────────────────────────────────────┘
```

### CoA Implementation in Test

```yaml
# Get accounting session ID
- check:
    name: Get Session_Id from subscriber
    commands:
      - get subscribers/.VAR.SUBSCRIBER_ID.
    jsonPath: json.accounting.accounting_session_id
    cache:
      variable: Account_Session_Id

# Send CoA request with new filter
- check:
    name: Send COA request
    POST:
      sendCoa:
        attributes:
          Acct-Session-Id: .VAR.Account_Session_Id[0].
          Ascend-Data-Filter: family=ipv4 direction=out action=discard dport=11010 dportq=2
        targetIp: <LEAF_IP>
        targetPort: 3799
        code: 43  # CoA-Request
```

---

## 🎭 Multi-Stage ADF Tests (046-055)

These tests verify that **multiple ADF filters can be applied simultaneously** to one subscriber.

### What is Multi-Stage?

**Multi-Stage ADF** = **Multiple filters active at the same time**:

```
Subscriber Session
    │
    ├─→ Filter #1: Block src=BB7 (IPv4)
    ├─→ Filter #2: Block src=BB9 (IPv4)
    └─→ Filter #3: Block src=BB11 (IPv6)
        
All three filters active simultaneously!
```

### Test Scenarios

| Test # | Name | Initial Filter | After CoA | Filters Count |
|--------|------|---------------|-----------|---------------|
| **046** | `IPv4_IPv4_downstream` | 1 filter (src=BB7) | 2 filters (+src=BB9) | 2 IPv4 |
| **047** | `IPv4_IPv6_downstream` | 1 filter (src=BB7 v4) | 2 filters (+src=BB9 v6) | 1 IPv4 + 1 IPv6 |
| **048** | `IPv4_TCP_P2_IPv6_UDP_P2_down` | 1 filter (src=BB7) | 2 filters (TCP+UDP) | Protocol specific |
| **049** | `IPv4_P2_IPv6_P2_downstream` | 1 filter | 2 filters (port-based) | Port filters |
| **050** | `IPv4_IPv4_upstream` | 1 filter (dst=BB7) | 2 filters (+dst=BB9) | 2 upstream IPv4 |
| **051** | `IPv4_IPv6_upstream` | 1 filter (dst=BB7 v4) | 2 filters (+dst=BB9 v6) | Upstream mixed |
| **052** | `IPv4_TCP_IPv6_UDP_upstream` | 1 filter | 2 filters (TCP v4 + UDP v6) | Protocol mixed |
| **053** | `IPv4_P3_IPv6_P1_upstream` | 1 filter | 2 filters (different ports) | Port priorities |
| **054** | `large_scale` | 1 filter | Multiple filters | Scalability test |
| **055** | `CoA_deactivate` | 2 filters active | Remove filters via CoA | Deactivation |

### Test Flow: Multi-Stage ADF

```
┌────────────────────────────────────────────────────────────────────┐
│            MULTI-STAGE ADF TEST FLOW                                │
├────────────────────────────────────────────────────────────────────┤
│  1. SETUP                                                           │
│     └─→ Set FIRST filter in RADIUS:                                │
│         "family=ipv4 direction=out action=discard src=BB7"          │
│                                                                     │
│  2. START SESSION & TRAFFIC                                         │
│     ├─→ Establish PPPoE session                                    │
│     ├─→ Start 10 traffic streams (IPv4 + IPv6)                     │
│     └─→ Wait 10 seconds                                            │
│                                                                     │
│  3. CHECK FIRST FILTER                                              │
│     ├─→ IPv4 from BB7: BLOCKED ❌                                   │
│     ├─→ IPv4 from BB9: ALLOWED ✅                                   │
│     └─→ IPv6 streams: ALLOWED ✅                                    │
│                                                                     │
│  4. SEND CoA TO ADD SECOND FILTER                                   │
│     └─→ Send CoA with TWO filters:                                 │
│         Filter #1: "family=ipv4 direction=out action=discard       │
│                     src=BB7"                                        │
│         Filter #2: "family=ipv4 direction=out action=discard       │
│                     src=BB9"                                        │
│                                                                     │
│  5. VERIFY MULTI-STAGE                                              │
│     └─→ Check router now has BOTH filters active                   │
│                                                                     │
│  6. RESET & TEST WITH BOTH FILTERS                                  │
│     ├─→ Reset counters                                             │
│     └─→ Wait 20 seconds                                            │
│                                                                     │
│  7. CHECK BOTH FILTERS WORKING                                      │
│     ├─→ IPv4 from BB7: BLOCKED ❌ (Filter #1)                      │
│     ├─→ IPv4 from BB9: BLOCKED ❌ (Filter #2) ← NEW!               │
│     ├─→ Other IPv4: ALLOWED ✅                                      │
│     └─→ IPv6: ALLOWED ✅                                            │
│                                                                     │
│  8. CLEANUP                                                         │
└────────────────────────────────────────────────────────────────────┘
```

### Multi-Stage Filter Example

```yaml
# Initial RADIUS reply (1 filter)
Ascend-Data-Filter:
  - "family=ipv4 direction=out action=discard src=10.1.1.7"

# After CoA (2 filters)
Ascend-Data-Filter:
  - "family=ipv4 direction=out action=discard src=10.1.1.7"
  - "family=ipv4 direction=out action=discard src=10.1.1.9"
```

---

## 🔬 Complete Test Flow Explained

### Tools Used

| Tool | Purpose | Where It Runs |
|------|---------|---------------|
| **BNG Blaster** | Traffic generator (simulates subscribers) | NEBULA/LOKI server |
| **PyRAD PFS** | Test RADIUS server | ROCKET server |
| **LEAF Router** | BNG device under test | Test lab |
| **tcpdump** | Packet capture | Test servers |

### Detailed Step-by-Step

#### Phase 1: Initialization

```yaml
# 1. Check testbed ready
LEAF_INTERFACE_STATUS_CHECK()
  → Verify physical interface UP
  → Example: ifp-0/1/4

LEAF_DEFAULT_ROUTE_CHECK()
  → Verify routing works
  → Ping test to upstream
```

#### Phase 2: Configure RADIUS

```yaml
# 2. Set VLAN profile on router
LEAF_GLOBAL_VLAN_PROFILE_SET(interface, inner-vlan=7, outer-vlan=303, aaa-name)
  → Create subscriber access profile
  → Associate with RADIUS server

# 3. Configure PyRAD with ADF filter
POST /controlServerStatus
  setAccessReply:
    DALAB.A4.BLANK_BBL_1:  # Remote-ID
      attributes:
        Ascend-Data-Filter: "family=ipv4 direction=in action=discard sport=11010 sportq=2"
        Framed-IP-Address: "-> fromPool"
        ... other attributes ...
      code: 2  # Access-Accept
```

#### Phase 3: Start Traffic Generator

```yaml
# 4. Configure BNG Blaster
PUT /api/v1/instances/instance_pppoe_single_adf_1
  json:
    interfaces:
      access:
        interface: enp1s0f0
        type: pppoe
        outer-vlan-min: 303
        inner-vlan-min: 7
        authentication-protocol: PAP
    ppp:
      authentication:
        username: bngblaster
        password: test
    streams:
      - name: "IPv4_TCP_stream_1"
        type: ipv4
        direction: both
        source-port: 11010  # This will be BLOCKED by ADF
        destination-port: 11000
        raw-tcp: true
      - name: "IPv4_TCP_stream_2"
        type: ipv4
        source-port: 13030  # This will be ALLOWED
        destination-port: 13000

# 5. Start BNG Blaster
POST /api/v1/instances/instance_pppoe_single_adf_1/_start
  json:
    logging: true
    pcap_capture: true
```

#### Phase 4: Verify Session

```yaml
# 6. Wait for PPPoE session established
await:
  POST /api/v1/instances/instance_pppoe_single_adf_1/_command
    json:
      command: session-info
      arguments:
        session-id: 1
  jsonPath: 'session-info.session-state'
  evalValue: '{} == "Established"'

# 7. Get subscriber ID from router
GET /bds/table/walk
  json:
    table:
      table_name: "local.pppoe.ppp.sessions"
      ... filters by MAC/VLAN ...
  → Extract subscriber_id
```

#### Phase 5: Verify ADF Applied

```yaml
# 8. Check ADF filter on router
show subscriber <SUBSCRIBER_ID> acl subscriber-filter-ppp-0/1/4/<SUBSCRIBER_ID>
  → Should show:
    Rule: subscriber-filter-ppp-0/1/4/<SUBSCRIBER_ID>
    family=ipv4 direction=in action=discard sport=11010 sportq=2
```

#### Phase 6: Test Traffic

```yaml
# 9. Let traffic flow
sleep: 20 sec

# 10. Check each stream result
for ITER in range(1, 17):
  POST /api/v1/instances/.../_command
    json:
      command: stream-info
      arguments:
        flow-id: <ITER>
  jsonPath: stream-info.rx-bps-l2
  
  Expected results:
    Flow 1 (IPv4 TCP upstream, port 11010):   rx-bps-l2 == 0  ❌ BLOCKED
    Flow 2 (IPv4 TCP downstream):             rx-bps-l2 > 0   ✅ ALLOWED
    Flow 3 (IPv4 TCP upstream, port 13030):   rx-bps-l2 > 0   ✅ ALLOWED
    ...
```

#### Phase 7: CoA (Replace Tests Only)

```yaml
# 11. Get accounting session ID
GET /subscribers/<SUBSCRIBER_ID>
  jsonPath: json.accounting.accounting_session_id
  → Store in Account_Session_Id

# 12. Send CoA request
POST /controlServerStatus
  sendCoa:
    attributes:
      Acct-Session-Id: <Account_Session_Id>
      Ascend-Data-Filter: "family=ipv4 direction=out action=discard dport=11010 dportq=2"
    targetIp: <LEAF_LOOPBACK_IP>
    targetPort: 3799
    code: 43  # CoA-Request

# 13. Verify filter changed
show subscriber <SUBSCRIBER_ID> acl ...
  → Should show NEW filter

# 14. Reset counters and test again
POST .../_command
  json:
    command: stream-reset
sleep: 20 sec

# 15. Check streams again with new filter
→ Different flows should now be blocked!
```

#### Phase 8: Cleanup

```yaml
# 16. Stop BNG Blaster
POST /api/v1/instances/.../_stop

# 17. Wait for subscriber termination
sleep: 5 sec
LEAF_SUBSCRIBER_TERMINATION_AWAIT(910)

# 18. Delete VLAN profile
LEAF_GLOBAL_VLAN_PROFILE_DELETE(...)

# 19. Save captures
GET /api/v1/instances/.../run.pcap
  → Save to file system
GET /api/v1/instances/.../run_report.json
  → Save report
GET /api/v1/instances/.../run.log
  → Save logs

# 20. Attach files to test report
attachFile: <pcap_file>
attachFile: <report_file>

# 21. Reset RADIUS reply (remove ADF)
POST /controlServerStatus
  setAccessReply:
    DALAB.A4.BLANK_BBL_1:
      attributes:
        # Same attributes but WITHOUT Ascend-Data-Filter
      code: 2
```

---

## 🌍 Real-World Use Cases

### 1. Parental Controls 👨‍👩‍👧‍👦

```
Use Case: Block adult content websites for a subscriber

ADF Filter:
  family=ipv4 direction=out action=discard dst=<adult_site_ip>
  
How it works:
  - RADIUS sends filter during authentication
  - All traffic to blocked IPs is dropped
  - Can be changed via CoA when parental controls disabled
```

### 2. Temporary Service Suspension 💸

```
Use Case: Customer didn't pay bill, restrict internet but allow billing portal

Initial Filter: (full service)
  No filter

After CoA: (payment required)
  family=ipv4 direction=out action=discard dst=0.0.0.0/0
  family=ipv4 direction=out action=permit dst=<billing_portal_ip>
  
Result: Can only access billing website
```

### 3. Port Blocking (Security) 🔒

```
Use Case: Block potentially dangerous ports (e.g., Telnet, SSH from home users)

ADF Filter:
  family=ipv4 direction=in action=discard dport=23 dportq=2   # Block Telnet
  family=ipv4 direction=in action=discard dport=22 dportq=2   # Block SSH
  
Result: User cannot run Telnet/SSH servers
```

### 4. Dynamic Threat Mitigation 🚨

```
Use Case: Subscriber device infected, attacking specific IP

Normal State: No filter

After Detection (via CoA):
  family=ipv4 direction=in action=discard dst=<attack_target_ip>
  
After Cleanup (via CoA):
  Remove filter
  
Result: Attack stopped without disconnecting subscriber
```

### 5. Lawful Interception Support 👮

```
Use Case: Redirect specific traffic for legal monitoring

ADF Filter:
  family=ipv4 direction=both action=redirect dst=<target_ip> redirect-to=<li_server>
  
(Note: This is conceptual; actual LI is more complex)
```

---

## 📊 Test Results Interpretation

### Stream Result Validation

Each test checks traffic streams with this pattern:

```yaml
TRAFFIC_STREAM_RESULTS:
  - ["IPv4_TCP_stream_1 upstream",    "{} == 0" ]  # BLOCKED
  - ["IPv4_TCP_stream_1 downstream",  "{} > 0" ]   # ALLOWED
```

**Interpretation:**
- `{} == 0` means `rx-bps-l2 == 0` → **NO bytes received** → **BLOCKED** ❌
- `{} > 0` means `rx-bps-l2 > 0` → **Bytes received** → **ALLOWED** ✅

### Multi-Stage Result Table

```yaml
# Three columns: [Stream Name, Before CoA, After CoA]
TRAFFIC_STREAM_RESULTS:
  - ["IPv4_TCP_stream_1 upstream",    "{} > 0"   , "{} > 0"  ]
  - ["IPv4_TCP_stream_1 downstream",  "{} == 0"  , "{} == 0" ]  # Blocked both times
  - ["IPv4_UDP_stream_3 downstream",  "{} > 0"   , "{} == 0" ]  # Newly blocked after CoA
```

---

## 🎯 Key Takeaways

### What These Tests Verify

✅ **Single-Stage Tests (001-019):**
- ADF filters are applied correctly
- Filters work for IPv4/IPv6
- Direction filtering (upstream/downstream)
- Protocol filtering (TCP/UDP)
- Port filtering (source/destination)

✅ **Replace Tests (020-045):**
- Filters can be changed mid-session via CoA
- No session interruption during filter change
- New filter takes effect immediately
- Old filter is replaced (not added)

✅ **Multi-Stage Tests (046-055):**
- Multiple filters work simultaneously
- Filters can be added via CoA
- Each filter operates independently
- Large-scale filter sets work

### Why This Matters

🎯 **Operational Flexibility:**
- Change subscriber policies without disconnection
- Implement security measures in real-time
- Support dynamic service offerings

🎯 **Customer Experience:**
- No service interruption when changing policies
- Instant activation of new services/restrictions

🎯 **Business Value:**
- Enable parental controls
- Implement dynamic pricing/quotas
- Support compliance requirements

---

## 📝 Summary Table: All 55 ADF Tests

| Category | Tests | Focus Area |
|----------|-------|------------|
| **Single-Stage (001-019)** | 19 | Basic filtering: IPv4/IPv6, TCP/UDP, ports, directions |
| **Replace (020-045)** | 26 | Filter replacement via CoA: IP→IP, IP→Port, Protocol changes |
| **Multi-Stage (046-055)** | 10 | Multiple concurrent filters, scalability, deactivation |

**Total:** 55 comprehensive ADF test scenarios ensuring robust access control functionality!

---

*Document: PPPOE_ADF_RUNBOOK_GUIDE.md*  
*Version: 1.0*  
*Generated: December 11, 2025*  
*Author: GitHub Copilot*

---

*This guide explains the complete ADF testing strategy in the PPPoE module. Each test follows a similar pattern but validates different filter configurations and transitions, ensuring the BNG router correctly implements access control for subscriber sessions.*
