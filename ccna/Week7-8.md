# CCNA Study Guide — Weeks 7–8: IPv6, ACLs, NAT & WAN

## 🎯 Goals
- Understand IPv6 addressing, types, and configuration
- Configure and verify standard and extended ACLs (IPv4 & IPv6)
- Understand and configure NAT/PAT
- Understand WAN technologies and PPP/GRE
- Review and prepare for exam day

---

## Day-by-Day Plan

### Day 43–44 — IPv6 Addressing

**Why IPv6?**
- IPv4 exhaustion (~4.3 billion addresses)
- IPv6 = 128-bit addresses = 3.4 × 10^38 addresses
- No NAT needed (every device gets a public address)
- Built-in IPsec, simplified header, no broadcast (multicast/anycast instead)

**IPv6 Address Format:**
- 8 groups of 4 hex digits separated by colons
- Example: `2001:0DB8:0000:0000:0000:0000:0000:0001`
- Shortened: `2001:DB8::1` (leading zeros removed; one `::` for consecutive zero groups)

**IPv6 Address Types:**
| Type | Prefix | Description |
|------|--------|-------------|
| Global Unicast (GUA) | 2000::/3 | Public, routable (like IPv4 public) |
| Link-Local | FE80::/10 | Auto-generated, not routable, required on every interface |
| Unique Local | FC00::/7 (FD00::/8 used) | Private, like RFC 1918 |
| Multicast | FF00::/8 | One-to-many |
| Loopback | ::1/128 | Localhost |
| Unspecified | ::/128 | Like 0.0.0.0 |

**Important Multicast Addresses:**
| Address | Scope | Purpose |
|---------|-------|---------|
| FF02::1 | Link-local | All nodes |
| FF02::2 | Link-local | All routers |
| FF02::5 | Link-local | All OSPF routers |
| FF02::6 | Link-local | All OSPF DR/BDR |
| FF02::9 | Link-local | All RIPng routers |
| FF02::A | Link-local | All EIGRP routers |
| FF02::1:FF00:0/104 | Link-local | Solicited-node multicast |

**EUI-64 (Interface ID Generation):**
1. Take MAC address (e.g., AA:BB:CC:DD:EE:FF)
2. Split in half: AA:BB:CC | DD:EE:FF
3. Insert FFFE: AA:BB:CC:FF:FE:DD:EE:FF
4. Flip 7th bit (U/L bit): A8:BB:CC:FF:FE:DD:EE:FF
5. Result: `A8BB:CCFF:FEDD:EEFF`

**IPv6 Configuration:**
```
R1(config)# ipv6 unicast-routing              ! Enable IPv6 routing globally

R1(config)# interface Gi0/0
R1(config-if)# ipv6 address 2001:DB8:ACAD:1::1/64         ! Static GUA
R1(config-if)# ipv6 address FE80::1 link-local            ! Static link-local
R1(config-if)# no shutdown

! SLAAC (Stateless Address Autoconfiguration) — hosts auto-configure
! Router sends RA (Router Advertisement) with prefix info
! Host generates address using prefix + EUI-64 or random interface ID

! DHCPv6 Stateless (SLAAC for address, DHCPv6 for DNS/domain)
R1(config-if)# ipv6 nd other-config-flag

! DHCPv6 Stateful (DHCPv6 assigns address + DNS)
R1(config-if)# ipv6 nd managed-config-flag
```

**NDP (Neighbor Discovery Protocol) — replaces ARP:**
| Message | ICMPv6 Type | Purpose |
|---------|------------|---------|
| Router Solicitation (RS) | 133 | Host asks for router info |
| Router Advertisement (RA) | 134 | Router announces prefix, flags, lifetime |
| Neighbor Solicitation (NS) | 135 | Like ARP request; also DAD |
| Neighbor Advertisement (NA) | 136 | Like ARP reply |
| Redirect | 137 | Better next-hop notification |

**DAD (Duplicate Address Detection):**
- Before using an address, host sends NS to its own solicited-node multicast
- If someone responds with NA → duplicate detected → address not used

