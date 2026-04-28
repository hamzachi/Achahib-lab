<div align="center">

# 🌐 Networking — Network Device Manager

[![SSH](https://img.shields.io/badge/SSH-Paramiko-4CAF50?style=flat-square)](.)
[![Cisco](https://img.shields.io/badge/Cisco-IOS%20%7C%20Switch%20%7C%20AP-1BA0D7?style=flat-square)](.)
[![OPNsense](https://img.shields.io/badge/OPNsense-Firewall-D94F00?style=flat-square)](.)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)](.)
[![Status](https://img.shields.io/badge/Status-Active%20Development-orange?style=flat-square)](.)

**Part of the [HMIND Platform](../README.md)**

</div>

---

## 📌 Overview

The Networking module is HMIND's **live network device inventory and SSH configuration collector**. It maintains a database of all physical network devices in the lab and automatically connects to each one via SSH to pull live configuration data — then makes that data available to the AI chat engine.

> 💡 **Key principle:** Every device in your network is queryable in natural language. Ask HMIND anything about your infrastructure — it already knows the answer from the live data it collected.

---

## 🏗️ Architecture

<div align="center">

![Networking Architecture](https://raw.githubusercontent.com/hamzachi/Achahib-lab/main/hmind%20AI%20Agent/Architecture/networking_architecture.svg)

</div>

> **Legend:** Blue = Cisco devices · Red = OPNsense · Orange = SSH collection engine · Green = parser · Yellow = database · Purple/Teal = outputs

---

## 🖥️ Supported Device Types

| Device Type | What Gets Collected |
|---|---|
| **Cisco Router** | Running config · interfaces · routing table · OSPF · BGP · CDP neighbors · version |
| **Cisco Switch** | VLANs · spanning tree · interface status · trunks · access ports |
| **Cisco AP** | SSIDs · 802.1X config · associated clients · RADIUS stats · BSSIDs |
| **OPNsense** | Interfaces · WireGuard tunnels · OSPF/BGP · OpenVPN · DNS · firewall rules · aliases |

---

## ⚙️ SSH Collection Engine

### Commands per Device Type

```python
COMMANDS = {
    "cisco_ios": [
        "show running-config",
        "show ip interface brief",
        "show ip route",
        "show ip ospf neighbor",
        "show ip bgp summary",
        "show cdp neighbors detail",
        "show version"
    ],
    "cisco_switch": [
        # All cisco_ios commands +
        "show vlan brief",
        "show spanning-tree summary",
        "show interfaces status"
    ],
    "cisco_ap": [
        "show running-config",
        "show dot11 associations all-client",
        "show ip interface brief",
        "show dot11 bssid",
        "show radius local-server statistics"
    ],
    "opnsense": [
        "hostname", "ifconfig", "netstat",
        # + WireGuard, OSPF/BGP, OpenVPN, DNS, NTP,
        # + pfctl rules, firewall aliases (parsed from XML)
    ]
}
```

### Special SSH Handling

> ⚠️ Enterprise and legacy network hardware requires special SSH treatment — standard paramiko defaults don't work reliably.

| Challenge | Solution |
|---|---|
| **Old Cisco hardware** | Legacy key exchange algorithms enabled — modern `rsa-sha2-256/512` disabled for compatibility |
| **Cisco SG250 switch** | Uses `Transport + none-auth` — the switch handles authentication in the shell itself, not at the SSH layer |
| **Cisco IOS auth** | Uses `Transport + auth_password` directly — avoids paramiko 4.x keyboard-interactive probe that burns Cisco's limited auth retry budget |
| **Output pagination** | All sessions set `terminal length 0` + `terminal width 200` — prevents paging and truncation on large configs |

---

## 🔍 What Gets Parsed — networking_parser.py

Raw SSH output is parsed into structured JSON and stored in the database:

```
Raw SSH output
      │
      ▼
networking_parser.py
      │
      ├── Interfaces      → IP, mask, shutdown state, VLAN, trunk/access mode, HSRP
      ├── VLANs           → VLAN ID + name
      ├── ACLs            → Named + numbered, standard + extended
      ├── OSPF            → Process ID, router-id, networks, passive interfaces
      ├── BGP             → AS number, neighbors, advertised networks
      ├── Static routes   → Next-hop, metric, AD
      ├── AAA             → RADIUS/TACACS servers, auth method lists
      ├── WireGuard       → Peers, endpoints, listen port, allowed IPs
      ├── Firewall rules  → OPNsense rules parsed from config.xml
      ├── Aliases         → OPNsense alias groups (IPs, networks, ports)
      └── SSIDs           → SSID names, 802.1X config (Cisco AP)
```

---

## 🤖 How Networking Connects to HChat

The collected device data is **injected into the AI context** whenever a networking-related question is asked.

### Intent Detection

```python
NETWORK_TRIGGERS = [
    'router', 'switch', 'firewall', 'opnsense', 'cisco',
    'vlan', 'ospf', 'bgp', 'interface', 'routing',
    'wireguard', 'vpn', 'tunnel',
    'ap', 'wifi', 'ssid', 'wireless',
    'firewall rule', 'alias', 'pfctl',
    'dns', 'dhcp', 'nat', 'subnet', 'acl', 'route table',
    'network device', 'network topology', 'network status',
    # ... and more
]
```

When any of these keywords appear in a chat message, `_build_network_context()` runs and injects into the LLM system prompt:

- Every device's parsed JSON (interfaces, VLANs, routes, OSPF, WireGuard, ACLs)
- Device status (online / error / collecting) and data age
- Firewall rules and aliases for OPNsense

### Dedicated Network AI Endpoint

`/api/networking/ask` — a mini AI chat built directly into the Networking page. It uses device parsed data as context and has **automatic fallback across 3 LLM models**.

---

## 🗄️ Database Schema

```sql
-- Network device inventory
CREATE TABLE network_devices (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT NOT NULL,
    ip_address      TEXT NOT NULL,
    device_type     TEXT NOT NULL,      -- cisco_ios / cisco_switch / cisco_ap / opnsense
    ssh_port        INTEGER DEFAULT 22,
    ssh_legacy      BOOLEAN DEFAULT FALSE,
    status          TEXT DEFAULT 'unknown', -- unknown / collecting / online / error
    last_collected  DATETIME,
    error_message   TEXT,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Parsed configuration data per device
CREATE TABLE network_device_data (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id   INTEGER REFERENCES network_devices(id),
    data_type   TEXT,       -- interfaces / vlans / routes / ospf / acls / wireguard / ...
    data_json   TEXT,       -- full structured JSON
    collected_at DATETIME
);
```

---

## 🔌 API Endpoints

| Method | Endpoint | Description |
|:---:|---|---|
| `GET` | `/api/networking/devices` | List all devices with status |
| `POST` | `/api/networking/devices` | Add new device |
| `DELETE` | `/api/networking/devices/{id}` | Remove device |
| `POST` | `/api/networking/collect/{id}` | Trigger manual SSH collection |
| `GET` | `/api/networking/devices/{id}/data` | Get parsed device data |
| `POST` | `/api/networking/ask` | AI chat about network devices |

---

## 🚀 Planned Enhancements

| Feature | Description | Priority |
|---|---|:---:|
| **Live topology map** | Visual diagram of devices, links, VLANs — auto-generated from CDP + parsed data | 🔴 High |
| **SNMP support** | Pull real-time interface counters and MIB data without SSH | 🔴 High |
| **Config diff** | Compare device config between two collection runs — detect unauthorized changes | 🟡 Medium |
| **VLAN audit** | Detect rogue VLANs, unauthorized port changes, new devices on trunk ports | 🟡 Medium |
| **Auto-healing** | If interface goes down → AI diagnoses → triggers remediation playbook | 🟡 Medium |
| **NetFlow support** | Pull flow data from routers for bandwidth per-flow visibility | 🟢 Low |

---

<div align="center">

*[← Back to HMIND Platform](../README.md) · [HNTA — Traffic Analyzer →](./hnta.md)*

</div>
