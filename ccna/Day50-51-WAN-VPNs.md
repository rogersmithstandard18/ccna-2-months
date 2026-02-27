# Days 50-51: WAN Technologies & VPNs

## 🎯 What You'll Learn
WAN connection types, VPN technologies, GRE tunnels, and SD-WAN concepts.

---

## WAN Connection Types

| Type | Speed | Description | Use Case |
|------|-------|-------------|----------|
| **Leased Line** | T1 (1.544M), T3 (44.7M) | Dedicated point-to-point circuit | Guaranteed bandwidth; expensive |
| **Metro Ethernet** | 10M–100G | Ethernet-based MAN/WAN from carrier | Campus/metro connections |
| **MPLS** | Various | Label-switching; carrier-managed | Enterprise WAN; multiple sites |
| **Broadband DSL** | Up to ~100M | Over phone lines (ADSL asymmetric) | Small offices, remote workers |
| **Cable** | Up to ~1G | Over coaxial cable (DOCSIS) | Home/small office |
| **Fiber (DIA)** | 100M–100G | Dedicated Internet Access | Data centers, large offices |
| **Cellular (4G/5G)** | Variable | Wireless WAN | Mobile/backup connectivity |
| **Satellite** | Variable | High latency (~600ms RTT) | Extremely remote locations |

### MPLS (Multi-Protocol Label Switching)
- Carrier-managed WAN service
- Uses **labels** to forward packets (faster than IP lookup)
- Provides **any-to-any** connectivity between sites
- Supports QoS, traffic engineering
- You don't manage the MPLS network — the carrier does
- **Layer 3 MPLS VPN:** Carrier provides routing between your sites
- **Layer 2 MPLS VPN (VPLS):** Carrier provides Layer 2 connectivity (like a big switch)

---

## VPN Types

A **VPN (Virtual Private Network)** creates a secure, encrypted tunnel over a public network (usually the internet).

### Site-to-Site VPN
```
[HQ Network] ──── [HQ Router] ════ Internet ════ [Branch Router] ──── [Branch Network]
                        └──────── IPsec Tunnel ────────┘
```
- Connects entire networks together
- Always-on (permanent tunnel)
- Transparent to end users
- Uses IPsec for encryption

### Remote Access VPN
```
[Remote User's Laptop] ═══ Internet ═══ [VPN Gateway] ──── [Corporate Network]
         └───────── SSL/IPsec Tunnel ─────────┘
```
- Individual users connect from anywhere
- Client software required (AnyConnect, GlobalProtect)
- On-demand (connect when needed)

### VPN Technologies

| Technology | Layer | Encryption | Use Case |
|-----------|-------|-----------|----------|
| **IPsec** | 3 | Yes (ESP) | Site-to-site; remote access |
| **SSL/TLS VPN** | 4-7 | Yes (TLS) | Clientless browser VPN; remote access |
| **GRE** | 3 | ❌ No | Tunnel multicast/routing protocols |
| **GRE over IPsec** | 3 | Yes | GRE tunnel + encryption |
| **DMVPN** | 3 | Yes (IPsec) | Hub-and-spoke with dynamic spoke-to-spoke |

---

## GRE Tunnels

**GRE (Generic Routing Encapsulation)** wraps one protocol inside another. It creates a virtual point-to-point link over the internet.

**Why GRE?** IPsec alone can't carry multicast or routing protocols (OSPF, EIGRP). GRE can. So you run GRE for the tunnel, then wrap it in IPsec for encryption.

```
Original packet:
[IP Header (10.0.0.1→10.0.0.2)] [Data]

After GRE encapsulation:
[New IP Header (203.0.113.1→198.51.100.1)] [GRE Header] [Original IP Header] [Data]
  "Delivery" header (public IPs)              "Passenger" packet (private IPs)
```

