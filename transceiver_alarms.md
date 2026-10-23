# Transceiver Missing and Unsupported Alarms

This guide explains the two transceiver alarms in simple terms. They sound similar, but they are triggered in different ways.

## At a glance

| Alarm action | What starts it? | When does it happen? | Output |
| --- | --- | --- | --- |
| Raise **Transceiver Missing** | Prometheus alert sent to the webhook | Alert is `firing` and physical inventory has no equipment for the port's transceiver slot | A4 event published through the Prometheus producer pipeline. In the example logs, Kafka reports topic `business-events-management`. |
| Clear **Transceiver Missing** | GELF event `STA0031` (module added) | A module-add event arrives; this handlear builds an INFO clear event for the port | GELF alarms pipeline output subject, default `alarms`; can be changed with `ALARM_PRODUCER_TOPIC`. |
| Raise **Transceiver Unsupported** | GELF event `STA0031` (module added) | Module class is eligible, but the optic material number does not validate against the catalogue | GELF alarms pipeline output subject, default `alarms`; can be changed with `ALARM_PRODUCER_TOPIC`. |
| Clear **Transceiver Unsupported** | GELF event `STA0033` (module removed) | A previous active unsupported alarm exists for the port | GELF alarms pipeline output subject, default `alarms`; can be changed with `ALARM_PRODUCER_TOPIC`. |

The `business-events-management` destination above is confirmed by the Prometheus alarm delivery logs. The GELF pipeline code calls its destination an output subject and configures a NATS sink; check the deployment's `ALARM_PRODUCER_TOPIC` value to know whether its default is overridden.

## 1. Transceiver Missing: raise from Prometheus

A simple way to think about this alarm: **Prometheus says a transceiver should be there, and the actor checks inventory to see whether one is recorded there.**

1. Prometheus sends an alert to the webhook. The alert has `alarmType=FabricTransceiverMissingAlarm`, a port label such as `ifp-0/1/20`, and status `firing` or `resolved`.
2. The webhook finds the network element and the matching network-element port (NEP).
3. It finds the transceiver slot for that port, waits six seconds, then asks physical inventory for `Equipment` associated with the slot.
4. If the alert is `firing` and inventory returns **zero** equipment, the handler builds a `TransceiverMissingAlarm` event. It uses the alert's severity and identifies the port as the event source.
5. If inventory returns **one or more** equipment resources, no missing alarm is raised. A resolved alert also does not create a raise event in this handler.
6. The webhook controller serializes the event and publishes it through its producer pipeline.

There is one port exception: for an A10NSP network element, the missing alarm is skipped on `ifp-0/0/0` through `ifp-0/0/3`.

**Log clues:** `Received Alert for Prometheus Webhook handler` means the webhook received an alert. `Fetched 0 resources` followed by `Raising Transceiver Missing Alarm` means the raise condition was met. `delivered message ... to topic business-events-management` confirms Kafka delivery. If the handler finds equipment, it logs the resource and returns no alarm payload; the controller may then log `evtPayload is nil`. That message alone is not proof of a publish failure.

Code: [transceiver_missing_alarm.go](../pkg/controllers/prometheus-webhook/handlers/transceiver_missing_alarm.go), [webhook factory](../pkg/controllers/prometheus-webhook/prometheuswebhook/factory.go), and [webhook controller](../pkg/controllers/prometheus-webhook/prometheuswebhook/prometheusWebhook_be_controller.go).

## 2. Transceiver Missing: clear from a module-added event

The GELF alarms pipeline also registers a handler to clear Transceiver Missing when it receives `STA0031` (module added). It reads the module name, finds the NEP, and emits an INFO `TransceiverMissingAlarm` with a clear description.

This is a separate path from the Prometheus raise path. In the code shown, this clear handler does not first check whether a matching active missing alarm exists.

Code: [transceiver_missing_clear_alarm.go](../pkg/fixtures/gelf/alarms/transceiver_missing_clear_alarm.go), registered in [alarms.go](../pkg/fixtures/gelf/alarms/alarms.go).

## 3. Transceiver Unsupported: raise and clear from GELF/STA

This alarm is based on module events reported by a switch, not on the Prometheus webhook.

1. The GELF pipeline receives `STA0031` for a module addition or `STA0033` for a module removal on the `fabric-events` input.
2. It reads the module name, module class, and slot location. Only module classes `sfp` and `qsfp28` continue through this handler; other classes are ignored.
3. It asks RBFS for the optic details and checks the optic's material number against the product catalogue.
4. If validation returns `true`, the optic passed validation and this handler stops without an unsupported alarm.
5. If validation returns `false`, it proceeds as an unsupported optic. On `STA0031`, it builds a **CRITICAL** `TransceiverUnsupportedAlarm`.
6. On `STA0033`, it asks BEM whether an active unsupported alarm exists. If yes, it emits an **INFO** clear. If not, it skips the clear.

**Important caveat:** validation returns `false` for several reasons, including a missing/invalid material number or a catalogue lookup error. The caller treats any `false` result as a reason to proceed with the unsupported alarm. So a catalogue problem can look like an unsupported transceiver in this code path.

Code: [transceiver_unsupported_alarm.go](../pkg/fixtures/gelf/alarms/transceiver_unsupported_alarm.go), registered in [alarms.go](../pkg/fixtures/gelf/alarms/alarms.go). The event codes are defined in [update_physical_inventory.go](../pkg/fixtures/gelf/inventory/update_physical_inventory.go).

## Where messages go

There are separate producer configurations:

- **Prometheus webhook alarms:** the webhook producer is created with `actorAPI.DefaultSubject` in [configs.go](../pkg/controllers/prometheus-webhook/configs/configs.go). The log you shared confirms the resolved Kafka topic is `business-events-management` for both Transceiver Missing and Transceiver High Temperature.
- **GELF/STA alarms:** [alarms.go](../pkg/fixtures/gelf/alarms/alarms.go) reads `ALARM_PRODUCER_TOPIC`; if it is unset, the default output subject is `alarms`. The sink type in this pipeline is `NatsSinkType`. This is not the same configuration as the Prometheus webhook producer.

Neither handler writes the alarm itself into physical inventory. Inventory is queried to find the network element, port, slot, or equipment. The alarm is emitted as an event through the configured producer pipeline.

## Quick log-reading guide

- `alarmType:FabricTransceiverMissingAlarm` indicates the Prometheus Missing path.
- `STA0031` is a module-added event. It can trigger the GELF Unsupported raise and the Missing clear handler.
- `STA0033` is a module-removed event. It can trigger an Unsupported clear if an active alarm exists.
- `Raising ... Alarm` means the handler reached its raise branch; look for the later producer delivery log to confirm publication.
- `delivered message ... to topic ...` is the clearest evidence in these logs that the producer delivered an event to its broker topic.
