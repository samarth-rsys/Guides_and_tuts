# GELF_ALARM_TEST_002_PORT_DOWN_ALARM_RAISED – End-to-End Flow (Beginner's Guide)

---

## 🧭 What Is This Test Trying to Prove?

In a real telecom network, physical switches have many ports (like network sockets). When a cable is unplugged or a port fails, the switch emits a **log message** called a GELF event saying *"port X is down"*.

This test checks that the whole system correctly:
1. Receives that raw log message from the switch
2. Enriches it with inventory data (what port is it? which switch? what is its physical label?)
3. Raises a structured **alarm** with severity `CRITICAL`
4. Saves that alarm into the database

If all of this works correctly, the test **passes**.

---

## 📚 Key Concepts to Understand First

Before diving into the steps, here is a glossary of terms used throughout:

| Term | What It Means |
|---|---|
| **GELF** | Graylog Extended Log Format – a structured JSON log message format that network switches send when something happens (port down, fan failure, etc.) |
| **Kafka** | A message queue / event bus – like a post office. Services publish messages to *topics*, and other services read from those topics |
| **Topic** | A named channel inside Kafka. Like a mailbox label. This test uses `fabric-events` (incoming) and `alarms` (outgoing) |
| **gelf_actor** | A Go microservice (`rtbrick-rbfs-gelf-actor`) that reads GELF messages from Kafka, processes them, and publishes structured alarms |
| **NE** | Network Element – a physical switch (e.g. BLR1SPINE02 is a spine switch in location BLR1) |
| **NEP** | Network Element Port – a single physical port on the switch (e.g. `ifp-0/1/24` is port 24 in slot 1) |
| **NEL** | Network Element Link – a logical link proving that the port is wired up and provisioned. Without a NEL, the port is considered unconfigured |
| **Inventory API** | A REST API (`localhost:8080`) that stores the NE/NEP/NEL data – think of it as a database of "what exists in the network" |
| **Catalogues API / Ignite** | A GraphQL API (`localhost:9082`) backed by Apache Ignite (an in-memory cache) that stores **port mapping** data – the relationship between logical port names and physical labels |
| **Port Mapping** | A translation table: given a switch type + logical port label → what is the physical port label printed on the hardware? |
| **Alarm** | A structured event indicating a problem in the network. Has fields like `severity`, `description`, `notificationType` |
| **persistent_events** | A PostgreSQL table where all raised alarms are permanently stored for querying |
| **pao-trigger-event** | A Go CLI tool used in tests to simulate a switch by publishing fake GELF events to Kafka |
| **Robot Framework** | The test automation framework that orchestrates all the steps (written in `.robot` files) |

---

## 🏗️ Test Infrastructure (Docker Compose)

The entire test environment runs locally as a set of Docker containers. All containers sit on the same private Docker network called `rtbrick_actor_test`, so they can talk to each other by container name.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Docker Network: rtbrick_actor_test             │
│                                                                  │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────────┐  │
│  │  a4-kafka   │    │ postgresql-0 │    │ a4-logical-        │  │
│  │  (port 9092)│    │ (port 5432)  │    │ inventory          │  │
│  │             │    │              │    │ (port 8080)        │  │
│  └──────┬──────┘    └──────┬───────┘    └────────────────────┘  │
│         │                  │                                      │
│  ┌──────┴──────────┐  ┌────┴──────────────┐  ┌───────────────┐  │
│  │ rtbrick-rbfs-   │  │ a4-notifications  │  │ apache-ignite │  │
│  │ gelf-actor      │  │ (pod-evts-storage)│  │ + a4-inventory│  │
│  │ (consumes Kafka)│  │ (writes to DB)    │  │ -searcher     │  │
│  └─────────────────┘  └───────────────────┘  │ (port 9082)   │  │
│                                               └───────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Container Roles

