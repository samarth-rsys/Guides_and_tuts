# Leaf Reboot Flow — A Beginner's Guide

> **Audience:** Developers new to the `pao-switch-core` repository.
> This document walks through **every step** that happens when a leaf switch
> reboots, from the moment it goes offline to the moment it is fully
> operational again.

---

## Table of Contents

1. [High-Level Overview (Diagram)](#1-high-level-overview)
2. [Architecture & Key Concepts](#2-architecture--key-concepts)
   - [The Three Main Systems (Adapter, pao-switch-core, EMS)](#21-the-three-main-systems)
   - [Communication Channels (Proto Bus & Event Pipeline)](#22-communication-channels)
   - [Resource Model Glossary](#23-resource-model-glossary)
3. [Step-by-Step Flow](#3-step-by-step-flow)
   - [Phase A — Switch Goes Down](#phase-a--switch-goes-down)
   - [Phase B — Switch Comes Back Up](#phase-b--switch-comes-back-up)
   - [Phase C — IP Assignment & Startup](#phase-c--ip-assignment--startup)
   - [Phase D — Configuration & Operational Data Patching](#phase-d--configuration--operational-data-patching)
   - [Phase E — Config Applied & Startup Finished](#phase-e--config-applied--startup-finished)
   - [Phase F — Post-Startup (Subscriber Restoration)](#phase-f--post-startup-subscriber-restoration)
4. [Kafka Topics & Event Subscriptions](#4-kafka-topics--event-subscriptions)
5. [Important Files & Functions Reference](#5-important-files--functions-reference)
6. [The PatchOperationalData(working) Bug (Fixed)](#6-the-patchoperationaldataworking-bug)
7. [Glossary](#7-glossary)

---

## 1. High-Level Overview

```
Leaf Switch                    pao-switch-core                       EMS / Inventory
     |                               |                                    |
     |--- reboots ------------------>|                                    |
     |                               |                                    |
  PHASE A: Switch Goes Down          |                                    |
     |  NeLostIndication(DOWN)  ---->|                                    |
     |                               |-- raise CRITICAL alarm ---------->|
     |                               |                                    |
  PHASE B: Switch Comes Back Up      |                                    |
     |  NeLostIndication(UP)    ---->|                                    |
     |                               |-- clear alarm (INFO) ------------>|
     |  SystemUpIndication      ---->|                                    |
     |                               |-- emit NeStartupConfigLoaded ---->|
     |                               |                                    |
  PHASE C: IP Assignment & Startup   |                                    |
     |  IpAddressAssignment     ---->|                                    |
     |                               |-- emit StartupStarted ---------->|
     |                               |                                    |
  PHASE D: Config & OpData Patching  |                                    |
     |                          <----|  ObjectChanged events from EMS     |
     |                               |-- blacklist/whitelist filter       |
     |                               |-- generate config                  |
     |                               |-- apply config to adapter          |
     |                               |-- patch DHCP options ------------->|
     |                               |-- patch artefact versions -------->|
     |                               |-- patch speed (if needed) -------->|
     |                               |                                    |
  PHASE E: Config Applied            |                                    |
     |  RunningConfigStatusInd  ---->|                                    |
     |  (REASON_AFTER_REBOOT)        |-- emit StartupFinished ---------->|
     |                               |                                    |
  PHASE F: Post-Startup              |                                    |
     |                               |<-- StartupFinished event          |
     |                               |-- clear NeStartupFailedAlarm ---->|
     |                               |-- restore retail subscribers       |
     |                               |-- restore L2BSA subscribers        |
     |                               |                                    |
     |  ✅ Leaf is fully operational  |                                    |
```

---

## 2. Architecture & Key Concepts

Before diving into the reboot flow, let's understand the **three main systems**
that talk to each other and the communication channels between them.

### 2.1 The Three Main Systems

```
┌────────────────────┐       gRPC        ┌──────────────────────┐    HTTP/REST    ┌────────────────────┐
│                    │ ◄───────────────► │                      │ ◄────────────► │                    │
│   ADAPTER          │                   │   pao-switch-core    │                │   EMS / Inventory  │
│   (rbfs-adapter)   │                   │   (this repo)        │                │   (pod-inventory)  │
│                    │                   │                      │                │                    │
│ Talks directly to  │   Kafka (proto)   │  The "brain" that    │  Kafka (events)│  The database of   │
│ the physical       │ ─────────────────►│  orchestrates the    │ ◄─────────────►│  all network       │
│ switch hardware    │  NeLostIndication  │  entire boot flow    │  ObjectChanged  │  resources (NEs,   │
│                    │  SystemUpIndication│                      │  PatchOpData    │  NEPs, NELs, etc.) │
└────────────────────┘  RunningConfigInd  └──────────────────────┘                └────────────────────┘
```

#### 🔧 What is the Adapter?

The **adapter** (also called `rbfs-adapter`) is a separate microservice that
acts as a **translator between pao-switch-core and the physical switch**.

Think of it like this:
- pao-switch-core says: *"Apply this configuration to switch XYZ"*
- The adapter knows *how* to actually talk to the hardware (via RBFS CLI,
  NETCONF, or vendor-specific protocols)
- The adapter reports back: *"Configuration applied successfully"*

**pao-switch-core communicates with the adapter via gRPC.** There are 7 gRPC
client types, each for a different domain:

| gRPC Client | What it controls |
|-------------|------------------|
| `SystemConfigurationsClient` | Main config operations: set, get, delete, generate startup config, apply config, decommission |
| `ArtefactsClient` | Software/firmware version management |
| `VlanClient` | VLAN configuration |
| `DhcpOptionsClient` | DHCP relay options |
| `IfAdminModeClient` | Interface admin up/down |
| `L2BsaClient` | L2 Bitstream Access (wholesale broadband) |
| `L2xconnectClient` | Layer 2 cross-connects |
| `LawfulInterceptClient` | Lawful intercept tap configuration |

**Key gRPC RPCs sent to the adapter:**

| RPC | When it's used |
|-----|---------------|
| `SetConfiguration()` | Push incremental config changes |
| `GenerateStartupConfiguration()` | Ask the adapter to build a full config |
| `ApplyConfiguration()` | Tell the adapter to commit config to the switch |
| `DeleteConfiguration()` | Remove config entries |
| `DecommissionSwitch()` | Wipe a switch clean |
| `SetBringUp()` | First-time provisioning of a new switch |

**The adapter also SENDS messages TO pao-switch-core** via Kafka protobuf
topics (not gRPC — these are asynchronous notifications):

| Kafka Message | What it means |
|---------------|---------------|
| `NeLostIndication` | *"I lost/regained connection to the switch"* |
| `SystemUpIndication` | *"The switch just booted up and loaded its startup config"* |
| `RunningConfigStatusInd` | *"The running config was applied (after reboot/upgrade)"* |
| `UpgradeStatusInd` | *"A software upgrade finished (success/failure)"* |

> 📁 Adapter gRPC clients: `internal/adaptergw/`
> 📁 Adapter simulator (for testing): `cmd/adapter-simulator/`

#### 🗄️ What is EMS / Inventory?

**EMS** (Element Management System) = **pod-inventory** = the central database
that stores everything about your network: every switch, every port, every
link, every configuration.

pao-switch-core talks to EMS in **two ways**:

**1. Reading data (HTTP REST):**

```
pao-switch-core  ──HTTP GET──►  pod-inventory
                                    │
                              Returns NE details:
                              - serial number
                              - hostname
                              - mat number
                              - NE type (leaf/spine/A10)
                              - lifecycle state
                              - operational data
                              - ports, links, etc.
```

Configured via environment variables:
- `POD_INVENTORY_ADDRESS` — base URL of the inventory service
- `POD_INVENTORY_TIMEOUT` — HTTP timeout
- `POD_INVENTORY_RETRIES` — max retries

Key functions:
- `FetchNetworkElementByZtpIdent(serial)` — look up a switch by serial number
- `FetchResourceById(id)` — look up any resource by ID
- `GetAllSwitches()`, `GetAllSpines()`, `ResolveSwitches()` — bulk queries

> 📁 Inventory client: `pkg/clients/inventory/`

**2. Writing data (Event-based PatchOperationalData):**

When pao-switch-core needs to **update** a field in inventory (e.g., set the
port speed, update DHCP options, update artefact versions), it does NOT make
a direct HTTP PUT/PATCH call. Instead:

```
pao-switch-core  ──publishes event──►  Kafka/NATS message bus
                                            │
                                    pod-inventory consumes
                                    the event and applies
                                    the patch to its database
                                            │
                                    pod-inventory emits
                                    PatchOperationalDataSuccess(working)
                                    ──────────────────────►  (back on the bus)
```

This is **asynchronous** — pao-switch-core fires-and-forgets the patch event.
The inventory service processes it in the background. This is why after a
reboot, if you send the same patch twice quickly, you get two
`PatchOperationalDataSuccess(working)` events (the bug we fixed!).

> 📁 PatchOperationalData client: `pkg/clients/patchoperationaldata/`

**3. Receiving change notifications (Event Pipeline):**

When *anything* changes in inventory (an operator edits a switch, a port comes
online, an IP gets assigned), inventory publishes `ObjectChanged`,
`ObjectsCreated`, `ObjectDeleted` events. pao-switch-core subscribes to these
and reacts accordingly.

### 2.2 Communication Channels

There are **two separate messaging systems** used in this project:

#### Proto Bus (Protobuf over Kafka)

Used for **adapter → pao-switch-core** communication.

- **Transport:** Kafka
- **Serialization:** Protocol Buffers (binary-efficient)
- **Consumer group:** `pao-switch-core-proto`
- **Library:** `pao-sha` shared library (wraps Kafka consumer/producer)
- **Env var:** `MESSAGE_BUS_TYPE` selects Kafka vs NATS backend

How it works:
```
Adapter publishes proto message  ──►  Kafka topic (e.g., "ne-lost-indication")
                                            │
                                    pao-switch-core's proto bus consumer
                                    picks up the message
                                            │
                                    Routes to handler via topic→handler map:
                                      NeLostIndicationTopic → neReachabilityAlarm.On()
                                      SystemUpIndicationTopic → systemUpIndicator.On()
                                      etc.
```

> 📁 Proto bus setup: `internal/bootstrap/app.go` → `setupProtoBusSubscriptions()`
> 📁 Message bus package: `pkg/messagebus/`

#### Event Pipeline (pod-actor framework)

Used for **inventory ↔ pao-switch-core** communication and internal events.

- **Transport:** Kafka or NATS (configurable)
- **Framework:** `pod-actor` — a pipeline library that provides:
  - Input sources (subscribe to topics)
  - Handlers (process messages by schema)
  - Output sinks (publish results)
- **Consumer group:** `pao-switch-core`

How it works:
```
Inventory publishes event  ──►  Kafka/NATS topic (e.g., "inventory.logical.default")
                                      │
                              pod-actor input layer receives it
                                      │
                              Dispatches by event schema:
                                ObjectChanged_1 → HandleObjectChanged()
                                ObjectsCreated_1 → HandleObjectsCreated()
                                PortStateChange_1_1 → HandlePortStateChangeAlarm()
                                etc.
```

pao-switch-core also **produces** events via pod-actor producer pipelines:
- **Alarm pipeline** — publishes alarm events (NeReachabilityStateChange, etc.)
- **Default pipeline** — publishes lifecycle events (StartupStarted, StartupFinished, etc.)
- **Internal pipeline** — publishes internal coordination events

> 📁 Event pipeline setup: `internal/bootstrap/app.go` → `setupEventPipeline()`
> 📁 Event handlers: `internal/core/assurance/`

### 2.3 Resource Model Glossary

| Concept | What it means |
|---------|---------------|
| **NE** | Network Element — a physical switch (leaf, spine, A10NSP) |
| **NEP** | Network Element Port — a port on the switch |
| **NEL** | Network Element Link — a logical link between ports |
| **NEG** | Network Element Group — a group/cluster of switches |
| **NSP** | Network Service Profile (A10NSP) |
| **ZTP Ident** | Zero Touch Provisioning identifier = the switch's serial number |
| **Lifecycle State** | Where a switch is in its lifecycle: `PLANNING` → `INSTALLING` → `OPERATING` → `RETIRING` → `UNINSTALLING` |
| **Operational Data** | Runtime data stored on the NE in inventory (DHCP options, speed, artefact versions, etc.) |
| **PatchOperationalData** | An **asynchronous event** published to the message bus that tells inventory to update a field in a resource's operational data |
| **Artefact Versions** | Software versions installed on the switch (firmware, config template, application stack) |
| **DACS** | Deployable Artefacts Compatibility Set — defines which artefact versions are compatible |
| **Bringup** | First-time configuration of a switch (fresh install) |
| **StartupFinished** | Event indicating the switch completed its boot-up sequence and is ready |
| **Reconciliation** | Restoring subscriber sessions (retail, L2BSA) after a reboot |

---

## 3. Step-by-Step Flow

### Phase A — Switch Goes Down

When the leaf switch reboots, the **adapter** (hardware controller) detects the
connection loss and publishes a protobuf message on Kafka.

#### Step 1: `NeLostIndication(CONNECTION_STATE_DOWN)`

| | |
|---|---|
| **Kafka Topic** | `NeLostIndicationTopic` (= `"ne-lost-indication"`) |
| **Consumer Group** | `pao-switch-core-proto` |
| **Handler** | `neReachabilityAlarm.On()` |
| **File** | `internal/core/assurance/nereachabilityalarm.go` |

**How does this message reach pao-switch-core?**

```
Physical Switch loses connection
        │
        ▼
Adapter (rbfs-adapter) detects TCP/gRPC connection drop
        │
        ▼
Adapter publishes a protobuf NeLostIndication message to Kafka
  topic: "ne-lost-indication"
        │
        ▼
pao-switch-core's Proto Bus consumer (consumer group: "pao-switch-core-proto")
  picks up the message from Kafka
        │
        ▼
Proto Bus routes it by topic name → neReachabilityAlarm.On()
        │
        ▼
Handler deserializes protobuf bytes → NeLostIndication struct
  using protojson.Unmarshal()
```

**What happens in the handler:**

1. The handler parses the protobuf message containing:
   - `SerialNumber` (ZTP ident of the switch)
   - `ConnectionState` = `CONNECTION_STATE_DOWN`
2. It resolves the NE by calling `FetchNetworkElementByZtpIdent(serial)`
   — this is an **HTTP GET** to the EMS/pod-inventory service.
3. Validates the NE is in `OPERATING` lifecycle state and has a hostname.
4. **Raises a `NeReachabilityStateChange` alarm** with `CRITICAL` severity
   — published via the **alarm producer pipeline** to the message bus.
   - Skips if a CRITICAL alarm already exists for this NE (BEM dedup check).

> 💡 At this point the system knows the switch is offline and has raised
> an alarm so operators can see it.

---

### Phase B — Switch Comes Back Up

The leaf finishes its hardware boot sequence and reconnects to the adapter.

#### Step 2: `NeLostIndication(CONNECTION_STATE_UP)`

| | |
|---|---|
| **Kafka Topic** | `NeLostIndicationTopic` |
| **Handler** | `neReachabilityAlarm.On()` |
| **File** | `internal/core/assurance/nereachabilityalarm.go` |

**What happens:**

1. Parses the indication — `ConnectionState` = `CONNECTION_STATE_UP`.
2. **Clears the reachability alarm** by emitting with `INFO` severity.
3. Determines the outage reason:
   - If a `StartupFinished` event was emitted in the last 5 minutes → reason =
     `"NE was not reachable due to reboot/restart"`
   - Otherwise → reason = `"NE was not reachable due to network connectivity issue"`

#### Step 3: `SystemUpIndication`

| | |
|---|---|
| **Kafka Topic** | `SystemUpIndicationTopic` |
| **Handler** | `systemUpIndicator.On()` |
| **File** | `internal/core/assurance/systemupindicator.go` |

**What happens:**

1. Parses the protobuf `SystemUpIndication` message (contains serial number).
2. Resolves NE by ZTP ident. Checks `inventory.IsRelevant(category)` — only
   processes `LeafSwitch`, `A10NspSwitch`, `SpineSwitch`.
3. **Produces a `NeStartupConfigLoaded` ephemeral event:**
   ```
   Schema: schemas.NeStartupConfigLoaded_1
   Payload: { NeSerial: <serial> }
   SourceObjectID: <NE ID>
   ```
4. This event is picked up by the artefact update consumer.

#### Step 4: `HandleStartUpConfigLoadedUpdateEvent`

| | |
|---|---|
| **Schema** | `schemas.NeStartupConfigLoaded_1` |
| **Handler** | `ArtefactUpdateConsumer.HandleStartUpConfigLoadedUpdateEvent()` |
| **File** | `internal/core/artefactupdate/consumers/nestartupconfigloaded_update.go` |

**What happens:**

1. Type-asserts the message to `*installation.NeStartupConfigLoaded`.
2. Validates the payload.
3. Currently a no-op pass-through — the `StartupFinished` event will be
   emitted later based on `RunningConfigStatusInd`.

> 💡 The switch is now online but not yet configured.

---

### Phase C — IP Assignment & Startup

#### Step 5: `IpAddressAssignmentFinished`

| | |
|---|---|
| **Schema** | `schemas.IpAddressAssignmentFinished_1` |
| **Handler** | `SwitchCoreEvents.HandleIpAddressAssignmentFinished()` |
| **File** | `internal/core/assurance/ipaddressAssignmentFinishedConsumer.go` |

**What happens:**

1. Validates metadata:
   - `SourceObjectIdentifier` must be present.
   - `SourceObjectClass` must be `"NetworkElement"`.
   - `SourceObjectType` must NOT be in the ignored set (`A4-OLT-T2`, `A4-OLT-v1`).
2. Fetches NE details. Checks `inventory.IsRelevant(category)`.
3. **Emits `StartupStarted` event:**
   ```go
   switchCore.EmitStartupStarted(ctx, neID, "notification about NE startup triggered")
   ```
   - This calls `EphProducer.Produce(schemas.StartupStarted_1, …)`.

> 💡 The system now knows the switch is starting its configuration sequence.

---

### Phase D — Configuration & Operational Data Patching

As the switch boots, EMS (the inventory system) detects changes in the switch's
inventory data and publishes `ObjectChanged` events. These are the **main
workhorse events** during reboot.

#### Step 6: `ObjectChanged` events

| | |
|---|---|
| **Schema** | `schemas.ObjectChanged_1` |
| **Handler** | `SwitchCoreEvents.HandleObjectChanged()` |
| **File** | `internal/core/assurance/objectchanged.go` |

**What happens (simplified):**

1. **Parse** — Extract `sourceObjectID`, `sourceObjectClass`, and the list of
   `AttributesChangeInfo` (what changed).

2. **Detect changed operational data** — `getChangedOpDataCharacteristics()`
   compares old vs new operational data JSON to find which fields changed.

3. **Blacklist / Whitelist filter** — Decide if this change is worth processing:

   ```
   File: internal/core/assurance/blacklist.go
   ```

   **Blacklisted attributes** (always ignored — do NOT trigger a workflow):
   | Set | Attributes |
   |-----|-----------|
   | Base (all types) | `specificationVersion`, `operationalState`, `creationTime`, `lastUpdateTime`, `operationalData`, `objectVersion` |
   | OpData (all types) | `lastUpdateTime`, `creationTime`, `objectVersion`, `operationalStateA4Local` |
   | NE-specific opData | `dhcpOptionsMaps`, `dhcpOptionsMapsV2`, `artefactVersions`, `creationTime`, `resourceVersion` |
   | NEP-specific opData | `lagId` |
   | NEG-specific opData | `artefactVersions`, `assignedDacsBinariesSyncState` |

   **Logic:** If ALL changed attributes are blacklisted → **skip** (return).
   If ANY attribute is NOT blacklisted → **proceed**.

   > ⚠️ **Important for reboot:** `managementFragments` is NOT blacklisted, so
   > when management fragments change after reboot, the ObjectChanged event
   > passes through and triggers the config update workflow.

4. **Evaluate** — `evaluateObjectChange()` produces a plan:
   - `ProceedBringup` = should we do a fresh bringup?
   - `ProceedUpdate` = should we send config updates?
   - `Effects` = side effects like decommission

5. **Apply effects** — e.g., `Decommission` if lifecycle moved to PLANNING.

6. **Source-class-specific patching:**

   | Source Class | Handler | What it does |
   |-------------|---------|-------------|
   | **NEG** | `handleNEGPatch()` | Patches IP2 loopback addresses for SPINE switches + `PatchVersionsByDacs` |
   | **NEP** | `handleNEPNdo()` | Deactivates NDO config if `offNetworkLinkIf` changed |
   | **NE** | `handleNEDacs()` | `PatchVersionsByDacs` for the NE |

7. **Find impacted NE resources** — resolves which NEs are affected by this change.

8. **Bringup path** (if `ProceedBringup`):
   - `bringupWorkflow()` → validates lifecycle, ZTP, MAT conditions.
   - If conditions met: `triggerBringup()` → `setBringup()` → sends
     `SetBringUpRequest` to adapter via gRPC.

9. **Update path** (if `ProceedUpdate`):
   - `sendConfigRequests()` → builds incremental `SetConfigRequest` /
     `DeleteConfigRequest` payloads.
   - `triggerConfigGeneration()` for each impacted NE:

     ```
     File: internal/core/assurance/objectcrud_utils.go
     ```

     ```
     triggerConfigGeneration()
       ├── generateStartupConfig()     → sends GenerateStartupConfigRequest to adapter
       ├── applyConfig()               → sends ApplyConfigRequest to adapter
       └── SetActiveConfigTemplateVersions()  → patches artefact versions in opdata
     ```

#### Step 7: `PatchVersionsByDacs` (Artefact Version Patching)

```
File: internal/core/assurance/objectcrud_utils.go
```

| Source Class | Behavior |
|-------------|----------|
| **NEG** | Gets DACS ID, fetches all switches in PLANNING state, patches target artefact versions for each |
| **NE** | Fetches NE details, validates category + matNumber, calls `processSwitch()` which calls `PrepareTargetArtefactVersion()` and emits `PatchOperationalData` if versions differ |

After patching versions, also patches **DHCP options** (V1 + V2):

```
File: internal/core/assurance/dhcpoptions.go
```

- **`PatchDhcpOptionsV1`**: Patches DHCP options 114 (boot URL) and 210 (config URL).
- **`PatchDhcpOptionsV2`**: Patches MGMT_OUT_OF_BAND and IPMI option maps.

Both compare current values against desired values and only patch if different
(`IsDhcpPatchRequired`).

#### Step 8: Port State Change (UP/DOWN)

As ports come online during reboot:

| | |
|---|---|
| **Schema** | `schemas.PortStateChange_1_1` |
| **Handler** | `SwitchCoreEvents.HandlePortStateChangeAlarm()` |
| **File** | `internal/core/assurance/portstatechangealarm.go` |

**What happens:**

1. Fetches NEP and parent NE.
2. For `IfOperationalStateType_DOWN` or `_UP`: emits a port state change
   notification to the message bus.

#### Step 9: `buildRdInterface` & Speed Patching

During config generation, when building RD (Remote Device) interface configs:

```
File: internal/core/systemconfig/commonutils/utils_external.go
Function: buildRdInterface()
```

1. Calculates the desired link speed from the NEL's LSZ characteristic:
   - `99P` → `1G`, `99Y` → `10G`
2. Gets the NSP's current operational data.
3. **Checks if speed needs patching:**
   ```go
   if calculateSpeed(nspOperData) != linkSpeed {
       patchSpeed(nsp.ID, linkSpeed, patchOperData)
   }
   ```
   - `calculateSpeed()` maps `"10G_ETHERNET"` → `"10G"`, `"1G_ETHERNET"` → `"1G"`.
   - Only patches if the current stored speed **differs** from the desired speed.
   - This prevents duplicate `PatchOperationalDataSuccess` events
     (see [Section 6](#6-the-patchoperationaldataworking-bug)).

---

### Phase E — Config Applied & Startup Finished

After the adapter applies the configuration to the switch hardware, it reports
back.

#### Step 10: `RunningConfigStatusIndication`

| | |
|---|---|
| **Kafka Topic** | `RunningConfigStatusIndTopic` |
| **Handler** | `runningConfigStatusIndicator.On()` |
| **File** | `internal/core/assurance/runningconfigind.go` |

**What happens:**

1. Parses the protobuf `RunningConfigStatusInd`:
   - `SerialNumber`, `Code` (gRPC status code), `Reason`, `Description`
2. Looks up the NE resource by ZTP ident.
3. **Dispatches based on `Reason`:**

   | Reason | Follow-up |
   |--------|-----------|
   | `REASON_CONFIG_APPLIED_AFTER_REBOOT` | `emitStartupIndication(…, "device ready for service")` |
   | `REASON_CONFIG_APPLIED_AFTER_UPGRADE` | `emitStartupIndication(…, "ArtefactUpgradeFinished")` |

4. **`emitStartupIndication()`:**
   - Extracts NE details (ID, hostname, NETypeName, category).
   - **If code = OK:**
     ```go
     switchCore.EmitStartupFinished(ctx, neID, atType, hostName, neTypeName,
         "device ready for service", "")
     ```
     Produces `schemas.StartupFinished_1` ephemeral event.
   - **If code ≠ OK:**
     ```go
     switchCore.EmitNeStartupFailedEvents(ne, lifecycleState, failureDescription, "")
     ```
     Produces `schemas.StartupFailed_1` + raises `NeStartupFailedAlarm` (CRITICAL).

5. **For SPINE_SWITCH only** — also `handleSpineSwitch()`:
   - Fetches the LSR IP address.
   - Emits `RunningConfigPushed` event (used by downstream systems).

> 💡 **This is the pivotal moment** — the `StartupFinished` event signals the
> switch has been configured and is ready for service.

---

### Phase F — Post-Startup (Subscriber Restoration)

#### Step 11: `HandleStartupFinishedEvent`

| | |
|---|---|
| **Schema** | `schemas.StartupFinished_1` |
| **Handler** | `ArtefactUpdateConsumer.HandleStartupFinishedEvent()` |
| **File** | `internal/core/artefactupdate/consumers/startupfinishedconsumer.go` |

**What happens:**

1. Resolves NE from the `StartupFinished` message.
2. Checks `inventory.IsRelevant(category)`.
3. Runs **three concurrent tasks**:

   | Task | What it does |
   |------|-------------|
   | **clearStartupFailedAlarm** | Emits `NeStartupFailedAlarm` with `INFO` severity (clears the alarm). Skips if already cleared. |
   | **restoreSubscribers** | Depends on NE category (see below). |
   | **updateStartupTime** | (A10 only) Updates `StartUpTime` in topology via gRPC. |

4. **Subscriber restoration** (by NE category):

   | Category | What happens |
   |----------|-------------|
   | **LeafSwitch** | Waits `RBFS_CONFIG_DELAY` (default **60 seconds**, configurable via env var), then **concurrently**: `ReconcileRetail()` + `ReconcileL2BSA()` |
   | **A10NspSwitch** | Waits delay, then `ReconcileL2BSA()` only |
   | Other | Skipped |

   The delay allows the switch to fully apply its RBFS (Routing and Bridging
   Forwarding Stack) configuration before subscriber sessions are restored.

> ✅ **After this step, the leaf switch is fully operational** — alarms are
> cleared, configuration is applied, and subscriber sessions are restored.

---

## 4. Kafka Topics & Event Subscriptions

### Proto Bus (Protobuf over Kafka)

| Topic Constant | Kafka Topic | Handler | Registration |
|---------------|-------------|---------|-------------|
| `NeLostIndicationTopic` | `ne-lost-indication` | `neReachabilityAlarm.On()` | `setupProtoBusSubscriptions()` |
| `SystemUpIndicationTopic` | `system-up-indication` | `systemUpIndicator.On()` | `setupProtoBusSubscriptions()` |
| `UpgradeStatusIndTopic` | `upgrade-status-ind` | `ArtefactUpdateConsumer.On()` | `setupProtoBusSubscriptions()` |
| `RunningConfigStatusIndTopic` | `running-config-status-ind` | `runningConfigStatusIndicator.On()` | `setupProtoBusSubscriptions()` |

### Event Pipeline (NATS/Kafka via pod-actor)

| Schema | Handler Method |
|--------|---------------|
| `PortStateChange_1_1` | `HandlePortStateChangeAlarm` |
| `ObjectsCreated_1` | `HandleObjectsCreated` |
| `ObjectChanged_1` | `HandleObjectChanged` |
| `ObjectDeleted_1` | `HandleObjectDeleted` |
| `PoolChangedEvent_1` | `HandlePoolChanged` |
| `FragmentChangedEvent_2` | `HandleFragmentedEvent` |
| `IpAddressAssignmentFinished_1` | `HandleIpAddressAssignmentFinished` |
| `HibernateNEResponse_1` | `HandleHibernateLiResponse` |
| `TriggerArtefactUpdate_4` | `HandleTriggerArtefactUpdateEvent` |
| `StartupFinished_1` | `HandleStartupFinishedEvent` |
| `NeStartupConfigLoaded_1` | `HandleStartUpConfigLoadedUpdateEvent` |

---

## 5. Important Files & Functions Reference

### Entry Points (Event Handlers)

| File | Function | Triggered By |
|------|----------|-------------|
| `internal/core/assurance/nereachabilityalarm.go` | `On()` | `NeLostIndication` proto message |
| `internal/core/assurance/systemupindicator.go` | `On()` | `SystemUpIndication` proto message |
| `internal/core/assurance/runningconfigind.go` | `On()` | `RunningConfigStatusInd` proto message |
| `internal/core/assurance/events.go` | `HandleObjectChanged()` | `ObjectChanged_1` event |
| `internal/core/assurance/events.go` | `HandleObjectsCreated()` | `ObjectsCreated_1` event |
| `internal/core/assurance/events.go` | `HandleIpAddressAssignmentFinished()` | `IpAddressAssignmentFinished_1` event |
| `internal/core/assurance/events.go` | `HandlePortStateChangeAlarm()` | `PortStateChange_1_1` event |
| `internal/core/artefactupdate/consumers/startupfinishedconsumer.go` | `HandleStartupFinishedEvent()` | `StartupFinished_1` event |
| `internal/core/artefactupdate/consumers/nestartupconfigloaded_update.go` | `HandleStartUpConfigLoadedUpdateEvent()` | `NeStartupConfigLoaded_1` event |
| `internal/core/artefactupdate/consumers/upgradestatusconsumer.go` | `On()` | `UpgradeStatusInd` proto message |

### Core Domain Logic

| File | Key Functions |
|------|--------------|
| `internal/core/assurance/objectchanged.go` | `processObjectChanged()`, `evaluateObjectChange()`, `bringupWorkflow()`, `sendConfigRequests()` |
| `internal/core/assurance/objectcreated.go` | `handleObjectCreatedEvent()` |
| `internal/core/assurance/objectcrud_utils.go` | `triggerConfigGeneration()`, `PatchVersionsByDacs()`, `setBringup()`, `applyConfig()`, `generateStartupConfig()` |
| `internal/core/assurance/dhcpoptions.go` | `PatchDhcpOptionsV1()`, `PatchDhcpOptionsV2()` |
| `internal/core/assurance/blacklist.go` | `ContainsBlacklistedCharacteristics()`, `IsUpdateWorkflowSkipped()` |
| `internal/core/assurance/whitelist.go` | `ContainsWhitelistedCharacteristics()` |
| `internal/core/assurance/portstatechangealarm.go` | `handlePortStateChangeAlarm()` |
| `internal/core/assurance/ipaddressAssignmentFinishedConsumer.go` | `handleIpAddressAssignmentFinished()` |

### Emit Functions (Core)

| File | Function | Produces |
|------|----------|---------|
| `internal/core/core.go` | `EmitStartupStarted()` | `schemas.StartupStarted_1` |
| `internal/core/core.go` | `EmitStartupFinished()` | `schemas.StartupFinished_1` |
| `internal/core/core.go` | `EmitStartupFailed()` | `schemas.StartupFailed_1` |
| `internal/core/core.go` | `EmitNeStartupFailedEvents()` | `StartupFailed_1` + `NeStartupFailedAlarm` (CRITICAL) |
| `internal/core/core.go` | `EmitNeStartUpFailedAlarm()` | `NeStartupFailedAlarm_1` |

### Config Generation & Patching

| File | Key Functions |
|------|--------------|
| `internal/core/systemconfig/commonutils/utils_external.go` | `buildRdInterface()`, `patchSpeed()`, `calculateSpeed()`, `calculateDataRate()`, `BuildConfigWithInterfaceLabel()` |
| `internal/core/artefactupdate/versions.go` | `PrepareTargetArtefactVersion()`, `PrepareActiveArtefactVersions()`, `SetActiveConfigTemplateVersions()` |
| `pkg/clients/patchoperationaldata/patchoperationaldata.go` | `PatchOperationalData()`, `UpdateDhcpOption()`, `UpdateArtefactVersions()`, `UpdateNepLagId()`, `DeactivateNdoConfigForNeAndNep()`, `UpdateIp2LoopbackAddress()` |

### Bootstrap / Wiring

| File | What it does |
|------|-------------|
| `internal/bootstrap/app.go` | Wires all Kafka subscriptions, event pipeline handlers, creates the switch core |
| `internal/core/startup.go` | Runs startup tasks (SysInfo, config generation, bringup, IP pools) |

---

## 6. The PatchOperationalData(working) Bug

### The Problem

After a leaf reboot, `PatchOperationalDataSuccess(working)` events from EMS
were being raised **multiple times** instead of once.

### Root Cause

In `buildRdInterface()` (`utils_external.go`), the old code was:

```go
// OLD (buggy):
if calculateSpeed(nspOperData) == "" {
    patchSpeed(nsp.ID, linkSpeed, p)
}
```

**Two issues:**

1. **Race condition:** Multiple `ObjectChanged` events arrive in quick
   succession after reboot (e.g., `managementFragments` changes). Each event
   sees `calculateSpeed == ""` (the first patch hasn't been reflected in
   inventory yet) and fires a `PatchOperationalData` call. EMS emits a
   `PatchOperationalDataSuccess(working)` for each one.

2. **SpeedUnknown loop:** `calculateSpeed()` returns `""` for both "not set"
   **and** `SpeedUnknown = "unknown"`. So if speed was patched to `"unknown"`,
   the next event would still see `== ""` and re-patch indefinitely.

### The Fix

```go
// NEW (fixed):
if calculateSpeed(nspOperData) != linkSpeed {
    patchSpeed(nsp.ID, linkSpeed, p)
}
```

Now `patchSpeed` is only called when the current speed **differs from the
desired speed**. If the speed is already correct (e.g., `"10G_ETHERNET"` stored
and `"10G"` desired), the patch is skipped — preventing duplicate events.

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **Adapter (rbfs-adapter)** | A separate microservice that talks directly to the physical switch hardware. pao-switch-core sends gRPC commands to it ("apply this config") and receives async Kafka notifications from it ("switch rebooted"). See [Section 2.1](#21-the-three-main-systems). |
| **EMS / Inventory (pod-inventory)** | The central database of all network resources. pao-switch-core reads from it via HTTP REST and writes to it via async PatchOperationalData events. See [Section 2.1](#21-the-three-main-systems). |
| **RBFS** | Routing and Bridging Forwarding Stack — the switch's operating system (made by RtBrick) |
| **Proto Bus** | Kafka message bus used for protobuf messages from the adapter. Consumer group: `pao-switch-core-proto`. See [Section 2.2](#22-communication-channels). |
| **Event Pipeline (pod-actor)** | A framework that subscribes to Kafka/NATS topics and dispatches events to handler functions by schema type. Used for inventory events and internal coordination. See [Section 2.2](#22-communication-channels). |
| **PatchOperationalData** | An asynchronous event published to the message bus. Tells EMS/inventory to update a specific field on a resource. NOT a direct HTTP call — it's fire-and-forget. |
| **Ephemeral Event** | A fire-and-forget event produced via `EphProducer.Produce()` — e.g., StartupStarted, StartupFinished |
| **gRPC** | Google's RPC framework — used for synchronous request/response calls between pao-switch-core and the adapter |
| **DACS** | Deployable Artefacts Compatibility Set — defines which software versions are compatible with each other |
| **Bringup** | First-time provisioning of a switch (sends `SetBringUpRequest` to adapter via gRPC) |
| **Reconciliation** | Re-establishing subscriber sessions (retail + L2BSA) after a reboot, via the reconciliation client |
| **L2BSA** | Layer 2 Bitstream Access — a type of wholesale broadband product |
| **NDO** | Network Device Operations — configuration applied to NE/NEP |
| **LSZ** | Leitungsschlüsselzahl — a German telecom line type identifier (e.g., `99P` = 1G, `99Y` = 10G) |
| **MAT Number** | Material number — identifies the hardware model of the switch |
| **BEM** | Business Event Manager — handles alarm deduplication (prevents raising the same alarm twice) |
| **pao-sha** | Shared library used by pao-switch-core for Kafka/NATS connectivity |
| **pod-actor** | Pipeline framework library for event-driven microservices (input → handler → output) |


┌─────────────────────────────────────────────────────────────────────┐
│                    NE REBOOT FUNCTION CALL FLOW                     │
│                  (leaf-switch21 / WLG1D4VS0000CP5)                  │
└─────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════
PHASE 1: NE GOES DOWN (06:46:41Z)
═══════════════════════════════════════════════════════════════════════

  Adapter detects NE down
       │
       ▼
  Kafka: ne-lost-indication-topic
       │ (ConnectionState_CONNECTION_STATE_DOWN)
       ▼
  ┌─ internal/core/assurance/nereachabilityalarm.go ─────────────┐
  │  neReachabilityAlarm.On(ctx, eventData)                      │
  │    ├─ parseNeLostIndication(eventData)                       │
  │    ├─ switchCore.GetInventoryClient()                        │
  │    │    .FetchNetworkElementByZtpIdent(ctx, "WLG1D4VS0000CP5")│
  │    ├─ validateNeForAlarm(ne)                                 │
  │    └─ case CONNECTION_STATE_DOWN:                            │
  │         └─ c.raiseAlarm(neID, hostname, neTypeName)          │
  │              └─ Publishes NeReachabilityStateChange alarm    │
  └──────────────────────────────────────────────────────────────┘
       │
       │  NOTE: nereachabilityalarm does NOT patch
       │  operationalStateA4Local. It only raises alarms.
       │
       ▼
  Meanwhile, pao-switch-core also sends:
  ┌─ internal/core/startup.go (or similar) ──────────────────────┐
  │  PatchOperationalData request → Kafka                        │
  │    operationalStateA4Local = "FAILED"                        │
  │    (fire-and-forget to pod-inventory)                        │
  └──────────────────────────────────────────────────────────────┘
       │
       ▼
  pod-inventory receives PatchOperationalData, applies it, publishes:
       │
       ▼
  Kafka: business-events-management
  ┌──────────────────────────────────────────────────────────────┐
  │  offset 3204: IpAddressAssignmentFinished (06:46:41Z)       │
  │  offset 3205: StartupStarted (06:46:41Z)                    │
  │  offset 3206: PatchOperationalDataSuccess FAILED (06:47:06Z)│ ← from pod-inventory
  │  offset 3207: PatchOperationalDataSuccess FAILED (06:47:06Z)│ ← DUPLICATE (~64ms later)
  └──────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════
PHASE 2: OBJECTCHANGED (06:47:06Z)
═══════════════════════════════════════════════════════════════════════

  pod-inventory publishes ObjectChanged after patching opData
       │
       ▼
  Kafka: inventory.logical.default [offset 5241]
       │ (operationalStateA4Local: WORKING → FAILED)
       │ (dhcpOptionsMaps also changed)
       ▼
  ┌─ internal/core/assurance/events.go:304 ──────────────────────┐
  │  ObjectChanged received: id=00001801-...000021               │
  │    type=A4-LEAF-Switch-v2 class=NetworkElement               │
  └──────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─ internal/core/assurance/objectchanged.go:376 ───────────────┐
  │  ObjectChanged processing: id=...021 class=NetworkElement    │
  │    │                                                         │
  │    ├─ assurance/utils.go:124                                 │
  │    │    opData changed: dhcpOptionsMaps (new vs old)         │
  │    │    opData changed: operationalStateA4Local              │
  │    │                    new=FAILED old=WORKING               │
  │    │                                                         │
  │    ├─ objectchanged.go:385                                   │
  │    │    changed attrs = [dhcpOptionsMaps,                    │
  │    │                     operationalStateA4Local]            │
  │    │                                                         │
  │    ├─ assurance/blacklist.go:134                              │
  │    │    "Blacklisted change for element type NetworkElement"  │
  │    │    (both dhcpOptionsMaps & operationalStateA4Local       │
  │    │     are blacklisted)                                    │
  │    │                                                         │
  │    └─ objectchanged.go:191                                   │
  │         "ObjectChanged skipped: no relevant attributes       │
  │          changed" → bringup=false, update=false              │
  └──────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════
PHASE 3: NE COMES BACK UP (06:47:14Z)
═══════════════════════════════════════════════════════════════════════

  Adapter detects NE is up, sends SystemUpIndication
       │
       ▼
  Kafka: systemup-indication-topic [offset 26]
       │ {"serial_number":"WLG1D4VS0000CP5"}
       ▼
  ┌─ internal/core/assurance/systemupindicator.go ───────────────┐
  │  systemUpIndicator.On(ctx, eventData)                        │
  │    │                                                         │
  │    ├─ protojson.Unmarshal → SystemUpIndication               │
  │    │    ztpIdent = "WLG1D4VS0000CP5"                         │
  │    │                                                         │
  │    ├─ switchCore.GetInventoryClient()                        │
  │    │    .FetchNetworkElementByZtpIdent(ctx, ztpIdent)        │
  │    │    → neID = "00001801-0000-4c45-4146-000000000021"      │
  │    │    → category = "LEAF_SWITCH"                           │
  │    │                                                         │
  │    ├─ inventory.IsRelevant("LEAF_SWITCH") → true             │
  │    │                                                         │
  │    └─ switchCore.GetArtefactUpdateClient().EphProducer       │
  │         .Produce(ctx, schemas.NeStartupConfigLoaded_1, ...)  │
  │         → Kafka: business-events-management [offset 3208]    │
  │                                                              │
  │    LOG: "SystemUpIndication: produced                        │
  │          NeStartupConfigLoaded(WLG1D4VS0000CP5/...021)"      │
  └──────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════
PHASE 4: NeStartupConfigLoaded CONSUMED (06:47:14Z)
═══════════════════════════════════════════════════════════════════════

  pao-switch-core consumes its own NeStartupConfigLoaded from BEM
       │
       ▼
  Kafka: business-events-management [offset 3208]
       │
       ▼
  ┌─ internal/core/artefactupdate/consumers/                     │
  │  nestartupconfigloaded_update.go ────────────────────────────┐
  │  ArtefactUpdateConsumer.HandleStartUpConfigLoadedUpdateEvent  │
  │    │                                                         │
  │    ├─ go c.consumeHandle(ctx, msg)  ← async goroutine       │
  │    │    │                                                    │
  │    │    ├─ type assert → *NeStartupConfigLoaded              │
  │    │    └─ "no action" (comment: StartupFinished will be     │
  │    │        emitted based on RunningConfigStatusInd)          │
  │    │                                                         │
  │    └─ return nil, nil                                        │
  └──────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════
PHASE 5: RunningConfigStatusInd (from adapter via Proto Bus)
═══════════════════════════════════════════════════════════════════════

  Adapter sends RunningConfigStatusInd after config push
       │
       ▼
  Kafka: running-config-status-indication-topic
       │
       ▼
  ┌─ internal/core/assurance/runningconfigind.go ────────────────┐
  │  runningConfigStatusIndicator.On(ctx, eventData)             │
  │    │                                                         │
  │    ├─ protojson.Unmarshal → RunningConfigStatusInd           │
  │    │    serial = "WLG1D4VS0000CP5"                           │
  │    │    reason = STARTUP (or similar)                        │
  │    │    code   = OK                                          │
  │    │                                                         │
  │    ├─ getResourceFromZtpIdent(ctx, serial)                   │
  │    │    → resolves NE from inventory                         │
  │    │                                                         │
  │    ├─ emitRunningConfigPushed(ctx, ind, res)                 │
  │    │    → Produces RunningConfigPushed event                 │
  │    │                                                         │
  │    └─ emitFollowUpEvents(ctx, ind, res, codeStr, event)      │
  │         │                                                    │
  │         ├─ followUpHandlers[STARTUP]                         │
  │         │                                                    │
  │         └─ emitStartupIndication(ctx, ind, res, codeOK, ...) │
  │              │                                               │
  │              ├─ inventory.ExtractNeDetails(res)               │
  │              │                                                │
  │              ├─ (code == OK) →                               │
  │              │    switchCore.EmitStartupFinished(             │
  │              │      actorCtx, neID, atType,                  │
  │              │      hostname, neTypeName,                    │
  │              │      "device ready for service", "")          │
  │              │                                               │
  │              └─ (code != OK) →                               │
  │                   switchCore.EmitNeStartupFailedEvents(...)   │
  └──────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════
PHASE 6: StartupFinished EMITTED & CONSUMED (06:47:16Z)
═══════════════════════════════════════════════════════════════════════

  ┌─ internal/core/core.go:607 ─────────────────────────────────┐
  │  SwitchCore.EmitStartupFinished(ctx, id, ...)               │
  │    │                                                         │
  │    ├─ startupFinished := management.StartupFinished{Id: id}  │
  │    └─ ephProducer.Produce(ctx,                               │
  │         schemas.StartupFinished_1, startupFinished, ...)     │
  │         → Kafka: business-events-management [offset 3209]    │
  └──────────────────────────────────────────────────────────────┘
       │
       ▼
  pao-switch-core consumes StartupFinished from BEM [offset 3209]
       │
       ▼
  ┌─ internal/core/artefactupdate/consumers/                     │
  │  startupfinishedconsumer.go ─────────────────────────────────┐
  │  ArtefactUpdateConsumer.HandleStartupFinishedEvent(ctx, msg) │
  │    │                                                         │
  │    ├─ resolveStartupFinishedNE(trCtx, msg, ev, start)        │
  │    │    ├─ type assert → *management.StartupFinished         │
  │    │    ├─ id = "00001801-0000-4c45-4146-000000000021"       │
  │    │    └─ FetchNEDetails(ctx, id) → ne                      │
  │    │                                                         │
  │    └─ collectStartupTaskResults(ctx, trCtx, ne)              │
  │         │                                                    │
  │         ├─ goroutine 1: clearStartupFailedAlarm(ne)          │
  │         │    └─ EmitNeStartUpFailedAlarm(severity=INFO)      │
  │         │                                                    │
  │         ├─ goroutine 2: restoreSubscriberResults(ctx, ne)    │
  │         │    └─ reconciliation client restore                │
  │         │                                                    │
  │         └─ (if A10NSP): updateStartupTimeResult(trCtx, neID)│
  └──────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════
PHASE 7: PatchOperationalDataSuccess WORKING (06:47:16Z, 06:48:44Z)
═══════════════════════════════════════════════════════════════════════

  pod-inventory patches operationalStateA4Local → WORKING
  (triggered by StartupFinished or reconciliation)
       │
       ▼
  Kafka: business-events-management
  ┌──────────────────────────────────────────────────────────────┐
  │  offset 3210: PatchOperationalDataSuccess WORKING (06:47:16Z)│
  │  offset 3211: PatchOperationalDataSuccess WORKING (06:48:44Z)│
  │               (88s later — separate reconciliation flow)     │
  └──────────────────────────────────────────────────────────────┘
       │
       ▼
  pao-switch-core: "not registered. Dropping!" (no handler)


═══════════════════════════════════════════════════════════════════════
PHASE 8: POST-STARTUP EVENTS
═══════════════════════════════════════════════════════════════════════

  offset 35354-35357: NepFlappingInterfaceAlarm (spine ports flapping
                      due to leaf reboot)
  offset 4949-4951:   PortStateChange_1_3 (ports going down/up)
                      → portstatechangealarm.go processes these
  
  HTTP calls:  getVLANConfigHandler, GetL2XEndpoints, GetL2Bsa
               → adapter queries for config reconciliation
