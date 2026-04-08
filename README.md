<div align="center">

# 🏗️ Achahib Lab

**A production-grade home lab simulating enterprise-level networking, security, virtualization, and cloud infrastructure.**

[![Firewall](https://img.shields.io/badge/Firewall-OPNsense%20%2B%20Zenarmor-blue?style=flat-square)](.)
[![SIEM](https://img.shields.io/badge/SIEM-Wazuh-red?style=flat-square)](.)
[![VPN](https://img.shields.io/badge/VPN-WireGuard%20%2B%20OpenVPN-green?style=flat-square)](.)
[![Virtualization](https://img.shields.io/badge/Virtualization-ESXi%20%2B%20Hyper--V-purple?style=flat-square)](.)
[![Cloud](https://img.shields.io/badge/Cloud-Azure%20%2B%20Cloudflare-orange?style=flat-square)](.)
[![NAC](https://img.shields.io/badge/NAC-Cisco%20ISE%20802.1X-lightgrey?style=flat-square)](.)

</div>

---

## 👤 About

Hi, I'm **Hamza** — an IT Engineer based in Morocco with 3+ years of experience in networking, system administration, and cybersecurity. This repository is the central documentation hub for **Achahib Lab** — a fully self-hosted, enterprise-simulated environment covering network security, identity management, SIEM, vulnerability management, hybrid cloud, and infrastructure automation.

> *"Built to learn by doing — every component here mirrors a real enterprise deployment scenario."*

---

## 🗺️ Lab Architecture

```
                        ┌────────────────────────────────────┐
                        │          INTERNET / ISP            │
                        └──────────────┬─────────────────────┘
                                       │
                        ┌──────────────▼─────────────────────┐
                        │         Cisco Router               │
                        │  Enterprise routing, distribution  │
                        │  layer, inter-VLAN routing         │
                        └──────────────┬─────────────────────┘
                                       │
                        ┌──────────────▼─────────────────────┐
                        │    OPNsense + Zenarmor             │
                        │  NAT · IDS/IPS · SNI Inspection    │
                        │  Traffic Filtering · DoH Gateway   │
                        └──┬────────────────────┬────────────┘
                           │                    │
              ┌────────────▼──────┐   ┌─────────▼──────────────┐
              │   Cisco Switch    │   │   Cloudflare           │
              │ VLANs · Trunking  │   │ Zero Trust WARP        │
              │ 802.1X via ISE    │   │ Gateway DNS DoH        │
              └────────┬──────────┘   └────────────────────────┘
                       │
       ┌───────────────┼────────────────────┐
       │               │                    │
┌──────▼──────┐  ┌─────▼──────────┐  ┌──────▼────────┐
│  VLAN 10   │  │   VLAN 20      │  │   VLAN 30     │
│ Management │  │   Servers      │  │  Security     │
└──────┬──────┘  └─────┬──────────┘  └──────┬────────┘
       │               │                    │
┌──────▼──────┐  ┌─────▼──────────┐  ┌──────▼────────┐
│ Cisco ISE  │  │ VMware ESXi   │  │   Wazuh       │
│ AAA · NAC  │  │  + Hyper-V    │  │  SIEM · EDR   │
│ 802.1X     │  │               │  │               │
└────────────┘  └─────┬──────────┘  └──────┬────────┘
                      │                    │
        ┌─────────────┼──────────┐         │
        │             │          │         │
   ┌────▼────┐  ┌─────▼──┐  ┌───▼───┐ ┌───▼──────┐
   │Win Srv  │  │Nextcld │  │Zabbix │ │ OpenVAS  │
   │ADDC·DNS │  │Private │  │Monitor│ │ Vuln Scan│
   │DHCP·GPO │  │Cloud   │  │Alerts │ │          │
   └────┬────┘  └────────┘  └───────┘ └──────────┘
        │
   ┌────▼─────────────────────────────────────────┐
   │              Azure Cloud                     │
   │   Hybrid AD · Azure AD Connect               │
   │   Cloud Services Integration                 │
   └──────────────────────────────────────────────┘

              VPN Layer (Cross-VLAN)
   ┌──────────────────────────────────────────────┐
   │  WireGuard ──► OpenVPN (Double Tunnel)       │
   │  OpenVPN Server: remote access + inspection  │
   └──────────────────────────────────────────────┘
```

---

## 📂 Repository Structure

```
achahib-lab/
├── 📁 network/
│   ├── opnsense/             # OPNsense + Zenarmor configs (sanitized)
│   ├── cisco-router/         # Cisco router configs (routing, ACLs)
│   ├── cisco-switch/         # VLAN, trunking, 802.1X port configs
│   ├── cisco-ise/            # ISE policies, NAC, 802.1X profiles
│   └── diagrams/             # Network topology diagrams
├── 📁 vpn/
│   ├── wireguard/            # WireGuard server/client configs
│   ├── openvpn/              # OpenVPN server setup & auth
│   └── double-tunnel/        # WireGuard → OpenVPN architecture
├── 📁 security/
│   ├── cloudflare-zt/        # Cloudflare Zero Trust WARP + Gateway
│   ├── wazuh/                # SIEM rules, agents, dashboards
│   ├── openvas/              # Scan configs, reports, remediation
│   └── zenarmor/             # IDS/IPS policies, SNI rules
├── 📁 virtualization/
│   ├── esxi/                 # VMware ESXi setup, VM inventory
│   └── hyper-v/              # Hyper-V configs, VM templates
├── 📁 servers/
│   ├── windows-server/       # AD, DNS, DHCP, GPO configurations
│   ├── azure/                # Hybrid AD, Azure AD Connect setup
│   ├── nextcloud/            # Nextcloud deployment, SSL, automation
│   └── zabbix/               # Monitoring templates, alert rules
├── 📁 automation/
│   ├── rudder/               # Rudder policies, Linux automation
│   └── scripts/              # Bash/Python utility scripts
├── 📁 backup/
│   └── urbackup/             # UrBackup server config, schedules
└── 📁 docs/
    ├── reports/              # PDF lab reports
    └── write-ups/            # Project write-ups & lessons learned
```

---

## 🧩 Lab Components

### 🔐 Network Security

| Component | Role | Key Features |
|---|---|---|
| **OPNsense** | Primary Firewall | NAT, traffic filtering, DoH gateway, VLAN routing |
| **Zenarmor** | Next-Gen Firewall Plugin | IDS/IPS, SNI inspection, app-layer control, TLS inspection |
| **Cisco Router** | Edge / Distribution | Enterprise routing, VLAN ACLs, inter-VLAN traffic control |
| **Cisco Switch** | Layer 2 Core | VLAN trunking, port segmentation, 802.1X enforcement |
| **Cisco ISE** | NAC / AAA | 802.1X auth, identity-based access, RADIUS, policy enforcement |
| **Cloudflare** | Zero Trust / DNS | WARP agent, Zero Trust ZTNA, Gateway DNS with DoH |

---

### 🔒 VPN Architecture

| Component | Role | Key Features |
|---|---|---|
| **OpenVPN Server** | Remote Access VPN | Certificate auth, full traffic redirection, inspection-ready |
| **WireGuard** | High-Performance Tunnel | Gateway for OpenVPN traffic — double tunneling architecture |

> 💡 **Double Tunnel Design:** All OpenVPN traffic is routed through a WireGuard tunnel, adding an extra encryption layer and obfuscating the VPN fingerprint at the network edge.

---

### 📊 Monitoring & SIEM

| Component | Role | Key Features |
|---|---|---|
| **Wazuh** | SIEM / EDR | Log ingestion, intrusion detection, FIM, compliance (PCI-DSS, CIS) |
| **Zabbix** | Infrastructure Monitoring | Real-time metrics, custom dashboards, alerting |
| **OpenVAS** | Vulnerability Scanner | Network-wide scans, CVE analysis, remediation tracking |

---

### 🖥️ Virtualization & Servers

| Component | Role | Key Features |
|---|---|---|
| **VMware ESXi** | Primary Hypervisor | Bare-metal virtualization, HA, VM lifecycle management |
| **Hyper-V** | Secondary Hypervisor | Multi-system simulations, Windows-native VMs |
| **Windows Server (ADDC)** | Identity & Directory | Active Directory, DNS, DHCP, GPO, centralized user management |
| **Azure Cloud** | Hybrid Cloud | Azure AD Connect, hybrid identity, cloud services integration |
| **Nextcloud** | Private Cloud Storage | Self-hosted file sharing, internal SSL, WebDAV automation |

---

### ⚙️ Automation & Backup

| Component | Role | Key Features |
|---|---|---|
| **Rudder** | Config Management | Linux automation, compliance policies, drift detection |
| **UrBackup** | Backup Solution | Image + file backups, incremental, centralized server |

---

## 🚀 Key Projects

### ✅ Completed

- **[DNS over HTTPS Gateway]** — OPNsense + Cloudflare Zero Trust Gateway enforcing network-wide DoH with category-based DNS filtering
- **[Zero Trust Remote Access]** — Cloudflare WARP + ZTNA policies replacing traditional VPN for application access
- **[Double VPN Architecture]** — WireGuard tunnel wrapping OpenVPN traffic for layered encryption
- **[802.1X NAC with Cisco ISE]** — Identity-based network access control with RADIUS across all switch ports
- **[Hybrid AD (On-prem → Azure)]** — Azure AD Connect syncing on-premises Active Directory to Azure AD
- **[Centralized SIEM with Wazuh]** — Log collection from all lab components with custom detection rules
- **[Infrastructure Monitoring with Zabbix]** — Dashboards covering network, servers, and services with alerting

### 🔨 In Progress

| Project | Description |
|---|---|
| `HMIND SecOps Platform` | AI-powered SOC automation platform with incident response workflows |
| `Jarvis AI Assistant` | Self-hosted AI assistant with tool-calling capabilities |
| `OpenVAS Automation` | Scheduled scans + auto-reporting pipeline |
| `Rudder Full Coverage` | Full Linux fleet automation with compliance policies |

---

## 🎓 Certifications

| Certification | Status |
|---|---|
| CCNA | ✅ Completed |
| CCNP Security — SCOR 350-701 | ✅ Completed |
| Microsoft AZ-900 | ✅ Completed |
| NDG Linux / Intro to Cybersecurity | ✅ Completed |
| PCCSE (Prisma Cloud) | 🔄 In Progress |
| AZ-500 (Azure Security Engineer) | 🔄 In Progress |

---

## ⚠️ Security Notice

All configurations published here are **sanitized**:
- ❌ No private keys (WireGuard, SSL/TLS)
- ❌ No credentials, tokens, or API keys
- ❌ No real public IPs or internal addressing
- ✅ All sensitive values replaced with safe placeholders

---

## 📬 Connect

- 💼 [LinkedIn](https://linkedin.com/in/hamzachi)
- 🐙 [GitHub](https://github.com/hamzachi)

---

<div align="center">

*Every component in this lab was deployed, broken, debugged, and redeployed — that's how real learning happens.*

</div>
