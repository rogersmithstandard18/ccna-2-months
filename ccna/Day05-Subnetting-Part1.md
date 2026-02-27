# Day 5: Subnetting Mastery — Part 1

## 🎯 What You'll Learn
How to subnet a network, calculate network/broadcast/usable range, and the block size method for fast subnetting. This is the #1 most tested skill on the CCNA.

---

## Why Subnet?

Without subnetting, you'd have one massive network:
```
10.0.0.0/8 = 16 million hosts on ONE broadcast domain
- Every broadcast hits every device
- Massive security risk (everyone sees everyone's traffic)
- Impossible to manage
```

**Subnetting** divides one large network into smaller, manageable pieces:
```
10.0.0.0/8 → subnet into /24s:
  10.0.0.0/24   (Sales — 254 hosts)
  10.0.1.0/24   (Engineering — 254 hosts)
  10.0.2.0/24   (Management — 254 hosts)
  ...
```

**Benefits:** Smaller broadcast domains, better security, easier management, efficient IP usage.

---

## The Subnetting Formula

You only need two formulas:

```
Subnets created = 2^s     (s = number of bits borrowed from the host portion)
Usable hosts    = 2^h – 2  (h = remaining host bits)
```

And one concept:

```
Network bits + Subnet bits + Host bits = 32 (always)
```

---

## The Block Size Method (Fastest Way)

This is the shortcut that makes subnetting fast on the exam.

**Block size = 256 – subnet mask value (in the interesting octet)**

The "interesting octet" is the one that's not 255 or 0 in the subnet mask.

### Example 1: /26 (255.255.255.192)

**Interesting octet value:** 192
**Block size:** 256 – 192 = **64**

The subnets start at multiples of 64 in the last octet:
```
Subnet 0: .0   → Network: .0,   Broadcast: .63,   Usable: .1–.62
Subnet 1: .64  → Network: .64,  Broadcast: .127,  Usable: .65–.126
Subnet 2: .128 → Network: .128, Broadcast: .191,  Usable: .129–.190
Subnet 3: .192 → Network: .192, Broadcast: .255,  Usable: .193–.254
```

**Pattern:** Each subnet starts at the previous start + block size. The broadcast is always one less than the next subnet's start.

### Example 2: /28 (255.255.255.240)

**Block size:** 256 – 240 = **16**

Subnets: .0, .16, .32, .48, .64, .80, .96, .112, .128, .144, .160, .176, .192, .208, .224, .240

For subnet starting at .48:
```
Network:   x.x.x.48
Usable:    x.x.x.49 – x.x.x.62
Broadcast: x.x.x.63
Hosts:     14 (2^4 – 2)
```

### Example 3: /27 (255.255.255.224)

**Block size:** 256 – 224 = **32**

Subnets: .0, .32, .64, .96, .128, .160, .192, .224

For subnet starting at .96:
```
Network:   x.x.x.96
Usable:    x.x.x.97 – x.x.x.126
Broadcast: x.x.x.127
Hosts:     30 (2^5 – 2)
```

---

## Quick Reference — Block Sizes

| CIDR | Mask (last octet) | Block Size | Subnets (in /24) | Hosts |
|------|-------------------|-----------|-------------------|-------|
| /25 | 128 | 128 | 2 | 126 |
| /26 | 192 | 64 | 4 | 62 |
| /27 | 224 | 32 | 8 | 30 |
| /28 | 240 | 16 | 16 | 14 |
| /29 | 248 | 8 | 32 | 6 |
| /30 | 252 | 4 | 64 | 2 |

> 💡 **Memorize this table.** It's the single most useful thing for the CCNA exam.

---

## Step-by-Step Subnetting Process

**Given an IP and mask, find the network, broadcast, and usable range:**

### Step 1: Find the block size
256 – (interesting octet mask value)

### Step 2: Find which subnet the IP falls into
Divide the host octet value by the block size. The subnet start = (quotient × block size).

### Step 3: Calculate
- Network address = subnet start
- Broadcast = next subnet start – 1
- Usable range = network + 1 to broadcast – 1

---

## Worked Examples

### Problem 1: What subnet does 192.168.10.130/26 belong to?

```
Step 1: Mask = /26 = 255.255.255.192
        Block size = 256 – 192 = 64

Step 2: Interesting octet = 130
        130 ÷ 64 = 2.03 → floor = 2
        Subnet start = 2 × 64 = 128

Step 3: 
  Network:   192.168.10.128
  Broadcast: 192.168.10.191  (128 + 64 – 1 = 191)
  Usable:    192.168.10.129 – 192.168.10.190
  Hosts:     62
```

### Problem 2: What subnet does 10.5.100.200/28 belong to?

```
Step 1: Mask = /28 = 255.255.255.240
        Block size = 256 – 240 = 16

Step 2: Interesting octet = 200
        200 ÷ 16 = 12.5 → floor = 12
        Subnet start = 12 × 16 = 192

Step 3:
  Network:   10.5.100.192
  Broadcast: 10.5.100.207  (192 + 16 – 1 = 207)
  Usable:    10.5.100.193 – 10.5.100.206
  Hosts:     14
```

### Problem 3: What subnet does 172.16.55.70/27 belong to?

