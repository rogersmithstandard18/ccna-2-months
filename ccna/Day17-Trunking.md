# Day 17: Trunking with 802.1Q

## 🎯 What You'll Learn
How trunk links carry multiple VLANs between switches, 802.1Q tagging, native VLANs, DTP, and trunk configuration.

---

## What Is a Trunk?

A **trunk** is a point-to-point link that carries traffic for **multiple VLANs**. Without trunks, you'd need a separate cable between switches for every VLAN.

```
WITHOUT trunks (terrible):            WITH a trunk (correct):
SW1 ──VLAN10──→ SW2                   SW1 ═══TRUNK═══ SW2
SW1 ──VLAN20──→ SW2                   (carries ALL VLANs
SW1 ──VLAN30──→ SW2                    on ONE link)
(3 cables!)                           (1 cable!)
```

---

## 802.1Q Tagging

When a frame travels across a trunk, the switch inserts a **4-byte 802.1Q tag** into the Ethernet frame header so the receiving switch knows which VLAN the frame belongs to.

```
Normal frame:
┌──────────┬──────────┬──────────┬──────────────┬─────┐
│ Dst MAC  │ Src MAC  │ Type     │    Data      │ FCS │
└──────────┴──────────┴──────────┴──────────────┴─────┘

Tagged frame (802.1Q):
┌──────────┬──────────┬──────────────┬──────────┬──────────────┬─────┐
│ Dst MAC  │ Src MAC  │  802.1Q Tag  │ Type     │    Data      │ FCS │
│          │          │   (4 bytes)  │          │              │(new)│
└──────────┴──────────┴──────────────┴──────────┴──────────────┴─────┘
```

**The 802.1Q tag (4 bytes):**
```
┌────────────────┬─────────┬─────┬──────────────┐
│  TPID (16 bit) │PCP(3bit)│DEI  │ VLAN ID(12b) │
│  0x8100        │  CoS    │(1b) │  0-4095      │
└────────────────┴─────────┴─────┴──────────────┘
```

| Field | Bits | Purpose |
|-------|------|---------|
| TPID | 16 | Tag Protocol Identifier — always 0x8100 (identifies the frame as tagged) |
| PCP | 3 | Priority Code Point — QoS priority (0-7; higher = more priority) |
| DEI | 1 | Drop Eligible Indicator — can this frame be dropped under congestion? |
| VLAN ID | 12 | The VLAN number (0-4095; 4096 possible VLANs) |

> 💡 **Because of the tag:** A tagged frame is 1522 bytes max (1518 + 4-byte tag). This is why 802.1Q frames are sometimes called "baby giants" — they exceed the normal 1518-byte limit.

---

## Native VLAN — The Exception

The **native VLAN** is the one VLAN whose traffic travels **untagged** on the trunk. By default, this is **VLAN 1**.

```
Frame from VLAN 10 on the trunk: TAGGED with VLAN 10
Frame from VLAN 20 on the trunk: TAGGED with VLAN 20
Frame from VLAN 1 (native) on the trunk: UNTAGGED

When the other switch receives an untagged frame on a trunk,
it puts it into its native VLAN.
```

### Native VLAN Mismatch — A Serious Problem

```
SW1 trunk: native VLAN = 1
SW2 trunk: native VLAN = 99

SW1 sends untagged frame (VLAN 1) → SW2 receives it → puts it in VLAN 99!
Traffic leaks between VLANs! Security vulnerability!
```

**Cisco switches detect this and log a warning:**
```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on...
```

**Best practices:**
1. **Always match native VLANs** on both ends of a trunk
2. **Change native VLAN from the default** (VLAN 1) to a dedicated, unused VLAN
3. **Don't assign any hosts to the native VLAN**

---

## Trunk Configuration

