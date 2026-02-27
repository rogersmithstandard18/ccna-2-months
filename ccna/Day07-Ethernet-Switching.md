# Day 7: Ethernet Fundamentals & Switching Basics

## 🎯 What You'll Learn
How Ethernet works at Layer 2, the Ethernet frame structure, how switches learn and forward frames, and MAC address fundamentals.

---

## Ethernet — The King of LANs

Ethernet (IEEE 802.3) dominates local area networking. Everything from your home network to data centers runs on Ethernet.

### Ethernet Standards You Need to Know

| Standard | Name | Speed | Media | Max Distance |
|----------|------|-------|-------|-------------|
| 802.3 | 10BASE-T | 10 Mbps | Cat 3+ UTP | 100m |
| 802.3u | 100BASE-TX | 100 Mbps | Cat 5+ UTP | 100m |
| 802.3ab | 1000BASE-T | 1 Gbps | Cat 5e+ UTP | 100m |
| 802.3z | 1000BASE-SX | 1 Gbps | Multi-mode fiber | 550m |
| 802.3z | 1000BASE-LX | 1 Gbps | Single-mode fiber | 5 km |
| 802.3an | 10GBASE-T | 10 Gbps | Cat 6a+ UTP | 100m |
| 802.3ae | 10GBASE-SR | 10 Gbps | Multi-mode fiber | 400m |
| 802.3ae | 10GBASE-LR | 10 Gbps | Single-mode fiber | 10 km |

**Naming convention decoded:**
```
1000BASE-T
│    │    │
│    │    └── T = Twisted pair (copper)
│    │        SX = Short wavelength (multi-mode fiber)
│    │        LX = Long wavelength (single-mode fiber)
│    └── BASE = Baseband (full bandwidth used for one signal)
└── Speed in Mbps (1000 = 1 Gbps)
```

---

## The Ethernet Frame

Every piece of data on a LAN travels inside an Ethernet frame:

```
┌──────────┬─────┬──────────┬──────────┬────────────┬────────────────┬──────┐
│ Preamble │ SFD │ Dest MAC │ Src MAC  │ Type/Len   │     Data       │  FCS │
│  7 bytes │ 1B  │ 6 bytes  │ 6 bytes  │  2 bytes   │  46-1500 bytes │ 4B   │
└──────────┴─────┴──────────┴──────────┴────────────┴────────────────┴──────┘
```

| Field | Size | Purpose |
|-------|------|---------|
| **Preamble** | 7 bytes | Alternating 1s and 0s for clock synchronization |
| **SFD** (Start Frame Delimiter) | 1 byte | Signals "the real frame starts now" (10101011) |
| **Destination MAC** | 6 bytes | Who this frame is for |
| **Source MAC** | 6 bytes | Who sent this frame |
| **Type/Length** | 2 bytes | EtherType (e.g., 0x0800 = IPv4, 0x0806 = ARP, 0x86DD = IPv6) |
| **Data (Payload)** | 46–1500 bytes | The actual data (Layer 3 packet) |
| **FCS** (Frame Check Sequence) | 4 bytes | CRC error detection |

**Frame sizes:**
- **Minimum frame:** 64 bytes (header + minimum 46-byte payload + FCS)
- **Maximum frame:** 1518 bytes (or 1522 with 802.1Q VLAN tag)
- **Runt:** Frame < 64 bytes → discarded (usually collision damage)
- **Giant:** Frame > 1518 bytes → discarded (unless jumbo frames enabled)
- **Jumbo frame:** Up to 9216 bytes (used in data centers; not standard)

> 💡 **Why minimum 46 bytes of data?** If the payload is smaller, padding (zeros) is added. This ensures the frame is at least 64 bytes, which is necessary for CSMA/CD collision detection to work properly.

---

## MAC Addresses

**Format:** 48 bits, displayed as 12 hex characters
```
Representations (all equivalent):
  AA:BB:CC:DD:EE:FF     (colons — most common)
  AA-BB-CC-DD-EE-FF     (hyphens — Windows style)
  AABB.CCDD.EEFF        (dots — Cisco style)

Structure:
  AA:BB:CC : DD:EE:FF
  └─ OUI ─┘ └─ Device ┘

OUI = Organizationally Unique Identifier (vendor ID)
  - Assigned by IEEE to manufacturers
  - Example: 00:1A:A1 = Cisco
  
Device = Unique serial assigned by the manufacturer
```

