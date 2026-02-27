# Days 31-32: OSPF Concepts

## 🎯 What You'll Learn
How OSPF works — neighbor discovery, the link-state database, SPF algorithm, DR/BDR election, area types, and LSA types. This is the most heavily tested routing protocol on the CCNA.

---

## What Is OSPF?

**Open Shortest Path First** — a link-state routing protocol that:
- Builds a complete map (topology) of the network
- Uses Dijkstra's SPF algorithm to find the shortest path to every destination
- Converges fast when changes occur
- Supports VLSM and CIDR (classless)
- Is an open standard (IETF) — works across vendors
- AD = **110**

**Link-state vs Distance-vector:**
| Feature | Link-State (OSPF) | Distance-Vector (RIP) |
|---------|-------------------|----------------------|
| Knowledge | Full topology map | Only knows next-hop and metric |
| Updates | Triggered (only on changes) | Periodic (every 30s for RIP) |
| Convergence | Fast | Slow |
| CPU/Memory | Higher | Lower |
| Loop prevention | Built-in (SPF algorithm) | Split horizon, hold-down timers |

---

## OSPF Metric: Cost

```
Cost = Reference Bandwidth / Interface Bandwidth
Default reference: 100 Mbps (100,000,000 bps)
```

| Interface | Bandwidth | Cost |
|-----------|-----------|------|
| Serial (T1) | 1.544 Mbps | 64 |
| FastEthernet | 100 Mbps | 1 |
| GigabitEthernet | 1 Gbps | 1 ← Problem! |
| 10 GigabitEthernet | 10 Gbps | 1 ← Same cost! |

**The problem:** With the default reference of 100 Mbps, everything ≥100 Mbps gets cost 1. A FastEthernet path looks identical to a 10GigE path.

**The fix:** Change the reference bandwidth:
```
R1(config-router)# auto-cost reference-bandwidth 10000
! 10000 Mbps = 10 Gbps reference

New costs:
  FastEthernet: 10000/100 = 100
  GigabitEthernet: 10000/1000 = 10
  10GigE: 10000/10000 = 1
```

> ⚠️ **Set the same reference bandwidth on ALL OSPF routers** in your network, or cost calculations will be inconsistent.

**Total path cost:** Sum of all link costs from source to destination. OSPF picks the path with the **lowest total cost**.

---

## OSPF Packets

| Type | Name | Purpose |
|------|------|---------|
| 1 | **Hello** | Discover and maintain neighbors |
| 2 | **DBD** (Database Description) | Exchange summaries of LSDB contents |
| 3 | **LSR** (Link-State Request) | Request specific LSAs the router is missing |
| 4 | **LSU** (Link-State Update) | Send the requested LSAs |
| 5 | **LSAck** | Acknowledge received LSAs |

**OSPF uses multicast:**
- **224.0.0.5** — All OSPF routers
- **224.0.0.6** — All OSPF DR/BDR routers

---

## Neighbor Discovery and Adjacency

### The Hello Protocol

OSPF routers discover each other by sending **Hello packets** on each OSPF-enabled interface.

**Hello packet contains:**
- Router ID
- Hello/Dead intervals
- Area ID
- Network mask
- Authentication data
- Neighbor list (Router IDs of known neighbors)
- DR/BDR (on multi-access networks)

### What Must Match to Become Neighbors

If ANY of these don't match, neighbors **will not form**:

| Parameter | Must Match? | Default |
|-----------|------------|---------|
| Hello interval | ✅ | 10s (broadcast), 30s (NBMA) |
| Dead interval | ✅ | 4× Hello (40s or 120s) |
| Area ID | ✅ | — |
| Subnet/mask | ✅ | — |
| Authentication | ✅ | None |
| Stub area flag | ✅ | — |
| MTU* | Must match for DBD exchange | 1500 |

*MTU mismatch won't prevent Hello exchange but will stall adjacency at ExStart/Exchange.

### Neighbor States — The Journey to Full

```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```

| State | What's Happening |
|-------|-----------------|
| **Down** | No Hellos received from this neighbor |
| **Init** | Hello received, but my Router ID isn't in their neighbor list yet |
| **2-Way** | Both routers see each other in Hello packets. **DR/BDR election happens here** (on multi-access networks). If neither DR nor BDR, stops at 2-Way. |
| **ExStart** | Master/slave negotiation for DBD exchange (higher Router ID = master) |
| **Exchange** | Exchanging DBD packets (summaries of their LSDBs) |
| **Loading** | Requesting (LSR) and receiving (LSU) missing LSAs |
| **Full** | Databases are synchronized. **Adjacency formed!** |

> 💡 **Stuck at certain states? Common causes:**
> - Stuck at **Init**: One-way communication (ACL blocking, interface issue)
> - Stuck at **2-Way**: Normal for DROther↔DROther (they only form full adjacency with DR/BDR)
> - Stuck at **ExStart/Exchange**: MTU mismatch!
> - Stuck at **Loading**: LSA database issues

---

## DR/BDR Election

On **multi-access networks** (like Ethernet), if every router formed a full adjacency with every other router, there'd be O(n²) adjacencies and massive LSA flooding.

**Solution:** Elect a **Designated Router (DR)** and **Backup DR (BDR)**. All other routers (DROthers) only form full adjacencies with the DR and BDR.

