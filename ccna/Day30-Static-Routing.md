# Day 30: Static Routing

## 🎯 What You'll Learn
How to configure static routes, default routes, floating static routes, and when to use each. Static routing is simple, predictable, and foundational.

---

## When to Use Static Routes

| Scenario | Use Static? |
|----------|------------|
| Small network (2-3 routers) | ✅ Perfect |
| Stub network (one way in/out) | ✅ Ideal |
| Default route to ISP | ✅ Always |
| Backup path (floating static) | ✅ Great use case |
| Large enterprise (50+ routers) | ❌ Use dynamic routing |
| Frequently changing topology | ❌ Too much manual work |

**Pros:** No CPU overhead, no bandwidth used, full admin control, predictable
**Cons:** Doesn't scale, no automatic failover, manual updates required

---

## Static Route Syntax

```
ip route <destination_network> <subnet_mask> <next-hop_IP | exit-interface> [AD]
```

### Type 1: Next-Hop IP (Most Common)
```
R1(config)# ip route 192.168.2.0 255.255.255.0 10.0.0.2
```
"To reach 192.168.2.0/24, send packets to 10.0.0.2"

The router must then do a **recursive lookup** to find which interface reaches 10.0.0.2.

### Type 2: Exit Interface
```
R1(config)# ip route 192.168.2.0 255.255.255.0 GigabitEthernet0/1
```
"To reach 192.168.2.0/24, send packets out Gi0/1"

⚠️ **Only use on point-to-point links** (serial, tunnel). On multi-access networks (Ethernet), the router doesn't know the next-hop MAC and proxy-ARPs for everything — very inefficient.

### Type 3: Fully Specified (Both)
```
R1(config)# ip route 192.168.2.0 255.255.255.0 GigabitEthernet0/1 10.0.0.2
```
"To reach 192.168.2.0/24, send out Gi0/1 to 10.0.0.2"

Best of both worlds — no recursive lookup AND knows the next-hop.

> 💡 **Best practice:** Use next-hop IP (Type 1) for Ethernet links. Use fully specified (Type 3) when you want to avoid recursive lookups.

---

## Default Route (Gateway of Last Resort)

A default route matches **any destination** that doesn't have a more specific route. It's your "if I don't know where to send it, send it here" rule.

```
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2
```

```
R1# show ip route
...
S*   0.0.0.0/0 [1/0] via 10.0.0.2
Gateway of last resort is 10.0.0.2 to network 0.0.0.0
```

**Use case:** Edge routers pointing to the ISP. You don't need routes for the entire internet — just send everything unknown to the ISP.

```
              Internet
                │
            [ISP Router]
                │ 10.0.0.2
                │
            [R1: 10.0.0.1]  ← Default route: 0.0.0.0/0 via 10.0.0.2
               / \
        LAN A     LAN B
```

---

## Floating Static Route (Backup Route)

A floating static route has a **higher AD** than the primary route, so it only activates when the primary route disappears.

```
Scenario:
  R1 has an OSPF route to 192.168.5.0/24 via primary link (AD 110)
  R1 also has a backup ISP link

! Primary route (learned via OSPF automatically — AD 110)
! Backup route (static with AD 200 — higher than OSPF's 110)
R1(config)# ip route 192.168.5.0 255.255.255.0 10.0.1.2 200
                                                          ↑
                                                   AD = 200 (backup only)
```

**How it works:**
```
Normal operation:
  OSPF route: 192.168.5.0/24 [110/20] via 10.0.0.2  ← ACTIVE (AD 110 < 200)
  Static route: 192.168.5.0/24 [200/0] via 10.0.1.2  ← HIDDEN (AD 200 > 110)

Primary link fails:
  OSPF route disappears
  Static route: 192.168.5.0/24 [200/0] via 10.0.1.2  ← NOW ACTIVE!

Primary link recovers:
  OSPF route reappears with AD 110
  Static route goes back to HIDDEN
```

> 💡 **Choose the AD carefully:** It must be higher than the primary protocol's AD. Common choices: 200 or 210 for backup static routes.

---

## Summary Routes (Static Supernet)

Instead of creating multiple static routes, summarize them:

```
! Instead of:
ip route 192.168.0.0 255.255.255.0 10.0.0.2
ip route 192.168.1.0 255.255.255.0 10.0.0.2
ip route 192.168.2.0 255.255.255.0 10.0.0.2
ip route 192.168.3.0 255.255.255.0 10.0.0.2

! Use one summary:
ip route 192.168.0.0 255.255.252.0 10.0.0.2
! /22 covers .0.0 through .3.255
```

---

## IPv6 Static Routes

Same concept, IPv6 syntax:

```
! Standard static route
R1(config)# ipv6 route 2001:DB8:ACAD:2::/64 2001:DB8:ACAD:3::2

! Default route
R1(config)# ipv6 route ::/0 2001:DB8:ACAD:3::2

! Fully specified (link-local next-hop requires exit interface)
R1(config)# ipv6 route 2001:DB8:ACAD:2::/64 GigabitEthernet0/1 FE80::2
```

> 💡 **IPv6 link-local next-hops:** Since link-local addresses are the same on every link, you MUST specify the exit interface when using a link-local next-hop.

---

## Verification

```
show ip route                    ! Full routing table
show ip route static             ! Only static routes
show ip route 192.168.2.0        ! Specific destination lookup
show ipv6 route                  ! IPv6 routing table
show ipv6 route static           ! IPv6 static routes
```

---

## Complete Lab Scenario

```
  [LAN A]          [LAN B]          [LAN C]
192.168.1.0/24  192.168.2.0/24  192.168.3.0/24
     |               |               |
   R1 ─── 10.0.0.0/30 ─── R2 ─── 10.0.0.4/30 ─── R3
  .1  .1              .1 .2  .1               .1 .2

R1 Configuration:
  ip route 192.168.2.0 255.255.255.0 10.0.0.2
  ip route 192.168.3.0 255.255.255.0 10.0.0.2
  ! Or summarize: ip route 192.168.2.0 255.255.254.0 10.0.0.2

R2 Configuration:
  ip route 192.168.1.0 255.255.255.0 10.0.0.1
  ip route 192.168.3.0 255.255.255.0 10.0.0.6

R3 Configuration:
  ip route 192.168.1.0 255.255.255.0 10.0.0.5
  ip route 192.168.2.0 255.255.255.0 10.0.0.5
  ! Or summarize: ip route 192.168.0.0 255.255.252.0 10.0.0.5
```

> 💡 **Key insight:** Static routing requires you to configure routes on EVERY router. Each router only knows about directly connected networks — you must tell it about everything else.

---

## Practice Questions

1. What is the syntax for a next-hop static route?
2. What is the default route destination/mask?
3. What is a floating static route?
4. Why shouldn't you use exit-interface-only routes on Ethernet?
5. What AD is assigned to static routes by default?
6. How do you create an IPv6 default route?

<details>
<summary>Answers</summary>

1. `ip route <destination> <mask> <next-hop-IP>`
2. 0.0.0.0 0.0.0.0
3. A static route with a higher-than-default AD, used as a backup when the primary dynamic route is unavailable
4. The router proxy-ARPs for every destination, sending ARP requests for every unknown host — very inefficient on multi-access networks
5. 1
6. `ipv6 route ::/0 <next-hop>`
</details>

---

*← [Day 29 — Routing Concepts](Day29-Routing-Concepts.md) | [Days 31-32 — OSPF Concepts](Day31-32-OSPF-Concepts.md) →*
