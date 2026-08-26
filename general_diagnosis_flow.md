# General Diagnosis Flow – How Diagnosis Works in This Project

> **For:** Freshers / New Engineers joining this project  
> **What you'll learn:** What "diagnosis" means here, how a request travels through the system, and what every piece does.

---

## 1. What is "Diagnosis"?

In a telecom network, **diagnosis** means automatically checking whether a piece of network equipment is working correctly — like a doctor running tests on a patient.

Instead of a human engineer manually logging into every switch to check its health, this system **automatically queries** network devices and returns a health report.

This project (`rtbrick-rbfs-actor`) specifically diagnoses **RTBrick RBFS** — the operating system running on fabric switches in a Deutsche Telekom Access 4 network.

---

## 2. Key Terms You Must Know Before Reading Anything Else

| Term | Plain English Meaning |
|---|---|
| **NE** | Network Element — a physical network device, like a switch in a data center |
| **NEP** | Network Element Port — one physical port/socket on that switch |
| **A4** | Access 4 — Deutsche Telekom's inventory and management platform that stores a database of all NEs and NEPs |
| **RBFS** | RTBrick Forwarding Software — the operating system running on the fabric switches |
| **Actor** | A small software component that sits and waits for messages, then processes them |
| **Actor Framework** | The `pod-actor` library — a system that delivers messages to the right actor automatically |
| **Protobuf** | Protocol Buffers — a compact, typed format for sending structured data between services (like JSON but binary and faster) |
| **JWT** | JSON Web Token — a secure credential (like a password ticket) used to prove identity when calling APIs |
| **Handler Function** | The Go function that processes one specific type of diagnosis request |
| **Port Mapping** | A translation table: A4 names ports one way, RBFS names them differently — this table bridges the two |
| **Operational Data** | Live, real-time data about a device — its current IP address, hostname, status |
| **Goroutine** | A lightweight Go "thread" — a small unit of work that can run concurrently |
| **LayerError** | A special error type in the actor framework, used only for infrastructure-level failures |

---

## 3. The Big Picture — Where This Project Sits

```
+----------------------------------------------------------+
|               Deutsche Telekom A4 Platform               |
|                                                          |
|  Operator/          sends request         Diagnosis      |
|  Automation   ----------------------->    Actor          |
|  System                (protobuf)         (THIS PROJECT) |
|                                               |          |
|                        +----------------------+          |
|                        |                                 |
|               A4 Inventory API                           |
|               (lookup switch info)                       |
+------------------------|---------------------------------+
                         |
                         | direct REST API call
                         v
               +---------------------+
               |  RBFS Switch (live) |   <-- the actual physical switch
               +---------------------+
```

The Diagnosis Actor sits **between** the A4 management platform and the physical RBFS switches.

What it does:
1. Receives a diagnosis **request** from A4
2. Looks up device details in the **A4 Inventory**
3. Calls the **RBFS API** directly on the physical switch
4. Returns a **structured diagnosis response** back to A4

---

## 4. The Actor Framework — How Messages Get Delivered

The `pod-actor` framework is a **message-driven** system. Think of it like a post office:

```
Incoming Protobuf Message
         |
         v
+---------------------------+
|    Actor Framework        |
|  (pod-actor library)      |
|                           |
|  1. Receives the message  |
|  2. Reads its type        |
|  3. Routes to the right   |
|     handler function      |
+------------+--------------+
             |
             v
    +--------+---------+
    |                  |
    v                  v
NepCounters()    BgpPeering()    ...etc
(this file)
```

Every diagnosis type has its own **handler function** registered with the actor framework. When a request comes in, the framework automatically calls the right one.

### ActorContext — What It Is

`ActorContext` is an object the framework gives to every handler call. It contains:
- **Task metadata** — what triggered this task, what kind of object it involves
- **Logger** — for writing structured log messages
- **Cancellation** — a signal to stop work if the request is cancelled upstream

### RawMessage — What It Is

`RawMessage` is the framework's generic message type. The framework doesn't need to know the exact protobuf type — it just passes the raw bytes. Each handler then **type-asserts** it to the specific type it expects, like:
```go
notification := msg.(*pediag.NepCountersRequest_2)
```

---

## 5. The `Commands` Struct — The Toolbox

All diagnosis logic is methods on a struct called `Commands`. Think of it as a **service class** that carries all the tools (dependencies) needed:

```go
type Commands struct {
    rbfs          RbfsClient      // client to talk to RBFS switches via REST API
    inventory     InventoryClient // client to talk to A4 inventory
    enableRouting bool            // feature flag: should we run async or sync?
    // ... other config
}
```

Because every diagnosis function is a method on `*Commands`, it always has access to the RBFS client, inventory client, and all configuration — without needing to pass them around manually.

---

## 6. General Diagnosis Flow — Step by Step

Every diagnosis in this project follows the **same general pattern**:

### STEP 1 — Input Validation
```
Is msg == nil?            --> return BAD_REQUEST immediately
Is source object class    --> return BAD_REQUEST immediately
  wrong for this
  diagnosis type?
```
This prevents the code from ever reaching a nil-pointer panic. Fast-fail at the top.

---

### STEP 2 — Routing Decision
```
Is enableRouting = true?
  YES --> processAsyncDiagnosis()   (runs in a goroutine pool, non-blocking)
  NO  --> diagnosisSync()           (runs directly, blocking)
```
Both paths eventually call the same sync function. The difference is only **when and where** in memory it runs.

---

