# Days 43-44: IPv6 Addressing

## 🎯 What You'll Learn
IPv6 address format, types, SLAAC, DHCPv6, NDP, and configuration. IPv6 is guaranteed on the CCNA exam.

---

## Why IPv6?

IPv4 has ~4.3 billion addresses. We've run out. NAT has been a band-aid, but it breaks end-to-end connectivity and adds complexity.

**IPv6 = 128-bit addresses = 3.4 × 10^38 addresses.** That's 340 undecillion — enough for every grain of sand on Earth to have billions of addresses.

**Other benefits:**
- No NAT needed (every device gets a globally unique address)
- Simplified header (faster processing)
- No broadcast (multicast and anycast replace it)
- Built-in IPsec support
- Autoconfiguration (SLAAC)

---

## IPv6 Address Format

**128 bits, written as 8 groups of 4 hex digits:**
```
Full:      2001:0DB8:0000:0000:0000:0000:0000:0001
Shortened: 2001:DB8::1
```

**Shortening rules:**
1. **Remove leading zeros** in each group: `0DB8` → `DB8`, `0000` → `0`
2. **Replace consecutive all-zero groups with `::`** (only once!)

```
Full:      2001:0DB8:0000:0000:0000:0000:0000:0001
Step 1:    2001:DB8:0:0:0:0:0:1
Step 2:    2001:DB8::1

Another example:
FE80:0000:0000:0000:0210:A4FF:FE01:3456
FE80:0:0:0:210:A4FF:FE01:3456
FE80::210:A4FF:FE01:3456
```

**⚠️ You can only use `::` ONCE per address.** Otherwise it's ambiguous (which group of zeros is longer?).

---

## IPv6 Address Types

| Type | Prefix | Scope | Description |
|------|--------|-------|-------------|
| **Global Unicast (GUA)** | 2000::/3 | Internet | Like a public IPv4 address |
| **Link-Local** | FE80::/10 | Single link | Auto-generated, required on every interface, not routable |
| **Unique Local** | FC00::/7 (FD00::/8) | Organization | Like private IPv4 (RFC 1918) |
| **Multicast** | FF00::/8 | Varies | One-to-many |
| **Loopback** | ::1/128 | Device | Like 127.0.0.1 |
| **Unspecified** | ::/128 | — | Like 0.0.0.0 |

### Global Unicast Address (GUA) — "Public" Address
```
2001:0DB8:ACAD:0001:0000:0000:0000:0001/64
├── Global Routing Prefix ──┤├ Subnet ┤├── Interface ID ──────┤
     (assigned by ISP/RIR)    (your    (device identifier)
        48 bits               subnets)    64 bits
                              16 bits
```

### Link-Local Address — Automatically Created
```
FE80::1
```
- Generated **automatically** on every IPv6-enabled interface
- Used for: neighbor discovery, routing protocol communication, next-hop addresses
- **Never routed** beyond the local link
- Routers use link-local addresses as next-hops (not global addresses)

### Multicast Addresses You Need to Know
| Address | Who Listens |
|---------|-------------|
| FF02::1 | All nodes on the link |
| FF02::2 | All routers on the link |
| FF02::5 | All OSPF routers |
| FF02::6 | All OSPF DR/BDR |
| FF02::9 | All RIPng routers |
| FF02::A | All EIGRP routers |
| FF02::1:FF00:0/104 | Solicited-node multicast (for NDP) |

---

## Interface ID Generation — EUI-64

SLAAC can auto-generate the 64-bit Interface ID from the 48-bit MAC address:

```
MAC: AA:BB:CC:DD:EE:FF

Step 1: Split in half:        AA:BB:CC | DD:EE:FF
Step 2: Insert FF:FE:         AA:BB:CC:FF:FE:DD:EE:FF
Step 3: Flip the 7th bit (U/L bit) of the first byte:
        AA = 10101010 → flip bit 7 → 10101000 = A8

Result: A8BB:CCFF:FEDD:EEFF (Interface ID)

Full address: 2001:DB8:ACAD:1:A8BB:CCFF:FEDD:EEFF/64
```

**Privacy concern:** EUI-64 embeds your MAC address (identifies your hardware). Modern OSes use **randomized interface IDs** instead (privacy extensions, RFC 4941).

---

## NDP — Neighbor Discovery Protocol

**NDP replaces ARP** in IPv6. It uses ICMPv6 messages:

