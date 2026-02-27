# Day 4: IPv4 Addressing Fundamentals

## 🎯 What You'll Learn
Binary-decimal conversion, IPv4 address structure, address classes, private vs public addresses, and special addresses. This is the foundation for subnetting (Days 5-6).

---

## Binary — The Language of Networking

Every IP address is a 32-bit binary number displayed as four decimal octets.

**The Binary Powers Chart (memorize this):**

| Bit Position | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|-------------|---|---|---|---|---|---|---|---|
| **Value** | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

Each octet = 8 bits = values **0 to 255** (because 128+64+32+16+8+4+2+1 = 255)

### Converting Decimal to Binary

**Method: Subtract from left to right**

Convert **192** to binary:
```
192 ≥ 128? YES → 1, remainder = 64
 64 ≥  64? YES → 1, remainder = 0
  0 ≥  32? NO  → 0
  0 ≥  16? NO  → 0
  0 ≥   8? NO  → 0
  0 ≥   4? NO  → 0
  0 ≥   2? NO  → 0
  0 ≥   1? NO  → 0

Result: 11000000
```

Convert **168** to binary:
```
168 ≥ 128? YES → 1, remainder = 40
 40 ≥  64? NO  → 0
 40 ≥  32? YES → 1, remainder = 8
  8 ≥  16? NO  → 0
  8 ≥   8? YES → 1, remainder = 0
  0 ≥   4? NO  → 0
  0 ≥   2? NO  → 0
  0 ≥   1? NO  → 0

Result: 10101000
```

**Full conversion of 192.168.1.1:**
```
192 = 11000000
168 = 10101000
  1 = 00000001
  1 = 00000001

Binary: 11000000.10101000.00000001.00000001
```

### Converting Binary to Decimal

Just add up the bit positions that have a 1:

```
10110100:
128 + 0 + 32 + 16 + 0 + 4 + 0 + 0 = 180
```

### Practice Conversions

Try these yourself before looking at the answers:
1. 10 → binary
2. 255 → binary
3. 172 → binary
4. 11001100 → decimal
5. 10000001 → decimal

<details>
<summary>Answers</summary>

1. 10 = 00001010
2. 255 = 11111111
3. 172 = 10101100
4. 11001100 = 128+64+8+4 = 204
5. 10000001 = 128+1 = 129
</details>

> 💡 **Exam tip:** You WILL need to do binary conversions on the exam. Practice until it's fast — don't rely on a calculator. Aim for under 15 seconds per conversion.

---

## IPv4 Address Structure

An IPv4 address has two parts:
```
   192.168.10.50  /  255.255.255.0
   └─ Network ─┘    └── Host ──┘

Network portion: Identifies WHICH network (like a street name)
Host portion:    Identifies WHICH device on that network (like a house number)
```

**The subnet mask determines where the split is:**
```
IP:   192.168.10.50     = 11000000.10101000.00001010.00110010
Mask: 255.255.255.0     = 11111111.11111111.11111111.00000000
                           ├── Network (1s) ──────┤├─ Host ─┤
```

**CIDR Notation:** Instead of writing 255.255.255.0, we write **/24** (because there are 24 network bits).

**Common CIDR to Subnet Mask conversions:**
| CIDR | Subnet Mask | Network Bits | Host Bits |
|------|------------|-------------|-----------|
| /8 | 255.0.0.0 | 8 | 24 |
| /16 | 255.255.0.0 | 16 | 16 |
| /24 | 255.255.255.0 | 24 | 8 |
| /25 | 255.255.255.128 | 25 | 7 |
| /26 | 255.255.255.192 | 26 | 6 |
| /27 | 255.255.255.224 | 27 | 5 |
| /28 | 255.255.255.240 | 28 | 4 |
| /29 | 255.255.255.248 | 29 | 3 |
| /30 | 255.255.255.252 | 30 | 2 |
| /32 | 255.255.255.255 | 32 | 0 |

---

## IPv4 Address Classes

Originally, IP addresses were divided into classes. This system is mostly obsolete (we use CIDR now), but **the CCNA exam still tests it.**

| Class | First Octet Range | First Bits | Default Mask | Purpose |
|-------|------------------|-----------|-------------|---------|
| A | 1–126 | 0xxxxxxx | /8 (255.0.0.0) | Very large networks |
| B | 128–191 | 10xxxxxx | /16 (255.255.0.0) | Medium networks |
| C | 192–223 | 110xxxxx | /24 (255.255.255.0) | Small networks |
| D | 224–239 | 1110xxxx | N/A | Multicast |
| E | 240–255 | 1111xxxx | N/A | Experimental/reserved |

**How to remember:**
- Class A starts with **0** in binary → first octet 1-126
- Class B starts with **10** → first octet 128-191
- Class C starts with **110** → first octet 192-223
- 127 is missing because it's reserved for **loopback**

**Number of networks and hosts per class:**
| Class | Network Bits | Host Bits | Networks | Hosts per Network |
|-------|-------------|-----------|----------|-------------------|
| A | 8 (first fixed) | 24 | 126 | 16,777,214 |
| B | 16 | 16 | 16,384 | 65,534 |
| C | 24 | 8 | 2,097,152 | 254 |

**Host formula:** Usable hosts = 2^(host bits) − 2
- Subtract 2 because: first address = **network address**, last address = **broadcast address**

