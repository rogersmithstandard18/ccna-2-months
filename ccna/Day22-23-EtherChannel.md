# Days 22-23: EtherChannel

## 🎯 What You'll Learn
How to bundle multiple physical links into one logical link for bandwidth and redundancy, LACP vs PAgP, configuration, load balancing, and troubleshooting.

---

## Why EtherChannel?

You have two switches connected by a single GigabitEthernet link. You need more bandwidth and redundancy. Adding a second cable creates a **loop** — STP blocks it!

```
Without EtherChannel:
SW1 ═══ Gi0/1 (active) ═══ SW2     1 Gbps
SW1 --- Gi0/2 (BLOCKED) --- SW2     STP blocks the redundant link

With EtherChannel:
SW1 ═══ Po1 (Gi0/1 + Gi0/2) ═══ SW2     2 Gbps logical link!
```

**EtherChannel bundles 2-8 physical links into ONE logical link.** STP sees one link — no blocking. You get both bandwidth aggregation and redundancy.

**Benefits:**
- **More bandwidth:** 2× to 8× the single-link speed
- **Redundancy:** If one link fails, traffic uses the remaining links
- **No STP blocking:** STP sees one logical link
- **Load balancing:** Traffic distributed across member links

---

## Negotiation Protocols

| Protocol | Standard | Modes | Notes |
|----------|----------|-------|-------|
| **LACP** | IEEE 802.3ad | Active / Passive | Open standard ← **Recommended** |
| **PAgP** | Cisco proprietary | Desirable / Auto | Cisco only |
| **Static** (mode on) | None | On | No negotiation — risky |

### LACP (Link Aggregation Control Protocol)
- **Active:** Initiates LACP negotiation
- **Passive:** Responds to LACP, doesn't initiate

| SW1 | SW2 | EtherChannel? |
|-----|-----|---------------|
| Active | Active | ✅ Yes |
| Active | Passive | ✅ Yes |
| Passive | Passive | ❌ No (both waiting) |

### PAgP (Port Aggregation Protocol)
- **Desirable:** Initiates PAgP negotiation
- **Auto:** Responds to PAgP, doesn't initiate

| SW1 | SW2 | EtherChannel? |
|-----|-----|---------------|
| Desirable | Desirable | ✅ Yes |
| Desirable | Auto | ✅ Yes |
| Auto | Auto | ❌ No (both waiting) |

### Static (mode on)
- No negotiation protocol — just forces the channel
- Both sides **must** be `on`
- **Dangerous:** No health checks; a misconfigured link stays in the channel

> 💡 **Best practice:** Use **LACP active** on both sides. It negotiates properly and detects problems.

---

## Configuration

### LACP EtherChannel

```
! SW1 — Configure member interfaces first
SW1(config)# interface range GigabitEthernet0/1 - 2
SW1(config-if-range)# channel-group 1 mode active
! Creates Port-channel 1 using LACP active mode
SW1(config-if-range)# no shutdown

! Configure the logical interface
SW1(config)# interface Port-channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# switchport trunk allowed vlan 10,20,30,99

! SW2 — Same, but can be active or passive
SW2(config)# interface range GigabitEthernet0/1 - 2
SW2(config-if-range)# channel-group 1 mode active
SW2(config-if-range)# no shutdown

SW2(config)# interface Port-channel 1
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk native vlan 99
SW2(config-if)# switchport trunk allowed vlan 10,20,30,99
```

### PAgP EtherChannel

```
SW1(config)# interface range Gi0/1 - 2
SW1(config-if-range)# channel-group 2 mode desirable

SW2(config)# interface range Gi0/1 - 2
SW2(config-if-range)# channel-group 2 mode desirable
```

### Layer 3 EtherChannel

```
SW1(config)# interface range Gi0/1 - 2
SW1(config-if-range)# no switchport
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# no shutdown

SW1(config)# interface Port-channel 1
SW1(config-if)# no switchport
SW1(config-if)# ip address 10.0.0.1 255.255.255.252
```

---

## Requirements — All Members MUST Match

**If any of these don't match, the EtherChannel won't form or will be suspended:**

