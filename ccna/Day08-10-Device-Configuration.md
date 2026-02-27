# Days 8–10: Initial Switch & Router Configuration

## 🎯 What You'll Learn
How to navigate the Cisco IOS CLI, perform initial device setup, secure management access, configure interfaces, and verify your work.

---

## Accessing a Cisco Device

**Console Access (Day 1 on the job):**
```
Laptop  ──[Rollover/Console Cable]──  Router/Switch Console Port
                                       (RJ-45 or USB mini/micro)

Terminal emulator settings:
  Speed: 9600 baud
  Data bits: 8
  Parity: None
  Stop bits: 1
  Flow control: None
```

Tools: PuTTY (Windows), `screen /dev/ttyUSB0 9600` (Linux/Mac), SecureCRT

**Other access methods:**
| Method | When to Use | Port |
|--------|-------------|------|
| Console | Initial setup, password recovery, out-of-band | Physical |
| SSH | Remote management (encrypted) ✅ | TCP 22 |
| Telnet | Remote management (unencrypted) ❌ | TCP 23 |
| Aux | Modem dial-in (legacy) | Physical |

---

## The IOS CLI Modes

```
                    ┌─────────────────────┐
                    │  User EXEC Mode     │  Switch>
                    │  (Limited commands)  │  Can: show, ping, traceroute
                    └──────┬──────────────┘
                           │ enable
                           ▼
                    ┌─────────────────────┐
                    │  Privileged EXEC    │  Switch#
                    │  (Full access)      │  Can: show, debug, copy, reload, configure
                    └──────┬──────────────┘
                           │ configure terminal
                           ▼
                    ┌─────────────────────┐
                    │  Global Config      │  Switch(config)#
                    │  (Change settings)  │  Can: hostname, enable secret, interface, line
                    └──────┬──────────────┘
                           │ interface / line / router / vlan
                           ▼
                    ┌─────────────────────┐
                    │  Sub-config modes   │  Switch(config-if)#
                    │                     │  Switch(config-line)#
                    │                     │  Switch(config-router)#
                    └─────────────────────┘
```

**Navigation shortcuts:**
| Action | Command |
|--------|---------|
| User → Privileged | `enable` |
| Privileged → User | `disable` |
| Privileged → Global Config | `configure terminal` (or `conf t`) |
| Any config → Privileged | `end` or `Ctrl+Z` |
| Sub-config → Global | `exit` |
| Any mode → go back one level | `exit` |

**Helpful CLI features:**
| Feature | How |
|---------|-----|
| Tab completion | Press `Tab` to complete a command |
| Command help | `?` shows available commands; `sh?` shows commands starting with "sh" |
| Abbreviation | `sh run` = `show running-config` |
| Up arrow | Recall previous command |
| `do` | Run privileged command from config mode: `do show ip int brief` |
| Pipe filters | `show run | include hostname` or `show run | section interface` |

---

## Complete Initial Configuration — Switch

Here's a full Day-1 switch configuration with explanations:

```
! ============================================
! STEP 1: Set the hostname
! ============================================
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW1
! Why: Identifies the device. "Switch" is the default — useless in a real network.

! ============================================
! STEP 2: Secure privileged EXEC mode
! ============================================
SW1(config)# enable secret C1sc0Str0ng!
! Why: Protects privileged mode with an MD5-hashed password.
! NOTE: "enable password" stores in plaintext — NEVER use it. Always use "enable secret."

! ============================================
! STEP 3: Secure console access
! ============================================
SW1(config)# line console 0
SW1(config-line)# password ConS0le!
SW1(config-line)# login
! "login" tells the device to actually ASK for the password.
! Without "login," the password is set but never prompted.

SW1(config-line)# logging synchronous
! Why: Prevents syslog messages from interrupting your typing.
! Without this, log messages break up your commands mid-type. Extremely annoying.

SW1(config-line)# exec-timeout 5 0
! Why: Auto-logout after 5 minutes of inactivity.
! Format: exec-timeout <minutes> <seconds>
! "exec-timeout 0 0" = never timeout (DON'T do this in production — security risk)

SW1(config-line)# exit

! ============================================
! STEP 4: Secure VTY lines (remote access)
! ============================================
SW1(config)# line vty 0 15
! There are 16 VTY lines (0-15), meaning 16 simultaneous remote sessions.
SW1(config-line)# password Vty@ccess1
SW1(config-line)# login
SW1(config-line)# transport input ssh
! Why: Only allow SSH (not Telnet). Telnet sends passwords in plaintext!
SW1(config-line)# exec-timeout 5 0
SW1(config-line)# logging synchronous
SW1(config-line)# exit

! ============================================
! STEP 5: Encrypt all plaintext passwords
! ============================================
SW1(config)# service password-encryption
! Why: Applies weak (Type 7) encryption to ALL plaintext passwords in the config.
! This includes console, VTY, and "enable password" (if used).
! NOTE: Type 7 is easily reversible — it's just obfuscation, not real security.
! "enable secret" uses Type 5 (MD5) or Type 9 (scrypt) — much stronger.

! ============================================
! STEP 6: Set a login banner
! ============================================
SW1(config)# banner motd # WARNING: Authorized Access Only! #
! Why: Legal requirement in many jurisdictions. Warns unauthorized users.
! The # is the delimiter — any character works as long as it's not in the message.
! DON'T say "Welcome" — a lawyer could argue you invited them in.

! ============================================
! STEP 7: Configure management IP (SVI)
! ============================================
SW1(config)# interface vlan 1
SW1(config-if)# ip address 192.168.1.2 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit
! Why: Switches don't need IPs to forward frames, but they need one for
! remote management (SSH), SNMP, syslog, etc.
! VLAN 1 is the default management VLAN. In production, use a dedicated VLAN.

! ============================================
! STEP 8: Set default gateway
! ============================================
SW1(config)# ip default-gateway 192.168.1.1
! Why: Allows the switch to reach networks beyond its local subnet.
! Required for SSH access from a different network.
! NOTE: This command is for Layer 2 switches only. L3 switches use "ip route."

! ============================================
! STEP 9: Configure SSH (best practice)
! ============================================
SW1(config)# ip domain-name example.com
! Required for RSA key generation

SW1(config)# crypto key generate rsa modulus 2048
! Generates the encryption key. 2048 bits is the recommended minimum.
! This enables SSH on the device.

SW1(config)# ip ssh version 2
! Force SSHv2 (v1 has known vulnerabilities)

SW1(config)# username admin privilege 15 secret Admin$ecure1
! Create a local user account with full privileges
! privilege 15 = privileged EXEC access

! Update VTY to use local usernames instead of simple password
SW1(config)# line vty 0 15
SW1(config-line)# login local
! "login local" = authenticate using local username database
! (replaces the simple "login" + "password" method)
SW1(config-line)# exit

! ============================================
! STEP 10: Save the configuration!
! ============================================
SW1# copy running-config startup-config
! or shortcut:
SW1# write memory
! or even shorter:
SW1# wr
```

> 💡 **Critical concept:** There are TWO configuration files:
> - **running-config** — Active in RAM; lost on reboot if not saved
> - **startup-config** — Stored in NVRAM; loaded on boot
> 
> If you configure something and don't save → it's gone after a reboot!

---

## Router Interface Configuration

Routers need IP addresses on their interfaces to route traffic:

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# description === LAN Connection to SW1 ===
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# description === WAN Link to ISP ===
R1(config-if)# ip address 10.0.0.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface Loopback0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
! Loopback interfaces are always up — no "no shutdown" needed
! Used as Router ID for OSPF, management, testing
```

> 💡 **Router interfaces are DOWN by default.** You MUST type `no shutdown` to activate them. Switches ports are UP by default.

---

## Verification Commands — Your Diagnostic Toolkit

### show running-config
Shows the current active configuration:
```
SW1# show running-config
! Full config — everything. Can pipe to filter:
SW1# show run | include interface
SW1# show run | section line vty
SW1# show run | begin interface
```

### show ip interface brief
**The most-used command in networking:**
```
R1# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     192.168.1.1     YES manual up                    up
GigabitEthernet0/1     10.0.0.1        YES manual up                    up
GigabitEthernet0/2     unassigned      YES unset  administratively down down
Loopback0              1.1.1.1         YES manual up                    up
```

**Status/Protocol combinations:**
| Status | Protocol | Meaning |
|--------|----------|---------|
| up | up | Working ✅ |
| up | down | Layer 2 issue (encapsulation, keepalive) |
| down | down | Physical issue (cable, no connection) |
| administratively down | down | `shutdown` command applied |

### show interfaces
Detailed statistics per interface:
```
R1# show interfaces GigabitEthernet0/0
! Shows: MAC address, bandwidth, duplex, speed, MTU, 
!        input/output packets, errors, CRC, collisions
```

### show version
```
R1# show version
! Shows: IOS version, uptime, hardware model, RAM, flash,
!        config register, license info
```

### Other essential commands
```
show mac address-table          ! Switch MAC table
show vlan brief                 ! VLAN assignments
show interfaces status          ! Quick port status (speed, duplex, VLAN)
show ip route                   ! Routing table
show arp                        ! ARP cache
show cdp neighbors              ! Directly connected Cisco devices
show lldp neighbors             ! Vendor-neutral neighbor discovery
```

---

## Configuration Management

### Saving and Restoring

```
! Save running to startup
copy running-config startup-config

