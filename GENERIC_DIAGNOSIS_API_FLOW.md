# Generic Diagnosis API Flow - Complete Guide

## Table of Contents
1. [Overview](#overview)
2. [Architecture Components](#architecture-components)
3. [Complete Request Flow](#complete-request-flow)
4. [Detailed Component Breakdown](#detailed-component-breakdown)
5. [Code Locations Reference](#code-locations-reference)
6. [How to Add a New Diagnosis API](#how-to-add-a-new-diagnosis-api)

---

## Overview

This document explains how diagnosis APIs work in the RBFS (RtBrick Full Stack) Actor system. Diagnosis APIs are used to run diagnostic operations on network elements (switches, ports, etc.) and retrieve their status, configuration, or perform operations like ping, trace route, or retrieve interface counters.

### What is a Diagnosis API?
A diagnosis API allows external systems to:
- Query network element status
- Execute diagnostic commands on switches
- Retrieve interface information and counters
- Test connectivity
- Check BGP/ISIS neighbors
- And many more network diagnostics

---

## Architecture Components

The system consists of several layers that work together:

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP REST API Layer                      │
│              (pkg/resources/diagnoses.go)                   │
│                   Port: 2604 (default)                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  NATS Message Bus Layer                     │
│        Subject: diagnosis-validated-requests                │
│        Subject: diagnosis-responses                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Actor Pipeline Processing Layer                │
│          (pkg/fixtures/diagnosis/diagnosis.go)              │
│       - Message routing & handler registration              │
│       - Concurrency control (optional)                      │
│       - Validation                                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Diagnosis Handler Functions                    │
│    (pkg/fixtures/diagnosis/ndo_*.go, list_*.go, etc.)      │
│       - Business logic                                      │
│       - Inventory lookups                                   │
│       - RBFS interactions                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              RBFS Switch Communication                      │
│      (pkg/controllers/diagnostics/diagnostics.go)           │
│       - REST API calls to RBFS switches                     │
│       - Authentication handling                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Components:

1. **HTTP REST API Layer**: Entry point for external diagnosis requests
2. **NATS Message Bus**: Asynchronous message routing between components
3. **Actor Pipeline**: Message processing and routing to specific handlers
4. **Diagnosis Handlers**: Business logic for each diagnosis type
5. **RBFS Client**: Communication with actual network switches
6. **Inventory Service**: Network element information lookup

---

## Complete Request Flow

### Step-by-Step Flow Diagram

```
External Client
    │
    │ 1. HTTP POST /diagnoses
    │    {diagnosisType, diagnosisParameters, metadata}
    ▼
┌──────────────────────────────────────┐
│  HTTP Handler (runDiagnosis)         │
│  - Deserialize request               │
│  - Extract diagnosis type & params   │
│  - Validate metadata                 │
└──────────┬───────────────────────────┘
           │
           │ 2. Publish to NATS
           │    Subject: diagnosis-validated-requests
           ▼
┌──────────────────────────────────────┐
│  NATS Message Bus                    │
│  - Route message to diagnosis actor  │
└──────────┬───────────────────────────┘
           │
           │ 3. Actor Pipeline receives message
           ▼
┌──────────────────────────────────────┐
│  Pipeline Message Router             │
│  - Match message type to handler     │
│  - Check concurrency limits          │
│  - Create actor context              │
└──────────┬───────────────────────────┘
           │
           │ 4. Invoke specific handler
           ▼
┌──────────────────────────────────────┐
│  Diagnosis Handler Function          │
│  - Validate input parameters         │
│  - Fetch network element info        │
│  - Prepare authorization             │
│  - Call RBFS switch API              │
│  - Process response                  │
│  - Build diagnosis response          │
└──────────┬───────────────────────────┘
           │
           │ 5. Handler uses Inventory
           │
           ▼
┌──────────────────────────────────────┐
│  Inventory Service                   │
│  - Lookup network element            │
│  - Get management IP                 │
│  - Get port mappings                 │
└──────────┬───────────────────────────┘
           │
           │ 6. Handler interacts with RBFS
           ▼
┌──────────────────────────────────────┐
│  RBFS Diagnostics Service            │
│  - HTTP/HTTPS to switch              │
│  - Execute diagnostic command        │
│  - Return results                    │
└──────────┬───────────────────────────┘
           │
           │ 7. Response flows back
           ▼
┌──────────────────────────────────────┐
│  Handler builds response object      │
│  - ReturnCode (OK/ERROR)             │
│  - DiagnosisResult (data)            │
└──────────┬───────────────────────────┘
           │
           │ 8. Publish to NATS
           │    Subject: diagnosis-responses
           ▼
┌──────────────────────────────────────┐
│  NATS Message Bus                    │
│  - Route response back to HTTP layer │
└──────────┬───────────────────────────┘
           │
           │ 9. HTTP response
           ▼
External Client receives JSON response
```

---

## Detailed Component Breakdown

### 1. HTTP REST API Layer

**Location**: `pkg/resources/diagnoses.go`

**Purpose**: Provides the HTTP endpoint for diagnosis requests

**Key Functions**:

```go
// Main HTTP endpoint handler
func (d *diagnoses) runDiagnosis(w http.ResponseWriter, r *http.Request)
```

**What it does**:
1. **Receives HTTP POST** to `/diagnoses`
2. **Deserializes JSON** request body containing:
   - `diagnosisType`: Fully qualified event type (e.g., `io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest`)
   - `diagnosisParameters`: JSON object with diagnosis-specific parameters
   - `metadata`: Source object information (identifier, class, type)
3. **Validates** the request structure
4. **Publishes** to NATS message bus
5. **Waits** for response (with timeout)
6. **Sends HTTP response** back to client

**Request Example**:
```json
{
  "diagnosisType": "io.leitstand.actor.events.diagnosis.ndo.A4GetInterfaceCountersDiagnosisRequest",
  "diagnosisParameters": {
    "resourceId": "port-uuid-12345",
    "resetCounters": false
  },
  "metadata": {
    "sourceObjectIdentifier": "port-uuid-12345",
    "sourceObjectClass": "NetworkElementPort",
    "sourceObjectType": "A4-Port"
  }
}
```

**Response Example**:
```json
{
  "id": "request-uuid",
  "responseData": {
    "diagnosisResult": {
      "receivedBytes": 123456,
      "sentBytes": 654321,
      ...
    }
  }
}
```

---

### 2. NATS Message Bus

**Purpose**: Asynchronous communication between HTTP adapter and diagnosis actor

**Subjects**:
- **Input**: `diagnosis-validated-requests` - Where diagnosis requests are published
- **Output**: `diagnosis-responses` - Where diagnosis results are published

**Message Flow**:
1. HTTP layer publishes `DiagnosisRequest` event to input subject
2. Diagnosis actor subscribes to input subject
3. Diagnosis actor processes request
4. Diagnosis actor publishes `DiagnosisResponse` to output subject
5. HTTP layer receives response and forwards to client

---

### 3. Actor Pipeline

**Location**: `pkg/fixtures/diagnosis/diagnosis.go`

**Purpose**: Routes incoming diagnosis requests to appropriate handler functions

**Key Components**:

#### A. Pipeline Creation

```go
func CreatePipeline() actorAPI.Layer {
    return buildPipeline(inputSubject, outputSubject, registerDiagnosisHandlers)
}
```

This creates the main pipeline that:
- Listens to NATS input subject
- Routes messages to handlers
- Publishes responses to output subject

#### B. Handler Registration

```go
func registerDiagnosisHandlers(pipelineBuilder *pipelines.HighLevelPipelineBuilder, commands *Commands) {
    pipelineBuilder.
        On(schemas.A4GetInterfaceCountersDiagnosisRequest_1.GetName(), commands.NDOInterfaceCounters).
        On(schemas.ListBgpPeeringsRequest_1.GetName(), commands.ListBGPPeerings).
        On(schemas.A4ExecutePingDiagnosisRequest_1.GetName(), commands.ExecutePing).
        // ... more handlers
}
```

Each `On()` call maps a diagnosis type to a handler function.

#### C. Commands Structure

```go
type Commands struct {
    inventory           inventory.Client          // Network element lookup
    authService         auth.ServiceClient        // Authentication
    rbfs                diagnostics.DiagnosticsService  // RBFS communication
    dbHandler           database.DBHandler        // Database operations
    bemClient           bemclient.EventsService   // Event publishing
    publisher           kafka.Producer            // Kafka publishing
    concurrencyLimiter  chan struct{}            // Optional: concurrency control
    activeDiagnosis     *sync.Map                // Optional: track active diagnosis
    maxConcurrency      int                      // Optional: max parallel diagnosis
    enableRouting       bool                     // Optional: async processing
}
```

#### D. Concurrency Control (Optional Feature)

The system supports parallel diagnosis processing:

```go
func (cmds *Commands) processAsyncDiagnosis(
    ctx actorAPI.ActorContext,
    msg actorAPI.RawMessage,
    diagnosisType string,
    processor func(...) (...)
) ([]actorAPI.RawMessage, actorAPI.LayerError)
```

When enabled (via `ENABLE_ROUTING_LOGIC=true`):
- Diagnosis requests run in goroutines
- Concurrency limited by `MAX_CONCURRENT_DIAGNOSIS` (default: 50)
- Prevents system overload during high diagnosis load

---

### 4. Diagnosis Handler Functions

**Location**: `pkg/fixtures/diagnosis/` (various files)

**Purpose**: Implement the actual diagnosis logic

**Common Pattern** (every handler follows this):

```go
func (cmds *Commands) NDOInterfaceCounters(
    actorCtx actorAPI.ActorContext, 
    msg actorAPI.RawMessage
) ([]actorAPI.RawMessage, actorAPI.LayerError) {
    
    // Step 1: Early Validation
    // - Check if message is nil
    // - Validate metadata (source object class, etc.)
    
    // Step 2: Decide sync vs async (if routing enabled)
    if cmds.enableRouting {
        return cmds.processAsyncDiagnosis(actorCtx, msg, "NDOInterfaceCounters", cmds.ndoInterfaceCountersSync)
    }
    
    // Step 3: Call synchronous implementation
    return cmds.ndoInterfaceCountersSync(actorCtx, msg)
}

func (cmds *Commands) ndoInterfaceCountersSync(
    actorCtx actorAPI.ActorContext,
    msg actorAPI.RawMessage
) ([]actorAPI.RawMessage, actorAPI.LayerError) {
    
    // Step 4: Parse request
    notification := msg.(*ndo.A4GetInterfaceCountersDiagnosisRequest)
    
    // Step 5: Inventory lookups
    // - Fetch network element
    // - Get port information
    // - Get port mapping (physical to logical)
    
    // Step 6: Prepare RBFS connection
    // - Get authentication token (if auth enabled)
    // - Get management IP address
    // - Build RBFS endpoint URL
    
    // Step 7: Execute diagnosis on RBFS switch
    // - Create RBFS context
    // - Call specific RBFS API
    // - Handle errors
    
    // Step 8: Process and format response
    // - Extract relevant data
    // - Build response object
    // - Set return code (OK, ERROR, etc.)
    
    // Step 9: Return response
    return []actorAPI.RawMessage{&response}, nil
}
```

**Key Sub-operations**:

#### A. Fetching Network Element Information

```go
// For port-based diagnosis
a4Spine, a4Port, err := cmds.fetchSwitchByPort(ctx.ActorContext, resourceId)

// For network element diagnosis
networkElement, err := cmds.inventory.FetchFabricSwitch(ctx, resourceId)
```

This retrieves:
- Network element (switch) details
- Port information
- Operational data (IP addresses, hostname, etc.)

#### B. Getting Port Mapping

```go
portMapping, err := cmds.inventory.GetPortMapping(
    inventory.NETypeName(a4Spine),
    inventory.PlannedMatNumber(a4Spine),
    functionalPortLabel,
)
```

Maps logical port name to physical CLI label used by RBFS (e.g., `ifp-0/1/2`)

#### C. Preparing Authorization

```go
token, port, protocol, err := cmds.prepareAuthorization(ctx)
```

Returns:
- `token`: JWT token if authentication is enabled (empty if not)
- `port`: `12321` (HTTPS) or `19091` (HTTP)
- `protocol`: `https` or `http`

#### D. Creating RBFS Context

```go
endpoint, _ := url.Parse(fmt.Sprintf("%s://%s:%s", protocol, mgmtIP, port))
rbfsContext, _ := commons.NewRbfsContext(timeout, endpoint, hostname, commons.RbfsAccessToken(token))
```

This creates a context object that:
- Contains endpoint URL
- Includes authentication token
- Has timeout settings
- Provides the switch hostname

---

### 5. RBFS Diagnostics Service

**Location**: `pkg/controllers/diagnostics/diagnostics.go`

**Purpose**: Low-level HTTP client for communicating with RBFS switches

**Interface**:

```go
type DiagnosticsService interface {
    GetPhysicalInterface(ctx commons.RbfsContext, ifpName string) (*state.PhysicalInterfaceDetail, error)
    ClearInterfaceCounters(ctx commons.RbfsContext, ifName string) error
    ListBGPPeerings(ctx commons.RbfsContext) ([]*BGPInstanceSummary, error)
    GetBGPPeering(ctx commons.RbfsContext, instanceName, peerIP string) (*state.BgpPeering, error)
    RunPing(commons.RbfsContext, *ping.Ping) (state.PingStatus, error)
    // ... many more
}
```

**What it does**:
1. Takes RBFS context (with endpoint, auth, etc.)
2. Makes HTTP REST calls to RBFS switch
3. Parses RBFS responses
4. Returns Go structs representing switch data

**Example - GetPhysicalInterface**:

```go
func (s *service) GetPhysicalInterface(ctx commons.RbfsContext, ifpName string) (*state.PhysicalInterfaceDetail, error) {
    // Get RBFS API endpoint
    endpoint, err := ctx.GetServiceEndpoint(commons.OpsdServiceName)
    
    // Create API client
    client := commons.GetAPIClient(s.client, endpoint)
    
    // Make HTTP call to RBFS
    opts := state.InterfacesApiGetPhysicalInterfaceOpts{}
    ifp, _, err := client.InterfacesApi.GetPhysicalInterface(ctx, ifpName, &opts)
    
    return &ifp, err
}
```

This translates to an HTTP call like:
```
GET https://<switch-ip>:12321/api/v1/interfaces/physical/ifp-0/1/2
Authorization: Bearer <jwt-token>
```

---

### 6. Response Building

All diagnosis handlers must return a response object with:

1. **ReturnCode**: Indicates success or failure
   ```go
   type DiagReturnCode struct {
       Code            string  // OK, BAD_REQUEST, INTERNAL_ERROR, NOT_FOUND, etc.
       CodeDescription string  // Human-readable error message
   }
   ```

2. **DiagnosisResult**: The actual diagnosis data (structure varies per diagnosis)

**Success Response**:
```go
response := &ndo.A4GetInterfaceCountersDiagnosisResponse{
    ReturnCode: ndo.DiagReturnCode{
        Code:            ndo.RetCodeValueOK,
        CodeDescription: "OK",
    },
    DiagnosisResult: ndo.DiagnosisResult{
        ReceivedBytes: 123456,
        SentBytes: 654321,
        // ... more counters
    },
}
```

**Error Response**:
```go
response := &ndo.A4GetInterfaceCountersDiagnosisResponse{
    ReturnCode: ndo.DiagReturnCode{
        Code:            ndo.RetCodeValueINTERNAL_ERROR,
        CodeDescription: fmt.Sprintf("Failed to connect to switch: %v", err),
    },
}
```

---

## Code Locations Reference

| Component | File Path |
|-----------|-----------|
| HTTP Entry Point | `pkg/resources/diagnoses.go` |
| Main Actor | `cmd/actors/diagnosis/main.go` |
| Pipeline Setup | `pkg/fixtures/diagnosis/diagnosis.go` |
| Handler Functions | `pkg/fixtures/diagnosis/ndo_*.go`, `list_*.go`, etc. |
| RBFS Service Interface | `pkg/controllers/diagnostics/diagnostics.go` |
| RBFS Interface Calls | `pkg/controllers/diagnostics/interfaces.go` |
| RBFS BGP Calls | `pkg/controllers/diagnostics/bgp.go` |
| Inventory Client | `pkg/fixtures/inventory/` |
| Auth Service | `pkg/fixtures/auth/` |

---

## How to Add a New Diagnosis API

### Step 1: Define the Event Schema

Create protobuf or Avro schema for your diagnosis request and response.

### Step 2: Create Handler Function

Create a new file in `pkg/fixtures/diagnosis/`:

```go
func (cmds *Commands) MyNewDiagnosis(
    actorCtx actorAPI.ActorContext,
    msg actorAPI.RawMessage,
) ([]actorAPI.RawMessage, actorAPI.LayerError) {
    
    // Validate
    if msg == nil {
        return []actorAPI.RawMessage{&ErrorResponse{}}, nil
    }
    
    // Async routing (optional)
    if cmds.enableRouting {
        return cmds.processAsyncDiagnosis(actorCtx, msg, "MyNewDiagnosis", cmds.myNewDiagnosisSync)
    }
    
    return cmds.myNewDiagnosisSync(actorCtx, msg)
}

func (cmds *Commands) myNewDiagnosisSync(
    actorCtx actorAPI.ActorContext,
    msg actorAPI.RawMessage,
) ([]actorAPI.RawMessage, actorAPI.LayerError) {
    
    // 1. Parse request
    req := msg.(*MyNewDiagnosisRequest)
    
    // 2. Fetch network element
    ne, err := cmds.inventory.FetchFabricSwitch(actorCtx, req.ResourceId)
    if err != nil {
        return []actorAPI.RawMessage{&ErrorResponse{...}}, nil
    }
    
    // 3. Prepare authorization
    token, port, protocol, err := cmds.prepareAuthorization(fixtures.NewContext(actorCtx))
    if err != nil {
        return []actorAPI.RawMessage{&ErrorResponse{...}}, nil
    }
    
    // 4. Get management IP
    mgmtIP, err := inventory.ManagementIPAddress(ne.GetExtendedData())
    if err != nil {
        return []actorAPI.RawMessage{&ErrorResponse{...}}, nil
    }
    
    // 5. Create RBFS context
    endpoint, _ := url.Parse(fmt.Sprintf("%s://%s:%s", protocol, mgmtIP, port))
    rbfsContext, _ := commons.NewRbfsContext(context.Background(), endpoint, hostname, token)
    
    // 6. Call RBFS service
    result, err := cmds.rbfs.YourRBFSMethod(rbfsContext, ...)
    if err != nil {
        return []actorAPI.RawMessage{&ErrorResponse{...}}, nil
    }
    
    // 7. Build response
    response := &MyNewDiagnosisResponse{
        ReturnCode: DiagReturnCode{Code: "OK", CodeDescription: "OK"},
        DiagnosisResult: result,
    }
    
    return []actorAPI.RawMessage{response}, nil
}
```

### Step 3: Register Handler

In `pkg/fixtures/diagnosis/diagnosis.go`:

```go
func registerDiagnosisHandlers(pipelineBuilder *pipelines.HighLevelPipelineBuilder, commands *Commands) {
    pipelineBuilder.
        // ... existing handlers
        On(schemas.MyNewDiagnosisRequest_1.GetName(), commands.MyNewDiagnosis)
}
```

### Step 4: Add RBFS Service Method (if needed)

If you need a new RBFS API call, add to `pkg/controllers/diagnostics/diagnostics.go`:

```go
type DiagnosticsService interface {
    // ... existing methods
    YourRBFSMethod(ctx commons.RbfsContext, param string) (*YourResult, error)
}

func (s *service) YourRBFSMethod(ctx commons.RbfsContext, param string) (*YourResult, error) {
    endpoint, err := ctx.GetServiceEndpoint(commons.OpsdServiceName)
    client := commons.GetAPIClient(s.client, endpoint)
    
    result, _, err := client.YourApi.YourMethod(ctx, param)
    return &result, err
}
```

### Step 5: Write Tests

Create `pkg/fixtures/diagnosis/my_new_diagnosis_test.go`:

```go
func TestMyNewDiagnosis(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()
    
    mockInventory := inventory.NewMockClient(ctrl)
    mockRBFS := diagnostics.NewMockDiagnosticsService(ctrl)
    
    // Set up mocks
    mockInventory.EXPECT().FetchFabricSwitch(...)
    mockRBFS.EXPECT().YourRBFSMethod(...)
    
    // Create commands
    cmds := NewCommands(mockInventory, ..., mockRBFS, ...)
    
    // Create test request
    req := &MyNewDiagnosisRequest{...}
    
    // Execute
    response, err := cmds.MyNewDiagnosis(ctx, req)
    
    // Assert
    assert.Nil(t, err)
    // ... more assertions
}
```

### Step 6: Test via HTTP

```bash
curl -X POST http://localhost:2604/diagnoses \
  -H "Content-Type: application/json" \
  -d '{
    "diagnosisType": "io.leitstand.actor.events.diagnosis.MyNewDiagnosisRequest",
    "diagnosisParameters": {
      "resourceId": "some-uuid"
    },
    "metadata": {
      "sourceObjectIdentifier": "some-uuid",
      "sourceObjectClass": "NetworkElement"
    }
  }'
```

---

## Common Error Scenarios

### 1. Network Element Not Found
- **Cause**: Invalid resourceId in request
- **Code**: `ManagementErrorsLogicalResourceNotFound`
- **Fix**: Verify resourceId exists in inventory

### 2. Authentication Failed
- **Cause**: Invalid or expired JWT token
- **Code**: `UNAUTHORIZED`
- **Fix**: Check auth service configuration

### 3. RBFS Connection Failed
- **Cause**: Switch unreachable or wrong IP
- **Code**: `INTERNAL_ERROR`
- **Fix**: Verify management IP and network connectivity

### 4. Invalid Source Object Class
- **Cause**: Diagnosis only supports certain object classes
- **Code**: `BAD_REQUEST`
- **Fix**: Use correct sourceObjectClass in metadata

### 5. Timeout
- **Cause**: Slow response from RBFS or network issues
- **Code**: `SERVICE_UNAVAILABLE`
- **Fix**: Increase timeout or check network

---

## Best Practices

1. **Always validate early**: Check nil messages and metadata before processing
2. **Use proper error messages**: Include context (NE ID, port name, etc.) in errors
3. **Log important steps**: Use `logger.Infof()` for tracking diagnosis flow
4. **Handle all error cases**: Every RBFS call, inventory lookup should have error handling
5. **Return proper return codes**: Use appropriate code (OK, BAD_REQUEST, INTERNAL_ERROR, etc.)
6. **Test with mocks**: Mock inventory and RBFS services in unit tests
7. **Consider concurrency**: Use async processing for long-running diagnosis
8. **Clean up resources**: Use `defer cancel()` for contexts with timeout

---

## Performance Considerations

### Concurrency Control

The system supports configurable concurrency:

```bash
# Environment variables
MAX_CONCURRENT_DIAGNOSIS=50      # Max parallel diagnosis (default: 50)
ENABLE_ROUTING_LOGIC=true        # Enable async processing
```

### Timeout Settings

Default timeouts:
- HTTP request to diagnosis actor: **60 seconds**
- RBFS API calls: **30 seconds** (configurable per handler)

### Caching

Some components use caching:
- Material information: **10 minute TTL**
- Can be extended for other frequently accessed data

---

## Troubleshooting

### Check Logs

```bash
# Diagnosis actor logs
kubectl logs -f deployment/pod-rtbrick-rbfs-diagnosis-actor

# Look for:
# - "Concurrency status" - shows active diagnosis count
# - "Full response of <DiagnosisName>" - shows successful responses
# - Error messages with network element IDs
```

### Common Log Messages

**Good**:
```
Concurrency status for NDOInterfaceCounters: 5/50 active diagnoses (10.0% utilization)
Full response of A4GetInterfaceCounters: {...}
```

**Problems**:
```
Cannot fetch the Switch info by port: ...
Cannot create RBFS access token: ...
Cannot retrive in-band mgmt IP address: ...
```

### Testing Individual Components

```go
// Test inventory lookup
ne, err := inventory.FetchFabricSwitch(ctx, "some-uuid")

// Test RBFS connection
rbfsCtx, _ := commons.NewRbfsContext(...)
ifp, err := diagnosticsService.GetPhysicalInterface(rbfsCtx, "ifp-0/1/2")
```

---

## Summary

The diagnosis API flow follows this pattern:

1. **HTTP POST** → `/diagnoses` endpoint
2. **NATS Publish** → `diagnosis-validated-requests` subject
3. **Actor Pipeline** → Routes to handler based on diagnosis type
4. **Handler Function** → Validates, looks up inventory, calls RBFS
5. **RBFS Service** → Makes HTTP call to actual switch
6. **Response Building** → Formats data with return code
7. **NATS Publish** → `diagnosis-responses` subject
8. **HTTP Response** → Returns JSON to client

Every diagnosis follows this flow with variations in:
- What inventory data is needed
- Which RBFS API is called
- How the response is structured

This architecture provides:
- ✅ **Decoupling**: HTTP layer separate from business logic
- ✅ **Scalability**: Async processing with concurrency control
- ✅ **Testability**: Each component can be mocked and tested
- ✅ **Maintainability**: Clear separation of concerns
- ✅ **Extensibility**: Easy to add new diagnosis types

---

**Document Version**: 1.0  
**Last Updated**: February 2026  
**Maintained By**: RBFS Actor Development Team
