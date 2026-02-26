# CCNA Study Guide — Weeks 5–6: IP Routing & Dynamic Routing Protocols

## 🎯 Goals
- Understand how routers make forwarding decisions
- Configure and verify static and default routes
- Understand OSPF single-area and multi-area concepts
- Configure and verify OSPFv2
- Understand EIGRP basics (for comparison)
- Understand administrative distance and route selection

---

## Day-by-Day Plan

### Day 29 — How Routing Works

**Routing Decision Process:**
1. Packet arrives on an interface
2. Router checks destination IP against the routing table
3. Longest prefix match wins (most specific route)
4. If match found → forward out the exit interface / next-hop
5. If no match → drop packet (unless default route exists)

**Routing Table Components:**
```
R1# show ip route
Codes: C - connected, S - static, O - OSPF, D - EIGRP, B - BGP, * - candidate default

Gateway of last resort is 10.0.0.2 to network 0.0.0.0

     10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       10.0.0.0/30 is directly connected, GigabitEthernet0/1
L       10.0.0.1/32 is directly connected, GigabitEthernet0/1
O       172.16.0.0/16 [110/20] via 10.0.0.2, 00:05:30, GigabitEthernet0/1
S       192.168.50.0/24 [1/0] via 10.0.0.2
S*      0.0.0.0/0 [1/0] via 10.0.0.2
```

**Reading a route entry:** `O 172.16.0.0/16 [110/20] via 10.0.0.2, 00:05:30, Gi0/1`
- `O` = OSPF learned
- `172.16.0.0/16` = destination network
- `[110/20]` = [Administrative Distance / Metric]
- `via 10.0.0.2` = next-hop IP
- `Gi0/1` = exit interface

**Administrative Distance (AD) — Route Trustworthiness:**
| Source | AD |
|--------|-----|
| Connected | 0 |
| Static | 1 |
| EIGRP Summary | 5 |
| eBGP | 20 |
| EIGRP (internal) | 90 |
| IGRP | 100 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 170 |
| iBGP | 200 |
| Unknown/Unreachable | 255 |

Lower AD = more trusted. If two protocols know the same route, the one with lower AD wins.

---

### Day 30 — Static Routing

**Types of Static Routes:**
```
! Standard static route (next-hop IP)
R1(config)# ip route 192.168.20.0 255.255.255.0 10.0.0.2

! Static route (exit interface) — use only on point-to-point links
R1(config)# ip route 192.168.20.0 255.255.255.0 GigabitEthernet0/1

! Fully specified (next-hop + exit interface)
R1(config)# ip route 192.168.20.0 255.255.255.0 GigabitEthernet0/1 10.0.0.2

! Default route (gateway of last resort)
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2

! Floating static route (backup; higher AD than primary)
R1(config)# ip route 192.168.20.0 255.255.255.0 10.0.1.2 5
! AD of 5 — only used if EIGRP (AD 90) route disappears... wait, that's lower.
! Correct: set AD higher than the primary route's protocol
R1(config)# ip route 192.168.20.0 255.255.255.0 10.0.1.2 200
! AD 200 — only used if OSPF (AD 110) route disappears
```

**IPv6 Static Routes:**
```
R1(config)# ipv6 route 2001:DB8:ACAD:2::/64 2001:DB8:ACAD:3::2
R1(config)# ipv6 route ::/0 2001:DB8:ACAD:3::2    ! Default route
```

**Verification:**
```
show ip route
show ip route static
show ip route 192.168.20.0
show ipv6 route
```

---

### Day 31–32 — OSPF Concepts

**What is OSPF?**
- Open Shortest Path First
- Link-state routing protocol (each router builds a complete topology map)
- Uses Dijkstra's SPF algorithm to calculate shortest path
- Classless (supports VLSM/CIDR)
- AD = 110
- Metric = Cost (based on bandwidth): Cost = Reference BW / Interface BW
  - Default reference: 100 Mbps
  - FastEthernet (100 Mbps) = cost 1
  - GigabitEthernet (1 Gbps) = cost 1 (same! Must adjust reference)

**OSPF Packet Types:**
| Type | Name | Purpose |
|------|------|---------|
| 1 | Hello | Discover/maintain neighbors |
| 2 | DBD (Database Description) | Summary of LSDB contents |
| 3 | LSR (Link-State Request) | Request specific LSAs |
| 4 | LSU (Link-State Update) | Send requested LSAs |
| 5 | LSAck | Acknowledge LSAs |

**OSPF Neighbor Requirements (must match):**
- Hello/Dead timers
- Area ID
- Subnet mask (on the shared link)
- Authentication (if configured)
- Stub area flag
- MTU (for DBD exchange; mismatch causes stuck in ExStart/Exchange)

**OSPF Neighbor States:**
```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```
- **2-Way**: Bidirectional communication confirmed (DR/BDR election happens here on multi-access networks)
- **Full**: LSDB synchronized; adjacency formed