**Verification:**
```
show ipv6 interface brief
show ipv6 interface Gi0/0
show ipv6 route
show ipv6 neighbors
```

---

### Day 45–47 — Access Control Lists (ACLs)

**What are ACLs?**
- Ordered list of permit/deny rules applied to interfaces
- Filter traffic based on source/destination IP, protocol, port
- Processed top-down; first match wins
- Implicit `deny any` at the end of every ACL

**ACL Types:**
| Type | Number Range | Named | Matches On |
|------|-------------|-------|------------|
| Standard | 1–99, 1300–1999 | Yes | Source IP only |
| Extended | 100–199, 2000–2699 | Yes | Source/Dest IP, protocol, port |

**Standard ACL Configuration:**
```
! Numbered
R1(config)# access-list 10 permit 192.168.10.0 0.0.0.255
R1(config)# access-list 10 deny 192.168.20.0 0.0.0.255
R1(config)# access-list 10 permit any              ! Explicit permit (otherwise implicit deny)

! Named
R1(config)# ip access-list standard ALLOW_LAN
R1(config-std-nacl)# permit 192.168.10.0 0.0.0.255
R1(config-std-nacl)# deny any log

! Apply to interface
R1(config)# interface Gi0/0
R1(config-if)# ip access-group 10 in               ! Inbound
R1(config-if)# ip access-group ALLOW_LAN out        ! Outbound
```

**Standard ACL Placement:** As close to the **destination** as possible (since they only match source IP, placing near source would block too much).

**Extended ACL Configuration:**
```
! Numbered
R1(config)# access-list 100 permit tcp 192.168.10.0 0.0.0.255 host 10.0.0.5 eq 80
R1(config)# access-list 100 permit tcp 192.168.10.0 0.0.0.255 any eq 443
R1(config)# access-list 100 permit icmp any any
R1(config)# access-list 100 deny ip any any log

! Named
R1(config)# ip access-list extended WEB_ACCESS
R1(config-ext-nacl)# 10 permit tcp 192.168.10.0 0.0.0.255 any eq 80
R1(config-ext-nacl)# 20 permit tcp 192.168.10.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# 30 permit icmp any any echo
R1(config-ext-nacl)# 40 permit icmp any any echo-reply
R1(config-ext-nacl)# 50 deny ip any any log

! Apply
R1(config-if)# ip access-group WEB_ACCESS in
```

**Extended ACL Placement:** As close to the **source** as possible (to prevent unwanted traffic from crossing the network).

**Wildcard Masks:**
- Inverse of subnet mask
- 0 = must match | 1 = don't care
- Examples:
  - `0.0.0.0` = match exactly one host (use `host` keyword)
  - `0.0.0.255` = match /24 network
  - `0.0.3.255` = match /22 block
  - `255.255.255.255` = match any (use `any` keyword)

**Common ACL Keywords:**
```
host 10.0.0.1          →  10.0.0.1 0.0.0.0
any                    →  0.0.0.0 255.255.255.255
eq 80                  →  equal to port 80
gt 1023                →  greater than port 1023
lt 1024                →  less than port 1024
range 1024 65535       →  port range
neq 22                 →  not equal to port 22
established            →  TCP with ACK or RST bit set (return traffic)
log                    →  log matches to console/syslog
```

**ACL on VTY Lines (restrict SSH/Telnet access):**
```
R1(config)# access-list 5 permit 192.168.1.0 0.0.0.255
R1(config)# line vty 0 15
R1(config-line)# access-class 5 in
R1(config-line)# transport input ssh
```

**IPv6 ACLs:**
```
R1(config)# ipv6 access-list BLOCK_TELNET
R1(config-ipv6-acl)# deny tcp any any eq 23
R1(config-ipv6-acl)# permit ipv6 any any

R1(config)# interface Gi0/0
R1(config-if)# ipv6 traffic-filter BLOCK_TELNET in
```
Note: IPv6 ACLs are always named; no numbered option. Implicit entries include `permit icmp ... nd-na` and `permit icmp ... nd-ns` (for NDP).

