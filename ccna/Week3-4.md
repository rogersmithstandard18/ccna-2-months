# CCNA Study Guide — Weeks 3–4: Switching Technologies & VLANs

## 🎯 Goals
- Understand VLANs and their purpose in network segmentation
- Configure and verify VLANs and trunking
- Understand and configure inter-VLAN routing
- Master Spanning Tree Protocol (STP) and its variants
- Configure and verify EtherChannel

---

## Day-by-Day Plan

### Day 15 — VLAN Concepts

**What is a VLAN?**
- A logical grouping of switch ports into separate broadcast domains
- Without VLANs: all ports on a switch share one broadcast domain
- With VLANs: each VLAN is its own broadcast domain — broadcasts stay within the VLAN

**Why VLANs?**
- **Security**: Isolate sensitive traffic (e.g., management, finance)
- **Performance**: Reduce broadcast domain size → less unnecessary traffic
- **Flexibility**: Group users logically, not physically
- **Cost**: Fewer switches needed; use one switch for multiple segments

**VLAN Types:**
| Type | VLAN ID | Purpose |
|------|---------|---------|
| Default | 1 | All ports belong here initially; cannot be deleted |
| Data | 2–1001 | User-assigned for data traffic |
| Native | Usually 1 | Untagged traffic on trunks |
| Management | User-defined | Used for switch management (SSH, SNMP) |
| Voice | User-defined | Prioritized for VoIP traffic |
| Extended | 1006–4094 | Requires VTP transparent mode |

**VLAN Ranges:**
- Normal: 1–1005 (stored in vlan.dat)
- Extended: 1006–4094 (stored in running-config; requires VTP transparent/off)
- Reserved: 1002–1005 (FDDI/Token Ring — legacy, cannot be deleted)

---

### Day 16 — VLAN Configuration

**Creating VLANs:**
```
SW1(config)# vlan 10
SW1(config-vlan)# name SALES
SW1(config)# vlan 20
SW1(config-vlan)# name ENGINEERING
SW1(config)# vlan 30
SW1(config-vlan)# name MANAGEMENT
SW1(config)# vlan 99
SW1(config-vlan)# name NATIVE_VLAN
```

**Assigning Ports to VLANs:**
```
SW1(config)# interface FastEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10

! Assign a range of ports
SW1(config)# interface range FastEthernet0/2 - 5
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 20
```

**Voice VLAN Configuration:**
```
SW1(config)# interface FastEthernet0/6
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport voice vlan 50
! Port carries data VLAN 10 (untagged) + voice VLAN 50 (tagged with 802.1Q)
```

**Verification Commands:**
```
show vlan brief                    ! All VLANs and assigned ports
show vlan id 10                    ! Specific VLAN details
show interfaces switchport         ! Port mode, VLAN assignments
show interfaces fa0/1 switchport   ! Specific port details
```

---

### Day 17 — Trunking (802.1Q)

**What is a Trunk?**
- A link that carries traffic for multiple VLANs between switches (or switch↔router)
- Uses 802.1Q tagging: inserts a 4-byte tag into the Ethernet frame header
- **Native VLAN**: Frames on the native VLAN are sent **untagged** on the trunk

**802.1Q Tag Format:**
```
| TPID (2B: 0x8100) | PCP (3 bits) | DEI (1 bit) | VLAN ID (12 bits) |
```
- VLAN ID: 0–4095 (4096 possible VLANs)
- PCP: Priority Code Point (CoS for QoS)

**Trunk Configuration:**
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport trunk encapsulation dot1q    ! Required on some switches
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# switchport trunk allowed vlan 10,20,30,99

! Remove a VLAN from trunk
SW1(config-if)# switchport trunk allowed vlan remove 30

