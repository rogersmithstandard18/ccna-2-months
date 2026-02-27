# Day 18: VTP (VLAN Trunking Protocol)

## 🎯 What You'll Learn
How VTP synchronizes VLAN databases across switches, VTP modes, versions, and why it can be dangerous.

---

## What VTP Does

VTP lets you create a VLAN on **one switch** and have it automatically appear on **all other switches** in the same VTP domain. Without VTP, you'd manually create VLANs on every switch.

```
Without VTP:                          With VTP:
Create VLAN 10 on SW1 ✓              Create VLAN 10 on SW1 (server)
Create VLAN 10 on SW2 ✓              → Automatically appears on SW2, SW3, SW4
Create VLAN 10 on SW3 ✓
Create VLAN 10 on SW4 ✓
(Tedious and error-prone)            (Automatic — but risky!)
```

**VTP operates over trunk links only.** It uses the trunk to send advertisements.

---

## VTP Modes

| Mode | Create/Delete VLANs | Forwards Ads | Syncs from Ads | Stores in vlan.dat |
|------|---------------------|-------------|----------------|-------------------|
| **Server** (default) | ✅ Yes | ✅ Yes | ✅ Yes | Yes |
| **Client** | ❌ No | ✅ Yes | ✅ Yes | Yes (v2)/No (v1) |
| **Transparent** | ✅ Yes (local only) | ✅ Forwards (passes through) | ❌ No | Yes |
| **Off** (v3 only) | ✅ Yes (local) | ❌ No | ❌ No | Yes |

**Server mode:**
- Full control — create, modify, delete VLANs
- Sends VTP advertisements to clients
- Syncs its VLAN database from other servers with higher revision numbers

**Client mode:**
- Cannot create or delete VLANs
- Receives and applies VLAN changes from servers
- Forwards advertisements to other switches

**Transparent mode:**
- Creates/deletes VLANs **locally only** (doesn't share)
- Passes VTP advertisements through to other switches
- Does **not** sync its own database from advertisements
- **This is the recommended mode in most production networks**

---

## VTP Revision Number — The Danger

Every VTP domain has a **revision number** that starts at 0 and increments by 1 every time a VLAN change is made.

```
SW1 (server): revision 10, VLANs: 1, 10, 20, 30
SW2 (client): revision 10, VLANs: 1, 10, 20, 30
SW3 (client): revision 10, VLANs: 1, 10, 20, 30
```

**The switch with the HIGHEST revision number wins.** All other switches sync to it.

### The Disaster Scenario

```
1. You have a production network: revision 15, VLANs 1,10,20,30,40,50
2. You pull an old switch from the closet
3. That switch was in the same VTP domain with revision 25 and only VLAN 1
4. You connect it to the network via a trunk
5. Because revision 25 > 15, ALL switches sync to the old switch's database
6. VLANs 10,20,30,40,50 are DELETED from EVERY switch
7. Every port assigned to those VLANs goes DEAD
8. Your entire network goes down 💀
```

**This has happened in production networks.** It's a real risk.

### Prevention

Before connecting any switch to the network:

```
! Option 1: Change VTP domain to a dummy name and back (resets revision to 0)
SW-NEW(config)# vtp domain DUMMY
SW-NEW(config)# vtp domain PRODUCTION

! Option 2: Set VTP mode to transparent
SW-NEW(config)# vtp mode transparent

! Option 3: Use VTP version 3 (primary server concept prevents this)
```

---

## VTP Configuration

```
! Set VTP domain (must match on all switches)
SW1(config)# vtp domain MYCOMPANY

! Set VTP mode
SW1(config)# vtp mode server          ! Default
SW1(config)# vtp mode client
SW1(config)# vtp mode transparent     ! Recommended

! Set VTP password (must match on all switches in the domain)
SW1(config)# vtp password SecretVTP

! Set VTP version
SW1(config)# vtp version 2
```

**Verification:**
```
SW1# show vtp status
VTP Version capable           : 1 to 3
VTP version running           : 2
VTP Domain Name               : MYCOMPANY
VTP Pruning Mode              : Disabled
VTP Traps Generation          : Disabled
Device ID                     : aabb.ccdd.eeff
Configuration last modified by 192.168.1.2 at 2-26-26 15:00:00
Local updater ID is 192.168.1.2

Feature VLAN:
VTP Operating Mode            : Server
Maximum VLANs supported locally : 1005
Number of existing VLANs      : 8
Configuration Revision        : 15     ← Watch this number!
```

---

## VTP Versions

| Feature | VTPv1 | VTPv2 | VTPv3 |
|---------|-------|-------|-------|
| VLAN range | 1-1005 | 1-1005 | 1-4094 (extended!) |
| Transparent forwarding | Only same version | Any version | Any version |
| Consistency checks | No | Yes | Yes |
| Primary server | No | No | Yes (prevents accidental overwrites) |
| Private VLANs | No | No | Yes |
| Per-port config | No | No | Yes |

**VTPv3 primary server:**
- Only the designated primary server can make VLAN changes
- Prevents the "rogue switch" scenario
- Must be explicitly promoted: `vtp primary`

---

## VTP Pruning

VTP Pruning automatically stops flooding broadcast traffic to switches that don't have any ports in that VLAN.

```
Without pruning:
SW1 sends VLAN 10 broadcast → floods to SW2 (has VLAN 10 ports) ✅
SW1 sends VLAN 10 broadcast → floods to SW3 (NO VLAN 10 ports) ❌ Wasted!

With pruning:
SW1 sends VLAN 10 broadcast → floods to SW2 (has VLAN 10 ports) ✅
SW1 sends VLAN 10 broadcast → does NOT flood to SW3 ✅ Efficient!
```

```
SW1(config)# vtp pruning
! Enables pruning on the entire VTP domain (from a server)
```

---

## Modern Recommendation

Many network engineers **disable VTP** entirely:
- Set all switches to **transparent mode** or VTP **off** (v3)
- Manually create VLANs on each switch
- Automation tools (Ansible, Python scripts) can handle VLAN consistency
- Eliminates the risk of VTP disasters

> 💡 **Exam tip:** Know how VTP works and the revision number danger. In practice, most shops use transparent mode or disable VTP.

---

## Practice Questions

1. What is VTP's purpose?
2. Which VTP mode can create VLANs but doesn't sync from advertisements?
3. What determines which switch's VLAN database "wins"?
4. How do you reset the VTP revision number to 0?
5. What does VTP pruning do?
6. What VTP version supports extended VLANs (1006-4094)?

<details>
<summary>Answers</summary>

1. Synchronizes VLAN databases across switches in the same VTP domain
2. Transparent mode
3. The highest revision number wins
4. Change the VTP domain to a different name and back
5. Stops flooding VLANs to switches that have no ports in that VLAN
6. VTPv3
</details>

---

*← [Day 17 — Trunking](Day17-Trunking.md) | [Day 19 — Inter-VLAN Routing](Day19-InterVLAN-Routing.md) →*
