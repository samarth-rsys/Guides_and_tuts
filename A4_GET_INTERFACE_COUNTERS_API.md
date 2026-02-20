# A4GetInterfaceCounters Diagnosis API - Detailed Guide

## Table of Contents
1. [Overview](#overview)
2. [API Endpoint](#api-endpoint)
3. [Complete Code Flow](#complete-code-flow)
4. [Step-by-Step Breakdown](#step-by-step-breakdown)
5. [Code Examples](#code-examples)
6. [Response Structure](#response-structure)
7. [Error Scenarios](#error-scenarios)
8. [Testing](#testing)

---

## Overview

The `A4GetInterfaceCounters` diagnosis API retrieves network interface statistics (counters) for a specific network element port. This includes transmitted/received bytes, packets, errors, drops, and multicast traffic.

### Use Cases
- Monitor interface traffic
- Diagnose network issues
- Reset interface counters for debugging
- Collect performance metrics

### Key Features
- ✅ Retrieves TX/RX statistics
- ✅ Optional counter reset
- ✅ Works on NetworkElementPort resources
- ✅ Supports both synchronous and asynchronous processing

---

## API Endpoint

### HTTP Request

```http
POST http://localhost:2604/diagnoses
Content-Type: application/json
```

### Request Body

```json
{
  "diagnosisType": "io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest",
  "diagnosisParameters": {
    "resourceId": "port-uuid-12345-6789-abcd",
    "resetCounters": false
  },
  "metadata": {
    "sourceObjectIdentifier": "port-uuid-12345-6789-abcd",
    "sourceObjectClass": "NetworkElementPort",
    "sourceObjectType": "A4-Port"
  }
}
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `diagnosisType` | string | Yes | Must be `io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest` |
| `diagnosisParameters.resourceId` | string (UUID) | Yes | UUID of the NetworkElementPort |
| `diagnosisParameters.resetCounters` | boolean | Yes | `true` to reset counters after reading, `false` to just read |
| `metadata.sourceObjectIdentifier` | string (UUID) | Yes | Same as resourceId |
| `metadata.sourceObjectClass` | string | Yes | Must be `NetworkElementPort` |
| `metadata.sourceObjectType` | string | No | Optional port type information |

### Response Body (Success)

```json
{
  "id": "request-uuid-9876-5432-efgh",
  "responseData": {
    "diagnosisResult": {
      "resetCounter": false,
      "receivedBytes": 1234567890,
      "receivedPackets": 9876543,
      "receivedMulticast": 12345,
      "receivedDropped": 0,
      "receivedErrors": 0,
      "sentBytes": 9876543210,
      "sentPackets": 12345678,
      "sentMulticast": 54321,
      "sentDropped": 0,
      "sentErrors": 0
    }
  }
}
```

### Response Body (Error)

```json
{
  "message": "NDO Interface Counters diagnosis is only supported for NetworkElementPort class, not for: NetworkElement"
}
```

---

## Complete Code Flow

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         HTTP Client                             │
│                                                                 │
│  POST /diagnoses                                                │
│  { diagnosisType, diagnosisParameters, metadata }              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 1. HTTP Request
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│            pkg/resources/diagnoses.go                           │
│            runDiagnosis(w, r)                                   │
│                                                                 │
│  • Deserialize JSON request                                    │
│  • Extract diagnosisType and parameters                        │
│  • Validate metadata                                           │
│  • Publish to NATS: diagnosis-validated-requests               │
│  • Wait for response (60s timeout)                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 2. NATS Message
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│            pkg/fixtures/diagnosis/diagnosis.go                  │
│            registerDiagnosisHandlers()                          │
│                                                                 │
│  • Message arrives from NATS                                   │
│  • Pipeline routes based on diagnosisType                      │
│  • Matches: A4GetInterfaceCountersDiagnosisRequest_1           │
│  • Invokes: commands.NDOInterfaceCounters                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 3. Handler Invocation
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  pkg/fixtures/diagnosis/ndo_get_interface_counters.go           │
│  NDOInterfaceCounters(actorCtx, msg)                            │
│                                                                 │
│  STEP 1: Early Validation                                       │
│  • Check if msg is nil                                         │
│  • Validate sourceObjectClass = NetworkElementPort             │
│                                                                 │
│  STEP 2: Routing Decision                                      │
│  • If enableRouting: processAsyncDiagnosis() ──┐              │
│  • Else: ndoInterfaceCountersSync()            │              │
└────────────────────────┬────────────────────────┼───────────────┘
                         │                        │
                         │ Sync Path              │ Async Path
                         │                        │ (Optional)
                         │                        │
                         │                        ▼
                         │              ┌─────────────────────┐
                         │              │ Goroutine spawned   │
                         │              │ Concurrency limited │
                         │              │ (max 50 parallel)   │
                         │              └──────────┬──────────┘
                         │                         │
                         ▼◄────────────────────────┘
┌─────────────────────────────────────────────────────────────────┐
│  ndoInterfaceCountersSync(actorCtx, msg)                        │
│                                                                 │
│  STEP 3: Parse Request                                          │
│  • Cast msg to A4GetInterfaceCountersDiagnosisRequest          │
│  • Extract: resourceId, resetCounters                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 4. Inventory Lookup
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: Fetch Switch by Port                                   │
│  • cmds.fetchSwitchByPort(resourceId)                           │
│    ├─ Get Port info from inventory                             │
│    ├─ Get parent NetworkElement (switch)                       │
│    └─ Return: a4Spine, a4Port                                  │
│                                                                 │
│  Returns:                                                       │
│  • a4Spine: The parent switch/network element                  │
│  • a4Port: The port itself                                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 5. Port Mapping
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 5: Get Port Mapping                                       │
│  • Extract functionalPortLabel from port                       │
│  • cmds.inventory.GetPortMapping()                             │
│    ├─ NETypeName: e.g., "BNG-LEAF"                            │
│    ├─ PlannedMatNumber: e.g., "12345"                         │
│    └─ functionalPortLabel: e.g., "X1"                         │
│                                                                 │
│  Returns:                                                       │
│  • portMapping.CliLabel: e.g., "ifp-0/1/2"                    │
│    (This is the physical interface name in RBFS)               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 6. Authorization
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 6: Prepare Authorization                                  │
│  • cmds.prepareAuthorization(ctx)                              │
│                                                                 │
│  If Auth Service Active:                                        │
│    • Get JWT token from auth service                           │
│    • port = "12321"                                            │
│    • protocol = "https"                                        │
│  Else:                                                          │
│    • token = ""                                                │
│    • port = "19091"                                            │
│    • protocol = "http"                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 7. Management IP
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 7: Get Management IP Address                              │
│  • Get extended operational data from a4Spine                  │
│  • Extract hostname                                            │
│  • Extract mgmtIP from operational data                        │
│                                                                 │
│  Example:                                                       │
│  • hostname = "leaf-01-berlin"                                 │
│  • mgmtIP = "192.168.1.10"                                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 8. RBFS Context
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 8: Create RBFS Context                                    │
│  • Build endpoint URL:                                         │
│    https://192.168.1.10:12321                                  │
│  • Create RbfsContext with:                                    │
│    ├─ timeout: 30 seconds                                      │
│    ├─ endpoint: URL                                            │
│    ├─ hostname: "leaf-01-berlin"                              │
│    └─ token: JWT (if auth enabled)                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 9. Reset Counters (Optional)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 9: Reset Counters (if requested)                          │
│  • If notification.ResetCounters == true:                      │
│    └─ cmds.rbfs.ClearInterfaceCounters(rbfsContext, "ifp-0/1/2")│
│                                                                 │
│  This makes HTTP call to RBFS:                                  │
│  DELETE https://192.168.1.10:12321/api/v1/interfaces/ifp-0/1/2/counters
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 10. Get Interface Details
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 10: Get Physical Interface                                │
│  • cmds.rbfs.GetPhysicalInterface(rbfsContext, "ifp-0/1/2")   │
│                                                                 │
│  HTTP Call:                                                     │
│  GET https://192.168.1.10:12321/api/v1/interfaces/physical/ifp-0/1/2
│  Authorization: Bearer <jwt-token>                             │
│                                                                 │
│  Returns: state.PhysicalInterfaceDetail                        │
│  • IfpCounters.Rx (receive stats)                             │
│  • IfpCounters.Tx (transmit stats)                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 11. Process Response
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 11: Build Diagnosis Response                              │
│  • Extract counter values from ifp.IfpCounters                 │
│  • Map to DiagnosisResult structure:                           │
│    ├─ ReceivedBytes     = Rx.BytesReceived                     │
│    ├─ ReceivedPackets   = Rx.PacketsReceived                   │
│    ├─ ReceivedMulticast = Rx.MulticastPacketsReceived          │
│    ├─ ReceivedDropped   = Rx.PacketsDropped                    │
│    ├─ ReceivedErrors    = Rx.PacketsDropped                    │
│    ├─ SentBytes         = Tx.BytesSent                         │
│    ├─ SentPackets       = Tx.PacketsSent                       │
│    ├─ SentMulticast     = Tx.MulticastPacketsSent              │
│    ├─ SentDropped       = Tx.PacketsDropped                    │
│    └─ SentErrors        = Tx.PacketsDropped                    │
│                                                                 │
│  • Create A4GetInterfaceCountersDiagnosisResponse              │
│    ├─ ReturnCode: OK                                           │
│    └─ DiagnosisResult: <counters>                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 12. Return Response
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Return []actorAPI.RawMessage{&a4Response}                     │
│                                                                 │
│  • Response published to NATS: diagnosis-responses             │
│  • HTTP layer receives response                                │
│  • Returns JSON to client                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Breakdown

### Step 1: Early Validation

**File**: `pkg/fixtures/diagnosis/ndo_get_interface_counters.go`  
**Function**: `NDOInterfaceCounters()`

```go
func (cmds *Commands) NDOInterfaceCounters(actorCtx actorAPI.ActorContext, msg actorAPI.RawMessage) ([]actorAPI.RawMessage, actorAPI.LayerError) {
    // Validation 1: Check nil message
    if msg == nil {
        logger.Errorf("NDO Interface Counters validation failed: request message is nil")
        return []actorAPI.RawMessage{&ndo.A4GetInterfaceCountersDiagnosisResponse{
            ReturnCode: ndo.DiagReturnCode{
                Code:            ndo.RetCodeValueBAD_REQUEST,
                CodeDescription: "Validation failed: request message is nil",
            },
        }}, nil
    }

    // Validation 2: Check source object class
    sourceObjectClass := actorCtx.OriginalTask().Metadata.SourceObjectClass
    if sourceObjectClass != string(cliApi.NetworkElementPort) {
        errorMsg := fmt.Sprintf("NDO Interface Counters diagnosis is only supported for NetworkElementPort class, not for: %s", sourceObjectClass)
        logger.Errorf("NDO Interface Counters validation failed: %s", errorMsg)
        return []actorAPI.RawMessage{&ndo.A4GetInterfaceCountersDiagnosisResponse{
            ReturnCode: ndo.DiagReturnCode{
                Code:            ndo.RetCodeValueBAD_REQUEST,
                CodeDescription: errorMsg,
            },
        }}, nil
    }
    
    // ... continues to routing decision
}
```

**Why This Matters**:
- Prevents null pointer exceptions
- Ensures diagnosis only runs on correct resource types (ports, not switches)
- Fails fast with clear error messages

---

### Step 2: Routing Decision (Sync vs Async)

```go
// Check if parallel processing is enabled
if cmds.enableRouting {
    return cmds.processAsyncDiagnosis(actorCtx, msg, "NDOInterfaceCounters", cmds.ndoInterfaceCountersSync)
}
return cmds.ndoInterfaceCountersSync(actorCtx, msg)
```

**Async Processing** (when `ENABLE_ROUTING_LOGIC=true`):
- Spawns a goroutine
- Limits concurrency (default: max 50 parallel diagnosis)
- Tracks active diagnosis in sync.Map
- Returns immediately, processes in background

**Sync Processing** (default):
- Blocks until completion
- Simpler, easier to debug
- Good for low-traffic scenarios

---

### Step 3: Parse Request

```go
func (cmds *Commands) ndoInterfaceCountersSync(actorCtx actorAPI.ActorContext, msg actorAPI.RawMessage) ([]actorAPI.RawMessage, actorAPI.LayerError) {
    ctx := fixtures.NewContext(actorCtx)
    notification := msg.(*ndo.A4GetInterfaceCountersDiagnosisRequest)
    logger.Infof("A4GetInterfaceCountersRequest: %v", notification)
    
    // notification now contains:
    // - ResourceId: "port-uuid-12345"
    // - ResetCounters: false
}
```

**Type Assertion**:
- The generic `RawMessage` is cast to specific request type
- This gives us access to typed fields

---

### Step 4: Fetch Switch by Port

```go
a4Spine, a4Port, a4Err := cmds.fetchSwitchByPort(ctx.ActorContext, notification.ResourceId)
if a4Err != nil {
    logger.Errorf("error while fetching switch for port %v, error %v", notification.ResourceId, a4Err)
    a4Response := ndo.A4GetInterfaceCountersDiagnosisResponse{
        ReturnCode: ndo.DiagReturnCode{
            Code:            ndo.RetCodeValueINTERNAL_ERROR,
            CodeDescription: fmt.Sprintf(ErrorInternalCannotLookupNetworkElement, a4Err),
        },
    }
    return []actorAPI.RawMessage{&a4Response}, nil
}
```

**What `fetchSwitchByPort` does**:

```go
func (cmds *Commands) fetchSwitchByPort(ctx actorAPI.ActorContext, portID string) (*inventoryAPI.CLILogicalResource, *inventoryAPI.CLILogicalResource, actorAPI.LayerError) {
    // 1. Get port from inventory
    port, err := cmds.inventory.FetchNetworkElementPortByID(ctx, portID)
    if err != nil {
        return nil, nil, err
    }

    // 2. Find parent element (the switch)
    elementRelation := parentElementRelationShip(port.ResourceRelationship)
    if elementRelation == nil {
        err = actorAPI.NewLayerError(int(errors.ManagementErrorsLogicalResourceNotFound), fmt.Errorf("cannot resolve parent element for port %v", portID))
        return nil, nil, err
    }

    // 3. Fetch the parent switch
    element, err := cmds.inventory.FetchFabricSwitch(ctx, elementRelation.ResourceRef.ID)
    
    return element, port, err
}
```

**Returns**:
- `a4Spine`: The parent switch (contains hostname, IP, type, etc.)
- `a4Port`: The port itself (contains characteristics like functional label)

---

### Step 5: Get Port Mapping

```go
functionalPortLabel := a4Port.GetCharacteristicValue("functionalPortLabel")

portMapping, a4Err := cmds.inventory.GetPortMapping(
    inventory.NETypeName(a4Spine),
    inventory.PlannedMatNumber(a4Spine),
    functionalPortLabel,
)
if a4Err != nil {
    a4Response := ndo.A4GetInterfaceCountersDiagnosisResponse{
        ReturnCode: ndo.DiagReturnCode{
            Code:            ndo.RetCodeValueINTERNAL_ERROR,
            CodeDescription: fmt.Sprintf(ErrorInternalCannotLookupPortMapping, a4Err),
        },
    }
    return []actorAPI.RawMessage{&a4Response}, nil
}
```

**Why Port Mapping?**

- Inventory stores ports with logical names (e.g., "X1", "Port 1")
- RBFS uses physical interface names (e.g., "ifp-0/1/2")
- Port mapping translates: `functionalPortLabel → CLI Label`

**Example**:
- Input: `functionalPortLabel = "X1"`
- Output: `portMapping.CliLabel = "ifp-0/1/2"`

---

### Step 6: Prepare Authorization

```go
token, port, protocol, a4Err := cmds.prepareAuthorization(ctx)
if a4Err != nil {
    a4Response := &ndo.A4GetInterfaceCountersDiagnosisResponse{
        ReturnCode: ndo.DiagReturnCode{
            Code:            ndo.RetCodeValueINTERNAL_ERROR,
            CodeDescription: fmt.Sprintf(ErrorInternalCannotCreateRBFSToken, cmds.newElementError(a4Spine, a4Err)),
        },
    }
    return []actorAPI.RawMessage{&a4Response}, nil
}
```

**Implementation** (`pkg/fixtures/diagnosis/diagnosis.go`):

```go
func (cmds *Commands) prepareAuthorization(ctx *fixtures.Context) (token, port, protocol string, err actorAPI.LayerError) {
    token = ""
    port = "19091"  // Default HTTP port
    protocol = "http"
    
    if cmds.authService.IsActive() {
        // Authentication enabled
        token, err = cmds.authService.GetToken(ctx.ActorContext)
        port = "12321"  // HTTPS port
        protocol = "https"
        if err != nil {
            logger.Errorf("failed to get token from auth service with error %v", err)
            return "", "", "", err
        }
    }
    return token, port, protocol, nil
}
```

**Two Modes**:
1. **With Auth** (production): `https://switch-ip:12321` + JWT token
2. **Without Auth** (development): `http://switch-ip:19091` + no token

---

### Step 7: Get Management IP

```go
timeout, cancel := context.WithTimeout(ctx.ActorContext, 30*time.Second)
defer cancel()

a4spineoperationaldata, ok := a4Spine.GetExtendedData()
if !ok {
    logger.Warnf("failed to get operational data for nep %v", a4Spine.ID)
}

hostname := inventory.Hostname(a4spineoperationaldata)

mgmtIP, err := inventory.ManagementIPAddress(a4spineoperationaldata)
if err != nil {
    a4Response := &ndo.A4GetInterfaceCountersDiagnosisResponse{
        ReturnCode: ndo.DiagReturnCode{
            Code:            ndo.RetCodeValueINTERNAL_ERROR,
            CodeDescription: fmt.Sprintf(ErrorInternalCannotGetMgmtAddress, cmds.newElementError(a4Spine, err)),
        },
    }
    return []actorAPI.RawMessage{&a4Response}, nil
}
```

**Extracts**:
- `hostname`: e.g., `"leaf-01-berlin"`
- `mgmtIP`: e.g., `"192.168.1.10"` (in-band management IP)

---

### Step 8: Create RBFS Context

```go
endpoint, err := url.Parse(fmt.Sprintf("%s://%s:%s", protocol, mgmtIP, port))
if err != nil {
    a4Response := ndo.A4GetInterfaceCountersDiagnosisResponse{
        ReturnCode: ndo.DiagReturnCode{
            Code:            ndo.RetCodeValueINTERNAL_ERROR,
            CodeDescription: fmt.Sprintf(ErrorInternalCannotComputeRBFSEndpoint, cmds.newElementError(a4Spine, err)),
        },
    }
    return []actorAPI.RawMessage{&a4Response}, nil
}

rbfsContext, err := commons.NewRbfsContext(timeout, endpoint, hostname, commons.RbfsAccessToken(token))
if err != nil {
    a4Response := ndo.A4GetInterfaceCountersDiagnosisResponse{
        ReturnCode: ndo.DiagReturnCode{
            Code:            ndo.RetCodeValueINTERNAL_ERROR,
            CodeDescription: fmt.Sprintf(ErrorInternalCannotCreateRBFSContext, cmds.newElementError(a4Spine, err)),
        },
    }
    return []actorAPI.RawMessage{&a4Response}, nil
}
```

**RbfsContext contains**:
- Endpoint URL: `https://192.168.1.10:12321`
- Hostname: For identifying the switch
- Token: JWT for authentication
- Timeout: 30 seconds for this operation

---

### Step 9: Reset Counters (Optional)

```go
if notification.ResetCounters {
    err = cmds.rbfs.ClearInterfaceCounters(rbfsContext, portMapping.CliLabel)
    if err != nil {
        a4Response := ndo.A4GetInterfaceCountersDiagnosisResponse{
            ReturnCode: ndo.DiagReturnCode{
                Code:            ndo.RetCodeValueINTERNAL_ERROR,
                CodeDescription: fmt.Sprintf(ErrorInternalCannotResetInterfaceCounters, cmds.newElementError(a4Spine, err)),
            },
        }
        return []actorAPI.RawMessage{&a4Response}, nil
    }
}
```

**ClearInterfaceCounters** (`pkg/controllers/diagnostics/interfaces.go`):

```go
func (s *service) ClearInterfaceCounters(ctx commons.RbfsContext, ifName string) error {
    endpoint, err := ctx.GetServiceEndpoint(commons.OpsdServiceName)
    if err != nil {
        return err
    }
    client := commons.GetAPIClient(s.client, endpoint)
    
    _, err = client.InterfacesApi.ClearInterfaceCounters(ctx, ifName)
    return err
}
```

**HTTP Call**:
```
DELETE https://192.168.1.10:12321/api/v1/interfaces/ifp-0/1/2/counters
Authorization: Bearer <jwt-token>
```

---

### Step 10: Get Physical Interface

```go
ifp, err := cmds.rbfs.GetPhysicalInterface(rbfsContext, portMapping.CliLabel)
if err != nil {
    a4Response := ndo.A4GetInterfaceCountersDiagnosisResponse{
        ReturnCode: ndo.DiagReturnCode{
            Code:            ndo.RetCodeValueINTERNAL_ERROR,
            CodeDescription: fmt.Sprintf(ErrorInternalCannotQueryRBFSPort, cmds.newElementError(a4Spine, err)),
        },
    }
    return []actorAPI.RawMessage{&a4Response}, nil
}
```

**GetPhysicalInterface** (`pkg/controllers/diagnostics/interfaces.go`):

```go
func (s *service) GetPhysicalInterface(ctx commons.RbfsContext, ifpName string) (*state.PhysicalInterfaceDetail, error) {
    endpoint, err := ctx.GetServiceEndpoint(commons.OpsdServiceName)
    if err != nil {
        return nil, err
    }

    client := commons.GetAPIClient(s.client, endpoint)
    opts := state.InterfacesApiGetPhysicalInterfaceOpts{}

    ifp, _, err := client.InterfacesApi.GetPhysicalInterface(ctx, ifpName, &opts)
    return &ifp, err
}
```

**HTTP Call**:
```
GET https://192.168.1.10:12321/api/v1/interfaces/physical/ifp-0/1/2
Authorization: Bearer <jwt-token>
```

**RBFS Response** (simplified):
```json
{
  "ifp-name": "ifp-0/1/2",
  "admin-status": "up",
  "oper-status": "up",
  "ifp-counters": {
    "rx": {
      "bytes-received": 1234567890,
      "packets-received": 9876543,
      "multicast-packets-received": 12345,
      "packets-dropped": 0
    },
    "tx": {
      "bytes-sent": 9876543210,
      "packets-sent": 12345678,
      "multicast-packets-sent": 54321,
      "packets-dropped": 0
    }
  }
}
```

---

### Step 11: Build Diagnosis Response

```go
var counters ndo.DiagnosisResult
if ifp.IfpCounters == nil {
    counters = ndo.DiagnosisResult{
        ResetCounter: notification.ResetCounters,
    }
} else {
    counters = ndo.DiagnosisResult{
        ResetCounter:      notification.ResetCounters,
        ReceivedBytes:     int64(ifp.IfpCounters.Rx.BytesReceived),
        ReceivedDropped:   int64(ifp.IfpCounters.Rx.PacketsDropped),
        ReceivedErrors:    int64(ifp.IfpCounters.Rx.PacketsDropped),
        ReceivedMulticast: int64(ifp.IfpCounters.Rx.MulticastPacketsReceived),
        ReceivedPackets:   int64(ifp.IfpCounters.Rx.PacketsReceived),
        SentBytes:         int64(ifp.IfpCounters.Tx.BytesSent),
        SentErrors:        int64(ifp.IfpCounters.Tx.PacketsDropped),
        SentDropped:       int64(ifp.IfpCounters.Tx.PacketsDropped),
        SentMulticast:     int64(ifp.IfpCounters.Tx.MulticastPacketsSent),
        SentPackets:       int64(ifp.IfpCounters.Tx.PacketsSent),
    }
}

a4Response := ndo.A4GetInterfaceCountersDiagnosisResponse{
    ReturnCode: ndo.DiagReturnCode{
        Code:            ndo.RetCodeValueOK,
        CodeDescription: "OK",
    },
    DiagnosisResult: counters,
}
```

**Mapping**:
| RBFS Field | Diagnosis Field |
|------------|-----------------|
| `ifp.IfpCounters.Rx.BytesReceived` | `ReceivedBytes` |
| `ifp.IfpCounters.Rx.PacketsReceived` | `ReceivedPackets` |
| `ifp.IfpCounters.Rx.MulticastPacketsReceived` | `ReceivedMulticast` |
| `ifp.IfpCounters.Rx.PacketsDropped` | `ReceivedDropped` / `ReceivedErrors` |
| `ifp.IfpCounters.Tx.BytesSent` | `SentBytes` |
| `ifp.IfpCounters.Tx.PacketsSent` | `SentPackets` |
| `ifp.IfpCounters.Tx.MulticastPacketsSent` | `SentMulticast` |
| `ifp.IfpCounters.Tx.PacketsDropped` | `SentDropped` / `SentErrors` |

---

### Step 12: Return Response

```go
s, _ := json.Marshal(a4Response)
logger.Infof("Full response of A4GetInterfaceCounters: %v", string(s))

return []actorAPI.RawMessage{&a4Response}, nil
```

**Flow back to client**:
1. Response returned to actor pipeline
2. Pipeline publishes to NATS `diagnosis-responses`
3. HTTP layer receives via NATS callback
4. HTTP layer formats as JSON
5. Client receives HTTP 200 with response body

---

## Code Examples

### Example 1: Successful Request

**cURL**:
```bash
curl -X POST http://localhost:2604/diagnoses \
  -H "Content-Type: application/json" \
  -d '{
    "diagnosisType": "io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest",
    "diagnosisParameters": {
      "resourceId": "550e8400-e29b-41d4-a716-446655440000",
      "resetCounters": false
    },
    "metadata": {
      "sourceObjectIdentifier": "550e8400-e29b-41d4-a716-446655440000",
      "sourceObjectClass": "NetworkElementPort"
    }
  }'
```

**Response**:
```json
{
  "id": "diag-123e4567-e89b-12d3-a456-426614174000",
  "responseData": {
    "diagnosisResult": {
      "resetCounter": false,
      "receivedBytes": 982374982374,
      "receivedPackets": 12398471,
      "receivedMulticast": 8273,
      "receivedDropped": 0,
      "receivedErrors": 0,
      "sentBytes": 123498273498,
      "sentPackets": 29834712,
      "sentMulticast": 9274,
      "sentDropped": 0,
      "sentErrors": 0
    }
  }
}
```

---

### Example 2: Reset Counters

**cURL**:
```bash
curl -X POST http://localhost:2604/diagnoses \
  -H "Content-Type: application/json" \
  -d '{
    "diagnosisType": "io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest",
    "diagnosisParameters": {
      "resourceId": "550e8400-e29b-41d4-a716-446655440000",
      "resetCounters": true
    },
    "metadata": {
      "sourceObjectIdentifier": "550e8400-e29b-41d4-a716-446655440000",
      "sourceObjectClass": "NetworkElementPort"
    }
  }'
```

**Response**:
```json
{
  "id": "diag-223e4567-e89b-12d3-a456-426614174001",
  "responseData": {
    "diagnosisResult": {
      "resetCounter": true,
      "receivedBytes": 0,
      "receivedPackets": 0,
      "receivedMulticast": 0,
      "receivedDropped": 0,
      "receivedErrors": 0,
      "sentBytes": 0,
      "sentPackets": 0,
      "sentMulticast": 0,
      "sentDropped": 0,
      "sentErrors": 0
    }
  }
}
```

---

### Example 3: Port with No Counters

Some interfaces may not have counter data:

**Response**:
```json
{
  "id": "diag-333e4567-e89b-12d3-a456-426614174002",
  "responseData": {
    "diagnosisResult": {
      "resetCounter": false
    }
  }
}
```

---

## Response Structure

### Success Response

```json
{
  "id": "string (UUID)",
  "responseData": {
    "diagnosisResult": {
      "resetCounter": "boolean",
      "receivedBytes": "int64",
      "receivedPackets": "int64",
      "receivedMulticast": "int64",
      "receivedDropped": "int64",
      "receivedErrors": "int64",
      "sentBytes": "int64",
      "sentPackets": "int64",
      "sentMulticast": "int64",
      "sentDropped": "int64",
      "sentErrors": "int64"
    }
  }
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID string | Unique identifier for this diagnosis request |
| `resetCounter` | boolean | Whether counters were reset |
| `receivedBytes` | int64 | Total bytes received on this interface |
| `receivedPackets` | int64 | Total packets received |
| `receivedMulticast` | int64 | Multicast packets received |
| `receivedDropped` | int64 | Received packets dropped |
| `receivedErrors` | int64 | Errors on received packets |
| `sentBytes` | int64 | Total bytes transmitted |
| `sentPackets` | int64 | Total packets transmitted |
| `sentMulticast` | int64 | Multicast packets transmitted |
| `sentDropped` | int64 | Transmitted packets dropped |
| `sentErrors` | int64 | Errors on transmitted packets |

---

## Error Scenarios

### Error 1: Invalid Source Object Class

**Request**:
```json
{
  "diagnosisType": "io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest",
  "diagnosisParameters": {
    "resourceId": "switch-uuid-12345",
    "resetCounters": false
  },
  "metadata": {
    "sourceObjectIdentifier": "switch-uuid-12345",
    "sourceObjectClass": "NetworkElement"
  }
}
```

**Response**:
```json
{
  "message": "NDO Interface Counters diagnosis is only supported for NetworkElementPort class, not for: NetworkElement"
}
```

**HTTP Status**: 400 Bad Request

**Fix**: Use a port UUID, not a switch UUID

---

### Error 2: Port Not Found

**Request**:
```json
{
  "diagnosisType": "io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest",
  "diagnosisParameters": {
    "resourceId": "non-existent-uuid",
    "resetCounters": false
  },
  "metadata": {
    "sourceObjectIdentifier": "non-existent-uuid",
    "sourceObjectClass": "NetworkElementPort"
  }
}
```

**Response**:
```json
{
  "message": "Cannot lookup network element: error fetching Logical resource from inventory: resource not found"
}
```

**HTTP Status**: 500 Internal Server Error

**Fix**: Verify the port UUID exists in inventory

---

### Error 3: Port Mapping Not Found

**Scenario**: Port exists but has no mapping to physical interface

**Response**:
```json
{
  "message": "Cannot lookup port mapping: port mapping not found for functionalPortLabel X1"
}
```

**HTTP Status**: 500 Internal Server Error

**Fix**: Ensure port mapping data is configured in inventory

---

### Error 4: Switch Unreachable

**Scenario**: Network connectivity issues to switch

**Response**:
```json
{
  "message": "Cannot query RBFS port: connection refused to 192.168.1.10:12321"
}
```

**HTTP Status**: 500 Internal Server Error

**Fix**: 
- Check network connectivity
- Verify switch is powered on
- Check management IP address
- Verify firewall rules

---

### Error 5: Authentication Failed

**Scenario**: Invalid or expired JWT token

**Response**:
```json
{
  "message": "Cannot create RBFS access token: authentication service returned 401 Unauthorized"
}
```

**HTTP Status**: 500 Internal Server Error

**Fix**: 
- Check auth service is running
- Verify auth service credentials
- Check token expiry settings

---

### Error 6: RBFS Interface Not Found

**Scenario**: Port mapping points to non-existent interface on switch

**Response**:
```json
{
  "message": "Cannot query RBFS port: interface ifp-0/1/2 not found on switch"
}
```

**HTTP Status**: 500 Internal Server Error

**Fix**: 
- Verify port mapping is correct
- Check interface exists on switch (login to switch CLI)
- Update port mapping if incorrect

---

## Testing

### Unit Test Example

**File**: `pkg/fixtures/diagnosis/ndo_get_interface_counters_test.go`

```go
func TestNDOInterfaceCounters(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // Setup mocks
    inventoryMock := inventory.NewMockClient(ctrl)
    diagnosticsMock := diagnostics.NewMockDiagnosticsService(ctrl)
    authMock := auth.NewMockServiceClient(ctrl)

    // Create test data
    portID := "port-uuid-12345"
    switchID := "switch-uuid-67890"
    
    port := &inventoryAPI.CLILogicalResource{
        ID: portID,
        // ... port details
    }
    
    spine := &inventoryAPI.CLILogicalResource{
        ID: switchID,
        // ... switch details
    }
    
    ifp := state.PhysicalInterfaceDetail{
        IfpCounters: &state.IfpCounters{
            Rx: state.RxCounters{
                BytesReceived: 123456,
                PacketsReceived: 9876,
            },
            Tx: state.TxCounters{
                BytesSent: 654321,
                PacketsSent: 1234,
            },
        },
    }

    // Setup expectations
    inventoryMock.EXPECT().
        FetchNetworkElementPortByID(gomock.Any(), portID).
        Return(port, nil)
    
    inventoryMock.EXPECT().
        FetchFabricSwitch(gomock.Any(), switchID).
        Return(spine, nil)
    
    inventoryMock.EXPECT().
        GetPortMapping(gomock.Any(), gomock.Any(), gomock.Any()).
        Return(&PortMapping{CliLabel: "ifp-0/0/0"}, nil)
    
    authMock.EXPECT().
        IsActive().
        Return(false)
    
    diagnosticsMock.EXPECT().
        GetPhysicalInterface(gomock.Any(), "ifp-0/0/0").
        Return(&ifp, nil)

    // Create commands
    cmds := NewCommands(inventoryMock, authMock, diagnosticsMock, ...)

    // Create request
    msg := &ndo.A4GetInterfaceCountersDiagnosisRequest{
        ResourceId:    portID,
        ResetCounters: false,
    }

    // Execute
    response, err := cmds.NDOInterfaceCounters(ctx, msg)

    // Assert
    assert.Nil(t, err)
    assert.NotNil(t, response)
    assert.Len(t, response, 1)
    
    resp := response[0].(*ndo.A4GetInterfaceCountersDiagnosisResponse)
    assert.Equal(t, "OK", resp.ReturnCode.Code)
    assert.Equal(t, int64(123456), resp.DiagnosisResult.ReceivedBytes)
    assert.Equal(t, int64(654321), resp.DiagnosisResult.SentBytes)
}
```

---

### Integration Test (Robot Framework)

**File**: `test/lib/commands.robot`

```robot
A4 Get Interface Counters
    [Arguments]    ${res_id}    ${bool}=false
    ${resp}=        POST    http://localhost:9080/diagnoses
    ...    {"diagnosisType": "io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest", 
    ...     "diagnosisParameters": {"resourceId": "${res_id}", "resetCounters": ${bool}},
    ...     "metadata": {"sourceObjectIdentifier":"${res_id}","sourceObjectClass":"NetworkElementPort"}}
    Status Should Be    200    ${resp}
    ${response_data}=    Get From Dictionary    ${resp.json()}    responseData
    ${result}=    Get From Dictionary    ${response_data}    diagnosisResult
    Should Not Be Empty    ${result}
    [Return]    ${result}
```

---

### Manual Testing with cURL

```bash
# 1. Get a valid port UUID from inventory
export PORT_ID="550e8400-e29b-41d4-a716-446655440000"

# 2. Test without reset
curl -X POST http://localhost:2604/diagnoses \
  -H "Content-Type: application/json" \
  -d "{
    \"diagnosisType\": \"io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest\",
    \"diagnosisParameters\": {
      \"resourceId\": \"$PORT_ID\",
      \"resetCounters\": false
    },
    \"metadata\": {
      \"sourceObjectIdentifier\": \"$PORT_ID\",
      \"sourceObjectClass\": \"NetworkElementPort\"
    }
  }" | jq

# 3. Test with reset
curl -X POST http://localhost:2604/diagnoses \
  -H "Content-Type: application/json" \
  -d "{
    \"diagnosisType\": \"io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest\",
    \"diagnosisParameters\": {
      \"resourceId\": \"$PORT_ID\",
      \"resetCounters\": true
    },
    \"metadata\": {
      \"sourceObjectIdentifier\": \"$PORT_ID\",
      \"sourceObjectClass\": \"NetworkElementPort\"
    }
  }" | jq

# 4. Test with invalid source class (should fail)
curl -X POST http://localhost:2604/diagnoses \
  -H "Content-Type: application/json" \
  -d "{
    \"diagnosisType\": \"io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest\",
    \"diagnosisParameters\": {
      \"resourceId\": \"$PORT_ID\",
      \"resetCounters\": false
    },
    \"metadata\": {
      \"sourceObjectIdentifier\": \"$PORT_ID\",
      \"sourceObjectClass\": \"NetworkElement\"
    }
  }" | jq