| Message | Type | Purpose | IPv4 Equivalent |
|---------|------|---------|----------------|
| **Router Solicitation (RS)** | 133 | "Any routers out there?" | — |
| **Router Advertisement (RA)** | 134 | "Here I am! Here's the prefix and flags" | — |
| **Neighbor Solicitation (NS)** | 135 | "What's the MAC for this IP?" | ARP Request |
| **Neighbor Advertisement (NA)** | 136 | "Here's my MAC" | ARP Reply |
| **Redirect** | 137 | "Use this better next-hop" | ICMP Redirect |

### Address Resolution (replaces ARP)
```
Host A wants to reach 2001:DB8::2:

1. Host A sends NS to the solicited-node multicast:
   FF02::1:FF00:0002 → "Who has 2001:DB8::2?"

2. Host B (owner of that IP) replies with NA (unicast):
   "That's me! My MAC is AA:BB:CC:DD:EE:FF"
```

**Solicited-node multicast:** Formed from the last 24 bits of the target IP:
```
Target: 2001:DB8:ACAD:1::1234:ABCD
Solicited-node: FF02::1:FF34:ABCD
```

### DAD (Duplicate Address Detection)
Before using any IPv6 address, a device checks if it's already in use:
1. Send NS for its OWN address
2. If someone replies with NA → duplicate! Address not used.
3. If no reply → safe to use ✅

---

## Address Assignment Methods

### SLAAC (Stateless Address Autoconfiguration)
The host configures itself — no DHCP server needed:
1. Host sends RS (Router Solicitation) to FF02::2
2. Router responds with RA containing: prefix, prefix length, default gateway
3. Host combines the prefix with its Interface ID (EUI-64 or random)
4. Host runs DAD
5. Host has a working IPv6 address!

### Stateless DHCPv6
SLAAC for the address, DHCPv6 for additional info (DNS, domain name):
```
Router(config-if)# ipv6 nd other-config-flag
! Sets the O flag in RA — tells hosts to query DHCPv6 for DNS etc.
```

### Stateful DHCPv6
DHCPv6 server assigns everything (address, DNS, domain):
```
Router(config-if)# ipv6 nd managed-config-flag
! Sets the M flag in RA — tells hosts to use DHCPv6 for address assignment
```

| Method | Address From | DNS From | Server Needed? |
|--------|-------------|----------|---------------|
| SLAAC | RA prefix + self-generated | Not provided | No |
| Stateless DHCPv6 | RA prefix + self-generated | DHCPv6 | Yes (DNS only) |
| Stateful DHCPv6 | DHCPv6 server | DHCPv6 | Yes (full) |

---

## IPv6 Configuration

```
! Enable IPv6 routing globally
R1(config)# ipv6 unicast-routing

! Configure a GUA
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ipv6 address 2001:DB8:ACAD:1::1/64
R1(config-if)# no shutdown

! Configure a link-local address manually
R1(config-if)# ipv6 address FE80::1 link-local

! Enable IPv6 without assigning an address (link-local only)
R1(config-if)# ipv6 enable
```

### Verification
```
show ipv6 interface brief         ! Quick view — addresses per interface
show ipv6 interface Gi0/0         ! Detailed (multicast groups, ND, DAD)
show ipv6 route                   ! IPv6 routing table
show ipv6 neighbors               ! NDP cache (like "show arp")
```

---

## Dual-Stack, Tunneling, Translation

**Dual-Stack:** Run IPv4 and IPv6 simultaneously. Most common transition method. Every interface has both an IPv4 and IPv6 address.

**Tunneling:** Encapsulate IPv6 packets inside IPv4 packets to cross IPv4-only networks (6to4, ISATAP, GRE).

**NAT64:** Translates between IPv6 and IPv4 — allows IPv6-only hosts to reach IPv4-only servers.

---

## Practice Questions

1. How many bits in an IPv6 address?
2. Shorten: 2001:0DB8:0000:0000:0000:00AB:0000:1234
3. What is the link-local prefix?
4. What replaces ARP in IPv6?
5. What is SLAAC?
6. What command enables IPv6 routing on a Cisco router?
7. What does DAD do?
8. What multicast address reaches all routers on the link?

<details>
<summary>Answers</summary>

1. 128 bits
2. 2001:DB8::AB:0:1234
3. FE80::/10
4. NDP (Neighbor Discovery Protocol) — specifically NS/NA messages
5. Stateless Address Autoconfiguration — hosts automatically configure their own IPv6 address using the prefix from router advertisements
6. `ipv6 unicast-routing`
7. Duplicate Address Detection — checks that an address isn't already in use before assigning it
8. FF02::2
</details>

---

*← [Day 37 — FHRP](Day37-FHRP.md) | [Days 45-47 — ACLs](Day45-47-ACLs.md) →*