**DR/BDR Election (multi-access networks like Ethernet):**
- DR (Designated Router): Collects and distributes LSAs on behalf of the segment
- BDR (Backup Designated Router): Takes over if DR fails
- DROther: All other routers; form full adjacency only with DR and BDR
- Election criteria: Highest OSPF priority → Highest Router ID
- Priority 0 = cannot be DR/BDR
- Election is **non-preemptive** (a new router with higher priority won't take over until DR/BDR fails)

**OSPF Areas:**
- **Area 0 (Backbone)**: All areas must connect to Area 0
- **Regular Area**: Connected to Area 0 via ABR
- **Stub Area**: No external (Type 5) LSAs; uses default route
- **Totally Stubby**: No external or inter-area (Type 3) LSAs except default
- **NSSA**: Not-So-Stubby Area; allows limited external routes via Type 7 LSAs
- **ABR** (Area Border Router): Connects two or more areas
- **ASBR** (Autonomous System Boundary Router): Redistributes routes from another protocol/AS

**OSPF LSA Types (Key ones):**
| Type | Name | Generated By | Flooded Within |
|------|------|-------------|----------------|
| 1 | Router LSA | Every router | Area |
| 2 | Network LSA | DR | Area |
| 3 | Summary LSA | ABR | Other areas |
| 4 | ASBR Summary | ABR | Other areas |
| 5 | External LSA | ASBR | Entire OSPF domain |
| 7 | NSSA External | ASBR in NSSA | NSSA only (converted to Type 5 at ABR) |

---

### Day 33–34 — OSPF Configuration

**Single-Area OSPF:**
```
R1(config)# router ospf 1                           ! Process ID (local significance)
R1(config-router)# router-id 1.1.1.1               ! Manually set Router ID (recommended)
R1(config-router)# network 192.168.10.0 0.0.0.255 area 0
R1(config-router)# network 10.0.0.0 0.0.0.3 area 0

! Passive interface (don't send hellos; still advertise the network)
R1(config-router)# passive-interface GigabitEthernet0/0

! Make all interfaces passive by default, then activate specific ones
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1

! Adjust reference bandwidth (IMPORTANT for Gigabit+ links)
R1(config-router)# auto-cost reference-bandwidth 10000   ! 10 Gbps reference
! GigE cost = 10000/1000 = 10; 10GigE cost = 1

! Set cost on an interface directly
R1(config)# interface Gi0/0
R1(config-if)# ip ospf cost 50

! Advertise default route (on the ASBR/edge router)
R1(config-router)# default-information originate
```

**OSPF Interface-Level Configuration (preferred method):**
```
R1(config)# interface Gi0/0
R1(config-if)# ip ospf 1 area 0    ! Assign to OSPF process 1, area 0
```

**OSPF Authentication:**
```
! Interface-level MD5 authentication
R1(config)# interface Gi0/1
R1(config-if)# ip ospf authentication message-digest
R1(config-if)# ip ospf message-digest-key 1 md5 MyOSPFKey

! Area-level authentication
R1(config-router)# area 0 authentication message-digest
```

**OSPF Timers:**
```
! Default: Hello=10s, Dead=40s (broadcast/point-to-point)
!          Hello=30s, Dead=120s (NBMA)
R1(config-if)# ip ospf hello-interval 5
R1(config-if)# ip ospf dead-interval 20
! Must match on both ends!
```

**OSPF Network Types:**
| Type | Hello | DR/BDR | Example |
|------|-------|--------|---------|
| Broadcast | 10s | Yes | Ethernet |
| Point-to-Point | 10s | No | Serial, GRE tunnel |
| NBMA | 30s | Yes | Frame Relay (legacy) |
| Point-to-Multipoint | 30s | No | Hub-and-spoke |

**Verification:**
```
show ip ospf
show ip ospf neighbor
show ip ospf interface
show ip ospf interface brief
show ip ospf database
show ip route ospf
show ip protocols
```

---

### Day 35 — Multi-Area OSPF

**Why Multi-Area?**
- Reduces LSDB size per area
- Limits SPF recalculations to the affected area
- Reduces routing table size with summarization at ABRs

**Configuration (same commands, different area IDs):**
```
! ABR configuration — interfaces in different areas
R-ABR(config)# router ospf 1
R-ABR(config-router)# router-id 2.2.2.2
R-ABR(config-router)# network 192.168.10.0 0.0.0.255 area 0
R-ABR(config-router)# network 172.16.0.0 0.0.0.255 area 1

! Route summarization at ABR
R-ABR(config-router)# area 1 range 172.16.0.0 255.255.252.0
```

---

### Day 36 — EIGRP Overview (Comparison)

**EIGRP (Enhanced Interior Gateway Routing Protocol):**
- Cisco proprietary (now partially open via RFC 7868)
- Advanced distance-vector (hybrid) protocol
- AD = 90 (internal), 170 (external)
- Metric: Composite of bandwidth, delay, (optionally load, reliability)
- Uses DUAL algorithm for loop-free, fast convergence
- Supports unequal-cost load balancing (variance command)
- Maintains neighbor table, topology table, routing table

**EIGRP vs OSPF:**
| Feature | OSPF | EIGRP |
|---------|------|-------|
| Type | Link-state | Advanced distance-vector |
| Algorithm | Dijkstra (SPF) | DUAL |
| AD | 110 | 90 |
| Metric | Cost (bandwidth) | Composite (BW + delay) |
| Areas | Yes (hierarchical) | No (uses AS numbers) |
| Convergence | Fast (RSTP-like) | Very fast (feasible successors) |
| Standards | Open (IETF) | Cisco (mostly) |
| VLSM/CIDR | Yes | Yes |
| Multicast | 224.0.0.5/6 | 224.0.0.10 |
| Equal-cost LB | Yes (up to 4 paths default) | Yes (up to 4 default) |
| Unequal-cost LB | No | Yes (variance) |

**Basic EIGRP Configuration:**
```
R1(config)# router eigrp 100                ! AS number (must match neighbors)
R1(config-router)# network 192.168.10.0 0.0.0.255
R1(config-router)# no auto-summary          ! Disable classful summarization
R1(config-router)# passive-interface Gi0/0
```

---

### Day 37 — First Hop Redundancy Protocols (FHRP)

**Problem:** If the default gateway router fails, all hosts lose connectivity.

**HSRP (Hot Standby Router Protocol) — Cisco:**
- Active/Standby model
- Virtual IP + Virtual MAC (0000.0c07.acXX, XX = group#)
- Preemption optional
- Default timers: Hello 3s, Hold 10s

```
R1(config)# interface Gi0/0
R1(config-if)# standby 1 ip 192.168.10.1
R1(config-if)# standby 1 priority 110         ! Default is 100; higher = active
R1(config-if)# standby 1 preempt               ! Take over if priority is higher

R2(config)# interface Gi0/0
R2(config-if)# standby 1 ip 192.168.10.1
R2(config-if)# standby 1 priority 100
```

**VRRP (Virtual Router Redundancy Protocol) — Open Standard:**
- Master/Backup model
- Virtual MAC: 0000.5e00.01XX
- Preemption enabled by default
- IP owner concept (if router's real IP = virtual IP, it's always master)

**GLBP (Gateway Load Balancing Protocol) — Cisco:**
- Active Virtual Gateway (AVG) + Active Virtual Forwarders (AVF)
- Provides load balancing across multiple routers (not just failover)
- Up to 4 virtual MACs per group

**FHRP Comparison:**
| Feature | HSRP | VRRP | GLBP |
|---------|------|------|------|
| Standard | Cisco | IEEE | Cisco |
| Active/Standby | 1 active | 1 master | 1 AVG + up to 4 AVFs |
| Load Balancing | No (track objects only) | No | Yes |
| Preemption | Disabled by default | Enabled by default | Disabled by default |
| Virtual MAC | 0000.0c07.acXX | 0000.5e00.01XX | 0007.b400.XXYY |

---

### Days 38–42 — Labs & Review

**Lab 1: Static Routing**
- 3 routers in a line: R1 — R2 — R3
- Configure static routes so all networks are reachable
- Add a default route on R1 pointing to R2
- Add a floating static as backup

**Lab 2: Single-Area OSPF**
- 3 routers, all in Area 0
- Configure OSPF with router-id, network statements
- Set passive interfaces on LAN-facing ports
- Adjust reference bandwidth to 10000
- Verify neighbor adjacencies, LSDB, routing table

**Lab 3: Multi-Area OSPF**
- 4 routers: R1-R2 in Area 0, R2-R3 in Area 1, R2-R4 in Area 2
- R2 is the ABR
- Verify inter-area routes (Type 3 LSAs) on R3 and R4
- Configure route summarization at R2

**Lab 4: HSRP**
- Two routers sharing a virtual gateway IP
- Configure priorities and preemption
- Simulate primary failure; verify failover
- Verify with `show standby`

---

## 📝 Review Quiz — Weeks 5–6

1. What is the OSPF administrative distance?
2. What does "longest prefix match" mean?
3. What is the OSPF default Hello timer on a broadcast network?
4. What are the OSPF neighbor states in order?
5. What command advertises a default route in OSPF?
6. What is a floating static route?
7. What is the DR election based on?
8. Why should you adjust the OSPF reference bandwidth?
9. Name the three FHRP protocols discussed.
10. What is the EIGRP AD for internal routes?

<details>
<summary>Answers</summary>

1. 110
2. The most specific (longest subnet mask) matching route is used
3. 10 seconds
4. Down → Init → 2-Way → ExStart → Exchange → Loading → Full
5. `default-information originate`
6. A static route with a higher AD, used as backup when a dynamic route is unavailable
7. Highest OSPF priority, then highest Router ID
8. Default reference (100 Mbps) gives FastEthernet and GigabitEthernet the same cost (1)
9. HSRP, VRRP, GLBP
10. 90
</details>
