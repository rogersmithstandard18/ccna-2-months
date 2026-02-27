# Day 29: How Routing Works

## 🎯 What You'll Learn
How routers make forwarding decisions, the routing table structure, longest prefix match, administrative distance, and the complete packet forwarding process.

---

## What Does a Router Do?

A router connects **different networks** and forwards packets between them. While switches forward based on MAC addresses (Layer 2), routers forward based on **IP addresses (Layer 3)**.

```
Network A: 192.168.1.0/24        Network B: 192.168.2.0/24
[PC-A: .10] ─── [SW1] ─── [R1] ─── [SW2] ─── [PC-B: .10]
                        Gi0/0  Gi0/1
                       .1       .1
```

PC-A wants to reach PC-B:
1. PC-A sees that 192.168.2.10 is on a **different network** (ANDs its IP with mask)
2. PC-A sends the packet to its **default gateway** (192.168.1.1 = R1)
3. R1 receives the packet, looks at destination IP (192.168.2.10)
4. R1 checks its **routing table** → finds 192.168.2.0/24 on Gi0/1
5. R1 forwards the packet out Gi0/1
6. PC-B receives the packet

---

## The Routing Table

The routing table is the router's map of all known networks:

```
R1# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       * - candidate default

Gateway of last resort is 10.0.0.2 to network 0.0.0.0

      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.0/30 is directly connected, GigabitEthernet0/1
L        10.0.0.1/32 is directly connected, GigabitEthernet0/1
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0
O        192.168.2.0/24 [110/20] via 10.0.0.2, 00:05:30, GigabitEthernet0/1
S        192.168.3.0/24 [1/0] via 10.0.0.2
S*       0.0.0.0/0 [1/0] via 10.0.0.2
```

### Route Types

| Code | Type | How It Gets There |
|------|------|-------------------|
| **C** | Connected | Interface has an IP and is up/up — automatic |
| **L** | Local | /32 route for the router's own IP — automatic |
| **S** | Static | Manually configured by admin |
| **O** | OSPF | Learned from OSPF |
| **D** | EIGRP | Learned from EIGRP |
| **B** | BGP | Learned from BGP |
| **S*** | Default static | Gateway of last resort |

### Reading a Route Entry

```
O    192.168.2.0/24 [110/20] via 10.0.0.2, 00:05:30, Gi0/1
│         │            │  │       │          │          │
│         │            │  │       │          │          └─ Exit interface
│         │            │  │       │          └─ Route age (how long it's been known)
│         │            │  │       └─ Next-hop IP (send packets here)
│         │            │  └─ Metric (OSPF cost = 20)
│         │            └─ Administrative Distance (OSPF = 110)
│         └─ Destination network/prefix
└─ Route source (O = OSPF)
```

---

## The Routing Decision Process

When a packet arrives, the router follows this exact process:

```
1. Receive packet on an interface
2. Check: Is the destination IP for ME? (local route /32)
   → Yes: Process locally (management traffic, routing protocol)
   → No: Continue to step 3
3. Look up destination IP in the routing table
4. Find ALL matching routes
5. Select the BEST match using LONGEST PREFIX MATCH
6. If multiple routes with same prefix length:
   a. Compare Administrative Distance → lowest wins
   b. If AD is tied, compare Metric → lowest wins
   c. If metric is tied → equal-cost load balancing (ECMP)
7. Forward packet to the next-hop / exit interface
8. If NO match found → drop packet (unless default route exists)
```

---

## Longest Prefix Match — The Golden Rule

**The most specific (longest prefix) matching route ALWAYS wins.**

```
Routing table has:
  10.0.0.0/8        via 10.1.1.1
  10.1.0.0/16       via 10.2.2.2
  10.1.1.0/24       via 10.3.3.3
  10.1.1.128/25     via 10.4.4.4

Destination: 10.1.1.200

Which route wins?
  10.0.0.0/8      → matches ✓ (8 bits match)
  10.1.0.0/16     → matches ✓ (16 bits match)
  10.1.1.0/24     → matches ✓ (24 bits match)
  10.1.1.128/25   → matches ✓ (25 bits match) ← WINNER! Most specific!

Packet sent via 10.4.4.4
```

**Why does this matter?**
- Default route (0.0.0.0/0) matches everything but has the shortest prefix (0 bits)
- It's always the **last resort** — any more specific route wins
- This is why you can have a default route AND specific routes — specific routes take priority

> 💡 **Exam tip:** Longest prefix match questions are common. Always count the prefix length, not the metric or AD, when determining which route is used.

---

## Administrative Distance (AD)

When **different routing protocols** know about the same destination, AD determines which one is trusted more. **Lower AD = more trusted.**

| Route Source | AD | Memory Trick |
|-------------|-----|-------------|
| Connected | 0 | "I'm directly attached — ultimate trust" |
| Static | 1 | "The human told me — very trusted" |
| eBGP | 20 | "External BGP — trusted internet route" |
| EIGRP (internal) | 90 | "Cisco's baby — 90 for internal" |
| OSPF | 110 | "Open standard — 110" |
| IS-IS | 115 | "The other link-state — 115" |
| RIP | 120 | "Old and slow — 120" |
| EIGRP (external) | 170 | "EIGRP external — less trusted" |
| iBGP | 200 | "Internal BGP — 200" |
| Unknown | 255 | "Never use — unreachable" |

