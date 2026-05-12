# GELF_ALARM_TEST_002_PORT_DOWN_ALARM_RAISED – End-to-End Flow

## Overview

This test validates the full lifecycle of a **port_down_alarm** event:  
from a simulated GELF message injected into Kafka, through the `gelf_actor` processing pipeline, all the way to a persisted alarm in PostgreSQL.

---

## Infrastructure (Docker Compose)

The test environment is spun up in `GELF_ALARM_TEST_001` (Docker Bringup) and kept running for subsequent tests. Key containers involved:

| Container | Role |
|---|---|
| `a4-kafka` | Kafka broker – receives GELF events on topic `fabric-events` |
| `a4-kafka-zookeeper` | Kafka coordination |
| `postgresql-0` | PostgreSQL – persists alarms in `persistent_events` table |
| `gelf_actor` | rtbrick-rbfs-gelf-actor – processes GELF events and raises A4 alarms |
| `a4-inventory-searcher` | Logical inventory API (`localhost:8080`) |
| `a4-notifications` | Notification/alarm store downstream |

Network: all containers share the `rtbrick_actor_test` Docker bridge network.

---

## Test Steps

### Step 1 – `Setup_Alarm_Test_Environment`

1. **`Create_Table`** – Copies `db_init.sql` into the `postgresql-0` container and executes it to initialise the `persistent_events` schema.
2. **ALTER TABLE** – Makes the `payload` column nullable so that alarm rows without a payload can be inserted without errors.

---

### Step 2 – `Setup_Port_Mapping_Data`

Seeds all inventory data the `gelf_actor` needs to resolve an alarm.

#### 2a – `Create_Table_ports`
- Copies `port_mapping.sql` and the port catalogue CSV into `postgresql-0`.
- Executes the SQL to create the `mapped_port` table.
- Imports rows from `port_mapping_catalogue_V2.9.0-27-01-26.csv` via `COPY`.
- Copies the CSV into the `apache-ignite` container and strips the unwanted columns before loading it into the Ignite cache.

#### 2b – `Update_Catalogue_Portmapping` (filter = `RTBRICK`)
- Reads `port_mapping_catalogue_V2.9.0-27-01-26.csv`.
- Keeps only rows where `cli_label_type` is empty **and** `cli_label` starts with `ifp` (RTBrick convention).
- Sends filtered rows in batches of 450 to the Catalogues GraphQL API at `http://localhost:9082/graphql` via `insertMappedPortResource` mutation.

#### 2c – `Update_NE_Info` (NE = `BLR1SPINE02`)
PUTs the Network Element (NE) to the Logical Inventory API:
```
PUT http://localhost:8080/v2/logicalResource/91000000-0080-0000-0101-120000000000
```
NE data:
| Field | Value |
|---|---|
| ID | `91000000-0080-0000-0101-120000000000` |
| Name / Hostname | `91_080_101_7KDB` |
| Type | `A4-SPINE-Switch-v2` |
| Planned Mat Number | `40990639` |
| Category | `SPINE_SWITCH` |

#### 2d – `Update_NEP_Info` (NEP for BLR1SPINE02)
PUTs the Network Element Port (NEP) from `nep_blr1spine02.json`:
```
PUT http://localhost:8080/v2/logicalResource/452f3049-747a-5207-a978-16ee912fb720
```
Key NEP data:
| Field | Value |
|---|---|
| NEP ID | `452f3049-747a-5207-a978-16ee912fb720` |
| functionalPortLabel | `100G_024` |
| CLI label (ifp_name) | `ifp-0/1/24` |
| Parent NE | `91000000-0080-0000-0101-120000000000` |

#### 2e – `Update_NEL_Info`
PUTs the Network Element Link (NEL) JSON into inventory. This is critical: the `gelf_actor` **requires** a NEL to exist for the NEP before it raises an alarm (unconfigured ports at boot are intentionally ignored).

#### 2f – Sleep 10 s
Waits for inventory caches / indexing to settle.

---

### Step 3 – `Send_Port_Down_Alarm_Event`

**Robot keyword**: `send_any_event  kafka/a4-kafka:9092  port_down_alarm`

