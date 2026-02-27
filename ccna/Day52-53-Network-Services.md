# Days 52-53: Network Services (DHCP, DNS, NTP, SNMP, Syslog)

## 🎯 What You'll Learn
How essential network services work, how to configure them on Cisco devices, and what to expect on the CCNA exam.

---

## DHCP (Dynamic Host Configuration Protocol)

Automatically assigns IP addresses, subnet masks, default gateways, and DNS servers to hosts.

### How DHCP Works — DORA

```
Client                              Server
  │── DISCOVER (broadcast) ──────────→│  "Anyone have an IP for me?"
  │←── OFFER (unicast) ──────────────│  "Here's 192.168.1.10"
  │── REQUEST (broadcast) ───────────→│  "I'll take 192.168.1.10"
  │←── ACK (unicast) ────────────────│  "It's yours for 24 hours"
```

**D**iscover → **O**ffer → **R**equest → **A**ck = **DORA**

> 💡 **Why is REQUEST a broadcast?** To inform ALL DHCP servers on the segment which offer was accepted (so they can release other offered IPs).

### Cisco Router as DHCP Server
```
R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
! Reserve .1-.10 for static assignments (routers, servers, printers)

R1(config)# ip dhcp pool LAN_POOL
R1(dhcp-config)# network 192.168.1.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.1.1
R1(dhcp-config)# dns-server 8.8.8.8 8.8.4.4
R1(dhcp-config)# domain-name example.com
R1(dhcp-config)# lease 7
! Lease time: 7 days (default is 1 day)
```

### DHCP Relay (ip helper-address)

DHCP Discover is a broadcast — it can't cross routers. If the DHCP server is on a different subnet, the router must **relay** the request.

```
[Clients: 192.168.1.0/24] ──── [R1] ──── [R2] ──── [DHCP Server: 10.0.0.5]
                                  ↑
                          ip helper-address here!

R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip helper-address 10.0.0.5
! Converts DHCP broadcasts to unicast and forwards to 10.0.0.5
```

> 💡 `ip helper-address` also forwards other UDP broadcasts by default (TFTP, DNS, TACACS, NetBIOS). Use `no ip forward-protocol udp <port>` to disable specific ones.

### Verification
```
show ip dhcp binding              ! Active leases
show ip dhcp pool                 ! Pool statistics
show ip dhcp conflict             ! Detected address conflicts
show ip dhcp server statistics    ! Server stats
```

---

## DNS (Domain Name System)

Translates domain names (www.cisco.com) to IP addresses (72.163.4.185).

### DNS Hierarchy
```
Root (.)
├── .com (TLD)
│   ├── cisco.com
│   │   ├── www.cisco.com → 72.163.4.185
│   │   └── mail.cisco.com → 72.163.4.186
│   └── google.com
├── .org
└── .net
```

### DNS Record Types
| Type | Purpose | Example |
|------|---------|---------|
| **A** | Name → IPv4 | www.cisco.com → 72.163.4.185 |
| **AAAA** | Name → IPv6 | www.cisco.com → 2001:DB8::1 |
| **CNAME** | Alias | shop.cisco.com → www.cisco.com |
| **MX** | Mail server | cisco.com → mail.cisco.com |
| **PTR** | IP → Name (reverse) | 72.163.4.185 → www.cisco.com |
| **NS** | Authoritative nameserver | cisco.com → ns1.cisco.com |

### DNS on Cisco Devices
```
! Configure DNS for the router itself
R1(config)# ip name-server 8.8.8.8
R1(config)# ip domain-lookup
! Now R1 can resolve hostnames

! Configure the router as a DNS server
R1(config)# ip dns server
R1(config)# ip host SERVER1 192.168.1.10
! Static hostname entry
```

---

## NTP (Network Time Protocol)

Synchronizes clocks across all network devices. **Critical** for log correlation, certificates, authentication, and troubleshooting.

### NTP Hierarchy — Stratum Levels
```
Stratum 0: Atomic clock / GPS (reference clock)
Stratum 1: Directly connected to Stratum 0 (NTP primary server)
Stratum 2: Syncs from Stratum 1
Stratum 3: Syncs from Stratum 2
...
Stratum 15: Maximum (Stratum 16 = unsynchronized)
```

Lower stratum = more accurate. Each hop adds 1 to the stratum level.

### Configuration
```
! Set timezone
R1(config)# clock timezone MST -7

! Configure NTP client
R1(config)# ntp server 10.0.0.1
! Or use a public server:
R1(config)# ntp server pool.ntp.org

! Configure NTP authentication (optional, recommended)
R1(config)# ntp authenticate
R1(config)# ntp authentication-key 1 md5 NTP_Secret
R1(config)# ntp trusted-key 1
R1(config)# ntp server 10.0.0.1 key 1

! Make this router an NTP server for others
R1(config)# ntp master 3
! Stratum 3 — only use if no external NTP source available
```