```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport trunk encapsulation dot1q
! Required on switches that support both ISL and 802.1Q
! Newer switches may only support dot1q (command not needed)

SW1(config-if)# switchport mode trunk
! Forces the port into trunk mode

SW1(config-if)# switchport trunk native vlan 99
! Changes native VLAN from 1 to 99

SW1(config-if)# switchport trunk allowed vlan 10,20,30,99
! ONLY allows these VLANs on the trunk (security!)
! By default, ALL VLANs (1-4094) are allowed

SW1(config-if)# switchport nonegotiate
! Disables DTP — the port won't try to negotiate trunking
```

### Managing Allowed VLANs

```
! Add a VLAN to the allowed list
SW1(config-if)# switchport trunk allowed vlan add 40

! Remove a VLAN from the allowed list
SW1(config-if)# switchport trunk allowed vlan remove 30

! Allow only specific VLANs (replaces the list)
SW1(config-if)# switchport trunk allowed vlan 10,20,99

! Reset to all VLANs
SW1(config-if)# switchport trunk allowed vlan all

! ⚠️ COMMON MISTAKE:
! If you type "allowed vlan 40" without "add", it REPLACES the list!
! Only VLAN 40 will be allowed. Always use "add" to add to existing list.
```

---

## DTP (Dynamic Trunking Protocol)

DTP automatically negotiates whether a link should be a trunk or access port. **It's enabled by default on many Cisco switches.**

| Mode | Behavior | Command |
|------|----------|---------|
| **trunk** | Always trunk; sends DTP frames | `switchport mode trunk` |
| **access** | Always access; never trunks | `switchport mode access` |
| **dynamic auto** | Passive — only trunks if other side initiates | `switchport mode dynamic auto` |
| **dynamic desirable** | Active — tries to form a trunk | `switchport mode dynamic desirable` |

**Will a trunk form?**

| | trunk | desirable | auto | access |
|---|---|---|---|---|
| **trunk** | ✅ Trunk | ✅ Trunk | ✅ Trunk | ❌ Mismatch |
| **desirable** | ✅ Trunk | ✅ Trunk | ✅ Trunk | ❌ Access |
| **auto** | ✅ Trunk | ✅ Trunk | ❌ Access | ❌ Access |
| **access** | ❌ Mismatch | ❌ Access | ❌ Access | ❌ Access |

> 💡 **auto + auto = NO trunk!** Both sides are passive, waiting for the other to initiate. Neither does.

**Best practice:** **Disable DTP.** Manually set ports to either `trunk` or `access`, then add `switchport nonegotiate`.

```
! For trunk ports:
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport nonegotiate

! For access ports:
SW1(config-if)# switchport mode access
! Access mode automatically disables DTP negotiation
```

**Why disable DTP?**
- **Security:** An attacker could plug in a device, negotiate a trunk, and access ALL VLANs (VLAN hopping attack)
- **Predictability:** You know exactly what each port does

---

## Verification

```
SW1# show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Gi0/1       on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gi0/1       10,20,30,99

Port        Vlans allowed and active in management domain
Gi0/1       10,20,30,99

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       10,20,30,99
```

```
SW1# show interfaces Gi0/1 switchport
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Trunking Native Mode VLAN: 99
Trunking VLANs Enabled: 10,20,30,99
```

---

## Practice Questions

1. What protocol tags frames on a trunk?
2. What happens to native VLAN traffic on a trunk?
3. What's the default native VLAN?
4. What does `switchport nonegotiate` do?
5. If both sides are set to `dynamic auto`, will a trunk form?
6. What command limits which VLANs can cross a trunk?
7. How do you add VLAN 50 to an existing trunk's allowed list without removing others?

<details>
<summary>Answers</summary>

1. 802.1Q (dot1q)
2. It's sent **untagged**
3. VLAN 1
4. Disables DTP — no trunk negotiation frames are sent
5. No — both are passive; neither initiates
6. `switchport trunk allowed vlan <list>`
7. `switchport trunk allowed vlan add 50`
</details>

---

*← [Day 16 — VLAN Configuration](Day16-VLAN-Configuration.md) | [Day 18 — VTP](Day18-VTP.md) →*