```
         [DR]
        / | \
       /  |  \
   [BDR] [R3] [R4]
   
DR  ↔ BDR:    Full adjacency ✅
DR  ↔ R3:     Full adjacency ✅
DR  ↔ R4:     Full adjacency ✅
BDR ↔ R3:     Full adjacency ✅
BDR ↔ R4:     Full adjacency ✅
R3  ↔ R4:     2-Way ONLY (not full) — saves resources
```

### Election Rules

1. **Highest OSPF priority** wins (default: 1, range: 0-255)
2. If priority is tied → **Highest Router ID** wins
3. **Priority 0 = cannot be DR or BDR** (guaranteed DROther)
4. **Election is NON-PREEMPTIVE** — if a router with higher priority joins later, it does NOT take over. Only when the DR fails does the BDR become DR and a new BDR is elected.

```
! Set OSPF priority
R1(config)# interface Gi0/0
R1(config-if)# ip ospf priority 255     ! Highest — will be DR (if first)
R1(config-if)# ip ospf priority 0       ! Never DR/BDR
```

### Router ID Selection

OSPF Router ID is chosen (in order):
1. Manually configured: `router-id 1.1.1.1`
2. Highest loopback interface IP
3. Highest active physical interface IP

> 💡 **Always set the Router ID manually.** It prevents unexpected changes when interfaces go up/down.

---

## OSPF Areas

Large OSPF networks are divided into **areas** to reduce LSDB size and limit SPF recalculation scope.

```
         ┌──── Area 1 ────┐
         │  R3 ─── R4     │
         │       |        │
         └───── ABR ──────┘
                 │
         ┌── Area 0 ──────┐
         │  R1 ─── R2     │  ← Backbone (ALL areas must connect here)
         └───── ABR ──────┘
                 │
         ┌──── Area 2 ────┐
         │  R5 ─── R6     │
         └────────────────┘
```

**Rules:**
- **Area 0 is the backbone** — every other area must connect to it
- **ABR (Area Border Router):** Has interfaces in multiple areas; summarizes between them
- **ASBR (Autonomous System Boundary Router):** Redistributes routes from non-OSPF sources

### Area Types

| Area Type | External Routes (Type 5) | Inter-Area (Type 3) | Default Route |
|-----------|------------------------|---------------------|---------------|
| Normal | ✅ Allowed | ✅ Allowed | Optional |
| **Stub** | ❌ Blocked | ✅ Allowed | ✅ Injected by ABR |
| **Totally Stubby** | ❌ Blocked | ❌ Blocked (except default) | ✅ Injected |
| **NSSA** | Via Type 7 LSAs | ✅ Allowed | Optional |
| **Totally NSSA** | Via Type 7 LSAs | ❌ Blocked | ✅ Injected |

**When to use:**
- **Stub:** Remote area that doesn't need external routes (uses default route instead)
- **Totally Stubby:** Even smaller routing table — only default route from ABR
- **NSSA:** Stub-like but needs to redistribute a local routing source (e.g., a static route to a local ISP)

---

## OSPF LSA Types

| Type | Name | Created By | Scope | Purpose |
|------|------|-----------|-------|---------|
| 1 | Router LSA | Every router | Within area | Describes router's links and costs |
| 2 | Network LSA | DR | Within area | Lists routers on a multi-access segment |
| 3 | Summary LSA | ABR | Between areas | Advertises routes from other areas |
| 4 | ASBR Summary | ABR | Between areas | Points to the ASBR |
| 5 | External LSA | ASBR | Entire domain | Routes from outside OSPF |
| 7 | NSSA External | ASBR in NSSA | NSSA only | External routes in NSSA (converted to Type 5 at ABR) |

> 💡 **Exam focus:** Know Types 1, 2, 3, and 5. Type 1 is per-router, Type 2 is per-segment (DR), Type 3 is inter-area, Type 5 is external.

---

## OSPF Network Types

| Type | Hello | Dead | DR/BDR | Example |
|------|-------|------|--------|---------|
| Broadcast | 10s | 40s | Yes | Ethernet |
| Point-to-Point | 10s | 40s | No | Serial, GRE, P2P Ethernet |
| NBMA | 30s | 120s | Yes | Frame Relay (legacy) |
| Point-to-Multipoint | 30s | 120s | No | Hub-and-spoke |

```
! Change network type (useful for Ethernet P2P links)
R1(config-if)# ip ospf network point-to-point
! No DR/BDR election, faster adjacency
```

---

## Practice Questions

1. What algorithm does OSPF use?
2. What is the default OSPF Hello interval on Ethernet?
3. What multicast address do all OSPF routers listen on?
4. What state do DROther routers stay at with each other?
5. Is DR election preemptive?
6. What is the default OSPF reference bandwidth?
7. Name three things that must match for OSPF neighbors to form.
8. What LSA type is created by the DR?

<details>
<summary>Answers</summary>

1. Dijkstra's SPF (Shortest Path First) algorithm
2. 10 seconds (Dead = 40 seconds)
3. 224.0.0.5
4. 2-Way (they form full adjacency only with DR and BDR)
5. No — a new router with higher priority won't become DR until the current DR fails
6. 100 Mbps (100,000 kbps)
7. Hello/Dead intervals, Area ID, subnet mask, authentication, stub area flag (any three)
8. Type 2 (Network LSA)
</details>

---

*← [Day 30 — Static Routing](Day30-Static-Routing.md) | [Days 33-34 — OSPF Configuration](Day33-34-OSPF-Configuration.md) →*