**Editing Named ACLs (resequencing):**
```
R1(config)# ip access-list extended WEB_ACCESS
R1(config-ext-nacl)# no 30                            ! Delete line 30
R1(config-ext-nacl)# 25 permit tcp any any eq 22     ! Insert at sequence 25

! Resequence
R1(config)# ip access-list resequence WEB_ACCESS 10 10  ! Start at 10, increment 10
```

**Verification:**
```
show access-lists
show ip access-lists
show ip access-lists 100
show running-config | include access-list
show ip interface Gi0/0 | include access
```

---

### Day 48–49 — NAT (Network Address Translation)

**Why NAT?**
- Conserves IPv4 addresses (many private hosts share few public IPs)
- Provides a layer of security (internal addresses hidden)

**NAT Terminology:**
| Term | Meaning |
|------|---------|
| Inside Local | Private IP of internal host (192.168.1.10) |
| Inside Global | Public IP representing the internal host (203.0.113.5) |
| Outside Local | IP of external host as seen from inside (usually same as Outside Global) |
| Outside Global | Public IP of external host (8.8.8.8) |

**NAT Types:**
| Type | Description | Use Case |
|------|-------------|----------|
| Static NAT | 1:1 mapping (private ↔ public) | Servers that need consistent public IP |
| Dynamic NAT | Pool of public IPs; 1:1 but assigned dynamically | Group of hosts; limited public IPs |
| PAT (NAT Overload) | Many:1; uses port numbers to distinguish | Most common; entire network shares 1 public IP |

**Static NAT:**
```
R1(config)# ip nat inside source static 192.168.1.10 203.0.113.10

R1(config)# interface Gi0/0
R1(config-if)# ip nat inside

R1(config)# interface Gi0/1
R1(config-if)# ip nat outside
```

**Dynamic NAT:**
```
! Define the pool
R1(config)# ip nat pool MYPOOL 203.0.113.10 203.0.113.20 netmask 255.255.255.0

! Define which inside hosts can use NAT
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255

! Map ACL to pool
R1(config)# ip nat inside source list 1 pool MYPOOL

! Set interfaces
R1(config)# interface Gi0/0
R1(config-if)# ip nat inside
R1(config)# interface Gi0/1
R1(config-if)# ip nat outside
```

**PAT (NAT Overload) — Most Common:**
```
! Using interface IP (single public IP)
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
R1(config)# ip nat inside source list 1 interface Gi0/1 overload

! Using a pool with overload
R1(config)# ip nat inside source list 1 pool MYPOOL overload
```

**Verification:**
```
show ip nat translations
show ip nat statistics
debug ip nat                    ! Real-time NAT operations (use carefully)
clear ip nat translation *     ! Clear all translations
```

---

### Day 50–51 — WAN Technologies & VPNs

**WAN Connection Types:**
| Type | Description | Speed | Example |
|------|-------------|-------|---------|
| Leased Line | Dedicated point-to-point | T1 (1.544 Mbps), T3 (44.7 Mbps) | Corporate offices |
| Metro Ethernet | Ethernet-based MAN/WAN | 10 Mbps–100 Gbps | Carrier Ethernet |
| MPLS | Label-switching WAN | Various | Enterprise WAN |
| Broadband | Shared, consumer/business | DSL, Cable, Fiber | Branch offices |
| Cellular | 4G/5G wireless | Variable | Remote/mobile sites |
| Satellite | High-latency wireless | Various | Extremely remote |
| SD-WAN | Software-defined WAN | Various | Modern enterprise (overlay) |

**VPN Types:**
| Type | Description |
|------|-------------|
| Site-to-Site | Connects two networks (e.g., HQ ↔ Branch) |
| Remote Access | Individual users connect to corporate network |
| IPsec | Encrypts at Layer 3; site-to-site or remote access |
| SSL/TLS VPN | Browser-based or client; uses HTTPS (port 443) |
| GRE over IPsec | GRE tunnel for multicast/routing + IPsec for encryption |
| DMVPN | Dynamic Multipoint VPN; hub-and-spoke with dynamic spoke-to-spoke |

