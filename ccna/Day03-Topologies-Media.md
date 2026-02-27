# Day 3: Network Topologies & Media Types

## 🎯 What You'll Learn
Physical and logical network layouts, cable types and standards, connector types, and when to use each. This is the hands-on, physical-world knowledge you need.

---

## Network Topologies

A topology is the arrangement of devices and connections in a network. There's a **physical** topology (how cables are laid out) and a **logical** topology (how data flows).

### Star Topology ⭐ (Most Common)
```
        [PC1]
          |
[PC2]---[SWITCH]---[PC3]
          |
        [PC4]
```
- Every device connects to a central switch/hub
- **Pros:** Easy to add devices, failure of one link doesn't affect others, easy to troubleshoot
- **Cons:** Central device is a single point of failure; if the switch dies, everyone goes down
- **Used in:** Almost every modern LAN

### Mesh Topology
```
Full Mesh:              Partial Mesh:
[A]----[B]              [A]----[B]
|\ /\ /|               |      |
| X  X |               |      |
|/ \/ \|               [C]----[D]
[C]----[D]
```
- **Full mesh:** Every device connects to every other device
  - Links needed: n(n-1)/2 (e.g., 4 devices = 6 links)
  - Maximum redundancy but very expensive
- **Partial mesh:** Some devices interconnected, not all
- **Used in:** WAN connections between sites, data center cores

### Bus Topology (Legacy)
```
[PC1]---[PC2]---[PC3]---[PC4]
         |
    (backbone cable)
```
- All devices share one cable (backbone/trunk)
- Terminators at each end prevent signal reflection
- **Cons:** One cable break = entire network down; collision-prone
- **Used in:** Old 10BASE2/10BASE5 Ethernet (obsolete)

### Ring Topology
```
[PC1] ──→ [PC2]
  ↑          ↓
[PC4] ←── [PC3]
```
- Data travels in one direction around the ring
- Token passing: only the device with the "token" can transmit
- **Dual ring** (FDDI): Two rings in opposite directions for redundancy
- **Used in:** Legacy Token Ring, FDDI, some SONET/SDH rings

### Hybrid Topology
- Combination of topologies (e.g., star-bus, star-ring)
- Most real networks are hybrids
- Example: Star topology within each floor, mesh between floors

---

## Cable Types — The Physical Layer

### Copper Cabling (Twisted Pair)

**Why twisted?** The twisting reduces electromagnetic interference (EMI) and crosstalk between wire pairs.

