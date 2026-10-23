# RX Abnormal Attenuation Rise alarm: end-to-end walkthrough

This guide follows the code that raises `TransceiverRxAbnormalAttenuationRiseAlarm`. It is written for someone new to the project. Start with the **short version**, then follow the detailed path. "RX" means received optical signal; "dBm" is a logarithmic unit of optical power. An NE is a network element (switch), and an NEP is its network-element port.

> **Scope and important distinction:** This is the `optics_rx_dbm_raise` path through the **GELF alarms actor**, not the separate Prometheus webhook handlers for high/low RX power. The repository specifies the switch rule, HTTP GELF conversion path, actor pipeline, inventory/catalogue lookups, and emitted event. The external switch alert engine, upstream GELF forwarding, NATS delivery, and downstream alarm store/UI are not implemented here; their behavior should not be inferred from this handler alone.

## Short version

1. A switch time-series configuration defines an `optics_rx_dbm_raise` alert: a 10-second window of `optics_dom_dbm` values has a range of **at least 3 dB**, with a configured evaluation interval of `10s` and `for: "10s"`.
2. An incoming GELF notification with `log_event=optics_rx_dbm_raise` is delivered to the alarms actor. It needs an NE ID in the actor task metadata and `ifp_name` and `alert_status` in the GELF additional fields.
3. The actor looks up the port (NEP), fabric switch (NE), and physical port label from the port-mapping catalogue.
4. **Only if** `alert_status` is exactly `firing` **and** the port has an eligible inventory `type`, the actor emits a `TransceiverRxAbnormalAttenuationRiseAlarm` with the NEP as its source, threshold `3`, and a severity derived from the GELF level.
5. On `resolved` or another non-firing status it emits **nothing**. This alarm is deliberately **not cleared automatically**; the handler's comment calls for a manual clear after confirming that no unexpected device was attached.

## The route through the system

```text
Switch optics DOM data (RX and TX optical power)
  -> switch time-series metric optics_dom_dbm and alert optics_rx_dbm_raise
  -> upstream GELF delivery (outside this repository)
  -> /gelf HTTP ingestion, when using this repository's GELF endpoint
     -> normalize _-prefixed additional fields, serialize GelfNotification
     -> resolve switch serial number to NE ID, put it in task metadata
     -> publish task on the actor's request subject
  -> alarms actor consumes fabric-events GELF notifications
     -> log_event filter selects optics_rx_dbm_raise
     -> fetch port + switch from inventory, port mapping from catalogue
     -> check firing + eligible port type
     -> build A4 alarm + metadata -> NATS sink output subject
  -> downstream consumer / manual-clearing workflow (outside this handler)
```

**Do not equate the subjects:** the REST `/gelf` request uses `api.DefaultSubject`; the alarms pipeline's configured input is `fabric-events`. The exact bridge/routing between those subjects and any alternative upstream publishers is supplied by the actor/message-bus infrastructure, not shown by this handler. The `/gelf` route demonstrates how this repository accepts a GELF HTTP message; other publishers may also feed `fabric-events`.

### 1. Where the threshold comes from

The adapter's switch templates define the `optics_dom_dbm` gauge from `global.chassis_0.resource.optics.module`. It includes separate `rx_power_dbm` and `tx_power_dbm` attributes, each labeled `direction` (`rx`/`tx`), `ifp_name` (from `port_name`), and `lane_id`. See [the metric definition](../deployments/pod-rtbrick-rbfs-adapter/switchtemplates/21/include_time_series) and [the generated embedded templates](../pkg/adapters/templates/switchtemplateengine/bindata/bindata.go). Equivalent alert definitions also appear in the `11`, `61`, and `99999999` switch-template directories.

The template's alert is effectively:

```text
name:     optics_rx_dbm_raise
group:    optics
interval: 10s
for:      10s
level:    2
expr:     max(max_over_time(optics_dom_dbm[10s]) -
              min_over_time(optics_dom_dbm[10s]))
          by (element_name, bd_name, ifp_name) >= 3
```

