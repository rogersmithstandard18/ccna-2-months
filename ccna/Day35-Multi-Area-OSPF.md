# Day 35: Multi-Area OSPF

## 🎯 What You'll Learn
Why OSPF needs areas, how ABRs work, route summarization between areas, stub areas, and verifying multi-area deployments.

---

## Why Multi-Area?

Single-area OSPF works fine for small networks. But as the network grows:

| Problem | Impact |
|---------|--------|
| Large LSDB | More memory used on every router |
| Full SPF recalculation | Any change triggers SPF on ALL routers in the area |
| Large routing table | Every router carries every route |

**Multi-area OSPF solves this** by dividing the network into smaller areas. Changes in Area 1 only trigger SPF in Area 1 — Area 2 routers are unaffected.

---

## Area 0 — The Backbone

**Every OSPF area MUST connect to Area 0** (the backbone area). This is a fundamental OSPF rule.

```
                   ┌────── Area 0 (Backbone) ──────┐
                   │     R1 ────── R2               │
                   │      │         │               │
                   └──────┼─────────┼───────────────┘
                          │         │
                   ┌──── Area 1 ───┐│┌──── Area 2 ────┐
                   │  R3 ──── R4   │││  R5 ──── R6    │
                   └───────────────┘│└─────────────────┘
                                    │
                          R2 is an ABR
                    (connects Area 0 to Area 2)
```

**If an area can't directly connect to Area 0**, you need a **virtual link** through a transit area (advanced topic, rarely tested).

---

## Router Types

| Type | Description | Where |
|------|-------------|-------|
| **Internal Router** | All interfaces in one area | Inside any single area |
| **Backbone Router** | At least one interface in Area 0 | Area 0 |
| **ABR** (Area Border Router) | Interfaces in multiple areas | Border of two+ areas |
| **ASBR** (AS Boundary Router) | Redistributes external routes into OSPF | Edge of OSPF domain |

A router can be multiple types simultaneously (e.g., ABR + Backbone Router).

---

## How Inter-Area Routing Works

1. **Inside Area 1:** Routers share Type 1 (Router) and Type 2 (Network) LSAs — full topology knowledge
2. **At the ABR:** ABR takes Area 1's routes and creates **Type 3 Summary LSAs** for Area 0
3. **In Area 0:** Type 3 LSAs propagate through the backbone
4. **At another ABR:** That ABR creates Type 3 LSAs for Area 2
5. **Inside Area 2:** Routers see inter-area routes as `O IA` in their routing table

```
R6# show ip route ospf
O IA  192.168.1.0/24 [110/30] via 10.0.0.5, Gi0/0   ← Inter-area route
O     10.0.2.0/30 [110/10] via 10.0.0.9, Gi0/1       ← Intra-area route
```

**Key insight:** Routers in Area 2 don't know the full topology of Area 1 — they just see summary routes through the ABR. This is by design.

---

## Route Summarization at ABRs

ABRs can summarize multiple routes from one area into a single advertisement for another area.

```
! Area 1 has these networks:
  172.16.0.0/24
  172.16.1.0/24
  172.16.2.0/24
  172.16.3.0/24

! Without summarization: 4 Type 3 LSAs sent to Area 0
! With summarization: 1 Type 3 LSA

ABR(config-router)# area 1 range 172.16.0.0 255.255.252.0
! Summarizes all four /24s into one /22
```

**Benefits:**
- Smaller routing tables in other areas
- Fewer LSAs = less SPF calculation
- More stable — a single subnet flap doesn't affect other areas

**Important:** Summarization only happens at ABRs (inter-area) and ASBRs (external). OSPF does NOT automatically summarize.

---

## Stub Areas

Stub areas reduce the routing table even further by blocking external (Type 5) LSAs and replacing them with a default route.

### Stub Area
```
! Configure on ALL routers in the stub area
R3(config-router)# area 1 stub
R4(config-router)# area 1 stub
ABR(config-router)# area 1 stub

! ABR automatically injects a default route (Type 3 LSA) into Area 1
```

**Effect:** Area 1 routers don't see any external routes — they use the default route through the ABR to reach external destinations.

### Totally Stubby Area (Cisco proprietary)
```
! On ABR only — add "no-summary"
ABR(config-router)# area 1 stub no-summary

! Other routers in the area: just "area 1 stub"
R3(config-router)# area 1 stub
```

**Effect:** Area 1 routers only see intra-area routes + one default route. No Type 3 (inter-area) or Type 5 (external) LSAs at all. Smallest possible routing table.

### NSSA (Not-So-Stubby Area)
```
ABR(config-router)# area 1 nssa
R3(config-router)# area 1 nssa
```

**Effect:** Like a stub area, but allows the ASBR inside the area to redistribute external routes using **Type 7 LSAs** (which the ABR converts to Type 5 for the rest of the OSPF domain).

**Use case:** A remote office that needs to redistribute a static route to a local ISP but doesn't need the full external routing table.

---

## Verification

```
show ip ospf                          ! Areas, router type
show ip ospf database                 ! Full LSDB
show ip ospf database summary         ! Type 3 LSAs
show ip ospf database external        ! Type 5 LSAs
show ip ospf border-routers           ! ABRs and ASBRs
show ip route ospf                    ! OSPF routes (O, O IA, O E1/E2)
show ip ospf interface brief          ! Which interfaces in which areas
```

**Route codes to know:**
```
O      = Intra-area (within the same area)
O IA   = Inter-area (from another area via ABR)
O E1   = External Type 1 (metric includes internal cost)
O E2   = External Type 2 (metric is only the external cost — default)
O N1   = NSSA Type 1
O N2   = NSSA Type 2
```

---

## Practice Questions

1. What must every OSPF area connect to?
2. What is an ABR?
3. What LSA type carries inter-area routes?
4. What command summarizes Area 1 routes at an ABR?
5. What's the difference between a stub area and a totally stubby area?
6. What does NSSA allow that a stub area doesn't?

<details>
<summary>Answers</summary>

1. Area 0 (the backbone)
2. A router with interfaces in two or more areas (Area Border Router)
3. Type 3 (Summary LSA)
4. `area 1 range <network> <mask>` (under `router ospf`)
5. Stub blocks Type 5 (external) LSAs. Totally Stubby also blocks Type 3 (inter-area) LSAs — only a default route remains.
6. NSSA allows an ASBR within the area to redistribute external routes (using Type 7 LSAs), while stub areas block all external routes.
</details>

---

*← [Days 33-34 — OSPF Configuration](Day33-34-OSPF-Configuration.md) | [Day 36 — EIGRP Overview](Day36-EIGRP-Overview.md) →*
