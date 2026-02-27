# Days 48-49: NAT (Network Address Translation)

## 🎯 What You'll Learn
Static NAT, Dynamic NAT, PAT (overload), NAT terminology, configuration, and troubleshooting.

---

## Why NAT?

Private IP addresses (10.x, 172.16-31.x, 192.168.x) can't be routed on the internet. NAT translates private addresses to public addresses so internal hosts can reach the internet.

```
Inside your network:              On the internet:
192.168.1.10 ─┐                   
192.168.1.20 ─┤── [Router+NAT] ── 203.0.113.5 ──→ Internet
192.168.1.30 ─┘   
              Private IPs          Public IP
```

---

## NAT Terminology — MUST KNOW

| Term | Meaning | Example |
|------|---------|---------|
| **Inside Local** | Private IP of an internal host | 192.168.1.10 |
| **Inside Global** | Public IP representing that host on the internet | 203.0.113.5 |
| **Outside Local** | How an external host appears from inside (usually same as Outside Global) | 8.8.8.8 |
| **Outside Global** | Real public IP of the external host | 8.8.8.8 |

**Memory trick:** Think of it as a matrix:

```
           Local (real IP      Global (translated IP
            on that side)       as seen by other side)
Inside     192.168.1.10         203.0.113.5
Outside    8.8.8.8              8.8.8.8 (usually same)
```

> 💡 **Exam tip:** Inside Local / Inside Global are the most asked. "Inside Local = the private IP your host actually has. Inside Global = the public IP it appears as on the internet."

---

## NAT Types

### Static NAT — One-to-One Mapping

One private IP permanently maps to one public IP. Used for servers that need a consistent public address.

```
192.168.1.10  ←→  203.0.113.10  (always)
```

**Configuration:**
```
R1(config)# ip nat inside source static 192.168.1.10 203.0.113.10

! Mark interfaces
R1(config)# interface Gi0/0
R1(config-if)# ip nat inside

R1(config)# interface Gi0/1
R1(config-if)# ip nat outside
```

**Use case:** Web server, email server, any device that needs to be reachable from the internet at a fixed public IP.

---

### Dynamic NAT — Pool-Based

A pool of public IPs is shared. Internal hosts get a public IP from the pool when they need one. When they're done, it goes back to the pool.

```
Pool: 203.0.113.10 - 203.0.113.20 (11 public IPs)
192.168.1.10 → 203.0.113.10 (while active)
192.168.1.20 → 203.0.113.11 (while active)
192.168.1.30 → 203.0.113.12 (while active)
...
12th host tries → NO FREE IP → connection fails!
```

**Configuration:**
```
! Define the pool
R1(config)# ip nat pool MYPOOL 203.0.113.10 203.0.113.20 netmask 255.255.255.0

! Define which inside hosts can use NAT (ACL)
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255

! Map ACL to pool
R1(config)# ip nat inside source list 1 pool MYPOOL

! Mark interfaces
R1(config)# interface Gi0/0
R1(config-if)# ip nat inside
R1(config)# interface Gi0/1
R1(config-if)# ip nat outside
```

**Limitation:** If all pool addresses are in use, new connections are dropped. That's why PAT is much more common.

---

### PAT (Port Address Translation / NAT Overload) — Most Common

**Many private IPs share ONE public IP.** The router tracks connections using unique port numbers.

```
192.168.1.10:50001 → 203.0.113.5:50001
192.168.1.20:50002 → 203.0.113.5:50002
192.168.1.30:50003 → 203.0.113.5:50003
          ↑                      ↑
     Different                Same public IP!
     source ports             Different ports distinguish them
```

**Configuration (using interface IP — most common):**
```
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
R1(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
!                                                                    ↑
!                                                              "overload" = PAT!

R1(config)# interface Gi0/0
R1(config-if)# ip nat inside
R1(config)# interface Gi0/1
R1(config-if)# ip nat outside
```

**Using a pool with overload:**
```
R1(config)# ip nat inside source list 1 pool MYPOOL overload
! Even a pool of 1 IP works — overload handles thousands of connections
```

> 💡 **This is what your home router does.** Every device in your house shares one public IP via PAT.

---

## NAT Comparison

| Feature | Static | Dynamic | PAT |
|---------|--------|---------|-----|
| Mapping | 1:1 permanent | 1:1 temporary | Many:1 |
| Public IPs needed | One per host | Pool (one per active host) | As few as ONE |
| Inbound connections | ✅ Yes (fixed mapping) | ❌ Only while mapping exists | ❌ No (need port forwarding) |
| Use case | Servers | Limited scenario | **Everyone else** |
| Keyword | `static` | pool name | `overload` |

---

## Verification

```
R1# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
tcp 203.0.113.5:50001  192.168.1.10:50001 8.8.8.8:443        8.8.8.8:443
tcp 203.0.113.5:50002  192.168.1.20:60123 142.250.80.46:80   142.250.80.46:80
--- 203.0.113.10       192.168.1.10       ---                ---

R1# show ip nat statistics
Total active translations: 15 (1 static, 14 dynamic; 14 extended)
Outside interfaces: GigabitEthernet0/1
Inside interfaces: GigabitEthernet0/0
Hits: 1523  Misses: 12

! Clear translations (useful when troubleshooting)
R1# clear ip nat translation *

! Debug (watch NAT in real-time — use carefully!)
R1# debug ip nat
```

---

## NAT Troubleshooting

| Problem | Check |
|---------|-------|
| No translations appearing | ACL matching the right source IPs? Interfaces marked inside/outside? |
| "Pool exhausted" | Dynamic NAT pool too small; switch to PAT (overload) |
| One direction works, other doesn't | Make sure both inside and outside interfaces are configured |
| Can't reach internal server from outside | Need static NAT for inbound connections |
| Translations stuck | Clear with `clear ip nat translation *` |

---

## Practice Questions

1. What does PAT use to distinguish multiple internal hosts sharing one public IP?
2. What keyword enables PAT?
3. What is the Inside Local address?
4. What NAT type is needed for a public-facing web server?
5. What command shows active NAT translations?
6. What must you configure on each interface for NAT to work?

<details>
<summary>Answers</summary>

1. Port numbers — each connection gets a unique source port
2. `overload`
3. The private IP address of the internal host (e.g., 192.168.1.10)
4. Static NAT (one-to-one permanent mapping)
5. `show ip nat translations`
6. `ip nat inside` or `ip nat outside` (depending on the interface's role)
</details>

---

*← [Days 45-47 — ACLs](Day45-47-ACLs.md) | [Days 50-51 — WAN & VPNs](Day50-51-WAN-VPNs.md) →*