Read the expression from inside out: for each metric series, subtract its minimum over 10 seconds from its maximum over 10 seconds, then take the maximum of those ranges grouped by element, `bd_name`, and interface. A result **equal to 3 also qualifies**. The rule's `summary` names the port and lane, `description` says "Abnormal RX attenuation raise.", and `labels` include interface, lane, and direction. See [the alert rule](../deployments/pod-rtbrick-rbfs-adapter/switchtemplates/21/include_time_series).

**Subtleties worth knowing:** the expression has no `{direction="rx"}` selector even though the metric contains RX **and** TX series, and grouping does not preserve `direction` or `lane_id`. It measures a **range** (max minus min), not a signed increase, a current measured value, or optical path attenuation calculated from two ends. The label interpolation for lane/direction in the rule does not change this expression. Therefore the rule as written does **not itself guarantee** that the qualifying change was RX-only or an upward change; check the actual alert engine's resulting notification and telemetry before making that claim. The Go handler does not recompute or verify the metric.

### 2. How GELF reaches the actor

The gelf actor's [main](../cmd/actors/gelf/main.go) starts the alarms pipeline, an HTTP service (default address `:2604`), and a **separate** Prometheus webhook service. The RX abnormal attenuation handler is on the GELF pipeline, **not** the Prometheus webhook path.

For the HTTP GELF route, [the `/gelf` handler](../pkg/resources/gelf.go) decodes JSON, normalizes fields, converts the data to `GelfNotification`, resolves `serial_number` through inventory to exactly one NE ID, and sets that ID in `env.Metadata.SourceObjectIdentifier`. If the serial cannot be resolved uniquely it returns HTTP `422`. [The GELF reader](../pkg/resources/gelf_reader.go) strips leading underscores from field names (for example `_ifp_name` becomes `ifp_name`), places non-standard GELF fields into `AdditionalFields`, and keeps `log_event`, `level`, and `serial_number` in the main notification. This explains why the alarm handler reads `ifp_name` from additional fields but the NE ID from **task metadata**, not from the alert's `element_name`.

The [alarms pipeline](../pkg/fixtures/gelf/alarms/alarms.go) subscribes using `NewDefaultInputLayer("fabric-events", "pod-rtbrick-rbfs-gelf-actor")`, builds a GELF notification pipeline, and registers `logEventFilter("optics_rx_dbm_raise")` with `transceiverRXAbnormalAttenuationRiseAlarm`. That filter checks that the message is a `GelfNotificationWrapper` and that its `LogEvent` equals the event name. It does **not** check status or severity; the handler does that later. The actor framework creates the wrapper and context; its internal bus transport is an external dependency.

### 3. Exactly what the handler does

Follow [the handler](../pkg/fixtures/gelf/alarms/transceiver_rx_abnormal_attenuation_alarm.go) in execution order:

| Step | Source | Result |
| --- | --- | --- |
| Read event and NE ID | GELF wrapper and `ctx.OriginalTask().Metadata.SourceObjectIdentifier` | Log received event/NE; expect the input to be a GELF wrapper with task metadata. |
| Read interface | `AdditionalFields["ifp_name"]` as a string | If missing/wrong type, log an error and return no event. |
| Fetch NEP | `FetchNetworkElementPort(ctx, neID, ifpName)` | Inventory looks up the NEP by CLI label and NE ID. On error, log and return no event. |
| Fetch NE | `FetchFabricSwitch(ctx, neID)` | Inventory accepts fabric categories A10NSP, Leaf, or Spine. On error/non-fabric NE, log and return no event. |
| Map port | `GetPortMapping(neTypeName, plannedMatNumber, functionalPortLabel)` | Catalogue must return exactly **one** mapping; use its `PhysicalLabel`. No/ambiguous mappings or lookup error suppress the event. |
| Set prospective metadata | NEP `ID`, notification class `ALARM` | Switches source identity from NE to NEP. |
| Decide whether to emit | `isFiring(rbfsAlarm) && monitorSignalPower(localNEP)` | Emit only for exact `alert_status == "firing"` and a listed port type; otherwise no output. |
| Build and return | A4 payload plus optional metadata | Return a single `RawMessageWithMetadata` via `GetRawMessagesMetadata`; the pipeline sends produced messages to its sink. |

