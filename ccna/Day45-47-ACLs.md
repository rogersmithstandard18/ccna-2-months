# Days 45-47: Access Control Lists (ACLs)

## 🎯 What You'll Learn
Standard and extended ACLs, wildcard masks, placement rules, IPv6 ACLs, and VTY line ACLs. ACLs are heavily tested on the CCNA.

---

## What Are ACLs?

ACLs are **ordered lists of permit/deny rules** applied to router interfaces to filter traffic. Think of them as a firewall checklist.

**Key rules:**
1. **Processed top-down** — first match wins
2. **Implicit deny any** at the end of every ACL (if nothing matches, packet is dropped)
3. Applied **per-interface, per-direction** (inbound or outbound)
4. One ACL per interface, per direction, per protocol

---

## Standard vs Extended ACLs

| Feature | Standard | Extended |
|---------|----------|---------|
| Matches on | **Source IP only** | Source IP, Dest IP, Protocol, Port |
| Number range | 1-99, 1300-1999 | 100-199, 2000-2699 |
| Named | Yes | Yes |
| Placement | **Near destination** | **Near source** |
| Precision | Low (can only filter by source) | High (very granular) |

---

## Wildcard Masks

ACLs use **wildcard masks** (inverse of subnet masks):
- **0** = must match this bit
- **1** = don't care (any value)

```
Subnet mask:  255.255.255.0   = 11111111.11111111.11111111.00000000
Wildcard:     0.0.0.255       = 00000000.00000000.00000000.11111111
```

**Quick conversion:** 255.255.255.255 – subnet mask = wildcard mask

| Subnet Mask | Wildcard | Matches |
|-------------|----------|---------|
| 255.255.255.255 | 0.0.0.0 | Exactly one host |
| 255.255.255.0 | 0.0.0.255 | Entire /24 |
| 255.255.0.0 | 0.0.255.255 | Entire /16 |
| 255.255.255.240 | 0.0.0.15 | /28 subnet |
| 255.255.252.0 | 0.0.3.255 | /22 block |

**Shortcuts:**
- `host 10.0.0.1` = `10.0.0.1 0.0.0.0` (match exactly this IP)
- `any` = `0.0.0.0 255.255.255.255` (match any IP)

---

## Standard ACLs

Match on **source IP only**. Place **near the destination**.

### Numbered Standard ACL
```
R1(config)# access-list 10 permit 192.168.1.0 0.0.0.255
R1(config)# access-list 10 deny 192.168.2.0 0.0.0.255
R1(config)# access-list 10 permit any
! Without "permit any," the implicit deny blocks everything else!

! Apply to interface
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip access-group 10 out
! "out" = filter traffic LEAVING this interface
```

### Named Standard ACL
```
R1(config)# ip access-list standard ALLOW_SALES
R1(config-std-nacl)# permit 192.168.10.0 0.0.0.255
R1(config-std-nacl)# deny any log
! "log" keyword — logs every match to syslog (useful for troubleshooting)

R1(config)# interface Gi0/0
R1(config-if)# ip access-group ALLOW_SALES out
```

### Why Place Standard ACLs Near the Destination?

Because they only match source IP, placing them near the source would block that source from reaching **everything**, not just the intended destination.

```
R1 ──── R2 ──── R3
             ↑
        Place standard ACL here (near destination R3)
        
If placed on R1 (near source), it blocks traffic to R2 AND R3.
If placed on R3 (near destination), it only blocks traffic to R3.
```

---

## Extended ACLs

Match on **source IP, destination IP, protocol, and port**. Place **near the source**.

### Syntax
```
access-list <num> {permit|deny} <protocol> <source> <wildcard> 
    <destination> <wildcard> [operator port] [log]
```

### Examples
```
! Allow HTTP from Sales to web server
R1(config)# access-list 100 permit tcp 192.168.10.0 0.0.0.255 host 10.0.0.5 eq 80

! Allow HTTPS from Sales to anywhere
R1(config)# access-list 100 permit tcp 192.168.10.0 0.0.0.255 any eq 443

! Allow ICMP (ping) from anywhere to anywhere
R1(config)# access-list 100 permit icmp any any

! Allow DNS
R1(config)# access-list 100 permit udp any any eq 53
R1(config)# access-list 100 permit tcp any any eq 53

! Deny everything else (explicit for logging)
R1(config)# access-list 100 deny ip any any log

! Apply INBOUND (near the source)
R1(config)# interface Gi0/0
R1(config-if)# ip access-group 100 in
```

### Named Extended ACL
```
R1(config)# ip access-list extended WEB_ONLY
R1(config-ext-nacl)# 10 permit tcp 192.168.10.0 0.0.0.255 any eq 80
R1(config-ext-nacl)# 20 permit tcp 192.168.10.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# 30 permit icmp any any echo
R1(config-ext-nacl)# 40 permit icmp any any echo-reply
R1(config-ext-nacl)# 50 deny ip any any log
```

### Why Place Extended ACLs Near the Source?

Because they can match on destination, they're precise enough to block only the specific traffic you want — so block it early to save bandwidth.

---

