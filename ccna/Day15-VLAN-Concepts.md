# Day 15: VLAN Concepts — Why and How Networks Are Segmented

## 🎯 What You'll Learn
What VLANs are, why they exist, the types of VLANs, and how they transform a flat network into a segmented, secure, and efficient one.

---

## The Problem: One Big Broadcast Domain

Without VLANs, every port on a switch is in the **same broadcast domain**:

```
┌─────────────────────────────────────────────┐
│                  SWITCH                      │
│  Fa0/1  Fa0/2  Fa0/3  Fa0/4  Fa0/5  Fa0/6 │
└──┬───────┬──────┬──────┬──────┬──────┬──────┘
   │       │      │      │      │      │
 [Sales] [Sales] [Eng] [Eng] [Mgmt] [Mgmt]

ALL devices share ONE broadcast domain.
When Sales-PC sends a broadcast → EVERYONE hears it.
```

**Problems with this:**
- 🔊 **Noise:** Every broadcast hits every device (ARP, DHCP, etc.)
- 🔓 **Security:** Engineering can sniff Sales traffic; Management traffic is exposed
- 🐌 **Performance:** More devices = more broadcasts = more wasted bandwidth
- 🔧 **Management:** No logical separation; physical moves require cable changes

---

## The Solution: VLANs

A **VLAN (Virtual Local Area Network)** is a logical grouping of switch ports into separate broadcast domains — **regardless of physical location**.

```
┌─────────────────────────────────────────────┐
│                  SWITCH                      │
│  Fa0/1  Fa0/2  Fa0/3  Fa0/4  Fa0/5  Fa0/6 │
│  VLAN10 VLAN10 VLAN20 VLAN20 VLAN30 VLAN30 │
└──┬───────┬──────┬──────┬──────┬──────┬──────┘
   │       │      │      │      │      │
 [Sales] [Sales] [Eng] [Eng] [Mgmt] [Mgmt]

Now: Sales broadcasts stay in VLAN 10.
     Engineering broadcasts stay in VLAN 20.
     Management broadcasts stay in VLAN 30.
     They CANNOT communicate without a router (Layer 3 device).
```

**Think of VLANs as invisible walls inside your switch.** Each VLAN acts like a completely separate switch.

---

## Benefits of VLANs

| Benefit | Explanation |
|---------|-------------|
| **Security** | Sensitive traffic (finance, management) is isolated from general users |
| **Performance** | Smaller broadcast domains = less unnecessary traffic |
| **Flexibility** | Group users logically, not physically — Sales on floor 1 and floor 3 can be in the same VLAN |
| **Cost savings** | One physical switch serves multiple logical networks (instead of buying separate switches) |
| **Easier management** | Add/move users by changing port VLAN assignment, not rewiring |

---

## VLAN Types

### Data VLAN (User VLAN)
- Carries regular user traffic
- You create these: VLAN 10 for Sales, VLAN 20 for Engineering, etc.
- Most of your VLANs will be data VLANs

### Default VLAN (VLAN 1)
- **All ports belong to VLAN 1 by default** on a new switch
- Cannot be deleted, renamed, or shut down
- ⚠️ **Best practice:** Move all ports OUT of VLAN 1 — it's a security risk to leave everything there

### Native VLAN
- The VLAN whose traffic is sent **untagged** on trunk links
- Default is VLAN 1 (which is why you should change it!)
- If a trunk receives an untagged frame, it goes to the native VLAN
- **Both sides of a trunk MUST agree on the native VLAN** (mismatch = security vulnerability)

### Management VLAN
- Used for remote management access (SSH, SNMP, syslog)
- The SVI (Switch Virtual Interface) you assign an IP to
- Can be any VLAN — just keep it separate from user traffic

### Voice VLAN
- Dedicated VLAN for VoIP traffic
- Gets priority treatment (QoS)
- A single switch port can carry BOTH a data VLAN and a voice VLAN
  - IP phone connects to the switch port
  - PC connects to the phone's passthrough port
  - Phone traffic → voice VLAN (tagged)
  - PC traffic → data VLAN (untagged)

