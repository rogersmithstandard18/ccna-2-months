# Day 2: TCP/IP Model & Protocol Suite

## 🎯 What You'll Learn
How the TCP/IP model works, how it maps to the OSI model, and a deep understanding of every key protocol in the suite. This is the model the internet *actually* runs on.

---

## OSI vs TCP/IP — What's the Difference?

The OSI model is a **theoretical** framework (7 layers). The TCP/IP model is the **practical** model the internet uses (4 layers). They describe the same thing differently.

```
    OSI Model              TCP/IP Model
┌──────────────┐      ┌──────────────────┐
│ 7 Application│      │                  │
├──────────────┤      │   Application    │  ← HTTP, DNS, DHCP, FTP, SSH, SMTP
│ 6 Presentatn │      │                  │
├──────────────┤      │                  │
│ 5 Session    │      ├──────────────────┤
├──────────────┤      │   Transport      │  ← TCP, UDP
│ 4 Transport  │      ├──────────────────┤
├──────────────┤      │   Internet       │  ← IP, ICMP, ARP
│ 3 Network    │      ├──────────────────┤
├──────────────┤      │   Network Access │  ← Ethernet, Wi-Fi, PPP
│ 2 Data Link  │      │   (Link)         │
├──────────────┤      │                  │
│ 1 Physical   │      └──────────────────┘
└──────────────┘
```

**Key insight:** OSI Layers 5, 6, 7 collapse into one TCP/IP "Application" layer. OSI Layers 1 and 2 collapse into "Network Access."

> 💡 **Exam tip:** Cisco uses BOTH models. Know how to map between them. When they say "Layer 3," they mean OSI Layer 3 = TCP/IP Internet layer.

---

## The TCP/IP Layers in Detail

### Layer 4 — Application Layer
This is where users interact with the network. Every protocol here has a specific job.

**Protocol Deep Dives:**

#### DNS (Domain Name System) — Port 53
**What it does:** Translates human-readable names (google.com) to IP addresses (142.250.80.46).

**How DNS resolution works:**
```
1. You type "google.com" in your browser
2. Your PC checks its local DNS cache
3. If not cached → asks your configured DNS server (recursive resolver)
4. Resolver checks its cache → if not found, asks Root DNS server
5. Root says "I don't know google.com, but .com is handled by these TLD servers"
6. Resolver asks .com TLD server
7. TLD says "google.com is handled by ns1.google.com"
8. Resolver asks Google's authoritative DNS server
9. Authoritative server responds: "google.com = 142.250.80.46"
10. Resolver caches the answer and sends it to your PC
11. Your PC connects to 142.250.80.46
```

**DNS Record Types:**
| Record | Purpose | Example |
|--------|---------|---------|
| A | Name → IPv4 address | google.com → 142.250.80.46 |
| AAAA | Name → IPv6 address | google.com → 2607:f8b0:4004:800::200e |
| CNAME | Alias to another name | www.google.com → google.com |
| MX | Mail server for domain | google.com → smtp.google.com |
| PTR | IP → Name (reverse DNS) | 142.250.80.46 → google.com |
| NS | Authoritative name server | google.com NS → ns1.google.com |
| SOA | Start of Authority | Zone info, serial number, timers |
| TXT | Text records | SPF, DKIM (email security) |

**DNS uses both UDP and TCP:**
- UDP port 53: Normal queries (fast, small responses)
- TCP port 53: Zone transfers (large data) or responses > 512 bytes

---

#### DHCP (Dynamic Host Configuration Protocol) — Ports 67/68
**What it does:** Automatically assigns IP addresses and network configuration to devices.

**The DORA Process:**
```
   Client                         DHCP Server
     |                                |
     |------ DISCOVER (broadcast) --->|   "Anyone have an IP for me?" (UDP 67)
     |         src: 0.0.0.0           |   dst: 255.255.255.255
     |                                |
     |<------ OFFER ------------------|   "How about 192.168.1.50?" (UDP 68)
     |                                |
     |------ REQUEST (broadcast) ---->|   "Yes, I'll take 192.168.1.50"
     |                                |   (broadcast so other DHCP servers know)
     |                                |
     |<------ ACK --------------------|   "It's yours for 24 hours"
     |                                |
```

**What DHCP provides:**
- IP address
- Subnet mask
- Default gateway
- DNS server(s)
- Lease time
- Domain name (optional)

**Key terms:**
- **Lease:** How long the client can use the address
- **Renewal (T1):** At 50% of lease time, client tries to renew
- **Rebind (T2):** At 87.5% of lease time, client broadcasts for any DHCP server
- **DHCP Relay (ip helper-address):** Forwards DHCP broadcasts across routers to a remote DHCP server

