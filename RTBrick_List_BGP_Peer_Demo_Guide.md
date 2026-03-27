# 🎯 RTBrick — List BGP Peer Flow (Complete Demo Guide)

## Table of Contents

1. [What is the RTBrick Switch Simulator?](#1-what-is-the-rtbrick-switch-simulator)
2. [RTBrick vs Juniper Simulator — Key Differences](#2-rtbrick-vs-juniper-simulator--key-differences)
3. [The `${True}` Flag — What It Does](#3-the-true-flag--what-it-does)
4. [Architecture Overview](#4-architecture-overview)
5. [List BGP Peer — Complete End-to-End Flow](#5-list-bgp-peer--complete-end-to-end-flow)
6. [Simulator Code Structure](#6-simulator-code-structure)
7. [Deep Dive: handleListBGPPeering Handler](#7-deep-dive-handlelistbgppeering-handler)
8. [Currently Supported Simulator Endpoints](#8-currently-supported-simulator-endpoints)
9. [How to Add a New Endpoint — Step-by-Step](#9-how-to-add-a-new-endpoint--step-by-step)
10. [How to Write a Robot Test for It](#10-how-to-write-a-robot-test-for-it)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. What is the RTBrick Switch Simulator?

The **RTBrick Switch Simulator** is a **fake RTBrick RBFS switch** that runs as a Docker container (`switch-simulator-1`).

In production, the **RBFS Diagnosis Actor** talks to a **real RTBrick switch** (e.g., BLR1SPINE02 at IP `172.27.65.186`) over HTTPS using a **REST API**. The actor sends HTTP requests like `GET /api/v1/rbfs/elements/<switchName>/services/opsd/proxy/bgp/peerings` to fetch BGP data.

During **automated testing**, we don't have a real switch. So we built a **Go HTTP server** that:

- Listens on HTTPS (port 12321) just like a real RTBrick switch controller
- Exposes the same REST API endpoints (e.g., `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/bgp/peerings`)
- Returns **realistic fake JSON responses** (matching exactly what a real RBFS switch would return)

> **In simple words:** The simulator pretends to be a real RTBrick RBFS switch so tests can run without any hardware.

---

## 2. RTBrick vs Juniper Simulator — Key Differences

| Aspect | Juniper Simulator | RTBrick Simulator |
|---|---|---|
| **Protocol** | JSON-RPC (single POST endpoint) | REST API (multiple GET/POST endpoints) |
| **API Style** | Single `POST /api/v1/rpc` with command routing | Separate URL per operation (RESTful) |
| **Port** | 443 (HTTPS) | 12321 (HTTPS) |
| **Command Dispatch** | Parses `"command"` field in JSON body | URL-based routing by Gin framework |
| **Routing Logic** | Manual `if/else` in `handleRPC()` | Gin's built-in URL router |
| **Package** | `pkg/juniper/pkg/simulator/` | `pkg/rtbrick/pkg/switchclient/` (lives with the client code) |

### Juniper: One door, many rooms 🚪
```
POST /api/v1/rpc  →  {"command": "show isis adjacency"}  →  router picks handler
POST /api/v1/rpc  →  {"command": "show bgp neighbor"}    →  router picks handler
```

### RTBrick: Many doors, one room each 🚪🚪🚪
```
GET /api/v1/rbfs/elements/:switchName/services/opsd/proxy/bgp/peerings     →  BGP handler
GET /api/v1/rbfs/elements/:switchName/services/opsd/proxy/isis/neighbors   →  ISIS handler
GET /api/v1/rbfs/elements/:switchName/services/opsd/proxy/pim/neighbors    →  PIM handler
```

---

## 3. The `${True}` Flag — What It Does

When you see this in the RTBrick test:

```robot
${resp}=    Update_Logical_Resource_Info    BLR1SPINE02    ${True}
```

The `${True}` flag tells the system: **"Point the device to the simulator container instead of the real switch."**

### What happens step by step:

```
Step 1: Get Docker container IP of "switch-simulator-1"
         └─> runs: docker inspect switch-simulator-1 → gets e.g., 172.19.0.15

Step 2: Update the NEMS operational data:
         ├─ hostname → "switch-simulator-1"   (instead of real hostname)
         └─ managementIpAddresses → 172.19.0.15  (instead of real IP 172.27.65.186)

Step 3: Store this updated info in Logical Inventory + NEMS Inventory databases
```

### Result:

| Field | `${True}` (Simulator) | `${False}` (Real Switch) |
|---|---|---|
| hostname | `switch-simulator-1` | `91_080_101_7KDB` |
| IP Address | Docker container IP (`172.19.0.15`) | Real switch IP (`172.27.65.186`) |

Later, when the **RBFS Diagnosis Actor** processes a diagnosis request, it looks up the device from inventory and connects to the **simulator** instead of the **real switch**.

---

## 4. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    LIST BGP PEER — COMPLETE FLOW                         │
│                                                                          │
│  Robot Test Case (RtbrickDiagnosisTests.robot)                           │
│  RBFS_ACTOR_TEST_004                                                     │
│       │                                                                  │
│       │ 1. Create_Table          → Sets up PostgreSQL tables             │
│       │ 2. Update_Logical_Resource_Info(BLR1SPINE02, ${True})            │
│       │    └─> Registers device with simulator's IP in inventory         │
│       │ 3. Update_NEG_INFO(BLR1SPINE02)                                  │
│       │    └─> Adds Network Element Group data                           │
│       │                                                                  │
│       │ 4. Bgp_Peering_Request(${BLR1SPINE02["id"]})                     │
│       │    └─> POST http://localhost:9080/diagnoses                      │
│       │         Body: {                                                  │
│       │           "diagnosisType": "...ListBgpPeeringsRequest",          │
│       │           "diagnosisParameters": {                               │
│       │             "resourceId": "91000000-0080-0000-0101-120000000000" │
│       │           }                                                      │
│       │         }                                                        │
│       ▼                                                                  │
│  ┌─────────────────────────────────┐                                     │
│  │  RBFS Diagnosis Actor           │ (port 9080 → container port 2604)   │
│  │  (a4-rbfs-diagnosis-actor)      │                                     │
│  └──────────┬──────────────────────┘                                     │
│             │                                                            │
│             │ 5. Looks up device in Logical Inventory                    │
│             │    → Finds hostname: "switch-simulator-1"                  │
│             │    → Finds IP: 172.19.0.15                                 │
│             │                                                            │
│             │ 6. GET https://172.19.0.15:12321/api/v1/rbfs/elements/     │
│             │        switch-simulator-1/services/opsd/proxy/bgp/peerings │
│             ▼                                                            │
│  ┌─────────────────────────────────┐                                     │
│  │  RTBrick Switch Simulator       │ (port 12321)                        │
│  │  (switch-simulator-1)           │                                     │
│  │                                 │                                     │
│  │  Gin Router matches URL:        │                                     │
│  │  GET /api/v1/rbfs/elements/     │                                     │
│  │      :switchName/services/opsd/ │                                     │
│  │      proxy/bgp/peerings         │                                     │
│  │           │                     │                                     │
│  │           ▼                     │                                     │
│  │  handleListBGPPeering()         │                                     │
│  │  Returns dummy BGP peering JSON │                                     │
│  └──────────┬──────────────────────┘                                     │
│             │                                                            │
│             │ 7. Returns JSON response:                                  │
│             │    [{                                                      │
│             │      "instance_name": "default",                           │
│             │      "asn": 65000,                                         │
│             │      "router_id": "1.1.1.1",                               │
│             │      "peerings": {                                         │
│             │        "established_count": 1,                             │
│             │        "peerings": [{                                      │
│             │          "bgp_state": "ESTABLISHED",                       │
│             │          "ipv4_address": "198.51.100.42",                  │
│             │          ...                                               │
│             │        }]                                                  │
│             │      }                                                     │
│             │    }]                                                      │
│             ▼                                                            │
│  ┌─────────────────────────────────┐                                     │
│  │  Diagnosis Actor processes      │                                     │
│  │  the response, transforms it    │                                     │
│  │  into the diagnosis format,     │                                     │
│  │  returns to test via Kafka      │                                     │
│  └──────────┬──────────────────────┘                                     │
│             │                                                            │
│             │ 8. HTTP 200 + diagnosis result                             │
│             ▼                                                            │
│  Robot Test: validates status_code == 200                                │
│  Check_Status: iterates through bgpPeeringsList,                        │
│                logs each peer IP + state (ESTABLISHED)                   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 5. List BGP Peer — Complete End-to-End Flow

Let's trace **RBFS_ACTOR_TEST_004** step by step:

### The Test Code:
```robot
RBFS_ACTOR_TEST_004
    Create_Table
    ${resp}=    Update_Logical_Resource_Info    BLR1SPINE02    ${True}
    ${resp}=    Update_NEG_INFO    BLR1SPINE02
    ${output}=  Bgp_Peering_Request    ${BLR1SPINE02["id"]}
    Should Be Equal    "${output.status_code}"    "200"
    Check_Status    ${output.content}
```

### Step 1: `Create_Table`
- Copies `db_init.sql` into the PostgreSQL container (`postgresql-0`)
- Executes it to create the logical inventory tables
- This gives us a clean database for the test

### Step 2: `Update_Logical_Resource_Info BLR1SPINE02 ${True}`
- Looks up `BLR1SPINE02` from `variables.py`:
  ```python
  BLR1SPINE02 = {
      "ip_addr": "172.27.65.186",
      "hostname": "91_080_101_7KDB",
      "typename": "A4-SPINE-Switch-v2",
      "id": "91000000-0080-0000-0101-120000000000",
      ...
  }
  ```
- Because `${True}` is passed:
  - Gets simulator container IP: `docker inspect switch-simulator-1` → e.g., `172.19.0.15`
  - Overwrites `hostname` → `"switch-simulator-1"`
  - Overwrites `managementIpAddresses[0].address` → `"172.19.0.15"`
- Sends `PUT http://localhost:8080/v2/logicalResource/<id>` to register device in inventory
- Updates NEMS operational data with simulator hostname + IP

### Step 3: `Update_NEG_INFO BLR1SPINE02`
- Registers the Network Element Group (the "pod" that contains this switch)
- This is needed because the diagnosis actor validates that the device belongs to a group

### Step 4: `Bgp_Peering_Request ${BLR1SPINE02["id"]}`
Sends this HTTP request:
```
POST http://localhost:9080/diagnoses
Body: {
    "diagnosisType": "io.leitstand.actor.events.diagnosis.ListBgpPeeringsRequest",
    "diagnosisParameters": {
        "resourceId": "91000000-0080-0000-0101-120000000000"
    },
    "metadata": {
        "sourceObjectIdentifier": "91000000-0080-0000-0101-120000000000",
        "sourceObjectClass": "NetworkElement",
        "sourceObjectType": "A4-SPINE-Switch-v2"
    }
}
```

### Step 5: Diagnosis Actor processes the request
1. Reads `resourceId` from the request
2. Queries Logical Inventory: `GET http://a4-logical-inventory:8080/v2/logicalResource/<id>`
3. Gets back the device info with `hostname: switch-simulator-1` and `IP: 172.19.0.15`
4. Constructs the RBFS API URL:
   ```
   GET https://172.19.0.15:12321/api/v1/rbfs/elements/switch-simulator-1/services/opsd/proxy/bgp/peerings
   ```

### Step 6: Simulator receives the request
The Gin framework matches the URL pattern:
```go
getUrlListBgpPeering = "/api/v1/rbfs/elements/:switchName/services/opsd/proxy/bgp/peerings"
```
And routes it to `handleListBGPPeering()`.

### Step 7: Simulator returns dummy BGP data
```json
[{
    "instance_name": "default",
    "asn": 65000,
    "router_id": "1.1.1.1",
    "hostname": "switch-simulator",
    "peerings": {
        "active_count": 1,
        "established_count": 1,
        "peerings": [{
            "administrative_state": "up",
            "bgp_state": "ESTABLISHED",
            "asn": 65100,
            "ipv4_address": "198.51.100.42",
            "peering_type": "ebgp",
            "peer": {
                "ipv4_address": "198.51.100.42",
                "router_id": "2.2.2.2",
                "asn": 65200,
                "hostname": "peer-router"
            }
        }]
    }
}]
```

### Step 8: Test validates the response
```robot
Should Be Equal    "${output.status_code}"    "200"
Check_Status    ${output.content}
```
`Check_Status` parses the JSON, iterates through `bgpPeeringsList[0].instances`, and for each peering logs:
```
198.51.100.42 : ESTABLISHED
```

---

## 6. Simulator Code Structure

```
pkg/rtbrick/pkg/switchclient/
├── simulator.go                    # Core setup: creates server, registers ALL routes
├── simulator_ctrld_api.go          # Handlers for ctrld API (reboot, version, interfaces, ISIS, ping)
├── simulator_sim_api.go            # Handlers for simulator-specific APIs (version mgmt)
├── simulator_l2bsa_services.go     # Handlers for L2BSA service endpoints
├── simulator_l2bsa_subscribers.go  # Handlers for L2BSA subscriber endpoints
├── simulator_l2x_services.go       # Handlers for L2X service endpoints
├── simulator_test.go               # Unit tests
├── client.go                       # The real switch client (used in production)
├── models.go                       # Data structures (apiVersion, rtBrickJob, etc.)
├── urls.go                         # All URL constants + URL mapping

pkg/rtbrick/cmd/switch-simulator/
└── main.go                         # Entry point — starts simulator on port 12321
```

### Key Difference from Juniper:
In RTBrick, the simulator code lives **inside the `switchclient` package** (alongside the real client). In Juniper, it's in a separate `simulator` package.

### How Routes Are Registered (in `simulator.go`):
```go
func (s *Simulator) addLocalRoutes() *Simulator {
    // REST API routes — each URL gets its own handler
    s.Server.Engine.GET(getUrlListBgpPeering, s.handleListBGPPeering)
    s.Server.Engine.GET(urlGetBGPPeeringDetail, s.handleGetBGPPeeringDetail)
    s.Server.Engine.GET(urlGetBGPInstance, s.handleGetBGPInstance)
    s.Server.Engine.GET(urlGetPIMNeighbors, s.handleGetPIMNeighbors)
    s.Server.Engine.GET(urlISISNeighbors, s.handleListISISNeighbors)
    s.Server.Engine.GET(urlISISInterface, s.handleGetISISInterface)
    s.Server.Engine.POST(urlPOSTOpsdPing, s.pingRequest)
    // ... more routes
}
```

---

## 7. Deep Dive: handleListBGPPeering Handler

This is the handler that powers **RBFS_ACTOR_TEST_004**:

```go
// URL: GET /api/v1/rbfs/elements/:switchName/services/opsd/proxy/bgp/peerings
func (s *Simulator) handleListBGPPeering(c *gin.Context) {
    // Log which switch is being queried
    log.Infof("bgpPeeringRequest received: switchName: '%s'", c.Param("switchName"))

    // Read and discard request body (not needed for GET but good practice)
    body := c.Request.Body
    defer body.Close()
    io.ReadAll(body)

    // Build fake but realistic response
    response := []map[string]interface{}{
        {
            "instance_name": "default",
            "asn":           65000,
            "router_id":     "1.1.1.1",
            "hostname":      "switch-simulator",
            "peerings": map[string]interface{}{
                "active_count":      1,
                "established_count": 1,
                "peerings": []map[string]interface{}{
                    {
                        "bgp_state":    "ESTABLISHED",
                        "asn":          65100,
                        "ipv4_address": "198.51.100.42",
                        "peering_type": "ebgp",
                        "peer": map[string]interface{}{
                            "ipv4_address": "198.51.100.42",
                            "router_id":    "2.2.2.2",
                            "asn":          65200,
                            "hostname":     "peer-router",
                        },
                    },
                },
            },
        },
    }

    // Return as JSON with HTTP 200
    c.JSON(http.StatusOK, response)
}
```

### What makes this different from Juniper's BGP handler:

| Aspect | Juniper | RTBrick |
|---|---|---|
| How it's called | `POST /api/v1/rpc` with `"command": "show bgp neighbor"` | `GET /api/v1/rbfs/elements/:switchName/.../bgp/peerings` |
| Response format | Juniper XML-to-JSON structure with nested `DataField` arrays | Flat REST JSON with direct key-value pairs |
| Switch name | Not in URL (from inventory lookup) | Embedded in URL as `:switchName` parameter |
| Data model | Uses `systemops.BGPPeerDetail` structs | Uses `map[string]interface{}` (untyped) |

---

## 8. Currently Supported Simulator Endpoints

| URL Pattern | Handler | HTTP Method | Used By Test |
|---|---|---|---|
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/bgp/peerings` | `handleListBGPPeering` | GET | TEST_004 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/bgp/instances/:instance_name/peerings/:peerIP` | `handleGetBGPPeeringDetail` | GET | TEST_012 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/bgp/instances/:instance_name` | `handleGetBGPInstance` | GET | TEST_012 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/isis/neighbors` | `handleListISISNeighbors` | GET | TEST_006 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/isis/instances/:instance_name/interfaces/*ifl_name` | `handleGetISISInterface` | GET | TEST_013 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/pim/neighbors` | `handleGetPIMNeighbors` | GET | TEST_005, TEST_013 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/actions/ping` | `pingRequest` | POST | TEST_007 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/interfaces/physicals/*interfacename` | `handlePhysicalInterface` | GET | TEST_009, TEST_011 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/subscribers/statistics` | `handleGetShowSubscriberStatics` | GET/POST | TEST_014 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/ndo/port-info/:portName` | `handleNDOPortInfo` | GET | TEST_011 |
| `/api/v1/rbfs/elements/:switchName/services/opsd/proxy/transceivers/*portName` | `handleGetTransceivers` | GET | TEST_020 |
| `/api/v1/ctrld/system/_reboot` | `handleRebootSystem` | POST | TEST_010 |
| `/api/v1/ctrld/system/_reinstall` | `handleSystemReboot` | POST | TEST_010 |
| `/api/v1/ctrld/system/firmware` | `handleGetFwVersion` | GET | — |
| `/api/v1/ctrld/system/image` | `handleSystemImage` | GET | — |

---

## 9. How to Add a New Endpoint — Step-by-Step

Let's say you want to add: **`GET /api/v1/rbfs/elements/:switchName/services/opsd/proxy/route/summary`**

### Step 1: Add the URL constant in `urls.go`

```go
// In urls.go
const (
    // ... existing URLs ...
    urlGetRouteSummary = "/api/v1/rbfs/elements/:switchName/services/opsd/proxy/route/summary"
)
```

### Step 2: Write the handler in `simulator_ctrld_api.go` (or `simulator.go`)

```go
func (s *Simulator) handleGetRouteSummary(c *gin.Context) {
    switchName := c.Param("switchName")
    log.Infof("Route summary request received: switchName='%s'", switchName)

    response := map[string]interface{}{
        "instance_name": "default",
        "tables": []map[string]interface{}{
            {
                "table_name":   "inet.0",
                "active_routes": 15,
                "total_routes":  20,
            },
            {
                "table_name":   "inet6.0",
                "active_routes": 8,
                "total_routes":  10,
            },
        },
    }

    c.JSON(http.StatusOK, response)
}
```

### Step 3: Register the route in `simulator.go`'s `addLocalRoutes()`

```go
func (s *Simulator) addLocalRoutes() *Simulator {
    // ... existing routes ...

    // ADD THIS LINE:
    s.Server.Engine.GET(urlGetRouteSummary, s.handleGetRouteSummary)

    // ... rest of routes ...
}
```

### Step 4: Write a unit test in `simulator_test.go`

```go
func TestSimulator_HandleGetRouteSummary(t *testing.T) {
    s := NewSimulator("0.0.0.0:8080")
    require.NotNil(t, s)

    server := httptest.NewTLSServer(s.Server.Engine)
    defer server.Close()
    client := server.Client()

    req, _ := http.NewRequestWithContext(
        context.Background(),
        http.MethodGet,
        server.URL+"/api/v1/rbfs/elements/test-switch/services/opsd/proxy/route/summary",
        nil,
    )

    resp, err := client.Do(req)
    require.NoError(t, err)
    assert.Equal(t, http.StatusOK, resp.StatusCode)

    var response map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&response)
    require.NoError(t, err)
    resp.Body.Close()

    assert.Equal(t, "default", response["instance_name"])
    assert.NotNil(t, response["tables"])
}
```

### Step 5: Run tests
```bash
cd pkg/rtbrick/pkg/switchclient
go test -v ./...
```

### Step 6: Build & Deploy
```bash
go build -o artifacts/rtbrick-switch-simulator ./pkg/rtbrick/cmd/switch-simulator/
docker build -f pkg/rtbrick/build/rtbrick-switch-simulator.dockerfile -t rtbrick-switch-simulator:latest .
```

---

## 10. How to Write a Robot Test for It

### Step 1: Add keyword in `DiagnosisCommands.robot`
```robot
Route_Summary_Request
    [Arguments]     ${res_id}
    Log To Console  Sending Route Summary Request
    ${resp}=    POST    http://localhost:9080/diagnoses
    ...    {"diagnosisType": "io.leitstand.actor.events.diagnosis.RouteSummaryRequest",
    ...     "diagnosisParameters": {"resourceId": "${res_id}"},
    ...     "metadata": {"sourceObjectIdentifier":"${res_id}",
    ...                   "sourceObjectClass":"NetworkElement",
    ...                   "sourceObjectType":"A4-SPINE-Switch-v2"}}
    RETURN  ${resp}
```

### Step 2: Add test in `RtbrickDiagnosisTests.robot`
```robot
RBFS_ACTOR_TEST_XXX_ROUTE_SUMMARY
    [Documentation]    Verifies route summary diagnosis works with simulator
    Create_Table
    ${resp}=    Update_Logical_Resource_Info    BLR1SPINE02    ${True}
    ${resp}=    Update_NEG_INFO    BLR1SPINE02
    ${output}=  Route_Summary_Request    ${BLR1SPINE02["id"]}
    Should Be Equal    "${output.status_code}"    "200"
    Log To Console    Response: ${output.content}
```

---

## 11. Quick Reference Cheat Sheet

### Files You Touch When Adding a New RTBrick Endpoint:

| # | File | What to Do |
|---|---|---|
| 1 | `pkg/rtbrick/pkg/switchclient/urls.go` | Add URL constant |
| 2 | `pkg/rtbrick/pkg/switchclient/simulator_ctrld_api.go` | Add handler function |
| 3 | `pkg/rtbrick/pkg/switchclient/simulator.go` | Register route in `addLocalRoutes()` |
| 4 | `pkg/rtbrick/pkg/switchclient/simulator_test.go` | Add unit test |
| 5 | `pkg/lib/DiagnosisCommands.robot` | Add Robot keyword |
| 6 | `pkg/rtbrick/RtbrickDiagnosisTests.robot` | Add Robot test case with `${True}` |

### The Pattern for Every RTBrick Handler:

```go
func (s *Simulator) handleYourEndpoint(c *gin.Context) {
    switchName := c.Param("switchName")           // 1. Get switch name from URL
    log.Infof("Request received: %s", switchName) // 2. Log it

    response := map[string]interface{}{            // 3. Build fake response
        "key": "value",
    }

    c.JSON(http.StatusOK, response)                // 4. Return as JSON
}
```

### Docker Compose Entry:
```yaml
rtbrick-switch-simulator:
    image: rtbrick-switch-simulator:latest
    container_name: switch-simulator-1
    networks:
      actor_test:
    ports:
      - 127.0.0.1:12321:12321
```

### Key Containers in the Flow:

| Container | Role | Port |
|---|---|---|
| `a4-rbfs-diagnosis-actor` | Receives diagnosis requests, calls switch | 9080 → 2604 |
| `switch-simulator-1` | Fake RTBrick switch | 12321 |
| `a4-logical-inventory` | Stores device info (hostname, IP) | 8080 |
| `postgresql-0` | Database backend | 5432 |
| `a4-kafka` | Message bus for events | 9092 |

---

## Summary

```
RBFS_ACTOR_TEST_004 (List BGP Peer):

1. Create tables in DB
2. Register BLR1SPINE02 pointing to simulator (${True})
3. Register Network Element Group
4. Send "ListBgpPeeringsRequest" to Diagnosis Actor (port 9080)
5. Actor looks up device → finds simulator IP
6. Actor calls GET https://<sim-ip>:12321/.../bgp/peerings
7. Simulator returns fake BGP data (ESTABLISHED peer at 198.51.100.42)
8. Actor transforms response → returns to test
9. Test validates: status=200, peer state=ESTABLISHED ✅
```
