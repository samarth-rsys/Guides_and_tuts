# NEP Counters – Deep Dive

> **File:** `pkg/fixtures/diagnosis/nep_counters.go`  
> **For:** Freshers who want to understand every single line of this diagnosis  
> **Prerequisite:** Read `01_general_diagnosis_flow.md` first

---

## 1. What Are NEP Counters?

**NEP** = Network Element Port = one physical port on a fabric switch.

**Counters** = traffic statistics that the switch hardware counts automatically every second. They tell you:
- How much traffic is flowing through a port
- Whether packets are being dropped (bad sign!)
- Whether there are transmission errors (also bad!)

Think of it like the speedometer and odometer in a car — passive measurements that tell you what is happening.

---

## 2. The 14 Counters Explained

The response contains exactly 14 counter values, split into **Rx** (Receive = incoming traffic) and **Tx** (Transmit = outgoing traffic):

### Receive (Rx) Counters — Traffic Coming INTO the Port

| Counter Key | What It Counts | Why It Matters |
|---|---|---|
| `total_rx_bytes` | Total bytes received | Overall traffic volume |
| `total_rx_packets` | Total packets received | Overall packet count |
| `total_rx_unicast_packets` | Point-to-point packets received | Normal traffic |
| `total_rx_multicast_packets` | One-to-many group packets received | Can indicate flooding if abnormally high |
| `total_rx_broadcast_packets` | Packets sent to all devices received | High values = possible broadcast storm |
| `total_rx_dropped_packets` | Packets received but thrown away | **RED FLAG** — indicates congestion or buffer overflow |
| `total_rx_error_packets` | Corrupted/malformed packets received | **RED FLAG** — indicates physical layer problems |

### Transmit (Tx) Counters — Traffic Going OUT of the Port

| Counter Key | What It Counts | Why It Matters |
|---|---|---|
| `total_tx_bytes` | Total bytes sent | Overall traffic volume |
| `total_tx_packets` | Total packets sent | Overall packet count |
| `total_tx_unicast_packets` | Point-to-point packets sent | Normal traffic |
| `total_tx_multicast_packets` | One-to-many group packets sent | |
| `total_tx_broadcast_packets` | Broadcast packets sent | |
| `total_tx_dropped_packets` | Packets queued to send but dropped | **RED FLAG** — output queue congestion |
| `total_tx_error_packets` | Packets that failed to transmit | **RED FLAG** — hardware or cabling issue |

> All counter values are returned as **strings** (e.g., `"1234567"`), not numbers. This is because protobuf maps require a consistent type, and large counter values can exceed 32-bit integer limits.

---

## 3. The Two Functions

The code has two functions that work together:

```
NepCounters()         <-- public entry point, called by actor framework
    |
    v
nepCountersSync()     <-- private function containing all the actual logic
```

### Why Two Functions?
`NepCounters()` does only **validation and routing**. The actual work lives in `nepCountersSync()`. This separation allows `nepCountersSync` to be passed as a function reference to `processAsyncDiagnosis()` for parallel execution without duplicating any logic.

---

## 4. Line-by-Line Walk Through

### 4.1 Function Signature

```go
func (cmds *Commands) NepCounters(
    actorCtx actorAPI.ActorContext,
    msg actorAPI.RawMessage,
) ([]actorAPI.RawMessage, actorAPI.LayerError)
```

- `cmds *Commands` — the receiver. Has access to `cmds.rbfs` (RBFS client) and `cmds.inventory` (A4 client)
- `actorCtx` — the actor framework context. Contains task metadata and logger
- `msg` — the raw incoming protobuf message (will be cast to `NepCountersRequest_2` later)
- Returns a **slice** of messages (always one element here) and a framework-level error

---

### 4.2 Guard: Nil Message Check

```go
if msg == nil {
    logger.Errorf("NEP Counters validation failed: request message is nil")
    return []actorAPI.RawMessage{&pediag.NepCountersResponse_2{
        ReturnCode: pediag.DiagReturnCode{
            Code:            pediag.RetCodeValueBAD_REQUEST,
            CodeDescription: "Validation failed: request message is nil",
        },
    }}, nil
}
```

**Why?** If `msg` is nil and we try to use it below (e.g., `msg.(*pediag.NepCountersRequest_2)`), the program panics and crashes. This check prevents that.