## Port Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `eq` | Equal to | `eq 80` (port 80) |
| `gt` | Greater than | `gt 1023` (ports 1024+) |
| `lt` | Less than | `lt 1024` (ports 0-1023) |
| `neq` | Not equal to | `neq 22` (anything but SSH) |
| `range` | Port range | `range 8080 8090` |

### The `established` Keyword
```
access-list 100 permit tcp any 192.168.1.0 0.0.0.255 established
```
Allows **return TCP traffic** (ACK or RST bit set). Blocks new incoming connections but allows responses to connections initiated from inside.

---

## Editing ACLs

### Adding/Removing Lines
```
R1(config)# ip access-list extended WEB_ONLY
R1(config-ext-nacl)# no 30              ! Delete line 30
R1(config-ext-nacl)# 25 permit tcp any any eq 22    ! Insert at sequence 25
```

### Resequencing
```
R1(config)# ip access-list resequence WEB_ONLY 10 10
! Renumber starting at 10, incrementing by 10
! Result: 10, 20, 30, 40, 50...
```

---

## ACL on VTY Lines

Restrict **who can SSH/Telnet** to the router:
```
R1(config)# access-list 5 permit 192.168.1.0 0.0.0.255
! Only allow the admin subnet

R1(config)# line vty 0 15
R1(config-line)# access-class 5 in
! Note: "access-class" for VTY, not "access-group"
R1(config-line)# transport input ssh
```

---

## IPv6 ACLs

IPv6 ACLs are **always named** (no numbered option).

```
R1(config)# ipv6 access-list BLOCK_TELNET
R1(config-ipv6-acl)# deny tcp any any eq 23
R1(config-ipv6-acl)# permit ipv6 any any

R1(config)# interface Gi0/0
R1(config-if)# ipv6 traffic-filter BLOCK_TELNET in
! Note: "traffic-filter" for IPv6, not "access-group"
```

**IPv6 ACLs have implicit entries** at the end:
```
permit icmp any any nd-na      ! Allow Neighbor Advertisement
permit icmp any any nd-ns      ! Allow Neighbor Solicitation
deny ipv6 any any              ! Deny everything else
```

The ND permits are critical — without them, IPv6 addressing breaks (NDP is required for basic operation).

---

## ACL Processing — The Flow

```
Packet arrives on interface
        │
        ▼
Is there an INBOUND ACL?
├── No → Continue routing
└── Yes → Check ACL rules top-down
         ├── Match PERMIT → Continue routing
         ├── Match DENY → Drop packet
         └── No match → Implicit DENY → Drop packet

After routing decision:
        │
        ▼
Is there an OUTBOUND ACL on exit interface?
├── No → Forward packet
└── Yes → Check ACL rules top-down
         ├── Match PERMIT → Forward packet
         ├── Match DENY → Drop packet
         └── No match → Implicit DENY → Drop packet
```

---

## Verification

```
show access-lists                    ! All ACLs with match counts
show ip access-lists                 ! IPv4 ACLs only
show ip access-lists 100             ! Specific ACL
show ipv6 access-list                ! IPv6 ACLs
show ip interface Gi0/0 | include access   ! Which ACL is applied
show running-config | section access-list  ! ACL config
```

**Match counts** — invaluable for troubleshooting:
```
R1# show access-lists 100
Extended IP access list 100
    10 permit tcp 192.168.10.0 0.0.0.255 any eq www (150 matches)
    20 permit tcp 192.168.10.0 0.0.0.255 any eq 443 (89 matches)
    30 deny ip any any log (12 matches)
```

---

## Common ACL Scenarios

**Scenario 1: Block a specific host from accessing a server**
```
ip access-list extended BLOCK_HOST
 deny ip host 192.168.1.50 host 10.0.0.5
 permit ip any any
```

**Scenario 2: Allow only SSH to the router, deny Telnet**
```
access-list 110 permit tcp any host 192.168.1.1 eq 22
access-list 110 deny tcp any host 192.168.1.1 eq 23
access-list 110 permit ip any any
```

**Scenario 3: Allow pings from inside, block pings from outside**
```
ip access-list extended ALLOW_OUT_PING
 permit icmp 192.168.1.0 0.0.0.255 any echo
 permit icmp any 192.168.1.0 0.0.0.255 echo-reply
 deny icmp any 192.168.1.0 0.0.0.255 echo
 permit ip any any
```

---

## Practice Questions

1. What is the implicit rule at the end of every ACL?
2. Where should you place a standard ACL?
3. What wildcard matches the network 172.16.0.0/16?
4. What's the difference between `access-group` and `access-class`?
5. What does the `established` keyword match?
6. Can you use numbered IPv6 ACLs?
7. What command shows ACL match counts?

<details>
<summary>Answers</summary>

1. `deny any` (implicit deny all)
2. As close to the **destination** as possible
3. 0.0.255.255
4. `access-group` applies ACLs to interfaces; `access-class` applies ACLs to VTY lines
5. TCP packets with the ACK or RST bit set (return traffic from established connections)
6. No — IPv6 ACLs are always named
7. `show access-lists` (shows match count next to each rule)
</details>

---

*← [Days 43-44 — IPv6](Day43-44-IPv6.md) | [Days 48-49 — NAT](Day48-49-NAT.md) →*