| Container | Port | Role |
|---|---|---|
| `a4-kafka` | 9092 | Message broker. Receives GELF events on topic `fabric-events`. Delivers alarms on topic `alarms` |
| `a4-kafka-zookeeper` | 2181 | Required coordinator for Kafka (manages leader election and topic metadata) |
| `postgresql-0` | 5432 | Database. Stores all persisted alarms in `persistent_events` table |
| `a4-logical-inventory` | 8080 | REST API to store/retrieve NE, NEP, NEL objects |
| `apache-ignite` | 10800 | In-memory data grid (cache) for port mapping catalogue data |
| `a4-inventory-searcher` | 9082 | GraphQL API on top of Ignite for port mapping queries |
| `rtbrick-rbfs-gelf-actor` | — | The actor being tested. Reads from `fabric-events`, processes alarms, writes to `alarms` |
| `a4-notifications` | — | Downstream service. Reads from `alarms` topic and writes rows to `persistent_events` |

> **Note:** `GELF_ALARM_TEST_001` starts all of these containers using `docker-compose up` before `TEST_002` runs.

---

## 🔬 The Real-World Scenario Being Simulated

Imagine this situation:

```
Physical World:
  Switch BLR1SPINE02 (located in Berlin, rack 7KDB)
    └── Port ifp-0/1/24  ← cable gets unplugged!
          │
          │  Switch firmware detects port is down
          │  and sends a GELF log message
          ▼
       GELF event: { log_event: "port_down_alarm", ifp_name: "ifp-0/1/24", alert_status: "firing" }
```

In the test, instead of a real switch, the `pao-trigger-event` tool plays the role of the switch and publishes this fake GELF message to Kafka.

---

## 📋 Test Steps – Detailed Walkthrough

### Step 1 – `Setup_Alarm_Test_Environment`

**What happens:** The test prepares the PostgreSQL database so it can receive alarms.

**Actions:**
1. `Create_Table` – Copies `db_init.sql` from the test machine into the `postgresql-0` container and runs it. This creates the `persistent_events` table (and others) if they don't exist yet.
2. `ALTER TABLE` – Runs this SQL:
   ```sql
   ALTER TABLE persistent_events ALTER COLUMN payload DROP NOT NULL;
   ```
   This makes the `payload` column optional so that alarm rows that don't carry a raw payload can still be inserted without a database error.

**Why this matters:** If the table doesn't exist or the schema is wrong, alarms can't be saved and the test verification will always fail.

---

### Step 2 – `Setup_Port_Mapping_Data`

**What happens:** The test seeds all the reference data that `gelf_actor` needs to process the alarm.

Think of it like this: when the actor receives `"port ifp-0/1/24 is down on switch 91000000-..."`, it needs to look up:
- *What is the functional name of this port?* (`100G_024`)
- *Is this port actually wired (does it have a Network Element Link)?*
- *What is the physical label printed on the hardware?* (`24`)

Without this data, the actor cannot build a meaningful alarm. This step seeds all of it.

---

#### Sub-Step 2a – `Create_Table_ports`

Creates the `mapped_port` table in PostgreSQL and the port mapping cache in Apache Ignite.

```
Test machine
    ├── docker cp port_mapping.sql       → postgresql-0:/port_mapping.sql
    ├── docker cp port_mapping_catalogue_V2.9.0-27-01-26.csv → postgresql-0:
    └── docker cp port_mapping_catalogue_V2.9.0-27-01-26.csv → apache-ignite:

postgresql-0: runs port_mapping.sql    → creates mapped_port table
postgresql-0: COPY mapped_port FROM STDIN → loads CSV rows into table
apache-ignite: sed strips columns, then loads CSV into Ignite cache
```

---

#### Sub-Step 2b – `Update_Catalogue_Portmapping` (RTBRICK filter)

The port mapping catalogue contains entries for many switch vendors (Juniper, RTBrick, etc). This step filters and loads **only RTBrick ports** into the Catalogues GraphQL API (Ignite).

**RTBrick port identification rule:**
- `cli_label_type` column is **empty**
- `cli_label` column **starts with `ifp`** (RTBrick interface naming convention, e.g. `ifp-0/1/24`)

The filtered ports are sent in batches of 450 to:
```
POST http://localhost:9082/graphql
mutation { insertMappedPortResource(resources: [...]) }
```

**Example row loaded:**
| NeTypeName | FunctionalPortLabel | PlannedMatNumber | PhysicalLabel | CliLabel |
|---|---|---|---|---|
| `A4-SPINE-Switch-v2` | `100G_024` | `40990639` | `24` | `ifp-0/1/24` |

