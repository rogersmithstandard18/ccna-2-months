# Days 20-21: Spanning Tree Protocol (STP)

## 🎯 What You'll Learn
Why Layer 2 loops are catastrophic, how STP prevents them, root bridge election, port roles/states, STP variants, and PortFast/BPDU Guard.

---

## The Problem: Layer 2 Loops

Redundant links between switches are great for fault tolerance — but they create **loops** that destroy your network.

```
      SW1 ──────── SW2
       │  \      /  │
       │   \    /   │
       │    \  /    │
       │     \/     │
       │     /\     │
       │    /  \    │
       └──SW3───────┘

Three switches, redundant links. Looks great — until a broadcast happens.
```

**What happens without STP:**
1. PC-A sends a broadcast
2. SW1 floods it to SW2 and SW3
3. SW2 floods it to SW3 (and back to SW1!)
4. SW3 floods it to SW1 (and back to SW2!)
5. Each switch keeps flooding, forever
6. **Broadcast storm** — bandwidth consumed, CPUs pegged, network dies

**Three deadly effects:**
| Problem | Description |
|---------|-------------|
| **Broadcast storm** | Broadcasts multiply infinitely, consuming all bandwidth |
| **MAC table instability** | The same MAC appears on different ports as duplicates arrive from different directions |
| **Duplicate frames** | Applications receive the same data multiple times |

**STP's job:** Block redundant paths to eliminate loops, while keeping them available as backups.

---

## How STP Works — The Big Picture

STP (IEEE 802.1D) runs on all switches automatically. It:
1. Elects a **Root Bridge** (the "center" of the spanning tree)
2. Calculates the **shortest path** from every switch to the root
3. **Blocks** redundant ports that would create loops
4. **Unblocks** them if an active path fails (convergence)

---

## Step 1: Root Bridge Election

**Every switch has a Bridge ID:**
```
Bridge ID = Bridge Priority (16 bits) + MAC Address (48 bits)

Default priority: 32768 (all switches start equal)
```

**The switch with the LOWEST Bridge ID becomes the Root Bridge.**

Since all switches start with priority 32768, the one with the **lowest MAC address** wins by default. But you should **manually set** the root bridge by lowering its priority.

```
! Make SW1 the root bridge for VLAN 10
SW1(config)# spanning-tree vlan 10 root primary
! Sets priority to 24576 (or 4096 less than current root)

! Make SW2 the backup root
SW2(config)# spanning-tree vlan 10 root secondary
! Sets priority to 28672

! Or set priority manually (must be multiple of 4096)
SW1(config)# spanning-tree vlan 10 priority 4096
```

**With PVST+ (Per-VLAN Spanning Tree):**
The Bridge ID actually includes the VLAN ID:
```
Bridge ID = Priority (4 bits) + VLAN ID (12 bits) + MAC Address
Priority: 0, 4096, 8192, 12288, 16384, 20480, 24576, 28672, 32768...
Default for VLAN 1: 32768 + 1 = 32769
```

> 💡 **Exam tip:** The root bridge should be a powerful, central switch — not some random access switch. Always configure it deliberately.

---

## Step 2: Port Roles

After the root bridge is elected, every port on every switch gets a role:

| Role | Description | How Many |
|------|-------------|----------|
| **Root Port (RP)** | Port closest to the root bridge | ONE per non-root switch |
| **Designated Port (DP)** | Best port on each segment toward the root | ONE per segment/link |
| **Blocked/Alternate Port** | Redundant port — blocked to prevent loops | Everything else |

**On the Root Bridge:** All ports are **Designated Ports** (it IS the root — everything points to it).

### Determining the Root Port

Each non-root switch picks its Root Port based on (in order):
1. **Lowest root path cost** (cumulative cost to reach the root)
2. **Lowest sender Bridge ID** (if costs are tied)
3. **Lowest sender Port ID** (if Bridge IDs are tied)