### STEP 3 — A4 Inventory Lookup
```
Use notification.ResourceId (UUID of the port)
  --> Query A4 to find the NE (switch) and NEP (port)
  --> Get port characteristics (e.g., functionalPortLabel)
  --> Get NE operational data (hostname, management IP)
```
This is necessary because the request only contains a UUID. All actual device details must be looked up from A4.

---

### STEP 4 — Port Name Translation (Port Mapping)
```
A4 calls the port "IFP-0/0/1" (functional port label)
RBFS calls the same port "ifp-0/0/1" (CLI label)

Port mapping table: IFP-0/0/1 on hardware BNG-32 --> ifp-0/0/1
```
Without this translation, the RBFS API would not know which port we are asking about.

---

### STEP 5 — Authentication
```
prepareAuthorization()
  --> returns: JWT token (credential), port number, protocol (http/https)
```
The RBFS API requires a valid JWT token. This step fetches one.

---

### STEP 6 — Build RBFS API Context
```
endpoint = "https://10.1.2.3:8080"
rbfsContext = NewRbfsContext(timeout=10s, endpoint, hostname, token)
```
Bundles all connection info into one context object ready for API calls.

---

### STEP 7 — Query the Switch (The Actual API Call)
```
rbfs.GetPhysicalInterface(rbfsContext, "ifp-0/0/1")
  --> makes an HTTP REST call to the physical switch
  --> waits up to 10 seconds for a response
  --> returns real-time interface data
```
This is the moment the code actually "touches" the physical network device.

---

### STEP 8 — Build and Return Response
```
Map RBFS data --> protobuf response struct
Set ReturnCode, Result, Rating
Return []actorAPI.RawMessage{&response}
```

---

## 7. Async vs Sync — What is the Difference?

### Synchronous (Sync) Mode
```
Request --> Handler runs --> RBFS API call --> Response
           (blocks the goroutine until done)
```
Simple. One request at a time per goroutine.

### Asynchronous (Async) Mode — when `enableRouting = true`
```
Request --> processAsyncDiagnosis() --> dispatches to a worker pool
                                            |
                              many diagnoses run in parallel
                                            |
                              results collected and returned
```
Better performance when many diagnosis requests arrive at the same time. Controlled by the `enableRouting` boolean in config.

---

## 8. Response Structure

Every diagnosis response has the same skeleton:

```
DiagnosisResponse {
    ReturnCode {
        Code            -- OK / NOT_FOUND / BAD_REQUEST / INTERNAL_ERROR
        CodeDescription -- human-readable explanation
    }
    Result {
        Result   -- diagnosis-specific value (COUNTERS, NOT_FOUND, etc.)
        Rating   -- OK or NOK
    }
    ... diagnosis-specific data fields
}
```

### Return Codes

| Code | When It Is Used |
|---|---|
| `OK` | Everything worked, data is in the response |
| `BAD_REQUEST` | Invalid request (wrong object class, nil message) |
| `NOT_FOUND` | The NE/NEP was not found in A4 inventory |
| `INTERNAL_ERROR` | Something broke internally (RBFS unreachable, auth failed, etc.) |

### Rating

| Rating | Meaning |
|---|---|
| `OK` | The diagnosed component appears healthy |
| `NOK` | The diagnosed component is **not healthy** — action may be needed |

---

## 9. Error Handling — The Key Design Rule

**Business errors go inside the response. `LayerError` is only for the framework itself breaking.**

```go
// WRONG way -- do not use LayerError for business logic failures
return nil, someLayerError

// CORRECT way -- embed the error in the typed response
return []actorAPI.RawMessage{&Response{
    ReturnCode: DiagReturnCode{
        Code:            NOT_FOUND,
        CodeDescription: "switch sw-01 not found in A4",
    },
}}, nil
```

This design means callers **always** get a structured, typed result -- they can programmatically check the `ReturnCode` instead of catching exceptions.

---

## 10. Project File Structure

```
rtbrick-rbfs-actor/
|
+-- cmd/actors/diagnosis/        <-- entry point: registers all handlers
|
+-- pkg/fixtures/diagnosis/      <-- all diagnosis handler functions
|   +-- nep_counters.go          <-- NEP port counter diagnosis
|   +-- bgp_peering.go           <-- BGP session diagnosis
|   +-- commands.go              <-- Commands struct + shared helper functions
|
+-- pkg/fixtures/inventory/      <-- helpers to read A4 inventory data
|   +-- hostname.go
|   +-- management_ip.go
|   +-- port_mapping.go
|
+-- pkg/clients/switchclient/    <-- RBFS switch REST API client
|
+-- docs/                        <-- documentation
    +-- 01_general_diagnosis_flow.md   <-- THIS FILE
    +-- 02_nep_counters.md             <-- NEP counters deep dive
```

---

## 11. The Golden Path Summary (One Sentence Each Step)

1. Operator asks "diagnose port abc-123"
2. Actor framework receives the request and calls `NepCounters()`
3. Validate: message is not nil, object class is `NetworkElementPort`
4. Look up port abc-123 in A4: belongs to switch `sw-01.berlin`, port `IFP-0/0/1`
5. Translate port name: `IFP-0/0/1` on `BNG-32` hardware = RBFS name `ifp-0/0/1`
6. Get JWT token + management IP `10.1.2.3` for the switch
7. Call RBFS REST API: `GET /interface/ifp-0/0/1` at `https://10.1.2.3:8080` (10s timeout)
8. RBFS responds with real-time counters
9. Build and return `NepCountersResponse_2` with all 14 counters filled in
