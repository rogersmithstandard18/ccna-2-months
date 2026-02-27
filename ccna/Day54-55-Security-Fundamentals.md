# Days 54-55: Security Fundamentals

## 🎯 What You'll Learn
Common threats, port security, DHCP snooping, Dynamic ARP Inspection, AAA, and security best practices for the CCNA exam.

---

## Common Network Threats

| Threat | Description | Mitigation |
|--------|-------------|-----------|
| **MAC Flooding** | Attacker floods switch with fake MACs → CAM table fills → switch acts like a hub | Port security |
| **DHCP Spoofing** | Rogue DHCP server gives clients wrong gateway → MITM attack | DHCP snooping |
| **ARP Spoofing/Poisoning** | Attacker sends fake ARP replies → redirects traffic through attacker | Dynamic ARP Inspection (DAI) |
| **VLAN Hopping** | Attacker crafts double-tagged frames to reach other VLANs | Disable DTP, change native VLAN |
| **CDP/LLDP Reconnaissance** | Attacker learns device info (model, IP, IOS version) from CDP | Disable CDP on access ports |
| **Brute Force** | Repeatedly trying passwords | AAA, login block-for |
| **DoS/DDoS** | Overwhelm a device/service with traffic | Rate limiting, ACLs, upstream filtering |
| **Man-in-the-Middle** | Intercepting communication between two parties | Encryption, DAI, DHCP snooping |
| **Social Engineering** | Manipulating humans to reveal credentials | Security awareness training |

---

## Port Security

Limits the number of MAC addresses allowed on a switch port. Prevents MAC flooding attacks.

### Configuration
```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
! Port security only works on ACCESS ports (not trunk)

SW1(config-if)# switchport port-security
! Enable port security (default: max 1 MAC)

SW1(config-if)# switchport port-security maximum 2
! Allow up to 2 MAC addresses

SW1(config-if)# switchport port-security mac-address sticky
! Dynamically learn and "stick" MACs to the running config

SW1(config-if)# switchport port-security violation restrict
! What to do when violated
```

### Violation Modes

| Mode | Action | Drops Traffic | Logs | Port State |
|------|--------|:------------:|:----:|-----------|
| **Shutdown** (default) | Err-disables port | ✅ | ✅ | err-disabled |
| **Restrict** | Drops violating frames | ✅ | ✅ | Up |
| **Protect** | Silently drops | ✅ | ❌ | Up |

### Recovering from err-disabled
```
SW1(config)# interface Gi0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown

! Or enable automatic recovery
SW1(config)# errdisable recovery cause psecure-violation
SW1(config)# errdisable recovery interval 300
! Auto-recover after 300 seconds
```

### Verification
```
show port-security                         ! Summary of all ports
show port-security interface Gi0/1         ! Specific port details
show port-security address                 ! Learned MAC addresses
```

---

## DHCP Snooping

Prevents rogue DHCP servers. The switch inspects DHCP messages and only allows DHCP responses from **trusted** ports.

```
                    [Legit DHCP Server]
                          │ (trusted port)
                       [SWITCH]
                      /    |    \
                 [PC1]  [PC2]  [Attacker's DHCP]
              untrusted  untrusted  untrusted
                                    ↑ BLOCKED!
```

### Configuration
```
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10,20
! Enable for specific VLANs

! Trust the uplink to the legitimate DHCP server
SW1(config)# interface GigabitEthernet0/24
SW1(config-if)# ip dhcp snooping trust

! All other ports are UNTRUSTED by default
! They can send DHCP Discover/Request but NOT Offer/Ack

! Optional: rate limit DHCP on access ports (prevents DoS)
SW1(config)# interface range Gi0/1-23
SW1(config-if-range)# ip dhcp snooping limit rate 6
! Max 6 DHCP packets per second
```

**DHCP snooping builds a binding table:**
```
show ip dhcp snooping binding

MacAddress          IpAddress     Lease(sec)  Type     VLAN  Interface
AA:BB:CC:DD:EE:01   192.168.1.10  86400       dhcp-snooping  10  Gi0/1
AA:BB:CC:DD:EE:02   192.168.1.11  86400       dhcp-snooping  10  Gi0/2
```

This binding table is used by **DAI** and **IP Source Guard**.

---

## DAI (Dynamic ARP Inspection)

Validates ARP packets against the DHCP snooping binding table. Prevents ARP spoofing.