### Verification
```
show ntp status                  ! Sync status, stratum, reference
show ntp associations            ! NTP peers and their stratum
show clock                       ! Current time
```

---

## Syslog

Centralized logging — devices send log messages to a syslog server for storage, analysis, and alerting.

### Severity Levels — MUST MEMORIZE

| Level | Name | Description | Mnemonic |
|-------|------|-------------|----------|
| 0 | **Emergency** | System unusable | **E**very |
| 1 | **Alert** | Immediate action needed | **A**wesome |
| 2 | **Critical** | Critical conditions | **C**isco |
| 3 | **Error** | Error conditions | **E**ngineer |
| 4 | **Warning** | Warning conditions | **W**ill |
| 5 | **Notification** | Normal but significant | **N**eed |
| 6 | **Informational** | Informational messages | **I**ce |
| 7 | **Debugging** | Debug messages | **C**ream |

> 💡 **Lower number = more severe.** When you set logging at level 4, you get levels 0-4 (Emergency through Warning).

### Configuration
```
! Log to a syslog server
R1(config)# logging host 10.0.0.100
R1(config)# logging trap informational
! Send levels 0-6 to the syslog server

! Log to console
R1(config)# logging console warnings
! Show levels 0-4 on console

! Log to internal buffer
R1(config)# logging buffered 16384 debugging
! Store all levels in 16KB buffer

! Add timestamps to logs (essential!)
R1(config)# service timestamps log datetime msec
```

### Verification
```
show logging                     ! Logging config and buffered messages
```

---

## SNMP (Simple Network Management Protocol)

Allows a **Network Management System (NMS)** to monitor and manage devices remotely.

### Components
```
[NMS / Monitoring Server] ←── SNMP ──→ [Router/Switch with SNMP Agent]
         │                                        │
    Polls devices                          Sends traps/informs
    for stats                              when events occur
```

- **NMS:** Management station (SolarWinds, PRTG, Nagios)
- **Agent:** Software running on the managed device
- **MIB:** Management Information Base — database of device variables (CPU, memory, interface stats)
- **OID:** Object Identifier — unique ID for each MIB variable

### SNMP Versions

| Version | Auth | Encryption | Model |
|---------|------|-----------|-------|
| v1 | Community string (plaintext) | ❌ | Obsolete |
| v2c | Community string (plaintext) | ❌ | Common but insecure |
| **v3** | Username/password | ✅ AES/DES | **Recommended** |

### SNMPv2c Configuration
```
! Read-only community
R1(config)# snmp-server community PUBLIC_RO ro
! NMS can read stats but not change config

! Read-write community (careful!)
R1(config)# snmp-server community PRIVATE_RW rw

! Send traps to NMS
R1(config)# snmp-server host 10.0.0.100 version 2c PUBLIC_RO
R1(config)# snmp-server enable traps
```

### SNMP Operations
| Operation | Direction | Purpose |
|-----------|-----------|---------|
| **Get** | NMS → Agent | Read a specific variable |
| **GetNext** | NMS → Agent | Read the next variable in MIB |
| **GetBulk** | NMS → Agent | Read many variables at once (v2c+) |
| **Set** | NMS → Agent | Change a variable (if RW) |
| **Trap** | Agent → NMS | Unsolicited alert (no acknowledgment) |
| **Inform** | Agent → NMS | Acknowledged alert (v2c+) |

---

## Practice Questions

1. What are the four DHCP steps?
2. What does `ip helper-address` do?
3. What DNS record type maps a name to an IPv6 address?
4. What is NTP Stratum 0?
5. What syslog severity level is "Warning"?
6. What is the difference between an SNMP trap and an inform?
7. Which SNMP version supports encryption?

<details>
<summary>Answers</summary>

1. DORA: Discover, Offer, Request, Acknowledge
2. Relays DHCP broadcasts (and other UDP broadcasts) as unicast to a remote DHCP server
3. AAAA
4. The reference clock source (atomic clock, GPS) — the most accurate time source
5. Level 4
6. Traps are fire-and-forget (no acknowledgment); informs are acknowledged by the NMS
7. SNMPv3
</details>

---

*← [Days 50-51 — WAN & VPNs](Day50-51-WAN-VPNs.md) | [Days 54-55 — Security Fundamentals](Day54-55-Security-Fundamentals.md) →*
