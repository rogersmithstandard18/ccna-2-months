# Day 6: Subnetting Part 2 — VLSM & Route Summarization

## 🎯 What You'll Learn
Variable Length Subnet Masking (VLSM) for efficient IP allocation, route summarization (supernetting), and real-world network design with subnetting.

---

## The Problem VLSM Solves

With fixed-length subnetting, every subnet is the same size:
```
192.168.1.0/24 → split into /26s:
  Each subnet: 62 hosts

But what if you need:
  LAN A: 50 users  → /26 works (62 hosts) ✅ but wastes 12
  LAN B: 20 users  → /26 works (62 hosts) ❌ wastes 42!
  WAN link: 2 IPs  → /26 works (62 hosts) ❌ wastes 60!
```

**VLSM = use different subnet sizes for different needs.**

---

## VLSM Process

**Rule #1:** Always allocate the LARGEST subnet first, then fill in smaller ones in the remaining space.

**Rule #2:** Each subnet must start on a valid boundary (multiple of its block size).

### Full Example

**Given:** 192.168.1.0/24
**Requirements:**
- LAN A: 50 hosts
- LAN B: 25 hosts  
- LAN C: 10 hosts
- WAN link 1: 2 hosts
- WAN link 2: 2 hosts

**Step 1: Sort by size (largest first)**
```
LAN A: 50 hosts → needs /26 (62 usable) — block size 64
LAN B: 25 hosts → needs /27 (30 usable) — block size 32
LAN C: 10 hosts → needs /28 (14 usable) — block size 16
WAN 1: 2 hosts  → needs /30 (2 usable)  — block size 4
WAN 2: 2 hosts  → needs /30 (2 usable)  — block size 4
```

**Step 2: Allocate sequentially**
```
LAN A:  192.168.1.0/26    (192.168.1.0 – 192.168.1.63)     62 hosts ✅
LAN B:  192.168.1.64/27   (192.168.1.64 – 192.168.1.95)    30 hosts ✅
LAN C:  192.168.1.96/28   (192.168.1.96 – 192.168.1.111)   14 hosts ✅
WAN 1:  192.168.1.112/30  (192.168.1.112 – 192.168.1.115)  2 hosts  ✅
WAN 2:  192.168.1.116/30  (192.168.1.116 – 192.168.1.119)  2 hosts  ✅

Remaining: 192.168.1.120 – 192.168.1.255 (available for future use!)
```

**Visual map:**
```
192.168.1.0                                                    192.168.1.255
├── LAN A /26 ──────┤── LAN B /27 ─┤── C /28 ┤W1┤W2┤── free ──────────┤
0                  63|64           95|96    111|  |  |120            255
```

**Without VLSM (fixed /26):**
```
We'd need 5 × /26 = 5 × 64 = 320 addresses
But we only have 256! Not enough!

With VLSM:
64 + 32 + 16 + 4 + 4 = 120 addresses
136 addresses saved for future growth!
```

---

## VLSM Practice Problem

**Given:** 10.0.0.0/24
**Requirements:**
- Engineering: 100 hosts
- Sales: 50 hosts
- Management: 20 hosts
- Server farm: 5 hosts
- Point-to-point link: 2 hosts

Solve it yourself, then check:

<details>
<summary>Solution</summary>

**Sort largest first:**
```
Engineering: 100 hosts → /25 (126 usable) — block 128
Sales: 50 hosts → /26 (62 usable) — block 64
Management: 20 hosts → /27 (30 usable) — block 32
Server farm: 5 hosts → /29 (6 usable) — block 8
P2P link: 2 hosts → /30 (2 usable) — block 4
```

**Allocate:**
```
Engineering:  10.0.0.0/25     (10.0.0.0 – 10.0.0.127)     126 hosts
Sales:        10.0.0.128/26   (10.0.0.128 – 10.0.0.191)   62 hosts
Management:   10.0.0.192/27   (10.0.0.192 – 10.0.0.223)   30 hosts
Server farm:  10.0.0.224/29   (10.0.0.224 – 10.0.0.231)   6 hosts
P2P link:     10.0.0.232/30   (10.0.0.232 – 10.0.0.235)   2 hosts

Remaining: 10.0.0.236 – 10.0.0.255 (20 addresses free)
Total used: 236 of 256 — 92% efficient!
```
</details>

---

## Route Summarization (Supernetting)

The **opposite** of subnetting. Combine multiple smaller networks into one larger route advertisement. This reduces routing table size.

### How to Summarize

**Step 1:** List all networks in binary
**Step 2:** Find the common bits from the left
**Step 3:** The summary = those common bits, with the rest as 0s. Mask = number of common bits.

### Example: Summarize these networks

```
192.168.0.0/24
192.168.1.0/24
192.168.2.0/24
192.168.3.0/24
```

**Binary (third octet):**
```
0 = 00000000
1 = 00000001
2 = 00000010
3 = 00000011
    ^^^^^^── first 6 bits are identical
          ^^── these 2 bits differ
```

**Common bits:** 22 (16 from first two octets + 6 from third)
**Summary route:** 192.168.0.0/22

**Verification:**
- 192.168.0.0/22 covers 192.168.0.0 – 192.168.3.255 ✅
- That's exactly our four networks

### Another Example