| Setting | Must Match? |
|---------|------------|
| Speed | ✅ All same speed |
| Duplex | ✅ All same duplex |
| VLAN mode (access/trunk) | ✅ |
| Access VLAN (if access) | ✅ |
| Trunk native VLAN | ✅ |
| Trunk allowed VLANs | ✅ |
| STP settings | ✅ |

> 💡 **Common mistake:** Configuring settings on the Port-channel interface AFTER creating it. Best practice: configure member interfaces identically first, then apply channel-group, then configure the Port-channel interface.

**What happens on mismatch:**
```
%EC-5-CANNOT_BUNDLE2: Gi0/2 is not compatible with Po1 and will be suspended
```

---

## Load Balancing

EtherChannel doesn't split one flow across links — it assigns entire flows to links based on a hash.

```
! View current method
SW1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
  src-dst-ip

! Change method
SW1(config)# port-channel load-balance src-dst-ip
```

**Load balancing methods:**
| Method | Hash Based On | Best For |
|--------|--------------|----------|
| src-mac | Source MAC | Many sources to one destination |
| dst-mac | Destination MAC | One source to many destinations |
| src-dst-mac | Both MACs | General L2 traffic |
| src-ip | Source IP | Many sources to one destination |
| dst-ip | Destination IP | One source to many destinations |
| **src-dst-ip** | Both IPs | **Best general choice** |
| src-port | Source TCP/UDP port | |
| dst-port | Dest TCP/UDP port | |

> 💡 **Important:** If all traffic has the same source and destination IP (e.g., one server talking to one client), ALL traffic goes over ONE link regardless of how many links are in the channel. Use `src-dst-ip` or port-based balancing for best distribution.

---

## Verification

### show etherchannel summary (most useful)
```
SW1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use

Group  Port-channel  Protocol    Ports
------+-------------+-----------+--------------------
1      Po1(SU)        LACP      Gi0/1(P)  Gi0/2(P)

SU = Layer2, in Use
P  = Bundled (working!)
```

**Flag meanings:**
| Flag | Meaning |
|------|---------|
| P | Bundled in port-channel ✅ |
| D | Down |
| s | Suspended (mismatch!) |
| I | Stand-alone (not bundled) |
| H | Hot-standby (LACP — beyond max active links) |

### Other verification
```
show etherchannel port-channel      ! Detailed channel info
show etherchannel detail            ! Everything
show interfaces port-channel 1      ! Stats for logical interface
show lacp neighbor                  ! LACP partner info
show pagp neighbor                  ! PAgP partner info
```

---

## Troubleshooting EtherChannel

**Channel won't form:**
1. Check mode compatibility (active↔active ✅, passive↔passive ❌)
2. Verify all interfaces have matching speed/duplex
3. Verify all interfaces have matching VLAN configuration
4. Check for protocol mismatch (LACP on one side, PAgP on other = fail)
5. Check `show etherchannel summary` for (s) suspended ports

**Performance seems low:**
1. Check load balancing method (`show etherchannel load-balance`)
2. If all traffic uses one link, change to `src-dst-ip` or port-based
3. Verify all links are (P) bundled, not (D) or (s)

---

## Practice Questions

1. How many physical links can an EtherChannel bundle?
2. What happens if LACP is set to passive on both sides?
3. What configuration must match on all EtherChannel member ports?
4. What does the `(s)` flag mean in `show etherchannel summary`?
5. What is the best general load-balancing method?
6. Can you mix LACP and PAgP on the same EtherChannel?
7. Why is LACP preferred over static `mode on`?

<details>
<summary>Answers</summary>

1. 2 to 8 physical links
2. No EtherChannel forms — both sides are passive, neither initiates
3. Speed, duplex, VLAN mode (access/trunk), VLAN assignments, native VLAN, allowed VLANs
4. Suspended — a configuration mismatch was detected
5. `src-dst-ip` — considers both source and destination IP for best distribution
6. No — both sides must use the same protocol
7. LACP provides health monitoring and negotiation; `mode on` forces the channel with no checks, which can cause issues if the other side is misconfigured
</details>

---

*← [Days 20-21 — STP](Day20-21-STP.md) | [Day 29 — How Routing Works](Day29-Routing-Concepts.md) →*