> 💡 **Exam tip:** Remember **DORA** — Discover, Offer, Request, Acknowledge. This is heavily tested.

---

#### HTTP/HTTPS — Ports 80/443
- **HTTP:** Hypertext Transfer Protocol; unencrypted web traffic
- **HTTPS:** HTTP + TLS encryption; the standard for modern web

**HTTP Methods (for automation/programmability section):**
| Method | Action | Analogy |
|--------|--------|---------|
| GET | Retrieve data | "Show me this page" |
| POST | Create new data | "Here's a new entry" |
| PUT | Replace existing data | "Replace this with that" |
| PATCH | Partially update data | "Change just this field" |
| DELETE | Remove data | "Delete this entry" |

---

#### FTP / TFTP — Ports 20-21 / 69
**FTP (File Transfer Protocol):**
- Uses **two connections**: Control (port 21) + Data (port 20)
- Supports authentication (username/password)
- Active vs Passive mode (passive is more firewall-friendly)

**TFTP (Trivial File Transfer Protocol):**
- Uses UDP port 69
- No authentication
- Simple and lightweight
- Used for: Cisco IOS upgrades, config file transfers, PXE boot

> 💡 **Cisco context:** You'll use TFTP to back up and restore router/switch configs and IOS images.

---

#### Email Protocols — Ports 25, 110, 143
| Protocol | Port | Direction | Purpose |
|----------|------|-----------|---------|
| SMTP | 25 (587 w/TLS) | Outbound | Send email |
| POP3 | 110 (995 w/TLS) | Inbound | Download email (removes from server) |
| IMAP | 143 (993 w/TLS) | Inbound | Access email (stays on server) |

---

### Layer 3 — Transport Layer

Already covered TCP and UDP in Day 1. Here's the extra detail:

**TCP Header (20 bytes minimum):**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port          │       Destination Port         │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                    Sequence Number                            │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                 Acknowledgment Number                         │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│Offset│Res│  Flags (URG,ACK,PSH,RST,SYN,FIN)  │  Window Size  │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│           Checksum            │       Urgent Pointer          │
└─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
```

**TCP Flags you need to know:**
| Flag | Meaning | Used In |
|------|---------|---------|
| SYN | Synchronize sequence numbers | Connection setup |
| ACK | Acknowledgment | Almost every packet after handshake |
| FIN | Finish (graceful close) | Connection teardown |
| RST | Reset (abort connection) | Errors, rejected connections |
| PSH | Push (deliver immediately) | Real-time data |
| URG | Urgent data | Rarely used |

**TCP Windowing (Flow Control):**
```
Window Size = how many bytes the receiver can accept before needing an ACK

Sender: "Here's bytes 1-1000"
Sender: "Here's bytes 1001-2000"
Sender: "Here's bytes 2001-3000"  ← Window of 3000
Receiver: "ACK 3001, my window is now 5000"  ← Receiver can handle more
Sender: sends up to 5000 bytes before waiting
```

**Sliding Window:** The window size adjusts dynamically based on network conditions and receiver capacity.

**UDP Header (8 bytes — simple!):**
```
│  Source Port  │  Dest Port  │
│    Length     │  Checksum   │
```

That's it. No sequence numbers, no ACKs, no windowing. Just "here's the data, good luck."

---

### Layer 2 — Internet Layer

**IP (Internet Protocol):**
- Connectionless, best-effort delivery
- No guarantee of delivery, ordering, or duplicate prevention
- That's TCP's job (Layer 4)

**IPv4 Header (20 bytes minimum):**
Key fields you should know:
| Field | Purpose |
|-------|---------|
| Version | IPv4 (4) or IPv6 (6) |
| TTL (Time to Live) | Decremented at each router; prevents infinite loops; packet dropped at 0 |
| Protocol | Identifies Layer 4 protocol (6=TCP, 17=UDP, 1=ICMP) |
| Source IP | Where the packet came from |
| Destination IP | Where the packet is going |
| Header Checksum | Error detection for the header |

**ICMP (Internet Control Message Protocol):**
- Error reporting and diagnostics
- **ping** uses ICMP Echo Request (Type 8) and Echo Reply (Type 0)
- **traceroute** uses ICMP Time Exceeded (Type 11)
- Not used for data transfer

**ARP (Address Resolution Protocol):**
```
"I know the IP address 192.168.1.1, but what's the MAC address?"

1. Host sends ARP Request (broadcast): "Who has 192.168.1.1?"
   Destination MAC: FF:FF:FF:FF:FF:FF (broadcast)
   
2. Device with 192.168.1.1 sends ARP Reply (unicast): "That's me! My MAC is AA:BB:CC:DD:EE:FF"