! Add a VLAN to trunk
SW1(config-if)# switchport trunk allowed vlan add 40
```

**DTP (Dynamic Trunking Protocol):**
| Mode | Description |
|------|-------------|
| `switchport mode trunk` | Forces trunk; sends DTP frames |
| `switchport mode access` | Forces access; never trunks |
| `switchport mode dynamic auto` | Trunks only if other side is trunk or desirable (default on many switches) |
| `switchport mode dynamic desirable` | Actively tries to trunk |
| `switchport nonegotiate` | Disables DTP (use with mode trunk or access) |

**Best Practice:** Always manually set trunk or access mode; disable DTP with `switchport nonegotiate`.

**Verification:**
```
show interfaces trunk
show interfaces gi0/1 switchport
show interfaces gi0/1 trunk
```

**Native VLAN Mismatch:**
- If two switches have different native VLANs on a trunk, frames leak between VLANs
- CDP/Cisco will log a native VLAN mismatch warning
- **Always match native VLAN on both ends** and use a non-default VLAN (not VLAN 1)

---

### Day 18 — VTP (VLAN Trunking Protocol)

**Purpose:** Synchronize VLAN databases across switches automatically.

**VTP Modes:**
| Mode | Creates/Deletes VLANs | Forwards Advertisements | Syncs VLAN DB |
|------|----------------------|------------------------|---------------|
| Server (default) | Yes | Yes | Yes |
| Client | No | Yes | Yes |
| Transparent | Yes (local only) | Yes (passes through) | No |
| Off (v3) | No | No | No |

**VTP Versions:**
- v1: Original; supports VLANs 1–1005
- v2: Adds Token Ring support, consistency checks
- v3: Supports extended VLANs (1006–4094), private VLANs, per-port configuration, primary server concept

**VTP Configuration:**
```
SW1(config)# vtp domain MYCOMPANY
SW1(config)# vtp mode server
SW1(config)# vtp password SecretVTP
SW1(config)# vtp version 2
```

**⚠️ VTP Danger:**
- A switch with a **higher revision number** will overwrite the VLAN database on all VTP clients/servers
- **Before adding a switch to the network:** reset its VTP revision to 0 by changing VTP domain to a dummy name and back, or set it to transparent mode
- Many production networks use VTP transparent mode or disable VTP entirely

---

### Day 19 — Inter-VLAN Routing

**Problem:** VLANs are separate broadcast domains; hosts in different VLANs can't communicate without a Layer 3 device.

**Method 1: Router-on-a-Stick (ROAS)**
- Single physical link from switch to router
- Router uses sub-interfaces, each tagged with a VLAN

```
! Switch side: trunk to router
SW1(config)# interface Gi0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99

! Router side: sub-interfaces
R1(config)# interface Gi0/0
R1(config-if)# no shutdown

R1(config)# interface Gi0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0

R1(config)# interface Gi0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0

R1(config)# interface Gi0/0.99
R1(config-subif)# encapsulation dot1Q 99 native
R1(config-subif)# ip address 192.168.99.1 255.255.255.0
```

**Method 2: Layer 3 Switch (SVI)**
- Create SVIs (Switch Virtual Interfaces) for each VLAN
- Enable IP routing on the switch

```
SW1(config)# ip routing

SW1(config)# interface vlan 10
SW1(config-if)# ip address 192.168.10.1 255.255.255.0
SW1(config-if)# no shutdown

SW1(config)# interface vlan 20
SW1(config-if)# ip address 192.168.20.1 255.255.255.0
SW1(config-if)# no shutdown
```

**Method 3: Routed Port on L3 Switch**
```
SW1(config)# interface Gi0/1
SW1(config-if)# no switchport
SW1(config-if)# ip address 10.0.0.1 255.255.255.252
```

**Comparison:**
| Method | Scalability | Performance | Cost |
|--------|-------------|-------------|------|
| ROAS | Low (bandwidth bottleneck) | Lower | Low (basic router) |
| SVI (L3 switch) | High | High (hardware switching) | Higher |
| Routed port | Point-to-point links | High | L3 switch required |

---

### Day 20–21 — Spanning Tree Protocol (STP)

**Why STP?**
- Redundant links between switches create **Layer 2 loops**
- Loops cause: broadcast storms, MAC table instability, duplicate frames
- STP (IEEE 802.1D) prevents loops by blocking redundant paths

**STP Process:**
1. **Elect Root Bridge**: Lowest Bridge ID (Priority + MAC)
   - Default priority: 32768
   - Priority must be a multiple of 4096
2. **Determine Root Ports**: Each non-root switch selects the port with lowest cost to root
3. **Determine Designated Ports**: One designated port per segment (lowest cost to root)
4. **Block remaining ports**: Ports that are neither root nor designated go to blocking state

**STP Port States (802.1D):**
| State | Duration | Sends/Receives BPDUs | Learns MACs | Forwards Data |
|-------|----------|---------------------|-------------|---------------|
| Blocking | — | Receives only | No | No |
| Listening | 15 sec (Forward Delay) | Yes | No | No |
| Learning | 15 sec (Forward Delay) | Yes | Yes | No |
| Forwarding | — | Yes | Yes | Yes |
| Disabled | — | No | No | No |

**STP Timers:**
- Hello: 2 seconds (BPDU interval)
- Forward Delay: 15 seconds (listening→learning, learning→forwarding)
- Max Age: 20 seconds (how long to wait before reconverging)
- Total convergence: ~30-50 seconds

**STP Path Costs:**
| Bandwidth | Original Cost | Revised Cost (802.1D-1998) |
|-----------|--------------|---------------------------|
| 10 Mbps | 100 | 100 |
| 100 Mbps | 19 | 19 |
| 1 Gbps | 4 | 4 |
| 10 Gbps | 2 | 2 |

**STP Variants:**
| Protocol | Standard | Convergence | Instances |
|----------|----------|-------------|-----------|
| STP (CST) | 802.1D | ~50 sec | 1 (all VLANs) |
| PVST+ | Cisco | ~50 sec | 1 per VLAN |
| RSTP | 802.1w | ~1-2 sec | 1 (all VLANs) |
| Rapid PVST+ | Cisco | ~1-2 sec | 1 per VLAN |
| MSTP | 802.1s | ~1-2 sec | Mapped (multiple VLANs per instance) |

**Configuring STP:**
```
! Set root bridge (macro — sets priority low enough to win)
SW1(config)# spanning-tree vlan 10 root primary
SW1(config)# spanning-tree vlan 10 root secondary  ! Backup root