**Notice:** Even the error response is wrapped in a slice: `[]actorAPI.RawMessage{...}`. The actor framework always expects a slice back.

---

### 4.3 Guard: Object Class Check

```go
sourceObjectClass := actorCtx.OriginalTask().Metadata.SourceObjectClass
if sourceObjectClass != string(cliApi.NetworkElementPort) {
    errorMsg := fmt.Sprintf("NEP Counters diagnosis is only supported for NetworkElementPort class, not for: %s", sourceObjectClass)
    ...
    return BAD_REQUEST response
}
```

**What is `SourceObjectClass`?**  
Every A4 object has a "class" — its type. Common classes:
- `NetworkElementPort` — a physical port on a switch
- `NetworkElement` — a switch itself
- `NetworkElementLink` — a link between two switches

NEP Counters only make sense for `NetworkElementPort`. If someone sends a request for a `NetworkElement` (the whole switch), this check catches it and returns a clear error instead of crashing deeper in the code.

**`cliApi.NetworkElementPort`** is a typed constant (not a raw string) defined in the CLI API library. Converting to string with `string(...)` allows comparison.

---

### 4.4 Routing Decision

```go
if cmds.enableRouting {
    return cmds.processAsyncDiagnosis(actorCtx, msg, "NEPCounters", cmds.nepCountersSync)
}
return cmds.nepCountersSync(actorCtx, msg)
```

`cmds.nepCountersSync` is passed as a **function value** (first-class function in Go). `processAsyncDiagnosis` stores it and calls it in a worker goroutine.

---

### 4.5 Build Context and Cast Message

```go
ctx := fixtures.NewContext(actorCtx)
notification := msg.(*pediag.NepCountersRequest_2)
```

**`fixtures.NewContext(actorCtx)`** — wraps the actor context with extra helpers like a structured logger.

**`msg.(*pediag.NepCountersRequest_2)`** — this is a Go **type assertion**. It says "I know this `RawMessage` is actually a `*NepCountersRequest_2` — give me the typed version." If it is the wrong type at runtime, this panics — but that should never happen because the actor framework only delivers the right message type to the right handler.

`notification.ResourceId` is now accessible — this is the UUID of the NEP in A4.

---

### 4.6 Fetch Switch and Port from A4

```go
a4Ne, a4Port, a4Err := cmds.fetchSwitchByPort(ctx.ActorContext, notification.ResourceId)
```

This calls the **A4 inventory API** using the port UUID. Returns:
- `a4Ne` — the Network Element (switch) object. Has fields like `a4Ne.ID` (UUID), operational data
- `a4Port` — the port object. Has "characteristics" (key-value properties stored in A4)

**Special handling for non-fabric switches:**
```go
if strings.Contains(errMsg, "not a fabric switch") {
    ctx.Logger().Infof("NEPCounters: NEP belongs to non-fabric; skipping...")
    return nil, nil
}
```

Returning `nil, nil` (no messages, no error) tells the actor framework "there's nothing to process here." This is intentional — non-fabric switches are not managed by RBFS, so there is no point trying to query them. This avoids polluting logs with expected errors.

---

### 4.7 Get Functional Port Label

```go
functionalPortLabel := a4Port.GetCharacteristicValue("functionalPortLabel")
```

**Characteristics** in A4 are key-value pairs attached to objects. `"functionalPortLabel"` is one such key — its value is a human-readable port identifier like `"IFP-0/0/1"`.

This label is stored in the final response so the caller knows which port was diagnosed in human-readable form.

---

### 4.8 Port Mapping — Translating Names

```go
portMapping, a4Err := cmds.inventory.GetPortMapping(
    inventory.NETypeName(a4Ne),       // e.g., "BNG-32"
    inventory.PlannedMatNumber(a4Ne), // hardware material number
    functionalPortLabel,              // e.g., "IFP-0/0/1"
)
```

**Why is this needed?**  
A4 inventory uses one naming convention for ports. RBFS uses a completely different one. Without translation, the RBFS API would not recognize the port name.

`GetPortMapping()` looks up a translation table and returns `portMapping.CliLabel` — the RBFS-native name (e.g., `"ifp-0/0/1"`) used in the actual API call.

The lookup uses both the **NE type** and the **material number** because the same functional port label might map to different CLI labels on different hardware models.