Summarize:
```
10.0.16.0/24
10.0.17.0/24
10.0.18.0/24
10.0.19.0/24
10.0.20.0/24
10.0.21.0/24
10.0.22.0/24
10.0.23.0/24
```

**Third octet in binary:**
```
16 = 00010000
17 = 00010001
18 = 00010010
19 = 00010011
20 = 00010100
21 = 00010101
22 = 00010110
23 = 00010111
     ^^^^^─── first 5 bits identical in third octet
```

**Common bits:** 16 + 5 = 21
**Summary:** 10.0.16.0/21

**Verification:** 10.0.16.0/21 covers 10.0.16.0 – 10.0.23.255 ✅

---

## When Summarization Doesn't Work Cleanly

Sometimes networks don't summarize perfectly:

```
192.168.1.0/24
192.168.2.0/24
192.168.3.0/24
```

Binary (third octet):
```
1 = 00000001
2 = 00000010
3 = 00000011
    ^^^^^^── only 6 bits match
```

Summary: 192.168.0.0/22 — but this includes 192.168.0.0/24, which wasn't in our original list!

**This is called "covering" an extra network.** It's often acceptable (that network may not exist), but be aware of it. You might need two summary routes instead:
```
192.168.2.0/23  (covers .2 and .3)
192.168.1.0/24  (covers .1)
```

---

## Design Scenario — Putting It All Together

**You're designing a network for a small company:**

```
Main Office:
  - Sales: 45 users
  - Engineering: 80 users
  - Management: 12 users
  - Servers: 6 servers
  
Branch Office:
  - Staff: 25 users
  - Servers: 3 servers

WAN links:
  - Main ↔ Branch: 1 link
  - Main ↔ Internet: 1 link

Given: 172.16.0.0/16
```

**Design solution:**
```
Engineering:  172.16.0.0/25    (126 hosts)  — room to grow
Sales:        172.16.0.128/26  (62 hosts)   — room to grow
Branch Staff: 172.16.0.192/27  (30 hosts)   — room to grow
Management:   172.16.1.0/28    (14 hosts)   — room to grow
Main Servers: 172.16.1.16/29   (6 hosts)
Branch Srvrs: 172.16.1.24/29   (6 hosts)
WAN 1:        172.16.1.32/30   (2 hosts)
WAN 2:        172.16.1.36/30   (2 hosts)

Total used: ~320 addresses out of 65,536 — plenty of room!
```

**Good design practices:**
- Leave room for growth (don't use the tightest possible mask)
- Use /30 for point-to-point WAN links (or /31 per RFC 3021)
- Keep server subnets separate from user subnets
- Document everything!

---

## /31 and /32 Subnets — Special Cases

**/31 (255.255.255.254):**
- Only 2 addresses, NO network or broadcast (per RFC 3021)
- Used exclusively for point-to-point links
- Saves 2 IPs compared to /30

```
10.0.0.0/31: 10.0.0.0 and 10.0.0.1 (both usable)
10.0.0.2/31: 10.0.0.2 and 10.0.0.3 (both usable)
```

**/32 (255.255.255.255):**
- One single address — used for:
  - Loopback interfaces on routers
  - Host routes in routing tables
  - Identifying a specific host

```
Router(config)# interface Loopback0
Router(config-if)# ip address 1.1.1.1 255.255.255.255
```

---

## Practice Problems

1. VLSM: Given 192.168.10.0/24, create subnets for: Dept A (60 hosts), Dept B (28 hosts), Dept C (12 hosts), 2 WAN links (2 hosts each).

2. Summarize: 172.16.32.0/24, 172.16.33.0/24, 172.16.34.0/24, 172.16.35.0/24

3. What's the summary route for 10.1.0.0/24 through 10.1.7.0/24?

4. Given 192.168.0.0/22, how many /24 subnets can you create? How many /27 subnets?

<details>
<summary>Answers</summary>

**1. VLSM solution:**
```
Dept A:  192.168.10.0/26   (.0–.63)      62 hosts ✅
Dept B:  192.168.10.64/27  (.64–.95)     30 hosts ✅
Dept C:  192.168.10.96/28  (.96–.111)    14 hosts ✅
WAN 1:   192.168.10.112/30 (.112–.115)   2 hosts ✅
WAN 2:   192.168.10.116/30 (.116–.119)   2 hosts ✅
Free:    192.168.10.120–.255
```

**2.** Third octet: 32=00100000, 33=00100001, 34=00100010, 35=00100011 → 6 common bits in third octet → /22
Summary: **172.16.32.0/22**

**3.** Third octet: 0-7 → 00000000 to 00000111 → 5 common bits → /21
Summary: **10.1.0.0/21**

**4.** /22 = 1024 addresses. /24 = 256 addresses → 1024÷256 = **4 /24 subnets**. /27 = 32 addresses → 1024÷32 = **32 /27 subnets**
</details>

---

## Key Takeaways

1. **VLSM = different mask sizes in the same network** — allocate largest first
2. **Summarization = combine small routes into one** — find common bits
3. **/30 for WAN links** (or /31 if supported)
4. **/32 for loopbacks** and host routes
5. **Always leave room for growth** in real designs

---

*← [Day 5 — Subnetting Part 1](Day05-Subnetting-Part1.md) | [Day 7 — Ethernet & Switching](Day07-Ethernet-Switching.md) →*
