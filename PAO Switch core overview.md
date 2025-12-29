# PAO Switch Core - Beginner's Guide 🚀

> A simple explanation of what this project does and how it works

## Table of Contents
1. [What is This Project?](#what-is-this-project)
2. [The Big Picture](#the-big-picture)
3. [Key Components Explained](#key-components-explained)
4. [How It Works (Step by Step)](#how-it-works-step-by-step)
5. [Project Structure](#project-structure)
6. [Main Technologies Used](#main-technologies-used)
7. [Getting Started](#getting-started)
8. [Common Terms Explained](#common-terms-explained)

---

## What is This Project?

**PAO Switch Core** is like a **smart traffic controller** for network switches (the devices that connect computers and devices in a network). It is a **key component of the EMS (Element Management System)** that orchestrates switch configurations in the PAO platform.

### In Simple Words:
Imagine you have a big office building with thousands of network connections. This project helps:
- **Configure** network switches automatically (set them up)
- **Monitor** if switches are working properly (like a health check)
- **Fix issues** when something goes wrong
- **Manage** who can connect to the network and how

Instead of manually configuring each switch, this software does it automatically by talking to different types of network equipment from various vendors (like Juniper, RTBrick, etc.).

### Relationship with EMS:
This project is **part of the larger EMS ecosystem**. It integrates with:
- **EMS Pod Inventory** (`ems-pod-customers-web-cli`) - for getting network element information
- **EMS Software Update Service** - for firmware/software updates
- **EMS Connectivity State Service** - for monitoring device connectivity
- **EMS Patch Operational Data** - for updating device operational status

---

## The Big Picture

Think of this project as a **translator and manager** between:
1. **Your requests** (like "I want to add a new user to VLAN 100")
2. **Network switches** (the actual hardware that needs configuration)

```
┌─────────────────────────────────────────────────────────────┐
│                    EMS & External Systems                    │
│   (EMS Inventory, OSS, Management Tools, API Clients)       │
│                                                              │
│  • EMS Pod Inventory (device info)                          │
│  • EMS Software Update (firmware updates)                   │
│  • EMS Connectivity State (device status)                   │
│  • EMS Operational Data (device operational info)           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
          ┌──────────────────────────────┐
          │   PAO Switch Core            │
          │   (This Application)         │
          │                              │
          │  - Receives requests         │
          │  - Queries EMS Inventory     │
          │  - Processes logic           │
          │  - Manages workflows         │
          │  - Updates EMS systems       │
          └──────────┬───────────────────┘
                     │
                     ▼
          ┌──────────────────────────────┐
          │   Vendor Adapters            │
          │   (Talk to specific brands)  │
          └──────────┬───────────────────┘
                     │
                     ▼
          ┌──────────────────────────────┐
          │   Network Switches           │
          │   (Actual Hardware)          │
          └──────────────────────────────┘
```

---

## Key Components Explained

### 1. **HTTP/gRPC APIs** (The Front Door)
- **What it does**: Accepts requests from external systems
- **Think of it as**: The reception desk of a hotel - it receives your requests
- **Example**: "Add user ABC to VLAN 100"

### 2. **Core Orchestrator** (The Brain)
- **What it does**: Decides what needs to be done and in what order
- **Think of it as**: The hotel manager who coordinates everything
- **Example**: Figures out which switch to configure and what settings to apply

### 3. **Adapter Gateway** (The Translator)
- **What it does**: Translates generic commands into vendor-specific language
- **Think of it as**: A translator who speaks multiple languages
- **Example**: Converts "set VLAN" into Juniper-specific or RTBrick-specific commands

### 4. **Vendor Adapters** (The Communicators)
- **What it does**: Actually talks to the network equipment
- **Think of it as**: The person who physically goes to each switch
- **Example**: Sends configuration commands to a Juniper switch

### 5. **Event Bus** (The Notification System)
- **What it does**: Sends and receives messages about what's happening
- **Think of it as**: The hotel's intercom system
- **Example**: "Switch XYZ is now online" or "Alarm detected on Port 5"

### 6. **Storage (Redis)** (The Memory)
- **What it does**: Stores temporary data and events
- **Think of it as**: The hotel's filing cabinet
- **Example**: Remembers recent configuration changes

### 7. **EMS Integration** (The Information Hub)
- **What it does**: Gets network device information from EMS Inventory system
- **Think of it as**: The hotel's customer database
- **Example**: Looks up switch details, port mappings, and device characteristics

---

## How It Works (Step by Step)

### Scenario: Adding a New Internet Subscriber

**Step 1: Request Arrives**
```
External System → HTTP API
"Please set up VLAN 200 for customer on Leaf Switch 1, Port 10"
```

**Step 2: Core Processes Request**
```
Core receives → Validates request → Queries EMS Inventory for switch details
            → Gets device characteristics from EMS
```

**Step 3: Adapter Gateway Prepares**
```
Gateway → Finds correct adapter for "Leaf Switch 1" (let's say it's Juniper)
        → Translates request into Juniper format
```

**Step 4: Adapter Executes**
```
Juniper Adapter → Connects to physical switch
                → Sends configuration commands
                → Waits for confirmation
```
**Step 5: Response Returns**
```
Switch confirms → Adapter reports success → Core updates records
                → Updates EMS Operational Data
                → HTTP API sends "Success" to requester
```             → HTTP API sends "Success" to requester
```

**Step 6: Events Published**
```
Event Bus → Publishes "VLAN configured on Switch 1"
          → Other systems can react to this event
```

---

## Project Structure

Here's what each folder contains:

```
pao-switch-core/
│
├── cmd/                          # Main programs (executables)
│   ├── pao-switch-core/          # The main application
│   ├── adapter-simulator/        # Fake adapter for testing
│   └── artefact-simulator/       # Fake artefact system for testing
│
├── internal/                     # Private code (core logic)
│   ├── core/                     # Main business logic
│   │   ├── assurance/            # Health monitoring & alarms
│   │   ├── systemconfig/         # Switch setup & configuration
│   │   └── subscriberconfig/     # User/subscriber management
│   ├── adaptergw/                # Adapter gateway (translator)
│   └── ippool/                   # IP address management
│
├── pkg/                          # Reusable packages
│   ├── httpinterface/            # HTTP API handlers
│   ├── grpcinterface/            # gRPC API handlers
│   ├── clients/                  # Clients for external services
│   │   ├── inventory/            # Talks to inventory system
│   │   ├── topology/             # Talks to topology system
│   │   └── storage/              # Talks to Redis
│   └── config/                   # Configuration structures
│
├── config/                       # Configuration files
│   └── pao-switch-core/
│       └── configmap.yaml        # Main config file
│
├── deployments/                  # Kubernetes deployment files
│   └── charts/                   # Helm charts
│
├── scripts/                      # Helper scripts
│   └── summarize-rpc-events.py   # Analyze performance
│
└── docs/                         # Documentation
    ├── Architecture.md           # Detailed architecture
    └── index.md                  # Documentation home
```

---

## Main Technologies Used
### Programming Language
- **Go (Golang)**: A fast, modern programming language used for building the application

### Communication Protocols
- **HTTP/REST**: Simple web-based API for requests
- **gRPC**: Fast, efficient protocol for talking to adapters
- **Protobuf**: Efficient data format for messages

### Infrastructure
- **Kafka/NATS**: Message bus for events (like a post office)
- **Redis**: Fast in-memory database for temporary storage
- **Kubernetes**: Container orchestration (runs the app in production)

### EMS Integration Libraries
- **ems-pod-customers-web-cli**: Core EMS APIs for inventory, connectivity state, and software updates
- **pod-inventory/v2**: EMS Inventory data models and client
- **software-update/v1**: EMS Software update service APIs
- **pod-connectivitystate/v1**: EMS Connectivity monitoring
- **patch-operational-data/v1**: EMS Operational data updates

### Testing
- **Simulators**: Fake adapters and switches for testing without real hardware
- **MockGen**: Creates fake versions of code for unit testingout real hardware
- **MockGen**: Creates fake versions of code for unit testing

---

## Getting Started

### Prerequisites
- Go 1.x installed
- Basic understanding of networking (VLANs, switches)
- (Optional) Docker for running with containers

### Quick Start

**1. Download dependencies**
```bash
go mod download
```

**2. Build the application**
```bash
make build
```

**3. Run with simulators (no real hardware needed)**

Terminal 1 - Start adapter simulator:
```bash
go run ./cmd/adapter-simulator
```

Terminal 2 - Start the main application:
```bash
CONFIG_PATH=./config/pao-switch-core/configmap.yaml \
go run ./cmd/pao-switch-core
```

**4. Test it**
The application will be running on:
- HTTP API: `http://localhost:8080`
- gRPC API: `localhost:8081`

### Example API Call
```bash
# Set a VLAN configuration
curl -X PUT http://localhost:8080/api/v1/vlan \
  -H "Content-Type: application/json" \
  -d '{
    "leafResourceID": "switch-1",
    "leafPort": "eth0/1",
    "vlanID": "100"
  }'
```

---

## Common Terms Explained

### Network Terms

| Term | Simple Explanation |
|------|-------------------|
| **Switch** | A device that connects computers/devices in a network |
| **VLAN** | Virtual LAN - a way to separate network traffic into groups |
| **Port** | A physical connection point on a switch |
| **Adapter** | Software that talks to specific hardware brands |
| **Leaf/Spine** | Network architecture - Leaf switches connect to devices, Spine switches connect Leaf switches |

### Technical Terms

| **Artefact** | Configuration or data package |
| **Assurance** | Monitoring and ensuring things work correctly |
| **EMS** | Element Management System - manages network elements |

### Application-Specific Termsen different systems |
| **RPC** | Remote Procedure Call - calling functions on another computer |
| **Protobuf** | A way to efficiently send structured data |
| **Event Bus** | A system for publishing and subscribing to notifications |
| **Artefact** | Configuration or data package |
| **Assurance** | Monitoring and ensuring things work correctly |

### Application-Specific Terms

| Term | Simple Explanation |
| **PAO** | The name of the overall platform/system (includes EMS components) |
| **NE (Network Element)** | Any piece of network equipment |
| **Subscriber Config** | Settings for end-users/customers |
| **System Config** | Settings for the switches themselves |
| **L2X/L2BSA** | Layer 2 connection types |
| **Bring-up** | The process of initializing a new switch |
| **Info Model** | EMS data model for network resources and characteristics |
| **Logical Resource** | EMS representation of network elements in inventory |
| **Bring-up** | The process of initializing a new switch |

---

## What Can This System Do?

### Main Features

1. **Subscriber Management**
   - Add/remove customers
   - Configure VLANs
   - Set up internet connections (PPPoE)

2. **Switch Configuration**
   - Initialize new switches
   - Update switch settings
   - Apply firmware/software updates

3. **Monitoring & Alarms**
   - Detect when switches go offline
   - Monitor port status
   - Handle alarm notifications

4. **Automation**
   - Automatic configuration based on inventory
   - Self-healing when issues are detected
   - Bulk operations on multiple switches

5. **Multi-Vendor Support**
   - Works with different switch brands
   - Translates commands for each vendor
   - Unified interface regardless of hardware

---

## Development Workflow

### For Developers

**Adding a New Feature (Example: New VLAN Type)**

1. **Define the API** (`pkg/httpinterface/` or `pkg/grpcinterface/`)
   - Add endpoint handler

2. **Add Business Logic** (`internal/core/`)
   - Implement the feature logic

3. **Add Adapter Support** (`internal/adaptergw/`)
   - Add translation for each vendor

4. **Test with Simulator** (`internal/adaptersimulator/`)
   - Test without real hardware

5. **Write Unit Tests**
   - Ensure code works correctly

6. **Deploy**
   - Update Kubernetes manifests in `deployments/`

---

## Debugging & Troubleshooting

### Debug Endpoints

The application provides debug endpoints to see what's happening:

```bash
# View recent RPC events (calls to adapters)
curl http://localhost:8080/api/v1/debug/systemconfig/rpc-events

# View assurance events (alarms, monitoring)
curl http://localhost:8080/api/v1/debug/object-changed/events

# View IP assignment events
curl http://localhost:8080/api/v1/debug/ip-assignment/events
```

### Log Analysis

Check logs for:
- **INFO**: Normal operations
- **WARN**: Potential issues
- **ERROR**: Things that failed

### Performance Analysis

Use the included script to analyze performance:
```bash
# Collect events
./scripts/collect-rpc-events.sh OUT_DIR=./events

# Analyze performance
python3 scripts/summarize-rpc-events.py ./events --sort-by p95
```

---

### Use Case 1: New Building Setup
**Scenario**: A new office building with 100 network ports needs to be connected.

**What happens**:
1. Network equipment is installed and powered on
2. EMS Inventory system registers new switches (via discovery)
3. PAO Switch Core queries EMS Inventory for new devices
4. Retrieves device characteristics from EMS (model, ports, capabilities)
5. Automatically applies base configuration via adapters
6. Updates EMS Operational Data with configuration status
7. Tests all ports
8. Reports ready for use in EMS base configuration
### Use Case 2: Customer Onboarding
**Scenario**: A new internet subscriber signs up.

**What happens**:
1. OSS system sends subscriber details to PAO Switch Core
2. Core queries EMS Inventory to find available port on nearest switch
3. Retrieves port and switch characteristics from EMS
4. Configures VLAN and PPPoE settings on the switch
5. Assigns IP address from pool
6. Updates EMS Operational Data with subscriber assignment
7. Activates connection
8. Publishes event to EMS message bus
9. Customer can now use internettings
4. Assigns IP address from pool
### Use Case 3: Fault Detection
**Scenario**: A switch loses power.

**What happens**:
1. EMS Connectivity State service detects switch offline
2. Publishes reachability alarm to event bus
3. PAO Switch Core receives alarm event
4. Queries EMS Inventory for affected switch details
5. Logs incident to EMS Operational Data
6. Attempts reconnection when device comes back online
7. Updates EMS with recovery status
8. Notifies operations team if needed
4. Logs incident
5. Attempts reconnection
6. Notifies operations team if needed

---

| File | What It Does |
|------|-------------|
| `cmd/pao-switch-core/main.go` | Entry point - starts the application |
| `config/pao-switch-core/configmap.yaml` | Main configuration file |
| `internal/core/core.go` | Main orchestration logic (integrates with EMS) |
| `internal/adaptergw/*` | Adapter gateway controllers |
| `pkg/clients/inventory/*` | EMS Inventory client - fetches device info |
| `pkg/clients/patchoperationaldata/*` | Updates EMS Operational Data |
| `pkg/httpinterface/vlan.go` | HTTP API for VLAN operations |
| `Makefile` | Build and test commands |ay controllers |
| `pkg/httpinterface/vlan.go` | HTTP API for VLAN operations |
| `Makefile` | Build and test commands |

---

## Next Steps for Learning

1. **Start with Simulators**: Run the application with simulators to see it in action
2. **Read the Architecture Doc**: Check `docs/Architecture.md` for deeper details
3. **Follow the Code**: Trace a request from HTTP API → Core → Adapter
4. **Experiment**: Try making API calls and watch the logs
5. **Contribute**: Pick a small issue and try fixing it

---

## Resources

- **Main Documentation**: `docs/index.md`
- **Architecture Details**: `docs/Architecture.md`
- **API Documentation**: `api/README.md`
- **Contributing Guide**: `CONTRIBUTING.md`

---

## Summary

**PAO Switch Core** is a sophisticated network automation system that:
- ✅ Automates network switch configuration
- ✅ Supports multiple vendor types
- ✅ Monitors network health
- ✅ Manages subscribers/customers
- ✅ Provides REST and gRPC APIs
- ✅ Handles events and alarms
- ✅ Scales to manage thousands of switches

It's like having a smart assistant that manages your entire network infrastructure automatically! 🎯

---

**Happy Learning! 🚀**

*For questions or issues, check the project's issue tracker or reach out to the development team.*