The port types accepted by `monitorSignalPower` (exact, case-sensitive inventory `type` values) are:

```text
1G_ETHERNET
10G_ETHERNET_PORT
100G_ETHERNET_PORT
400G_ETHERNET_PORT
1G_10G_COMBO_ETHERNET_PORT
```

Note the order: even a `resolved` event or an ineligible port still incurs **all three** lookups before the final decision. These are inventory/catalogue queries, not writes to inventory. The handler logs lookup errors but returns `(nil, nil)`, so a caller sees no alarm message **and no propagated layer error** in these cases. The code assumes successful lookups return non-nil resources/mapping; it does not guard nil values. There is no local deduplication or prior-alarm lookup in this handler.

### 4. What is emitted

The payload type is `TransceiverRxAbnormalAttenuationRiseAlarm` from the external `pod-events` generated package. The actor fills these fields:

| Payload field | Value |
| --- | --- |
| `PortInfo.CliPortLabel` | The incoming `ifp_name` (for example `ifp-0/1/20`). |
| `PortInfo.FunctionalPortLabel` | NEP characteristic `functionalPortLabel`. |
| `PortInfo.PhysicalPortLabel` | Catalogue mapping's `PhysicalLabel`. |
| `ThresholdCrossedInfo.PhysicalResourceId` | NE ID from original task metadata (**not** the NEP ID). |
| `ThresholdCrossedInfo.Threshold` | Literal `3`. |
| `ThresholdCrossedInfo.TypeOfCurrentValue` | `DBM` enum; **no measured current value** is populated here. |
| `ThresholdCrossedInfo.ThresholdDescription` | Formatted external `notifsummary.TRANSCEIVER_RX_ABNORMAL_ATT_RISE_ALARM_RAISED_1` with `element_name` and `ifp_name`. |