**STP Path Costs:**
| Bandwidth | Cost (802.1D-1998) |
|-----------|--------------------|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

**Example:**
```
         [Root Bridge SW1]
        /        \
    cost 4      cost 4
      /            \
   [SW2]          [SW3]
      \            /
    cost 4      cost 4
        \        /
         [SW4]

SW4's path to root:
  Via SW2: cost 4 + 4 = 8
  Via SW3: cost 4 + 4 = 8
  Tied! → Compare SW2 vs SW3 Bridge IDs → lower BID wins
  If SW2 has lower BID → SW4's Root Port faces SW2
```

---

## Step 3: Port States

After determining roles, ports transition through states:

| State | Duration | Forwards Data | Learns MACs | Sends/Receives BPDUs |
|-------|----------|--------------|-------------|---------------------|
| **Blocking** | Indefinite | ❌ | ❌ | Receives only |
| **Listening** | 15 sec | ❌ | ❌ | ✅ Both |
| **Learning** | 15 sec | ❌ | ✅ | ✅ Both |
| **Forwarding** | Indefinite | ✅ | ✅ | ✅ Both |
| **Disabled** | Indefinite | ❌ | ❌ | ❌ |

**Total convergence time: ~30-50 seconds** (Max Age 20s + Listening 15s + Learning 15s)

```
Link comes up → Blocking → Listening (15s) → Learning (15s) → Forwarding

That's 30 seconds before a port can forward traffic!
(This is why Rapid STP was created)
```

---

## BPDUs (Bridge Protocol Data Units)

BPDUs are messages switches send to each other to build and maintain the spanning tree.