---

### 4.9 Prepare Authorization

```go
token, port, protocol, a4Err := cmds.prepareAuthorization(ctx)
```

Returns:
- `token` — JWT Bearer token (a long encoded string). Included in every RBFS API call as `Authorization: Bearer <token>`
- `port` — the TCP port number the RBFS API listens on (e.g., `"8080"`)
- `protocol` — `"http"` or `"https"` depending on configuration

---

### 4.10 Get Operational Data

```go
a4NeOperationalData, ok := a4Ne.GetExtendedData()
hostname := inventory.Hostname(a4NeOperationalData)
mgmtIP, err := inventory.ManagementIPAddress(a4NeOperationalData)
```

**Operational data** is live data synced from the network into A4. Unlike static inventory data (which just says what *should* be there), operational data reflects what *is* there right now.

- `hostname` — the switch's DNS name (e.g., `"sw-01.berlin.telekom.de"`). Used in TLS certificate verification
- `mgmtIP` — the **in-band management IP address** of the switch. This is the IP we send REST API calls to

**Note:** If `GetExtendedData()` returns `ok = false`, the code logs a warning but continues. Hostname might be empty in that case, but management IP failure causes an error response because you literally cannot reach the switch without it.

---

### 4.11 Build Endpoint URL

```go
endpoint, err := url.Parse(fmt.Sprintf("%s://%s:%s", protocol, mgmtIP, port))
// Result: e.g., "https://10.1.2.3:8080"
```

`url.Parse()` validates the URL structure. If for some reason the combination of protocol/IP/port produces an invalid URL, the error is caught and returned before any network call is made.

---

### 4.12 Create RBFS Context

```go
timeout, cancel := context.WithTimeout(ctx.ActorContext, 10*time.Second)
defer cancel()

rbfsContext, err := commons.NewRbfsContext(timeout, endpoint, hostname, commons.RbfsAccessToken(token))
```

**`context.WithTimeout`** creates a Go context that automatically cancels after 10 seconds. Any API call using this context will be aborted if it takes longer than 10 seconds. `defer cancel()` ensures the timeout timer is cleaned up when the function returns.

**`commons.NewRbfsContext`** bundles the timeout context, endpoint URL, hostname, and JWT token into one object that the RBFS client library understands.

---

### 4.13 Query RBFS — The Actual Switch Call

```go
ifp, err := cmds.rbfs.GetPhysicalInterface(rbfsContext, portMapping.CliLabel)
```

**This is the moment the code talks to the real physical switch.**

`GetPhysicalInterface()` makes an HTTP GET request to:
```
https://10.1.2.3:8080/some/rbfs/api/path/ifp-0/0/1
```

The switch responds with a JSON body that gets deserialized into the `ifp` struct. This struct has:
- `ifp.IfpCounters.Rx` — all receive-side counters
- `ifp.IfpCounters.Tx` — all transmit-side counters

If the switch is unreachable, returns an error → `INTERNAL_ERROR` response.

---

### 4.14 Extract Counter Values

```go
nepCounters := map[string]string{
    "total_rx_bytes":             "0",
    "total_rx_packets":           "0",
    // ... all 14 keys initialized to "0"
}
countersRating := pediag.NspPacketCountersRatingOK
countersResultValue := pediag.CountersResultValueNO_COUNTERS_AVAILABLE

if ifp.IfpCounters != nil {
    countersResultValue = pediag.CountersResultValueCOUNTERS
    if ifp.IfpCounters.Rx != nil {
        nepCounters["total_rx_bytes"] = strconv.FormatInt(int64(ifp.IfpCounters.Rx.BytesReceived), 10)
        // ... 6 more Rx counters
    }
    if ifp.IfpCounters.Tx != nil {
        nepCounters["total_tx_bytes"] = strconv.FormatInt(int64(ifp.IfpCounters.Tx.BytesSent), 10)
        // ... 6 more Tx counters
    }
}
```

**Why initialize to "0"?**  
Pre-filling with "0" guarantees the map always has all 14 keys. If only Rx is available (Tx is nil), the caller still gets a complete map — Tx counters just show "0".

**`strconv.FormatInt(int64(...), 10)`** — converts the counter number to a decimal string:
- `int64(...)` — safely converts the unsigned integer from RBFS to a signed 64-bit int (handles large values without overflow)
- `strconv.FormatInt(..., 10)` — converts the number to base-10 string