! View saved config (what loads on boot)
show startup-config

! Erase startup config (factory reset)
write erase
! or
erase startup-config

! Then reload to reset
reload
```

### Backing Up Configuration

```
! Copy config to TFTP server
copy running-config tftp:
! Prompts for TFTP server IP and filename

! Copy from TFTP to running-config
copy tftp: running-config

! Copy IOS image to/from TFTP
copy flash: tftp:
copy tftp: flash:
```

### Configuration Register

The **config register** controls boot behavior:
```
R1# show version | include register
Configuration register is 0x2102

Common values:
  0x2102 — Normal boot (default)
  0x2142 — Skip startup-config on boot (password recovery!)
  0x2100 — Boot to ROMmon
```

---

## Password Recovery (Know the Process)

If you're locked out of a router:

```
1. Power cycle the router
2. Press Ctrl+Break during boot to enter ROMmon
3. Change config register to skip startup-config:
   rommon> confreg 0x2142
   rommon> reset
4. Router boots with no config (no passwords!)
5. Enter enable mode (no password required)
6. Load the old config: copy startup-config running-config
7. Change the password: enable secret NewPassword
8. Reset config register: config-register 0x2102
9. Save: copy running-config startup-config
10. Reload
```

---

## CDP and LLDP — Neighbor Discovery

**CDP (Cisco Discovery Protocol):**
- Cisco proprietary
- Layer 2 protocol — works even without IP configured
- Shows: hostname, IP, platform, IOS version, interface

```
show cdp neighbors          ! Summary
show cdp neighbors detail   ! Full details including IP addresses
show cdp entry *            ! All entries detailed

! Disable CDP (security — don't leak device info to untrusted links)
SW1(config)# no cdp run                        ! Globally
SW1(config-if)# no cdp enable                  ! Per interface
```

**LLDP (Link Layer Discovery Protocol):**
- IEEE 802.1AB — open standard (vendor-neutral)
- Must be explicitly enabled on Cisco devices

```
SW1(config)# lldp run                          ! Enable globally
SW1(config-if)# lldp transmit                  ! Send on this interface
SW1(config-if)# lldp receive                   ! Receive on this interface

show lldp neighbors
show lldp neighbors detail
```

---

## Practice Lab

**Setup:** Use Packet Tracer, GNS3, or EVE-NG. Create:
- 1 Router (R1) with 2 interfaces
- 2 Switches (SW1, SW2)
- 4 PCs (2 per switch)

**Tasks:**
1. Configure hostnames on all devices
2. Set `enable secret` on all devices
3. Secure console and VTY lines
4. Configure SSH on the router
5. Set banner MOTD on all devices
6. Configure R1 Gi0/0 = 192.168.1.1/24, Gi0/1 = 192.168.2.1/24
7. Configure management IPs on switches (SW1: .2, SW2: .3 in their respective subnets)
8. Set default gateways on switches
9. Verify with `show ip interface brief`, `show running-config`
10. Ping between PCs on the same network
11. Ping between PCs on different networks (through the router)
12. SSH from a PC to the router
13. Save all configurations

---

## Practice Questions

1. What is the difference between `enable password` and `enable secret`?
2. What does `logging synchronous` do?
3. What command saves the running config?
4. A router interface shows "administratively down." What's wrong?
5. What does `login local` do on a VTY line?
6. What config register value skips the startup-config on boot?
7. What's the minimum RSA key size for SSH version 2?
8. What command shows directly connected Cisco devices?

<details>
<summary>Answers</summary>

1. `enable password` stores in plaintext (or weak Type 7). `enable secret` uses MD5 (Type 5) or scrypt (Type 9). If both exist, `enable secret` takes precedence. **Always use `enable secret`.**
2. Prevents syslog messages from interrupting your command typing by reprinting your current command line after the message.
3. `copy running-config startup-config` (or `write memory` or `wr`)
4. The `shutdown` command is applied. Fix with `no shutdown`.
5. Authenticates using the local username database (configured with `username` command) instead of a simple line password.
6. 0x2142
7. 768 bits minimum, but **2048 is recommended**.
8. `show cdp neighbors`
</details>

---

## Key Takeaways

1. **Always use `enable secret`** — never `enable password`
2. **Router interfaces are shutdown by default** — always `no shutdown`
3. **Two configs:** running-config (RAM, active) and startup-config (NVRAM, saved)
4. **SSH > Telnet** — configure SSH, disable Telnet
5. **`show ip interface brief`** — your most-used diagnostic command
6. **Save your config** — `copy running-config startup-config` or you'll lose everything on reboot

---

*← [Day 7 — Ethernet & Switching](Day07-Ethernet-Switching.md) | [Days 11-14 — Labs & Review](Day11-14-Labs-Review.md) →*