**Key BPDU fields:**
- Root Bridge ID (who the sender thinks is root)
- Cost to Root (sender's cost to reach the root)
- Sender Bridge ID
- Sender Port ID

**Hello Timer:** 2 seconds (how often the root sends BPDUs)
**Forward Delay:** 15 seconds (listening and learning durations)
**Max Age:** 20 seconds (how long to keep a BPDU before assuming the root is gone)

---

## STP Variants — Know the Differences

| Protocol | Standard | Convergence | Instances | Notes |
|----------|----------|-------------|-----------|-------|
| **STP (CST)** | 802.1D | ~50 sec | 1 for ALL VLANs | Original; obsolete |
| **PVST+** | Cisco | ~50 sec | 1 per VLAN | Cisco default; allows per-VLAN root bridges |
| **RSTP** | 802.1w | ~1-2 sec | 1 for ALL VLANs | Fast convergence |
| **Rapid PVST+** | Cisco | ~1-2 sec | 1 per VLAN | **Best of both worlds** ← Use this |
| **MSTP** | 802.1s | ~1-2 sec | Grouped | Maps multiple VLANs to instances |

### RSTP (Rapid Spanning Tree Protocol)

RSTP dramatically improves convergence from ~50 seconds to **1-2 seconds**.

**RSTP Port Roles (adds two):**
| Role | Replaces | Purpose |
|------|----------|---------|
| Root Port | Same as STP | Best path to root |
| Designated Port | Same as STP | Best port per segment |
| **Alternate Port** | STP Blocked | Backup root port (instant failover!) |
| **Backup Port** | STP Blocked | Backup designated port (same switch) |

**RSTP Port States (simplified):**
| RSTP State | STP Equivalent |
|-----------|----------------|
| Discarding | Blocking + Listening |
| Learning | Learning |
| Forwarding | Forwarding |

**Why is RSTP faster?**
- No Forward Delay for edge ports (like PortFast)
- Alternate ports can take over instantly (no Max Age wait)
- Active proposal/agreement mechanism between switches

```
! Enable Rapid PVST+
SW1(config)# spanning-tree mode rapid-pvst
```

---

## PortFast & BPDU Guard

### PortFast
Normally, a port takes 30 seconds to forward (Blocking→Listening→Learning→Forwarding). For access ports connected to PCs/servers, this is unnecessary and delays DHCP.

**PortFast skips straight to Forwarding:**

```
! Per-interface
SW1(config-if)# spanning-tree portfast
! Only use on ACCESS ports connected to end devices!
! NEVER on ports connected to switches (would bypass loop protection!)

! Globally (all access ports)
SW1(config)# spanning-tree portfast default
```

### BPDU Guard
**What if someone plugs a switch into a PortFast port?** That could create a loop. BPDU Guard shuts the port down immediately if it receives a BPDU.

```
! Per-interface
SW1(config-if)# spanning-tree bpduguard enable

! Globally (all PortFast ports)
SW1(config)# spanning-tree portfast bpduguard default
```

**When BPDU Guard triggers:**
```
%PM-4-ERR_DISABLE: bpduguard error detected on Fa0/1, putting Fa0/1 in err-disable state

! Port is shut down. To recover:
SW1(config)# interface Fa0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown

! Or enable automatic recovery:
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300   ! Retry after 5 minutes
```

### Other STP Enhancements

| Feature | Purpose |
|---------|---------|
| **Root Guard** | Prevents a port from becoming a root port (blocks superior BPDUs). Use on ports facing access layer switches to prevent them from becoming root. |
| **Loop Guard** | Prevents a port from transitioning to forwarding if BPDUs stop arriving (could indicate a unidirectional link failure) |
| **UplinkFast** | Faster failover for access-layer root port failure (PVST+ only; built into RSTP) |
| **BackboneFast** | Faster convergence on indirect failures (PVST+ only; built into RSTP) |

---

## Verification Commands

```
show spanning-tree                    ! Full STP status
show spanning-tree vlan 10            ! STP for specific VLAN
show spanning-tree summary            ! Overview with counts
show spanning-tree root               ! Root bridge info per VLAN
show spanning-tree interface Fa0/1    ! Specific port STP status
show spanning-tree detail             ! Detailed timers and info
```

**Reading `show spanning-tree` output:**
```
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106            ← Root bridge priority + VLAN
             Address     aabb.ccdd.1111  ← Root bridge MAC
             Cost        4               ← This switch's cost to root
             Port        1 (GigabitEthernet0/1)  ← Root port
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778           ← THIS switch's priority + VLAN
             Address     aabb.ccdd.2222  ← THIS switch's MAC

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- ----
Gi0/1            Root FWD 4         128.1    P2p      ← Root Port, Forwarding
Fa0/1            Desg FWD 19        128.2    Edge     ← Designated, PortFast
Fa0/2            Altn BLK 19        128.3    P2p      ← Alternate, Blocked
```

---

## Practice Questions

1. What three problems do Layer 2 loops cause?
2. How is the root bridge elected?
3. What is the default STP priority?
4. How long does 802.1D take to converge?
5. What are the three main STP port roles?
6. What does PortFast do, and when should you use it?
7. What happens when BPDU Guard is triggered?
8. What is an RSTP Alternate Port?
9. What command makes a switch the root bridge for VLAN 10?
10. Why is Rapid PVST+ preferred over PVST+?

<details>
<summary>Answers</summary>

1. Broadcast storms, MAC table instability, duplicate frames
2. Lowest Bridge ID (priority + MAC address). Lowest priority wins; if tied, lowest MAC wins.
3. 32768
4. ~30-50 seconds (Max Age + 2× Forward Delay)
5. Root Port, Designated Port, Blocked/Alternate Port
6. Skips Listening and Learning states, goes straight to Forwarding. Use ONLY on access ports connected to end devices.
7. The port is put into err-disabled state (shut down)
8. A backup root port — provides instant failover if the root port fails
9. `spanning-tree vlan 10 root primary`
10. Same per-VLAN spanning tree benefits, but converges in 1-2 seconds instead of ~50 seconds
</details>

---

*← [Day 19 — Inter-VLAN Routing](Day19-InterVLAN-Routing.md) | [Days 22-23 — EtherChannel](Day22-23-EtherChannel.md) →*
