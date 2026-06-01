# NE Connectivity V2 — Complete Code Flow Explained

> This document explains the **entire code flow** of how the rtbrick-rbfs-actor checks if a fabric switch is reachable, and how it sends alarm messages to pao-switch-core. Written for someone new to the codebase.

---

## Table of Contents

1. [The Big Picture — What Are We Doing?](#1-the-big-picture)
2. [The Key Players (Files & Their Roles)](#2-the-key-players)
3. [The Protobuf Contract — How Systems Talk](#3-the-protobuf-contract)
4. [Step-by-Step Code Flow](#4-step-by-step-code-flow)
5. [V1 vs V2 — What Changed and Why](#5-v1-vs-v2)
6. [The Mock Switch Client — For Testing](#6-the-mock-switch-client)
7. [Data Flow Diagram](#7-data-flow-diagram)
8. [Where Does pao-switch-core Pick This Up?](#8-where-does-pao-switch-core-pick-this-up)
9. [NeStatus Struct — The "Memory" of Each Switch](#9-nestatus-struct)
10. [Glossary](#10-glossary)

---

## 1. The Big Picture

Imagine you're a building security guard 🏢. Your job is to check if every office (switch) in the building is responding. You monitor each office via **three channels**:

- **Out-of-band phone** (dedicated management line) — you call them every 100s
- **In-band phone** (regular network line) — you call them every 100s
- **GELF heartbeat** (the office calls YOU periodically) — you check if they called at least twice every 90s

If an office doesn't pick up 3 times in a row on one phone, you report: "Office X's out-of-band phone is down." If the office stops sending heartbeats, you report: "Office X's GELF interface is down." If ALL THREE channels are down (both phones + heartbeats), you report: "Office X is completely unreachable!"

**Important:** The rtbrick-rbfs-actor only **sends events** (NeLostIndication messages) to Kafka. The actual **alarm raising** (NeCommunicationInterfaceStateChange and NeReachabilityStateChange) is done by pao-switch-core on the receiving end.

That's exactly what `rtbrick-rbfs-actor` does. It pings fabric switches and monitors GELF heartbeats, then sends NeLostIndication events to `pao-switch-core` via **Kafka** (a message bus).

---

## 2. The Key Players

Here's every file involved and what it does:

### In `rtbrick-rbfs-actor`:

| File | Role | Think of it as... |
|------|------|-------------------|
| `pkg/fixtures/adapter/neConnectivity/neConnectivity.go` | **The main brain** — runs the periodic ping loop, tracks failures, sends alarms | The security guard |
| `pkg/clients/switchclient/client_systemops.go` | **The phone** — actually makes HTTP requests to ping switches | The telephone |
| `pkg/clients/switchclient/systemops/systemops.go` | **The phone's instruction manual** — defines the interface (what methods exist) | Phone manual |
| `pkg/clients/switchclient/api.go` | **Combines all interfaces** into one `SwitchClient` | The master interface |
| `pkg/clients/switchclient/mocks/mockswitchclient.go` | **Fake phone for testing** — pretends to ping without actually calling a switch | Testing dummy |
| `pkg/adapters/store/` | **Address book** — stores switch metadata (IPs, serial numbers) in Redis | Contact list |
| `cmd/adapters/main.go` | **The starter** — boots everything up, creates the connectivity checker, starts the loop | Power button |

### In `pao-switch-api-defs` (shared protobuf definitions):

| File | Role |
|------|------|
| `adapter/protos/v1/indications.proto` | **The message template** — defines what a `NeLostIndication` message looks like |
| `adapter/generated/v1/indications.pb.go` | **Auto-generated Go code** from the proto file — gives us Go structs and enums |

### In `pao-switch-core` (the consumer):

| File | Role |
|------|------|
| `internal/core/assurance/neReachabilityAlarm.go` | **The alarm handler** — receives our messages and creates proper alarms |

---

## 3. The Protobuf Contract — How Systems Talk

### What is a Protobuf?

Protobuf (Protocol Buffers) is like a **shared form** that two systems agree on. If I want to tell you "Switch X is down because out-of-band failed", I fill out the form and send it. You know how to read it because we both have the same form definition.

### The Form: `NeLostIndication`

Defined in `pao-switch-api-defs/adapter/protos/v1/indications.proto`:

```protobuf
message NeLostIndication {
    string serial_number           = 1;   // Which switch? (e.g., "ABC123")
    ConnectionState state          = 2;   // UP or DOWN?
    CommunicationLossReason reason = 3;   // WHY? Which interface?
}
```

### The "Why" — `CommunicationLossReason` enum:

```protobuf
enum CommunicationLossReason {
    COMMUNICATION_LOSS_REASON_UNSPECIFIED     = 0;  // Unknown/default
    COMMUNICATION_LOSS_REASON_INBAND          = 1;  // In-band management interface
    COMMUNICATION_LOSS_REASON_OUTOFBAND       = 2;  // Out-of-band management interface
    COMMUNICATION_LOSS_REASON_GELF            = 3;  // GELF log interface
    COMMUNICATION_LOSS_REASON_NOCOMMUNICATION = 4;  // ALL interfaces are down!
}
```

### The "State" — `ConnectionState` enum:

```protobuf
enum ConnectionState {
    CONNECTION_STATE_UNSPECIFIED = 0;
    CONNECTION_STATE_DOWN       = 1;  // Interface/switch is unreachable
    CONNECTION_STATE_UP         = 2;  // Interface/switch is back
}
```

### How it becomes Go code:

After running `buf generate` (or `make generate`), the proto becomes Go structs in `adapter/generated/v1/indications.pb.go`:

```go
type NeLostIndication struct {
    SerialNumber  string                  // "ABC123"
    State         ConnectionState         // CONNECTION_STATE_DOWN or UP
    Reason        CommunicationLossReason // INBAND, OUTOFBAND, GELF, NOCOMMUNICATION
}
```

**This is what rtbrick-rbfs-actor fills in and sends. This is what pao-switch-core receives and reads.**

---

## 4. Step-by-Step Code Flow

### Step 0: Application Starts

**File:** `cmd/adapters/main.go`

```
main() starts
  → creates a SwitchClient (the real HTTP client that can ping switches)
  → creates NewNeConnectivityCheck(switchClient, logger)
  → starts CheckNEConnectivity(ctx) in a goroutine (background loop)
```

### Step 1: The Ticker Loop — Two Timers Running in Parallel

**File:** `neConnectivity.go` → `CheckNEConnectivity()`

```
Two tickers run simultaneously:

  NeConnectivityTicker (every 100s, configurable via PING_INTERVAL):
    1. PerformLocalReconciliation()  ← sync local cache with Redis inventory
    2. checkForConnectivity()        ← ping ALL switches (out-of-band + in-band)

  HeartBeatCheckTicker (every 90s, configurable via HEART_BEAT_INTERVAL):
    1. checkHeartBeatCount()         ← check GELF heartbeats for ALL switches
```

Think of it as: Every 100 seconds, the security guard calls each office. Every 90 seconds, they check: "Did each office call me back at least twice?"

### Step 2: Get the List of Switches

**File:** `neConnectivity.go` → `PerformLocalReconciliation()`

```
1. Read from Redis: "What switches exist?" (via readInventory())
2. Compare with local cache: "Do I already know about these?"
3. Add new switches, remove deleted ones
4. Each switch gets a NeStatus entry in the cache:
   {
     PingAlarmRaised:      false,
     HeartBeatAlarmRaised: false,
     FailureCount:         0,
     OutOfBandAlarmRaised: false,  // V2 NEW
     InBandAlarmRaised:    false,  // V2 NEW
     OutOfBandFailCount:   0,      // V2 NEW
     InBandFailCount:      0,      // V2 NEW
     NoCommunication:      false,  // V2 NEW
   }
```

The cache key is `"SN-<serial_number>"`, e.g., `"SN-ABC123"`.

### Step 3: Ping Each Switch

**File:** `neConnectivity.go` → `checkForConnectivity()`

```
For each switch in the cache (up to 16 in parallel):
  → go checkSpecificNeConnectivity(ctx, serialNumber, neStatus)
```

### Step 4: Check One Specific Switch (THIS IS THE V2 CHANGE)

**File:** `neConnectivity.go` → `checkSpecificNeConnectivity()`

```
1. Get switch metadata from Redis (IP addresses, NE type, etc.)
   Key: "/pao/neconfig/<serialNumber>"

2. Ping OUT-OF-BAND interface:
   _, oobErr := c.switchclient.PingOutOfBand(ctx, metadata)
   
   This calls: GET https://<outband-ip>:12321/api/v1/ctrld/ping
   Expects: HTTP 204 (No Content) = success

3. Ping IN-BAND interface:
   _, ibErr := c.switchclient.PingInBand(ctx, metadata)
   
   This calls: GET https://<inband-ip>:12321/api/v1/ctrld/ping
   Expects: HTTP 204 (No Content) = success

4. Handle each result independently:
   if outOfBand failed → handleInterfaceFailure(..., OUTOFBAND)
   if outOfBand OK     → handleInterfaceSuccess(..., OUTOFBAND)
   if inBand failed    → handleInterfaceFailure(..., INBAND)
   if inBand OK        → handleInterfaceSuccess(..., INBAND)

5. Check if ALL THREE are down (OOB + IB + GELF):
   if all three down AND not already reported NOCOMMUNICATION:
     → sendNeLostIndication(serial, DOWN, NOCOMMUNICATION)
   if at least one recovered AND was NOCOMMUNICATION:
     → sendNeLostIndication(serial, UP, NOCOMMUNICATION)
```

### Step 5: Handle Interface Failure (Per-Interface)

**File:** `neConnectivity.go` → `handleInterfaceFailure()`

```
failCount++ for this specific interface

if failCount >= threshold (default 3) AND event not already sent:
  → sendNeLostIndication(serial, DOWN, OUTOFBAND or INBAND)
  → mark as sent for this interface
```

**V2 key difference from V1:** Each interface event is sent **independently** — there is NO mutual exclusion with other interfaces. If GELF is already down AND out-of-band fails, the out-of-band event is STILL sent. Switch-core needs to know about each interface separately.

Why threshold of 3? Because a single failed ping might be a temporary glitch. We wait for 3 consecutive failures before reporting.

### Step 6: Handle Interface Success (Per-Interface)

**File:** `neConnectivity.go` → `handleInterfaceSuccess()`

```
if DOWN event was previously sent for this interface:
  → sendNeLostIndication(serial, UP, OUTOFBAND or INBAND)
  → mark as cleared
  → reset fail count to 0
```

### Step 7: Send the Message to Kafka

**File:** `neConnectivity.go` → `sendNeLostIndication()`

This is where the protobuf gets used!

```go
func sendNeLostIndication(ctx, serialNumber, state, reason) {
    // 1. CREATE the protobuf message
    msg := &api.NeLostIndication{
        SerialNumber: serialNumber,                    // e.g., "ABC123"
        State:        state,                           // DOWN or UP
        Reason:       reason,                          // OUTOFBAND, INBAND, or NOCOMMUNICATION
    }

    // 2. SERIALIZE to JSON bytes
    bytes := protojson.Marshal(msg)
    // Result: {"serialNumber":"ABC123","state":"CONNECTION_STATE_DOWN","reason":"COMMUNICATION_LOSS_REASON_OUTOFBAND"}

    // 3. CONNECT to Kafka
    bus.NewBusConnectionForProtoMessage(bus.PaoActorRd, bus.PaoActorRd, bus.KafkaProducer)

    // 4. PUBLISH to Kafka topic "NeLostIndication"
    bus.Publish(bus.NeLostIndicationTopic, bytes)
}
```

**After this, the message is on the Kafka bus.** pao-switch-core is listening on the other side.

### Step 8: GELF Heartbeat Monitoring (Separate from Ping)

GELF is monitored **completely differently** from out-of-band/in-band. It's not ping-based — it's **heartbeat-based**.

#### How it works:

RBFS switches proactively send periodic GELF log messages with event code `HTB0001` roughly every ~45 seconds. The system doesn't "call" the switch — **the switch "calls home"**.

```
              In-band / Out-of-band:   rtbrick-actor PINGS the switch  →  "Are you there?"
              GELF:                     Switch SENDS heartbeats to us  ←  "I'm still here!"
```

#### Phase 1 — Counting heartbeats (runs continuously):

```
Switch sends HTB0001 GELF event
  → arrives on Kafka "fabric-events" topic
    → GELF pipeline handler (HandleHeartBeatService) processes it
      → Redis counter incremented: connectivity-HeartBeatCount/<serialNumber> ++
```

#### Phase 2 — Checking the count (every 90 seconds via HeartBeatCheckTicker):

**File:** `neConnectivity.go` → `checkHeartBeatCount()`

```
For each switch in Redis heartbeat counters:
  1. Read the counter value
  2. Reset counter to 0
  3. Check: did we get ≥ 2 heartbeats in the last 90 seconds?

     YES (≥ 2): GELF is healthy
       → if HeartBeatAlarmRaised was true:
           sendNeLostIndication(serial, UP, GELF)   ← clear GELF alarm
           HeartBeatAlarmRaised = false

     NO (< 2): GELF heartbeats are missing!
       → if HeartBeatAlarmRaised is false AND PingAlarmRaised is false:
           HeartBeatAlarmRaised = true
           sendNeLostIndication(serial, DOWN, GELF)  ← raise GELF alarm
```

#### Why threshold of 2?

With heartbeats sent every ~45s and a check window of 90s, we expect at least 2 heartbeats per window. Fewer than 2 means the GELF connection is likely down.

### HeartBeat and Ping — Independent in V2

In V1, HeartBeat and Ping had mutual exclusion: if one alarm was raised, the other was skipped. **In V2, this mutual exclusion is removed.** Each interface sends its events independently:

| Interface | Monitored by | Event sent |
|-----------|-------------|------------|
| Out-of-band | Ping timer (100s) | reason=OUTOFBAND |
| In-band | Ping timer (100s) | reason=INBAND |
| GELF | Heartbeat timer (90s) | reason=GELF |

All three are **independent**. If GELF is already down and then OOB also fails, the OOB DOWN event is still sent. Switch-core needs to know about each interface separately to raise the correct `NeCommunicationInterfaceStateChange` alarm for each.

`NOCOMMUNICATION` is only sent when **all three** (OOB + IB + GELF) are simultaneously down. This tells switch-core to raise `NeReachabilityStateChange` and set operationalState = NOT_MANAGEABLE.

---

## 5. V1 vs V2 — What Changed and Why

### V1 (Old Behavior):

```
checkSpecificNeConnectivity():
  statusCode, err := c.switchclient.Ping(ctx, metadata)  // Pings BOTH interfaces as ONE call
  if err != nil:
    handlePingFailure()   // One failure = alarm raised
  else:
    handlePingSuccess()   // Both OK = alarm cleared
```

**Problem:** The old `Ping()` method checked both out-of-band AND in-band. If EITHER failed, it returned an error. So losing just one interface = "switch unreachable" alarm. But the switch was still reachable via the other interface!

### V2 (New Behavior):

```
checkSpecificNeConnectivity():
  _, oobErr := c.switchclient.PingOutOfBand(ctx, metadata)  // Check out-of-band SEPARATELY
  _, ibErr  := c.switchclient.PingInBand(ctx, metadata)      // Check in-band SEPARATELY

  // Handle EACH interface independently (NO mutual exclusion with other interfaces)
  if oobErr → handleInterfaceFailure(OUTOFBAND)   // sends NeLostIndication with reason=OUTOFBAND
  if ibErr  → handleInterfaceFailure(INBAND)       // sends NeLostIndication with reason=INBAND

  // GELF is checked separately by HeartBeatCheckTicker (sends reason=GELF)

  // ONLY when ALL THREE are down (OOB + IB + GELF) → "truly unreachable"
  if allDown → sendNeLostIndication(DOWN, NOCOMMUNICATION)
```

### What messages are sent in different scenarios:

| Scenario | V1 Messages | V2 Messages |
|----------|------------|------------|
| Out-of-band fails, in-band OK | 1 alarm: "switch unreachable" ❌ | 1 event: reason=OUTOFBAND ✅ |
| In-band fails, out-of-band OK | 1 alarm: "switch unreachable" ❌ | 1 event: reason=INBAND ✅ |
| GELF heartbeats stop | 1 alarm: "switch unreachable" ❌ | 1 event: reason=GELF ✅ |
| OOB+IB fail, GELF still OK | 1 alarm: "switch unreachable" | 2 events: reason=OUTOFBAND + reason=INBAND (NO NOCOMMUNICATION since GELF is still up) |
| ALL THREE fail (OOB+IB+GELF) | 1 alarm: "switch unreachable" | 4 events: reason=OUTOFBAND + reason=INBAND + reason=GELF + reason=NOCOMMUNICATION |
| Out-of-band recovers from full outage | 1 alarm: "switch reachable" | 2 events: reason=OUTOFBAND UP + reason=NOCOMMUNICATION UP |
| GELF heartbeats resume | 1 alarm: "switch reachable" | 1-2 events: reason=GELF UP + possibly NOCOMMUNICATION UP |

---

## 6. The Mock Switch Client — For Testing

### Why do we need a mock?

When running **unit tests**, we don't have real switches to ping. So we use a "fake" switch client that pretends to respond.

### Where is it?

`pkg/clients/switchclient/mocks/mockswitchclient.go`

### How is it generated?

The file header says `// Code generated by MockGen. DO NOT EDIT.` — it's auto-generated from the `SwitchClient` interface using the [gomock](https://github.com/golang/mock) library.

To regenerate it (after adding `PingInBand` to the interface):
```bash
mockgen -source=api.go -destination=mocks/mockswitchclient.go -package=mocks
```

### How does a test use the mock?

```go
func TestCheckSpecificNeConnectivity(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // Create the mock
    mockClient := mocks.NewMockSwitchClient(ctrl)

    // Tell the mock what to expect:
    // "When PingOutOfBand is called, return success"
    mockClient.EXPECT().PingOutOfBand(gomock.Any(), gomock.Any()).Return(200, nil)
    
    // "When PingInBand is called, return failure"
    mockClient.EXPECT().PingInBand(gomock.Any(), gomock.Any()).Return(0, fmt.Errorf("timeout"))

    // Now run the code under test — it will use the mock instead of real HTTP calls
    check := &Check{switchclient: mockClient, ...}
    check.checkSpecificNeConnectivity(ctx, "ABC123", neStatus)
    
    // Verify: should have raised alarm for INBAND but not OUTOFBAND
}
```

### The three ping mock methods:

| Mock Method | What It Fakes |
|-------------|---------------|
| `mockClient.EXPECT().Ping(...)` | The old V1 combined ping (both interfaces) |
| `mockClient.EXPECT().PingOutOfBand(...)` | Out-of-band only ping |
| `mockClient.EXPECT().PingInBand(...)` | In-band only ping (NEW in V2) |

---

## 7. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        rtbrick-rbfs-actor                               │
│                                                                         │
│  ┌──────────────┐    ┌───────────────────────────────────────────────┐  │
│  │  Redis DB     │    │  neConnectivity.go                            │  │
│  │              │    │                                               │  │
│  │ /pao/neconfig│◄───│  1. PerformLocalReconciliation()              │  │
│  │ /SN-ABC123   │    │     - reads switch list from Redis            │  │
│  │              │    │     - syncs with local cache                   │  │
│  │ nestatus/    │◄──►│                                               │  │
│  │ SN-ABC123    │    │  2. checkForConnectivity()                    │  │
│  │ (NeStatus)   │    │     - for each switch in cache:               │  │
│  └──────────────┘    │                                               │  │
│                      │  3. checkSpecificNeConnectivity()              │  │
│                      │     │                                         │  │
│                      │     ├─► PingOutOfBand(metadata) ──► Switch    │  │
│  ┌──────────────┐    │     │   GET https://<oob-ip>:12321/ping       │  │
│  │ switchclient  │    │     │                                         │  │
│  │ client_       │◄───│     ├─► PingInBand(metadata) ───► Switch     │  │
│  │ systemops.go  │    │     │   GET https://<ib-ip>:12321/ping        │  │
│  └──────────────┘    │     │                                         │  │
│                      │     ▼                                         │  │
│                      │  4. handleInterfaceFailure/Success()           │  │
│                      │     │                                         │  │
│                      │     ▼                                         │  │
│                      │  5. sendNeLostIndication()                    │  │
│                      │     │                                         │  │
│                      │     │  NeLostIndication {                     │  │
│                      │     │    serial: "ABC123",                    │  │
│                      │     │    state: DOWN,                         │  │
│                      │     │    reason: OUTOFBAND                    │  │
│                      │     │  }                                      │  │
│                      │     │                                         │  │
│                      └─────┼─────────────────────────────────────────┘  │
│                            │                                            │
│                            ▼                                            │
│                   ┌────────────────┐                                    │
│                   │  Kafka Bus      │                                    │
│                   │  Topic:         │                                    │
│                   │  NeLostIndica-  │                                    │
│                   │  tionTopic      │                                    │
│                   └───────┬────────┘                                    │
└───────────────────────────┼─────────────────────────────────────────────┘
                            │
                            │ (message travels through Kafka)
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                         pao-switch-core                                │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  Kafka Consumer Registration (in bootstrap code):               │  │
│  │                                                                 │  │
│  │  bus.NewTopicConsumerPair(                                      │  │
│  │    bus.NeLostIndicationTopic,         ← same topic name!        │  │
│  │    assurance.NewNeReachabilityAlarm(swCore),  ← the handler     │  │
│  │  )                                                              │  │
│  └──────────────────────────────┬──────────────────────────────────┘  │
│                                 │                                      │
│                                 ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │  neReachabilityAlarm.go → On(ctx, eventData)                  │     │
│  │                                                               │     │
│  │  1. parseNeLostIndication(eventData)                          │     │
│  │     - unmarshals JSON back to NeLostIndication struct         │     │
│  │     - reads serial_number, state, reason                      │     │
│  │                                                               │     │
│  │  2. FetchNetworkElementByZtpIdent(serial)                     │     │
│  │     - looks up the NE in inventory by serial number           │     │
│  │                                                               │     │
│  │  3. Based on reason:                                          │     │
│  │     INBAND/OUTOFBAND/GELF                                     │     │
│  │       → raise/clear NeCommunicationInterfaceStateChange alarm │     │
│  │     NOCOMMUNICATION                                           │     │
│  │       → raise/clear NeReachabilityStateChange alarm           │     │
│  │                                                               │     │
│  │  4. Produce alarm CloudEvent to "alarms" Kafka topic          │     │
│  └──────────────────────────────────────────────────────────────┘     │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Where Does pao-switch-core Pick This Up?

### Registration

In pao-switch-core's bootstrap code, there's a line like:

```go
bus.NewTopicConsumerPair(bus.NeLostIndicationTopic, assurance.NewNeReachabilityAlarm(swCore))
```

This means: "When a message arrives on the `NeLostIndication` Kafka topic, call the `On()` method of `neReachabilityAlarm`."

### The Handler

In `pao-switch-core/internal/core/assurance/neReachabilityAlarm.go`:

```go
func (c neReachabilityAlarm) On(ctx context.Context, eventData []byte) {
    // 1. Parse the protobuf JSON
    ind := parseNeLostIndication(eventData)
    // ind.SerialNumber = "ABC123"
    // ind.State = CONNECTION_STATE_DOWN
    // ind.Reason = COMMUNICATION_LOSS_REASON_OUTOFBAND

    // 2. Look up the NE in inventory
    ne := c.switchCore.GetInventoryClient().FetchNetworkElementByZtpIdent(ctx, ind.SerialNumber)

    // 3. Based on reason, decide which alarm to create
    switch ind.Reason {
    case OUTOFBAND, INBAND, GELF:
        // → NeCommunicationInterfaceStateChange alarm
        //   with sourceObjectAdditionalKey = "OUT_OF_BAND_MGMT" / "IN_BAND_MGMT" / "GELF"
    case NOCOMMUNICATION:
        // → NeReachabilityStateChange alarm
        //   sets operationalState = NOT_MANAGEABLE (down) or WORKING (up)
    }

    // 4. Publish to "alarms" Kafka topic → goes to BEM (Business Event Manager)
}
```

### The complete journey of a message:

```
Switch doesn't respond to out-of-band ping
  → rtbrick-rbfs-actor creates NeLostIndication{serial:"ABC123", state:DOWN, reason:OUTOFBAND}
    → serialized to JSON bytes
      → published to Kafka topic "NeLostIndication"
        → pao-switch-core consumes from Kafka topic "NeLostIndication"
          → On() method parses the JSON back to NeLostIndication struct
            → looks up NE in inventory by serial number
              → creates NeCommunicationInterfaceStateChange alarm
                → publishes alarm CloudEvent to Kafka "alarms" topic
                  → BEM stores the alarm
                    → inventory updates NE status
```

---

## 9. NeStatus Struct — The "Memory" of Each Switch

The `NeStatus` struct is stored in Redis and local cache for each switch. It remembers the current alarm state:

```go
type NeStatus struct {
    // Legacy V1 fields (still used for heartbeat compatibility):
    PingAlarmRaised      bool  // V1 combined ping alarm
    HeartBeatAlarmRaised bool  // heartbeat-based alarm
    FailureCount         int   // V1 combined failure counter

    // NEW V2 fields:
    OutOfBandAlarmRaised bool  // is out-of-band alarm currently raised?
    InBandAlarmRaised    bool  // is in-band alarm currently raised?
    OutOfBandFailCount   int   // consecutive out-of-band ping failures
    InBandFailCount      int   // consecutive in-band ping failures
    NoCommunication      bool  // is NOCOMMUNICATION alarm raised? (both down)
}
```

### Example lifecycle — Ping-based (Out-of-band + In-band):

```
Time 0:   NeStatus{OOB:false, IB:false, NoCom:false, HB:false}    — all OK
Time 100: Out-of-band fails (count=1)                               — not yet alarmed
Time 200: Out-of-band fails (count=2)                               — not yet alarmed
Time 300: Out-of-band fails (count=3)                               — EVENT! Send DOWN+OUTOFBAND
          NeStatus{OOB:true, IB:false, NoCom:false, HB:false}
Time 400: In-band also fails (count=1)                              — not yet for IB
Time 500: In-band fails (count=2)
Time 600: In-band fails (count=3)                                   — EVENT! Send DOWN+INBAND
          NeStatus{OOB:true, IB:true, NoCom:false, HB:false}
          Note: NOCOMMUNICATION NOT sent yet — GELF is still up!
Time 650: GELF heartbeat also fails (detected by heartbeat timer)   — EVENT! Send DOWN+GELF
          Now all three are down                                     — EVENT! Send DOWN+NOCOMMUNICATION
          NeStatus{OOB:true, IB:true, NoCom:true, HB:true}
Time 700: Out-of-band recovers!                                     — EVENT! Send UP+OUTOFBAND
          Not all three down anymore                                 — EVENT! Send UP+NOCOMMUNICATION
          NeStatus{OOB:false, IB:true, NoCom:false, HB:true}
```

### Example lifecycle — GELF heartbeat:

```
Time 0:   HeartBeatAlarmRaised=false, Redis counter=0     — normal
Time 45:  Switch sends HTB0001                             — counter=1
Time 90:  checkHeartBeatCount: count=1 (<2)                — count too low, but
          PingAlarmRaised=false, HeartBeatAlarmRaised=false
          → ALARM! Send DOWN+GELF                          — NeStatus{HB:true}
Time 135: Switch still not sending heartbeats               — counter stays 0
Time 180: checkHeartBeatCount: count=0 (<2)                — alarm already raised, skip
Time 225: Switch starts sending heartbeats again            — counter=1
Time 270: Switch sends another heartbeat                    — counter=2
Time 270: checkHeartBeatCount: count=2 (≥2)                — CLEAR! Send UP+GELF
          NeStatus{HB:false}
```

---

## 10. Glossary

| Term | What it means |
|------|---------------|
| **NE** | Network Element — a physical device like a switch or server |
| **Serial Number** | Unique ID of a switch, used to identify it (like a license plate) |
| **Out-of-band (OOB)** | A dedicated management network, separate from regular traffic. Like having a private phone line to each office |
| **In-band (IB)** | Management over the same network as regular data traffic. Like using the office's regular phone |
| **Protobuf** | Protocol Buffers — a way to define message formats that different systems can share |
| **Kafka** | A message bus — like a post office. You publish messages to "topics" and other services subscribe to those topics |
| **NeLostIndication** | The specific protobuf message type used to report "this switch's interface is down/up" |
| **CommunicationLossReason** | An enum telling WHY the message was sent (which interface failed) |
| **Redis** | A fast in-memory database used to store switch metadata and alarm states |
| **NeStatus** | A struct stored in Redis/cache that remembers if alarms are raised for each switch |
| **BEM** | Business Event Manager — the system that stores and queries alarm events |
| **CloudEvent** | A standard format for events, used by pao-switch-core to wrap alarms |
| **gomock / MockGen** | A Go testing library that auto-generates fake implementations of interfaces |
| **GELF** | Graylog Extended Log Format — a log streaming interface to the switch |
| **BMC** | Baseboard Management Controller — a hardware management interface on servers |
| **Threshold (numPingFailures)** | Number of consecutive failures before raising an alarm (default: 3) |
| **Ticker** | A Go `time.Ticker` that fires at regular intervals — drives the ping loop |
| **Reconciliation** | Syncing the local cache with the inventory in Redis — adding new switches, removing deleted ones |
