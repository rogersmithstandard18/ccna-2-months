# Days 58-60: Network Automation, Programmability & Final Review

## 🎯 What You'll Learn
REST APIs, JSON/YAML/XML, configuration management tools (Ansible, Puppet, Chef), SDN, Cisco DNA Center, and a final review checklist for the CCNA exam.

---

## Why Network Automation?

**Traditional (CLI-based):**
- Configure one device at a time via SSH
- Error-prone (typos, inconsistency)
- Slow to scale (100+ devices = weeks of work)
- No version control

**Automated:**
- Configure thousands of devices from one place
- Consistent, repeatable, auditable
- Version-controlled (Git)
- Fast deployment and rollback

---

## Data Formats

Network automation tools exchange data in structured formats. You MUST recognize all three.

### JSON (JavaScript Object Notation) — Most Common
```json
{
  "hostname": "R1",
  "interfaces": [
    {
      "name": "GigabitEthernet0/0",
      "ip": "192.168.1.1",
      "mask": "255.255.255.0",
      "status": "up"
    },
    {
      "name": "GigabitEthernet0/1",
      "ip": "10.0.0.1",
      "mask": "255.255.255.252",
      "status": "up"
    }
  ]
}
```
- Curly braces `{}` = object (key-value pairs)
- Square brackets `[]` = array (list)
- Used by REST APIs

### YAML (YAML Ain't Markup Language) — Human-Friendly
```yaml
hostname: R1
interfaces:
  - name: GigabitEthernet0/0
    ip: 192.168.1.1
    mask: 255.255.255.0
    status: up
  - name: GigabitEthernet0/1
    ip: 10.0.0.1
    mask: 255.255.255.252
    status: up
```
- Indentation-based (like Python)
- Dashes `-` for list items
- Used by Ansible playbooks

### XML (Extensible Markup Language) — NETCONF Uses This
```xml
<device>
  <hostname>R1</hostname>
  <interfaces>
    <interface>
      <name>GigabitEthernet0/0</name>
      <ip>192.168.1.1</ip>
      <mask>255.255.255.0</mask>
      <status>up</status>
    </interface>
  </interfaces>
</device>
```
- Tag-based (like HTML)
- Verbose but very structured
- Used by NETCONF

---

## REST APIs

**REST (Representational State Transfer)** — the standard way to interact with network controllers and services programmatically.

### HTTP Methods (CRUD)

| Method | Action | Description |
|--------|--------|-------------|
| **GET** | Read | Retrieve data |
| **POST** | Create | Create new resource |
| **PUT** | Update/Replace | Replace entire resource |
| **PATCH** | Update/Modify | Modify part of a resource |
| **DELETE** | Delete | Remove a resource |

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | OK (success) |
| 201 | Created (POST success) |
| 400 | Bad Request (client error) |
| 401 | Unauthorized (auth failed) |
| 403 | Forbidden (no permission) |
| 404 | Not Found |
| 500 | Internal Server Error |

### REST API Example
```
GET https://dna-center.example.com/api/v1/network-device

Response (JSON):
{
  "response": [
    {
      "hostname": "R1",
      "managementIpAddress": "10.0.0.1",
      "platformId": "C9300-48T",
      "softwareVersion": "17.6.3"
    }
  ]
}
```

**Key REST characteristics:**
- **Stateless:** Each request is independent (no session memory)
- Uses **URI** (URL) to identify resources
- Data format: usually **JSON**
- Authentication: API keys, tokens, or OAuth

---

## Configuration Management Tools

| Tool | Language | Model | Agent | Created By |
|------|----------|-------|-------|-----------|
| **Ansible** | YAML (playbooks) | Push (agentless) | ❌ No agent | Red Hat |
| **Puppet** | Puppet DSL | Pull (agent-based) | ✅ Agent | Puppet Inc |
| **Chef** | Ruby DSL | Pull (agent-based) | ✅ Agent | Progress |
| **SaltStack** | YAML | Push or Pull | Optional | VMware |

### Ansible — Most Popular for Networking
```yaml
# Example Ansible Playbook
---
- name: Configure VLANs on switches
  hosts: access_switches
  tasks:
    - name: Create VLAN 10
      cisco.ios.ios_vlans:
        config:
          - vlan_id: 10
            name: SALES
        state: merged

    - name: Configure interface
      cisco.ios.ios_l2_interfaces:
        config:
          - name: GigabitEthernet0/1
            access:
              vlan: 10
        state: merged
```

**Why Ansible for networking?**
- **Agentless** — connects via SSH (no software to install on devices)
- **Idempotent** — running the same playbook twice produces the same result
- **Human-readable** YAML playbooks
- Large community and module library

### Puppet
- Uses a **Puppet Master** server
- Agents on managed devices **pull** their config every 30 minutes
- Declarative: you describe the desired state, Puppet enforces it
- Uses its own DSL (Domain-Specific Language)

### Chef
- Uses a **Chef Server**
- **Recipes** (Ruby) define configurations
- **Cookbooks** = collections of recipes
- Agent-based (pull model)

> 💡 **CCNA focus:** Know Ansible is agentless/push/YAML, Puppet is agent-based/pull/DSL, Chef is agent-based/pull/Ruby. Don't need deep config knowledge.

---

## SDN (Software-Defined Networking)

Traditional networking: control plane and data plane are on the **same device** (each router makes its own decisions).

SDN: **separates** the control plane from the data plane. A central controller makes decisions; devices just forward packets.

### The Three Planes

| Plane | Function | Example |
|-------|----------|---------|
| **Management** | Configure and monitor devices | SSH, SNMP, API |
| **Control** | Routing decisions, building tables | OSPF, STP, ARP |
| **Data (Forwarding)** | Actually moves packets | Switching, routing, NAT |

