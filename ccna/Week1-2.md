# CCNA Study Guide — Weeks 1–2: Network Fundamentals

## 🎯 Goals
- Understand the OSI and TCP/IP models in depth
- Identify network topologies, media types, and cabling standards
- Configure basic switch and router interfaces
- Understand IPv4 addressing, subnetting, and CIDR

---

## Day-by-Day Plan

### Day 1 — OSI Model Deep Dive
**Concepts:**
- 7 layers: Physical, Data Link, Network, Transport, Session, Presentation, Application
- Each layer's PDU (Protocol Data Unit): Bits → Frames → Packets → Segments → Data
- Encapsulation and de-encapsulation process
- How data flows from application to wire and back

**Key Details:**
| Layer | Name | PDU | Protocols/Devices | Function |
|-------|------|-----|-------------------|----------|
| 7 | Application | Data | HTTP, DNS, DHCP, FTP, SMTP | User interface to network |
| 6 | Presentation | Data | SSL/TLS, JPEG, ASCII | Data formatting, encryption |
| 5 | Session | Data | NetBIOS, RPC, PPTP | Session management |
| 4 | Transport | Segment | TCP, UDP | Reliable/unreliable delivery |
| 3 | Network | Packet | IP, ICMP, OSPF, EIGRP | Logical addressing, routing |
| 2 | Data Link | Frame | Ethernet, 802.11, ARP | MAC addressing, framing |
| 1 | Physical | Bits | Cables, hubs, repeaters | Signal transmission |

**Study Tips:**
- Mnemonic: "All People Seem To Need Data Processing" (Layer 7→1)
- Reverse: "Please Do Not Throw Sausage Pizza Away" (Layer 1→7)

---

### Day 2 — TCP/IP Model & Protocol Suite
**Concepts:**
- 4-layer model: Network Access → Internet → Transport → Application
- Mapping OSI layers to TCP/IP layers
- Key protocols per layer

**TCP vs UDP:**
| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, sequencing, ACKs | Best-effort, no ACKs |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Use Cases | HTTP, FTP, SSH, SMTP | DNS queries, VoIP, streaming, DHCP |
| Header Size | 20 bytes minimum | 8 bytes |
| Flow Control | Yes (windowing) | No |

**TCP 3-Way Handshake:**
1. Client → Server: SYN (seq=x)
2. Server → Client: SYN-ACK (seq=y, ack=x+1)
3. Client → Server: ACK (ack=y+1)

**TCP Connection Teardown (4-way):**
1. FIN → 2. ACK → 3. FIN → 4. ACK

**Common Port Numbers (MUST memorize):**
| Port | Protocol | Service |
|------|----------|---------|
| 20 | TCP | FTP Data |
| 21 | TCP | FTP Control |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP (server/client) |
| 69 | UDP | TFTP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 143 | TCP | IMAP |
| 161/162 | UDP | SNMP |
| 443 | TCP | HTTPS |
| 3389 | TCP | RDP |

---

### Day 3 — Network Topologies & Media
**Topologies:**
- **Star**: Central switch/hub; most common in LANs; single point of failure at center
- **Mesh**: Every device connects to every other; full mesh = n(n-1)/2 links; expensive but resilient
- **Bus**: Single backbone cable; legacy; collision-prone
- **Ring**: Data travels in one direction; token passing; FDDI uses dual ring for redundancy
- **Hybrid**: Combination (e.g., star-bus, star-ring)

**Cabling Standards:**
- **UTP (Unshielded Twisted Pair):**
  - Cat 5e: 1 Gbps, 100m
  - Cat 6: 1 Gbps (10 Gbps up to 55m), 100m
  - Cat 6a: 10 Gbps, 100m
  - Cat 7: 10 Gbps, shielded, 100m
  - Cat 8: 25/40 Gbps, 30m (data centers)

- **Fiber Optic:**
  - Single-mode (SMF): Small core (8-10μm), long distance (up to 80km+), laser light, yellow jacket
  - Multi-mode (MMF): Larger core (50-62.5μm), shorter distance (up to 2km), LED/VCSEL, orange/aqua jacket
  - Connectors: LC, SC, ST, MTRJ

- **Cable Types:**
  - Straight-through: Host→Switch, Router→Switch (T-568B both ends)
  - Crossover: Switch→Switch, Host→Host, Router→Router (T-568A one end, T-568B other)
  - Rollover/Console: PC→Router/Switch console port (reversed pinout)

**T-568B Pinout:** White-Orange, Orange, White-Green, Blue, White-Blue, Green, White-Brown, Brown

---

### Day 4 — IPv4 Addressing Fundamentals
**Binary-Decimal Conversion:**
- Each octet = 8 bits = values 0-255
- Practice: Convert 192.168.1.1 to binary: 11000000.10101000.00000001.00000001

