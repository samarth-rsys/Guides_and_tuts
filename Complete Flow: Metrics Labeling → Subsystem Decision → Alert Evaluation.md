# Complete Flow: Metrics Labeling → Subsystem Decision → Alert Evaluation

This guide shows the **complete end-to-end flow** from API request through alert rule evaluation, with focus on:
1. How metrics get labeled
2. How subsystem type is decided
3. What API is sent and received
4. How those labels are used in alert expressions

---

## Table of Contents
1. [Overview Diagram](#overview-diagram)
2. [Step 1: Device Registration via API](#step-1-device-registration-via-api)
3. [Step 2: Subsystem Decision Logic](#step-2-subsystem-decision-logic)
4. [Step 3: Label Assignment](#step-3-label-assignment)
5. [Step 4: Prometheus Scrapes and Labels Metrics](#step-4-prometheus-scrapes-and-labels-metrics)
6. [Step 5: Alert Rules Query Labeled Metrics](#step-5-alert-rules-query-labeled-metrics)
7. [Step 6: Values Used in Expressions](#step-6-values-used-in-expressions)
8. [Real-World Examples](#real-world-examples)

---

## Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. User/System sends API request to register a device                   │
│    POST /pod-observability-metrics/configuration                        │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. rest_server.py receives JSON                                          │
│    Extracts: metricsHwController, elementName, targets, etc.            │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. ne_add_delete.py:add_ne_targets() processes                          │
│    Matches metricsHwController against default_target_configs           │
│    DECISION: SWITCH_C → source_subsystem="ne"                           │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. Creates target_template with labels                                   │
│    Labels include: source_subsystem, element_name, job, etc.            │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. Stores in Kubernetes ConfigMap                                        │
│    ConfigMap: ne-target-configmap or libox-target-configmap             │
│    Response: 200/202 OK                                                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. Prometheus Operator detects ConfigMap change                          │
│    Updates Prometheus configuration automatically                       │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 7. Prometheus begins scraping metrics from target                        │
│    Adds all ConfigMap labels to every scraped metric                    │
│    Metric: optics_dom_voltage{source_subsystem="ne", element_name="..."} │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 8. Alert rules query metrics using labels from alarms-map.json           │
│    Query: optics_dom_voltage{source_subsystem="ne"} > 3.5                │
│    Only matches metrics labeled as source_subsystem="ne"                │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 9. Alert fires with details                                              │
│    Annotations include $labels.element_name, $labels.ifp_name, etc.     │
│    Response: Alert notification with all labels                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Device Registration via API

### API Endpoint
**File**: [prometheus-configuration-manager/configure-ne/rest_server.py](prometheus-configuration-manager/configure-ne/rest_server.py)

```python
@api.route("/pod-observability-metrics/configuration", methods=['POST'])
def configuring_ne():
    json_object = request.get_json()
    if not json_object:
        return get_invalid_json_data_error()
    
    res, status_code = add_ne_targets(json_object)
    return response
```

### API Request Format

**Endpoint**: `POST /pod-observability-metrics/configuration`

**Content-Type**: `application/json`

**Request Body**:
```json
{
  "metricsHwController": "SWITCH_C",
  "isCombinedHostname": false,
  "neMetricsEndpoint": {
    "elementName": "spine-switch11-49-6151-07-7kde",
    "oauthEnabled": false,
    "metricFormat": "PROMETHEUS",
    "scheme": "http",
    "scrapeInterval": "15s",
    "metricPath": "/api/v1/metrics",
    "targets": "192.168.1.11:9090"
  },
  "labels": {
    "additional_label": "value"
  }
}
```

**API Schema Reference**: [prometheus-configuration-manager/swagger/swagger.yaml](prometheus-configuration-manager/swagger/swagger.yaml)

### Request Fields Explained

| Field | Required | Example | Purpose |
|-------|----------|---------|---------|
| `metricsHwController` | ✅ Yes | `SWITCH_C` | **Determines subsystem type** |
| `isCombinedHostname` | ✅ Yes | `false` | Whether element has combined hostname |
| `neMetricsEndpoint` | ✅ Yes | `{...}` | Where to scrape metrics from |
| `elementName` | ✅ Yes | `spine-switch11-49-6151-07-7kde` | Device identifier |
| `targets` | ✅ Yes | `192.168.1.11:9090` | IP:Port to scrape |
| `metricPath` | ✅ Yes | `/api/v1/metrics` | Prometheus metrics endpoint |
| `labels` | ✅ Yes | `{...}` | Custom labels for metrics |

### Valid metricsHwController Values

From [prometheus-configuration-manager/swagger/swagger.yaml](prometheus-configuration-manager/swagger/swagger.yaml):

```yaml
metricsHwController:
  type: string
  enum:
  - SWITCH_C       # Fabric/Network Element
  - LI_C           # LTE Interface Controller
  - PON_C          # PON Controller
  - DPU_C          # Data Processor Unit
```

**Note**: Only `SWITCH_C` and `XDP_LI_BOX_ADAPTER` (LiBox variant) are configured in the system.

---

## Step 2: Subsystem Decision Logic

### Decision Point

**File**: [prometheus-configuration-manager/configure-ne/ne_add_delete.py](prometheus-configuration-manager/configure-ne/ne_add_delete.py) (lines 1-40)

```python
# Hardcoded configuration lookup table
default_target_configs = [
    {
        TARGET_METRICS_HW_CONTROLLER_TAG: NE_TARGET_METRICS_HW_CONTROLLER,      # "SWITCH_C"
        TARGET_SOURCE_SUB_SYSTEM: NE_TARGET_SOURCE_SUBSYSTEM,                   # "ne"
        TARGET_CONFIG_JOB_NAME_TAG: NE_TARGET_CONFIG_JOB_NAME,                  # "ne-federation"
        TARGET_CONFIG_CONFIGMAP_TAG: NE_TARGET_CONFIG_CONFIGMAP_NAME,           # "ne-target-configmap"
    },
    {
        TARGET_METRICS_HW_CONTROLLER_TAG: LIBOX_TARGET_METRICS_HW_CONTROLLER,   # "XDP_LI_BOX_ADAPTER"
        TARGET_SOURCE_SUB_SYSTEM: LIBOX_TARGET_SOURCE_SUBSYSTEM,                # "libox"
        TARGET_CONFIG_JOB_NAME_TAG: LIBOX_TARGET_CONFIG_JOB_NAME,               # "libox-federation"
        TARGET_CONFIG_CONFIGMAP_TAG: LIBOX_TARGET_CONFIG_CONFIGMAP_NAME,        # "libox-target-configmap"
    }
]

def add_ne_targets(targetData):
    conf = {}
    
    # MATCHING LOGIC
    for configs in default_target_configs:
        if PAYLOAD_METRIC_HW_CONTROLLER_TAG not in targetData:
            continue
        
        # THIS IS WHERE SUBSYSTEM IS DECIDED!
        # Compare incoming metricsHwController with hardcoded values
        if targetData[PAYLOAD_METRIC_HW_CONTROLLER_TAG] == configs.get(TARGET_METRICS_HW_CONTROLLER_TAG):
            conf = configs  # Found matching config!
            break
    
    if not conf:
        return "Target type is not present", 400  # Error response
    
    # Continue with target_template creation (see Step 3)
    ...
```

### Constant Definitions

**File**: [prometheus-configuration-manager/configure-ne/constants.py](prometheus-configuration-manager/configure-ne/constants.py)

```python
# Label names
TARGET_SOURCE_SUB_SYSTEM = "source_subsystem"
PAYLOAD_METRIC_HW_CONTROLLER_TAG = "metricsHwController"

# NE (Fabric Switch) Configuration
NE_TARGET_METRICS_HW_CONTROLLER = "SWITCH_C"
NE_TARGET_SOURCE_SUBSYSTEM = "ne"
NE_TARGET_CONFIG_JOB_NAME = "ne-federation"
NE_TARGET_CONFIG_CONFIGMAP_NAME = "ne-target-configmap"

# LiBox Configuration
LIBOX_TARGET_METRICS_HW_CONTROLLER = "XDP_LI_BOX_ADAPTER"
LIBOX_TARGET_SOURCE_SUBSYSTEM = "libox"
LIBOX_TARGET_CONFIG_JOB_NAME = "libox-federation"
LIBOX_TARGET_CONFIG_CONFIGMAP_NAME = "libox-target-configmap"
```

### Decision Tree

```
┌─────────────────────────────────────────────────┐
│ Incoming API Request                            │
│ metricsHwController: "SWITCH_C"                │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ Check config 1 │
        │ SWITCH_C == "SWITCH_C"?
        │ YES ✓         │
        └────────┬───────┘
                 │
                 ▼
    ┌─────────────────────────────┐
    │ Select Config 1             │
    │ source_subsystem = "ne"    │
    │ job = "ne-federation"      │
    │ configmap = "ne-target-configmap" │
    └─────────────────────────────┘

                 OR

┌─────────────────────────────────────────────────┐
│ Incoming API Request                            │
│ metricsHwController: "XDP_LI_BOX_ADAPTER"      │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ Check config 1 │
        │ XDP_LI_BOX_ADAPTER == "SWITCH_C"?
        │ NO             │
        └────────┬───────┘
                 │
                 ▼
        ┌────────────────┐
        │ Check config 2 │
        │ XDP_LI_BOX_ADAPTER == "XDP_LI_BOX_ADAPTER"?
        │ YES ✓         │
        └────────┬───────┘
                 │
                 ▼
    ┌─────────────────────────────┐
    │ Select Config 2             │
    │ source_subsystem = "libox" │
    │ job = "libox-federation"   │
    │ configmap = "libox-target-configmap" │
    └─────────────────────────────┘
```

---

## Step 3: Label Assignment

### Creating the Target Template

**File**: [prometheus-configuration-manager/configure-ne/ne_add_delete.py](prometheus-configuration-manager/configure-ne/ne_add_delete.py) (lines 40-60)

```python
def add_ne_targets(targetData):
    # ... previous matching logic ...
    
    # CREATE TARGET WITH LABELS
    target_template = {
        "targets": [targetData['neMetricsEndpoint']['targets']],
        "labels": {
            "job": conf.get(TARGET_CONFIG_JOB_NAME_TAG),           # "ne-federation"
            "element_name": targetData['neMetricsEndpoint']['elementName'],
            TARGET_SOURCE_SUB_SYSTEM: conf.get(TARGET_SOURCE_SUB_SYSTEM),  # "ne" or "libox"
            # Any additional labels from the API request
            **targetData.get('labels', {})
        }
    }
    
    # Write to appropriate ConfigMap
    configmap_name = conf.get(TARGET_CONFIG_CONFIGMAP_TAG)
    # ... Kubernetes API call to store in ConfigMap ...
    
    return "Device added successfully", 200
```

### Example Output for SWITCH_C Request

When you send:
```json
{
  "metricsHwController": "SWITCH_C",
  "neMetricsEndpoint": {
    "elementName": "spine-switch11",
    "targets": "192.168.1.11:9090",
    ...
  },
  "labels": {
    "region": "EU"
  }
}
```

The function creates:
```json
{
  "targets": ["192.168.1.11:9090"],
  "labels": {
    "job": "ne-federation",
    "element_name": "spine-switch11",
    "source_subsystem": "ne",
    "region": "EU"
  }
}
```

This gets stored in ConfigMap: `ne-target-configmap`

### Example Output for LiBox Request

When you send:
```json
{
  "metricsHwController": "XDP_LI_BOX_ADAPTER",
  "neMetricsEndpoint": {
    "elementName": "libox-01",
    "targets": "10.0.0.5:8090",
    ...
  }
}
```

The function creates:
```json
{
  "targets": ["10.0.0.5:8090"],
  "labels": {
    "job": "libox-federation",
    "element_name": "libox-01",
    "source_subsystem": "libox"
  }
}
```

This gets stored in ConfigMap: `libox-target-configmap`

---

## Step 4: Prometheus Scrapes and Labels Metrics

### How Prometheus Uses ConfigMap Labels

1. **Prometheus Operator watches ConfigMaps**
   - Detects `ne-target-configmap` and `libox-target-configmap`
   
2. **Updates Prometheus configuration**
   - Adds new scrape job with labels from ConfigMap
   
3. **Prometheus scrapes metrics**
   - Connects to target: `192.168.1.11:9090`
   - Collects raw metrics: `optics_dom_voltage 3.6`
   
4. **Applies all ConfigMap labels to each metric**
   - Adds: `source_subsystem="ne"`, `element_name="spine-switch11"`, `job="ne-federation"`

### Example: Metric Before and After

**Before labeling** (raw from exporter):
```
optics_dom_voltage 3.6
optics_dom_voltage 3.4
optics_dom_temperature_celsius 45.2
```

**After Prometheus applies labels** (stored in Prometheus):
```
optics_dom_voltage{source_subsystem="ne", element_name="spine-switch11", job="ne-federation", ifp_name="eth-0/0/0"} 3.6
optics_dom_voltage{source_subsystem="ne", element_name="spine-switch11", job="ne-federation", ifp_name="eth-0/0/1"} 3.4
optics_dom_temperature_celsius{source_subsystem="ne", element_name="spine-switch11", job="ne-federation", vendor_material_number="40907039"} 45.2
```

**Note**: Some labels like `ifp_name` and `vendor_material_number` come from the exporter/device itself, while `source_subsystem` and `element_name` come from the ConfigMap.

---

## Step 5: Alert Rules Query Labeled Metrics

### Alert Rules Use Labels to Filter

**File**: [templates/prometheus/rules-1.14/juniper-fabric-transceiver-high-voltage-alarm.rule.yaml](templates/prometheus/rules-1.14/juniper-fabric-transceiver-high-voltage-alarm.rule.yaml)

```yaml
- alert: FabricTransceiverHighVoltageAlarm40907039
  annotations:
    description: 'Fabric Transceiver {{ $labels.ifp_name }} on node {{ $labels.element_name }} detected high voltage'
    summary: 'High voltage detected'
  expr: |-
    max(avg_over_time(optics_dom_voltage{source_subsystem="ne"}[1m]) > 3.5 
        AND on (element_name, ifp_name) 
        optics_dom_temperature_celsius{vendor_material_number=~".*40907039", source_subsystem="ne"}) 
    by (element_name,ifp_name)
  for: 5m
  labels:
    severity: MAJOR
```

### How the Query Works

**PromQL Expression Breakdown**:

```promql
max(
  avg_over_time(
    optics_dom_voltage{source_subsystem="ne"}[1m]    ← STEP 1: Select metrics
  ) > 3.5                                             ← STEP 2: Filter by threshold
  
  AND on (element_name, ifp_name)                     ← STEP 3: Join with another metric
  
  optics_dom_temperature_celsius{
    vendor_material_number=~".*40907039",             ← Match specific transceiver model
    source_subsystem="ne"                             ← Only NE devices
  }
) 
by (element_name,ifp_name)                            ← STEP 4: Group results
```

### Step-by-Step Evaluation

**STEP 1**: Select only metrics labeled `source_subsystem="ne"`
```
optics_dom_voltage{source_subsystem="ne", element_name="spine-switch11", ifp_name="eth-0/0/0"} 3.6
optics_dom_voltage{source_subsystem="ne", element_name="spine-switch12", ifp_name="eth-1/0/1"} 2.8
```

**STEP 2**: Average over 1 minute and filter > 3.5V
```
3.6 > 3.5 ✓ TRUE  (spine-switch11, eth-0/0/0)
2.8 > 3.5 ✗ FALSE (spine-switch12, eth-1/0/1) - filtered out
```

**STEP 3**: Join with temperature metrics for same element/interface AND same material number
```
optics_dom_voltage{source_subsystem="ne", ...} 3.6
AND
optics_dom_temperature_celsius{vendor_material_number="40907039", source_subsystem="ne", ...} 47.2
← Only match if BOTH conditions are true
```

**STEP 4**: Return results grouped by element_name and ifp_name
```
Result: {element_name="spine-switch11", ifp_name="eth-0/0/0"} 3.6
```

### Alert Fires with These Annotations

```yaml
Description: 'Fabric Transceiver eth-0/0/0 on node spine-switch11 detected high voltage'
Summary: 'High voltage detected'
severity: MAJOR
elementName: 'spine-switch11'
ifName: 'eth-0/0/0'
threshold: "3.5"
```

---

## Step 6: Values Used in Expressions

### Where Expression Values Come From

**File**: [prometheus-configuration-manager/configure-ne/alarms-map.json](prometheus-configuration-manager/configure-ne/alarms-map.json)

The `alarms-map.json` file maps hardware models to alert expressions with placeholders:

```json
"40907039": {
  "TRANSCEIVER_VOLTAGE_MAX_WARNING_THRESHOLD": [{
    "ruleFile": "observability-rules-fabric-transceiver-high-voltage-alert",
    "ruleName": "FabricTransceiverHighVoltageAlarm40907039",
    "expr": "max(avg_over_time(optics_dom_voltage{source_subsystem=\"ne\"}[1m]) > __VALUE__ AND on (element_name, ifp_name) optics_dom_temperature_celsius{vendor_material_number=~\".*40907039\", source_subsystem=\"ne\"}) by (element_name,ifp_name)",
    "replaceValueInExpr": "__VALUE__"
  }],
  ...
}
```

### Expression Values Explained

| Value | Source | Example | Purpose |
|-------|--------|---------|---------|
| `__VALUE__` | TCA Threshold Database | `3.5` | Voltage threshold in volts |
| `optics_dom_voltage` | Metric name from exporter | `optics_dom_voltage` | Actual voltage reading |
| `[1m]` | Configuration | `1 minute` | Time window for averaging |
| `source_subsystem=\"ne\"` | API request decision | `ne` or `libox` | Device type filter |
| `vendor_material_number=~\".*40907039\"` | Device inventory | `40907039` | Specific transceiver model |
| `element_name` | API request | `spine-switch11` | Device identifier |
| `ifp_name` | Exporter/Device | `eth-0/0/0` | Interface name |

### Value Resolution Flow

```
┌────────────────────────────────────────┐
│ 1. TCA Database                        │
│    Hardware Model: 40907039            │
│    Threshold Type: VOLTAGE_MAX_WARNING │
│    Value: 3.5 VOLTS                   │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│ 2. Lookup in alarms-map.json           │
│    Find rule for 40907039              │
│    Get expr with __VALUE__ placeholder │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│ 3. Replace __VALUE__ with 3.5         │
│    expr: "... > 3.5 AND ..."           │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│ 4. Helm Template generates PrometheusRule
│    Adds this expr to alert rule        │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│ 5. Prometheus Evaluates expr           │
│    Queries: optics_dom_voltage{...}    │
│    Compares: value > 3.5               │
│    Result: Alert fires if TRUE         │
└────────────────────────────────────────┘
```

---

## Real-World Examples

### Example 1: Fabric Switch High Voltage Alert

#### 1️⃣ API Request to Register Switch

```bash
curl -X POST http://api-server:5000/pod-observability-metrics/configuration \
  -H "Content-Type: application/json" \
  -d '{
    "metricsHwController": "SWITCH_C",
    "isCombinedHostname": false,
    "neMetricsEndpoint": {
      "elementName": "spine-switch11-49-6151-07-7kde",
      "oauthEnabled": false,
      "metricFormat": "PROMETHEUS",
      "scheme": "http",
      "scrapeInterval": "15s",
      "metricPath": "/api/v1/metrics",
      "targets": "192.168.1.11:9090"
    },
    "labels": {}
  }'
```

#### 2️⃣ API Response

```json
{
  "status": "success",
  "message": "Device added successfully",
  "configMap": "ne-target-configmap"
}
```

HTTP Status: **200 OK**

#### 3️⃣ Decision in ne_add_delete.py

```python
# Decision logic
metricsHwController = "SWITCH_C"
for config in default_target_configs:
    if metricsHwController == config["SWITCH_C"]:
        source_subsystem = "ne"  # ← DECIDED HERE!
        break
```

#### 4️⃣ Labels Created

```json
{
  "targets": ["192.168.1.11:9090"],
  "labels": {
    "job": "ne-federation",
    "element_name": "spine-switch11-49-6151-07-7kde",
    "source_subsystem": "ne"
  }
}
```

Stored in: **ConfigMap `ne-target-configmap`**

#### 5️⃣ Prometheus Scrapes Metrics

```
Scraped from: http://192.168.1.11:9090/api/v1/metrics

Raw metric:
optics_dom_voltage 3.6

With labels added:
optics_dom_voltage{
  source_subsystem="ne",
  element_name="spine-switch11-49-6151-07-7kde",
  job="ne-federation",
  ifp_name="eth-0/0/0",
  vendor_material_number="40907039"
} 3.6
```

#### 6️⃣ Alert Rule Evaluates

From: [juniper-fabric-transceiver-high-voltage-alarm.rule.yaml](templates/prometheus/rules-1.14/juniper-fabric-transceiver-high-voltage-alarm.rule.yaml) line 56-65

```yaml
- alert: FabricTransceiverHighVoltageAlarm40907039
  expr: |-
    max(avg_over_time(optics_dom_voltage{source_subsystem="ne"}[1m]) > 3.5 
        AND on (element_name, ifp_name) 
        optics_dom_temperature_celsius{vendor_material_number=~".*40907039", source_subsystem="ne"}) 
    by (element_name,ifp_name)
```

**Evaluation**:
```
Step 1: Select metrics with source_subsystem="ne"
        ✓ Found: optics_dom_voltage{source_subsystem="ne", ...} 3.6

Step 2: Average over 1 minute
        Result: 3.6

Step 3: Compare 3.6 > 3.5
        ✓ TRUE

Step 4: Check temperature metric exists for same device
        ✓ Found: optics_dom_temperature_celsius{vendor_material_number="40907039", source_subsystem="ne"} 47.2

Step 5: Return grouped result
        Result: {element_name="spine-switch11-49-6151-07-7kde", ifp_name="eth-0/0/0"} 3.6
```

#### 7️⃣ Alert Fires

```yaml
Alert: FabricTransceiverHighVoltageAlarm40907039
Severity: MAJOR
Description: Fabric Transceiver eth-0/0/0 on node spine-switch11-49-6151-07-7kde detected high voltage
ElementName: spine-switch11-49-6151-07-7kde
IfName: eth-0/0/0
CurrentValue: 3.6
Threshold: 3.5
TypeOfCurrentValue: VOLTS
```

---

### Example 2: LiBox Device - Different Path

#### 1️⃣ API Request for LiBox

```bash
curl -X POST http://api-server:5000/pod-observability-metrics/configuration \
  -H "Content-Type: application/json" \
  -d '{
    "metricsHwController": "XDP_LI_BOX_ADAPTER",
    "isCombinedHostname": false,
    "neMetricsEndpoint": {
      "elementName": "libox-01",
      "targets": "10.0.0.5:8090",
      "metricPath": "/metrics",
      ...
    },
    "labels": {}
  }'
```

#### 2️⃣ Decision Different Path

```python
# Decision logic
metricsHwController = "XDP_LI_BOX_ADAPTER"
for config in default_target_configs:
    if metricsHwController == config["XDP_LI_BOX_ADAPTER"]:
        source_subsystem = "libox"  # ← DIFFERENT DECISION!
        configmap_name = "libox-target-configmap"
        break
```

#### 3️⃣ Labels Created Differently

```json
{
  "targets": ["10.0.0.5:8090"],
  "labels": {
    "job": "libox-federation",
    "element_name": "libox-01",
    "source_subsystem": "libox"
  }
}
```

Stored in: **ConfigMap `libox-target-configmap`** (Different ConfigMap!)

#### 4️⃣ Different Alert Rule Used

From: [libox-transceiver-high-voltage-alarm.rule.yaml](templates/prometheus/rules-1.14/libox-transceiver-high-voltage-alarm.rule.yaml)

```yaml
- alert: LiboxTransceiverHighVoltageAlarm40318676
  expr: |-
    avg_over_time(transceiver_module_voltage_volts{source_subsystem="libox"}[1m]) 
    * on (element_name, interface) 
    transceiver_telekom_mat_nr{source_subsystem="libox", telekom_mat_nr=~".*40318676"} 
    > 5.2
```

**Key Differences**:
- Filter: `source_subsystem="libox"` (not `"ne"`)
- Metric: `transceiver_module_voltage_volts` (not `optics_dom_voltage`)
- Threshold: `5.2` (not `3.5`)
- Different label names: `interface` instead of `ifp_name`

---

### Example 3: Invalid Device Type (Error Case)

#### Request with unsupported controller

```bash
curl -X POST http://api-server:5000/pod-observability-metrics/configuration \
  -H "Content-Type: application/json" \
  -d '{
    "metricsHwController": "PON_C",
    "neMetricsEndpoint": {
      "elementName": "pon-01",
      ...
    }
  }'
```

#### Response: Error

```json
{
  "errorCode": 400,
  "description": "Target type is not present"
}
```

HTTP Status: **400 Bad Request**

**Why**: `PON_C` is not configured in `default_target_configs` in [ne_add_delete.py](prometheus-configuration-manager/configure-ne/ne_add_delete.py). Only `SWITCH_C` and `XDP_LI_BOX_ADAPTER` are supported.

---

## Summary Table: How Values Flow Through the System

| Stage | File | Value | Purpose |
|-------|------|-------|---------|
| **1. API Request** | rest_server.py | `metricsHwController: "SWITCH_C"` | Device type indicator |
| **2. Decision** | ne_add_delete.py | Lookup in `default_target_configs` | Match controller to subsystem |
| **3. Label Assignment** | ne_add_delete.py | `source_subsystem: "ne"` | Add to target configuration |
| **4. ConfigMap Storage** | Kubernetes | Labels stored in `ne-target-configmap` | Prometheus reads these |
| **5. Metric Scraping** | Prometheus | Applies all labels to metrics | Every metric gets labeled |
| **6. Alert Query** | alarms-map.json | `{source_subsystem="ne"}` | Filter metrics by type |
| **7. Expression Evaluation** | juniper-fabric-transceiver-high-voltage-alarm.rule.yaml | `> 3.5` | Threshold comparison |
| **8. Alert Output** | Prometheus Alertmanager | `$labels.element_name`, `$labels.ifp_name` | Notification details |

---

## Key Files Reference

| File | Purpose | Critical Sections |
|------|---------|-------------------|
| [prometheus-configuration-manager/configure-ne/rest_server.py](prometheus-configuration-manager/configure-ne/rest_server.py) | REST API endpoint | Line 1-50: `configuring_ne()` function |
| [prometheus-configuration-manager/configure-ne/ne_add_delete.py](prometheus-configuration-manager/configure-ne/ne_add_delete.py) | Decision + labeling logic | Line 1-40: `default_target_configs`, Line 40-60: `add_ne_targets()` |
| [prometheus-configuration-manager/configure-ne/constants.py](prometheus-configuration-manager/configure-ne/constants.py) | Label definitions | `TARGET_SOURCE_SUB_SYSTEM`, subsystem values |
| [prometheus-configuration-manager/swagger/swagger.yaml](prometheus-configuration-manager/swagger/swagger.yaml) | API spec | `metricsHwController` enum |
| [prometheus-configuration-manager/configure-ne/alarms-map.json](prometheus-configuration-manager/configure-ne/alarms-map.json) | Expression templates | PromQL expressions with `__VALUE__` |
| [templates/prometheus/rules-1.14/juniper-fabric-transceiver-high-voltage-alarm.rule.yaml](templates/prometheus/rules-1.14/juniper-fabric-transceiver-high-voltage-alarm.rule.yaml) | Alert rules for Fabric | Expressions using `{source_subsystem="ne"}` |
| [templates/prometheus/rules-1.14/libox-transceiver-high-voltage-alarm.rule.yaml](templates/prometheus/rules-1.14/libox-transceiver-high-voltage-alarm.rule.yaml) | Alert rules for LiBox | Expressions using `{source_subsystem="libox"}` |
| [values.yaml](values.yaml) | Helm configuration | `customRules.enabled`, alert enable/disable |