Its optional **event metadata** sets notification class `ALARM`, source object identifier to the **NEP ID**, `OccurredTime` to `alert_starts_at` if present (otherwise the actor's current RFC3339 time), `PerceivedSeverity` from GELF `Level`, and `Description` to the same formatted summary. `element_name` is read from additional fields for the description; if absent, its string is empty. The exact sentence in the summary comes from the external `pod-events/notifsummary` dependency, not this repository. See [metadata helpers](../pkg/fixtures/gelf/alarms/model.go).

The GELF level-to-perceived-severity conversion is:

| GELF `Level` | A4 perceived severity |
| --- | --- |
| `< 3` | `CRITICAL` |
| `3` | `MAJOR` |
| `4` | `WARNING` |
| `5` | `MINOR` |
| `>= 6` | `INFO` |

The template sets `level: "2"`, so a notification retaining that level maps to `CRITICAL`. The handler uses **GELF level**, not a `severity` string in additional fields. `alert_starts_at` and `alert_status` must be present in the incoming notification if the upstream system expects the actor to use them; the switch rule itself does not define their transport format.

### 5. Publication and clearing

The [pipeline sink](../pkg/fixtures/gelf/alarms/alarms.go) is configured as `NatsSinkType` for all produced messages. Its output subject is `ALARM_PRODUCER_TOPIC`, falling back to `alarms` if unset. In the checked-in [gelf actor Helm values](../deployments/pod-rtbrick-rbfs-gelf-actor/values.yaml), the producer topic is `raw.alarms`, and [the deployment](../deployments/pod-rtbrick-rbfs-gelf-actor/templates/deployment.yaml) passes that value as `ALARM_PRODUCER_TOPIC`. Check deployed overrides to know the actual subject. This is an **emitted event**, not an inventory alarm record written by this handler; downstream persistence, correlation, display, and a manual-clear API are outside this implementation.

On a GELF event with `alert_status="resolved"`, or any other non-`firing` status, the handler returns no event. It does **not** emit an INFO clear, call BEM to check alarm state, or clear an alarm on recovery. The explicit [source comment](../pkg/fixtures/gelf/alarms/transceiver_rx_abnormal_attenuation_alarm.go) says RX raise alarms must not be cleared automatically so a human can confirm no unexpected device was attached. This is a stated intent of this handler; the repository does not show the operator's actual manual-clear procedure.

## Example: following one event

Imagine a GELF notification with `log_event="optics_rx_dbm_raise"`, `level=2`, and additional fields `ifp_name="ifp-0/1/20"`, `alert_status="firing"`, `element_name="leaf-1"`, and `alert_starts_at="2026-10-23T04:00:00Z"`. Its task metadata identifies NE `ne-123`. Assume inventory returns NEP `port-456` with `type="10G_ETHERNET_PORT"` and `functionalPortLabel="uplink-20"`, the NE is a fabric switch, and the catalogue maps that label to physical port `20`.

The actor emits **one** `TransceiverRxAbnormalAttenuationRiseAlarm`: CLI/functional/physical labels are `ifp-0/1/20`, `uplink-20`, and `20`; physical resource ID is `ne-123`; threshold is `3` and unit enum is `DBM`. Metadata identifies source `port-456`, notification class `ALARM`, occurred time `2026-10-23T04:00:00Z`, and severity `CRITICAL`. The description uses the external summary template with `leaf-1` and `ifp-0/1/20`. If the same event instead has `alert_status="resolved"` or a port type not in the allowlist, the actor returns **no** alarm payload.

## Debugging checklist and test coverage

- No handler log? Check the deployed actor, input routing to `fabric-events`, wrapper type, and exact `log_event` value. A Prometheus webhook alert is a different route.
- `cannot fetch interface name`? Inspect normalized `ifp_name` and its string type. A leading underscore in HTTP GELF input is stripped by the reader.
- `Failed to fetch NEP`, `cannot fetch network element`, or `cannot fetch port mapping`? Verify serial-to-NE resolution/NE ID, NEP CLI label, fabric category, `neTypeName`, `plannedMatNumber`, and NEP `functionalPortLabel`; catalogue mapping must be unique.
- Lookups succeed but nothing is emitted? Verify exact `alert_status="firing"` and one of the five accepted port types. `element_name` affects the text, not the firing decision. The `Monitoring port type` log is debug-level.
- Handler logs `...Alarm Raised` but the alarm is not visible? That log proves the handler constructed an event, **not** successful NATS delivery or downstream storage; check the sink subject and external consumers.
- Alarm stays active after recovery? This handler intentionally does not auto-clear; follow the operational manual-clear process rather than expecting a resolved GELF event to emit a clear.

[The dedicated unit test](../pkg/fixtures/gelf/alarms/transceiver_rx_abnormal_attenuation_alarm_test.go) mocks the three lookups and exercises an allowed type and a non-allowed type with `firing` status. It asserts the CLI label **only if** an output exists, so it does **not** firmly verify that allowed types produce an alarm or that disallowed types do not. [The filter test](../pkg/fixtures/gelf/alarms/alarms_test.go) checks that the event name matches. These tests do not establish end-to-end delivery, `resolved` handling, severity mapping for this specific event, or manual clearing; those conclusions above come from the code paths, not integration tests.

## Source map

- Switch metric and rule: [include_time_series](../deployments/pod-rtbrick-rbfs-adapter/switchtemplates/21/include_time_series)
- HTTP GELF conversion: [gelf.go](../pkg/resources/gelf.go), [gelf_reader.go](../pkg/resources/gelf_reader.go)
- Actor startup, routing, and sink: [main.go](../cmd/actors/gelf/main.go), [alarms.go](../pkg/fixtures/gelf/alarms/alarms.go)
- Decision, output, and helpers: [transceiver_rx_abnormal_attenuation_alarm.go](../pkg/fixtures/gelf/alarms/transceiver_rx_abnormal_attenuation_alarm.go), [model.go](../pkg/fixtures/gelf/alarms/model.go)
- Inventory and catalogue implementations: [logical.go](../pkg/fixtures/inventory/logical.go), [inventory.go](../pkg/fixtures/inventory/inventory.go)