**`CountersResultValueNO_COUNTERS_AVAILABLE`** — if `IfpCounters` is nil, it means the switch returned an interface object but no counter data (can happen if the interface is new or counters haven't been populated yet). This is not an error — it's a valid state.

---

### 4.15 Build and Return the Final Response

```go
a4Response := &pediag.NepCountersResponse_2{
    ReturnCode: pediag.DiagReturnCode{
        Code:            pediag.RetCodeValueOK,
        CodeDescription: "",
    },
    CountersResult: pediag.CountersResult{
        Result: countersResultValue,
        Rating: countersRating,
    },
    FunctionalPortLabel: functionalPortLabel,
    NepID:               notification.ResourceId,
    NeInventoryData: pediag.NeInventoryData_2{
        Uuid:     a4Ne.ID,
        Hostname: hostname,
    },
    NepCounters: nepCounters,
}
s, _ := json.Marshal(a4Response)
logger.Debugf("Full response of NepCounters: %v", string(s))

return []actorAPI.RawMessage{a4Response}, nil
```

The response includes:
- `ReturnCode` — `OK` with empty description (no error)
- `CountersResult` — whether counters were available + the rating
- `FunctionalPortLabel` — human-readable port name from A4 (e.g., `"IFP-0/0/1"`)
- `NepID` — the original port UUID from the request (so caller can correlate)
- `NeInventoryData` — switch UUID and hostname for context
- `NepCounters` — the map of all 14 counter strings

The `json.Marshal` + `logger.Debugf` line is purely for **debug logging** — serializes the full response to JSON so it can be printed in logs when debug level is enabled. The `_` ignores the error from `json.Marshal` because this is only for logging, not critical.

---

## 5. Complete Error Handling Map

| What Went Wrong | Return Code | CountersResult | Rating |
|---|---|---|---|
| `msg` is nil | `BAD_REQUEST` | — | — |
| Wrong source object class (not a port) | `BAD_REQUEST` | — | — |
| Port UUID not found in A4 inventory | `NOT_FOUND` | `NOT_FOUND` | `NOK` |
| Port is on a non-fabric (non-RBFS) switch | `nil, nil` — **silently skipped** | — | — |
| Port mapping lookup failed | `NOT_FOUND` | `NOT_FOUND` | `NOK` |
| Could not get JWT auth token | `INTERNAL_ERROR` | `UNKNOWN` | `NOK` |
| Could not get management IP address | `INTERNAL_ERROR` | `UNKNOWN` | `NOK` |
| URL parsing failed | `INTERNAL_ERROR` | `UNKNOWN` | `NOK` |
| RBFS context creation failed | `INTERNAL_ERROR` | `UNKNOWN` | `NOK` |
| RBFS `GetPhysicalInterface` call failed | `INTERNAL_ERROR` | `UNKNOWN` | `NOK` |
| Switch responded but no counter data | `OK` | `NO_COUNTERS_AVAILABLE` | `OK` |
| Counters successfully retrieved | `OK` | `COUNTERS` | `OK` |

**Why `UNKNOWN` for internal errors?** When something goes wrong before the RBFS call, we genuinely do not know the port's health state. `UNKNOWN` is more honest than `NOT_FOUND` or `NOK`.

**Why silent skip for non-fabric?** Non-fabric switches are not managed by RBFS. Returning an error for them would be misleading. The actor framework sees `nil, nil` as "nothing to report" and moves on quietly.

---

## 6. Data Flow Diagram (Complete)

```
[Request arrives]
       |
       |  NepCountersRequest_2 { ResourceId: "uuid-of-nep" }
       |
[NepCounters() entry point]
       |
       +-- msg == nil?  --> BAD_REQUEST
       |
       +-- wrong object class? --> BAD_REQUEST
       |
       +-- enableRouting? --> async path OR sync path
       |
[nepCountersSync()]
       |
       +-- fetchSwitchByPort("uuid-of-nep")
       |       |
       |       A4 Inventory API
       |       |
       |       returns: a4Ne (switch), a4Port (port)
       |       |
       |       "not a fabric switch"? --> return nil, nil (skip)
       |       other error? --> NOT_FOUND response
       |
       +-- a4Port.GetCharacteristicValue("functionalPortLabel")
       |       returns: "IFP-0/0/1"
       |
       +-- inventory.GetPortMapping("BNG-32", matNum, "IFP-0/0/1")
       |       returns: portMapping.CliLabel = "ifp-0/0/1"
       |       error? --> NOT_FOUND response
       |
       +-- prepareAuthorization()
       |       returns: token, port="8080", protocol="https"
       |       error? --> INTERNAL_ERROR response
       |
       +-- a4Ne.GetExtendedData()
       |       returns: hostname="sw-01.berlin", mgmtIP="10.1.2.3"
       |       mgmtIP error? --> INTERNAL_ERROR response
       |
       +-- url.Parse("https://10.1.2.3:8080")
       |       error? --> INTERNAL_ERROR response
       |
       +-- commons.NewRbfsContext(timeout=10s, endpoint, hostname, token)
       |       error? --> INTERNAL_ERROR response
       |
       +-- rbfs.GetPhysicalInterface(rbfsContext, "ifp-0/0/1")
       |       |
       |       HTTP GET --> physical switch at 10.1.2.3:8080
       |       |
       |       returns: ifp { IfpCounters { Rx: {...}, Tx: {...} } }
       |       error? --> INTERNAL_ERROR response
       |
       +-- Extract all 14 counters from ifp.IfpCounters
       |       IfpCounters == nil? --> NO_COUNTERS_AVAILABLE (still OK)
       |       IfpCounters present? --> COUNTERS (OK)
       |
       +-- Build NepCountersResponse_2 {
               ReturnCode:          OK
               CountersResult:      { Result: COUNTERS, Rating: OK }
               FunctionalPortLabel: "IFP-0/0/1"
               NepID:               "uuid-of-nep"
               NeInventoryData:     { Uuid: "uuid-of-ne", Hostname: "sw-01.berlin" }
               NepCounters:         { total_rx_bytes: "1234567", ... }
           }
```

---

## 7. Key Design Decisions Explained

### Why are counters stored as strings, not numbers?
Protobuf map types require a consistent value type. Also, counter values can be very large (64-bit unsigned integers). Using strings avoids integer overflow in intermediate representations and keeps the API consistent.

### Why 10 seconds timeout?
In a telecom network, if a switch does not respond in 10 seconds, it is effectively unreachable (or severely overloaded). Waiting longer would block the goroutine unnecessarily and degrade performance for other requests.

### Why default all counters to "0" before reading?
This guarantees the response map always has all 14 keys. Callers can safely access `response.NepCounters["total_rx_bytes"]` without checking for key existence.

### Why `int64(ifp.IfpCounters.Rx.BytesReceived)`?
The RBFS library may return counter values as `uint32` or `uint64`. The `int64` cast ensures we can safely call `strconv.FormatInt` without a compile error, and handles the case where RBFS uses unsigned types.

---

## 8. Glossary for This File

| Term | Meaning |
|---|---|
| `NepCountersRequest_2` | The protobuf request message type. The `_2` suffix means version 2 of this message |
| `NepCountersResponse_2` | The protobuf response message type |
| `RetCodeValueOK` | Constant: the string/enum value for a successful return code |
| `RetCodeValueBAD_REQUEST` | Constant: invalid request |
| `RetCodeValueNOT_FOUND` | Constant: resource not found in inventory |
| `RetCodeValueINTERNAL_ERROR` | Constant: internal system failure |
| `NspPacketCountersRatingOK` | Constant: the port appears healthy |
| `NspPacketCountersRatingNOK` | Constant: the port appears unhealthy |
| `CountersResultValueCOUNTERS` | Constant: counter data was successfully retrieved |
| `CountersResultValueNO_COUNTERS_AVAILABLE` | Constant: the switch returned no counter data |
| `CountersResultValueNOT_FOUND` | Constant: the interface was not found on the switch |
| `CountersResultValueUNKNOWN` | Constant: the state could not be determined |
| `IFP` | Ingress Forwarding Port — RBFS internal term for a physical switch interface |
| `CliLabel` | The port name in RBFS CLI notation (e.g., `"ifp-0/0/1"`) |
| `functionalPortLabel` | The port name in A4 inventory notation (e.g., `"IFP-0/0/1"`) |
