# SetConfig Function - Complete Flow Documentation

## 📌 Overview

The `SetConfig` function is a **gRPC handler** that processes configuration requests for network switches (Juniper devices). It takes device information and configuration data, validates it, processes it based on the type of configuration, and stores it in the database.

Think of it like a **smart mailroom**:
1. A package (request) arrives
2. The mailroom checks if the package is valid
3. It figures out what type of package it is
4. Routes it to the correct department for processing
5. Stores a record in the filing cabinet (database)

---

## 🔄 Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL CLIENT (gRPC Call)                          │
│                           (e.g., another microservice)                          │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  STEP 1: grpcserver.go - SetConfig Handler                                      │
│  ─────────────────────────────────────────                                      │
│  • Receives: SetConfigRequest (DeviceInfo + MessageType + InterfaceContent)     │
│  • Validates DeviceInfo (SerialNumber, MatNumber)                               │
│  • Extracts & decodes InterfaceContent from protobuf Any type                   │
│  • Calls client.SetConfig()                                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  STEP 2: handler.go - Client.SetConfig                                          │
│  ─────────────────────────────────────────                                      │
│  • Validates inputs (serialNumber, matNumber, content)                          │
│  • Acquires lock for serial number (prevents concurrent modifications)          │
│  • Fetches existing NeData from database OR creates new one                     │
│  • Routes to handleConfigByType() based on MessageType                          │
│  • Saves updated configuration to database                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                        ┌───────────────┼───────────────┐
                        ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  STEP 3: handleConfigByType - Configuration Router                              │
│  ─────────────────────────────────────────────────                              │
│  Routes to specific processor based on MessageType:                             │
│                                                                                 │
│  • MESSAGE_TYPE_SYSTEM_CONFIG     → processSystemConfig()                       │
│  • MESSAGE_TYPE_IP_CONFIG         → processIPConfig()                           │
│  • MESSAGE_TYPE_LAG_CONFIG        → processLAGConfig()                          │
│  • MESSAGE_TYPE_GRAYLOG_CONFIG    → processSyslogConfig()                       │
│  • MESSAGE_TYPE_SECURITY_CONFIG   → processSecurityConfig()                     │
│  • MESSAGE_TYPE_BGP_PROFILE       → processBGPProfile()                         │
│  • MESSAGE_TYPE_ISIS_CONFIG       → processISISConfig()                         │
│  • MESSAGE_TYPE_INTERFACE_CONFIG  → processInterfaceConfig()                    │
│  • ... and more                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  STEP 4: Process Specific Config (e.g., processSystemConfig)                    │
│  ─────────────────────────────────────────────────────────                      │
│  • Updates NeData.Metadata (hostname, IP, neType, etc.)                         │
│  • Updates NeData.TemplateData based on device type (Spine/Leaf/A10NSP)         │
│  • Sets ZTP server IPs, JWKS endpoints, etc.                                    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  STEP 5: UpdateDbConfig - Save to Database                                      │
│  ─────────────────────────────────────────                                      │
│  • Persists NeData to Redis database                                            │
│  • Updates local cache for faster future access                                 │
│  • Returns success/failure status                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  FINAL OUTPUT: SetConfigResponse                                                │
│  ─────────────────────────────────                                              │
│  • Code: 0 (success) or 1 (error)                                               │
│  • Description: Human-readable result message                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📝 Step-by-Step Detailed Explanation

### Step 1: gRPC Handler - Entry Point
**File:** `pkg/adapters/server/grpcserver.go` (Line 277)

```go
func (h *grpcHandler) SetConfig(ctx context.Context, req *api.SetConfigRequest) (*api.SetConfigResponse, error)
```

#### What comes IN (Input):
```go
SetConfigRequest {
    DeviceInfo: {
        SerialNumber: "ABC123456",     // Unique switch identifier
        MatNumber: "40990640"           // Material/Model number
    },
    MessageType: MESSAGE_TYPE_SYSTEM_CONFIG,  // Type of config being set
    InterfaceContent: <protobuf Any>          // Actual config data (encoded)
}
```

#### What happens here:
1. **Validate DeviceInfo** - Checks if SerialNumber and MatNumber are present
2. **Extract InterfaceContent** - Decodes the protobuf `Any` type to actual Go struct
3. **Log the request** - For debugging/audit purposes
4. **Call the business logic** - Passes to `h.client.SetConfig()`