This row is what allows the actor to later translate `ifp-0/1/24` → physical label `24`.

---

#### Sub-Step 2c – `Update_NE_Info` (BLR1SPINE02)

Registers the **switch itself** in the Logical Inventory API. The actor uses this to know the switch type and material number (needed to look up the port mapping).

```
PUT http://localhost:8080/v2/logicalResource/91000000-0080-0000-0101-120000000000

Body:
{
  "@type": "NetworkElement",
  "id":    "91000000-0080-0000-0101-120000000000",
  "name":  "91_080_101_7KDB",
  "characteristic": [
    { "name": "category",         "value": "SPINE_SWITCH" },
    { "name": "neTypeName",       "value": "A4-SPINE-Switch-v2" },
    { "name": "plannedMatNumber", "value": "40990639" }
  ]
}
```

> **`neTypeName` + `plannedMatNumber`** together are the lookup key for port mapping. They uniquely identify the hardware model.

---

#### Sub-Step 2d – `Update_NEP_Info` (Port on BLR1SPINE02)

Registers the **specific port** (`ifp-0/1/24`) in the Logical Inventory API. This is the port that will go "down" in our test.

```
PUT http://localhost:8080/v2/logicalResource/452f3049-747a-5207-a978-16ee912fb720

Key characteristics:
  functionalPortLabel = "100G_024"   ← human-readable port name used in alarms
  cliLabel            = "ifp-0/1/24" ← CLI name used by the switch OS (RTBrick)
  type                = "100G_ETHERNET_PORT"
  parent NE           = "91000000-0080-0000-0101-120000000000"
```

> **`functionalPortLabel`** is the label that will appear in the final alarm description: *"port **100G_024** is down"*.

---

#### Sub-Step 2e – `Update_NEL_Info` (Link on the Port)

Registers a **Network Element Link** (NEL) for the port in the Logical Inventory API.

```
PUT http://localhost:8080/v2/logicalResource/91000002-0080-0000-0101-120000000002

Body:
{
  "@type": "NetworkElementLink",
  "id":    "91000002-0080-0000-0101-120000000002",
  "lifecycleState": "OPERATING",
  "resourceRelationship": [
    { "type": "requires",
      "resourceRef": { "id": "452f3049-747a-5207-a978-16ee912fb720",
                       "type": "NetworkElementPort" } }
  ]
}
```