**Address Classes:**
| Class | Range | Default Mask | Networks | Hosts/Network |
|-------|-------|-------------|----------|---------------|
| A | 1.0.0.0–126.255.255.255 | /8 (255.0.0.0) | 126 | 16,777,214 |
| B | 128.0.0.0–191.255.255.255 | /16 (255.255.0.0) | 16,384 | 65,534 |
| C | 192.0.0.0–223.255.255.255 | /24 (255.255.255.0) | 2,097,152 | 254 |
| D | 224.0.0.0–239.255.255.255 | N/A | Multicast | N/A |
| E | 240.0.0.0–255.255.255.255 | N/A | Experimental | N/A |

**Private Address Ranges (RFC 1918):**
- 10.0.0.0/8 (10.0.0.0 – 10.255.255.255)
- 172.16.0.0/12 (172.16.0.0 – 172.31.255.255)
- 192.168.0.0/16 (192.168.0.0 – 192.168.255.255)

**Special Addresses:**
- 127.0.0.0/8: Loopback
- 169.254.0.0/16: APIPA (link-local)
- 0.0.0.0: Default route / "this network"
- 255.255.255.255: Limited broadcast

---

### Day 5 — Subnetting Mastery (Part 1)
**Subnet Mask Basics:**
- Mask separates network bits from host bits
- CIDR notation: /24 = 255.255.255.0

**Subnetting Formula:**
- Subnets = 2^s (s = borrowed bits)
- Hosts per subnet = 2^h – 2 (h = remaining host bits; subtract network & broadcast)

**Subnetting Cheat Sheet (Class C /24 base):**
| CIDR | Mask | Subnets | Hosts | Block Size |
|------|------|---------|-------|------------|
| /25 | 255.255.255.128 | 2 | 126 | 128 |
| /26 | 255.255.255.192 | 4 | 62 | 64 |
| /27 | 255.255.255.224 | 8 | 30 | 32 |
| /28 | 255.255.255.240 | 16 | 14 | 16 |
| /29 | 255.255.255.248 | 32 | 6 | 8 |
| /30 | 255.255.255.252 | 64 | 2 | 4 |
| /31 | 255.255.255.254 | 128 | 2* | 2 |
| /32 | 255.255.255.255 | 256 | 1 | 1 |

**/31 Note:** Point-to-point links only (RFC 3021), no broadcast address.

**Subnetting Practice Example:**
Given: 192.168.10.0/26
- Block size: 256 – 192 = 64
- Subnet 0: 192.168.10.0 – 192.168.10.63 (usable: .1–.62, broadcast: .63)
- Subnet 1: 192.168.10.64 – 192.168.10.127 (usable: .65–.126, broadcast: .127)
- Subnet 2: 192.168.10.128 – 192.168.10.191 (usable: .129–.190, broadcast: .191)
- Subnet 3: 192.168.10.192 – 192.168.10.255 (usable: .193–.254, broadcast: .255)

---

### Day 6 — Subnetting Mastery (Part 2) & VLSM
**Variable Length Subnet Masking (VLSM):**
- Allows different subnet sizes in the same network
- Allocate largest subnets first, then fill remaining space with smaller ones

**VLSM Example:**
Given 192.168.1.0/24, create subnets for:
- LAN A: 50 hosts → /26 (62 usable) → 192.168.1.0/26
- LAN B: 25 hosts → /27 (30 usable) → 192.168.1.64/27
- LAN C: 10 hosts → /28 (14 usable) → 192.168.1.96/28
- WAN link: 2 hosts → /30 (2 usable) → 192.168.1.112/30

**Supernetting/Summarization (Route Aggregation):**
- Combine multiple subnets into one summary route
- Example: 192.168.0.0/24 through 192.168.3.0/24 → 192.168.0.0/22
- Find common bits: all share first 22 bits

---

### Day 7 — Ethernet Fundamentals & Switching Basics
**Ethernet Standards:**
| Standard | Speed | Media | Distance |
|----------|-------|-------|----------|
| 10BASE-T | 10 Mbps | Cat 3+ UTP | 100m |
| 100BASE-TX | 100 Mbps | Cat 5+ UTP | 100m |
| 1000BASE-T | 1 Gbps | Cat 5e+ UTP | 100m |
| 1000BASE-SX | 1 Gbps | MMF | 550m |
| 1000BASE-LX | 1 Gbps | SMF | 5km |
| 10GBASE-T | 10 Gbps | Cat 6a+ UTP | 100m |
| 10GBASE-SR | 10 Gbps | MMF | 400m |
| 10GBASE-LR | 10 Gbps | SMF | 10km |

**Ethernet Frame Format:**
```
| Preamble (7B) | SFD (1B) | Dest MAC (6B) | Src MAC (6B) | Type/Length (2B) | Data (46-1500B) | FCS (4B) |
```
- Minimum frame: 64 bytes | Maximum frame: 1518 bytes (1522 with 802.1Q tag)
- Runt: <64 bytes | Giant: >1518 bytes