#### Key Function: `ValidateDeviceInfo`
```go
func ValidateDeviceInfo(deviceInfo *api.DeviceInfo) error {
    if deviceInfo == nil {
        return fmt.Errorf("DeviceInfo is nil")
    }
    if deviceInfo.SerialNumber == "" {
        return fmt.Errorf("SerialNumber is missing")
    }
    if deviceInfo.MatNumber == "" {
        return fmt.Errorf("MaterialNumber is missing")
    }
    return nil
}
```

#### Key Function: `extractInterfaceContent`
```go
func extractInterfaceContent(iface *anypb.Any) (proto.Message, error) {
    // Decodes protobuf Any to actual message type
    // Example: Converts Any → *api.SystemInfo
    msg, err := anypb.UnmarshalNew(iface, proto.UnmarshalOptions{})
    return msg, err
}
```

---

### Step 2: Business Logic - Client.SetConfig
**File:** `pkg/adapters/grpcinterface/handler.go` (Line 212)

```go
func (c *Client) SetConfig(serialNumber, matNumber string, messageType api.MessageType, content interface{}) error
```

#### What happens here:

1. **Input Validation**
```go
if serialNumber == "" {
    return errors.New("serial number is required")
}
if matNumber == "" {
    return errors.New("mat number is required")
}
if content == nil {
    return errors.New("content cannot be nil")
}
```

2. **Lock the device** (prevents race conditions)
```go
c.keylocker.Lock(serialNumber)
defer c.keylocker.Unlock(serialNumber)
```
> 💡 **Why?** If two requests come for the same switch simultaneously, only one can modify at a time.

3. **Fetch existing NE Data from database**
```go
neData, err := c.FetchNeData(ctx, serialNumber)
```

4. **If no data exists, create new NE Data** (only for SYSTEM_CONFIG)
```go
if errors.Is(err, ErrNeDataNotFound) {
    if messageType != api.MessageType_MESSAGE_TYPE_SYSTEM_CONFIG {
        return error // Can't create for non-system-config
    }
    neData, err = createNeData(ctx, serialNumber, matNumber, c)
}
```

5. **Initialize TemplateData if missing**
```go
if neData.TemplateData == nil {
    store.DefineTemplateData(serialNumber, neData)
    c.UpdateDbConfig(ctx, serialNumber, neData)
}
```

6. **Route to appropriate handler**
```go
err := c.handleConfigByType(serialNumber, neData, messageType, content)
```

7. **Save changes to database**
```go
c.UpdateDbConfig(ctx, serialNumber, neData)
```

---

### Step 3: Configuration Router - handleConfigByType
**File:** `pkg/adapters/grpcinterface/handler.go` (Line 266)

This is like a **switch statement** that routes different config types to their processors:

```go
func (c *Client) handleConfigByType(serialNumber string, neData *store.NeData, 
                                     messageType api.MessageType, content interface{}) error {
    switch messageType {
    case api.MessageType_MESSAGE_TYPE_SYSTEM_CONFIG:
        cfg := content.(*api.SystemInfo)
        return c.processSystemConfig(serialNumber, neData, cfg)
    
    case api.MessageType_MESSAGE_TYPE_IP_CONFIG:
        cfg := content.(*api.IpPoolConfig)
        return c.processIPConfig(neData, cfg)
    
    case api.MessageType_MESSAGE_TYPE_LAG_CONFIG:
        cfg := content.(*api.LAGConfig)
        return c.processLAGConfig(serialNumber, neData, cfg)
    
    // ... more cases for other config types
    }
}
```

#### Supported Message Types:
| MessageType | Expected Content Type | Processor Function |
|------------|----------------------|-------------------|
| `MESSAGE_TYPE_SYSTEM_CONFIG` | `*api.SystemInfo` | `processSystemConfig()` |
| `MESSAGE_TYPE_IP_CONFIG` | `*api.IpPoolConfig` | `processIPConfig()` |
| `MESSAGE_TYPE_LAG_CONFIG` | `*api.LAGConfig` | `processLAGConfig()` |
| `MESSAGE_TYPE_GRAYLOG_CONFIG` | `*api.GraylogConfig` | `processSyslogConfig()` |
| `MESSAGE_TYPE_NE_THRESHOLDS_CONFIG` | `*api.NeThresholds` | `processNeThresholdsConfig()` |
| `MESSAGE_TYPE_SECURITY_CONFIG` | `*api.Auth` | `processSecurityConfig()` |
| `MESSAGE_TYPE_IP2LOOPBACK_CONFIG` | `*api.Ip2LoopbackConfig` | `processIP2LoopbackConfig()` |
| `MESSAGE_TYPE_BGP_PROFILE` | `*api.RoutingConfig` | `processBGPProfile()` |
| `MESSAGE_TYPE_ISIS_CONFIG` | `*api.IsisConfig` | `processISISConfig()` |
| `MESSAGE_TYPE_L2VRF_INSTANCE_CONFIG` | `*api.L2VrfInstance` | `processL2VRFInstance()` |
| `MESSAGE_TYPE_INTERFACE_CONFIG` | `*api.InterfaceConfig` | `processInterfaceConfig()` |
| `MESSAGE_TYPE_MGMT_CONFIG` | `*api.MgmtConfig` | `processMgmtConfig()` |
| `MESSAGE_TYPE_L3VRF_INSTANCE_CONFIG` | `*api.L3VrfInstance` | `processL3VRFInstance()` |