### GRE Configuration
```
! R1 — Headquarters
R1(config)# interface Tunnel0
R1(config-if)# ip address 10.10.10.1 255.255.255.252
! Tunnel interface gets a private IP (the "inside" of the tunnel)

R1(config-if)# tunnel source GigabitEthernet0/1
! Physical interface or public IP

R1(config-if)# tunnel destination 198.51.100.2
! Remote router's public IP

R1(config-if)# no shutdown

! Route remote network through the tunnel
R1(config)# ip route 172.16.0.0 255.255.0.0 10.10.10.2


! R2 — Branch
R2(config)# interface Tunnel0
R2(config-if)# ip address 10.10.10.2 255.255.255.252
R2(config-if)# tunnel source GigabitEthernet0/1
R2(config-if)# tunnel destination 203.0.113.1
R2(config-if)# no shutdown

R2(config)# ip route 192.168.0.0 255.255.0.0 10.10.10.1
```

### Verification
```
show interface Tunnel0
show ip route
ping 10.10.10.2           ! Test tunnel connectivity
```

---

## IPsec Fundamentals

IPsec provides **confidentiality, integrity, and authentication** for IP traffic.

**Two protocols:**
| Protocol | IP Proto | What It Does |
|----------|----------|-------------|
| **AH** (Authentication Header) | 51 | Integrity + authentication only (no encryption) |
| **ESP** (Encapsulating Security Payload) | 50 | Encryption + integrity + authentication ← **Used almost exclusively** |

**Two modes:**
| Mode | What's Encrypted | Use Case |
|------|-----------------|----------|
| **Transport** | Only the payload | Host-to-host (rare) |
| **Tunnel** | Entire original packet (new IP header added) | Site-to-site VPNs ← **Most common** |

**IKE (Internet Key Exchange):** Negotiates the encryption parameters and keys between VPN peers.
- **IKE Phase 1:** Authenticate peers, establish secure channel (ISAKMP SA)
- **IKE Phase 2:** Negotiate encryption for actual data (IPsec SA)

---

## SD-WAN

**Software-Defined WAN** — a modern approach that uses software to manage WAN connections.

```
Traditional WAN:                    SD-WAN:
┌────────────┐                    ┌────────────────────┐
│   MPLS     │ ← expensive        │  SD-WAN Controller │ ← centralized
│  (single   │                    │   (orchestrator)    │
│   path)    │                    └──────┬─────────────┘
└────────────┘                           │
                                  ┌──────┼──────┐
                                  │ MPLS │ Internet│ 4G │
                                  │      │ VPN    │    │
                                  └──────┴────────┴────┘
                                    Multiple paths!
```

**SD-WAN benefits:**
- Uses **multiple WAN links** (MPLS + broadband + 4G) simultaneously
- **Application-aware** routing (video over MPLS, email over broadband)
- **Centralized management** (configure all sites from one controller)
- **Encryption** built-in (IPsec between all sites)
- **Cost savings** (augment expensive MPLS with cheap broadband)

**Cisco SD-WAN components:**
| Component | Role |
|-----------|------|
| vManage | Management dashboard (GUI) |
| vSmart | Central controller (policies, routing) |
| vBond | Orchestrator (authenticates new devices) |
| vEdge/cEdge | WAN edge routers at each site |

---

## Practice Questions

1. What does GRE provide that IPsec alone doesn't?
2. What IPsec protocol provides encryption?
3. What is the difference between site-to-site and remote access VPNs?
4. What is the main advantage of SD-WAN over traditional WAN?
5. What WAN technology uses labels for forwarding?
6. Why would you use GRE over IPsec instead of just IPsec?

<details>
<summary>Answers</summary>

1. GRE can carry multicast traffic and routing protocol updates; IPsec alone cannot
2. ESP (Encapsulating Security Payload)
3. Site-to-site connects entire networks (always-on); remote access connects individual users (on-demand)
4. Uses multiple WAN links simultaneously with application-aware routing and centralized management, reducing cost and improving performance
5. MPLS (Multi-Protocol Label Switching)
6. To carry routing protocol traffic (OSPF, EIGRP) or multicast through the tunnel while still encrypting everything with IPsec
</details>

---

*← [Days 48-49 — NAT](Day48-49-NAT.md) | [Days 52-53 — Network Services](Day52-53-Network-Services.md) →*
