# Days 33-34: OSPF Configuration

## 🎯 What You'll Learn
How to configure single-area and multi-area OSPF, passive interfaces, authentication, timers, default route advertisement, and verification.

---

## Single-Area OSPF Configuration

### Method 1: Network Statement (Traditional)

```
R1(config)# router ospf 1
! Process ID = 1 (locally significant — doesn't need to match other routers)

R1(config-router)# router-id 1.1.1.1
! Always set manually — prevents surprises

R1(config-router)# network 192.168.1.0 0.0.0.255 area 0
! "Enable OSPF on interfaces matching 192.168.1.x and put them in Area 0"

R1(config-router)# network 10.0.0.0 0.0.0.3 area 0
! "Enable OSPF on interfaces matching 10.0.0.0-10.0.0.3"
```

**The wildcard mask in the network statement** is NOT a subnet mask — it tells the router which interfaces to activate OSPF on.

```
network 0.0.0.0 255.255.255.255 area 0
! Wildcard 255.255.255.255 = "match ANY IP" = enable OSPF on ALL interfaces
! Quick and dirty — fine for labs, not for production
```

### Method 2: Interface-Level (Preferred)

```
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1

R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip ospf 1 area 0
! Directly enables OSPF process 1, area 0 on this interface

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip ospf 1 area 0
```

**Why prefer interface-level?**
- Explicit — you see exactly which interfaces are OSPF-enabled
- No wildcard mask confusion
- Easier to audit and troubleshoot

---

## Passive Interfaces

A **passive interface** advertises its network in OSPF but does **not send Hello packets** on that interface. Use it on LAN-facing interfaces where no OSPF neighbor exists.

```
! Make one interface passive
R1(config-router)# passive-interface GigabitEthernet0/0
! OSPF still advertises 192.168.1.0/24 but won't try to form neighbors on Gi0/0

! Make ALL interfaces passive, then selectively activate
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1
! Only Gi0/1 will send Hellos (the WAN link to another router)
```

**Why use passive interfaces?**
- Security: Don't send OSPF Hellos to user networks (prevents rogue OSPF routers)
- Efficiency: No Hellos on interfaces with no OSPF neighbors
- The network is still advertised — hosts can still be reached

---

## Reference Bandwidth

```
R1(config-router)# auto-cost reference-bandwidth 10000
! Set on EVERY router in the OSPF domain
! 10000 = 10 Gbps
! Now: GigE = cost 10, FastEthernet = cost 100, 10GigE = cost 1
```

**Or set cost directly on an interface:**
```
R1(config)# interface Gi0/0
R1(config-if)# ip ospf cost 50
! Overrides the calculated cost
```

---

## Default Route Advertisement

Make one router (the edge/ASBR) advertise a default route to all other OSPF routers:

```
! First, create the default route on the edge router
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1

! Then tell OSPF to advertise it
R1(config-router)# default-information originate
! Only advertises if R1 actually HAS a default route in its table

! To advertise even without a default route in the table:
R1(config-router)# default-information originate always
```

All other OSPF routers will receive:
```
O*E2  0.0.0.0/0 [110/1] via 10.0.0.1, 00:01:00, GigabitEthernet0/0
```

---

## OSPF Authentication

### Interface-Level MD5 Authentication
```
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip ospf authentication message-digest
R1(config-if)# ip ospf message-digest-key 1 md5 MySecretKey

! Both sides must have the same key number and password!
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip ospf authentication message-digest
R2(config-if)# ip ospf message-digest-key 1 md5 MySecretKey
```

### Area-Level Authentication
```
R1(config-router)# area 0 authentication message-digest
! All interfaces in area 0 require authentication
! Still need the key on each interface:
R1(config-if)# ip ospf message-digest-key 1 md5 MySecretKey
```

---

## OSPF Timers

```
! Default: Hello=10s, Dead=40s (broadcast/P2P)
! Change timers (must match on both sides!)
R1(config-if)# ip ospf hello-interval 5
R1(config-if)# ip ospf dead-interval 20

! Fast hellos (sub-second failover)
R1(config-if)# ip ospf dead-interval minimal hello-multiplier 4
! Dead=1 second, Hello=250ms (4 per second)
```

---

## Multi-Area OSPF