```
Step 1: Mask = /27 = 255.255.255.224
        Block size = 256 – 224 = 32

Step 2: Interesting octet = 70
        70 ÷ 32 = 2.1875 → floor = 2
        Subnet start = 2 × 32 = 64

Step 3:
  Network:   172.16.55.64
  Broadcast: 172.16.55.95  (64 + 32 – 1 = 95)
  Usable:    172.16.55.65 – 172.16.55.94
  Hosts:     30
```

---

## Subnetting in the Third Octet

When the mask is /17 through /24, the interesting octet is the **third** octet.

### Problem: Subnet 172.16.0.0/20

```
Mask: 255.255.240.0
Interesting octet: Third (240)
Block size: 256 – 240 = 16 (in the THIRD octet)

Subnets:
  172.16.0.0/20   (172.16.0.0 – 172.16.15.255)     → 4094 hosts
  172.16.16.0/20  (172.16.16.0 – 172.16.31.255)     → 4094 hosts
  172.16.32.0/20  (172.16.32.0 – 172.16.47.255)     → 4094 hosts
  172.16.48.0/20  (172.16.48.0 – 172.16.63.255)     → 4094 hosts
  ... and so on up to 172.16.240.0/20
```

**Finding the broadcast when the interesting octet isn't the last:**
```
172.16.32.0/20:
  Block = 16 in third octet
  Next subnet = 172.16.48.0
  Broadcast = 172.16.47.255 (next subnet – 1, and last octet = 255)
  Usable: 172.16.32.1 – 172.16.47.254
```

---

## "How Many Hosts Do I Need?" — Choosing the Right Mask

**Common exam question:** "You need 50 hosts. What's the smallest subnet?"

```
2^h – 2 ≥ 50
2^6 – 2 = 62 ✅ (h = 6, meaning /26)
2^5 – 2 = 30 ❌ (not enough)

Answer: /26 (255.255.255.192) — supports 62 hosts
```

**Quick reference:**

| Need up to... | Host bits needed | CIDR | Mask |
|--------------|-----------------|------|------|
| 2 hosts | 2 | /30 | 255.255.255.252 |
| 6 hosts | 3 | /29 | 255.255.255.248 |
| 14 hosts | 4 | /28 | 255.255.255.240 |
| 30 hosts | 5 | /27 | 255.255.255.224 |
| 62 hosts | 6 | /26 | 255.255.255.192 |
| 126 hosts | 7 | /25 | 255.255.255.128 |
| 254 hosts | 8 | /24 | 255.255.255.0 |
| 510 hosts | 9 | /23 | 255.255.254.0 |
| 1022 hosts | 10 | /22 | 255.255.252.0 |

---

## Practice Problems

Solve these using the block size method. Find the network address, broadcast address, usable range, and number of usable hosts.

1. 192.168.1.57/29
2. 10.10.10.100/25
3. 172.16.200.200/28
4. 192.168.50.33/27
5. 10.0.0.1/30
6. You need subnets with at least 100 hosts each from 192.168.0.0. What mask do you use?
7. How many /28 subnets can you create from a /24?
8. What's the first usable IP in the subnet 172.16.128.0/25?

<details>
<summary>Answers</summary>

**1. 192.168.1.57/29**
Block = 256–248 = 8. 57÷8 = 7.x → subnet start = 56
- Network: 192.168.1.56
- Broadcast: 192.168.1.63
- Usable: 192.168.1.57–192.168.1.62
- Hosts: 6

**2. 10.10.10.100/25**
Block = 256–128 = 128. 100÷128 = 0.x → subnet start = 0
- Network: 10.10.10.0
- Broadcast: 10.10.10.127
- Usable: 10.10.10.1–10.10.10.126
- Hosts: 126

**3. 172.16.200.200/28**
Block = 256–240 = 16. 200÷16 = 12.5 → subnet start = 192
- Network: 172.16.200.192
- Broadcast: 172.16.200.207
- Usable: 172.16.200.193–172.16.200.206
- Hosts: 14

**4. 192.168.50.33/27**
Block = 256–224 = 32. 33÷32 = 1.x → subnet start = 32
- Network: 192.168.50.32
- Broadcast: 192.168.50.63
- Usable: 192.168.50.33–192.168.50.62
- Hosts: 30

**5. 10.0.0.1/30**
Block = 256–252 = 4. 1÷4 = 0.x → subnet start = 0
- Network: 10.0.0.0
- Broadcast: 10.0.0.3
- Usable: 10.0.0.1–10.0.0.2
- Hosts: 2

**6.** 100 hosts → 2^h–2 ≥ 100 → h=7 → /25 (255.255.255.128) → 126 hosts ✅

**7.** A /24 has 256 addresses. A /28 has 16 addresses. 256÷16 = **16 subnets**

**8.** 172.16.128.1 (network address + 1)
</details>

---

## Key Takeaways

1. **Block size = 256 – mask value** — this is your fastest tool
2. **Subnets start at multiples of the block size**
3. **Broadcast = next subnet – 1**
4. **Usable hosts = 2^h – 2** — always subtract 2
5. **Practice makes perfect** — do 10-20 problems a day until it's instant

---

*← [Day 4 — IPv4 Addressing](Day04-IPv4-Addressing.md) | [Day 6 — Subnetting Part 2 & VLSM](Day06-Subnetting-Part2-VLSM.md) →*