**Special MAC addresses:**
| Address | Purpose |
|---------|---------|
| FF:FF:FF:FF:FF:FF | Broadcast — sent to all devices on the LAN |
| 01:00:5E:xx:xx:xx | IPv4 multicast |
| 33:33:xx:xx:xx:xx | IPv6 multicast |
| 01:80:C2:00:00:00 | STP (Spanning Tree Protocol) |

**Unicast, Broadcast, Multicast:**
- **Unicast:** One-to-one (specific destination MAC)
- **Broadcast:** One-to-all (FF:FF:FF:FF:FF:FF)
- **Multicast:** One-to-many (specific group; interested devices join)

> 💡 **Exam tip:** Bit 0 of the first byte determines unicast vs multicast:
> - 0 = Unicast (even first hex digit: 0,2,4,6,8,A,C,E)
> - 1 = Multicast (odd first hex digit: 1,3,5,7,9,B,D,F)

---

## How Switches Work

A switch is a Layer 2 device that forwards frames based on MAC addresses. It's smarter than a hub (which just repeats everything everywhere).

### The MAC Address Table

Every switch maintains a **MAC address table** (also called CAM table):

| MAC Address | Port | VLAN | Type | Age |
|------------|------|------|------|-----|
| AAAA.BBBB.CCCC | Fa0/1 | 1 | Dynamic | 5 min |
| DDDD.EEEE.FFFF | Fa0/2 | 1 | Dynamic | 2 min |

### Three Switch Operations

**1. LEARN — Build the MAC table**
```
Frame arrives on port Fa0/1 with source MAC AAAA.BBBB.CCCC

Switch: "I'll remember that AAAA.BBBB.CCCC is reachable through Fa0/1"
        → Adds to MAC address table (or refreshes the timer if already there)
```

**2. FORWARD — Known unicast**
```
Frame arrives for destination MAC DDDD.EEEE.FFFF

Switch checks MAC table:
  "DDDD.EEEE.FFFF is on Fa0/2"
  → Sends frame ONLY out Fa0/2 (efficient!)
```

**3. FLOOD — Unknown unicast or broadcast**
```
Frame arrives for destination MAC that's NOT in the MAC table

Switch: "I don't know where this MAC is"
        → Sends frame out ALL ports EXCEPT the port it came in on

Also floods for:
  - Broadcast frames (dest FF:FF:FF:FF:FF:FF)
  - Multicast frames (unless IGMP snooping is configured)
```

**4. FILTER — Never forward back to source**
```
A frame is NEVER sent back out the port it arrived on.
```

### Step-by-Step Example

```
Initial MAC table: empty

     [PC-A]          [PC-B]          [PC-C]
    MAC: AAAA       MAC: BBBB       MAC: CCCC
       |               |               |
     Fa0/1           Fa0/2           Fa0/3
    ┌──┴───────────────┴───────────────┴──┐
    │              SWITCH                  │
    └──────────────────────────────────────┘

Step 1: PC-A sends frame to PC-B (dst: BBBB)
  - Switch LEARNS: AAAA on Fa0/1
  - Switch doesn't know BBBB → FLOODS out Fa0/2 and Fa0/3
  - PC-B receives it ✅, PC-C ignores it (wrong dest MAC)

Step 2: PC-B replies to PC-A (dst: AAAA)
  - Switch LEARNS: BBBB on Fa0/2
  - Switch knows AAAA is on Fa0/1 → FORWARDS only to Fa0/1
  - PC-C never sees this frame ✅

MAC table now:
  AAAA → Fa0/1
  BBBB → Fa0/2
```

### MAC Address Table Aging

- Dynamic entries age out after **300 seconds (5 minutes)** by default
- Each time a frame is received from that MAC, the timer resets
- Static entries (manually configured) never age out

```
show mac address-table              ! View the table
show mac address-table dynamic      ! Only dynamic entries
show mac address-table address AAAA.BBBB.CCCC   ! Specific MAC
clear mac address-table dynamic     ! Clear all dynamic entries
```

---

## CSMA/CD — Collision Handling (Half-Duplex)

**CSMA/CD = Carrier Sense Multiple Access with Collision Detection**

Used on half-duplex Ethernet (hubs, old shared media):