```
! R2 is an ABR — interfaces in Area 0 and Area 1
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2

R2(config)# interface Gi0/0
R2(config-if)# ip ospf 1 area 0        ! Backbone

R2(config)# interface Gi0/1
R2(config-if)# ip ospf 1 area 1        ! Area 1

! Route summarization at the ABR
R2(config-router)# area 1 range 172.16.0.0 255.255.252.0
! Summarizes all Area 1 routes into one Type 3 LSA for Area 0
```

---

## OSPFv3 (OSPF for IPv6)

```
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 router ospf 1
R1(config-rtr)# router-id 1.1.1.1       ! Still an IPv4-format ID

R1(config)# interface Gi0/0
R1(config-if)# ipv6 ospf 1 area 0
```

Key difference: OSPFv3 uses **link-local addresses** for neighbor communication (not global unicast).

---

## Complete Configuration Example

```
! ═══════════════════════════════════════════
!  R1 — Edge Router (connects to ISP)
! ═══════════════════════════════════════════
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# auto-cost reference-bandwidth 10000
R1(config-router)# passive-interface default
R1(config-router)# no passive-interface GigabitEthernet0/1
R1(config-router)# default-information originate

R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255

R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# ip ospf 1 area 0
R1(config-if)# description LAN

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 10.0.0.1 255.255.255.252
R1(config-if)# ip ospf 1 area 0
R1(config-if)# ip ospf network point-to-point
R1(config-if)# ip ospf authentication message-digest
R1(config-if)# ip ospf message-digest-key 1 md5 OSPF_Key1
R1(config-if)# description WAN to R2

R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
! Default route to ISP
```

---

## Verification Commands

```
show ip ospf                        ! OSPF process info, Router ID, areas
show ip ospf neighbor               ! Neighbor table — the first thing to check!
show ip ospf interface              ! Per-interface OSPF details
show ip ospf interface brief        ! Quick summary
show ip ospf database               ! The LSDB — all LSAs
show ip route ospf                  ! Only OSPF-learned routes
show ip protocols                   ! OSPF settings, networks, passive interfaces
```

### Reading `show ip ospf neighbor`
```
R1# show ip ospf neighbor

Neighbor ID  Pri  State           Dead Time  Address       Interface
2.2.2.2        1  FULL/DR         00:00:35   10.0.0.2      Gi0/1
3.3.3.3        1  FULL/BDR        00:00:33   10.0.0.6      Gi0/2
4.4.4.4        1  2WAY/DROTHER    00:00:38   10.0.0.10     Gi0/2
```

**What to look for:**
- **FULL** = adjacency complete ✅
- **2WAY** = normal for DROther↔DROther ✅
- **INIT, EXSTART, EXCHANGE, LOADING** = stuck (troubleshoot!) ❌
- **Dead Time** counting down = Hellos still arriving ✅

---

## Troubleshooting Checklist

| Symptom | Check |
|---------|-------|
| No neighbors at all | Interface up? OSPF enabled? Same area/subnet? Hello/Dead match? |
| Stuck at Init | One-way communication — ACL? Interface problem? |
| Stuck at ExStart/Exchange | MTU mismatch! Check `ip mtu` on both sides |
| Stuck at Loading | LSDB corruption; clear OSPF process |
| Routes missing | Passive interface on the wrong port? Wrong area? ACL on interface? |
| Suboptimal path | Check costs — probably need `auto-cost reference-bandwidth` |

```
! Nuclear option — restart OSPF process (brief outage)
R1# clear ip ospf process
Reset ALL OSPF processes? [no]: yes
```

---

## Practice Questions

1. What command enables OSPF directly on an interface?
2. What does `passive-interface` do in OSPF?
3. What command advertises a default route in OSPF?
4. What happens if reference bandwidth differs between routers?
5. What must match on both sides for OSPF authentication?
6. Where do you configure route summarization in multi-area OSPF?

<details>
<summary>Answers</summary>

1. `ip ospf <process-id> area <area-id>` (interface config mode)
2. Advertises the network but doesn't send Hello packets (no neighbor formation on that interface)
3. `default-information originate` (under `router ospf`)
4. Cost calculations will be inconsistent, leading to suboptimal routing
5. Authentication type, key number, and key string (password)
6. On the ABR: `area <area-id> range <network> <mask>` (under `router ospf`)
</details>

---

*← [Days 31-32 — OSPF Concepts](Day31-32-OSPF-Concepts.md) | [Day 35 — Multi-Area OSPF](Day35-Multi-Area-OSPF.md) →*