```
SW1(config)# ip arp inspection vlan 10,20

! Trust uplinks (same ports trusted for DHCP snooping)
SW1(config)# interface Gi0/24
SW1(config-if)# ip arp inspection trust

! All untrusted ports: ARP packets are validated against the DHCP snooping binding table
! If source MAC + source IP doesn't match a binding → DROP the ARP packet
```

**Prerequisite:** DHCP snooping must be enabled first (DAI uses its binding table).

For static IP hosts (no DHCP binding), create an ARP ACL:
```
SW1(config)# arp access-list STATIC_HOSTS
SW1(config-arp-nacl)# permit ip host 192.168.1.50 mac host AABB.CCDD.EE50
SW1(config)# ip arp inspection filter STATIC_HOSTS vlan 10
```

---

## AAA (Authentication, Authorization, Accounting)

| Component | Question | Example |
|-----------|----------|---------|
| **Authentication** | Who are you? | Username/password, certificate |
| **Authorization** | What can you do? | Privilege levels, command sets |
| **Accounting** | What did you do? | Log of commands, session duration |

### AAA Servers
| Protocol | Port | Encryption | Best For |
|----------|------|-----------|----------|
| **RADIUS** | 1812/1813 (or 1645/1646) | Password only | Network access (802.1X, VPN) |
| **TACACS+** | 49 | Entire packet | Device administration (CLI access) |

**RADIUS:** Open standard, combines authentication and authorization, encrypts only the password.
**TACACS+:** Cisco developed, separates AAA functions, encrypts the full packet. Preferred for managing network device CLI access.

### Basic AAA Configuration
```
SW1(config)# aaa new-model
! Enables AAA — disables old line-level authentication

SW1(config)# tacacs server MYSERVER
SW1(config-server-tacacs)# address ipv4 10.0.0.100
SW1(config-server-tacacs)# key TACACS_Secret

SW1(config)# aaa authentication login default group tacacs+ local
! Try TACACS+ first; fall back to local database if server unreachable

SW1(config)# username admin privilege 15 secret AdminPass123
! Local fallback account
```

---

## 802.1X (Port-Based Network Access Control)

Authenticates devices before granting network access on a switch port.

```
[Supplicant]  ←→  [Authenticator]  ←→  [Auth Server]
  (PC/device)       (Switch)            (RADIUS)

1. PC connects to switch port
2. Switch blocks all traffic except 802.1X (EAP)
3. PC provides credentials → switch relays to RADIUS
4. RADIUS approves → switch opens the port
5. If denied → port stays blocked (or goes to guest VLAN)
```

---

## Security Best Practices Checklist

```
✅ Disable unused ports:          switchport mode access → shutdown
✅ Assign unused ports to a "black hole" VLAN
✅ Enable port security on access ports
✅ Enable DHCP snooping on all VLANs
✅ Enable DAI on all VLANs
✅ Disable CDP/LLDP on access ports: no cdp enable
✅ Disable DTP on access ports:     switchport nonegotiate
✅ Change native VLAN from VLAN 1:  switchport trunk native vlan 999
✅ Use SSH, not Telnet:             transport input ssh
✅ Set enable secret (not enable password)
✅ Use AAA with TACACS+ or RADIUS
✅ Configure login banners (no personal info)
✅ Enable NTP + logging with timestamps
✅ Set exec-timeout on console/VTY lines
```

---

## Practice Questions

1. What are the three port security violation modes?
2. Which violation mode is the default?
3. What does DHCP snooping prevent?
4. What table does DAI use to validate ARP packets?
5. What is the difference between RADIUS and TACACS+?
6. What does 802.1X authenticate?
7. Why should you change the native VLAN from VLAN 1?

<details>
<summary>Answers</summary>

1. Shutdown, Restrict, Protect
2. Shutdown (err-disables the port)
3. Rogue DHCP servers from responding to clients with false information
4. The DHCP snooping binding table (MAC-to-IP mappings)
5. RADIUS encrypts only the password and combines auth+authz; TACACS+ encrypts the full packet and separates authentication, authorization, and accounting
6. Devices (supplicants) before allowing them access to the network via a switch port
7. To prevent VLAN hopping attacks (double-tagging exploits the default native VLAN 1)
</details>

---

*← [Days 52-53 — Network Services](Day52-53-Network-Services.md) | [Days 56-57 — Wireless Networking](Day56-57-Wireless.md) →*