```
   [PC] ──── [IP Phone] ──── [Switch Port Fa0/1]
              │                 │
         Data VLAN 10      Voice VLAN 50
         (untagged)         (tagged)
```

---

## VLAN ID Ranges

| Range | VLAN IDs | Name | Storage |
|-------|----------|------|---------|
| Normal | 1–1005 | Normal range | vlan.dat (flash) |
| Extended | 1006–4094 | Extended range | running-config |
| Reserved | 1002–1005 | FDDI/Token Ring (legacy) | Cannot delete |
| Special | 0, 4095 | Reserved by IEEE | Cannot use |

**Normal range (1-1005):**
- Supported by all switches
- Stored in a separate file called `vlan.dat`
- Shared via VTP (VLAN Trunking Protocol)

**Extended range (1006-4094):**
- Requires VTP transparent mode or VTP version 3
- Stored in running-config (not vlan.dat)
- Used in large enterprises that need more than 1005 VLANs

> 💡 **Exam tip:** Most CCNA scenarios use VLANs 1-1005. Know that extended VLANs exist and their requirements.

---

## How VLANs Work with Switches

**Within a single switch:**
- Ports assigned to the same VLAN can communicate
- Ports in different VLANs **cannot** communicate (even on the same switch!)
- The switch maintains a separate MAC address table per VLAN

**Across multiple switches:**
- VLANs need **trunk links** to span across switches
- Trunk links carry traffic for multiple VLANs using 802.1Q tagging
- Without a trunk, VLAN 10 on Switch A can't reach VLAN 10 on Switch B

```
  [VLAN 10 PC]     [VLAN 10 PC]
       |                 |
     SW1 ═══TRUNK═══ SW2
       |                 |
  [VLAN 20 PC]     [VLAN 20 PC]

The trunk carries BOTH VLAN 10 and VLAN 20 traffic between switches.
Each frame on the trunk is tagged with its VLAN ID (except native VLAN).
```

---

## Communication Between VLANs

VLANs create Layer 2 isolation. To communicate between VLANs, you need a **Layer 3 device** (router or Layer 3 switch).

This is called **inter-VLAN routing** — covered in detail on Day 19.

```
Without inter-VLAN routing:
  VLAN 10 ←→ VLAN 10: ✅ (same broadcast domain)
  VLAN 10 ←→ VLAN 20: ❌ (different broadcast domains)

With inter-VLAN routing:
  VLAN 10 ←→ VLAN 20: ✅ (router/L3 switch bridges them)
```

---

## Practice Questions

1. What is a VLAN?
2. What VLAN do all ports belong to by default?
3. Can two PCs in different VLANs on the same switch communicate directly?
4. What is the native VLAN?
5. Why should you move ports out of VLAN 1?
6. How does a voice VLAN work with a data VLAN on the same port?
7. What is the valid range for normal VLANs?
8. What Layer 3 device is needed for inter-VLAN communication?

<details>
<summary>Answers</summary>

1. A logical grouping of switch ports into a separate broadcast domain
2. VLAN 1 (the default VLAN)
3. No — different VLANs are separate broadcast domains; a Layer 3 device is required
4. The VLAN whose traffic is sent untagged on trunk links (default VLAN 1)
5. Security — VLAN 1 is well-known; many protocols use it by default (CDP, VTP, STP BPDUs). Keeping user traffic on VLAN 1 is a security risk.
6. The port carries data VLAN traffic untagged and voice VLAN traffic tagged with 802.1Q. The IP phone tags its own traffic.
7. 1–1005
8. A router or Layer 3 switch
</details>

---

*← [Days 8-10 — Device Configuration](Day08-10-Device-Configuration.md) | [Day 16 — VLAN Configuration](Day16-VLAN-Configuration.md) →*
