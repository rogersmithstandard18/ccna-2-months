# Day 19: Inter-VLAN Routing

## 🎯 What You'll Learn
How to enable communication between VLANs using Router-on-a-Stick (ROAS), Layer 3 switches with SVIs, and routed ports.

---

## The Problem

VLANs are separate broadcast domains. Hosts in VLAN 10 **cannot** talk to hosts in VLAN 20 — even on the same switch. You need a Layer 3 device to route between them.

```
  VLAN 10: 192.168.10.0/24          VLAN 20: 192.168.20.0/24
  [PC-A: .10]    [PC-B: .20]        [PC-C: .10]    [PC-D: .20]
       │              │                   │              │
     ┌─┴──────────────┴───────────────────┴──────────────┴─┐
     │                    SWITCH (Layer 2)                   │
     └──────────────────────────────────────────────────────┘
     
  PC-A can reach PC-B ✅ (same VLAN)
  PC-A CANNOT reach PC-C ❌ (different VLAN — needs routing)
```

---

## Method 1: Router-on-a-Stick (ROAS)

**Concept:** A single router interface connects to the switch via a trunk link. The router creates **sub-interfaces** — one per VLAN — each acting as that VLAN's default gateway.

```
                    ┌────────┐
                    │ ROUTER │
                    │  Gi0/0 │
                    └───┬────┘
                        │ TRUNK (carries all VLANs)
                    ┌───┴────────────────────────┐
                    │          SWITCH              │
                    │ Fa0/1(V10) Fa0/2(V20) Gi0/1 │
                    └──┬──────────┬───────────────┘
                     [PC-A]     [PC-B]
                    VLAN 10    VLAN 20
```

### Configuration

**Switch side — set up the trunk:**
```
SW1(config)# interface Gi0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# switchport trunk allowed vlan 10,20,99
```

**Router side — create sub-interfaces:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown
! The physical interface must be up; no IP address on it directly

R1(config)# interface GigabitEthernet0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0
! This sub-interface handles VLAN 10 traffic
! 192.168.10.1 is the default gateway for VLAN 10 hosts

R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0
! This sub-interface handles VLAN 20 traffic

R1(config)# interface GigabitEthernet0/0.99
R1(config-subif)# encapsulation dot1Q 99 native
! The "native" keyword means this sub-interface handles untagged traffic
R1(config-subif)# ip address 192.168.99.1 255.255.255.0
```

**Host configuration:**
- PC-A (VLAN 10): IP 192.168.10.10, Gateway **192.168.10.1**
- PC-B (VLAN 20): IP 192.168.20.10, Gateway **192.168.20.1**

**How traffic flows:**
```
PC-A (VLAN 10) wants to reach PC-B (VLAN 20):

1. PC-A sends to default gateway 192.168.10.1 (router sub-interface .10)
2. Frame goes on the trunk tagged VLAN 10 to the router
3. Router receives on Gi0/0.10, routes to Gi0/0.20 (different subnet)
4. Router sends frame back on the trunk tagged VLAN 20
5. Switch delivers to PC-B in VLAN 20
```

**Pros:** Simple, cheap (one router interface)
**Cons:** Single link is a bandwidth bottleneck; all inter-VLAN traffic goes through that one link

---

## Method 2: Layer 3 Switch with SVIs

**Concept:** A Layer 3 switch can route traffic internally without sending it to an external router. Create an **SVI (Switch Virtual Interface)** for each VLAN.

```
                ┌────────────────────────────────┐
                │      LAYER 3 SWITCH             │
                │  SVI VLAN 10: 192.168.10.1      │
                │  SVI VLAN 20: 192.168.20.1      │
                │  (routes internally!)           │
                │ Fa0/1(V10) Fa0/2(V20)           │
                └──┬──────────┬──────────────────┘
                 [PC-A]     [PC-B]
                VLAN 10    VLAN 20
```

### Configuration

```
! Enable IP routing on the switch
SW1(config)# ip routing
! This is the key command — without it, the switch is Layer 2 only

! Create SVIs
SW1(config)# interface vlan 10
SW1(config-if)# ip address 192.168.10.1 255.255.255.0
SW1(config-if)# no shutdown

SW1(config)# interface vlan 20
SW1(config-if)# ip address 192.168.20.1 255.255.255.0
SW1(config-if)# no shutdown
```

**That's it.** The switch routes between VLANs at hardware speed.

**SVI requirements:**
- The VLAN must exist on the switch (`vlan 10`)
- At least one port in that VLAN must be **up/up**
- The SVI must be `no shutdown`
- `ip routing` must be enabled

> 💡 **Why SVI > ROAS:** No bottleneck — traffic is switched in hardware (ASIC) at wire speed. The router-on-a-stick method forces all traffic through a single link.

---

## Method 3: Routed Port on L3 Switch

A Layer 3 switch port can act as a **router port** instead of a switch port:

```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# no switchport
! Converts from Layer 2 switch port to Layer 3 routed port
SW1(config-if)# ip address 10.0.0.1 255.255.255.252
SW1(config-if)# no shutdown
```

**Use case:** Point-to-point connections between Layer 3 switches or to external routers. The port acts exactly like a router interface.

---

## Comparison

| Feature | ROAS | L3 Switch (SVI) | Routed Port |
|---------|------|-----------------|-------------|
| Cost | Low (basic router) | Higher (L3 switch) | L3 switch |
| Performance | Limited (single link) | High (hardware switching) | High |
| Scalability | Low | High | Point-to-point |
| Complexity | Low | Low | Low |
| Best for | Small networks, labs | Enterprise | Uplinks, WAN |

---

## Troubleshooting Inter-VLAN Routing

**ROAS not working? Check:**
1. Physical interface is `no shutdown`
2. `encapsulation dot1Q <vlan-id>` matches the VLAN number
3. Switch port is actually trunking (`show interfaces trunk`)
4. VLANs are allowed on the trunk
5. Native VLAN matches on both sides
6. Hosts have correct default gateway

**SVI not working? Check:**
1. `ip routing` is enabled (`show ip route` should show connected routes)
2. VLAN exists on the switch
3. At least one port in the VLAN is up
4. SVI is `no shutdown` (`show ip interface brief` — check status)
5. Hosts have correct default gateway

```
! Useful verification commands
show ip route                  ! Are VLAN subnets in the routing table?
show ip interface brief        ! Are SVIs up/up?
show interfaces trunk          ! Is the trunk passing the right VLANs?
show vlan brief               ! Are ports in the right VLANs?
```

---

## Practice Questions

1. What is the purpose of inter-VLAN routing?
2. In ROAS, where are the VLAN gateway IPs configured?
3. What command enables routing on a Layer 3 switch?
4. What must be true for an SVI to be in the "up" state?
5. What does `encapsulation dot1Q 10` do on a router sub-interface?
6. Which method provides the best performance for inter-VLAN routing?

<details>
<summary>Answers</summary>

1. Allows communication between different VLANs (different broadcast domains)
2. On the router's sub-interfaces (e.g., Gi0/0.10, Gi0/0.20)
3. `ip routing`
4. The VLAN must exist, at least one port in that VLAN must be up, and the SVI must be `no shutdown`
5. Associates the sub-interface with VLAN 10 — it will process frames tagged with VLAN 10
6. Layer 3 switch with SVIs (hardware-based switching)
</details>

---

*← [Day 18 — VTP](Day18-VTP.md) | [Days 20-21 — Spanning Tree Protocol](Day20-21-STP.md) →*
