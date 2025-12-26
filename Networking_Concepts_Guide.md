# Networking Concepts Guide for Beginners

## A Complete Beginner's Guide to Understanding Network Architecture

---

## Table of Contents

**Part 1: Network Architecture Basics**
1. [What is a Network?](#1-what-is-a-network)
2. [Traditional vs Modern Network Design](#2-traditional-vs-modern)
3. [Spine-Leaf Architecture Explained](#3-spine-leaf)

**Part 2: Routing Protocols**
4. [What is Routing?](#4-what-is-routing)
5. [ISIS Protocol Explained](#5-isis)
6. [BGP Protocol Explained](#6-bgp)

**Part 3: Advanced Concepts**
7. [MPLS - Label Switching](#7-mpls)
8. [EVPN - Ethernet VPN](#8-evpn)
9. [L2VPN and Pseudowires](#9-l2vpn)

**Part 4: Putting It All Together**
10. [How BLR Testbed Uses These Technologies](#10-blr-testbed)

---

# Part 1: Network Architecture Basics

---

## 1. What is a Network? <a name="1-what-is-a-network"></a>

### The Simplest Explanation

A **network** is just a bunch of devices connected together so they can talk to each other.

```
Simple Network Example:

    Computer A ────────────── Computer B
                   │
                   │
              Computer C

All three computers can send messages to each other!
```

### Network Devices

| Device | What It Does | Real-World Analogy |
|--------|-------------|-------------------|
| **Switch** | Connects devices in the same building/area | Post office in your neighborhood |
| **Router** | Connects different networks together | Highway connecting cities |
| **Server** | Provides services (web, email, etc.) | A store that sells things |
| **Firewall** | Protects network from threats | Security guard at the entrance |

### Why Do We Need Switches?

Without switches, you'd need a cable from every device to every other device:

```
Without Switch (Mess!):          With Switch (Clean!):
                                 
   A ────── B                         A
   │╲      ╱│                         │
   │ ╲    ╱ │                    ┌────┴────┐
   │  ╲  ╱  │                    │         │
   │   ╲╱   │                B───│  SWITCH │───C
   │   ╱╲   │                    │         │
   │  ╱  ╲  │                    └────┬────┘
   │ ╱    ╲ │                         │
   C ────── D                         D

  6 cables needed!              Only 4 cables needed!
```

---

## 2. Traditional vs Modern Network Design <a name="2-traditional-vs-modern"></a>

### Traditional 3-Tier Architecture (Old Way)

In the past, networks were built with 3 layers:

```
Traditional 3-Tier Network:

                    ┌─────────────┐
                    │    CORE     │  ◄── Fastest, most expensive
                    │   (Layer 3) │      Connects everything together
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
    │ DISTRIBUTION│ │ DISTRIBUTION│ │ DISTRIBUTION│  ◄── Middle layer
    │  (Layer 2)  │ │  (Layer 2)  │ │  (Layer 2)  │      Aggregates traffic
    └──────┬──────┘ └──────┬──────┘ └──────┴──────┘
           │               │               │
    ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
    │   ACCESS    │ │   ACCESS    │ │   ACCESS    │  ◄── Where devices connect
    │  (Layer 1)  │ │  (Layer 1)  │ │  (Layer 1)  │      Cheapest switches
    └─────────────┘ └─────────────┘ └─────────────┘
          │               │               │
       Servers         Servers         Servers
```

### Problems with Traditional Design

1. **Traffic Bottleneck**: All traffic must go UP and DOWN through layers
2. **Not Flexible**: Hard to add new devices
3. **Single Points of Failure**: If distribution switch dies, many servers disconnected
4. **East-West Traffic**: Modern apps need servers to talk to each other (horizontally)

```
Problem: East-West Traffic

Server A wants to talk to Server B:

    Server A                              Server B
        │                                     │
        ▼                                     ▼
    [Access 1]                           [Access 2]
        │                                     │
        ▼                                     ▼
    [Distribution 1]                    [Distribution 2]
        │                                     │
        └──────────►[ CORE ]◄─────────────────┘
                       │
                       │
           Traffic goes all the way UP 
           then all the way DOWN!
           (Very inefficient)
```

---

## 3. Spine-Leaf Architecture Explained <a name="3-spine-leaf"></a>

### What is Spine-Leaf?

**Spine-Leaf** is a modern network design with only 2 layers:
- **Spine**: The "backbone" - connects all leaves together
- **Leaf**: Where devices (servers, customers) actually connect

```
Spine-Leaf Architecture:

         SPINE LAYER (Backbone)
    ┌──────────────────────────────────────┐
    │                                      │
    │   ┌─────────┐         ┌─────────┐   │
    │   │ SPINE 1 │─────────│ SPINE 2 │   │
    │   └────┬────┘         └────┬────┘   │
    │        │╲                 ╱│        │
    │        │ ╲               ╱ │        │
    │        │  ╲             ╱  │        │
    │        │   ╲           ╱   │        │
    │        │    ╲         ╱    │        │
    │        │     ╲       ╱     │        │
    └────────┼──────╲─────╱──────┼────────┘
             │       ╲   ╱       │
             │        ╲ ╱        │
         LEAF LAYER    ╳    (Access)
             │        ╱ ╲        │
    ┌────────┼───────╱───╲───────┼────────┐
    │        │      ╱     ╲      │        │
    │   ┌────┴────┐╱       ╲┌────┴────┐   │
    │   │ LEAF 1  │─────────│ LEAF 2  │   │
    │   └────┬────┘         └────┬────┘   │
    │        │                   │        │
    └────────┼───────────────────┼────────┘
             │                   │
         Customers           Customers
         Servers             Servers
```

### Key Rule: Every Leaf Connects to Every Spine

This is the magic of spine-leaf:

```
Full Mesh Connectivity:

    SPINE 1 ═══════════════════ SPINE 2
       ║ ╲                     ╱ ║
       ║  ╲                   ╱  ║
       ║   ╲                 ╱   ║
       ║    ╲               ╱    ║
       ║     ╲             ╱     ║
       ║      ╲           ╱      ║
       ║       ╲         ╱       ║
    LEAF 1 ═════╳═══════╳═════ LEAF 2 ═════ LEAF 3
               ╱ ╲     ╱ ╲
       Every leaf connects to every spine!
```

### Why Spine-Leaf is Better

| Feature | Traditional 3-Tier | Spine-Leaf |
|---------|-------------------|------------|
| **Latency** | Variable (depends on path) | Predictable (always 2 hops) |
| **Bandwidth** | Bottleneck at distribution | Evenly distributed |
| **Scaling** | Complex | Easy - just add more leaves/spines |
| **Redundancy** | Limited | Excellent - multiple paths |
| **East-West Traffic** | Poor | Excellent |

### How Traffic Flows in Spine-Leaf

```
Server A (on LEAF 1) wants to talk to Server B (on LEAF 2):

    Server A                              Server B
        │                                     ▲
        ▼                                     │
    ┌───────┐                             ┌───────┐
    │LEAF 1 │                             │LEAF 2 │
    └───┬───┘                             └───▲───┘
        │                                     │
        │         ┌─────────┐                 │
        └────────►│ SPINE 1 │─────────────────┘
                  └─────────┘

    Only 2 HOPS! (Leaf → Spine → Leaf)
    
    If SPINE 1 fails, traffic automatically goes through SPINE 2!
```

### BLR Testbed Spine-Leaf Example

```
BLR1 POD (Spine-Leaf):

                ┌─────────────┐         ┌─────────────┐
                │ BLR1SPINE01 │─────────│ BLR1SPINE02 │
                │  (91_7KDA)  │         │  (91_7KDB)  │
                └──────┬──────┘         └──────┬──────┘
                       │╲                     ╱│
                       │ ╲                   ╱ │
                       │  ╲                 ╱  │
                       │   ╲               ╱   │
                       │    ╲             ╱    │
                       │     ╲           ╱     │
                       │      ╲         ╱      │
                ┌──────┴───────╲───────╱───────┴──────┐
                │               ╲     ╱               │
                │           ┌────╲───╱────┐           │
                │           │ BLR1LEAF01  │           │
                │           │  (91_7KE0)  │           │
                │           └──────┬──────┘           │
                │                  │                  │
                │              Customers              │
                └─────────────────────────────────────┘

    BLR1LEAF01 connects to BOTH spines for redundancy!
```

---

## Spine vs Leaf - Quick Comparison

| Aspect | SPINE | LEAF |
|--------|-------|------|
| **Position** | Top layer (backbone) | Bottom layer (access) |
| **Connects to** | Other spines + all leaves | Spines + customers/servers |
| **Purpose** | Forward traffic between leaves | Provide customer access |
| **Customer Ports** | NO | YES |
| **Number in Network** | Few (2-4 typically) | Many (can be dozens) |
| **Traffic Type** | Transit only | Ingress/Egress |

```
Simple Memory Aid:

    SPINE = "Spine" of the network (backbone, holds everything together)
    LEAF  = "Leaves" where customers connect (like leaves on a tree)

            🌳 Think of a Tree!
            
                   SPINE
                  ╱     ╲
                 ╱       ╲
              LEAF       LEAF
             ╱    ╲     ╱    ╲
           🖥️    🖥️  🖥️    🖥️
         Servers/Customers
```

---

*Continue to Part 2: Routing Protocols (ISIS & BGP)...*

---

# Part 2: Routing Protocols

---

## 4. What is Routing? <a name="4-what-is-routing"></a>

### The Basic Question

When a packet needs to go from A to B, how does the network know which path to take?

**Answer: Routing!**

```
Routing = Finding the best path from source to destination

Example: Packet needs to go from LEAF 1 to LEAF 2

                    ┌─────────┐
            ┌──────►│ SPINE 1 │──────┐    Path 1: LEAF1 → SPINE1 → LEAF2
            │       └─────────┘      │
            │                        ▼
    ┌───────┴───┐               ┌───────────┐
    │   LEAF 1  │               │   LEAF 2  │
    └───────┬───┘               └───────────┘
            │       ┌─────────┐      ▲
            └──────►│ SPINE 2 │──────┘    Path 2: LEAF1 → SPINE2 → LEAF2
                    └─────────┘

    Which path to use? Routing protocols decide!
```

### Two Types of Routing

| Type | Description | Used For |
|------|-------------|----------|
| **IGP** (Interior Gateway Protocol) | Routes WITHIN a network | Inside your company/lab |
| **EGP** (Exterior Gateway Protocol) | Routes BETWEEN networks | Connecting to internet/other companies |

```
IGP vs EGP:

┌─────────────────────────────────────┐     ┌─────────────────────────────────────┐
│         YOUR NETWORK                │     │        ANOTHER NETWORK              │
│                                     │     │                                     │
│   IGP runs here                     │     │   IGP runs here                     │
│   (ISIS or OSPF)                    │     │   (ISIS or OSPF)                    │
│                                     │ EGP │                                     │
│         [Router 1]──────────────────┼─────┼──────────────[Router 2]             │
│             │                       │(BGP)│                   │                 │
│             │                       │     │                   │                 │
│         [Switch]                    │     │               [Switch]              │
│                                     │     │                                     │
└─────────────────────────────────────┘     └─────────────────────────────────────┘

IGP (ISIS) = Used INSIDE your network
EGP (BGP)  = Used BETWEEN different networks
```

---

## 5. ISIS Protocol Explained <a name="5-isis"></a>

### What is ISIS?

**ISIS** = Intermediate System to Intermediate System

It's a protocol that helps routers/switches inside a network:
1. **Discover** each other
2. **Learn** the network topology (who connects to whom)
3. **Calculate** the best paths to all destinations

### How ISIS Works - Simple Explanation

Think of ISIS like a group chat where everyone shares their connections:

```
Step 1: Each device says "Hello, I exist!"

    SPINE 1: "Hello everyone!"
    SPINE 2: "Hello everyone!"  
    LEAF 1:  "Hello everyone!"
    LEAF 2:  "Hello everyone!"

Step 2: Each device shares who it's connected to

    SPINE 1: "I'm connected to: SPINE 2, LEAF 1, LEAF 2"
    SPINE 2: "I'm connected to: SPINE 1, LEAF 1, LEAF 2"
    LEAF 1:  "I'm connected to: SPINE 1, SPINE 2"
    LEAF 2:  "I'm connected to: SPINE 1, SPINE 2"

Step 3: Everyone builds a map of the network

    Now every device knows the COMPLETE topology!
    They can calculate the shortest path to anywhere.
```

### ISIS in Picture Form

```
ISIS Network Discovery:

    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │   SPINE 1 ──────────────────────────── SPINE 2              │
    │      │ ╲                              ╱ │                   │
    │      │  ╲   "I know the whole       ╱  │                   │
    │      │   ╲   network map!"         ╱   │                   │
    │      │    ╲                       ╱    │                   │
    │      │     ╲                     ╱     │                   │
    │   LEAF 1 ───────────────────────── LEAF 2                  │
    │      │                              │                       │
    │      │   "I also know the whole     │                       │
    │      │    network map!"             │                       │
    │                                                             │
    │   Each device has IDENTICAL view of the network            │
    │   This is called "Link State Database" (LSDB)              │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

### ISIS Key Concepts

| Term | Meaning | Simple Explanation |
|------|---------|-------------------|
| **LSP** | Link State PDU (Packet) | "Here's my list of neighbors" message |
| **LSDB** | Link State Database | The map of the entire network |
| **SPF** | Shortest Path First | Algorithm to find best path |
| **Adjacency** | Two devices are connected and talking ISIS | "We're friends!" |
| **Area** | Group of devices | Like departments in a company |

### ISIS Metric (Cost)

ISIS uses **metric** (cost) to decide best path:

```
Lower metric = Better path

Example:
                        metric=10
    LEAF 1 ─────────────────────────────────── SPINE 1
        │                                         │
        │ metric=100                    metric=10 │
        │                                         │
        └──────────────── SPINE 2 ────────────────┘
                        metric=10

    Path 1: LEAF 1 → SPINE 1           Total cost = 10 ✓ WINNER!
    Path 2: LEAF 1 → SPINE 2 → SPINE 1 Total cost = 100 + 10 = 110
```

### Why BLR Uses ISIS

1. **Fast**: Converges quickly when links fail
2. **Scalable**: Works well in large networks
3. **Simple**: Easy to configure
4. **Protocol Independent**: Works with IPv4 and IPv6

---

## 6. BGP Protocol Explained <a name="6-bgp"></a>

### What is BGP?

**BGP** = Border Gateway Protocol

BGP is the routing protocol that connects different networks together. It's called the "glue of the internet" because it connects all the different companies/organizations.

### When to Use BGP vs ISIS

```
Think of it this way:

    ISIS = Directions INSIDE your house
           "To get from bedroom to kitchen, go through hallway"

    BGP  = Directions to OTHER houses
           "To get to John's house, take Main Street, then turn on Oak Avenue"
```

### BGP Types

| Type | Full Name | Used For |
|------|-----------|----------|
| **eBGP** | External BGP | Between different organizations (AS) |
| **iBGP** | Internal BGP | Within the same organization |

```
eBGP vs iBGP:

┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│      AS 64512 (Your Network)    │       │      AS 65000 (ISP Network)     │
│                                 │       │                                 │
│   [Router A]────iBGP────[Router B]──eBGP──[Router C]────iBGP────[Router D]
│                                 │       │                                 │
│   iBGP = same AS number         │       │   iBGP = same AS number         │
│                                 │       │                                 │
└─────────────────────────────────┘       └─────────────────────────────────┘
                                    │
                                    └── eBGP = different AS numbers
```

### AS Number (Autonomous System)

Every network has a unique **AS Number** (like a phone number for networks):

```
AS Number Examples:

    AS 3320  = Deutsche Telekom
    AS 64512 = BLR Testbed (private)
    AS 7922  = Comcast
    AS 15169 = Google

    When networks talk via BGP, they identify each other by AS number.
```

### How BGP Works

BGP doesn't just find paths - it chooses the BEST path based on many factors:

```
BGP Path Selection (Simplified):

    "I have 3 paths to reach Google (AS 15169)"

    Path 1: AS64512 → AS3320 → AS15169      (2 hops)
    Path 2: AS64512 → AS7922 → AS1234 → AS15169  (3 hops)
    Path 3: AS64512 → AS9999 → AS15169      (2 hops, but AS9999 is slow)

    BGP chooses based on:
    1. Shortest AS path? (fewer hops = better)
    2. Customer-preferred route?
    3. Link quality?
    4. Many other factors...
```

### BGP Address Families

BGP can carry different types of routing information:

| Address Family | What It Carries |
|---------------|-----------------|
| **IPv4 Unicast** | Normal IPv4 routes |
| **IPv6 Unicast** | Normal IPv6 routes |
| **IPv4 VPN** | VPN routes (L3VPN) |
| **IPv6 VPN** | VPN routes for IPv6 |
| **EVPN** | Ethernet VPN (L2VPN) |

```
BGP is like a truck that can carry different cargo:

    ┌─────────────────────────────────────┐
    │            BGP "Truck"              │
    │  ┌─────────┬─────────┬─────────┐   │
    │  │  IPv4   │  IPv6   │  EVPN   │   │
    │  │ Routes  │ Routes  │ Routes  │   │
    │  └─────────┴─────────┴─────────┘   │
    └─────────────────────────────────────┘

    Same protocol (BGP), carrying different types of routes!
```

### BGP in BLR Testbed

```
BLR Testbed BGP Neighbors:

    BLR1LEAF01 has iBGP sessions with:
    ├── BLR1SPINE01 (192.168.0.11)
    └── BLR1SPINE02 (192.168.0.12)

    What they exchange:
    ├── IPv6 unicast routes
    ├── IPv6 labeled unicast
    ├── IPv4 VPN routes  ◄── Used for L3VPN
    ├── IPv6 VPN routes
    └── IPv4 VPN multicast
```

### ISIS vs BGP Summary

| Feature | ISIS | BGP |
|---------|------|-----|
| **Type** | IGP (Interior) | EGP (Exterior) |
| **Scope** | Inside one network | Between networks |
| **What it learns** | Link topology | Reachable prefixes |
| **Speed** | Very fast convergence | Slower, but more stable |
| **Complexity** | Simple | Complex (many attributes) |
| **Use in BLR** | Path finding inside POD | VPN routes, external connectivity |

---

# Part 3: Advanced Concepts

---

## 7. MPLS - Label Switching <a name="7-mpls"></a>

### What is MPLS?

**MPLS** = Multi-Protocol Label Switching

Instead of looking at the full destination address, MPLS uses short **labels** (numbers) to forward packets. Think of it like airport baggage tags!

### The Baggage Tag Analogy

```
Without MPLS (IP Routing):
    Every router reads the FULL destination address
    Like every airport worker reading your full home address!

    ┌─────────────────────────────────┐
    │ Destination: Mr. John Smith    │
    │ 123 Main Street, Apartment 4B  │  ◄── Every router reads this
    │ Springfield, IL 62701          │      whole address!
    │ United States of America       │
    └─────────────────────────────────┘


With MPLS (Label Switching):
    Each router just reads a simple number
    Like baggage tags at airport!

    ┌───────────┐
    │ Label: 42 │  ◄── Much faster to read!
    └───────────┘

    Router sees "42" → Forward to next-hop → Done!
```

### How MPLS Labels Work

```
MPLS Label Journey:

    Source                                              Destination
       │                                                     │
       ▼                                                     ▼
    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
    │ LEAF │───►│SPINE │───►│SPINE │───►│SPINE │───►│ LEAF │
    │  1   │    │  1   │    │  2   │    │  3   │    │  2   │
    └──────┘    └──────┘    └──────┘    └──────┘    └──────┘
        │           │           │           │           │
        │           │           │           │           │
    [Packet]   [Label     [Label      [Label      [Packet]
    (no label)  added]     swapped]   removed]   (no label)
                 │           │           │
                 ▼           ▼           ▼
              Label=42   Label=58    Label=73
    
    PUSH        SWAP        SWAP        POP
    (add label) (change)   (change)   (remove label)
```

### Label Operations

| Operation | What Happens | When Used |
|-----------|-------------|-----------|
| **PUSH** | Add a new label | Packet enters MPLS network |
| **SWAP** | Replace label with new one | Transit through network |
| **POP** | Remove the label | Packet exits MPLS network |

### Why MPLS is Useful

1. **Speed**: Looking up a small number is faster than a long IP address
2. **Traffic Engineering**: Can force traffic to take specific paths
3. **VPNs**: Labels can identify which VPN a packet belongs to
4. **Service Separation**: Different customers can share the same network

### MPLS in BLR Testbed

```
BLR Testbed MPLS Example:

    Customer traffic from BLR1LEAF01 to BLR2LEAF01:

    BLR1LEAF01                                         BLR2LEAF01
        │                                                   │
        │ Original Packet: "Data for customer ABC"          │
        ▼                                                   ▼
    ┌────────────────────────────────────────────────────────────┐
    │ PUSH Label 1000 (VPN label for customer ABC)              │
    │ PUSH Label 2000 (Transport label to reach BLR2)           │
    └────────────────────────────────────────────────────────────┘
        │
        ▼
    ┌─────────────┐
    │ BLR1SPINE01 │  SWAP Label 2000 → 2001
    └─────────────┘
        │
        ▼
    ┌─────────────┐
    │ BLR2SPINE01 │  POP Label 2001 (transport done)
    └─────────────┘
        │
        ▼
    ┌─────────────┐
    │ BLR2LEAF01  │  POP Label 1000 (VPN label)
    │             │  Deliver to customer ABC
    └─────────────┘
```

---

## 8. EVPN - Ethernet VPN <a name="8-evpn"></a>

### What is EVPN?

**EVPN** = Ethernet Virtual Private Network

EVPN lets you extend Layer 2 (Ethernet) connectivity across a Layer 3 (IP/MPLS) network. It's like making two remote locations appear to be on the same local network!

### The Problem EVPN Solves

```
Without EVPN:
    Two offices can't be on the same Ethernet network
    
    Office A (New York)          Office B (London)
    ┌────────────────┐           ┌────────────────┐
    │ 192.168.1.0/24 │    ✗      │ 192.168.1.0/24 │
    │                │ Can't     │                │
    │ PC: 192.168.1.5│ directly  │ PC: 192.168.1.8│
    └────────────────┘ talk!     └────────────────┘


With EVPN:
    Both offices appear to be on the SAME network!
    
    Office A (New York)          Office B (London)
    ┌────────────────┐           ┌────────────────┐
    │ 192.168.1.0/24 │    ✓      │ 192.168.1.0/24 │
    │                │ Same      │                │
    │ PC: 192.168.1.5│ network!  │ PC: 192.168.1.8│
    └───────┬────────┘           └───────┬────────┘
            │                            │
            │      ┌──────────────┐      │
            └──────│  IP/MPLS     │──────┘
                   │  Network     │
                   │  (Internet)  │
                   └──────────────┘
            
            EVPN creates a "virtual bridge" across the internet!
```

### How EVPN Works

EVPN uses BGP to share information about:
1. **MAC addresses** - Where each device's hardware address is located
2. **IP addresses** - Which IP is at which location
3. **VPN membership** - Which sites belong to which VPN

```
EVPN Learning Process:

    1. PC at Office A (MAC: aa:bb:cc:11:22:33) sends a packet
    
    2. LEAF at Office A learns the MAC and tells BGP:
       "MAC aa:bb:cc:11:22:33 is reachable through me!"
    
    3. BGP distributes this to LEAF at Office B
    
    4. LEAF at Office B now knows:
       "To reach aa:bb:cc:11:22:33, send traffic to Office A's LEAF"
    
    ┌──────────────┐                      ┌──────────────┐
    │  LEAF A      │     BGP Update       │  LEAF B      │
    │              │─────────────────────►│              │
    │  "I have     │  "MAC aa:bb:cc..."   │  "Learned!   │
    │  this MAC"   │                      │   Now I know │
    │              │                      │   where it is"│
    └──────────────┘                      └──────────────┘
```

### EVPN Route Types

| Type | Name | What It Carries |
|------|------|-----------------|
| **Type 1** | Ethernet Auto-Discovery | Multi-homing information |
| **Type 2** | MAC/IP Advertisement | MAC and IP addresses |
| **Type 3** | Inclusive Multicast | BUM traffic handling |
| **Type 4** | Ethernet Segment | Multi-homing sync |
| **Type 5** | IP Prefix | Layer 3 routes |

```
Most Common: Type 2 (MAC/IP Advertisement)

    ┌─────────────────────────────────────────────────┐
    │  EVPN Type 2 Route                              │
    │                                                 │
    │  MAC Address: aa:bb:cc:11:22:33                │
    │  IP Address:  192.168.1.5                      │
    │  VPN ID:      41383 (Route Target)             │
    │  Next-Hop:    62.225.21.146 (LEAF's loopback)  │
    │                                                 │
    └─────────────────────────────────────────────────┘
    
    This tells other LEAFs: "To reach this MAC/IP, 
    send MPLS traffic to 62.225.21.146"
```

### Route Target (RT) - VPN Identifier

**Route Target** is like a mailing list for VPN routes:

```
Route Target Example:

    VPN for Customer ABC: RT 41383
    
    LEAF A (exports RT 41383):
        "Here are routes for VPN 41383"
        
    LEAF B (imports RT 41383):
        "I want routes for VPN 41383"
        
    LEAF C (doesn't import 41383):
        "I don't care about VPN 41383" (ignores the routes)


    ┌──────────┐     RT 41383      ┌──────────┐
    │  LEAF A  │─────────────────►│  LEAF B  │  ✓ Accepts
    │ (export) │                   │ (import) │
    └──────────┘                   └──────────┘
         │
         │        RT 41383      ┌──────────┐
         └──────────────────────│  LEAF C  │  ✗ Ignores
                                │(no import)│
                                └──────────┘
```

---

## 9. L2VPN and Pseudowires <a name="9-l2vpn"></a>

### What is L2VPN?

**L2VPN** = Layer 2 Virtual Private Network

L2VPN provides Layer 2 (Ethernet) connectivity between two or more sites over a service provider's network.

### Types of L2VPN

| Type | Full Name | Topology | Use Case |
|------|-----------|----------|----------|
| **VPWS** | Virtual Private Wire Service | Point-to-Point | Two offices |
| **VPLS** | Virtual Private LAN Service | Multipoint | Many offices |
| **EVPN** | Ethernet VPN | Multipoint | Modern replacement for VPLS |

### Pseudowire - The Virtual Cable

A **pseudowire** is a point-to-point L2VPN. Think of it as a virtual cable connecting two locations:

```
Pseudowire Concept:

    Physical Reality:
    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │   Office A                                     Office B     │
    │   ┌──────┐      Complex Network      ┌──────┐              │
    │   │ PC A │─────[Many routers/]───────│ PC B │              │
    │   └──────┘      [switches]           └──────┘              │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘

    What Customer Sees (Virtual):
    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │   Office A                                     Office B     │
    │   ┌──────┐                                   ┌──────┐      │
    │   │ PC A │═══════════════════════════════════│ PC B │      │
    │   └──────┘         "Virtual Wire"            └──────┘      │
    │                    (Pseudowire)                             │
    │                                                             │
    │   Looks like a direct cable connection!                    │
    └─────────────────────────────────────────────────────────────┘
```

### EVPN-VPWS (What BLR Testbed Uses)

**EVPN-VPWS** = EVPN + VPWS combined

It uses EVPN's BGP control plane with VPWS's point-to-point data plane:

```
EVPN-VPWS in BLR Testbed:

    BLR1LEAF01                                      BLR2LEAF01
    ┌────────────────┐                        ┌────────────────┐
    │                │                        │                │
    │  Site ID: 2314 │◄══════════════════════►│  Site ID: 2315 │
    │                │    EVPN-VPWS           │                │
    │  RT: 41383     │    Pseudowire          │  RT: 41383     │
    │                │                        │                │
    └───────┬────────┘                        └───────┬────────┘
            │                                         │
            │ ifp-0/1/11                     ifp-0/1/11 │
            │                                         │
        Customer A                               Customer A
        (same customer                           (other office)
         one office)
```

### Site ID - Endpoint Identifier

Each end of a pseudowire has a **Site ID**:

```
Site ID Matching Rules:

    BLR1LEAF01                     BLR2LEAF01
    ┌─────────────────┐            ┌─────────────────┐
    │ Local Site: 2314│◄══════════►│ Local Site: 2315│
    │Remote Site: 2315│            │Remote Site: 2314│
    └─────────────────┘            └─────────────────┘
    
    RULE: Local Site of A = Remote Site of B
          Local Site of B = Remote Site of A
    
    If these don't match, pseudowire won't come up!
```

### How Traffic Flows Through Pseudowire

```
Traffic Flow Example:

    Customer A at BLR1 sends a packet to Customer A at BLR2:

    Step 1: Packet enters BLR1LEAF01
    ┌────────────────────────────────────────┐
    │ Original: [Ethernet Frame from PC]    │
    └────────────────────────────────────────┘
    
    Step 2: BLR1LEAF01 adds MPLS labels
    ┌────────────────────────────────────────┐
    │ [VPN Label][Transport Label][Ethernet] │
    └────────────────────────────────────────┘
    
    Step 3: Packet travels through spines (MPLS)
    
    Step 4: BLR2LEAF01 removes labels
    ┌────────────────────────────────────────┐
    │ Original: [Ethernet Frame from PC]    │
    └────────────────────────────────────────┘
    
    Step 5: Packet delivered to Customer A at BLR2
    
    The customer's frame is UNCHANGED end-to-end!
```

### Attachment Circuit

The **Attachment Circuit (AC)** is the physical interface where customer traffic enters/exits:

```
Attachment Circuit:

    Customer                    Provider Network
    ┌──────┐                   ┌─────────────────────────────┐
    │  PC  │                   │                             │
    │      │                   │     BLR1LEAF01              │
    │      │═══════════════════│══► ifp-0/1/11 ──► Pseudowire │
    │      │    Attachment     │    (This is the             │
    │      │     Circuit       │     Attachment Circuit)     │
    └──────┘                   │                             │
                               └─────────────────────────────┘
    
    AC = The "door" where customer traffic enters the VPN
```

---

# Part 4: Putting It All Together

---

## 10. How BLR Testbed Uses These Technologies <a name="10-blr-testbed"></a>

### Complete Technology Stack

```
BLR Testbed Technology Layers:

    ┌─────────────────────────────────────────────────────────────┐
    │  Layer 5: SERVICES                                         │
    │  ├── PPPoE (Subscriber access)                             │
    │  ├── L2VPN/Pseudowire (Business connectivity)              │
    │  └── L3VPN (IP VPN services)                               │
    ├─────────────────────────────────────────────────────────────┤
    │  Layer 4: VPN CONTROL                                      │
    │  └── EVPN (via BGP)                                        │
    │      ├── Route Target matching                             │
    │      ├── MAC/IP learning                                   │
    │      └── Site ID for pseudowires                           │
    ├─────────────────────────────────────────────────────────────┤
    │  Layer 3: TRANSPORT                                        │
    │  └── MPLS                                                  │
    │      ├── Label switching                                   │
    │      └── Traffic engineering                               │
    ├─────────────────────────────────────────────────────────────┤
    │  Layer 2: ROUTING                                          │
    │  ├── ISIS (Interior routing - path calculation)            │
    │  └── BGP (VPN routes + external connectivity)              │
    ├─────────────────────────────────────────────────────────────┤
    │  Layer 1: PHYSICAL                                         │
    │  └── Spine-Leaf Architecture                               │
    │      ├── Spines (backbone)                                 │
    │      └── Leaves (access)                                   │
    └─────────────────────────────────────────────────────────────┘
```

### How a Pseudowire Test Works in BLR

```
Complete L2VPN Test Flow:

    ┌──────────────────────────────────────────────────────────────────┐
    │  STEP 1: Configuration                                          │
    │                                                                  │
    │  BLR1LEAF01:                    BLR2LEAF01:                     │
    │  - Create EVPL instance         - Create EVPL instance          │
    │  - RT: 41383                    - RT: 41383                     │
    │  - LocalSite: 2314              - LocalSite: 2315               │
    │  - RemoteSite: 2315             - RemoteSite: 2314              │
    │  - Interface: ifp-0/1/11        - Interface: ifp-0/1/11         │
    └──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
    ┌──────────────────────────────────────────────────────────────────┐
    │  STEP 2: BGP/EVPN Signaling                                     │
    │                                                                  │
    │  BLR1LEAF01 ─────── BGP Update ──────► BLR1SPINE01             │
    │     │                                      │                    │
    │     │ "I have EVPN route for              │                    │
    │     │  RT 41383, Site 2314"               │                    │
    │     │                                      ▼                    │
    │     │                              BLR1SPINE01 forwards         │
    │     │                              to BLR2SPINE01               │
    │     │                                      │                    │
    │     │                                      ▼                    │
    │     │                              BLR2LEAF01 receives:        │
    │     │                              "Site 2314 exists!"          │
    │     │                                      │                    │
    │     ◄──────────────────────────────────────┘                    │
    │     BLR2LEAF01 sends back:                                      │
    │     "Site 2315 exists!"                                         │
    │                                                                  │
    │  RESULT: Pseudowire established! (Site 2314 ←→ Site 2315)      │
    └──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
    ┌──────────────────────────────────────────────────────────────────┐
    │  STEP 3: Traffic Test                                           │
    │                                                                  │
    │  Headnode:                                                      │
    │  ├── BBL sends packets on eno2.1230 (VLAN 1230)                │
    │  │       │                                                      │
    │  │       ▼                                                      │
    │  │   NCS-5501 switch                                           │
    │  │       │                                                      │
    │  │       ▼                                                      │
    │  │   BLR1LEAF01 (ifp-0/1/11)                                   │
    │  │       │                                                      │
    │  │       │ [Add MPLS labels]                                   │
    │  │       ▼                                                      │
    │  │   BLR1SPINE01                                               │
    │  │       │                                                      │
    │  │       │ [MPLS forwarding]                                   │
    │  │       ▼                                                      │
    │  │   BLR2SPINE01                                               │
    │  │       │                                                      │
    │  │       │ [Remove MPLS labels]                                │
    │  │       ▼                                                      │
    │  │   BLR2LEAF01 (ifp-0/1/11)                                   │
    │  │       │                                                      │
    │  │       ▼                                                      │
    │  │   NCS-5501 switch                                           │
    │  │       │                                                      │
    │  │       ▼                                                      │
    │  └── BBL receives packets on eno2.2258 (VLAN 2258)             │
    │                                                                  │
    │  SUCCESS: Traffic traversed the pseudowire!                     │
    └──────────────────────────────────────────────────────────────────┘
```

### Summary: What Each Technology Does

| Technology | Role in BLR Testbed |
|------------|---------------------|
| **Spine-Leaf** | Physical architecture - spines connect leaves |
| **ISIS** | Finds paths between devices inside the network |
| **BGP** | Carries VPN routes (EVPN) between leaves |
| **MPLS** | Provides fast label-based forwarding |
| **EVPN** | Controls which sites belong to which VPN |
| **Pseudowire** | Creates virtual point-to-point L2 connection |

---

## Quick Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════════╗
║                     NETWORKING CONCEPTS CHEAT SHEET                      ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  SPINE-LEAF:                                                             ║
║  ├── Spine = Backbone (connects leaves)                                  ║
║  ├── Leaf = Access (connects customers)                                  ║
║  └── Every leaf connects to every spine                                  ║
║                                                                          ║
║  ROUTING PROTOCOLS:                                                      ║
║  ├── ISIS = Interior routing (inside network)                            ║
║  └── BGP = Exterior routing + VPN routes (between networks)              ║
║                                                                          ║
║  MPLS:                                                                   ║
║  ├── PUSH = Add label (entering MPLS)                                    ║
║  ├── SWAP = Change label (transit)                                       ║
║  └── POP = Remove label (exiting MPLS)                                   ║
║                                                                          ║
║  EVPN:                                                                   ║
║  ├── Extends Layer 2 over Layer 3 network                                ║
║  ├── Route Target (RT) = VPN identifier                                  ║
║  └── Uses BGP to distribute MAC/IP info                                  ║
║                                                                          ║
║  PSEUDOWIRE (L2VPN):                                                     ║
║  ├── Virtual point-to-point L2 connection                                ║
║  ├── Site ID = Endpoint identifier                                       ║
║  └── Local of A = Remote of B                                            ║
║                                                                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

## Glossary (Complete)

| Term | Full Name | Simple Explanation |
|------|-----------|-------------------|
| **AS** | Autonomous System | A network with its own routing policy |
| **BGP** | Border Gateway Protocol | Routes between different networks |
| **eBGP** | External BGP | BGP between different AS |
| **iBGP** | Internal BGP | BGP within same AS |
| **EVPN** | Ethernet VPN | L2 connectivity over L3 network |
| **IGP** | Interior Gateway Protocol | Routing inside a network (ISIS, OSPF) |
| **ISIS** | Intermediate System to IS | Interior routing protocol |
| **L2** | Layer 2 | Ethernet/MAC address layer |
| **L3** | Layer 3 | IP address layer |
| **L2VPN** | Layer 2 VPN | Ethernet connectivity as a service |
| **L3VPN** | Layer 3 VPN | IP routing as a service |
| **LSDB** | Link State Database | Network topology map |
| **LSP** | Link State PDU | ISIS hello/update packet |
| **MAC** | Media Access Control | Hardware address (aa:bb:cc:11:22:33) |
| **MPLS** | Multi-Protocol Label Switching | Label-based forwarding |
| **POP** | - | Remove MPLS label |
| **PUSH** | - | Add MPLS label |
| **RT** | Route Target | VPN identifier in BGP |
| **SPF** | Shortest Path First | Algorithm to find best path |
| **SWAP** | - | Change MPLS label |
| **VPLS** | Virtual Private LAN Service | Multipoint L2VPN |
| **VPWS** | Virtual Private Wire Service | Point-to-point L2VPN |

---

*Document created: 2025*
*For use with BLR Testbed learning*