**UTP (Unshielded Twisted Pair)** — Most common in LANs:
| Category | Max Speed | Max Distance | Bandwidth | Use Case |
|----------|-----------|-------------|-----------|----------|
| Cat 3 | 10 Mbps | 100m | 16 MHz | Old phone/10BASE-T (obsolete) |
| Cat 5 | 100 Mbps | 100m | 100 MHz | Legacy (don't install new) |
| Cat 5e | 1 Gbps | 100m | 100 MHz | Still widely used |
| Cat 6 | 1 Gbps (10G to 55m) | 100m | 250 MHz | Current standard |
| Cat 6a | 10 Gbps | 100m | 500 MHz | High-performance LANs |
| Cat 7 | 10 Gbps | 100m | 600 MHz | Shielded; data centers |
| Cat 8 | 25/40 Gbps | 30m | 2000 MHz | Data centers, short runs |

> 💡 **Exam tip:** Know Cat 5e (1 Gbps) and Cat 6a (10 Gbps) — these are the most commonly tested.

**STP (Shielded Twisted Pair):**
- Has a metallic shield around the wire pairs
- Better EMI protection
- More expensive, harder to install
- Used in high-interference environments (factories, near heavy machinery)

**Cable Wiring Standards:**

**T-568A:**
```
Pin 1: White-Green     Pin 5: White-Blue
Pin 2: Green           Pin 6: Orange
Pin 3: White-Orange    Pin 7: White-Brown
Pin 4: Blue            Pin 8: Brown
```

**T-568B (more common in the US):**
```
Pin 1: White-Orange    Pin 5: White-Blue
Pin 2: Orange          Pin 6: Green
Pin 3: White-Green     Pin 7: White-Brown
Pin 4: Blue            Pin 8: Brown
```

> 💡 **The difference:** T-568A and T-568B swap the orange and green pairs (pins 1-2 and 3-6).

**Cable Types and When to Use Each:**

| Cable Type | Both Ends | Connects | Example |
|-----------|-----------|----------|---------|
| **Straight-through** | Same standard (B-B or A-A) | Unlike devices | PC→Switch, Router→Switch |
| **Crossover** | Different standards (A-B) | Like devices | Switch→Switch, PC→PC, Router→Router |
| **Rollover (Console)** | Reversed pinout | PC→Console port | Laptop→Router/Switch console |

**Modern note:** Most newer switches and routers support **Auto-MDIX**, which automatically detects the cable type and adjusts. But the CCNA exam still tests the traditional rules!

```
When to use what (traditional rules):
  PC ↔ Switch:     Straight-through
  PC ↔ Router:     Crossover
  PC ↔ PC:         Crossover
  Switch ↔ Switch: Crossover
  Switch ↔ Router: Straight-through
  Router ↔ Router: Crossover

Easy rule: 
  Same type of device → Crossover
  Different type → Straight-through
  Exception: PC and Router are both "DTE" devices, so they're "same type"
```

---

### Fiber Optic Cabling

Uses light instead of electrical signals. Immune to EMI. Much longer distances.

**Two Types:**

| Feature | Single-Mode (SMF) | Multi-Mode (MMF) |
|---------|-------------------|-------------------|
| Core diameter | 8-10 μm | 50 or 62.5 μm |
| Light source | Laser | LED or VCSEL |
| Distance | Up to 80+ km | Up to 550m (OM3) / 2km (OM5) |
| Cost | More expensive | Less expensive |
| Jacket color | Yellow | Orange (OM1/OM2) or Aqua (OM3/OM4/OM5) |
| Use case | Long haul, WAN, campus backbone | Short runs, within buildings |

**Fiber Connectors:**
| Connector | Description | Common Use |
|-----------|-------------|-----------|
| LC (Lucent) | Small form factor, push-pull | Most common today |
| SC (Subscriber) | Square, push-pull | Older installations |
| ST (Straight Tip) | Round, bayonet twist-lock | Legacy |
| MTRJ | Small, RJ-45-like form factor | Duplex connections |
| MPO/MTP | Multi-fiber (12-24 fibers) | High-density data centers |

> 💡 **Exam tip:** LC is the most common modern connector. SC is the second most common. Know the colors (yellow = single-mode, orange/aqua = multi-mode).

**Fiber vs Copper:**
| Factor | Copper (UTP) | Fiber Optic |
|--------|-------------|-------------|
| Max distance | 100m | 80+ km (SMF) |
| Speed | Up to 10 Gbps (Cat 6a) | Up to 400 Gbps |
| EMI immunity | Susceptible | Immune |
| Cost | Low | Higher |
| Installation | Easy (RJ-45 crimping) | Requires splicing/polishing |
| Security | Can be tapped | Very difficult to tap |
| Power | Can carry PoE | Cannot carry power |

---

### Coaxial Cable (Legacy)
- Single copper conductor surrounded by insulation and shielding
- Used in: Cable internet (DOCSIS), old 10BASE2/10BASE5 Ethernet
- Connectors: BNC (old Ethernet), F-type (cable TV/internet)
- Not tested heavily on CCNA; just know it exists

---

## PoE (Power over Ethernet)

Delivers electrical power along with data over standard UTP cables. Powers devices like IP phones, wireless APs, and IP cameras without separate power cables.

| Standard | IEEE | Power at Source | Power at Device | Pairs Used |
|----------|------|----------------|----------------|------------|
| PoE | 802.3af | 15.4W | 12.95W | 2 pairs |
| PoE+ | 802.3at | 30W | 25.5W | 2 pairs |
| UPoE (PoE++) | 802.3bt Type 3 | 60W | 51W | 4 pairs |
| UPoE (PoE++) | 802.3bt Type 4 | 100W | 71.3W | 4 pairs |

**Key concepts:**
- **PSE (Power Sourcing Equipment):** The switch that provides power
- **PD (Powered Device):** The device receiving power (AP, IP phone, camera)
- Switch detects if device supports PoE before sending power (prevents damage)

---

## Wireless Media

**Frequency Bands:**
- **2.4 GHz:** Longer range, better wall penetration, more interference (microwaves, Bluetooth), fewer non-overlapping channels (1, 6, 11)
- **5 GHz:** Shorter range, less interference, more channels, higher speeds
- **6 GHz:** (Wi-Fi 6E) Even more channels, least interference, shortest range

**Key wireless concepts for CCNA:**
- **SSID:** Network name
- **BSS:** Basic Service Set (one AP + its clients)
- **ESS:** Extended Service Set (multiple APs with same SSID for roaming)
- **BSSID:** The AP's MAC address
- **Channel overlap:** In 2.4 GHz, only use channels 1, 6, 11 to avoid overlap

---

## Practice Questions

1. How many links are needed for a full mesh of 5 devices?
2. What cable type connects a switch to a router?
3. What's the max distance for Cat 6a UTP at 10 Gbps?
4. What color jacket indicates single-mode fiber?
5. Which connector is most common in modern fiber installations?
6. A Cat 5e cable is rated for what maximum speed?
7. What is the purpose of twisting wire pairs in UTP cable?
8. What does Auto-MDIX do?
9. In 2.4 GHz Wi-Fi, which three channels are non-overlapping?
10. What IEEE standard provides up to 30W of PoE?

<details>
<summary>Answers</summary>

1. 10 links: n(n-1)/2 = 5(4)/2 = 10
2. Straight-through (unlike devices)
3. 100 meters
4. Yellow
5. LC (Lucent Connector)
6. 1 Gbps (1000BASE-T)
7. To reduce electromagnetic interference (EMI) and crosstalk
8. Automatically detects cable type and adjusts, so straight-through or crossover both work
9. Channels 1, 6, and 11
10. 802.3at (PoE+)
</details>

---

## Key Takeaways

1. **Star topology** dominates modern LANs — know its pros and cons
2. **Cable types:** Straight-through (unlike devices), Crossover (like devices), Rollover (console)
3. **Cat 5e = 1 Gbps / 100m**, **Cat 6a = 10 Gbps / 100m** — most tested
4. **Single-mode fiber = long distance (yellow)**, **Multi-mode = short distance (orange/aqua)**
5. **LC connector** is the modern standard for fiber
6. **PoE** delivers power over Ethernet — know 802.3af (15.4W) and 802.3at (30W)

---

*← [Day 2 — TCP/IP Model](Day02-TCPIP-Model.md) | [Day 4 — IPv4 Addressing](Day04-IPv4-Addressing.md) →*