---

### Step 4: Config Processor Example - processSystemConfig
**File:** `pkg/adapters/grpcinterface/template_utils.go` (Line 850)

This function updates device metadata and template data based on the system configuration:

```go
func (c *Client) processSystemConfig(serialNumber string, neData *store.NeData, 
                                      sysCfg *api.SystemInfo) error {
    // 1. Update Metadata
    if sysCfg.GetSystemName() != "" {
        neData.Metadata.Hostname = sysCfg.GetSystemName()
    }
    if sysCfg.GetOutBand() != "" {
        neData.Metadata.IP = sysCfg.GetOutBand()
    }
    // ... more metadata updates
    
    // 2. Update TemplateData based on device type
    switch neData.Metadata.NeType {
    case adapters.CategorySpine:
        td := neData.TemplateData.(*models.SpineTemplateData)
        td.ElementName = sysCfg.GetSystemName()
        td.ZTPServerOutband = []models.ZTPServerOutband{{IPAddress: ztpServerOutband}}
        // ... more spine-specific config
        
    case adapters.CategoryLeaf:
        td := neData.TemplateData.(*models.LeafTemplateData)
        td.ElementName = sysCfg.GetSystemName()
        td.NasIdentifier = sysCfg.GetNasIdentifier()
        // ... more leaf-specific config
        
    case adapters.CategoryA10NSP:
        // A10NSP-specific configuration
    }
    return nil
}
```

---

### Step 5: Database Operations
**File:** `pkg/adapters/grpcinterface/inventory.go`

#### FetchNeData (Read from DB)
```go
func (c *Client) FetchNeData(ctx context.Context, serialNumber string) (*store.NEData, error) {
    nedata, err := c.GetDbConfig(ctx, serialNumber)
    if err != nil {
        if err.Error() == database.ErrKeyNotFound {
            return nil, ErrNeDataNotFound  // Device doesn't exist yet
        }
        return nil, err
    }
    return nedata, nil
}
```

#### UpdateDbConfig (Write to DB)
```go
func (c *Client) UpdateDbConfig(ctx context.Context, serialNumber string, nedata *store.NeData) error {
    // 1. Save to Redis database
    err := c.dbHandler.UpdatePhysicalInfo(ctx, BasePath+serialNumber, nedata)
    
    // 2. Update local cache for faster future reads
    if nedata.Metadata != nil {
        c.cache.Set(serialNumber, nedata.Metadata, cache.DefaultExpiration)
    }
    return err
}
```

#### createNeData (Create new device record)
```go
func createNeData(ctx context.Context, serialNumber, matNumber string, client *Client) (*store.NeData, error) {
    // 1. Create metadata from material number
    metadata, err := createMetadataFromMatNumber(matNumber, serialNumber)
    
    // 2. Initialize NeData
    neData := &store.NeData{Metadata: metadata}
    
    // 3. Define TemplateData based on device type
    store.DefineTemplateData(serialNumber, neData)
    
    // 4. Save to database
    client.UpdateDbConfig(ctx, serialNumber, neData)
    
    return neData, nil
}
```

---

## 📦 Data Structures

### NeData (Network Element Data)
```go
type NEData struct {
    Metadata     *Metadata        // Device identification info
    TemplateData interface{}      // Configuration template (Spine/Leaf/A10 specific)
}
```

### Metadata
```go
type Metadata struct {
    MatNumber             string         // Material/Model number (e.g., "40990640")
    IP                    string         // Out-of-band management IP
    NeId                  string         // Network Element ID
    SerialNumber          string         // Unique device identifier
    NeConfigID            int            // Configuration ID
    NeType                string         // "spine", "leaf", or "a10nsp"
    Hostname              string         // Device hostname
    TargetVersion         Version        // Target software version
    ActiveVersion         Version        // Current software version
    SwitchUpgradeStatus   UpgradeStatus  // Upgrade state
    SwitchReadyForService bool           // Ready status
}
```

