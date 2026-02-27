# Day 16: VLAN Configuration

## 🎯 What You'll Learn
How to create VLANs, assign ports, configure voice VLANs, and verify everything on Cisco switches.

---

## Creating VLANs

```
SW1# configure terminal

! Create VLAN 10 for Sales
SW1(config)# vlan 10
SW1(config-vlan)# name SALES
SW1(config-vlan)# exit

! Create VLAN 20 for Engineering
SW1(config)# vlan 20
SW1(config-vlan)# name ENGINEERING
SW1(config-vlan)# exit

! Create VLAN 30 for Management
SW1(config)# vlan 30
SW1(config-vlan)# name MANAGEMENT
SW1(config-vlan)# exit

! Create VLAN 99 for Native (trunk use)
SW1(config)# vlan 99
SW1(config-vlan)# name NATIVE_TRUNK
SW1(config-vlan)# exit
```

**Key points:**
- VLAN is created the moment you enter `vlan <id>`
- Name is optional but **highly recommended** for readability
- If you skip the name, it defaults to `VLAN00xx` (e.g., VLAN0010)
- VLANs are stored in `vlan.dat` on flash (not in running-config for normal range)

---

## Assigning Ports to VLANs

### Single Port
```
SW1(config)# interface FastEthernet0/1
SW1(config-if)# switchport mode access
! Forces the port into access mode — carries only ONE VLAN

SW1(config-if)# switchport access vlan 10
! Assigns this port to VLAN 10
```

### Range of Ports
```
SW1(config)# interface range FastEthernet0/2 - 5
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 20
! Ports Fa0/2, Fa0/3, Fa0/4, Fa0/5 are now all in VLAN 20
```

### Non-Contiguous Range
```
SW1(config)# interface range Fa0/6 - 8, Fa0/15, Fa0/20 - 22
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 30
```

> 💡 **What happens if you assign a port to a VLAN that doesn't exist?**
> The switch **automatically creates the VLAN** — but without a name. This can cause confusion, so always create VLANs explicitly first.

---

## Access Port vs Trunk Port

| Feature | Access Port | Trunk Port |
|---------|------------|------------|
| VLANs carried | ONE | Multiple |
| Tagging | None (untagged) | 802.1Q tagged |
| Connected to | End devices (PCs, printers, phones) | Switches, routers |
| Command | `switchport mode access` | `switchport mode trunk` |

**An access port:**
- Belongs to exactly one VLAN
- All traffic in/out is untagged
- The end device doesn't know about VLANs

**A trunk port:**
- Carries traffic for multiple VLANs
- Tags frames with VLAN IDs (802.1Q)
- Used between switches and between switch↔router

---

## Voice VLAN Configuration

An IP phone needs QoS prioritization. The voice VLAN lets you separate voice traffic from data traffic on the same physical port.

```
SW1(config)# interface FastEthernet0/6
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
! Data VLAN for the PC behind the phone

SW1(config-if)# switchport voice vlan 50
! Voice VLAN for the IP phone

! Optional: set QoS trust on the port
SW1(config-if)# mls qos trust cos
```

**How it works:**
```
[PC] ──data VLAN 10──→ [IP Phone] ──voice VLAN 50──→ [Switch Fa0/6]
                                   ──data VLAN 10──→

The phone tags its own traffic with VLAN 50.
PC traffic passes through the phone untagged (VLAN 10).
The switch port carries both VLANs.
```

The phone learns its VLAN from CDP or LLDP messages sent by the switch.

---

## Deleting and Modifying VLANs

```
! Delete a VLAN
SW1(config)# no vlan 20

! ⚠️ WARNING: Ports still assigned to VLAN 20 become INACTIVE!
! They won't forward any traffic until reassigned to an existing VLAN.
! The ports show up but don't work. This is a common exam trap.

! Rename a VLAN
SW1(config)# vlan 10
SW1(config-vlan)# name SALES_DEPT
```

**Removing a port from a VLAN:**
```
SW1(config)# interface Fa0/1
SW1(config-if)# no switchport access vlan
! Port returns to VLAN 1 (default)
```

---

## Best Practices

1. **Don't use VLAN 1** for any user or management traffic
2. **Name every VLAN** — `show vlan brief` is useless with unnamed VLANs
3. **Shut down unused ports** and assign them to a "black hole" VLAN
4. **Document your VLAN plan** — which VLANs, which ports, which subnets

```
! Security: Disable unused ports
SW1(config)# interface range Fa0/10 - 24
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 999
SW1(config-if-range)# shutdown

! VLAN 999 exists but connects to nothing — a "parking lot" VLAN
```

---

## Verification Commands

### show vlan brief
```
SW1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- --------------------------
1    default                          active    Fa0/9
10   SALES                            active    Fa0/1
20   ENGINEERING                      active    Fa0/2, Fa0/3, Fa0/4, Fa0/5
30   MANAGEMENT                       active    Fa0/6, Fa0/7, Fa0/8
99   NATIVE_TRUNK                     active
999  BLACKHOLE                        active    Fa0/10-24
```

> 💡 **Note:** Trunk ports do NOT show up in `show vlan brief`. Use `show interfaces trunk` instead.

### show vlan id
```
SW1# show vlan id 10
! Shows specific VLAN details
```

### show interfaces switchport
```
SW1# show interfaces Fa0/1 switchport
Name: Fa0/1
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Administrative Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 10 (SALES)
Trunking Native Mode VLAN: 1 (default)
Voice VLAN: 50
```

---

## Practice Questions

1. What command creates VLAN 25?
2. How do you assign port Fa0/3 to VLAN 25?
3. What happens to ports in VLAN 20 if you delete VLAN 20?
4. Where are normal-range VLANs stored?
5. Do trunk ports appear in `show vlan brief`?
6. What two VLANs does a voice VLAN port carry?

<details>
<summary>Answers</summary>

1. `vlan 25` (from global config)
2. `interface Fa0/3` → `switchport mode access` → `switchport access vlan 25`
3. They become inactive — no traffic is forwarded until reassigned
4. In `vlan.dat` on flash memory
5. No — use `show interfaces trunk` to see trunk ports
6. The data VLAN (access VLAN) and the voice VLAN
</details>

---

*← [Day 15 — VLAN Concepts](Day15-VLAN-Concepts.md) | [Day 17 — Trunking](Day17-Trunking.md) →*