**GRE Tunnel Configuration:**
```
R1(config)# interface Tunnel0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
R1(config-if)# tunnel source Gi0/1                     ! Physical interface or IP
R1(config-if)# tunnel destination 203.0.113.2           ! Remote router's public IP
R1(config-if)# no shutdown

R2(config)# interface Tunnel0
R2(config-if)# ip address 10.10.10.2 255.255.255.252
R2(config-if)# tunnel source Gi0/1
R2(config-if)# tunnel destination 198.51.100.1
R2(config-if)# no shutdown

! Route traffic through the tunnel
R1(config)# ip route 172.16.0.0 255.255.0.0 10.10.10.2
```

---

### Day 52–53 — Network Services: DHCP, DNS, NTP, SNMP, Syslog

**DHCP Server (Router as DHCP server):**
```
R1(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.10    ! Reserve IPs
R1(config)# ip dhcp pool LAN_POOL
R1(dhcp-config)# network 192.168.10.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.10.1
R1(dhcp-config)# dns-server 8.8.8.8 8.8.4.4
R1(dhcp-config)# domain-name example.com
R1(dhcp-config)# lease 7                                           ! 7 days

! DHCP Relay (on interface facing DHCP clients, when server is remote)
R2(config)# interface Gi0/0
R2(config-if)# ip helper-address 192.168.1.1                       ! DHCP server IP
```

**Verification:**
```
show ip dhcp binding
show ip dhcp pool
show ip dhcp conflict
```

**DNS:**
```
R1(config)# ip domain-name example.com
R1(config)# ip name-server 8.8.8.8
R1(config)# ip domain-lookup                   ! Enable DNS resolution (default)
```

**NTP (Network Time Protocol):**
```
R1(config)# ntp server 216.239.35.0            ! Google NTP
R1(config)# ntp server 216.239.35.4

! Make this router an NTP server for local devices
R1(config)# ntp master 3                        ! Stratum 3

! Verification
show ntp status
show ntp associations
show clock
```

**Syslog:**
```
R1(config)# logging host 192.168.1.100         ! Syslog server IP
R1(config)# logging trap informational          ! Severity level 6 and above
R1(config)# logging source-interface Loopback0
R1(config)# service timestamps log datetime msec

! Syslog Severity Levels:
! 0-Emergency, 1-Alert, 2-Critical, 3-Error, 4-Warning, 5-Notification, 6-Informational, 7-Debugging
! Mnemonic: "Every Awesome Cisco Engineer Will Need Ice cream Daily"
```

**SNMP:**
```
! SNMPv2c (community strings — less secure)
R1(config)# snmp-server community READONLY ro
R1(config)# snmp-server community READWRITE rw
R1(config)# snmp-server host 192.168.1.100 READONLY

! SNMPv3 (recommended — authentication + encryption)
R1(config)# snmp-server group ADMIN v3 priv
R1(config)# snmp-server user admin ADMIN v3 auth sha AuthPass1 priv aes 128 PrivPass1
```

---

### Day 54–55 — Network Security Fundamentals

**Port Security:**
```
SW1(config)# interface Fa0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 2
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation shutdown      ! shutdown | restrict | protect

! Recovery from err-disabled
SW1(config)# errdisable recovery cause psecure-violation
SW1(config)# errdisable recovery interval 300                    ! 5 minutes
```

**DHCP Snooping:**
```
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10,20
SW1(config)# interface Gi0/1                    ! Uplink to DHCP server/router
SW1(config-if)# ip dhcp snooping trust

! All other ports are untrusted by default — rogue DHCP servers blocked
SW1(config)# interface range Fa0/1 - 24
SW1(config-if-range)# ip dhcp snooping limit rate 15    ! Max 15 DHCP packets/sec
```

**Dynamic ARP Inspection (DAI):**
```
SW1(config)# ip arp inspection vlan 10,20
SW1(config)# interface Gi0/1
SW1(config-if)# ip arp inspection trust          ! Trust uplink (validated by DHCP snooping table)
```