3. Host stores the mapping in its ARP cache (temporary table)
```

**Verification:** `show arp` (router) or `arp -a` (PC)

> 💡 **Exam tip:** ARP is broadcast-based. This is why VLANs matter — they limit the broadcast domain, reducing ARP traffic.

---

### Layer 1 — Network Access Layer

Combines OSI Layers 1 and 2. Handles:
- Physical transmission (cables, signals)
- Data framing (Ethernet frames)
- MAC addressing
- Media access (CSMA/CD for Ethernet, CSMA/CA for Wi-Fi)

Covered in depth on Days 3 and 7.

---

## Complete Port Number Reference

**Absolute must-memorize for CCNA:**

| Port | TCP/UDP | Protocol | Memory Trick |
|------|---------|----------|-------------|
| 20 | TCP | FTP Data | FTP = **F**ile **T**wenty **P**rotocol (data on 20) |
| 21 | TCP | FTP Control | Control = 20 + 1 |
| 22 | TCP | SSH | **S**ecure = 22 (two 2s) |
| 23 | TCP | Telnet | 23 = "not secure" (one more than SSH) |
| 25 | TCP | SMTP | **S**end **M**ail = 25 |
| 53 | Both | DNS | DNS has **5** and **3** letters in "Domain Name" |
| 67 | UDP | DHCP Server | Server = 67 (bigger number serves) |
| 68 | UDP | DHCP Client | Client = 68 |
| 69 | UDP | TFTP | TFTP is the "fun" version of FTP → 69 |
| 80 | TCP | HTTP | **80** = "H" looks like 8, "O" looks like 0 |
| 110 | TCP | POP3 | POP = sounds like "one-ten" |
| 143 | TCP | IMAP | Just memorize it |
| 161 | UDP | SNMP | Just memorize it |
| 443 | TCP | HTTPS | 443 = "secure 80" |
| 3389 | TCP | RDP | Just memorize it |

---

## How It All Works Together — A Web Request

Let's trace what happens when you type `https://www.cisco.com` and press Enter:

```
1. APPLICATION: Browser uses HTTPS (port 443) to request the page
2. APPLICATION: DNS resolves "www.cisco.com" → IP address (if not cached)
3. TRANSPORT: TCP 3-way handshake with the web server on port 443
4. TRANSPORT: TLS handshake (encryption setup)
5. INTERNET: IP packet created with your IP as source, Cisco's IP as destination
6. INTERNET: Your PC checks — is Cisco's IP on my local network?
   - No → Send to default gateway (router)
   - ARP for the router's MAC address (if not cached)
7. NETWORK ACCESS: Ethernet frame created
   - Source MAC: Your NIC's MAC
   - Destination MAC: Router's MAC (NOT Cisco's MAC!)
8. NETWORK ACCESS: Frame converted to electrical signals on the wire
9. Router receives frame, strips Layer 2, reads Layer 3
10. Router looks up destination IP in routing table
11. Router creates NEW Layer 2 frame with next-hop's MAC address
12. Process repeats at each hop until reaching Cisco's server
13. Server processes request, sends response back the same way
```

> 💡 **The key insight:** IP addresses identify the **final destination**. MAC addresses identify the **next hop**. MAC changes at every router; IP stays the same.

---

## Practice Questions

1. How many layers does the TCP/IP model have? Name them.
2. A DNS query is typically sent over which transport protocol and port?
3. In the DHCP DORA process, which messages are broadcast?
4. What is the difference between POP3 and IMAP?
5. What does ARP resolve, and is it broadcast or unicast?
6. If TTL reaches 0, what happens to the packet?
7. What TCP flag is used to abruptly terminate a connection?
8. You can ping a server by IP but not by hostname. What protocol is likely failing?

<details>
<summary>Answers</summary>

1. 4 layers: Application, Transport, Internet, Network Access
2. UDP port 53 (TCP 53 for zone transfers or large responses)
3. DISCOVER and REQUEST are broadcast; OFFER and ACK can be broadcast or unicast
4. POP3 downloads and removes email from the server; IMAP accesses email while keeping it on the server
5. ARP resolves IP addresses to MAC addresses. ARP Request = broadcast; ARP Reply = unicast
6. The packet is dropped and an ICMP Time Exceeded message is sent back to the source
7. RST (Reset)
8. DNS (port 53)
</details>

---

## Key Takeaways

1. **TCP/IP is the practical model; OSI is the reference model** — know both
2. **DORA** for DHCP, **3-way handshake** for TCP — memorize the sequences
3. **Port numbers** — there's no shortcut, you need to memorize them
4. **ARP bridges Layer 2 and Layer 3** — resolves IP to MAC
5. **DNS is the backbone of the internet** — almost everything starts with a DNS query

---

*← [Day 1 — OSI Model](Day01-OSI-Model.md) | [Day 3 — Network Topologies & Media](Day03-Topologies-Media.md) →*
