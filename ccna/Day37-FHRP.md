# Day 37: First Hop Redundancy Protocols (FHRP)

## 🎯 What You'll Learn
HSRP, VRRP, and GLBP — how they provide gateway redundancy so hosts never lose connectivity when a router fails.

---

## The Problem

Every host has **one default gateway**. If that router dies, every host behind it loses all connectivity — even if a second router exists on the same LAN.

```
         Internet
            │
    ┌───────┴───────┐
    │ R1 (active)   │ R2 (standby)
    │ 192.168.1.1   │ 192.168.1.2
    └───────┬───────┘
            │
         [SWITCH]
          / | \
       [PC1][PC2][PC3]
       GW: 192.168.1.1

If R1 dies → PC1, PC2, PC3 have NO gateway! ❌
They'd need to manually change to 192.168.1.2
```

**FHRPs solve this** by creating a **virtual IP address** shared between routers. Hosts point to the virtual IP as their gateway. If the active router fails, the standby takes over — transparently.

```
    ┌───────────────────────┐
    │ R1 (active)  R2 (standby) │
    │ Real: .2     Real: .3     │
    │    Virtual IP: .1          │  ← Hosts use this as gateway
    └───────────┬───────────────┘
             [SWITCH]
              / | \
           [PC1][PC2][PC3]
           GW: 192.168.1.1 (virtual)
```

---

## HSRP (Hot Standby Router Protocol)

**Cisco proprietary.** The most commonly tested FHRP on the CCNA.

### How It Works
- Two (or more) routers share a **virtual IP** and **virtual MAC**
- One router is **Active**, one is **Standby**, others are in **Listen** state
- Active router responds to ARP requests for the virtual IP with the virtual MAC
- Standby monitors the Active via Hello messages
- If Active fails → Standby becomes Active (failover)

### HSRP Virtual MAC Address
```
0000.0c07.acXX
              ↑
         Group number (hex)

Group 1 → 0000.0c07.ac01
Group 10 → 0000.0c07.ac0a
```

### Configuration
```
! R1 — Make this the Active router
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.2 255.255.255.0
R1(config-if)# standby 1 ip 192.168.1.1
! Group 1, virtual IP .1

R1(config-if)# standby 1 priority 110
! Default priority is 100. Higher = wins election.

R1(config-if)# standby 1 preempt
! If R1 comes back after a failure, it takes over as Active again

! R2 — Standby router
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.1.3 255.255.255.0
R2(config-if)# standby 1 ip 192.168.1.1
R2(config-if)# standby 1 priority 100
! Lower priority = Standby
```

### HSRP States
```
Initial → Listen → Speak → Standby → Active
```

| State | Description |
|-------|-------------|
| Initial | HSRP starting up |
| Listen | Knows virtual IP; neither Active nor Standby |
| Speak | Sending Hello messages; participating in election |
| Standby | Backup; will become Active if Active fails |
| Active | Forwarding traffic for the virtual IP |

### HSRP Timers
- **Hello:** 3 seconds
- **Hold:** 10 seconds (3× Hello — if no Hello for 10s, Active is considered dead)

```
! Customize timers (optional)
R1(config-if)# standby 1 timers 1 3
! Hello every 1 second, Hold after 3 seconds (faster failover)

! Millisecond timers for sub-second failover
R1(config-if)# standby 1 timers msec 200 msec 700
```

### HSRP Versions
| Feature | HSRPv1 | HSRPv2 |
|---------|--------|--------|
| Group range | 0-255 | 0-4095 |
| Multicast | 224.0.0.2 | 224.0.0.102 |
| Virtual MAC | 0000.0c07.acXX | 0000.0c9f.fXXX |
| IPv6 support | No | Yes |

```
R1(config-if)# standby version 2
```

### Interface Tracking
If R1's uplink to the internet fails, R1 should stop being Active (even though its LAN interface is fine):

```
R1(config)# track 1 interface GigabitEthernet0/1 line-protocol
! Track the uplink interface

R1(config-if)# standby 1 track 1 decrement 20
! If Gi0/1 goes down, reduce HSRP priority by 20
! Priority goes from 110 to 90 → R2 (priority 100) takes over
```

---

## VRRP (Virtual Router Redundancy Protocol)

