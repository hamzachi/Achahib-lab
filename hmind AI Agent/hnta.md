<div align="center">

# 📡 HNTA — Network Traffic Analyzer

[![OPNsense](https://img.shields.io/badge/OPNsense-Live%20Traffic-D94F00?style=flat-square)](.)
[![Cisco](https://img.shields.io/badge/Cisco-LAN%20Analysis-1BA0D7?style=flat-square)](.)
[![AI](https://img.shields.io/badge/AI-Threat%20Detection-F55036?style=flat-square)](.)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)](.)
[![Status](https://img.shields.io/badge/Status-Active%20Development-orange?style=flat-square)](.)

**Part of the [HMIND Platform](../README.md)**

</div>

---

## 📌 Overview

HNTA (**HMIND Network Traffic Analyzer**) provides **real-time, live traffic visibility** into the network — what connections are active right now, who is on the LAN, what the firewall is blocking, and automatic AI threat detection.

Two independent data sources feed two separate panels:

| Panel | Source | What It Shows |
|---|---|---|
| **Panel 1** | OPNsense Firewall | Live connections · Interface stats · Blocked traffic · Threats |
| **Panel 2** | Cisco Router | LAN devices · ARP table · NAT sessions · Interface rates |

> 💡 **Key principle:** You should never need to open the firewall CLI or router console to understand what is happening on your network. HNTA brings everything into one view with AI explaining what it means.

---

## 🏗️ Architecture

<div align="center">

![HNTA Architecture](https://raw.githubusercontent.com/hamzachi/Achahib-lab/main/hmind%20AI%20Agent/Architecture/hnta_architecture.svg)

</div>

> **Legend:** Red = OPNsense source · Blue = Cisco source · Orange = SSH collectors · Green = parser + threat detection · Yellow = cache · Purple/Teal = outputs

---

## 🔥 Panel 1 — OPNsense Live Firewall Traffic

### How Data Is Collected

SSH connects to OPNsense and runs **4 commands simultaneously**:

```
1. Interface byte/packet counters
   → LAN, WAN, WireGuard interface stats
   → Parses the Link-layer row (actual bytes transferred)

2. Active firewall state table (pfctl)
   → All forwarded TCP/UDP connections
   → Deduplicates bidirectional entries (each connection appears twice)
   → Extracts real source IPs from NAT translations

3. Local process sockets (sockstat)
   → Firewall's own services (SSH, web UI, DNS)
   → Supplements state table with non-forwarded services

4. Firewall log (filter.log / clog / system.log)
   → Parses CSV log format: rule, interface, direction, proto, src, dst, ports
   → Recent blocks and rejected packets
```

### What Gets Returned

```python
{
  "interfaces": {
      "lan":  { "ibytes_h": "1.2 GB", "obytes_h": "800 MB", "ipkts": 892341 },
      "wan":  { "ibytes_h": "900 MB", "obytes_h": "1.1 GB", "ipkts": 741209 },
      "vpn":  { "ibytes_h": "120 MB", "obytes_h":  "95 MB", "ipkts":  83421 }
  },
  "connections": [
      { "src_ip": "...", "dst_ip": "...", "proto": "TCP",
        "src_port": ..., "dst_port": ..., "direction": "LAN→WAN" }
      # up to 150 active connections
  ],
  "conn_stats": {
      "total": 87, "tcp": 61, "udp": 26,
      "lan_to_wan": 54, "wan_to_lan": 8, "lan_to_vpn": 25
  },
  "top_destinations": [...],   # top 10 most-connected external IPs
  "top_sources":      [...],   # top 10 most-connecting internal IPs
  "recent_blocks":    [...],   # last 30 blocked connections
  "suspicious":       [...]    # auto-detected threats
}
```

### 🚨 Automatic Threat Detection

HNTA automatically analyzes live traffic for attack patterns — no manual rules needed.

| Detection | Logic | Severity |
|---|---|:---:|
| **PORT_SCAN** | Single source → more than 15 different destination ports | 🔴 HIGH |
| **HIGH_CONN_RATE** | External IP with more than 20 inbound connections | 🟠 MEDIUM |
| **SUSPICIOUS_PORT** | Connections to known malicious/backdoor ports | 🟠 MEDIUM |
| **BLOCKED_FLOOD** | Same IP blocked more than 5 times in recent log | 🟡 LOW |

```python
def _detect_suspicious(connections, fw_log):
    threats = []

    # PORT_SCAN detection
    src_ports = defaultdict(set)
    for conn in connections:
        src_ports[conn["src_ip"]].add(conn["dst_port"])

    for src_ip, ports in src_ports.items():
        if len(ports) > 15:
            threats.append({
                "type": "PORT_SCAN",
                "src_ip": src_ip,
                "port_count": len(ports),
                "severity": "HIGH"
            })

    # BLOCKED_FLOOD detection
    block_counts = Counter(
        entry["src_ip"] for entry in fw_log
        if entry.get("action") == "block"
    )
    for ip, count in block_counts.items():
        if count > 5:
            threats.append({
                "type": "BLOCKED_FLOOD",
                "src_ip": ip,
                "block_count": count,
                "severity": "LOW"
            })

    return threats
```

---

## 🔵 Panel 2 — Cisco Router LAN Traffic

> **"H-prt"** = HMIND Ports — live LAN device activity and NAT translation visibility.

### Commands Executed

```python
CISCO_COMMANDS = [
    "show interfaces",          # 5-min RX/TX rates, errors, MTU, utilization %
    "show ip arp",              # Full ARP table — every device on LAN right now
    "show ip route",            # Full routing table with protocol labels
    "show ip interface brief",  # Quick status of all interfaces
    "show cdp neighbors detail",# Adjacent Cisco devices (switches, APs)
    "show ip nat translations", # Active NAT sessions — who is talking to what
]
```

### What Gets Returned

```python
{
  "interfaces": [
      { "name": "GigabitEthernet0/0", "status": "up",
        "rx_rate": "12.4 Mbps", "tx_rate": "8.1 Mbps",
        "rx_util": "1.2%", "tx_util": "0.8%", "errors": 0 }
  ],
  "arp": [
      { "ip": "...", "mac": "...", "age": 0, "interface": "Vlan10" }
      # age=0 → active in last minute · age=high → idle/sleeping
  ],
  "routes": [
      { "network": "...", "type": "OSPF", "next_hop": "...", "metric": 110 }
  ],
  "cdp": [
      { "device": "SW1", "platform": "cisco WS-C2960",
        "local_int": "Gi0/1", "remote_int": "Gi0/24" }
  ],
  "nat_sessions": [
      { "inside_local": "...", "outside_global": "...", "proto": "TCP" }
  ],
  "summary": {
      "up_interfaces": 6,
      "arp_hosts": 23,
      "nat_sessions": 41,
      "total_rx": "124 MB",
      "total_tx": "89 MB"
  }
}
```

### ARP Age Interpretation

| ARP Age | Meaning |
|:---:|---|
| `0` | Device communicated in the last minute — **actively online** |
| `1–10` | Device recently active — likely online |
| `High` | Device idle or sleeping |
| `—` | Router's own interface (self) |

---

## 🤖 How HNTA Connects to HChat

HChat has **two separate trigger sets** for HNTA — one for firewall traffic, one for LAN data.

### OPNsense Traffic Triggers

```python
HNTA_TRIGGERS = [
    'traffic', 'connection', 'connected', 'packet', 'flow', 'bandwidth usage',
    'blocked', 'block', 'blocked traffic', 'firewall block', 'firewall log',
    'suspicious', 'scan', 'port scan', 'brute force', 'flood',
    'lan traffic', 'wan traffic', 'vpn traffic',
    'top connection', 'top destination', 'top source',
    'intrusion', 'attack traffic', 'anomaly', 'malicious traffic',
    'who is connected', 'what is connected', 'live traffic', 'real-time traffic',
]
```

### Cisco LAN Triggers

```python
CISCO_LAN_TRIGGERS = [
    'cisco', 'cisco router', 'cisco lan', 'lan device', 'lan host',
    'vlan', 'arp', 'arp table', 'who is on lan', 'what device',
    'nat translation', 'cisco nat', 'inside local',
    'lan bandwidth', 'lan rate', 'cisco bandwidth',
    'cisco analyze', 'cisco status', 'cisco summary',
]
```

When triggered, HChat injects the **full live snapshot** into the LLM:

- **OPNsense:** all interfaces · all connections (up to 150) · top destinations · recent blocks · detected threats
- **Cisco:** all interface rates · full ARP table · routing table · CDP neighbors · NAT sessions

---

## 🔌 Built-in AI Endpoints

HNTA has its own dedicated AI endpoints — no need to go through HChat:

| Method | Endpoint | Description |
|:---:|---|---|
| `POST` | `/api/hnta/analyze` | Full AI security report of OPNsense traffic |
| `POST` | `/api/hnta/explain-connection` | Streaming AI explanation of a clicked connection |
| `POST` | `/api/hnta/chat` | Streaming AI chat about OPNsense traffic |
| `POST` | `/api/hnta/cisco-analyze` | Full AI analysis of Cisco LAN data |
| `POST` | `/api/hnta/cisco-chat` | Streaming AI chat about Cisco LAN |
| `POST` | `/api/hnta/explain-host` | AI analysis of a clicked ARP host — MAC OUI identification, VLAN context, risk assessment |

---

## 🔄 Caching Strategy

| Source | Poll Interval | Cache TTL | Why |
|:---:|:---:|:---:|---|
| OPNsense | 15 seconds | 12 seconds | High-frequency — firewall state changes quickly |
| Cisco Router | 30 seconds | 30 seconds | Lower frequency — ARP/NAT tables change slower |

> **Why cache?** SSH connection setup takes 1–2 seconds per device. Caching prevents the UI from waiting on SSH for every page refresh while still showing near-real-time data.

---

## 🚀 Planned Enhancements

| Feature | Description | Priority |
|---|---|:---:|
| **Auto-blocking** | Detected PORT_SCAN or HIGH_CONN_RATE → HMIND adds block rule to firewall automatically | 🔴 High |
| **Historical traffic graphs** | Store interface byte counters over time → bandwidth graphs per interface | 🔴 High |
| **GeoIP enrichment** | Identify country of every external connection on the dashboard | 🟡 Medium |
| **Alert integration** | Detected threats → automatically create Wazuh-style alert in HSOC | 🟡 Medium |
| **NetFlow support** | Replace SSH polling with NetFlow/sFlow for more accurate traffic data | 🟡 Medium |
| **Packet capture trigger** | Suspicious connection detected → auto-trigger tcpdump on OPNsense | 🟢 Low |

---

<div align="center">

*[← Networking Module](./networking.md) · [Back to HMIND Platform](../README.md)*

</div>
