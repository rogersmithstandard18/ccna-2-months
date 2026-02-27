# Day 36: EIGRP Overview

## 🎯 What You'll Learn
EIGRP basics, how it compares to OSPF, the DUAL algorithm, feasible successors, and basic configuration. CCNA focuses on comparison — OSPF is the deep-dive protocol.

---

## What Is EIGRP?

**Enhanced Interior Gateway Routing Protocol** — Cisco's advanced distance-vector (hybrid) protocol.

- Originally Cisco proprietary; partially opened via RFC 7868
- AD = **90** (internal), **170** (external)
- Uses **DUAL** (Diffusing Update Algorithm) for loop-free, fast convergence
- Supports **unequal-cost load balancing** (unique among common protocols)
- Classless (VLSM/CIDR support)
- Multicast: **224.0.0.10**

---

## EIGRP vs OSPF — Side by Side

| Feature | OSPF | EIGRP |
|---------|------|-------|
| Type | Link-state | Advanced distance-vector |
| Algorithm | Dijkstra (SPF) | DUAL |
| AD | 110 | 90 (internal) |
| Metric | Cost (bandwidth) | Composite (bandwidth + delay) |
| Hierarchy | Areas (backbone required) | No areas (flat, uses AS numbers) |
| Convergence | Fast | Very fast (feasible successors) |
| Standard | Open (IETF) | Cisco (mostly) |
| Equal-cost LB | Yes (up to 4 paths, configurable) | Yes (up to 4, configurable) |
| Unequal-cost LB | ❌ No | ✅ Yes (variance command) |
| Multicast | 224.0.0.5 / .6 | 224.0.0.10 |
| Protocol # | 89 | 88 |

---

## EIGRP Key Concepts

### Three Tables

| Table | Content |
|-------|---------|
| **Neighbor Table** | All EIGRP neighbors (like OSPF's neighbor table) |
| **Topology Table** | All routes learned from neighbors, with metrics (like OSPF's LSDB) |
| **Routing Table** | Best routes chosen from topology table (successor routes) |

### Successor and Feasible Successor

**Successor:** The best route to a destination (lowest metric). This goes into the routing table.

**Feasible Successor (FS):** A backup route that's guaranteed to be loop-free. Stored in the topology table, ready for instant failover.

**Feasibility Condition:** A route is a feasible successor if its **Reported Distance (RD)** is less than the current **Feasible Distance (FD)**.

```
  FD = My total metric to the destination (via the successor)
  RD = The neighbor's metric to the destination (what they report to me)

  If RD < FD → loop-free backup → Feasible Successor ✅
  If RD ≥ FD → might be a loop → NOT a feasible successor ❌
```

**Why is this fast?** When the successor route fails, EIGRP immediately installs the feasible successor — no recalculation needed. Convergence in **sub-second** timeframes.

```
Example:
  Route to 10.0.0.0/24:
    Via R2: FD = 30 (successor — best path)
    Via R3: RD = 25 (their cost to destination)
            25 < 30? YES → Feasible Successor ✅
    Via R4: RD = 35
            35 < 30? NO → Not a feasible successor ❌

  If R2 link fails → R3 takes over instantly (no DUAL calculation)
  If R2 and R3 fail → DUAL must recalculate (queries neighbors)
```

---

## EIGRP Metric

EIGRP uses a **composite metric** based on:
- **Bandwidth** (minimum bandwidth along the path)
- **Delay** (cumulative delay along the path)
- Load and Reliability (available but NOT used by default — K values)

```
Default metric formula (simplified):
Metric = 256 × (10^7 / minimum_bandwidth_kbps + cumulative_delay/10)
```

**K Values (weights):**
```
K1 = 1 (Bandwidth)    ← Used by default
K2 = 0 (Load)         ← NOT used by default
K3 = 1 (Delay)        ← Used by default
K4 = 0 (Reliability)  ← NOT used by default
K5 = 0 (MTU)          ← NOT used by default
```

> 💡 **Exam tip:** Know that EIGRP uses bandwidth and delay by default. Don't memorize the full formula — just understand the concept.

---

## Basic Configuration

```
R1(config)# router eigrp 100
! AS number = 100 (MUST match all neighbors!)

R1(config-router)# eigrp router-id 1.1.1.1

R1(config-router)# network 192.168.1.0 0.0.0.255
R1(config-router)# network 10.0.0.0 0.0.0.3

R1(config-router)# no auto-summary
! CRITICAL — disables classful summarization
! Without this, EIGRP summarizes to classful boundaries (10.0.0.0/8)

R1(config-router)# passive-interface GigabitEthernet0/0
```

### Unequal-Cost Load Balancing

```
R1(config-router)# variance 2
! Allows routes with metric up to 2× the successor's metric
! to be used for load balancing (if they're feasible successors)
```

Example: Successor metric = 1000, variance = 2
- Route with metric 1500 → 1500 < 2000 (2×1000) → included ✅
- Route with metric 2500 → 2500 > 2000 → excluded ❌

---

## Verification

```
show ip eigrp neighbors          ! Neighbor table
show ip eigrp topology           ! Topology table (successors + FSs)
show ip eigrp topology all-links ! ALL routes, including non-feasible
show ip route eigrp              ! Only EIGRP routes in routing table
show ip protocols                ! EIGRP settings, AS, K values, networks
```

---

## EIGRP Named Mode (Modern)

```
R1(config)# router eigrp MYNET
R1(config-router)# address-family ipv4 unicast autonomous-system 100
R1(config-router-af)# eigrp router-id 1.1.1.1
R1(config-router-af)# network 192.168.1.0 0.0.0.255
R1(config-router-af)# af-interface GigabitEthernet0/0
R1(config-router-af-interface)# passive-interface
```

---

## CCNA Focus — What to Know

For the CCNA, you need to:
1. **Compare EIGRP to OSPF** (table above)
2. Know EIGRP's **AD values** (90 internal, 170 external)
3. Understand **successor vs feasible successor**
4. Know the **metric is based on bandwidth + delay**
5. Know EIGRP supports **unequal-cost load balancing**
6. Recognize basic configuration and verification commands

You do NOT need to calculate EIGRP metrics or deeply configure it — OSPF is the primary focus.

---

## Practice Questions

1. What is EIGRP's AD for internal routes?
2. What makes a route a feasible successor?
3. What is the EIGRP multicast address?
4. What does the `variance` command do?
5. Why must you configure `no auto-summary`?
6. What two metric components does EIGRP use by default?

<details>
<summary>Answers</summary>

1. 90
2. Its Reported Distance (RD) must be less than the current Feasible Distance (FD)
3. 224.0.0.10
4. Allows unequal-cost load balancing by including feasible successor routes with metrics up to (variance × successor metric)
5. Without it, EIGRP summarizes routes to classful boundaries (e.g., 10.1.1.0/24 becomes 10.0.0.0/8), causing routing problems
6. Bandwidth (minimum along path) and Delay (cumulative)
</details>

---

*← [Day 35 — Multi-Area OSPF](Day35-Multi-Area-OSPF.md) | [Day 37 — FHRP](Day37-FHRP.md) →*
