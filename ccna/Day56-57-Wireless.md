# Days 56-57: Wireless Networking

## 🎯 What You'll Learn
Wi-Fi standards, wireless architecture (autonomous vs controller-based), CAPWAP, WLC, AP modes, and wireless security.

---

## Wi-Fi Standards (802.11)

| Standard | Band | Max Speed | Year | Name |
|----------|------|-----------|------|------|
| 802.11a | 5 GHz | 54 Mbps | 1999 | — |
| 802.11b | 2.4 GHz | 11 Mbps | 1999 | — |
| 802.11g | 2.4 GHz | 54 Mbps | 2003 | — |
| 802.11n | 2.4 + 5 GHz | 600 Mbps | 2009 | **Wi-Fi 4** |
| 802.11ac | 5 GHz only | 6.9 Gbps | 2013 | **Wi-Fi 5** |
| 802.11ax | 2.4 + 5 + 6 GHz | 9.6 Gbps | 2020 | **Wi-Fi 6/6E** |

> 💡 **Memory trick for bands:**
> - 2.4 GHz = longer range, more interference (microwaves, Bluetooth)
> - 5 GHz = shorter range, less interference, more channels, faster
> - 6 GHz = Wi-Fi 6E only, even more channels

### 2.4 GHz Channels
```
Channels 1-14 available (region-dependent)
Only 3 non-overlapping channels: 1, 6, 11

Channel:  1   2   3   4   5   6   7   8   9  10  11  12  13
          ████████████
                              ████████████
                                                  ████████████
          ↑ Ch 1             ↑ Ch 6               ↑ Ch 11

Always use 1, 6, and 11 to avoid interference between APs.
```

---

## Wireless Architecture

### Autonomous APs (Standalone)
- Each AP is configured individually
- Each AP handles its own encryption, authentication, and management
- **Does not scale** — managing 100+ APs individually is a nightmare
- Suitable for very small networks (1-3 APs)

### Controller-Based (Lightweight APs) — Enterprise Standard
```
   ┌──────────────────────────────────────────┐
   │        WLC (Wireless LAN Controller)     │
   │  Central management, policies, security  │
   └──────┬──────────┬──────────┬─────────────┘
          │ CAPWAP   │ CAPWAP   │ CAPWAP
          │ tunnel   │ tunnel   │ tunnel
       [LAP 1]    [LAP 2]    [LAP 3]
       Floor 1    Floor 2    Floor 3
```

- **WLC** manages all APs centrally (SSIDs, security, RF, firmware)
- **LAP (Lightweight AP)** = "dumb" AP that gets its config from the WLC
- **CAPWAP** tunnel connects each LAP to the WLC

### Cloud-Based (Cisco Meraki)
- Controller is in the cloud (SaaS)
- APs managed via web dashboard
- Zero-touch provisioning
- Requires internet connectivity for management

---

## CAPWAP (Control And Provisioning of Wireless Access Points)

The protocol that connects lightweight APs to the WLC.

| Tunnel | UDP Port | Purpose |
|--------|----------|---------|
| **Control** | 5246 | Management, config, firmware (encrypted with DTLS) |
| **Data** | 5247 | Client traffic (optionally encrypted) |

### Split-MAC Architecture
The work of managing wireless is **split** between the AP and WLC:

| Function | AP | WLC |
|----------|:--:|:---:|
| RF transmission/reception | ✅ | |
| MAC-layer encryption/decryption | ✅ | |
| Beacons and probe responses | ✅ | |
| Authentication | | ✅ |
| Association/roaming | | ✅ |
| Security policies | | ✅ |
| QoS | | ✅ |
| RF management | | ✅ |

