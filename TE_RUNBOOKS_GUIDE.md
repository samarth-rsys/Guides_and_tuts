# 🎯 Complete Guide to te-runbooks (Test Executor Runbooks)

> **For Beginners**: This guide explains everything in simple terms. All networking concepts are explained as if you're new to networking!

---

## 📖 Table of Contents

1. [What is This Project?](#what-is-this-project)
2. [Networking Basics for Beginners](#-networking-basics-for-beginners)
3. [Project Structure](#-project-structure-explained)
4. [Understanding the Test Modules](#-understanding-the-test-modules)
5. [Key Components Deep Dive](#-key-components-deep-dive)
6. [How Tests Work](#-how-tests-work)
7. [Tools & Dependencies](#-tools--dependencies)
8. [Running Tests](#-running-tests)
9. [Quick Reference](#-quick-reference)
10. [Troubleshooting](#-troubleshooting-guide)

---

## What is This Project?

This project is a **test automation framework** for testing **network equipment** (routers/switches) from a company called **RtBrick**. It's used by Deutsche Telekom for their "Access 4.0" (A4) fiber network infrastructure.

### Simple Analogy 🏠

Think of it like testing a highway system:
- 🏠 **Your home** = A subscriber/customer
- 🛣️ **On-ramp to highway** = Leaf router (where you connect)
- 🛣️ **Main highway** = Spine router (carries traffic between areas)
- 🚗 **Your car** = Your data packets
- 🚦 **Toll booth** = RADIUS server (checks if you're allowed to enter)
- 📋 **This project** = Automated tests that verify all parts work correctly

---

## 🌐 Networking Basics for Beginners

Before diving into the tests, let's understand the networking concepts used throughout this project.

### 1. What is an IP Address? 🏷️

An IP address is like a **home address for your computer** on a network.

| Type | Example | Description |
|------|---------|-------------|
| **IPv4** | `192.168.1.100` | The "old" format - 4 numbers separated by dots (0-255 each) |
| **IPv6** | `2003:4:f021::1` | The "new" format - longer, uses numbers and letters |

**Why do we need both?**
- IPv4 is running out of addresses (like running out of phone numbers)
- IPv6 provides trillions more addresses
- "Dual Stack" means supporting BOTH at the same time

### 2. What is a MAC Address? 🔖

A **MAC address** is a unique hardware identifier for network devices - like a **serial number** for your network card.

```
Example: 00:0c:29:04:01:00
         ↑
         6 pairs of hexadecimal numbers
```

### 3. What is a VLAN? 🏢

**VLAN** (Virtual Local Area Network) is like having **separate floors in a building**.

```
Physical Network (One Building)
├── VLAN 100 (Sales Department - Floor 1)
├── VLAN 200 (Engineering - Floor 2)
└── VLAN 300 (HR - Floor 3)
```

Even though everyone is in the same building, each floor is isolated. Similarly, VLANs create **virtual separation** on the same physical network.

| Term | Meaning |
|------|---------|
| **Outer VLAN (S-VLAN)** | Service VLAN - identifies the service provider |
| **Inner VLAN (C-VLAN)** | Customer VLAN - identifies individual customer |

### 4. What is PPPoE? 📡

**PPPoE** (Point-to-Point Protocol over Ethernet) is how your home router **logs into your ISP**.

```
Your Router                    ISP's Router (BNG)
    │                               │
    │── PADI (I want to connect) ──→│
    │←── PADO (OK, here I am) ──────│
    │── PADR (Connect me please) ──→│
    │←── PADS (You're connected!) ──│
    │                               │
    │   Now authenticate (PAP/CHAP) │
    │   Get IP address              │
    │   Start browsing! 🌐          │
```

**Key PPPoE Terms:**

| Term | Full Name | What It Does |
|------|-----------|--------------|
| **PADI** | PPPoE Active Discovery Initiation | "Hello, is anyone there?" |
| **PADO** | PPPoE Active Discovery Offer | "Yes, I can connect you" |
| **PADR** | PPPoE Active Discovery Request | "Please connect me" |
| **PADS** | PPPoE Active Discovery Session-confirmation | "You're now connected" |
| **PADT** | PPPoE Active Discovery Terminate | "Goodbye, disconnecting" |

### 5. What is RADIUS? 🔐

**RADIUS** (Remote Authentication Dial-In User Service) is like a **bouncer at a club**:

1. **Authentication**: "Are you on the guest list?" (username/password check)
2. **Authorization**: "What areas can you access?" (permissions)
3. **Accounting**: "How long did you stay?" (usage tracking)

```
User → Router → RADIUS Server
         │           │
         │ "Is user123 allowed?" →
         │           │
         │ ← "Yes, give them IP 1.2.3.4"
         │           │
         │ "User123 connected for 2 hours" →
```

### 6. What are PAP and CHAP? 🔑

These are **authentication methods** (ways to prove who you are):

| Method | How It Works | Security Level |
|--------|-------------|----------------|
| **PAP** | Sends password in plain text | Low 🔓 |
| **CHAP** | Uses encrypted challenge-response | High 🔒 |

**CHAP Process:**
```
Server: "Here's a random challenge: ABC123"
Client: "My answer is: [ABC123 + password] hashed = XYZ789"
Server: "Correct! You're authenticated"
```

### 7. What is L2TP? 🚇

**L2TP** (Layer 2 Tunneling Protocol) creates a **virtual tunnel** through the internet.

Think of it like a **private subway tunnel**:
```
Your House ═══════════════════════════════ Destination
            [L2TP Tunnel - Private Path]
             (Data travels securely inside)
```

**Key Terms:**
- **LAC** (L2TP Access Concentrator): Entry point of the tunnel (your side)
- **LNS** (L2TP Network Server): Exit point of the tunnel (ISP side)

### 8. What is BGP? 🗺️

**BGP** (Border Gateway Protocol) is the **GPS of the internet**. It helps routers find the best path to send data.

```
     ┌─────────────────────────────────────┐
     │           THE INTERNET              │
     │  ┌───┐    ┌───┐    ┌───┐    ┌───┐  │
     │  │ A │────│ B │────│ C │────│ D │  │
     │  └───┘    └───┘    └───┘    └───┘  │
     │    │        │        │        │    │
     │   BGP tells each router the best   │
     │   path to reach any destination    │
     └─────────────────────────────────────┘
```

| Type | What It Means |
|------|---------------|
| **iBGP** | Internal BGP - between routers in the SAME organization |
| **eBGP** | External BGP - between DIFFERENT organizations |

### 9. What is IS-IS? 🛤️

**IS-IS** (Intermediate System to Intermediate System) is another **routing protocol** - it helps routers inside a network find paths to each other.

**Difference from BGP:**
- **IS-IS**: Used INSIDE a network (like a city map)
- **BGP**: Used BETWEEN networks (like a world map)

### 10. What is QoS? 🚦

**QoS** (Quality of Service) is like having **express lanes on a highway**:

```
All Traffic
    │
    ├─→ 🚑 Voice (VoIP) ────→ HIGHEST PRIORITY (Express lane)
    ├─→ 📺 Video (IPTV) ────→ HIGH PRIORITY
    ├─→ 🎮 Gaming ──────────→ MEDIUM PRIORITY
    └─→ 📧 Email/Web ───────→ BEST EFFORT (Normal lane)
```

**Common QoS Terms:**

| Term | Meaning |
|------|---------|
| **Shaping** | Limiting maximum speed (like a speed limit) |
| **Policing** | Dropping excess traffic (like a bouncer) |
| **Marking** | Labeling traffic priority (like VIP tags) |
| **Scheduling** | Deciding which packets go first |

### 11. What is Multicast? 📺

**Multicast** is sending data to **multiple receivers at once** (like TV broadcast):

```
                      ┌── User A 📺
Source ───→ Router ───┼── User B 📺   (One stream, many viewers)
                      └── User C 📺
```

**Unicast vs Multicast:**
- **Unicast**: One sender → One receiver (like a phone call)
- **Multicast**: One sender → Many receivers (like radio broadcast)

### 12. What is DHCP? 🏷️

**DHCP** (Dynamic Host Configuration Protocol) **automatically assigns IP addresses** to devices.

```
New Device: "I need an IP address!"
DHCP Server: "Here's 192.168.1.50, use it for 24 hours"
```

**DHCPv6-PD** (Prefix Delegation): Assigns a whole **block of IPv6 addresses** to a router.

### 13. What is MTU/MRU? 📦

| Term | Meaning | Simple Explanation |
|------|---------|-------------------|
| **MTU** | Maximum Transmission Unit | Maximum package size the network can carry |
| **MRU** | Maximum Receive Unit | Maximum package size a device can receive |

Think of it like **maximum box size** that fits through a door:
```
Data ═══════════════════════════════════
      ↓ Split into packets ↓
      [████] [████] [████] [██]
      Each packet ≤ MTU size
```

### 14. What is a Loopback Address? 🔄

A **loopback address** is a router's **permanent "home address"** that never changes:

```
Router has many interfaces:
├── ifp-0/1/1 → 10.1.1.1 (can go down)
├── ifp-0/1/2 → 10.2.2.2 (can go down)
└── lo0       → 192.168.100.1 (ALWAYS UP - the loopback)
```

### 15. What are Network Namespaces? 📦

**Network namespaces** create **isolated network environments** on one machine:

```
One Linux Server
├── Namespace A (its own network stack, IPs, routes)
├── Namespace B (completely separate network)
└── Namespace C (another isolated network)
```

Used for testing multiple network scenarios simultaneously.

### 16. What is LCP? 🔗

**LCP** (Link Control Protocol) is the **first negotiation** in a PPP connection:

```
Client                    Server
   │                         │
   │── LCP Config-Request ──→│  "Here's what I support"
   │←── LCP Config-Ack ──────│  "OK, I accept"
   │←── LCP Config-Request ──│  "Here's what I support"
   │── LCP Config-Ack ──────→│  "OK, I accept"
   │                         │
   │   Link is now established! ✅
```

**LCP Negotiates:**
- MRU (packet size)
- Authentication method (PAP/CHAP)
- Magic number (loop detection)
- Echo interval (keepalive)

### 17. What is IPCP? 📍

**IPCP** (Internet Protocol Control Protocol) negotiates **IPv4 settings** after LCP:

```
After LCP completes:
   │                         │
   │── IPCP Config-Request ─→│  "I need an IP address"
   │←── IPCP Config-Nak ─────│  "Use this IP: 1.2.3.4"
   │── IPCP Config-Request ─→│  "OK, I'll use 1.2.3.4"
   │←── IPCP Config-Ack ─────│  "Confirmed!"
```

### 18. What is CoA (Change of Authorization)? 🔄

**CoA** allows the RADIUS server to **change user settings mid-session**:

```
User connected with 100 Mbps
        │
RADIUS Server: "User upgraded, give them 1 Gbps"
        │
        ↓
Router applies new speed immediately!
```

**Use Cases:**
- Upgrade/downgrade speed
- Change QoS profile
- Add/remove services
- Enable/disable features

### 19. What is ADF (Access Data Filter)? 🚧

**ADF** is like a **security checkpoint** that filters traffic:

```
Incoming Traffic
       │
   ┌───┴───┐
   │  ADF  │
   └───┬───┘
       │
   ┌───┼───┐
   │   │   │
   ✅  ✅  ❌
Allow Allow Block
```

**ADF Can:**
- Block specific IPs
- Filter by protocol (TCP/UDP)
- Filter by port number
- Apply rate limits

### 20. What is a BNG? 🌐

**BNG** (Broadband Network Gateway) is the **main router** that:

1. Terminates subscriber sessions (PPPoE/IPoE)
2. Authenticates users via RADIUS
3. Assigns IP addresses
4. Applies QoS policies
5. Routes traffic to the internet

In this project, the **LEAF** routers act as BNGs.

---

## 📁 Project Structure Explained

### Complete Folder Tree

```
te-runbooks/
├── 📄 .gitlab-ci.yml          # CI/CD pipeline configuration
├── 📄 A4Hosts.yml             # All device credentials and IPs
├── 📄 A4Hosts_blr.yml         # Bangalore-specific hosts
├── 📄 README.md               # Project documentation
├── 📄 testlist.csv            # List of test cases
├── 📄 testcase_list_SE.csv    # SE test case list
│
├── 📁 00_Templates/           # Base templates for test generation
│   ├── 01_Check_Testbed/
│   ├── 04_PPPOE/
│   └── ...
│
├── 📁 01_Check_Testbed/       # Tests: Verify test environment ready
├── 📁 02_Documentation/       # Tests: Documentation verification
├── 📁 03_L2x/                 # Tests: Layer 2 switching
├── 📁 04_PPPOE/               # Tests: PPPoE subscriber sessions
├── 📁 05_L2TP/                # Tests: L2TP VPN tunnels
├── 📁 06_ISIS/                # Tests: IS-IS routing protocol
├── 📁 07_BGP/                 # Tests: BGP routing protocol
├── 📁 08_Multicast/           # Tests: Multicast/IPTV
├── 📁 09_L2BSA/               # Tests: Layer 2 Bitstream Access
├── 📁 10_Performance/         # Tests: Performance/load testing
├── 📁 11_LI/                  # Tests: Lawful Interception
├── 📁 12_Maintenance/         # Tests: Maintenance operations
├── 📁 13_Reliability/         # Tests: Failover/reliability
├── 📁 15_DHCP/                # Tests: DHCP protocol
├── 📁 16_NTP/                 # Tests: Time synchronization
├── 📁 17_Security/            # Tests: Security features
├── 📁 18_RD/                  # Tests: Route Distinguisher
├── 📁 19_End2End/             # Tests: End-to-end integration
├── 📁 20_QoS/                 # Tests: Quality of Service
├── 📁 21_P3OE/                # Tests: P3OE protocol
├── 📁 90_Configure_Testbed/   # Tests: Testbed configuration
│
├── 📁 macros/                 # Reusable test functions
│   ├── BLR1LEAF01.yml        # Device-specific variables
│   ├── FAB3LEAF01.yml        # Device-specific variables
│   ├── Leaf_generic.yml      # Common leaf router settings
│   ├── FABRIC-ALL.yml        # All fabric settings
│   ├── hostname_mapping.yml  # Hostname mappings
│   ├── 01_Check_Testbed/     # Testbed check macros
│   ├── 04_PPPOE/             # PPPOE-specific macros
│   │   ├── PPPOE_generic.yml
│   │   ├── ACCESS_generic_*.yml
│   │   └── pppoe_*.yml
│   ├── 05_L2TP/              # L2TP-specific macros
│   ├── 06_ISIS/              # IS-IS-specific macros
│   ├── 07_BGP/               # BGP-specific macros
│   └── modules/              # Function libraries
│       ├── PPPOE_Functions_generic.yml
│       ├── PPPOE_Functions_L2TP.yml
│       ├── PPPOE_Functions_PPPD.yml
│       ├── PPPOE_Functions_ACCOUNTING.yml
│       ├── PPPOE_Functions_LI.yml
│       ├── PYRAD_PFS_COMMANDS.yml
│       ├── STC.yml
│       └── Trex_Streams.yml
│
├── 📁 scripts/                # Helper scripts
│   ├── run_te4worker_dryrun.py  # Dry run test executor
│   ├── bdsparser.py            # Parse BDS data
│   ├── vlan-profile-ctl.py     # VLAN profile management
│   ├── update-rbfs.py          # Update RBFS
│   ├── blaster-stats.py        # BNG Blaster statistics
│   ├── icmp_monitor.py         # ICMP monitoring
│   └── rtbrick-tools/          # RtBrick utilities
│
├── 📁 template-data/          # Device configuration data
│   ├── FAB3LEAF01.yml
│   ├── FAB3LEAF02.yml
│   ├── BLR1LEAF01.yml
│   └── ...
│
├── 📁 template-engine/        # Template processing engine
│
├── 📁 tools/                  # Analysis & reporting tools
│   ├── jobscout              # Test result analysis
│   ├── jobscout-history      # Track result trends
│   ├── jobscout-diff         # Compare results
│   ├── rs2html.pl            # Generate HTML reports
│   └── README_JOBSCOUT.md    # Tool documentation
│
├── 📁 pyrad-pfs-config/       # RADIUS server configuration
│   └── pyrad-pfs-run99.yml
│
├── 📁 pyrad-pfs-data/         # RADIUS test data
│
├── 📁 grafana_dashboards/     # Monitoring dashboards
│   ├── create.py
│   ├── howto.md
│   └── rbfs-timeseries_clean.json
│
├── 📁 rsa_jkw_jwt_files/      # JWT authentication keys
│
├── 📁 hostdicts/              # Host dictionary files
│
├── 📁 business-templates/     # Business logic templates
│
├── 📁 borlab-files/           # BORLAB specific files
│
├── 📁 bngblaster-testcases/   # BNG Blaster test configs
│
├── 📁 bngblaster-api/         # BNG Blaster API
│
├── 📁 rtbrick-api/            # RtBrick API
│
├── 📁 fabric-api/             # Fabric API
│
└── 📁 tasteos-systemd/        # Systemd service files
```

---

## 🧪 Understanding the Test Modules

### Module Overview

| Module | Full Name | What It Tests | Example Tests |
|--------|-----------|---------------|---------------|
| **01** | Check Testbed | Is the lab ready? | Daemon status, interface status |
| **02** | Documentation | Config docs valid? | Configuration verification |
| **03** | L2x | Ethernet switching | Layer 2 connectivity |
| **04** | PPPoE | Home broadband login | Session setup, auth, IP assignment |
| **05** | L2TP | VPN tunnels | Tunnel setup, LAC/LNS |
| **06** | IS-IS | Internal routing | Adjacency, database, scalability |
| **07** | BGP | Internet routing | iBGP, eBGP, route learning |
| **08** | Multicast | IPTV/streaming | PIM, MVPN, IGMP |
| **09** | L2BSA | Bitstream Access | L2 wholesale access |
| **10** | Performance | Load testing | Throughput, scale |
| **11** | LI | Lawful Interception | Legal wiretapping |
| **12** | Maintenance | Upgrade procedures | Software upgrades |
| **13** | Reliability | Failover testing | Redundancy, recovery |
| **15** | DHCP | IP assignment | DHCPv4, DHCPv6 |
| **16** | NTP | Time sync | Clock synchronization |
| **17** | Security | Security features | ACLs, authentication |
| **18** | RD | Route Distinguisher | VRF routing |
| **19** | End2End | Integration | Full path testing |
| **20** | QoS | Traffic priority | Marking, shaping |
| **21** | P3OE | P3OE protocol | P3OE specific tests |
| **90** | Configure | Testbed setup | Initial configuration |

### Test Case Naming Convention

```
TC_04_001_000__pppoe_DS_Basic_PAP__BLR1LEAF01.yml
│  │   │   │      │     │    │         │
│  │   │   │      │     │    │         └── Device: BLR1LEAF01
│  │   │   │      │     │    └── Auth Type: PAP
│  │   │   │      │     └── Test Type: Basic
│  │   │   │      └── DS = Dual Stack (IPv4+IPv6)
│  │   │   └── Sub-number: 000
│  │   └── Category: 001
│  └── Module: 04 (PPPOE)
└── TC = Test Case
```

### Complete Test Categories - 04_PPPOE

| Category | What It Tests | Example |
|----------|--------------|---------|
| `001_xxx` | Basic Dual-Stack PAP | Basic connection with PAP auth |
| `002_xxx` | Dual-Stack CHAP | Connection with CHAP auth |
| `003_xxx` | IPv4-only PAP | IPv4 only, no IPv6 |
| `004_xxx` | IPv4-only CHAP | IPv4 only with CHAP |
| `007_xxx` | Access Reject PAP | Failed authentication |
| `008_xxx` | Access Reject CHAP | Failed CHAP auth |
| `011_xxx` | Idle timeout | Disconnect inactive users |
| `012_xxx` | Session timeout | Max connection time |
| `013_xxx` | Echo Request | Keepalive testing |
| `014_xxx` | IPCP Termination | Graceful disconnect |
| `015_xxx` | Duplicate IP | IP conflict handling |
| `016_xxx` | RADIUS accounting | Acct-Start/Stop |
| `017_xxx` | ADF filters | Traffic filtering |
| `018_xxx` | Ping tests | Connectivity |
| `019_xxx` | Residential QoS | Single-play QoS |
| `020_xxx` | Triple-play QoS | Voice+Video+Data |
| `021_xxx` | Autosensing | Auto-detect settings |
| `022_xxx` | CoA | Mid-session changes |
| `023_xxx` | Short cycle | Rapid connect/disconnect |
| `024_xxx` | QoS Accounting | QoS statistics |
| `025_xxx` | Bandwidth | Speed accuracy |
| `026_xxx` | Prefix list | Route filtering |
| `027_xxx` | RPF | Spoofing protection |
| `028_xxx` | Acct priority | Accounting order |
| `029_xxx` | H-Scheduling | Hierarchical QoS |
| `030_xxx` | Control plane | Control packet marking |
| `031_xxx` | MTU profile | Packet size handling |
| `032_xxx` | IP refuse | Reject client IP |
| `033_xxx` | Shaper | Rate limiting |
| `034_xxx` | Latency | Delay measurement |
| `035_xxx` | QoS CoA | QoS profile changes |
| `036_xxx` | H-Sched overbook | Overbooking tests |
| `037_xxx` | Multi-PON | Multiple PON tests |
| `038_xxx` | SRL changes | Service rate limit |
| `040_xxx` | NGTV/OTT | Video services |
| `050_xxx` | HTTP redirect | URL redirection |
| `051_xxx` | Wholebuy scale | Wholesale scaling |
| `052_xxx` | Wholebuy func | Wholesale functions |
| `082_xxx` | Fine shaping | Precise rate control |
| `999_xxx` | Utilities | Setup/cleanup |

### Complete Test Categories - 05_L2TP

| Category | What It Tests |
|----------|--------------|
| `001_xxx` | Basic session setup, address assignment |
| `004_xxx` | LNS session termination |
| `005_xxx` | LAC session/tunnel termination |
| `006_xxx` | Load balancing (2T, 5T, 16T, 32T) |
| `007_xxx` | Tunnel unavailability handling |
| `008_xxx` | Tunnel failover |
| `009_xxx` | Backup tunnels |
| `011_xxx` | LNS responds unknown IP |
| `013_xxx` | Accounting on tunnel failure |
| `014_xxx` | LNS not responding |
| `015_xxx` | Tunnel setup timing |
| `016_xxx` | DSL attributes handling |
| `017_xxx` | L2TP QoS (remarking, bandwidth) |
| `018_xxx` | CoA service change |
| `019_xxx` | Hierarchical scheduling |
| `020_xxx` | Bandwidth accuracy |
| `021_xxx` | PTA QoS scheduling |
| `022_xxx` | Control plane marking |
| `023_xxx` | Bandwidth attributes |
| `024_xxx` | Wholebuy scale |
| `025_xxx` | LCP priority |

### Complete Test Categories - 06_ISIS

| Category | What It Tests |
|----------|--------------|
| `001_xxx` | Database, ARP, Authentication |
| `002_xxx` | Scalability |
| `003_xxx` | BNG Blaster integration |
| `004_xxx` | Stability |
| `005_xxx` | Segment Routing (SR) |
| `006_xxx` | Adjacency Loss |

### Complete Test Categories - 07_BGP

| Category | What It Tests |
|----------|--------------|
| `001_xxx` | iBGP session, authentication, learning rate |
| `002_xxx` | eBGP session, authentication |
| `003_xxx` | BGP with BNG Blaster (1/2/3 FEEDs) |
| `004_xxx` | VRF MPLS labels |
| `005_xxx` | BGP protocols |
| `006_xxx` | BGP communities |
| `007_xxx` | L2VPN |
| `008_xxx` | Blackhole community |

### Complete Test Categories - 08_Multicast

| Category | What It Tests |
|----------|--------------|
| `001_xxx` | PIM neighbor, JOIN |
| `002_xxx` | MVPN AD, JOIN, STJ |

---

## 🔧 Key Components Deep Dive

### 1. A4Hosts.yml - The Device Inventory

This file contains **credentials and connection details** for ALL devices:

```yaml
# Connection method templates (defaults)
defaults:
  RTBssh:      # SSH to RtBrick device
    vendor: 'ubuntu'
    method: 'ssh'
    password: '...'
    port: 22
  
  RTBopsapi:   # REST API for operations
    method: 'restconf'
    contentType: 'application/json'
  
  RTBctrldapi: # Control daemon API
    method: 'restconf'
    contentType: 'application/json'
  
  BNGblasterctrl:  # BNG Blaster control
    method: 'rest'
    port: 8001

# Server definitions
TESTCLIENT-ssh: {device: '10.100.190.16', invoke: 'LINUXsshkey'}
GROOT-ssh:      {device: '10.100.190.16', invoke: 'LINUXsshkey'}
ROCKET-ssh:     {device: '10.100.190.192', invoke: 'LINUXsshkey'}
RADIUS-ssh:     {device: '217.89.29.22', invoke: 'LINUXsshkey'}

# Device definitions with multiple access methods
FAB3LEAF01-ssh:         {device: '10.100.190.133', invoke: 'PODssh'}
FAB3LEAF01-rtbssh:      {device: '10.100.190.133', invoke: 'RTBssh'}
FAB3LEAF01-cli2:        {device: '10.100.190.133', invoke: 'RTBsshcli2'}
FAB3LEAF01-opsapi:      {device: '10.100.190.133', port: 12321/..., invoke: 'RTBopsapi'}
FAB3LEAF01-pppoed:      {device: '10.100.190.133', port: 12321/..., invoke: 'RTBbdsapi'}
FAB3LEAF01-subscriberd: {device: '10.100.190.133', port: 12321/..., invoke: 'RTBbdsapi'}
FAB3LEAF01-l2tpd:       {device: '10.100.190.133', port: 12321/..., invoke: 'RTBbdsapi'}
FAB3LEAF01-bgpappd:     {device: '10.100.190.133', port: 12321/..., invoke: 'RTBbdsapi'}
```

**Connection Suffix Reference:**

| Suffix | Purpose | When to Use |
|--------|---------|-------------|
| `-ssh` | Basic SSH shell | General commands |
| `-rtbssh` | RtBrick SSH | Router commands |
| `-cli2` | RtBrick CLI v2 | Show commands |
| `-opsapi` | Operations API | Subscriber queries |
| `-bds` | BDS API | Table queries |
| `-pppoed` | PPPoE daemon | PPPoE specific |
| `-subscriberd` | Subscriber daemon | Session info |
| `-l2tpd` | L2TP daemon | Tunnel info |
| `-bgpappd` | BGP app daemon | BGP info |
| `-bgpiod` | BGP I/O daemon | BGP I/O |
| `-fibd` | FIB daemon | Forwarding info |
| `-ifmd` | Interface daemon | Interface info |
| `-confd` | Config daemon | Configuration |
| `-scp` | Secure copy | File transfer |
| `-sftp` | SFTP | File transfer |

### 2. Network Topology

```
                    ┌─────────────────────────────────────────┐
                    │            BACKBONE / INTERNET          │
                    └──────────────────┬──────────────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                    DA1 POP (Upstream)                   │
          │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐     │
          │  │RRA1  │  │RRB1  │  │RRA3  │  │SA2   │  │SB2   │     │
          │  │(Cisco)│  │(Cisco)│  │(Juniper)│(Cisco)│(Juniper)    │
          │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘     │
          └─────┼─────────┼─────────┼─────────┼─────────┼──────────┘
                │         │         │         │         │
    ════════════╪═════════╪═════════════════════════════╪═════════════
                │         │                             │
      ┌─────────┴─────────┴─────────────────────────────┴─────────┐
      │                      FABRIC LAYER                          │
      │                                                            │
      │  ┌──────────────────────────────────────────────────────┐ │
      │  │                     FAB3 (Fabric 3)                   │ │
      │  │  ┌─────────┐                         ┌─────────┐     │ │
      │  │  │ SPINE01 │◄───────────────────────►│ SPINE02 │     │ │
      │  │  │(RtBrick)│                         │(RtBrick)│     │ │
      │  │  └────┬────┘                         └────┬────┘     │ │
      │  │       │                                   │          │ │
      │  │  ┌────┴────────────┬─────────────────────┴────┐     │ │
      │  │  │                 │                          │     │ │
      │  │  ▼                 ▼                          ▼     │ │
      │  │ ┌───────┐      ┌───────┐               ┌───────┐   │ │
      │  │ │LEAF01 │      │LEAF02 │               │LEAF03 │   │ │
      │  │ │(BNG)  │      │(BNG)  │               │(BNG)  │   │ │
      │  │ └───┬───┘      └───┬───┘               └───┬───┘   │ │
      │  └─────┼──────────────┼───────────────────────┼───────┘ │
      │        │              │                       │         │
      │  ┌─────┴──────────────┴───────────────────────┴─────┐  │
      │  │              A10 SWITCHES / FANOUT               │  │
      │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐    │  │
      │  │  │A10SWITCH01│  │ FANOUT01  │  │A10SWITCH02│    │  │
      │  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘    │  │
      │  └────────┼──────────────┼──────────────┼──────────┘  │
      └───────────┼──────────────┼──────────────┼─────────────┘
                  │              │              │
    ══════════════╪══════════════╪══════════════╪══════════════════
                  │              │              │
      ┌───────────┴──────────────┴──────────────┴───────────┐
      │                    ACCESS LAYER                      │
      │  ┌─────────┐    ┌─────────┐    ┌─────────┐          │
      │  │   OLT   │    │  DSLAM  │    │ Switch  │          │
      │  └────┬────┘    └────┬────┘    └────┬────┘          │
      │       │              │              │                │
      │   ┌───┴───┐     ┌───┴───┐     ┌───┴───┐            │
      │   │  ONT  │     │ Modem │     │Router │            │
      │   └───┬───┘     └───┬───┘     └───┬───┘            │
      │       │              │              │                │
      │    🏠 Home        🏠 Home        🏠 Home             │
      └─────────────────────────────────────────────────────┘

      ┌─────────────────────────────────────────────────────┐
      │                    TEST SERVERS                      │
      │  ┌───────────┐  ┌───────────┐  ┌───────────┐       │
      │  │  GROOT    │  │  ROCKET   │  │  NEBULA   │       │
      │  │(TestClient)│ │(HeadNode) │  │(BNGBlaster)│      │
      │  │10.100.190.16│ │10.100.190.192│ │10.100.190.231│  │
      │  └───────────┘  └───────────┘  └───────────┘       │
      └─────────────────────────────────────────────────────┘
```

**Device Role Summary:**

| Device Type | Role | Software |
|-------------|------|----------|
| **SPINE** | Core routing | RtBrick RBFS |
| **LEAF** | BNG (subscriber termination) | RtBrick RBFS |
| **A10SWITCH** | Aggregation | RtBrick RBFS |
| **FANOUT** | Traffic distribution | Juniper |
| **RR** | Route Reflector | Cisco/Juniper |
| **GROOT** | Test client | Linux |
| **ROCKET** | Head node/PyRAD | Linux |
| **NEBULA/LOKI** | BNG Blaster | Linux |

### 3. Test Lab Locations

| Lab Code | Location | IP Range | Purpose |
|----------|----------|----------|---------|
| **BLR1** | Bangalore Pod 1 | 10.100.190.1xx | Dev/Test |
| **BLR2** | Bangalore Pod 2 | 10.100.190.1xx | Dev/Test |
| **BLR3** | Bangalore Pod 3 | 10.100.190.1xx | Dev/Test |
| **FAB1** | Fabric 1 | 10.100.190.11x | Test |
| **FAB2** | Fabric 2 | 10.100.190.12x | Test |
| **FAB3** | Fabric 3 | 10.100.190.13x | Primary Test |
| **FAB4** | Fabric 4 | 10.100.190.14x | Test |
| **TCT** | Test Center | 10.100.190.xx | Validation |
| **DA1** | Darmstadt | 10.100.18.xx | Upstream |
| **BORLAB** | BOR Lab | 10.100.190.23x | BOR Test |

---

## 🔄 How Tests Work

### Test Execution Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                         TEST EXECUTION                            │
├──────────────────────────────────────────────────────────────────┤
│  1. INITIALIZATION                                                │
│     ├─→ Load A4Hosts.yml (device credentials)                    │
│     ├─→ Load macro files (reusable functions)                    │
│     ├─→ Expand all #-MACRO- ... -# placeholders                  │
│     └─→ Establish connections to devices                          │
├──────────────────────────────────────────────────────────────────┤
│  2. SETUP PHASE                                                   │
│     ├─→ LEAF_INTERFACE_STATUS_CHECK (verify port is UP)          │
│     ├─→ LEAF_DEFAULT_ROUTE_CHECK (verify routing works)          │
│     ├─→ LEAF_GLOBAL_VLAN_PROFILE_SET (configure VLAN)            │
│     ├─→ CREATE_NAMESPACES (isolate test network)                 │
│     ├─→ START_CLIENT_TCP_DUMP (capture packets)                  │
│     └─→ START_PYRADIUS_RUN99_DUMP (capture RADIUS)               │
├──────────────────────────────────────────────────────────────────┤
│  3. EXECUTE PHASE                                                 │
│     ├─→ START_PPPD_CLIENTS (initiate PPPoE connection)           │
│     │     ├─→ PADI → PADO → PADR → PADS                          │
│     │     ├─→ LCP Negotiation                                    │
│     │     ├─→ Authentication (PAP/CHAP)                          │
│     │     ├─→ IPCP (get IPv4 address)                            │
│     │     └─→ IPv6CP (get IPv6 address)                          │
│     └─→ Session Established! ✅                                   │
├──────────────────────────────────────────────────────────────────┤
│  4. VERIFY PHASE                                                  │
│     ├─→ GET_SUBSCRIBER_ID (find session on router)               │
│     ├─→ Check MAC address matches                                │
│     ├─→ CHECK_IPv4_ADDR (verify IPv4)                            │
│     ├─→ CHECK_FRAMED_IPV6_ADDR (verify IPv6)                     │
│     ├─→ Additional test-specific checks                          │
│     └─→ Record results                                            │
├──────────────────────────────────────────────────────────────────┤
│  5. CLEANUP PHASE                                                 │
│     ├─→ STOP_PPPD_CLIENTS (disconnect)                           │
│     ├─→ LEAF_SUBSCRIBER_TERMINATION_AWAIT_FIXED (wait)           │
│     ├─→ STOP_CLIENT_TCP_DUMP (stop capture)                      │
│     ├─→ STOP_PYRADIUS_DUMP (stop RADIUS capture)                 │
│     ├─→ DELETE_NAMESPACES (cleanup isolation)                    │
│     ├─→ CLEANUP (reset configurations)                           │
│     └─→ LEAF_GLOBAL_VLAN_PROFILE_DELETE (remove VLAN)            │
└──────────────────────────────────────────────────────────────────┘
```

### PPPoE Session Flow (Detailed)

```
Test Client (PPPD)             LEAF (BNG)                  PyRAD PFS
     │                              │                          │
     │═══ PADI (Discovery) ════════►│                          │
     │   (I'm looking for a BNG)    │                          │
     │                              │                          │
     │◄══ PADO (Offer) ════════════│                          │
     │   (I can serve you)          │                          │
     │                              │                          │
     │═══ PADR (Request) ═════════►│                          │
     │   (Please give me a session) │                          │
     │                              │                          │
     │◄══ PADS (Session) ══════════│                          │
     │   (Session ID: 0x1234)       │                          │
     │                              │                          │
     │══════════════════════════════│══════════════════════════│
     │         PPP NEGOTIATION      │                          │
     │══════════════════════════════│══════════════════════════│
     │                              │                          │
     │◄── LCP Config-Request ──────│                          │
     │   (MRU=1492, Auth=CHAP)      │                          │
     │                              │                          │
     │─── LCP Config-Request ─────►│                          │
     │   (MRU=1492)                 │                          │
     │                              │                          │
     │◄── LCP Config-Ack ──────────│                          │
     │─── LCP Config-Ack ─────────►│                          │
     │                              │                          │
     │══════════════════════════════│══════════════════════════│
     │         AUTHENTICATION       │                          │
     │══════════════════════════════│══════════════════════════│
     │                              │                          │
     │◄── CHAP Challenge ──────────│                          │
     │   (Challenge: 0xABCD...)     │                          │
     │                              │                          │
     │─── CHAP Response ──────────►│                          │
     │   (Name: user, Response: hash)                          │
     │                              │                          │
     │                              │─── Access-Request ──────►│
     │                              │   (User: user, CHAP)     │
     │                              │                          │
     │                              │◄── Access-Accept ────────│
     │                              │   (IP: 1.2.3.4, IPv6:...)│
     │                              │                          │
     │◄── CHAP Success ────────────│                          │
     │                              │                          │
     │══════════════════════════════│══════════════════════════│
     │         IP ASSIGNMENT        │                          │
     │══════════════════════════════│══════════════════════════│
     │                              │                          │
     │◄── IPCP Config-Request ─────│                          │
     │   (Your IP: 1.2.3.4)         │                          │
     │                              │                          │
     │─── IPCP Config-Request ────►│                          │
     │   (Request IP: 0.0.0.0)      │                          │
     │                              │                          │
     │◄── IPCP Config-Nak ─────────│                          │
     │   (Use IP: 1.2.3.4)          │                          │
     │                              │                          │
     │─── IPCP Config-Request ────►│                          │
     │   (IP: 1.2.3.4)              │                          │
     │                              │                          │
     │◄── IPCP Config-Ack ─────────│                          │
     │─── IPCP Config-Ack ────────►│                          │
     │                              │                          │
     │══════════════════════════════│══════════════════════════│
     │       IPv6 ASSIGNMENT        │                          │
     │══════════════════════════════│══════════════════════════│
     │                              │                          │
     │◄── IPv6CP Config-Request ───│                          │
     │─── IPv6CP Config-Request ──►│                          │
     │◄── IPv6CP Config-Ack ───────│                          │
     │─── IPv6CP Config-Ack ──────►│                          │
     │                              │                          │
     │══════════════════════════════│══════════════════════════│
     │       SESSION ACTIVE! 🌐     │                          │
     │══════════════════════════════│══════════════════════════│
     │                              │                          │
     │                              │─── Acct-Start ──────────►│
     │                              │   (Session started)      │
```

---

## 🛠️ Tools & Dependencies

### Traffic Generators

| Tool | Type | Use Case |
|------|------|----------|
| **BNG Blaster** | Open-source | Scale testing (thousands of sessions) |
| **PPPD** | Linux daemon | Single session testing |
| **Spirent TestCenter** | Commercial | Precise traffic generation |
| **TRex** | Open-source | High-performance traffic |
| **iperf3** | Open-source | Bandwidth testing |

### RADIUS Server - PyRAD PFS

**Features:**
- Configurable Access-Accept/Reject responses
- IP pool management
- CoA (Change of Authorization) support
- Accounting message handling
- Dynamic attribute configuration

**Configuration Location:** `pyrad-pfs-config/`

### Monitoring Stack

| Tool | Purpose | Location |
|------|---------|----------|
| **Grafana** | Dashboards | `grafana_dashboards/` |
| **Graylog** | Log aggregation | External service |
| **tcpdump** | Packet capture | On test servers |
| **Wireshark** | Packet analysis | On test servers |

### Analysis Tools

| Tool | Location | Purpose |
|------|----------|---------|
| **jobscout** | `tools/jobscout` | Analyze single test |
| **jobscout-history** | `tools/jobscout-history` | Track trends |
| **jobscout-diff** | `tools/jobscout-diff` | Compare runs |
| **rs2html.pl** | `tools/rs2html.pl` | HTML reports |
| **macrofinder.pl** | `tools/macrofinder.pl` | Find macro usages |

### Server Inventory

| Server | IP | Role | Key Software |
|--------|-----|------|--------------|
| **GROOT** | 10.100.190.16 | Test Client | PPPD, tcpdump |
| **ROCKET** | 10.100.190.192 | Head Node | PyRAD, TestExecutor |
| **NEBULA** | 10.100.190.231 | BNG Blaster | bngblaster |
| **LOKI** | 10.100.190.235 | BNG Blaster | bngblaster |
| **DEADPOOL** | 10.100.190.230 | BORLAB | Various |
| **RADIUS** | 217.89.29.22 | Production RADIUS | FreeRADIUS |

### Required Packages

**Test Client (GROOT):**
```bash
# Core packages
python3.6+          # Scripting
ppp                 # PPP daemon
tcpdump             # Packet capture
wireshark           # Analysis
iperf3              # Bandwidth testing
zsh                 # Shell
```

**Head Node (ROCKET):**
```bash
# Core packages
python3.6+          # Scripting
git                 # Version control
redis-server        # Data store
docker              # Containers
wireshark           # Analysis

# RtBrick tools
rtbrick-ansible     # Configuration
rtbrick-ctrld       # Control daemon
rtbrick-imgstore    # Image storage

# Test tools
testExecutor        # Test framework
pyrad-pfs           # RADIUS server
stc-labserver       # Spirent control
```

---

## 🚀 Running Tests

### CI/CD Pipeline

```yaml
# .gitlab-ci.yml
default:
  tags: [rocket-docker]
  image: te-docker:0.0.10.5

stages:
  - test_runbooks

test:
  stage: test_runbooks
  script:
    - ./scripts/run_te4worker_dryrun.py --filter BLR
```

### Triggering Tests

**Method 1: Git Push**
```bash
# Run specific module
git push -o ci.variable=RUN_PIPELINE=04_PPPOE

# Run all tests
git push -o ci.variable=RUN_PIPELINE=all

# Convenient alias
git config alias.pci 'push -o ci.variable=RUN_PIPELINE=04_PPPOE'
git pci
```

**Method 2: GitLab Web**
1. Go to `CI/CD → Pipelines`
2. Click "Run Pipeline"
3. Add: `RUN_PIPELINE` = `04_PPPOE`
4. Click "Run Pipeline"

**Method 3: Manual (WebGUI)**
1. Connect VPN
2. Open TestExecutor (10.100.190.192:port)
3. Run setup: `TC_04_999_001__pppoe_pyrad_start__*.yml`
4. Run test
5. Run cleanup: `TC_04_999_002__pppoe_pyrad_clean__*.yml`

### Pipeline Stages

```
Pipeline
├── test_stage1 (parallel)
│   ├── TC_04_001_xxx tests
│   └── TC_04_002_xxx tests
├── test_stage2 (parallel)
│   ├── TC_04_003_xxx tests
│   └── TC_04_004_xxx tests
├── sequential_stage (one at a time)
│   └── TC_04_016_xxx tests (need config changes)
└── cleanup
    └── TC_04_999_002__pppoe_pyrad_clean
```

---

## 📚 Quick Reference

### Common Variables

| Variable | Example | Description |
|----------|---------|-------------|
| `OUTER_VLAN` | `400` | Service VLAN ID |
| `INNER_VLAN` | `7` | Customer VLAN |
| `MAC` | `00:0c:29:04:01:0` | MAC prefix |
| `BASE` | `0` | MAC suffix |
| `USER` | `siticom_test` | PPP username |
| `SUBS` | `1` | Subscriber count |
| `LEAF_IFP` | `ifp-0/1/4` | Router interface |
| `PHY_IF` | `enp0s10` | Client interface |

### Common Macros

| Macro | Purpose |
|-------|---------|
| `START_PPPD_CLIENTS` | Start PPPoE session |
| `STOP_PPPD_CLIENTS` | Stop session |
| `CLEANUP` | Reset configs |
| `LEAF_INTERFACE_STATUS_CHECK` | Verify port UP |
| `LEAF_DEFAULT_ROUTE_CHECK` | Verify routing |
| `CHECK_IPv4_ADDR` | Verify IPv4 |
| `CHECK_FRAMED_IPV6_ADDR` | Verify IPv6 |
| `CREATE_NAMESPACES` | Create isolation |
| `DELETE_NAMESPACES` | Remove isolation |
| `GET_SUBSCRIBER_ID` | Get session ID |

### Interface Naming

```
ifp-0/1/4     Physical interface (slot/subslot/port)
ifl-0/1/4/7   Logical interface (slot/subslot/port/unit)
lo-0/0/0/0    Loopback interface
```

### RtBrick Daemons

| Daemon | Purpose |
|--------|---------|
| pppoed | PPPoE handling |
| subscriberd | Session management |
| l2tpd | L2TP handling |
| bgpappd | BGP application |
| fibd | Forwarding tables |
| ifmd | Interface management |
| confd | Configuration |
| opsd | Operations |

---

## 🔍 Troubleshooting Guide

### Common Issues

#### Session Not Establishing
```bash
# Check router logs
cli -1 -n show log table pppoed.pppoed.1.log | tail -n 100

# Check RADIUS
show radius server pyrad-pfs-run99
```

#### IP Not Assigned
```bash
# Check subscriber
GET subscribers/{id}

# Check RADIUS response (look for Framed-IP-Address)
```

#### Test Timeout
- Check network connectivity
- Verify device is responding
- Check correct device name

### Debug Commands

```bash
# Router CLI
show subscriber session
show interface brief
show radius server
show log table subscriberd.subscriberd.1.log

# API
GET /subscribers/
POST /bds/table/walk {"table":{"table_name":"local.pppoe.ppp.sessions"}}
```

---

## 🎓 Summary

**te-runbooks** is a comprehensive network testing framework that:

✅ Simulates subscribers via PPPoE/L2TP  
✅ Tests authentication, IP assignment, QoS  
✅ Uses macros for maintainable tests  
✅ Supports multiple test labs  
✅ Runs automated CI/CD testing  
✅ Generates detailed reports  

---

*Document: TE_RUNBOOKS_GUIDE.md*  
*Version: 2.0 - Comprehensive with Networking Basics*  
*Generated: December 11, 2025*  
*Author: GitHub Copilot*
