# 🎯 Juniper Switch Simulator — Complete Demo Guide

## Table of Contents

1. [What is the Switch Simulator?](#1-what-is-the-switch-simulator)
2. [Why Do We Need It?](#2-why-do-we-need-it)
3. [The ${True} Flag — What It Does](#3-the-true-flag--what-it-does)
4. [Architecture Overview](#4-architecture-overview)
5. [How It Works — End-to-End Flow](#5-how-it-works--end-to-end-flow)
6. [Simulator Code Structure](#6-simulator-code-structure)
7. [Currently Supported Endpoints](#7-currently-supported-endpoints)
8. [How to Add a New Endpoint — Step-by-Step](#8-how-to-add-a-new-endpoint--step-by-step)
9. [How to Write a Test for the New Endpoint](#9-how-to-write-a-test-for-the-new-endpoint)
10. [Quick Reference Cheat Sheet](#10-quick-reference-cheat-sheet)

---

## 1. What is the Switch Simulator?

Think of the **Switch Simulator** as a **fake Juniper switch** that runs inside a Docker container.

In production, the diagnosis actor talks to a **real physical Juniper switch** (e.g., BLR3SPINE02 at IP 172.27.65.236) over HTTPS and sends commands like show isis adjacency, show bgp neighbor, ping, etc.

But during **automated testing**, we don't have a real switch available. So we built a **Go program** that:

- Listens on HTTPS (port 443) just like a real Juniper switch
- Accepts the same JSON-RPC commands (POST /api/v1/rpc)
- Returns **realistic fake responses** (dummy data that looks exactly like what a real switch would return)

> **In simple words:** The Switch Simulator pretends to be a real Juniper switch so that we can run tests without needing real hardware.

---

## 2. Why Do We Need It?

| Without Simulator | With Simulator |
|---|---|
| Need a real physical Juniper switch | No hardware needed |
| Tests can only run in the lab | Tests run anywhere (CI/CD pipelines, dev machines) |
| Switch might be down or busy | Simulator is always available |
| Test results depend on real switch state | Simulator returns consistent, predictable data |
| Slow — SSH/network latency | Fast — everything runs locally in Docker |

---

## 3. The ${True} Flag — What It Does

When you see this in the test cases:

    ${resp}=    Update_Logical_Resource_Info    BLR3SPINE02    ${True}

The ${True} is the **bool1 parameter** that tells the system: **"Use the Switch Simulator instead of the real switch."**

### What happens inside Update_Logical_Resource_Info when ${True} is passed:

    Step 1: Get the Docker container IP of "switch-simulator-1"
             └─> runs: docker inspect switch-simulator-1 → gets e.g., 172.18.0.15

    Step 2: Update the device's "hostname" to "switch-simulator-1"
             └─> instead of real hostname like "BLR3SPINE02"

    Step 3: Update the device's "managementIpAddresses" to the container IP
             └─> instead of real IP like 172.27.65.236

    Step 4: Store this updated info in the Logical Inventory database

### The result:

| Field | When ${True} (Simulator) | When ${False} (Real Switch) |
|---|---|---|
| hostname | switch-simulator-1 | BLR3SPINE02 |
| IP Address | Docker container IP (e.g., 172.18.0.15) | Real switch IP (172.27.65.236) |

So when the **Diagnosis Actor** later tries to run a diagnosis, it looks up the device info from the inventory and connects to the **simulator container** instead of the **real switch**.

---

## 4. Architecture Overview

    ┌──────────────────────────────────────────────────────────────────────┐
    │                        TEST EXECUTION FLOW                           │
    │                                                                      │
    │  Robot Test Case                                                     │
    │  (JuniperDiagnosisTests.robot)                                       │
    │       │                                                              │
    │       │ 1. Update_Logical_Resource_Info(BLR3SPINE02, ${True})        │
    │       │    └─> Registers device with simulator's IP in inventory     │
    │       │                                                              │
    │       │ 2. POST http://localhost:9080/diagnoses  (diagnosis request) │
    │       ▼                                                              │
    │  ┌─────────────────────────┐                                         │
    │  │  Juniper Diagnosis Actor│ (port 9080)                             │
    │  │  (a4-juniper-diagnosis- │                                         │
    │  │   actor container)      │                                         │
    │  └──────────┬──────────────┘                                         │
    │             │                                                        │
    │             │ 3. Looks up device IP from Logical Inventory           │
    │             │    → Gets simulator's container IP                     │
    │             │                                                        │
    │             │ 4. POST https://<simulator-ip>:443/api/v1/rpc          │
    │             │    Body: {"command": "show isis adjacency",            │
    │             │           "format": "json"}                            │
    │             ▼                                                        │
    │  ┌─────────────────────────┐                                         │
    │  │  Switch Simulator       │ (port 443, container: switch-simulator-1│
    │  │  (juniper-switch-       │                                         │
    │  │   simulator container)  │                                         │
    │  │                         │                                         │
    │  │  Receives the command,  │                                         │
    │  │  returns dummy JSON     │                                         │
    │  │  response               │                                         │
    │  └──────────┬──────────────┘                                         │
    │             │                                                        │
    │             │ 5. Returns fake but realistic JSON response             │
    │             ▼                                                        │
    │  ┌─────────────────────────┐                                         │
    │  │  Diagnosis Actor        │                                         │
    │  │  processes the response │                                         │
    │  │  and sends result back  │                                         │
    │  └──────────┬──────────────┘                                         │
    │             │                                                        │
    │             │ 6. HTTP 200 + diagnosis result                         │
    │             ▼                                                        │
    │  Robot Test Case validates the response                              │
    │  (checks status_code == 200, validates JSON structure)               │
    └──────────────────────────────────────────────────────────────────────┘

---

## 5. How It Works — End-to-End Flow

Let's trace **JUNIPER_TEST_006** (ISIS Adjacency Check) as a concrete example:

### Step 1: Test sets up the database

    Create_Table                                          # Creates tables in PostgreSQL
    ${resp}= Update_Logical_Resource_Info BLR3SPINE02 ${True}  # Registers device pointing to simulator
    ${resp}= Update_NEG_Info BLR3SPINE02                  # Adds Network Element Group info

### Step 2: Test sends a diagnosis request

    ${output}= ISIS_Adjacency_Check ${BLR3SPINE02["id"]}

This calls:

    POST http://localhost:9080/diagnoses
    Body: {
      "diagnosisType": "io.leitstand.actor.events.diagnosis.IsisAdjacencyRequest",
      "diagnosisParameters": {"resourceId": "91000000-0080-0000-0103-000000000012"},
      "metadata": {"sourceObjectType": "A4-SPINE-Switch-v2"}
    }

### Step 3: Diagnosis Actor looks up device in inventory
- Finds hostname: switch-simulator-1 and IP: <container_ip>

### Step 4: Diagnosis Actor sends JSON-RPC command to simulator

    POST https://<container_ip>:443/api/v1/rpc
    Body: {"command": "show isis adjacency", "format": "json"}

### Step 5: Simulator receives request and returns fake data

The simulator's handleRPC() method routes the command:

    "show isis adjacency" → handleISISAdjRPC() → handleISISAdjacency()

Returns dummy ISIS adjacency entries with interfaces like et-0/1/31.200, states like Up, etc.

### Step 6: Test validates the response

    Should Be Equal "${output.status_code}" "200"
    ISIS_Adjacency_List ${output.content}               # Validates JSON structure

---

## 6. Simulator Code Structure

    pkg/juniper/pkg/simulator/
    ├── simulator.go              # Core setup: creates server, adds routes, generates TLS cert
    ├── simulator_handlers.go     # All endpoint handler functions (the main logic)
    └── simulator_test.go         # Unit tests for each handler

    pkg/juniper/cmd/switch-simulator/
    └── main.go                   # Entry point — starts the simulator on port 443

### Key Files Explained:

| File | Purpose |
|---|---|
| simulator.go | Creates the HTTP server with TLS, defines the Simulator struct, registers routes |
| simulator_handlers.go | Contains all handler functions — one per Juniper command (ISIS, BGP, Ping, etc.) |
| simulator_test.go | Unit tests that verify each handler returns correct JSON responses |
| main.go | Entry point — reads env vars and starts the simulator |

### The Simulator Struct:

```go
type Simulator struct {
    configs         map[string]interface{}              // General configs
    versions        map[string]string                   // Version info
    isisNeighbors   map[string][]IsisInstanceNeighbors  // ISIS neighbor data
    pimNeighbors    map[string][]PimNeighbors           // PIM neighbor data
    interfaces      map[string]PhysicalInterface        // Interface counter data
    lock            *sync.RWMutex                       // Thread-safe access
    Server          *httpgin.GinServer                  // The HTTP server (Gin framework)
}
```

### The Single RPC Endpoint:

All commands go through **one single route**: POST /api/v1/rpc

The handleRPC() function acts as a **router** — it reads the "command" field from the JSON body and dispatches to the right handler:

```go
func (s *Simulator) handleRPC(c *gin.Context) {
    // Parse JSON body → get "command" string

    if contains(command, "show interfaces diagnostics optics") → handleInterfaceOptics()
    if startsWith(command, "show interfaces extensive")       → handleInterfaceCommands()
    if contains(command, "show isis interface")                → handleISISInterface()
    if contains(command, "show isis adjacency")                → handleISISAdjRPC()
    if contains(command, "show pim neighbor")                  → handlePIMNeighbors()

    // If nothing matches → 404
}
```

---

## 7. Currently Supported Endpoints

| Command | Handler Function | Used By Test |
|---|---|---|
| show isis adjacency | handleISISAdjacency() | JUNIPER_TEST_006 |
| show isis interface | handleISISInterface() | JUNIPER_TEST_013 (ISIS metrics) |
| show bgp neighbor | handleBGPState() | JUNIPER_TEST_012 |
| show pim neighbor | handlePIMNeighbors() | JUNIPER_TEST_005, TEST_013 (PIM) |
| show interfaces extensive <name> | handleInterfaceCommands() | JUNIPER_TEST_009, TEST_011 |
| clear interfaces statistics <name> | handleInterfaceCommands() | JUNIPER_TEST_009 |
| show interfaces diagnostics optics <name> | handleInterfaceOptics() | JUNIPER_TEST_020 |
| ping routing-instance ... | handleExecutePing() | JUNIPER_TEST_007 |
| request system reboot ... | handleNeReboot() | JUNIPER_TEST_010 |

---

## 8. How to Add a New Endpoint — Step-by-Step

Let's say you want to add support for a new Juniper command: **show route summary**

### Step 1: Define the response model (if needed)

If the response structure doesn't already exist in the systemops package, create a Go struct that matches what a real Juniper switch returns:

```go
type RouteSummaryResponse struct {
    RouteSummaryInformation []struct {
        RoutingTable []RoutingTable `+"`"+`json:"route-table"`+"`"+`
    } `+"`"+`json:"route-summary-information"`+"`"+`
}

type RoutingTable struct {
    TableName    []DataField `+"`"+`json:"table-name"`+"`"+`
    ActiveRoutes []DataField `+"`"+`json:"active-route-count"`+"`"+`
    TotalRoutes  []DataField `+"`"+`json:"total-route-count"`+"`"+`
}
```

### Step 2: Add the handler function in simulator_handlers.go

```go
// handleRouteSummary handles "show route summary" command
func (s *Simulator) handleRouteSummary(c *gin.Context) {
    response := RouteSummaryResponse{
        RouteSummaryInformation: []struct {
            RoutingTable []RoutingTable
        }{
            {
                RoutingTable: []RoutingTable{
                    {
                        TableName:    []DataField{{Data: "inet.0"}},
                        ActiveRoutes: []DataField{{Data: "15"}},
                        TotalRoutes:  []DataField{{Data: "20"}},
                    },
                    {
                        TableName:    []DataField{{Data: "inet6.0"}},
                        ActiveRoutes: []DataField{{Data: "8"}},
                        TotalRoutes:  []DataField{{Data: "10"}},
                    },
                },
            },
        },
    }

    c.JSON(http.StatusOK, response)
}
```

### Step 3: Register the command in the handleRPC router

Open simulator_handlers.go and add a new route in the handleRPC() function:

```go
func (s *Simulator) handleRPC(c *gin.Context) {
    // ... existing code ...

    // ADD THIS: Route to route summary handler
    if strings.Contains(command, "show route summary") {
        s.handleRouteSummary(c)
        return
    }

    // ... rest of existing routes ...
}
```

> **Important:** Place more specific commands **before** less specific ones. For example, "show isis interface" should come before "show isis adjacency" since both contain "isis".

### Step 4: Write a unit test in simulator_test.go

```go
func TestSimulator_HandleJSONRPC_ShowRouteSummary(t *testing.T) {
    s := NewSimulator("0.0.0.0:8080")
    require.NotNil(t, s)

    server := httptest.NewTLSServer(s.Server.Engine)
    defer server.Close()
    client := server.Client()

    rpcRequest := map[string]string{
        "command": "show route summary",
        "format":  "json",
    }
    body, _ := json.Marshal(rpcRequest)
    req, _ := http.NewRequestWithContext(
        context.Background(),
        http.MethodPost,
        server.URL+"/api/v1/rpc",
        bytes.NewReader(body),
    )
    req.Header.Set("Content-Type", "application/json")

    resp, err := client.Do(req)
    require.NoError(t, err)
    assert.Equal(t, http.StatusOK, resp.StatusCode)

    var response map[string]interface{}
    err = json.NewDecoder(resp.Body).Decode(&response)
    require.NoError(t, err)
    resp.Body.Close()

    assert.NotNil(t, response["route-summary-information"])
}
```

### Step 5: Run the tests

    cd pkg/juniper/pkg/simulator
    go test -v ./...

### Step 6: Build and update the Docker image

    # Build the simulator binary
    go build -o artifacts/juniper-switch-simulator ./pkg/juniper/cmd/switch-simulator/

    # Build the Docker image
    docker build -f pkg/juniper/build/juniper-switch-simulator.dockerfile -t juniper-switch-simulator:latest .

---

## 9. How to Write a Test for the New Endpoint

Once the simulator endpoint is ready, you write a Robot test case in JuniperDiagnosisTests.robot:

### Step 1: Add the keyword in DiagnosisCommands.robot

    Route_Summary_Request
        [Arguments]     ${res_id}
        Log To Console  Sending Route Summary Request
        ${resp}=    POST    http://localhost:9080/diagnoses    {"diagnosisType": "...", ...}
        RETURN  ${resp}

### Step 2: Add the test case in JuniperDiagnosisTests.robot

    JUNIPER_TEST_XXX_ROUTE_SUMMARY
        [Documentation]    This test case verifies the route summary diagnosis
        Create_Table
        ${resp}=    Update_Logical_Resource_Info    BLR3SPINE02    ${True}
        ${resp}=    Update_NEG_Info    BLR3SPINE02
        ${output}=    Route_Summary_Request    ${BLR3SPINE02["id"]}
        Should Be Equal    "${output.status_code}"    "200"
        Log To Console    The response code : ${output.status_code}

---

## 10. Quick Reference Cheat Sheet

### Files You'll Touch When Adding a New Endpoint:

| # | File | What to Do |
|---|---|---|
| 1 | pkg/juniper/pkg/simulator/simulator_handlers.go | Add handler function + register in handleRPC() |
| 2 | pkg/juniper/pkg/simulator/simulator_test.go | Add unit test for the handler |
| 3 | pkg/lib/DiagnosisCommands.robot | Add Robot keyword to call the diagnosis API |
| 4 | pkg/juniper/JuniperDiagnosisTests.robot | Add Robot test case using ${True} |

### The Pattern for Every Handler:

```go
func (s *Simulator) handleYourCommand(c *gin.Context) {
    // 1. Build a response struct with dummy/realistic data
    response := SomeResponseStruct{
        // ... fill with fake but realistic values
    }
    // 2. Return it as JSON with HTTP 200
    c.JSON(http.StatusOK, response)
}
```

### Key Environment Variables:

| Variable | Default | Description |
|---|---|---|
| SWITCH_CLIENT_SIM_ADDRESS | 0.0.0.0:8080 | Address the simulator listens on |
| SIM_TLS_CERT | /tmp/server-cert.pem | TLS certificate path |
| SIM_TLS_KEY | /tmp/server-key.pem | TLS key path |

### Docker Compose Entry:

```yaml
juniper-switch-simulator:
    image: juniper-switch-simulator:latest
    container_name: switch-simulator-1
    environment:
      - SWITCH_CLIENT_SIM_ADDRESS=0.0.0.0:443
    networks:
      actor_test:
```

---

## Summary

    ${True}  = "Hey, use the simulator, not the real switch!"
    ${False} = "Connect to the real hardware switch"

    The simulator is a Go HTTP server that:
      1. Listens on HTTPS (like a real Juniper switch)
      2. Accepts JSON-RPC commands at POST /api/v1/rpc
      3. Routes commands to handler functions
      4. Returns realistic fake JSON responses
      5. Enables fully automated testing without hardware