### FlexConnect (Formerly H-REAP)
For remote/branch offices: AP can **locally switch** client traffic (doesn't need to tunnel all data back to WLC at HQ). If WLC connection drops, the AP can still operate in standalone mode.

---

## AP Modes

| Mode | Purpose |
|------|---------|
| **Local** | Default. Serves clients + periodically scans channels for rogue APs |
| **FlexConnect** | Remote sites. Local switching + optional central auth |
| **Monitor** | Dedicated sensor. No client service. Only scans for rogues/interference |
| **Rogue Detector** | Listens on wired side to correlate rogue APs found by other APs |
| **Sniffer** | Captures wireless frames and sends to a packet analyzer (Wireshark) |
| **Bridge** | Point-to-point or point-to-multipoint wireless bridge between buildings |
| **SE-Connect** | Spectrum analysis (dedicated spectrum sensor) |

---

## Wireless Security

### Open — No Security ❌
Anyone can connect. No encryption. Use only for guest networks with a captive portal.

### WEP (Wired Equivalent Privacy) ❌
Broken. Crackable in minutes. **Never use WEP.**

### WPA (Wi-Fi Protected Access)
- Uses **TKIP** (improved WEP encryption)
- Better than WEP but still deprecated
- Transitional standard

### WPA2 — Current Standard ✅
- Uses **AES-CCMP** encryption (strong)
- **Personal mode (PSK):** Pre-shared key (password). For home/small office.
- **Enterprise mode (802.1X):** RADIUS authentication. Each user has unique credentials.

### WPA3 — Latest ✅✅
- **SAE (Simultaneous Authentication of Equals)** replaces PSK
  - Resistant to offline dictionary attacks
  - Forward secrecy (compromised password doesn't decrypt past traffic)
- **Enhanced Open (OWE):** Encrypts traffic even on open networks
- **192-bit security** for enterprise mode

| Standard | Encryption | Auth (Personal) | Auth (Enterprise) | Status |
|----------|-----------|-----------------|-------------------|--------|
| WEP | RC4 | Shared key | — | ❌ Broken |
| WPA | TKIP | PSK | 802.1X | ⚠️ Deprecated |
| WPA2 | AES-CCMP | PSK | 802.1X | ✅ Current |
| WPA3 | AES-GCMP | SAE | 802.1X (192-bit) | ✅ Latest |

---

## Wireless Troubleshooting Concepts

| Issue | Possible Cause |
|-------|---------------|
| Slow speeds | Too many clients per AP, interference, wrong channel |
| Can't connect | Wrong SSID/password, RADIUS down (enterprise), disabled band |
| Intermittent drops | Interference (microwave, Bluetooth), AP overloaded |
| Coverage gaps | Not enough APs, walls blocking signal |
| Rogue AP detected | Unauthorized AP plugged into network |

**RF Concepts:**
- **RSSI** (Received Signal Strength Indicator): Signal power at the receiver. > -67 dBm is good.
- **SNR** (Signal-to-Noise Ratio): Higher is better. > 25 dB is good.
- **Co-channel interference:** Same channel, different APs — they share airtime.
- **Adjacent-channel interference:** Overlapping channels — use only 1, 6, 11 on 2.4 GHz.

---

## Practice Questions

1. What are the three non-overlapping 2.4 GHz channels?
2. What protocol connects lightweight APs to a WLC?
3. What encryption does WPA2 use?
4. What is the difference between WPA2 Personal and Enterprise?
5. What AP mode is for dedicated rogue detection?
6. What replaced PSK in WPA3?
7. What is FlexConnect used for?

<details>
<summary>Answers</summary>

1. Channels 1, 6, and 11
2. CAPWAP (UDP ports 5246 for control, 5247 for data)
3. AES-CCMP
4. Personal uses a pre-shared key (one password for everyone); Enterprise uses 802.1X with RADIUS (unique credentials per user)
5. Monitor mode (or Rogue Detector mode for wired-side correlation)
6. SAE (Simultaneous Authentication of Equals) — resistant to offline dictionary attacks
7. Remote/branch office APs that can locally switch traffic and continue operating if the WLC connection is lost
</details>

---

*← [Days 54-55 — Security Fundamentals](Day54-55-Security-Fundamentals.md) | [Days 58-60 — Automation & Final Review](Day58-60-Automation-Review.md) →*