**Quick identification:**
```
10.5.5.5      → Class A (first octet 1-126)
172.16.0.1    → Class B (first octet 128-191)
192.168.1.1   → Class C (first octet 192-223)
224.0.0.5     → Class D (multicast)
```

---

## Private vs Public Addresses

**Private addresses** (RFC 1918) are free to use inside your network but **cannot be routed on the internet.** You need NAT to translate them to a public address.

| Class | Private Range | CIDR | Addresses |
|-------|-------------|------|-----------|
| A | 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | 16,777,216 |
| B | 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | 1,048,576 |
| C | 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | 65,536 |

> 💡 **Exam tip:** If you see 10.x.x.x, 172.16-31.x.x, or 192.168.x.x → it's **private**. Everything else (in Class A-C range) is **public**.

**Common trick question:** "Is 172.32.0.1 a private address?"
**Answer:** NO! Private Class B only goes up to 172.**31**.255.255. 172.32.0.1 is public.

---

## Special IPv4 Addresses

| Address/Range | Purpose | Details |
|--------------|---------|---------|
| 0.0.0.0 | "This network" / default route | Used by DHCP client before getting an address; also used as default route (0.0.0.0/0) |
| 127.0.0.0/8 | Loopback | 127.0.0.1 is most common; used to test local TCP/IP stack; never leaves the device |
| 169.254.0.0/16 | APIPA (Link-Local) | Auto-assigned when DHCP fails; only works on the local network segment |
| 255.255.255.255 | Limited broadcast | Broadcast to all devices on the local network; routers don't forward it |
| x.x.x.0 | Network address | First address in a subnet; identifies the network itself; not assignable to a host |
| x.x.x.255 | Broadcast address | Last address in a /24 subnet; sent to all hosts in that subnet |

**APIPA in practice:**
```
You plug in a PC. It tries DHCP. No DHCP server responds.
PC assigns itself: 169.254.x.x/16 (random within that range)
PC can talk to other 169.254.x.x devices on the same segment.
PC CANNOT reach the internet or other networks.

When you see 169.254.x.x → DHCP is broken.
```

---

## Network Address, Broadcast Address, Usable Range

For any given IP + mask, you need to identify three things:

**Example: 192.168.1.100/24**
```
Network address:    192.168.1.0     (all host bits = 0)
Broadcast address:  192.168.1.255   (all host bits = 1)
Usable range:       192.168.1.1 – 192.168.1.254
Usable hosts:       254 (2^8 – 2)
```

**Example: 10.0.0.50/8**
```
Network address:    10.0.0.0
Broadcast address:  10.255.255.255
Usable range:       10.0.0.1 – 10.255.255.254
Usable hosts:       16,777,214 (2^24 – 2)
```

**Example: 172.16.5.100/16**
```
Network address:    172.16.0.0
Broadcast address:  172.16.255.255
Usable range:       172.16.0.1 – 172.16.255.254
Usable hosts:       65,534 (2^16 – 2)
```

---

## Determining if Two Hosts are on the Same Network

**Rule:** AND the IP address with the subnet mask. If the result is the same for both hosts, they're on the same network.

**Example:** Are 192.168.1.10/24 and 192.168.1.200/24 on the same network?
```
192.168.1.10  AND 255.255.255.0 = 192.168.1.0
192.168.1.200 AND 255.255.255.0 = 192.168.1.0

Same result → SAME network ✅ (they can communicate directly)
```

**Example:** Are 192.168.1.10/24 and 192.168.2.10/24 on the same network?
```
192.168.1.10  AND 255.255.255.0 = 192.168.1.0
192.168.2.10  AND 255.255.255.0 = 192.168.2.0

Different result → DIFFERENT networks ❌ (need a router to communicate)
```

---

## Practice Questions

1. Convert 10.0.50.1 to binary.
2. What class is 191.200.5.5?
3. Is 172.20.0.1 private or public?
4. Is 172.33.0.1 private or public?
5. What's the broadcast address for 192.168.5.0/24?
6. How many usable hosts in a /24 network?
7. A PC has the IP 169.254.10.5. What's likely wrong?
8. What is the network address for 10.50.100.200/8?
9. Convert 11010011 to decimal.
10. Are 192.168.1.50/24 and 192.168.1.200/24 on the same network?

<details>
<summary>Answers</summary>

1. 00001010.00000000.00110010.00000001
2. Class B (128-191)
3. Private (172.16.0.0 – 172.31.255.255)
4. Public (172.33 is outside the private range)
5. 192.168.5.255
6. 254 (2^8 – 2)
7. DHCP is not working; the PC self-assigned an APIPA address
8. 10.0.0.0
9. 128+64+16+2+1 = 211
10. Yes — both AND with 255.255.255.0 = 192.168.1.0
</details>

---

## Key Takeaways

1. **Binary conversion is a must-have skill** — practice daily until it's automatic
2. **First octet determines the class**: A=1-126, B=128-191, C=192-223
3. **Private ranges**: 10/8, 172.16/12, 192.168/16 — anything else is public
4. **169.254.x.x = DHCP failure** — instant diagnostic
5. **Network address = all host bits 0; Broadcast = all host bits 1**
6. **Usable hosts = 2^h – 2** (subtract network and broadcast)

---

*← [Day 3 — Topologies & Media](Day03-Topologies-Media.md) | [Day 5 — Subnetting Part 1](Day05-Subnetting-Part1.md) →*
