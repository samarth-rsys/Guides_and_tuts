# POD Server Connectivity Self-Tests — Implementation Guide

## Table of Contents

1. [What is a POD Server?](#what-is-a-pod-server)
2. [What Are Self-Tests?](#what-are-self-tests)
3. [Architecture Overview](#architecture-overview)
4. [Test Definitions](#test-definitions)
5. [Implementation Details](#implementation-details)
6. [Reference: RTBrick vs Juniper](#reference-rtbrick-vs-juniper)
7. [How Alarms Work](#how-alarms-work)
8. [Code Walkthrough](#code-walkthrough)
9. [Key Concepts for Newcomers](#key-concepts-for-newcomers)

---

## What is a POD Server?

A **POD Server** is a physical server within a Point of Delivery (POD) in a telecom network. Think of it like this:

```
┌──────────────────────────────────────────────────────┐
│                    POD (Point of Delivery)            │
│                                                      │
│   ┌─────────┐       ┌─────────┐       ┌─────────┐   │
│   │  Spine  │◄─────►│  Spine  │◄─────►│   LSR   │   │
│   │ Switch  │       │ Switch  │       │(Router) │   │
│   └────┬────┘       └────┬────┘       └─────────┘   │
│        │                  │                          │
│   ┌────┴────┐       ┌────┴────┐                     │
│   │  Leaf   │       │  Leaf   │                     │
│   │ Switch  │       │ Switch  │                     │
│   └────┬────┘       └────┬────┘                     │
│        │                  │                          │
│   ┌────┴────┐       ┌────┴────┐                     │
│   │   POD   │       │   POD   │                     │
│   │ Server  │       │ Server  │  ◄── These!         │
│   └─────────┘       └─────────┘                     │
└──────────────────────────────────────────────────────┘
```

POD Servers typically host:
- **DHCP servers** — assign IP addresses to subscribers
- **RADIUS servers** — authenticate subscribers
- **Management services** — OLT management, DPU management, inband management

The **Spine Switch** connects to POD Servers directly and needs to verify:
1. The physical link is UP
2. Neighbors (POD Servers) are discoverable (RTBrick: neighbor entries with MAC are present)
3. POD Servers are reachable via ping (loopback IP)

---

## What Are Self-Tests?

When a network element (switch) finishes starting up, it emits a `StartupFinished` event. The **diagnosis actor** listens for this event and automatically runs a suite of self-tests to verify the switch is healthy.

```
Switch boots up
       │
       ▼
StartupFinished event emitted
       │
       ▼
Diagnosis actor receives event
       │
       ▼
90-second delay (allow initialization)
       │
       ▼
Run self-tests in parallel (with dependencies)
       │
       ▼
If any test FAILED → Raise alarm (Major severity)
If all PASSED → Clear alarm (Info severity)
```

---

## Architecture Overview

### File Structure

```
pkg/fixtures/diagnosis/
├── diagnosis.go                 # Commands struct, pipeline setup
├── ne_self_tests.go             # Test framework: orchestration, dependencies, alarm handling
├── ne_self_tests_alarm.go       # Alarm raising/clearing logic
├── spine_self_tests.go          # Spine-specific tests (including POD server tests) ← MODIFIED
├── execute_linktest.go          # Existing link test diagnosis (used as reference)
└── ne_connectivity_check.go     # Connectivity check (has buildMetadata helper)
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `Commands` struct | Holds all dependencies (switchClient, inventory, diagnostics, bemClient) |
| `TestDefinition` | Defines a test: name, index, function, dependencies |
| `TestResult` | Stores outcome: name, status (PASSED/FAILED/SKIPPED), index |
| `executeSelfTests()` | Runs all tests concurrently respecting dependency chains |
| `handleTestAlarms()` | Raises/clears alarms based on results |

---

## Test Definitions

The three POD server tests are defined in `getSpineTestDefinitions()`:

```go
{
    name:     "POD Server Link Test: link towards POD-Servers is UP",
    index:    POD_SERVER_LINK_TEST,
    testFunc: cmds.testPODServerLink,
    // No dependencies — runs immediately
},
{
    name:     "POD Server Neighbor Discovery: Neighbors are Discovered",
    index:    POD_SERVER_NEIGHBOR_TEST,
    testFunc: cmds.testPODServerNeighbors,
    dependsOn: []string{
        "POD Server Link Test: link towards POD-Servers is UP",
    },
    // Only runs if Link Test passes
},
{
    name:     "POD Server Reachability Test: Reachability is Working",
    index:    POD_SERVER_REACHABILITY_TEST,
    testFunc: cmds.testPODServerReachability,
    dependsOn: []string{
        "POD Server Neighbor Discovery: Neighbors are Discovered",
    },
    // Only runs if Neighbor Discovery passes
},
```

**Dependency Chain:**
```
POD Server Link Test
        │ (must pass)
        ▼
POD Server Neighbor Discovery
        │ (must pass)
        ▼
POD Server Reachability Test
```

If Link Test fails → Neighbor & Reachability are SKIPPED.

---

## Implementation Details

### Test 1: `testPODServerLink`

**Purpose:** Verify that physical links between the Spine switch and POD servers are operationally UP.

**Juniper Command Used:** `show interfaces extensive <interface-name>`

**Algorithm:**
1. Get extended data (management IP, neConfigID) from inventory
2. Build metadata for switch client API calls
3. Fetch all ports of the spine switch from inventory
4. For each port:
   - Fetch the Network Element Link (NEL) connected to this port
   - Skip if NEL is in PLANNING + NOT_WORKING state
   - Find the port on the other end of the link
   - Fetch the network element on the other end
   - Check if it's a POD_SERVER (by category attribute)
   - If yes: get the CLI label via port mapping, query interface state
   - Log PASSED if `oper-status` is "up", FAILED otherwise
5. If no POD server connections found → SKIPPED

**Key Data Flow:**
```
Spine NE
   │
   ├── Port A ──── NEL ──── Port X (POD Server 1) ✓ Check this link
   ├── Port B ──── NEL ──── Port Y (Other Spine)  ✗ Skip (not POD server)
   └── Port C ──── NEL ──── Port Z (POD Server 2) ✓ Check this link
```

### Test 2: `testPODServerNeighbors`

**Purpose:** Verify POD servers are discoverable as neighbors.

**RTBrick API Used (reference):** `getNeighbors(...)` (neighbor/ARP-style discovery data)

**Juniper Adaptation:** Uses `ping` (via `ExecutePingDiagnosis` API) as a proxy signal for neighbor/discovery health, because Juniper flow does not expose the same neighbor API shape used in RTBrick.

**Algorithm:**
1. **RTBrick reference behavior**
    - Fetch neighbors from RBFS (`getNeighbors`)
    - Validate required instances (for example `inband_mgmt`, `olt_mgmt`, `dpu_mgmt`, `ip2`, `ancp_mgmt`)
    - Ensure neighbor entries exist and each neighbor has a non-empty MAC address
2. **Juniper implementation in this repo**
    - For each known POD server neConfigID (1, 2, 3), compute destination IP in `inband_mgmt`
    - Execute ping via switch client
    - Mark discovery failed if no responses are received
3. Log PASSED/FAILED per discovered POD server hostname

#### PodServerNeighborsDiscovery: RTBrick vs Juniper (Very Easy)

Think of both systems as checking: **"Do I see POD server neighbors?"**
But they check it in different ways.

##### RTBrick way (direct neighbor table check)

- RTBrick first does explicit auth in this function (`prepareAuthorization`) to get token/port/protocol.
- Then it calls `getNeighbors(...)` from RBFS.
- It reads neighbor entries directly (like ARP/neighbor table data).
- It verifies each required instance has neighbors and each neighbor has a MAC.

Simple idea:

```
RTBrick asks: "Show me your neighbor list"
Device shows table with MAC entries
If table looks correct -> PASS
```

##### Juniper way (ping-based neighbor proof)

- Juniper does **not** call `prepareAuthorization` in this test.
- It builds `metadata` and uses `switchClient.ExecutePingDiagnosis(...)`.
- Auth is handled internally by the shared HTTP client.
- It computes POD server IPs from prefix + neConfigID and pings them.
- If ping replies are received, it treats neighbor discovery as healthy.

Simple idea:

```
Juniper asks: "Can I reach these POD server IPs by ping?"
If ping replies come -> PASS
```

##### One-line difference

- **RTBrick:** checks neighbor table entries directly.
- **Juniper:** uses successful ping as practical proof of neighbor/connectivity health.

### Test 3: `testPODServerReachability`

**Purpose:** Full bidirectional reachability test — ping from spine's own loopback to each POD server's loopback.

**Juniper Command Used:** `ping` with source IP (via `ExecutePingDiagnosis` API)

**Algorithm:**
1. Get spine's own neConfigID (used to compute source IP)
2. Determine instances to test:
   - Spine: only `inband_mgmt`
   - Leaf: `inband_mgmt`, `olt_mgmt`, `dpu_mgmt`
3. For each POD server neConfigID and each instance:
   - Compute source IP = `IP_PREFIX` + spine's neConfigID
   - Compute destination IP = `IP_PREFIX` + pod server's neConfigID
   - Ping with both source and destination specified
   - Verify `sent == received` (zero packet loss)
4. Log PASSED/FAILED per POD server

---

## Reference: RTBrick vs Juniper

The POD server tests were originally implemented in the **rtbrick-rbfs-actor** project. Here's how the two implementations compare:

### RTBrick Implementation (Reference)

Located in: `rtbrick-rbfs-actor/pkg/fixtures/diagnosis/podServer_connectivitytest.go`

| Aspect | RTBrick | Juniper (Our Implementation) |
|--------|---------|------------------------------|
| **Device Access** | RBFS REST API via `go-rbfs-client` | Juniper REST API via `switchclient` |
| **Context Creation** | `commons.NewRbfsContext()` with token/port/protocol | `buildMetadata()` + direct `switchClient` calls |
| **Interface Check** | `cmd.rbfs.GetPhysicalInterface(rbfsContext, cliLabel)` | `cmds.switchClient.GetPhysicalInterface(ctx, cliLabel, metadata)` |
| **Ping** | `cmd.rbfs.RunPing(rbfsContext, pingCmd)` using `ping.NewPing()` | `cmds.switchClient.ExecutePingDiagnosis(ctx, pingReq, metadata)` |
| **Neighbors** | Direct ARP/neighbor table query via RBFS API | Ping-based verification (Juniper doesn't expose same API) |
| **Port Discovery** | `FetchNetworkElementPortsOfNetworkElement()` | `FetchNetworkElementPorts()` (added to interface) |
| **NEL Fetch** | `FetchNetworkElementLink()` (existed) | `FetchNetworkElementLink()` (newly added) |
| **Auth** | Token-based via `prepareAuthorization()` | Handled internally by switchClient HTTP client |

### What We Borrowed from RTBrick

1. **Overall algorithm** — iterate ports → find NEL → filter POD servers → check interface
2. **Test dependency chain** — Link → Neighbors → Reachability
3. **IP computation logic** — `computeIPAddress()` using `netaddr-go` library with env-var prefixes
4. **Pod server neConfigIDs** — hardcoded `[1, 2, 3]`
5. **Test result logging pattern** — per-link granularity with hostname in test name
6. **Constants/templates** — `linkTowardsPodServerTemplate`, etc.

### What We Adapted for Juniper

1. **No RBFS context** — Juniper uses `store.Metadata` + direct switchClient calls
2. **Interface state field** — RTBrick checks `ifp.OperationalState != "UP"`, Juniper checks `GetDataFieldString(ifp.OperStatus) != "up"` (lowercase)
3. **Ping API** — RTBrick uses `ping.NewPing()` builder → `RunPing()`. Juniper uses `models.Ping{}` struct → `ExecutePingDiagnosis()`
4. **Neighbor discovery** — RTBrick queries ARP tables. Juniper uses ping-based approach since it doesn't have equivalent ARP API
5. **Added `FetchNetworkElementLink`** to inventory Client interface (Juniper didn't have it)

### Existing Juniper Diagnosis Used as Reference

| File | What We Learned |
|------|-----------------|
| `execute_linktest.go` | How to call `GetPhysicalInterface()`, check `operationalStateUp`, use metadata |
| `spine_internet_connectivity_check_command.go` | How to iterate ports, build metadata, use `FetchNetworkElementPort()` |
| `ne_connectivity_check.go` | The `buildMetadata()` helper function |
| `spine_self_tests.go` (testUserPool) | Overall pattern: get extended data → build metadata → create timeout → call API |

---

## How Alarms Work

The alarm logic is handled by `handleTestAlarms()` in `ne_self_tests.go`:

```
┌─────────────────────────────────────────────────┐
│            After all tests complete              │
├─────────────────────────────────────────────────┤
│                                                 │
│  Any test FAILED?                               │
│      YES → NeSelfTestsAlarm(cleared=false)      │
│             Raises MAJOR alarm with details     │
│                                                 │
│      NO (all PASSED) →                          │
│          Check latest alarm on this NE          │
│          If previous alarm was MAJOR →          │
│              NeSelfTestsAlarm(cleared=true)     │
│              Raises INFO alarm (clears it)      │
│          If no previous alarm or was INFO →     │
│              Do nothing (skip redundant INFO)   │
│                                                 │
└─────────────────────────────────────────────────┘
```

The alarm contains:
- NE ID (source object)
- All test results with their PASSED/FAILED/SKIPPED status
- Perceived severity: MAJOR (failed) or INFO (cleared)

---

## Code Walkthrough

### Changes Made to `spine_self_tests.go`

#### 1. New Imports Added

```go
"math"                           // For math.MaxUint32 in IP computation
"github.com/dspinhirne/netaddr-go"  // IPv4 network arithmetic
inventoryModel "...api"          // For CategoryAttrName, NetworkElement, LeafSwitch constants
"...models"                      // For models.Ping struct
"...systemops"                   // For GetDataFieldString() helper
```

#### 2. Constants Added

```go
linkTowardsPodServerTemplate      = "link towards PodServer [%s] [%s] is UP"
linkTowardsPodServerBaseTemplate  = "link towards PodServer is UP"
podServerNeighborsTemplate        = "POD-Server Neighbors [%v] are Discovered"
podServerNeighborsBaseTemplate    = "POD-Server Neighbors are Discovered"
podServerReachabilityTemplate     = "POD-Server %v Reachability is Working"
podServerReachabilityBaseTemplate = "POD-Server Reachability is Working"
```

These templates are used in `logTestResult()` to create human-readable test names with dynamic values (hostnames, IDs).

#### 3. Helper Functions Added

| Function | Purpose |
|----------|---------|
| `checkPodServerPort()` | Checks a single port's link to a POD server (extracted for readability) |
| `findOtherPortID()` | Finds the port ID on the other end of a NEL |
| `pingPodServer()` | Pings a single POD server by neConfigID |
| `reachabilityPingTest()` | Tests reachability to one pod server across all instances |
| `pingInstance()` | Executes a single ping for one instance |
| `computeIPAddress()` | Routes to correct IP range by instance name |
| `computeAddress()` | Computes IP from CIDR prefix + offset |
| `podServerResults()` | Logs results for each discovered pod server hostname |

### Changes Made to `pkg/fixtures/inventory/logical.go`

#### Added to Client Interface

```go
// FetchNetworkElementPorts fetches all network element ports for a given network element.
FetchNetworkElementPorts(ctx actorAPI.ActorContext, neID string) ([]*inventoryAPI.CLILogicalResource, actorAPI.LayerError)
// FetchNetworkElementLink returns the network element link the port with the given ID is connected with.
FetchNetworkElementLink(ctx actorAPI.ActorContext, portID string) (*inventoryAPI.CLILogicalResource, actorAPI.LayerError)
```

#### Added Implementation

```go
func (c *client) FetchNetworkElementLink(ctx actorAPI.ActorContext, portID string) (*inventoryAPI.CLILogicalResource, actorAPI.LayerError) {
    links, err := c.inventoryClient.FetchChildResourcesByParentId(
        context.Background(), portID, inventoryModel.NetworkElementLink)
    // ... error handling ...
    return &links[0], nil
}
```

This queries the inventory for NetworkElementLink resources that are children of a given port. A NEL represents the physical cable between two ports.

### Removed from `ne_self_tests.go`

The three dummy stubs that just returned `true`:
```go
// DELETED:
func (cmds *Commands) testPODServerLink(...) bool { return true }
func (cmds *Commands) testPODServerNeighbors(...) bool { return true }
func (cmds *Commands) testPODServerReachability(...) bool { return true }
```

---

## Understanding Nil SourceIP and 0 Interval (Simple Explanation)

### The Problem with Nil SourceIP and 0 Interval

When the Neighbor test pings a POD server, it creates a Ping object like this:

```go
pingReq := &models.Ping{
    DestinationIP: net.ParseIP("192.168.16.1"),  // Where to ping
    InstanceName:  "inband_mgmt",                // Which network (like an address book)
    Count:         3,                             // Ping 3 times
    Size:          56,                            // Packet size
    SourceIP:      nil,                           // NOT SET ← This is intentional
    Interval:      0,                             // NOT SET ← This is the default
}
```

Think of it like **sending a letter**:

- **DestinationIP** = Mailing address (required) ✓ Ping where?
- **SourceIP** = Your return address (optional) → Sometimes you want to specify which of YOUR addresses to use, sometimes let the mail system pick
- **Interval** = Wait time between letters (must be at least 1 second) → You can't send infinite letters instantly

### Why SourceIP is Nil

The **Neighbor test** doesn't care which spine interface sends the ping. It just wants to verify: **"Can the spine reach the POD server at all?"**

Think of it like:
```
Spine: "I'll call the POD server from whatever phone is available"
       (doesn't say "use office phone" or "use home phone")
```

The switch will automatically pick the right interface.

BUT the **Reachability test** DOES specify source:

```go
pingReq := &models.Ping{
    DestinationIP: net.ParseIP("192.168.16.1"),
    SourceIP:      net.ParseIP("192.168.16.5"),   // ← Specific! "Use spine's IP"
    InstanceName:  "inband_mgmt",
    Count:         3,
}
```

This asks: **"Can spine IP 192.168.16.5 reach POD server 192.168.16.1?"** (Bidirectional verification)

### Why Interval was 0 Seconds

When you create a Ping struct without setting Interval, Go defaults it to zero:

```go
pingReq := &models.Ping{...}  // Interval is time.Duration(0) by default
```

This becomes `0.000` in the Juniper ping command:

```
ping routing-instance inband_mgmt 192.168.16.1 count 3 interval 0.000 ← ❌ WRONG!
```

**Juniper doesn't accept `interval 0`** — you must wait at least 1 second between pings. You can't ping infinitely fast.

Think of it like:
```
You: "Ping this server as fast as possible!"
Juniper: "Sorry, I need at least 1 second between pings. I can't go faster."
```

### The Fix

The code now handles both cases:

```go
// Only add "source X" if SourceIP is actually set
if sourceIP != nil && sourceIP.String() != "" {
    // Include source in command
    command = "ping routing-instance inband_mgmt 192.168.16.1 source 192.168.16.5 count 3 interval 1.000"
} else {
    // Skip source if not needed
    command = "ping routing-instance inband_mgmt 192.168.16.1 count 3 interval 1.000"
}

// Default interval to 1 second if not set
if interval <= 0 {
    interval = 1.0
}
```

**Result:**
- Neighbor test: `ping routing-instance inband_mgmt 192.168.16.1 count 3 interval 1.000` ✓ Clean, no nil
- Reachability test: `ping routing-instance inband_mgmt 192.168.16.1 source 192.168.16.5 count 3 interval 1.000` ✓ Full test

---

## Key Concepts for Newcomers

### 1. Network Element (NE)
A physical or logical device in the network (switch, router, server). Each NE has:
- An **ID** (UUID)
- **Characteristics** (key-value pairs like `category=SPINE_SWITCH`)
- **Extended Data** (operational data like hostname, management IP, neConfigId)
- **Ports** (physical interfaces)

### 2. Network Element Link (NEL)
Represents a physical cable between two ports. A NEL has:
- Two **ResourceRelationships** pointing to the two ports it connects
- An **operationalState** (WORKING/NOT_WORKING)
- A **lifecycleState** (PLANNING/INSTALLING/OPERATING)

### 3. Network Element Group (NEG)
A logical grouping of NEs (e.g., all switches in one POD). Contains shared configuration like IP pools.

### 4. neConfigId
A numeric identifier (1, 2, 3...) assigned to each NE within a NEG. Used to compute unique IP addresses from a shared prefix. For example:
- Prefix: `10.0.0.0/24`
- Spine (neConfigId=5): `10.0.0.5`
- POD Server (neConfigId=1): `10.0.0.1`

### 5. Metadata (`store.Metadata`)
Contains everything needed to talk to a Juniper switch:
```go
type Metadata struct {
    MatNumber  string  // Material/model number
    IP         string  // Management IP address
    NeType     string  // Switch type name
    Hostname   string  // Switch hostname
    NeId       string  // NE UUID
    NeConfigID int     // Numeric config ID
}
```

### 6. Port Mapping
Maps between:
- **Functional Port Label** (inventory name, e.g., "uplink-1")
- **CLI Label** (Juniper interface name, e.g., "et-0/0/1")

The catalogue service provides this mapping based on the switch model (MatNumber) and type.

### 7. Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `INBAND_MGMT_IP_PREFIX` | CIDR for inband management IPs | `10.100.0.0/24` |
| `OLT_MGMT_IP_PREFIX` | CIDR for OLT management IPs | `10.101.0.0/24` |
| `DPU_MGMT_IP_PREFIX` | CIDR for DPU management IPs | `10.102.0.0/24` |

#### Why these IP prefixes are required (very simple)

Think of these prefixes as the **address blocks** from which test IPs are calculated.

- The POD server tests know POD server IDs (`1`, `2`, `3`), not fixed full IPs.
- So the code builds each target IP like:
    - `INBAND_MGMT_IP_PREFIX` + `neConfigId`
    - example: prefix `192.168.16.0/22` + ID `1` -> `192.168.16.1`

Without these prefixes, the code does not know where to ping.

- If the prefix is missing/empty, IP computation fails (parse error).
- If the prefix is wrong, ping goes to wrong IPs and neighbor/reachability tests fail.

In short: **Link test checks cable/interface state, but Neighbor/Reachability tests need IP prefixes to know the destination IPs.**

### 8. Test Concurrency

All tests run in goroutines. The framework uses:
- `sync.WaitGroup` — wait for all tests to finish
- `completionChannels` — signal when a test is done
- `testMu` mutex — protect shared state (testResultsMap, allResults)

Tests with dependencies **wait** on their dependency's channel before starting. If a dependency failed, the dependent test is **skipped**.

### 9. SwitchClient API Call Pattern

```go
// 1. Build metadata from inventory data
metadata := buildMetadata(networkElement, extended, neConfigID, mgmtIP, neID)

// 2. Create a timeout context
timeout, cancel := context.WithTimeout(actorCtx, 30*time.Second)
defer cancel()

// 3. Call the switch via REST API
result, err := cmds.switchClient.GetPhysicalInterface(timeout, "et-0/0/1", metadata)
// The switchClient internally constructs the URL and sends HTTP request to the switch
```

---

## Summary of All Files Changed

| File | Change Type | Description |
|------|-------------|-------------|
| `pkg/fixtures/diagnosis/spine_self_tests.go` | **Modified** | Added full implementations of `testPODServerLink`, `testPODServerNeighbors`, `testPODServerReachability` + helper functions |
| `pkg/fixtures/diagnosis/ne_self_tests.go` | **Modified** | Removed 3 dummy stubs that returned `true` |
| `pkg/fixtures/inventory/logical.go` | **Modified** | Added `FetchNetworkElementPorts` and `FetchNetworkElementLink` to Client interface + added `FetchNetworkElementLink` implementation |