! Set priority manually
SW1(config)# spanning-tree vlan 10 priority 4096

! Change to Rapid PVST+
SW1(config)# spanning-tree mode rapid-pvst

! Verification
show spanning-tree
show spanning-tree vlan 10
show spanning-tree summary
show spanning-tree interface fa0/1
```

**PortFast & BPDU Guard:**
```
! PortFast — skip listening/learning on access ports (host-facing)
SW1(config)# interface Fa0/1
SW1(config-if)# spanning-tree portfast

! Enable globally for all access ports
SW1(config)# spanning-tree portfast default

! BPDU Guard — shut down port if BPDU received (prevents rogue switches)
SW1(config-if)# spanning-tree bpduguard enable

! Enable globally
SW1(config)# spanning-tree portfast bpduguard default
```

**Other STP Enhancements:**
- **Root Guard**: Prevents a port from becoming root port (blocks superior BPDUs)
- **Loop Guard**: Prevents a blocked port from transitioning to forwarding if BPDUs stop
- **UplinkFast**: Faster failover for access-layer switches (PVST+ only)
- **BackboneFast**: Faster convergence on indirect link failures

---

### Day 22–23 — EtherChannel

**What is EtherChannel?**
- Bundle 2–8 physical links into one logical link
- Provides redundancy and increased bandwidth
- STP sees one logical link → no blocking
- Load balancing across member links

**EtherChannel Protocols:**
| Protocol | Standard | Negotiation |
|----------|----------|-------------|
| LACP | IEEE 802.3ad | Active/Passive |
| PAgP | Cisco proprietary | Desirable/Auto |
| Static (mode on) | — | No negotiation |

**LACP Modes:**
- Active: Initiates negotiation
- Passive: Responds only
- Active↔Active ✅ | Active↔Passive ✅ | Passive↔Passive ❌

**Configuration:**
```
! LACP EtherChannel
SW1(config)# interface range Gi0/1 - 2
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# no shutdown

SW1(config)# interface port-channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# switchport trunk allowed vlan 10,20,30,99

! PAgP EtherChannel
SW2(config)# interface range Gi0/1 - 2
SW2(config-if-range)# channel-group 2 mode desirable
```

**Requirements for EtherChannel:**
- All ports must have same: speed, duplex, VLAN configuration, trunk mode, native VLAN, allowed VLANs
- If any mismatch → EtherChannel won't form or will be suspended

**Load Balancing:**
```
SW1(config)# port-channel load-balance src-dst-mac
! Options: src-mac, dst-mac, src-dst-mac, src-ip, dst-ip, src-dst-ip, src-port, dst-port
show etherchannel load-balance
```

**Verification:**
```
show etherchannel summary
show etherchannel port-channel
show etherchannel detail
show interfaces port-channel 1
```

---

### Days 24–28 — Labs & Review

**Lab 1: VLAN & Trunk Configuration**
- Create VLANs 10, 20, 30 on two switches
- Assign ports to VLANs
- Configure trunk between switches with native VLAN 99
- Verify with `show vlan brief` and `show interfaces trunk`
- Test: hosts in same VLAN across switches should ping; different VLANs should not

**Lab 2: Inter-VLAN Routing (ROAS)**
- Configure router-on-a-stick with sub-interfaces
- Set default gateways on hosts
- Verify cross-VLAN communication

**Lab 3: STP Observation**
- Build a redundant triangle topology (3 switches)
- Identify root bridge, root ports, designated ports, blocked ports
- Change root bridge by setting priority
- Enable Rapid PVST+ and observe faster convergence

**Lab 4: EtherChannel**
- Bundle two links between switches using LACP
- Verify the channel is up with `show etherchannel summary`
- Disconnect one link and verify traffic continues

---

## 📝 Review Quiz — Weeks 3–4

1. What is the purpose of a VLAN?
2. What protocol tags frames on a trunk link?
3. What happens to native VLAN traffic on a trunk?
4. What is the default STP priority?
5. How long does 802.1D STP take to converge?
6. What does PortFast do?
7. How many links can an EtherChannel bundle?
8. What are the two EtherChannel negotiation protocols?
9. In ROAS, where do you configure the VLAN gateway IPs?
10. What command shows which ports are in each VLAN?

<details>
<summary>Answers</summary>

1. Segment a network into separate broadcast domains for security, performance, and flexibility
2. 802.1Q (dot1q)
3. It is sent untagged
4. 32768
5. ~30-50 seconds (listening + learning = 30s, plus max age)
6. Immediately transitions an access port to forwarding, skipping listening/learning
7. Up to 8 physical links
8. LACP (IEEE) and PAgP (Cisco)
9. On the router's sub-interfaces (e.g., Gi0/0.10)
10. `show vlan brief`
</details>