**Open standard (IEEE).** Very similar to HSRP.

| Feature | HSRP | VRRP |
|---------|------|------|
| Standard | Cisco | IEEE (RFC 5798) |
| Terminology | Active/Standby | Master/Backup |
| Default priority | 100 | 100 |
| Preemption | Disabled by default | **Enabled by default** |
| Virtual MAC | 0000.0c07.acXX | 0000.5e00.01XX |
| Multicast | 224.0.0.2 / .102 | 224.0.0.18 |
| IP owner | No concept | If router's real IP = virtual IP, it's always Master |
| Group range | 0-255 (v1) / 0-4095 (v2) | 0-255 |

```
R1(config-if)# vrrp 1 ip 192.168.1.1
R1(config-if)# vrrp 1 priority 110
! Preemption is ON by default — no need to configure it
```

**IP Owner concept:** If you configure the virtual IP to be the same as the router's real interface IP, that router is the "IP owner" and ALWAYS becomes Master (priority 255 automatically).

---

## GLBP (Gateway Load Balancing Protocol)

**Cisco proprietary.** Unlike HSRP/VRRP (active/standby), GLBP provides **load balancing** across multiple routers.

### How It Works
- **AVG (Active Virtual Gateway):** Elected like HSRP Active. Answers ARP requests.
- **AVF (Active Virtual Forwarder):** Up to 4 routers forward traffic. The AVG assigns different virtual MACs to different hosts.

```
Host A asks for gateway MAC → AVG gives MAC-1 (R1 forwards)
Host B asks for gateway MAC → AVG gives MAC-2 (R2 forwards)
Host C asks for gateway MAC → AVG gives MAC-1 (R1 forwards)

Both routers actively forward! Not just failover!
```

**Virtual MAC:** `0007.b400.XXYY` (XX = group, YY = forwarder number)

```
R1(config-if)# glbp 1 ip 192.168.1.1
R1(config-if)# glbp 1 priority 110
R1(config-if)# glbp 1 preempt

R2(config-if)# glbp 1 ip 192.168.1.1
R2(config-if)# glbp 1 priority 100
```

---

## FHRP Comparison Summary

| Feature | HSRP | VRRP | GLBP |
|---------|------|------|------|
| Standard | Cisco | IEEE | Cisco |
| Model | Active/Standby | Master/Backup | AVG + AVFs |
| Load balancing | ❌ (active/standby) | ❌ (master/backup) | ✅ Yes! |
| Preemption default | Off | **On** | Off |
| Virtual MAC | 0000.0c07.acXX | 0000.5e00.01XX | 0007.b400.XXYY |
| IPv6 support | v2 only | Yes | Yes |

> 💡 **CCNA focus:** HSRP is tested most. Know the configuration, states, preemption, and tracking. Know VRRP and GLBP exist and how they differ.

---

## Verification

```
show standby                 ! HSRP — all groups
show standby brief           ! HSRP — compact view
show vrrp                    ! VRRP status
show vrrp brief
show glbp                    ! GLBP status
show glbp brief
```

```
R1# show standby brief
                     P indicates configured to preempt.
Interface   Grp  Pri P State    Active      Standby     Virtual IP
Gi0/0       1    110 P Active   local       192.168.1.3 192.168.1.1
```

---

## Practice Questions

1. What problem do FHRPs solve?
2. What is the default HSRP priority?
3. Is HSRP preemption enabled by default?
4. What virtual MAC does HSRP group 5 use?
5. How does GLBP differ from HSRP?
6. What is the VRRP "IP owner" concept?
7. What does HSRP interface tracking do?

<details>
<summary>Answers</summary>

1. Single point of failure at the default gateway — if the gateway router fails, all hosts lose connectivity
2. 100
3. No — must be explicitly configured with `standby <group> preempt`
4. 0000.0c07.ac05
5. GLBP provides active-active load balancing across multiple routers; HSRP is active/standby (only one forwards)
6. If a router's real interface IP equals the VRRP virtual IP, it's the IP owner and always becomes Master with priority 255
7. If the tracked interface goes down, HSRP priority decreases, allowing the standby router to take over
</details>

---

*← [Day 36 — EIGRP Overview](Day36-EIGRP-Overview.md) | [Day 43-44 — IPv6](Day43-44-IPv6.md) →*