```

---

## Summary

The `A4GetInterfaceCounters` diagnosis API:

1. ✅ **Receives HTTP request** with port UUID
2. ✅ **Validates** source object class must be NetworkElementPort
3. ✅ **Looks up** port and parent switch from inventory
4. ✅ **Maps** logical port name to physical CLI label
5. ✅ **Prepares** authentication (JWT token if auth enabled)
6. ✅ **Connects** to RBFS switch via HTTPS/HTTP
7. ✅ **Optionally resets** interface counters
8. ✅ **Retrieves** physical interface details and counters
9. ✅ **Maps** RBFS counter values to diagnosis response format
10. ✅ **Returns** JSON response with TX/RX statistics

**Key Files**:
- Handler: [pkg/fixtures/diagnosis/ndo_get_interface_counters.go](../../../pkg/fixtures/diagnosis/ndo_get_interface_counters.go)
- RBFS Interface: [pkg/controllers/diagnostics/interfaces.go](../../../pkg/controllers/diagnostics/interfaces.go)
- HTTP Entry: [pkg/resources/diagnoses.go](../../../pkg/resources/diagnoses.go)
- Pipeline Setup: [pkg/fixtures/diagnosis/diagnosis.go](../../../pkg/fixtures/diagnosis/diagnosis.go)

**Related APIs**:
- `A4GetNepInfoAndState` - Get port operational state
- `A4ExecutePing` - Ping test from port
- `ShowTransceiverInfo` - Get transceiver/optic info
- `LinkTest` - Comprehensive port diagnostics

---

**Document Version**: 1.0  
**Last Updated**: February 2026  
**Maintained By**: RBFS Actor Development Team