**SSH Configuration (disable Telnet):**
```
R1(config)# hostname R1
R1(config)# ip domain-name example.com
R1(config)# crypto key generate rsa modulus 2048
R1(config)# ip ssh version 2
R1(config)# username admin privilege 15 secret StrongP@ss!

R1(config)# line vty 0 15
R1(config-line)# transport input ssh
R1(config-line)# login local
```

**AAA Overview (Authentication, Authorization, Accounting):**
- Centralized security using RADIUS or TACACS+
- RADIUS: UDP 1812/1813 (or 1645/1646); encrypts password only; common for network access
- TACACS+: TCP 49; encrypts entire payload; common for device administration; Cisco proprietary

---

### Day 56 — Wireless Fundamentals

**802.11 Standards (CCNA scope):**
| Standard | Frequency | Max Speed | Channels |
|----------|-----------|-----------|----------|
| 802.11a | 5 GHz | 54 Mbps | Non-overlapping: 24 |
| 802.11b | 2.4 GHz | 11 Mbps | Non-overlapping: 1,6,11 |
| 802.11g | 2.4 GHz | 54 Mbps | Non-overlapping: 1,6,11 |
| 802.11n (Wi-Fi 4) | 2.4/5 GHz | 600 Mbps | MIMO |
| 802.11ac (Wi-Fi 5) | 5 GHz | 6.93 Gbps | MU-MIMO, wider channels |
| 802.11ax (Wi-Fi 6/6E) | 2.4/5/6 GHz | 9.6 Gbps | OFDMA, BSS coloring |

**WLAN Architecture:**
- **Autonomous APs**: Standalone; each configured individually; small deployments
- **Lightweight APs (LAPs) + WLC**: Centralized management via Wireless LAN Controller
  - CAPWAP tunnel (UDP 5246 control, 5247 data) between AP and WLC
  - Split-MAC: AP handles real-time (beacons, ACKs); WLC handles management (auth, roaming, QoS)
- **Cloud-managed** (e.g., Meraki): Controller in the cloud

**Wireless Security:**
| Protocol | Encryption | Authentication | Status |
|----------|-----------|----------------|--------|
| WEP | RC4 (weak) | Shared key | Deprecated ❌ |
| WPA | TKIP | PSK or 802.1X | Legacy |
| WPA2 | AES-CCMP | PSK or 802.1X | Current standard ✅ |
| WPA3 | AES-GCMP | SAE or 802.1X | Latest ✅ |

---

### Days 57–60 — Automation, SDN & Final Review

**Network Automation Concepts:**
- **Controller-based networking**: Centralized control plane (SDN controller)
- **Data plane**: Forwards traffic (switches/routers)
- **Control plane**: Makes forwarding decisions (routing protocols, STP)
- **Management plane**: Configuration and monitoring (SSH, SNMP, APIs)

**SDN Architecture:**
```
┌──────────────────────────┐
│   Application Layer       │  ← Business apps, network management
│   (Northbound API: REST)  │
├──────────────────────────┤
│   Control Layer           │  ← SDN Controller (Cisco DNA Center, OpenDaylight)
│   (Southbound API)        │
├──────────────────────────┤
│   Infrastructure Layer    │  ← Switches, routers, APs
│   (OpenFlow, NETCONF)     │
└──────────────────────────┘
```

**Cisco DNA Center:**
- Intent-based networking controller
- Automates provisioning, policy, assurance
- Uses NETCONF/YANG, REST APIs

**Configuration Management Tools:**
| Tool | Language | Agent | Push/Pull |
|------|----------|-------|-----------|
| Ansible | YAML/Python | Agentless (SSH) | Push |
| Puppet | Ruby/DSL | Agent-based | Pull |
| Chef | Ruby/DSL | Agent-based | Pull |
| SaltStack | YAML/Python | Agent or agentless | Push/Pull |