**MAC Addresses:**
- 48 bits (6 bytes), written as hex: AA:BB:CC:DD:EE:FF
- First 24 bits: OUI (vendor) | Last 24 bits: device-specific
- Broadcast MAC: FF:FF:FF:FF:FF:FF

**Switch Operations:**
1. **Learn**: Record source MAC + ingress port in MAC address table
2. **Forward**: If destination MAC is known, send out that port only (unicast)
3. **Flood**: If destination MAC is unknown, send out all ports except ingress
4. **Filter**: Never forward a frame back out the port it came in on

---

### Days 8–10 — Initial Switch & Router Configuration

**Accessing the CLI:**
- Console cable (rollover) → Terminal emulator (PuTTY, screen, minicom)
- Speed: 9600 baud, 8-N-1

**CLI Modes:**
| Mode | Prompt | Access |
|------|--------|--------|
| User EXEC | `Switch>` | Default on login |
| Privileged EXEC | `Switch#` | `enable` |
| Global Config | `Switch(config)#` | `configure terminal` |
| Interface Config | `Switch(config-if)#` | `interface <type> <num>` |
| Line Config | `Switch(config-line)#` | `line console 0` / `line vty 0 15` |

**Essential Initial Configuration:**
```
! Hostname
Switch(config)# hostname SW1

! Secure privileged EXEC
SW1(config)# enable secret MyS3cret!

! Secure console
SW1(config)# line console 0
SW1(config-line)# password ConPass1
SW1(config-line)# login
SW1(config-line)# logging synchronous
SW1(config-line)# exec-timeout 5 0

! Secure VTY (remote access)
SW1(config)# line vty 0 15
SW1(config-line)# password VtyPass1
SW1(config-line)# login
SW1(config-line)# transport input ssh

! Encrypt passwords in config
SW1(config)# service password-encryption

! Banner
SW1(config)# banner motd # Authorized Access Only! #

! Management VLAN IP (for switch)
SW1(config)# interface vlan 1
SW1(config-if)# ip address 192.168.1.2 255.255.255.0
SW1(config-if)# no shutdown

! Default gateway (switch)
SW1(config)# ip default-gateway 192.168.1.1

! Save
SW1# copy running-config startup-config
```

**Router Interface Configuration:**
```
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip address 192.168.1.1 255.255.255.0
Router(config-if)# description LAN Connection
Router(config-if)# no shutdown

Router(config)# interface GigabitEthernet0/1
Router(config-if)# ip address 10.0.0.1 255.255.255.252
Router(config-if)# description WAN to ISP
Router(config-if)# no shutdown
```

**Verification Commands:**
```
show running-config
show startup-config
show ip interface brief
show interfaces
show mac address-table
show version
show vlan brief
```

---

### Days 11–14 — Labs & Review

**Lab 1: Cable Identification**
- Identify straight-through, crossover, and rollover cables
- Determine correct cable for each connection scenario

**Lab 2: Subnetting Drill**
- Subnet 10.0.0.0/8 into /16s, then /24s
- Given a host count requirement, determine the correct mask
- Identify network address, broadcast address, usable range, next subnet

**Lab 3: Basic Device Configuration (Packet Tracer / GNS3 / EVE-NG)**
- Configure hostname, passwords, banner on switch and router
- Assign IP addresses to interfaces
- Verify connectivity with ping
- Save configuration

**Lab 4: Packet Analysis (Wireshark)**
- Capture and examine an Ethernet frame
- Identify source/destination MAC, EtherType, payload
- Capture a TCP 3-way handshake
- Identify SYN, SYN-ACK, ACK and port numbers

---

## 📝 Review Quiz — Weeks 1–2

1. At which OSI layer does a switch primarily operate?
2. What is the TCP 3-way handshake sequence?
3. What port does HTTPS use?
4. How many usable hosts in a /27 network?
5. What is the broadcast address for 172.16.10.64/26?
6. What cable type connects a PC to a switch?
7. What command saves the running configuration?
8. What is the difference between `enable password` and `enable secret`?
9. A frame destined for FF:FF:FF:FF:FF:FF will be _____ by the switch.
10. Convert 11001000.00000001.00001010.11111110 to decimal.

<details>
<summary>Answers</summary>

1. Layer 2 (Data Link)
2. SYN → SYN-ACK → ACK
3. TCP 443
4. 30 (2^5 – 2)
5. 172.16.10.127
6. Straight-through
7. `copy running-config startup-config`
8. `enable secret` uses MD5 hash; `enable password` is plaintext (or type 7 weak encryption)
9. Flooded out all ports except the ingress port
10. 200.1.10.254
</details>

---

## 📚 Resources
- [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) — Free network simulator
- [Subnetting Practice](https://subnettingpractice.com) — Interactive drills
- [Professor Messer Network+](https://www.professormesser.com) — Free video lectures
- Jeremy's IT Lab CCNA (YouTube) — Full free CCNA course
- David Bombal (YouTube/Udemy) — Hands-on labs and practice