### Material Number Mapping
```go
// Material Number → Device Type
"40990638", "47225909"          → A10NSP (NeConfigID: 60)
"40990639", "47185339"          → Spine  (NeConfigID: 11)
"40990640", "47185340", "47185341" → Leaf   (NeConfigID: 21)
```

---

## 🔄 Response Structure

### Success Response
```go
SetConfigResponse {
    Code: 0,
    Description: "Config set successfully"
}
```

### Error Responses
```go
// Validation Error
SetConfigResponse {
    Code: 1,
    Description: "DeviceInfo is missing or invalid"
}

// Processing Error
SetConfigResponse {
    Code: 1,
    Description: "SetConfig failed : <error details>"
}
```

---

## 🎯 Simple Analogy

Think of SetConfig like **updating a patient's medical record**:

1. **Reception (gRPC Handler)**: Patient arrives with their ID card
   - Check ID is valid ✅
   - Determine what kind of update needed (blood test, x-ray, etc.)

2. **Medical Records (Client.SetConfig)**: 
   - Pull patient's existing file from cabinet
   - If new patient, create a file
   - Lock the file so only one doctor can update

3. **Specialist Department (handleConfigByType)**:
   - Blood test results → Send to Lab department
   - X-ray results → Send to Radiology
   - Each department knows how to process their data

4. **Update Records (processSystemConfig, etc.)**:
   - Write new information into patient file
   - Update different sections based on data type

5. **Filing Cabinet (UpdateDbConfig)**:
   - Save updated file back to storage
   - Keep a quick-reference copy in recent files (cache)

6. **Confirmation**: Tell reception "Patient record updated successfully!"

---

## 🔧 Key Points to Remember

1. **Thread Safety**: The `KeyedLocker` ensures only one request can modify a device's config at a time.

2. **SYSTEM_CONFIG is Special**: Only `MESSAGE_TYPE_SYSTEM_CONFIG` can create a new device record. All other config types require the device to already exist.

3. **Three Device Types**: The system supports Spine, Leaf, and A10NSP switches, each with their own template data structure.

4. **Two-Level Storage**: Data is stored in Redis (persistent) and cached in memory (fast access).

5. **Protobuf Any Type**: The `InterfaceContent` uses protobuf's `Any` type, allowing different message types to be sent through the same endpoint.

---

## 📁 File References

| File | Purpose |
|------|---------|
| `pkg/adapters/server/grpcserver.go` | gRPC handler entry point |
| `pkg/adapters/grpcinterface/handler.go` | Business logic for SetConfig |
| `pkg/adapters/grpcinterface/template_utils.go` | Config processors & createNeData |
| `pkg/adapters/grpcinterface/inventory.go` | Database operations |
| `pkg/adapters/store/store.go` | Data structures (NeData, Metadata) |

---

## 📝 Example Call Flow

```
Request: Set system config for a new Leaf switch

1. Client sends SetConfigRequest:
   - SerialNumber: "LEAF001"
   - MatNumber: "40990640" (Leaf)
   - MessageType: SYSTEM_CONFIG
   - InterfaceContent: SystemInfo{SystemName: "leaf-switch-01", OutBand: "192.168.1.10"}

2. grpcserver.SetConfig():
   - Validates DeviceInfo ✅
   - Extracts SystemInfo from InterfaceContent
   - Calls client.SetConfig("LEAF001", "40990640", SYSTEM_CONFIG, SystemInfo)

3. handler.SetConfig():
   - Locks "LEAF001"
   - FetchNeData returns ErrNeDataNotFound (new device)
   - Calls createNeData() → Creates NeData with LeafTemplateData
   - Calls handleConfigByType(SYSTEM_CONFIG)

4. handleConfigByType():
   - Routes to processSystemConfig()

5. processSystemConfig():
   - Sets neData.Metadata.Hostname = "leaf-switch-01"
   - Sets neData.Metadata.IP = "192.168.1.10"
   - Sets LeafTemplateData.ElementName = "leaf-switch-01"
   - Configures ZTP servers, JWKS endpoints

6. UpdateDbConfig():
   - Saves NeData to Redis: /pao/neconfig/LEAF001
   - Updates cache

7. Returns SetConfigResponse{Code: 0, Description: "Config set successfully"}
```
