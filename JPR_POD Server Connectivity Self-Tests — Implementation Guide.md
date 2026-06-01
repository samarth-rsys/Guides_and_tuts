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
2. Neighbors (POD Servers) are discoverable
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

**Purpose:** Verify POD servers are discoverable by pinging their loopback IPs in the inband management VRF.

**Juniper Command Used:** `ping` (via `ExecutePingDiagnosis` API)

**Algorithm:**
1. Get metadata for the spine
2. For each known POD server neConfigID (1, 2, 3):
   - Compute the expected IP from `INBAND_MGMT_IP_PREFIX` + neConfigID offset
   - Execute ping to that IP via the switch client
   - Check if any responses were received
3. Log overall PASSED/FAILED per discovered pod server hostname

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