```
1. LISTEN (Carrier Sense): Is anyone else transmitting?
2. If quiet → TRANSMIT
3. If busy → WAIT, then try again
4. If COLLISION detected during transmission:
   a. Stop transmitting
   b. Send JAM signal (tells everyone a collision happened)
   c. Wait a RANDOM time (backoff algorithm)
   d. Try again from step 1
```

**Modern switches use full-duplex** — no collisions possible because send and receive use separate wire pairs. CSMA/CD is effectively disabled on full-duplex links.

**Collision domain:** The set of devices that can cause collisions with each other.
- Hub: All ports are ONE collision domain
- Switch: Each port is its own collision domain (collisions isolated)

**Broadcast domain:** The set of devices that receive each other's broadcasts.
- Switch (without VLANs): All ports are ONE broadcast domain
- Router: Each interface is a separate broadcast domain
- VLANs: Each VLAN is a separate broadcast domain

---

## Duplex and Speed

| Setting | Description |
|---------|-------------|
| Half-duplex | Can send OR receive, not both simultaneously (uses CSMA/CD) |
| Full-duplex | Can send AND receive simultaneously (no collisions) |
| Auto-negotiation | Devices negotiate the best common speed and duplex |

**Duplex mismatch** — a common issue:
```
If one side is full-duplex and the other is half-duplex:
  - The half-duplex side detects "collisions" (the full-duplex side sends while it's sending)
  - Late collisions, FCS errors, runts appear
  - Performance is terrible
  - This is a VERY common troubleshooting scenario on the exam!
```

**Configuration:**
```
SW1(config)# interface Fa0/1
SW1(config-if)# speed 100              ! Force 100 Mbps
SW1(config-if)# duplex full            ! Force full-duplex

! Or let it auto-negotiate (recommended)
SW1(config-if)# speed auto
SW1(config-if)# duplex auto
```

**Verification:**
```
show interfaces fa0/1          ! Detailed: speed, duplex, errors, counters
show interfaces status         ! Quick overview of all ports
```

**Interface error counters to know:**
| Counter | Meaning |
|---------|---------|
| CRC errors | Frame failed integrity check (bad cable, EMI, duplex mismatch) |
| Runts | Frames < 64 bytes (collisions, bad NIC) |
| Giants | Frames > 1518 bytes |
| Late collisions | Collision after first 64 bytes — almost always duplex mismatch |
| Input errors | Total of all input-side errors |
| Output errors | Frames that couldn't be sent |

---

## Practice Questions

1. What is the minimum Ethernet frame size, and why?
2. A switch receives a frame with destination FF:FF:FF:FF:FF:FF. What does it do?
3. How long do dynamic MAC table entries last by default?
4. What is the EtherType value for IPv4?
5. What happens when a switch receives a frame for a MAC not in its table?
6. What is a duplex mismatch, and what symptoms does it cause?
7. How many collision domains does an 8-port switch create?
8. What's the difference between a collision domain and a broadcast domain?

<details>
<summary>Answers</summary>

1. 64 bytes. Required for CSMA/CD collision detection — the frame must be long enough that a collision is detected before transmission completes.
2. Floods it out all ports except the one it arrived on (broadcast).
3. 300 seconds (5 minutes).
4. 0x0800
5. Floods the frame out all ports except the ingress port (unknown unicast flooding).
6. One side is full-duplex, the other half-duplex. Causes late collisions, CRC errors, poor performance.
7. 8 collision domains (one per port). But still 1 broadcast domain (without VLANs).
8. Collision domain = devices that can collide (Layer 1 boundary). Broadcast domain = devices that receive broadcasts (Layer 2/3 boundary — separated by routers or VLANs).
</details>

---

## Key Takeaways

1. **Switches learn source MAC, forward/flood based on destination MAC**
2. **Unknown unicast and broadcast = flood; known unicast = forward**
3. **Full-duplex eliminates collisions** — always use it when possible
4. **Duplex mismatch = late collisions** — classic troubleshooting scenario
5. **Each switch port = 1 collision domain; all ports = 1 broadcast domain (without VLANs)**
6. **MAC table entries expire after 5 minutes** by default

---

*← [Day 6 — VLSM & Summarization](Day06-Subnetting-Part2-VLSM.md) | [Day 8-10 — Device Configuration](Day08-10-Device-Configuration.md) →*
