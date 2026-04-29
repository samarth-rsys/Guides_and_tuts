# NE Monitoring V2 — Alarm Flow & Required Changes

## Table of Contents

1. [Overview: V1 vs V2](#overview-v1-vs-v2)
2. [How the Current (V1) Alarm Flow Works — Step by Step](#how-the-current-v1-alarm-flow-works)
3. [What Changes in V2](#what-changes-in-v2)
4. [New Alarm: NeCommunicationInterfaceStateChange](#new-alarm-necommunicationinterfacestatechange)
5. [Updated Alarm: NeReachabilityStateChange (Fabric Switches Only)](#updated-alarm-nereachabilitystatechange-fabric-switches-only)
6. [KubernetesNodeDown → operationalStateA4Local for POD_SERVER](#kubernetesnodedown--operationalstatea4local-for-pod_server)
7. [Detailed Code Changes Required](#detailed-code-changes-required)
8. [End-to-End Flow Diagrams](#end-to-end-flow-diagrams)

---

## Overview: V1 vs V2

### V1 (Current Behavior)

- A single alarm **`NeReachabilityStateChange`** is raised when **ANY one** management interface goes down.
- Severity: `CRITICAL` (raised) / `INFO` (cleared).
- NE `operationalState` is set to `NOT_MANAGEABLE` on CRITICAL, `WORKING` on INFO.
- Problem: A single interface going down doesn't mean the device is truly unreachable — the other interface may still be working.

### V2 (New Behavior)

| Alarm | When Raised | When Cleared | Applies To |
|---|---|---|---|
| **NeCommunicationInterfaceStateChange** | A **specific** interface goes down | That **specific** interface comes back up | Fabric Switch: `OUT_OF_BAND_MGMT`, `IN_BAND_MGMT`, `GELF`; POD_SERVER: `BMC`; LI_BOX: `BMC` |
| **NeReachabilityStateChange** | **ALL** management interfaces (in-band + out-of-band) are down | **ANY** management interface comes back up | `LEAF_SWITCH`, `SPINE_SWITCH`, `A10NSP_SWITCH` only |

> **Key difference:** In V1, losing one interface = device unreachable. In V2, losing one interface = just that interface is down (communication alarm). The device is only "unreachable" when ALL interfaces are down.

---

## How the Current (V1) Alarm Flow Works

Think of it as a chain of 4 systems:

```
 ┌──────────────┐        ┌──────────────────────┐        ┌─────────────────┐       ┌───────────┐
 │  Fabric      │  gRPC  │  rtbrick-rbfs-actor   │ Kafka  │  pao-switch-core │ Kafka │  BEM /    │
 │  Switch      │◄──────►│  (Adapter)            │──────► │  (Controller)    │──────►│  Inventory│
 │  (Hardware)  │  ping  │                       │        │                  │       │           │
 └──────────────┘        └──────────────────────┘        └─────────────────┘       └───────────┘
```

### Step-by-step:

#### Step 1: Adapter pings the switch

**Where:** `rtbrick-rbfs-actor` → `pkg/adapters/client/neconnectivity.go`

The adapter runs a **periodic ping loop** to check if each fabric switch is reachable:
- It pings both the **out-of-band** and **in-band** management IP addresses.
- It tracks ping failures with a counter per NE.

#### Step 2: Ping fails → Adapter sends `NeLostIndication` to Kafka

When the ping failure count reaches a threshold (default: 3 consecutive failures):

```go
// In rtbrick-rbfs-actor/pkg/adapters/client/neconnectivity.go
func (c *Check) sendNeLostIndication(ctx, serialNumber, state, reason) {
    NeLostIndication := &api.NeLostIndication{
        SerialNumber: serialNumber,
        State:        state,    // CONNECTION_STATE_DOWN or CONNECTION_STATE_UP
        Reason:       reason,   // Currently a string (needs to become enum)
    }
    // Serialize to JSON and publish to Kafka topic "NeLostIndication"
    bus.Publish(bus.NeLostIndicationTopic, marshaledBytes)
}
```

This creates a protobuf message and publishes it to the **Kafka topic `NeLostIndication`**.

#### Step 3: Switch-Core consumes the Kafka message

**Where:** `pao-switch-core` → `internal/core/assurance/neReachabilityAlarm.go`

Switch-core has a Kafka consumer registered for the `NeLostIndication` topic:

```
bus.NewTopicConsumerPair(bus.NeLostIndicationTopic, assurance.NewNeReachabilityAlarm(swCore))
```

When a message arrives, the `On()` method is called:

1. **Parse** the `NeLostIndication` protobuf message (serial number, state, reason)
2. **Look up the NE** in inventory using the serial number (`FetchNetworkElementByZtpIdent`)
3. **Validate** the NE: must have hostname, NE type, and be in `Operating` lifecycle state
4. **Check for duplicate**: Query BEM to see if the same alarm already exists
5. **Raise or Clear** the alarm based on `state`:
   - `CONNECTION_STATE_DOWN` → raise `NeReachabilityStateChange` alarm (severity: `CRITICAL`)
   - `CONNECTION_STATE_UP` → clear `NeReachabilityStateChange` alarm (severity: `INFO`)

#### Step 4: Alarm is published to Kafka "alarms" topic

The alarm is built as a **CloudEvent** and published:

```go
event := alarms.NewNeReachabilityStateChange()
// ... set fields ...
builder.WithPerceivedSeverity("CRITICAL" or "INFO")
c.switchCore.GetAlarmPipeline().Produce(payload)
```

This goes to the BEM (Business Event Manager) and inventory systems, which update the NE's `operationalState`.

---

## What Changes in V2

### The `Reason` Field in NeLostIndication

The protobuf already defines a `CommunicationLossReason` enum in `pao-switch-api-defs/adapter/protos/v1/indications.proto`:

```protobuf
enum CommunicationLossReason {
    COMMUNICATION_LOSS_REASON_UNSPECIFIED     = 0;
    COMMUNICATION_LOSS_REASON_INBAND          = 1;  // In-band management interface
    COMMUNICATION_LOSS_REASON_OUTOFBAND       = 2;  // Out-of-band management interface
    COMMUNICATION_LOSS_REASON_GELF            = 3;  // GELF log interface
    COMMUNICATION_LOSS_REASON_NOCOMMUNICATION = 4;  // ALL interfaces are down
}
```

> ⚠️ **Issue Found:** The proto defines `reason` as the `CommunicationLossReason` enum, but the generated Go code (`indications.pb.go`) has it as a `string`. **The Go code needs to be regenerated** from the proto.

---

## New Alarm: NeCommunicationInterfaceStateChange

This is a **new alarm type** that does not exist yet. It needs to be created.

### When is it raised?

| NE Type | Interface | Raised When | Cleared When |
|---|---|---|---|
| Fabric Switch (LEAF/SPINE/A10NSP) | `OUT_OF_BAND_MGMT` | Out-of-band ping fails | Out-of-band ping succeeds |
| Fabric Switch (LEAF/SPINE/A10NSP) | `IN_BAND_MGMT` | In-band ping fails | In-band ping succeeds |
| Fabric Switch (LEAF/SPINE/A10NSP) | `GELF` | GELF connection lost | GELF connection restored |
| POD_SERVER | `BMC` | BMC interface down | BMC interface up |
| LI_BOX | `BMC` | BMC interface down | BMC interface up |

### Key fields:
- **`sourceObjectAdditionalKey`**: `OUT_OF_BAND_MGMT` | `IN_BAND_MGMT` | `GELF` | `BMC`
- **`perceivedSeverity`**: `CRITICAL` (raised) / `INFO` (cleared)

---

## Updated Alarm: NeReachabilityStateChange (Fabric Switches Only)

### New behavior for LEAF_SWITCH, SPINE_SWITCH, A10NSP_SWITCH:

- **Raised** ONLY when BOTH in-band AND out-of-band are down → `NOCOMMUNICATION`
- **Cleared** when ANY of in-band OR out-of-band comes back up
- NE `operationalState` → `NOT_MANAGEABLE` (raised) / `WORKING` (cleared)

### No change for OLT, DPU, BOR:
These devices have a single management interface, so the V1 behavior remains unchanged.

---

## KubernetesNodeDown → operationalStateA4Local for POD_SERVER

When a `KubernetesNodeDown` alarm is reported for a POD_SERVER NE:
- `perceivedSeverity: CRITICAL` → set `operationalStateA4Local = FAILED`
- `perceivedSeverity: INFO` → set `operationalStateA4Local = WORKING`

---

## Detailed Code Changes Required

### 1. `pao-switch-api-defs` — Proto & Code Generation

| File | Change |
|---|---|
| `adapter/protos/v1/indications.proto` | ✅ Already has `CommunicationLossReason` enum and `reason` field — **no proto change needed** |
| `adapter/generated/v1/indications.pb.go` | ⚠️ **Regenerate** — current generated code has `Reason` as `string` instead of `CommunicationLossReason` enum |

**Action:** Run `make generate` or `buf generate` to regenerate the Go code from protos.

---

### 2. `rtbrick-rbfs-actor` — Adapter (Sends NeLostIndication)

**File:** `pkg/adapters/client/neconnectivity.go`

| Current | New |
|---|---|
| Sends NeLostIndication with `Reason` as empty string or generic text | Must send specific `CommunicationLossReason` enum value |
| Raises alarm when ANY interface fails | Must track each interface independently |

**Changes needed:**

```
a) Track per-interface state:
   - outOfBandReachable: bool
   - inBandReachable: bool
   - gelfConnected: bool

b) When out-of-band ping fails:
   → sendNeLostIndication(serial, DOWN, COMMUNICATION_LOSS_REASON_OUTOFBAND)

c) When in-band ping fails:
   → sendNeLostIndication(serial, DOWN, COMMUNICATION_LOSS_REASON_INBAND)

d) When GELF connection lost:
   → sendNeLostIndication(serial, DOWN, COMMUNICATION_LOSS_REASON_GELF)

e) When ALL interfaces are down:
   → sendNeLostIndication(serial, DOWN, COMMUNICATION_LOSS_REASON_NOCOMMUNICATION)

f) When any interface recovers:
   → sendNeLostIndication(serial, UP, <which_interface_recovered>)
   → If previously NOCOMMUNICATION, also send UP with NOCOMMUNICATION reason
```

---

### 3. `pao-switch-core` — Alarm Handler (Consumes NeLostIndication)

**File:** `internal/core/assurance/neReachabilityAlarm.go`

This is the **biggest change**. The handler must now produce **two different alarm types** based on the `reason` field.

#### Current logic (V1):
```
NeLostIndication received →
  if DOWN → raise NeReachabilityStateChange (CRITICAL)
  if UP   → clear NeReachabilityStateChange (INFO)
```

#### New logic (V2):

```
NeLostIndication received → check NE type and reason:

IF reason = INBAND | OUTOFBAND | GELF:
  ├── if DOWN → raise NeCommunicationInterfaceStateChange (CRITICAL)
  │              with sourceObjectAdditionalKey = OUT_OF_BAND_MGMT / IN_BAND_MGMT / GELF
  └── if UP   → clear NeCommunicationInterfaceStateChange (INFO)
               with sourceObjectAdditionalKey = OUT_OF_BAND_MGMT / IN_BAND_MGMT / GELF

IF reason = NOCOMMUNICATION (ALL interfaces down):
  ├── NE type is LEAF_SWITCH / SPINE_SWITCH / A10NSP_SWITCH:
  │   ├── if DOWN → raise NeReachabilityStateChange (CRITICAL)
  │   │              set operationalState = NOT_MANAGEABLE
  │   └── if UP   → clear NeReachabilityStateChange (INFO)
  │                  set operationalState = WORKING
  └── NE type is OLT / DPU / BOR:
      └── (same as V1 — single interface, no change)

IF reason = BMC (for POD_SERVER / LI_BOX):
  ├── if DOWN → raise NeCommunicationInterfaceStateChange (CRITICAL)
  │              with sourceObjectAdditionalKey = BMC
  └── if UP   → clear NeCommunicationInterfaceStateChange (INFO)
               with sourceObjectAdditionalKey = BMC
```

**Specific code changes:**

| What | Where | Details |
|---|---|---|
| Read `reason` field from `NeLostIndication` | `parseNeLostIndication()` | Use `ind.Reason` (will be enum after regeneration) |
| New alarm builder | New function | `triggerCommunicationInterfaceAlarm()` using `alarms.NewNeCommunicationInterfaceStateChange()` (needs to be registered in pod-events library) |
| Route by reason | `On()` method | Add switch on `ind.Reason` to decide which alarm to raise |
| Map reason → sourceObjectAdditionalKey | New mapping | `INBAND → IN_BAND_MGMT`, `OUTOFBAND → OUT_OF_BAND_MGMT`, `GELF → GELF`, `BMC → BMC` |
| NE type check for NeReachabilityStateChange | `On()` method | Only raise for `LEAF_SWITCH`, `SPINE_SWITCH`, `A10NSP_SWITCH` |

---

### 4. `pod-events` Library (External Dependency)

A new alarm schema **`NeCommunicationInterfaceStateChange`** must be registered:

- Add `NeCommunicationInterfaceStateChange_1` schema definition
- Add `NewNeCommunicationInterfaceStateChange()` factory function
- Define the alarm with `sourceObjectAdditionalKey` support

---

### 5. `pao-switch-core` — KubernetesNodeDown Handler

A new or updated handler for `KubernetesNodeDown` alarm for POD_SERVER:

- On `CRITICAL` → `PatchOperationalStateA4Local(neId, "FAILED")`
- On `INFO` → `PatchOperationalStateA4Local(neId, "WORKING")`

---

## End-to-End Flow Diagrams

### Flow 1: Single Interface Down (e.g., Out-of-Band fails)

```
  Fabric Switch                rtbrick-rbfs-actor              pao-switch-core                BEM
       │                             │                               │                        │
       │  out-of-band ping fails     │                               │                        │
       │  (in-band still works)      │                               │                        │
       │                             │                               │                        │
       │◄─── ping timeout ──────────►│                               │                        │
       │                             │                               │                        │
       │                             │ sendNeLostIndication(          │                        │
       │                             │   serial, DOWN,               │                        │
       │                             │   OUTOFBAND)                  │                        │
       │                             │──── Kafka NeLostInd ─────────►│                        │
       │                             │                               │                        │
       │                             │                               │ Parse reason=OUTOFBAND  │
       │                             │                               │ NE type=LEAF_SWITCH     │
       │                             │                               │                        │
       │                             │                               │ Raise                   │
       │                             │                               │ NeCommunicationInterface│
       │                             │                               │ StateChange             │
       │                             │                               │ severity=CRITICAL       │
       │                             │                               │ additionalKey=          │
       │                             │                               │   OUT_OF_BAND_MGMT     │
       │                             │                               │────── Kafka alarms ────►│
       │                             │                               │                        │
       │                             │                               │ (NO NeReachability     │
       │                             │                               │  alarm — in-band is    │
       │                             │                               │  still up)             │
```

### Flow 2: All Interfaces Down → Device Unreachable

```
  Fabric Switch                rtbrick-rbfs-actor              pao-switch-core                BEM
       │                             │                               │                        │
       │  out-of-band already down   │                               │                        │
       │  now in-band also fails     │                               │                        │
       │                             │                               │                        │
       │◄─── ping timeout ──────────►│                               │                        │
       │                             │                               │                        │
       │                             │ sendNeLostIndication(          │                        │
       │                             │   serial, DOWN,               │                        │
       │                             │   INBAND)                     │                        │
       │                             │──── Kafka NeLostInd ─────────►│                        │
       │                             │                               │ Raise NeCommunication  │
       │                             │                               │ InterfaceStateChange   │
       │                             │                               │ additionalKey=         │
       │                             │                               │   IN_BAND_MGMT        │
       │                             │                               │────── Kafka alarms ────►│
       │                             │                               │                        │
       │                             │ sendNeLostIndication(          │                        │
       │                             │   serial, DOWN,               │                        │
       │                             │   NOCOMMUNICATION)            │                        │
       │                             │──── Kafka NeLostInd ─────────►│                        │
       │                             │                               │                        │
       │                             │                               │ Raise NeReachability   │
       │                             │                               │ StateChange            │
       │                             │                               │ severity=CRITICAL      │
       │                             │                               │────── Kafka alarms ────►│
       │                             │                               │                        │
       │                             │                               │ Set operationalState   │
       │                             │                               │   = NOT_MANAGEABLE     │
```

### Flow 3: Recovery — One Interface Comes Back

```
  Fabric Switch                rtbrick-rbfs-actor              pao-switch-core                BEM
       │                             │                               │                        │
       │  in-band recovers           │                               │                        │
       │  (out-of-band still down)   │                               │                        │
       │                             │                               │                        │
       │◄─── ping success ──────────►│                               │                        │
       │                             │                               │                        │
       │                             │ sendNeLostIndication(          │                        │
       │                             │   serial, UP,                 │                        │
       │                             │   INBAND)                     │                        │
       │                             │──── Kafka NeLostInd ─────────►│                        │
       │                             │                               │                        │
       │                             │                               │ Clear NeCommunication  │
       │                             │                               │ InterfaceStateChange   │
       │                             │                               │ additionalKey=         │
       │                             │                               │   IN_BAND_MGMT        │
       │                             │                               │────── Kafka alarms ────►│
       │                             │                               │                        │
       │                             │ sendNeLostIndication(          │                        │
       │                             │   serial, UP,                 │                        │
       │                             │   NOCOMMUNICATION)            │                        │
       │                             │──── Kafka NeLostInd ─────────►│                        │
       │                             │                               │                        │
       │                             │                               │ Clear NeReachability   │
       │                             │                               │ StateChange            │
       │                             │                               │ severity=INFO          │
       │                             │                               │────── Kafka alarms ────►│
       │                             │                               │                        │
       │                             │                               │ Set operationalState   │
       │                             │                               │   = WORKING            │
```

---

## Summary of All Changes by Repository

| Repository | Files to Change | Summary |
|---|---|---|
| **pao-switch-api-defs** | `adapter/generated/v1/indications.pb.go` | Regenerate Go code so `Reason` is `CommunicationLossReason` enum (not string) |
| **rtbrick-rbfs-actor** | `pkg/adapters/client/neconnectivity.go` | Track per-interface state; send correct `CommunicationLossReason` enum in `NeLostIndication`; send `NOCOMMUNICATION` when all interfaces fail |
| **pao-switch-core** | `internal/core/assurance/neReachabilityAlarm.go` | Route by `reason`: raise `NeCommunicationInterfaceStateChange` for individual interfaces, `NeReachabilityStateChange` only for `NOCOMMUNICATION`; add `sourceObjectAdditionalKey` |
| **pao-switch-core** | New file or update in `internal/core/assurance/` | Handle `KubernetesNodeDown` → set `operationalStateA4Local` to `FAILED`/`WORKING` for POD_SERVER |
| **pod-events** (external) | Schema definitions | Register new `NeCommunicationInterfaceStateChange` alarm schema |

---

## Glossary

| Term | Meaning |
|---|---|
| **NE** | Network Element — a physical device (switch, server, etc.) |
| **BEM** | Business Event Manager — stores and queries alarm events |
| **GELF** | Graylog Extended Log Format — log streaming interface |
| **BMC** | Baseboard Management Controller — hardware management interface on servers |
| **In-band** | Management traffic over the same network as data traffic |
| **Out-of-band** | Dedicated management network, separate from data traffic |
| **CloudEvent** | Standard event format used for alarm publishing |
| **ZTP Ident** | Zero Touch Provisioning identifier (serial number) |
| **operationalState** | Current operational status of the NE in inventory |