> ⚠️ **This is the most critical piece of data.** If no NEL exists for the port, the `gelf_actor` will **silently drop the alarm** with a "not found" error. This is by design — see the [NEL Guard section](#-why-the-nel-guard-exists) below.

---

#### Sub-Step 2f – Sleep 10 s

Waits 10 seconds for the Logical Inventory service to index the newly inserted data and for the Ignite cache to sync. Without this wait, the actor might query stale or missing data.

---

### Step 3 – `Send_Port_Down_Alarm_Event`

**What happens:** A fake GELF message is published to Kafka, simulating a real switch reporting a port-down event.

#### How the event is sent (layered call chain):

```
Robot Framework keyword:
    send_any_event  kafka/a4-kafka:9092  port_down_alarm
         │
         ▼
Python (EventsClient.send_any_event):
    docker run --rm \
      --network rtbrick_actor_test \
      -v /path/to/pao-trigger-event:/pao-trigger-event:ro \
      debian:bookworm-slim \
      /pao-trigger-event -messagebus=kafka/a4-kafka:9092 -eventName=port_down_alarm
         │
         ▼
Go binary (pao-trigger-event → createPortDownAlarmMessage):
    Builds and publishes a GelfNotification to Kafka topic "fabric-events"
```

> The temporary `debian:bookworm-slim` container is used because the binary needs glibc (which Alpine Linux doesn't have). By running it on the same Docker network, it can reach `a4-kafka` by hostname.

#### The GELF Message Published to Kafka (`fabric-events` topic):

```json
GelfNotification:
{
  "LogEvent":     "port_down_alarm",
  "Host":         "91_080_101_7KDB",
  "SerialNumber": "WJA1B87H00016P1",
  "AdditionalFields": {
    "alertname":    "port_down_alarm",
    "ifp_name":     "ifp-0/1/24",       ← which port went down
    "alert_status": "firing",            ← "firing" = alarm active, "resolved" = alarm cleared
    "device_id":    "91000000-0080-0000-0101-120000000000",
    "hostname":     "91_080_101_7KDB",
    "ip_addr":      "172.27.65.186"
  }
}

Metadata (envelope):
{
  "SourceObjectIdentifier": "91000000-0080-0000-0101-120000000000",  ← neID (the switch)
  "SourceObjectClass":      "NETWORK_ELEMENT",
  "NotificationId":         "<uuid>",
  "OccurredTime":           "<timestamp>"
}
```

After publishing, the test sleeps **5 seconds** to give the actor time to read and process the message from Kafka.

---

### Step 4 – `gelf_actor` Processes the Event

This is where the core business logic happens. The `gelf_actor` is a continuously running Go service that reads messages from the `fabric-events` Kafka topic.

#### 4a – Pipeline Routing

When the actor starts, it builds a processing pipeline with many condition-handler pairs:

```go
pipeline.
    OnCondition(logEventFilter("port_down_alarm"),  commands.portDownAlarm).
    OnCondition(logEventFilter("lag_down_alarm"),   commands.lagDownAlarm).
    OnCondition(logEventFilter("cpu_high"),         commands.cpuHighUtilizationAlarm).
    // ... many more ...
```

When a message arrives with `LogEvent = "port_down_alarm"`, only the `portDownAlarm` handler is called.

#### 4b – `portDownAlarm()` Step-by-Step Logic

Here is what happens inside the Go function, in plain English:

```
┌─────────────────────────────────────────────────────────────────────┐
│  INCOMING MESSAGE                                                   │
│  neID    = "91000000-0080-0000-0101-120000000000"  (from metadata)  │
│  ifpName = "ifp-0/1/24"                            (from GELF body) │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │ 1. Extract ifp_name    │  If missing → return error immediately
              │    from GELF fields    │  (cannot process without port name)
              └────────────┬───────────┘
                           │
                           ▼
              ┌────────────────────────────────────────────────────┐
              │ 2. FetchNetworkElementPort(neID, "ifp-0/1/24")     │
              │    → calls Inventory API                           │
              │    → returns NEP object:                           │
              │       id = "452f3049-747a-5207-a978-16ee912fb720"  │
              │       functionalPortLabel = "100G_024"             │
              └────────────────────────┬───────────────────────────┘
                                       │
                                       ▼
              ┌───────────────────────────────────────────────────────┐
              │ 3. FetchNetworkElementLink(nep.ID)                    │
              │    → calls Inventory API                              │
              │    → returns NEL object  ← "port is provisioned"     │
              │                                                       │
              │    ⚠️  If NEL is nil → STOP. Return error.            │
              │       (Port is unconfigured, do not raise alarm)      │
              └───────────────────────────────────────────────────────┘
                                       │
                                       ▼
              ┌───────────────────────────────────────────────────────┐
              │ 4. FetchFabricSwitch(neID)                            │
              │    → calls Inventory API                              │
              │    → returns NE (the switch):                         │
              │       neTypeName       = "A4-SPINE-Switch-v2"         │
              │       plannedMatNumber = "40990639"                   │
              └───────────────────────────────────────────────────────┘
                                       │
                                       ▼
              ┌────────────────────────────────────────────────────────────┐
              │ 5. GetPortMapping("A4-SPINE-Switch-v2", 40990639,          │
              │                  "100G_024")                               │
              │    → calls Catalogues API / Ignite                         │
              │    → returns MappedPort:                                   │
              │       PhysicalLabel = "24"  ← the label printed on the box │
              └───────────────────────────────────────────────────────────┘
                                       │
                                       ▼
              ┌──────────────────────────────────────────┐
              │ 6. Build PortInfo                        │
              │    FunctionalPortLabel = "100G_024"      │
              │    CliPortLabel        = "ifp-0/1/24"    │
              │    PhysicalPortLabel   = "24"            │
              │    LagId              = (if bonded port) │
              └──────────────────────────────────────────┘
                                       │
                                       ▼
              ┌─────────────────────────────────────────────────────┐
              │ 7. Is alert_status == "firing"?                     │
              └───────┬─────────────────────────────────────────────┘
                      │
          ┌───────────┴────────────┐
         YES                      NO (= "resolved")
          │                        │
          ▼                        ▼
  ┌───────────────────┐   ┌──────────────────────────────────┐
  │ Raise ALARM now   │   │ Start background goroutine       │
  │                   │   │ Wait 5 seconds, then raise       │
  │ severity=CRITICAL │   │ a CLEAR alarm with severity=INFO │
  │ description=      │   │ "port 100G_024 is up on ..."     │
  │  "port 100G_024   │   └──────────────────────────────────┘
  │   is down on      │
  │   91_080_101_7KDB"│
  └────────┬──────────┘
           │
           ▼
  logger: "Port Down Alarm Raised: 91000000-..."
           │
           ▼
  Publish RawMessageWithMetadata
  to output topic "alarms"
```

> **Why the 5 second delay for clear?**  
> When a cable is briefly unplugged and re-plugged, the switch may rapidly send a "firing" followed by a "resolved". The 5 second delay prevents a flood of clear alarms for brief transient link flaps.

---

### Step 5 – Alarm Is Saved to PostgreSQL

The `a4-notifications` / `pod-evts-storage-actor` service reads from the `alarms` Kafka topic and writes every alarm into the `persistent_events` table in PostgreSQL.

The row saved for our alarm looks like this:

| Column | Value |
|---|---|
| `notificationId` | `<uuid>` |
| `notificationType` | `...PortStateChange_1_3...` |
| `perceivedSeverity` | `CRITICAL` |
| `description` | `"port 100G_024 is down on 91_080_101_7KDB"` |
| `occurredTime` | timestamp of the alarm |

---

### Step 6 – `Verify_Port_Down_Alarm_Raised`

**What happens:** The test polls the database until it finds the expected alarm row.

The test runs this SQL query every 10 seconds, up to 12 times (= 2 minutes maximum wait):

```sql
SELECT "notificationId", "notificationType", "perceivedSeverity", description
FROM persistent_events
WHERE "notificationType"  LIKE '%PortStateChange%'
  AND "perceivedSeverity" = 'CRITICAL'
  AND description         LIKE '%port 100G_024 is down%'
  AND "occurredTime"      > NOW() - INTERVAL '300 seconds'
ORDER BY "occurredTime" DESC
LIMIT 5;
```

**Test passes** ✅ if at least one row is returned and `perceivedSeverity` is `CRITICAL`.  
**Test fails** ❌ after 12 retries with: `FAIL: No port_down PortStateChange event found in persistent_events within 300s`

---

## 🗺️ Complete End-to-End Data Flow

```
╔══════════════════════════════════════════════════════════════════════╗
║  PHASE 1: SEED TEST DATA                                            ║
╚══════════════════════════════════════════════════════════════════════╝

Test Runner
  │
  ├──[2a]──► postgresql-0 : CREATE TABLE mapped_port
  │           apache-ignite: load port catalogue CSV
  │
  ├──[2b]──► POST localhost:9082/graphql
  │           insertMappedPortResource(A4-SPINE-Switch-v2, 100G_024,
  │                                    matnum=40990639, phys=24, cli=ifp-0/1/24)
  │
  ├──[2c]──► PUT localhost:8080/v2/logicalResource/91000000-...-120000000000
  │           NetworkElement BLR1SPINE02
  │
  ├──[2d]──► PUT localhost:8080/v2/logicalResource/452f3049-747a-5207-...
  │           NetworkElementPort (ifp-0/1/24, functionalPortLabel=100G_024)
  │
  └──[2e]──► PUT localhost:8080/v2/logicalResource/91000002-...-120000000002
              NetworkElementLink (links to NEP above)

╔══════════════════════════════════════════════════════════════════════╗
║  PHASE 2: INJECT EVENT                                              ║
╚══════════════════════════════════════════════════════════════════════╝

Test Runner
  └──► Python EventsClient
         └──► docker run debian:bookworm-slim /pao-trigger-event
                └──► Kafka topic: "fabric-events"
                       GelfNotification {
                         LogEvent:     port_down_alarm
                         ifp_name:     ifp-0/1/24
                         alert_status: firing
                         neID:         91000000-...-120000000000
                       }

╔══════════════════════════════════════════════════════════════════════╗
║  PHASE 3: ACTOR PROCESSING                                          ║
╚══════════════════════════════════════════════════════════════════════╝

                       Kafka topic: "fabric-events"
                              │
                              ▼
                   rtbrick-rbfs-gelf-actor
                   logEventFilter matches "port_down_alarm"
                              │
                              ├──► GET localhost:8080  → NEP (100G_024)
                              ├──► GET localhost:8080  → NEL (exists? ✓)
                              ├──► GET localhost:8080  → NE  (A4-SPINE-Switch-v2)
                              └──► GET localhost:9082  → MappedPort (phys=24)
                              │
                              │  Build PortStateChange_1_3 alarm
                              │  severity = CRITICAL
                              │  description = "port 100G_024 is down on 91_080_101_7KDB"
                              │
                              ▼
                       Kafka topic: "alarms"

╔══════════════════════════════════════════════════════════════════════╗
║  PHASE 4: PERSIST & VERIFY                                          ║
╚══════════════════════════════════════════════════════════════════════╝

                       Kafka topic: "alarms"
                              │
                              ▼
                   a4-notifications / pod-evts-storage-actor
                              │
                              ▼
                   postgresql-0: persistent_events table
                   ┌──────────────────────────────────────┐
                   │ notificationType:  PortStateChange   │
                   │ perceivedSeverity: CRITICAL           │
                   │ description: port 100G_024 is down   │
                   └──────────────────────────────────────┘
                              ▲
                              │ SQL query (retry every 10s, max 12 times)
                   Test Runner: Verify_Port_Down_Alarm_Raised  ✅
```

---

## 📌 Key Identifiers Quick Reference

| What | ID / Value |
|---|---|
| **Switch (NE)** – BLR1SPINE02 | `91000000-0080-0000-0101-120000000000` |
| **Port (NEP)** – physical port 24 | `452f3049-747a-5207-a978-16ee912fb720` |
| **Link (NEL)** – proves port is wired | `91000002-0080-0000-0101-120000000002` |
| **CLI port name** (used by RTBrick OS) | `ifp-0/1/24` |
| **Functional port label** (used in alarms/UI) | `100G_024` |
| **Physical port label** (printed on hardware) | `24` |
| **Switch hardware type** | `A4-SPINE-Switch-v2` |
| **Hardware material number** | `40990639` |
| **Kafka topic (in)** | `fabric-events` |
| **Kafka topic (out)** | `alarms` |
| **Alarm severity** | `CRITICAL` |

---

## ❓ Why Does the NEL Guard Exist?

This is one of the most important design decisions in the `portDownAlarm` handler.

**The problem:** When a network switch boots up, it fires `port_down_alarm` for **every single port** that doesn't have an active link — including ports that are intentionally unused and have never been provisioned. A 64-port switch could flood the alarm system with 60+ false alarms on every restart.

**The solution:** The actor checks for a **Network Element Link (NEL)** before raising an alarm.
- A NEL only exists for ports that have been **deliberately provisioned** and wired in the inventory system.
- If no NEL → port is unconfigured / unused → **drop silently**, do not raise alarm.
- If NEL exists → port was supposed to be UP → the down event is a real problem → **raise CRITICAL alarm**.

```
Port fires "down"
       │
       ├── Has NEL? ──NO──► Ignore (boot-time noise from unconfigured port)
       │
       └── Has NEL? ──YES──► Raise CRITICAL alarm ← This test validates this path
```

---

## 🔁 What Happens When the Port Comes Back Up?

If a second GELF event arrives with `alert_status = "resolved"` (port came back up), the actor takes a different path:

1. It does **not** raise an alarm immediately.
2. Instead it starts a **background goroutine** and waits **5 seconds**.
3. After 5 seconds, it raises a **CLEAR alarm** with severity `INFO` and description: `"port 100G_024 is up on 91_080_101_7KDB"`.

The 5-second delay prevents false "port up" alarms from brief transient disconnections (e.g., cable re-seated quickly).