### SDN Architecture
```
┌─────────────────────────────────────┐
│          Application Layer          │  Business apps, analytics
│         (Northbound API: REST)      │
├─────────────────────────────────────┤
│          Controller Layer           │  Cisco DNA Center, OpenDaylight
│     (Southbound API: NETCONF,       │
│      OpenFlow, RESTCONF)            │
├─────────────────────────────────────┤
│       Infrastructure Layer          │  Routers, switches, APs
│         (Data/Forwarding)           │
└─────────────────────────────────────┘
```

**Northbound API:** Controller ↔ Applications (REST API)
**Southbound API:** Controller ↔ Network Devices (OpenFlow, NETCONF, RESTCONF)

---

## Cisco DNA Center

Cisco's **intent-based networking** controller. Manages the entire enterprise network from one dashboard.

**Key Features:**
- **Design:** Network hierarchy (sites, buildings, floors)
- **Policy:** Application policies, access control, QoS
- **Provision:** Automated device onboarding and configuration (Plug and Play)
- **Assurance:** AI-driven analytics, issue detection, client health monitoring

**APIs:**
- REST API for automation
- Northbound: applications interact with DNA Center
- DNA Center uses NETCONF/RESTCONF/CLI southbound to configure devices

---

## NETCONF and RESTCONF

| Protocol | Transport | Data Format | Port | Operations |
|----------|-----------|-------------|------|-----------|
| **NETCONF** | SSH | XML | 830 | get, get-config, edit-config, lock, unlock |
| **RESTCONF** | HTTPS | JSON or XML | 443 | GET, POST, PUT, PATCH, DELETE |

**NETCONF** = programmatic replacement for CLI. Full configuration management with rollback capability.

**RESTCONF** = REST API for YANG models. Easier to use than NETCONF for simple operations.

**YANG** = the data modeling language that defines what can be configured. Think of YANG as the "schema" and NETCONF/RESTCONF as the "transport."

---

## Final CCNA Review Checklist

### Networking Fundamentals (Days 1-7)
- [ ] OSI and TCP/IP models — layers, protocols, encapsulation
- [ ] IPv4 addressing and subnetting (including VLSM)
- [ ] Ethernet, MAC addresses, switching concepts

### Network Access (Days 8-28)
- [ ] VLANs, trunking (802.1Q), DTP
- [ ] STP (root bridge election, port roles/states, PortFast, BPDU Guard)
- [ ] EtherChannel (LACP, PAgP, static)
- [ ] Switch and router basic configuration

### IP Connectivity (Days 29-37)
- [ ] Static routing (default, floating static, summary)
- [ ] OSPF (single-area and multi-area, DR/BDR, LSA types, cost)
- [ ] EIGRP (concepts, successor/feasible successor, comparison to OSPF)
- [ ] FHRP (HSRP, VRRP, GLBP)

### IP Services (Days 43-53)
- [ ] IPv6 (address types, SLAAC, NDP, EUI-64)
- [ ] ACLs (standard, extended, named, placement)
- [ ] NAT/PAT (static, dynamic, overload)
- [ ] DHCP, DNS, NTP, SNMP, Syslog

### Security Fundamentals (Days 54-55)
- [ ] Port security, DHCP snooping, DAI
- [ ] AAA, RADIUS vs TACACS+, 802.1X
- [ ] Common threats and mitigations

### Wireless (Days 56-57)
- [ ] 802.11 standards, channels, bands
- [ ] WLC, CAPWAP, AP modes
- [ ] WPA2/WPA3, PSK vs Enterprise

### Automation & Programmability (Days 58-60)
- [ ] JSON, YAML, XML recognition
- [ ] REST API methods and status codes
- [ ] Ansible vs Puppet vs Chef
- [ ] SDN concepts, controller-based networking
- [ ] Cisco DNA Center, NETCONF, RESTCONF

---

## Exam Day Tips

1. **Time management:** ~100 questions in 120 minutes = ~72 seconds per question
2. **Read the whole question** — look for "best" vs "correct" answer
3. **Eliminate wrong answers** — usually 2 are clearly wrong
4. **Subnetting speed:** Practice until you can subnet in under 30 seconds
5. **Know your show commands** — they test output interpretation heavily
6. **Lab simulations:** Practice configuring OSPF, VLANs, ACLs, NAT, and port security in Packet Tracer or GNS3
7. **Don't overthink it** — CCNA tests fundamental understanding, not edge cases

---

## Practice Questions

1. What data format does Ansible use for playbooks?
2. Is Ansible agent-based or agentless?
3. What is the northbound API?
4. What HTTP method creates a new resource?
5. What does YANG define?
6. What is the difference between NETCONF and RESTCONF?
7. What are the three planes of network operation?

<details>
<summary>Answers</summary>

1. YAML
2. Agentless (connects via SSH, push model)
3. The API between the SDN controller and applications (usually REST API)
4. POST
5. YANG is a data modeling language that defines the structure of configuration and state data for network devices
6. NETCONF uses SSH+XML and is a full configuration management protocol; RESTCONF uses HTTPS+JSON/XML and is a simpler REST-based interface to the same YANG data models
7. Management plane, Control plane, Data (Forwarding) plane
</details>

---

*← [Days 56-57 — Wireless Networking](Day56-57-Wireless.md) | [Back to Week 7-8 Overview](Week7-8.md) →*

---

🎓 **Congratulations!** You've completed the 60-day CCNA study guide. Now practice labs, take practice exams, and review weak areas. You've got this!