**REST APIs:**
- CRUD operations via HTTP methods:
  - GET = Read
  - POST = Create
  - PUT = Update/Replace
  - PATCH = Partial Update
  - DELETE = Delete
- Data formats: JSON (most common), XML

**JSON Example:**
```json
{
  "interface": {
    "name": "GigabitEthernet0/0",
    "ip-address": "192.168.1.1",
    "subnet-mask": "255.255.255.0",
    "status": "up"
  }
}
```

---

## 📝 Final Review Quiz — Weeks 7–8

1. How many bits in an IPv6 address?
2. What is the IPv6 link-local prefix?
3. What replaces ARP in IPv6?
4. Where should you place a standard ACL?
5. What is PAT?
6. What are the inside local and inside global addresses in NAT?
7. What port does SSH use?
8. What is the purpose of DHCP snooping?
9. Name two configuration management tools.
10. What does the SDN controller sit between?
11. What is the difference between WPA2-PSK and WPA2-Enterprise?
12. What command enables IPv6 routing?

<details>
<summary>Answers</summary>

1. 128 bits
2. FE80::/10
3. NDP (Neighbor Discovery Protocol) — specifically Neighbor Solicitation/Advertisement
4. Close to the destination
5. Port Address Translation — many internal IPs share one public IP using port numbers
6. Inside Local = private IP of internal host; Inside Global = public IP representing it
7. TCP 22
8. Prevents rogue DHCP servers; only trusted ports can send DHCP offers
9. Ansible, Puppet, Chef, SaltStack (any two)
10. The application layer (northbound) and infrastructure layer (southbound)
11. PSK uses a shared password; Enterprise uses 802.1X with a RADIUS server for per-user authentication
12. `ipv6 unicast-routing`
</details>

---

## 🎓 Exam Day Preparation

**Exam Details:**
- Exam: 200-301 CCNA
- Duration: 120 minutes
- Questions: ~100 (multiple choice, drag-and-drop, simulations)
- Passing: ~825/1000 (Cisco doesn't publish exact score)
- Cost: $330 USD
- Valid: 3 years

**Last-Minute Tips:**
1. **Subnetting**: Practice until instant — you'll get many questions
2. **OSI/TCP-IP layers**: Know which protocols/devices operate where
3. **OSPF**: Neighbor states, DR/BDR election, LSA types, area types
4. **ACLs**: Wildcard masks, placement rules, standard vs extended
5. **VLANs/Trunking**: 802.1Q, native VLAN, DTP, VTP
6. **NAT/PAT**: Inside/outside, local/global terminology
7. **STP**: Root bridge election, port roles, port states, PortFast/BPDU Guard
8. **IPv6**: Address types, SLAAC vs DHCPv6, NDP
9. **Security**: Port security, DHCP snooping, DAI, SSH config
10. **Automation**: REST APIs, JSON, SDN concepts, DNA Center

**Practice Resources:**
- [Boson ExSim CCNA](https://www.boson.com) — Best practice exams (paid)
- [Pearson CCNA Simulator](https://www.pearsonitcertification.com) — Official practice
- [Cisco Packet Tracer Labs](https://www.netacad.com) — Free lab environment
- [Jeremy's IT Lab](https://www.youtube.com/c/JeremysITLab) — Full free CCNA course
- [David Bombal](https://www.youtube.com/c/DavidBombal) — Labs and real-world scenarios
- [NetworkChuck](https://www.youtube.com/c/NetworkChuck) — Engaging explanations
- [Subnetting.net](https://subnetting.net) — Practice drills
- [CBT Nuggets](https://www.cbtnuggets.com) — Video training (paid)

---

## 📚 Complete CCNA Exam Topics (200-301)

| Domain | Weight |
|--------|--------|
| 1.0 Network Fundamentals | 20% |
| 2.0 Network Access | 20% |
| 3.0 IP Connectivity | 25% |
| 4.0 IP Services | 10% |
| 5.0 Security Fundamentals | 15% |
| 6.0 Automation and Programmability | 10% |

**Good luck, maky! 🎓🚀**