**Example:** If OSPF (AD 110) and EIGRP (AD 90) both know about 192.168.5.0/24, the router uses the **EIGRP route** because 90 < 110.

> 💡 **AD is only compared when the prefix length is the same.** A /25 OSPF route beats a /24 EIGRP route because longest prefix match happens FIRST.

---

## Metrics — Choosing Within a Protocol

When the **same routing protocol** has multiple paths to the same destination, the **metric** (cost) determines the best path. Lower metric = better.

| Protocol | Metric Based On |
|----------|----------------|
| OSPF | Cost (reference bandwidth / interface bandwidth) |
| EIGRP | Composite (bandwidth + delay, optionally load + reliability) |
| RIP | Hop count |
| BGP | Complex (AS path, local pref, MED, etc.) |

---

## Connected and Local Routes

When you configure an IP address on an interface and bring it up:

```
R1(config)# interface Gi0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
```

**Two routes automatically appear:**
```
C    192.168.1.0/24 is directly connected, GigabitEthernet0/0
L    192.168.1.1/32 is directly connected, GigabitEthernet0/0
```

- **C (Connected):** "I can reach the 192.168.1.0/24 network through Gi0/0"
- **L (Local):** "192.168.1.1 is MY interface — packets for this IP are for me"

**If the interface goes down, both routes disappear from the table.**

---

## Packet Forwarding in Detail

Let's trace a packet step by step:

```
PC-A (192.168.1.10) → PC-B (192.168.2.20)

Network:
[PC-A] ─── [SW1] ─── [R1] ─── [R2] ─── [SW2] ─── [PC-B]
192.168.1.0/24     Gi0/0  Gi0/1  Gi0/0  Gi0/1     192.168.2.0/24
                   .1     .1     .2     .1
                        10.0.0.0/30
```

**Step 1: PC-A prepares the packet**
```
Layer 3: Src IP: 192.168.1.10 → Dst IP: 192.168.2.20
Layer 2: Src MAC: PC-A_MAC → Dst MAC: R1_Gi0/0_MAC  (default gateway's MAC!)
```

PC-A knows 192.168.2.20 is on a different network, so it sends to the **default gateway**. It uses ARP to find R1's MAC address.

**Step 2: R1 receives and routes**
```
R1 strips the Layer 2 header
R1 reads destination IP: 192.168.2.20
R1 checks routing table: 192.168.2.0/24 via 10.0.0.2 (R2)
R1 creates new Layer 2 header:
  Src MAC: R1_Gi0/1_MAC → Dst MAC: R2_Gi0/0_MAC
Layer 3 unchanged: Src IP: 192.168.1.10 → Dst IP: 192.168.2.20
```

**Step 3: R2 receives and routes**
```
R2 strips the Layer 2 header
R2 reads destination IP: 192.168.2.20
R2 checks routing table: 192.168.2.0/24 is directly connected on Gi0/1
R2 creates new Layer 2 header:
  Src MAC: R2_Gi0/1_MAC → Dst MAC: PC-B_MAC (ARP for PC-B's MAC)
Layer 3 unchanged: Src IP: 192.168.1.10 → Dst IP: 192.168.2.20
```

**Key takeaway:**
- **IP addresses stay the same** end-to-end (192.168.1.10 → 192.168.2.20)
- **MAC addresses change at every hop** (router rewrites Layer 2)
- **TTL decrements by 1** at each router

---

## Equal-Cost Multi-Path (ECMP)

When a router has multiple routes to the same destination with the **same AD and same metric**, it can load-balance across them:

```
R1# show ip route 10.10.10.0
  O   10.10.10.0/24 [110/20] via 10.0.0.2, Gi0/1
                     [110/20] via 10.0.1.2, Gi0/2
```

Both routes have AD 110 and metric 20 → router sends traffic across **both paths**. Default: up to 4 equal-cost paths (configurable up to 32 in some platforms).

---

## Practice Questions

1. What is the longest prefix match rule?
2. OSPF knows about 10.0.0.0/24 (AD 110) and EIGRP knows about 10.0.0.0/24 (AD 90). Which route is used?
3. What route code means "directly connected"?
4. What happens if a packet doesn't match any route in the table?
5. Do IP addresses change as a packet crosses routers?
6. What is a /32 "local" route?
7. What is ECMP?

<details>
<summary>Answers</summary>

1. The most specific (longest prefix length) matching route is always selected, regardless of AD or metric
2. EIGRP (AD 90 < AD 110)
3. C (Connected)
4. The packet is dropped (unless a default route 0.0.0.0/0 exists)
5. No — IP addresses stay the same end-to-end; MAC addresses change at each hop
6. A route for the router's own interface IP address — identifies traffic destined for the router itself
7. Equal-Cost Multi-Path — load balancing across routes with the same AD and metric
</details>

---

*← [Days 22-23 — EtherChannel](Day22-23-EtherChannel.md) | [Day 30 — Static Routing](Day30-Static-Routing.md) →*