**Python** (`EventsClient.send_any_event`):
- Launches a temporary `debian:bookworm-slim` container on the `rtbrick_actor_test` network.
- Mounts the pre-built `pao-trigger-event` binary at `/pao-trigger-event`.
- Calls: `/pao-trigger-event -messagebus=kafka/a4-kafka:9092 -eventName=port_down_alarm`

**Go binary** (`pao-trigger-event / createPortDownAlarmMessage`):
Constructs and publishes a `GelfNotification` to Kafka topic `fabric-events`:

```
GelfNotification {
  LogEvent:     "port_down_alarm"
  Host:         "91_080_101_7KDB"           // BLR1SPINE02 hostname
  SerialNumber: "WJA1B87H00016P1"
  AdditionalFields: {
    alertname:    "port_down_alarm"
    ifp_name:     "ifp-0/1/24"
    alert_status: "firing"
    device_id:    "91000000-0080-0000-0101-120000000000"
    hostname:     "91_080_101_7KDB"
  }
}
Metadata {
  SourceObjectIdentifier: "91000000-0080-0000-0101-120000000000"  // neID
  SourceObjectClass:      "NETWORK_ELEMENT"
  NotificationClass:      (set later by actor)
}
```

After publishing, the test sleeps **5 s** to allow the actor to process the message.

---

### Step 4 – `gelf_actor` Processing Pipeline (`portDownAlarm`)

The `gelf_actor` (Go service `rtbrick-rbfs-gelf-actor`) consumes messages from `fabric-events` and routes them through the pipeline built in `CreatePipeline()`.

#### 4a – Condition Match
```go
OnCondition(logEventFilter(portDownEvent), commands.portDownEventAlarm)
```
The filter checks `GelfNotification.LogEvent == "port_down_alarm"` → matches → calls `portDownAlarm()`.

#### 4b – `portDownAlarm()` Logic (step by step)

```
Incoming msg
    │
    ▼
Extract neID  ← ctx.OriginalTask().Metadata.SourceObjectIdentifier
    │             = "91000000-0080-0000-0101-120000000000"
    ▼
Extract ifp_name ← AdditionalFields["ifp_name"]
    │             = "ifp-0/1/24"
    ▼
FetchNetworkElementPort(ctx, neID, ifp_name)
    │   → returns NEP (id="452f3049-747a-5207-a978-16ee912fb720",
    │              functionalPortLabel="100G_024")
    ▼
FetchNetworkElementLink(ctx, nep.ID)
    │   → returns NEL (must NOT be nil – proves port is configured)
    ▼
FetchFabricSwitch(ctx, neID)
    │   → returns NE (typename="A4-SPINE-Switch-v2", matnum=40990639)
    ▼
GetPortMapping(NETypeName, PlannedMatNumber, functionalPortLabel)
    │   → returns MappedPort { PhysicalLabel: "24" }
    ▼
Build PortInfo_1_2 {
    FunctionalPortLabel: "100G_024"
    CliPortLabel:        "ifp-0/1/24"
    PhysicalPortLabel:   "24"
    LagId:               (from NEP extended data if present)
}
    ▼
Build PortStateChange_1_3 {
    PortInfo: (above)
    PotentiallyImpactedSubscriberCount: null (optional)
}
    ▼
isFiring? → alert_status == "firing" → YES
    ▼
Set metadata {
    NotificationClass:      ALARM
    SourceObjectIdentifier: nep.ID
    OccurredTime:           alertStartTime (from alert_starts_at field)
    PerceivedSeverity:      CRITICAL
    Description:            "port 100G_024 is down on 91_080_101_7KDB"
}
    ▼
logger.Infof("Port Down Alarm Raised: %v", neID)
    ▼
Return RawMessageWithMetadata → published to output topic "alarms"
```

> **Clear path (resolved):** If `alert_status == "resolved"`, a goroutine waits 5 seconds then produces a severity=INFO clear alarm describing the port as UP.

---

### Step 5 – Alarm Persistence

The published alarm (topic `alarms`) is consumed by the downstream notification/persistence service (`a4-notifications` / `pod-evts-storage-actor`) which writes a row to the `persistent_events` table in PostgreSQL:

| Column | Expected Value |
|---|---|
| `notificationType` | `...PortStateChange...` |
| `perceivedSeverity` | `CRITICAL` |
| `description` | contains `port 100G_024 is down` |
| `occurredTime` | within the last 300 s |

---

### Step 6 – `Verify_Port_Down_Alarm_Raised`

Calls `Verify_Port_Down_Alarm_In_PostgreSQL` with `ne_id = BLR1SPINE02.id`.

Executes the following SQL inside `postgresql-0` (retries every 10 s, up to 12 times = 120 s total):

```sql
SELECT "notificationId", "notificationType", "perceivedSeverity", description
FROM persistent_events
WHERE "notificationType" LIKE '%PortStateChange%'
  AND "perceivedSeverity" = 'CRITICAL'
  AND description LIKE '%port 100G_024 is down%'
  AND "occurredTime" > NOW() - INTERVAL '300 seconds'
ORDER BY "occurredTime" DESC
LIMIT 5;
```

**Pass condition**: At least one matching row is returned and `perceivedSeverity` contains `CRITICAL`.

---

## Complete Data Flow Diagram

```
Test Runner (Robot Framework)
        │
        │ Setup_Port_Mapping_Data
        ├──► Catalogues API (localhost:9082) ──► mapped_port (Ignite / PostgreSQL)
        ├──► Inventory API  (localhost:8080) ──► NE  (BLR1SPINE02)
        ├──► Inventory API  (localhost:8080) ──► NEP (ifp-0/1/24 → 100G_024)
        └──► Inventory API  (localhost:8080) ──► NEL (proves port is configured)

        │ Send_Port_Down_Alarm_Event
        └──► pao-trigger-event binary
                │
                ▼ Kafka topic: fabric-events
        ┌───────────────────────────┐
        │  GelfNotification         │
        │  LogEvent: port_down_alarm│
        │  neID: BLR1SPINE02        │
        │  ifp_name: ifp-0/1/24     │
        │  alert_status: firing     │
        └─────────────┬─────────────┘
                      │
                      ▼
        ┌─────────────────────────────────────┐
        │       rtbrick-rbfs-gelf-actor        │
        │                                     │
        │  1. logEventFilter("port_down_alarm")│
        │  2. FetchNetworkElementPort         │──► Inventory API
        │  3. FetchNetworkElementLink         │──► Inventory API (NEL guard)
        │  4. FetchFabricSwitch               │──► Inventory API
        │  5. GetPortMapping                  │──► Catalogues API (Ignite)
        │  6. Build PortStateChange_1_3       │
        │  7. Set severity=CRITICAL           │
        │  8. Produce alarm                   │
        └─────────────┬───────────────────────┘
                      │
                      ▼ Kafka/NATS topic: alarms
        ┌─────────────────────────┐
        │  Notification/Storage   │
        │  Service                │
        └────────────┬────────────┘
                     │
                     ▼
        ┌──────────────────────────────────┐
        │  PostgreSQL: persistent_events   │
        │  notificationType: PortStateChange│
        │  perceivedSeverity: CRITICAL     │
        │  description: port 100G_024 down │
        └──────────────────────────────────┘
                     ▲
                     │ SQL query (retry 12×10s)
        Test Runner: Verify_Port_Down_Alarm_Raised
```

---

## Key Identifiers Reference

| Entity | ID / Value |
|---|---|
| Network Element (BLR1SPINE02) | `91000000-0080-0000-0101-120000000000` |
| Network Element Port (NEP) | `452f3049-747a-5207-a978-16ee912fb720` |
| CLI Port Label (ifp_name) | `ifp-0/1/24` |
| Functional Port Label | `100G_024` |
| Physical Port Label | `24` |
| NE Type Name | `A4-SPINE-Switch-v2` |
| Planned Mat Number | `40990639` |
| Kafka topic (in) | `fabric-events` |
| Kafka topic (out) | `alarms` |
| Alarm severity | `CRITICAL` |

---

## Why the NEL Guard Exists

The `gelf_actor` intentionally checks for a Network Element Link before raising a `port_down_alarm`. During initial switch bringup, **all unconfigured ports** fire `port_down_alarm`. Without the NEL guard, the system would flood the alarm store with noise for every unplugged port. Only ports that have a NEL (i.e., are wired and provisioned) produce a real alarm.
