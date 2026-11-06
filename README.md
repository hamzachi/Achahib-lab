<div align="center">

# 🏗️ Achahib Lab

**A production-grade home lab simulating enterprise-level networking, security, virtualization, and cloud infrastructure.**

[![Firewall](https://img.shields.io/badge/Firewall-OPNsense%20%2B%20Zenarmor-blue?style=flat-square)](.)
[![SIEM](https://img.shields.io/badge/SIEM-Wazuh-red?style=flat-square)](.)
[![VPN](https://img.shields.io/badge/VPN-WireGuard%20%2B%20OpenVPN-green?style=flat-square)](.)
[![Virtualization](https://img.shields.io/badge/Virtualization-ESXi%20%2B%20Hyper--V-purple?style=flat-square)](.)
[![Cloud](https://img.shields.io/badge/Cloud-Azure%20%2B%20Cloudflare%20ZT-orange?style=flat-square)](.)
[![NAC](https://img.shields.io/badge/NAC-PacketFence%20EAP--TLS-lightgrey?style=flat-square)](.)

</div>

---

## 👤 About

Hi, I'm **Hamza** — an IT Engineer based in Morocco with **5+ years of experience** in networking, system administration, and cybersecurity. This repository is the central documentation hub for **Achahib Lab** — a fully self-hosted, enterprise-simulated environment covering network security, identity management, SIEM, vulnerability management, hybrid cloud, and infrastructure automation.

> *"Built to learn by doing — every component here mirrors a real enterprise deployment scenario."*

---

## 🗺️ Lab Architecture

<div align="center">

![Achahib Lab Network Topology](achahib_lab_topology.svg)

</div>

> **Legend:** Solid lines = physical connections · Orange dashed = Cloudflare Zero Trust tunnel · Blue dashed = Azure AD hybrid sync

---

## 📋 VLAN Table

| VLAN ID | Name | Purpose | Status |
|---|---|---|---|
| **1** | MGMT (Default) | All servers, management interfaces, infrastructure | ✅ Active |
| **11** | WIFI | Wireless clients | ✅ Active |
| **20** | LABs | Lab environments and testing machines | ✅ Active |
| **44** | VOIP | Voice over IP devices and PBX | ✅ Active |
| **50** | DMZ | Exposed services, public-facing servers | ✅ Active |
| **98** | INT-LABs | Internal isolated lab networks | ✅ Active |
| **100** | HOME | Home devices and personal endpoints | ✅ Active |
| **101** | Test | Temporary testing and sandboxing | ✅ Active |

---

## 📂 Documentation Layout

The tree below is the planned documentation layout; only published sections are linked in the index that follows.

```
achahib-lab/
├── 📁 network/
│   ├── opnsense/             # OPNsense + Zenarmor configs (sanitized)
│   ├── cisco-router/         # Inter-VLAN routing, ACLs, QoS
│   ├── cisco-switch/         # VLAN trunking, 802.1X port configs
│   ├── packetfence/            # ISE policies, RADIUS, NAC profiles
│   └── diagrams/             # Network topology diagrams
├── 📁 vlans/
│   ├── vlan-table.md         # Full VLAN documentation
│   └── acl-policies/         # Inter-VLAN ACL rules
├── 📁 vpn/
│   ├── wireguard/            # WireGuard OPNsense client config
│   ├── openvpn/              # OpenVPN server setup & auth
│   └── double-tunnel/        # WireGuard → OpenVPN architecture docs
├── 📁 zero-trust/
│   ├── cloudflare-warp/      # WARP client deployment & policies
│   ├── ztna-apps/            # Application access policies (on-prem)
│   └── gateway-dns/          # DoH policies, category filtering
├── 📁 security/
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

### Published documentation

- [Cloudflare Zero Trust](Cloudflare%20Zero%20Trust/cloudflare-zero-trust-setup.md)
- [PacketFence NAC implementation](network/packetfence/README.md)
- [Centralized automated backup](backup/urbackup/README.md)
- [HMIND Chat Agent](hmind%20AI%20Agent/chat-agent.md)
- [HMIND HNTA](hmind%20AI%20Agent/hnta.md)
- [HMIND HSOC](hmind%20AI%20Agent/hsoc.md)
- [HMIND Networking](hmind%20AI%20Agent/networking.md)

## 🧩 Lab Components

### 🔐 Network Security & Edge

| Component | Role | Key Features |
|---|---|---|
| **OPNsense** | Edge Firewall / Gateway | NAT, DoH gateway, VLAN routing, WireGuard client |
| **Zenarmor** | NGFW Plugin | IDS/IPS, SNI inspection, app-layer control, TLS inspection |
| **Cisco Router** | Distribution Layer | Inter-VLAN routing, ACLs, QoS, policy enforcement |
| **Cisco Switch** | Layer 2 Core | VLAN trunking, port segmentation, 802.1X enforcement |
| **PacketFence** | NAC / AAA | AD integration, RADIUS, wired 802.1X EAP-TLS; NOC-SURFACE validated on Catalyst 3750 |

---

### ☁️ Zero Trust & Cloud Access

| Component | Role | Key Features |
|---|---|---|
| **Cloudflare WARP** | Zero Trust Client | Encrypted tunnel to Cloudflare network from any device |
| **Cloudflare ZTNA** | App Access (Zero Trust) | Access on-prem applications without exposing them to internet |
| **Cloudflare Gateway** | Cloud Security | DNS/network/web policies, filtering and centralized logs |
| **Achahib-ZEROTRUST** | cloudflared on OPNsense | Outbound-only connector for published applications and configured private routes |
| **Achahib-FullTunnel** | Ubuntu Cloudflare Mesh node | Formerly WARP Connector; configured LAN Internet egress through Cloudflare |
| **Microsoft Entra ID** | Cloudflare identity provider | Hybrid identity with on-prem AD; Access policy authentication |

**Remote access:** WARP client → Cloudflare Access / configured private routes → Achahib-ZEROTRUST (cloudflared on OPNsense) → internal services. Access policies protect the configured applications; no inbound application port forwarding is required.

**Cloudflare Internet egress:** Selected LAN traffic → Cisco VLAN gateway → OPNsense + Zenarmor → Ubuntu Mesh node (Achahib-FullTunnel) → Cloudflare Gateway → Internet. Routing policy determines which networks use this path; existing VPN and other egress paths remain separate.

**Identity and domain services:** On-prem AD (`achahib.local`) synchronizes with Microsoft Entra ID. AD remains responsible for DNS, Kerberos and GPO; Cloudflare provides secure connectivity and access controls. Public source IP behavior depends on the selected egress mode.

---

### 🔒 VPN Architecture

| Component | Role | Key Features |
|---|---|---|
| **WireGuard (OPNsense)** | Outbound VPN Client | Tunnels traffic from OPNsense to VPS gateway |
| **VPS (WG Server)** | VPN Gateway | Acts as the WireGuard server and OpenVPN relay |
| **OpenVPN** | Remote Access VPN | Full traffic redirection through WireGuard tunnel |

> 💡 **Double Tunnel:** `Client → OpenVPN → WireGuard → VPS → Internet` — OpenVPN traffic is wrapped inside WireGuard for layered encryption and fingerprint obfuscation.

---

### 📊 Monitoring & SIEM

| Component | Role | Key Features |
|---|---|---|
| **Wazuh** | SIEM / EDR | Log ingestion, intrusion detection, FIM, compliance |
| **Zabbix** | Infrastructure Monitoring | Real-time metrics, dashboards, alerting |
| **OpenVAS** | Vulnerability Scanner | Network-wide scans, CVE analysis, remediation |

---

### 🖥️ Virtualization & Servers

| Component | Role | Key Features |
|---|---|---|
| **VMware ESXi** | Primary Hypervisor | Bare-metal virtualization, HA, VM management |
| **Hyper-V** | Secondary Hypervisor | Multi-system simulations, Windows-native VMs |
| **Windows Server (ADDC)** | Identity & Directory | Active Directory, DNS, DHCP, GPO |
| **Azure Cloud** | Hybrid Cloud | Azure AD Connect, hybrid identity integration |
| **Nextcloud** | Private Cloud Storage | Self-hosted file sharing, internal SSL, automation |

---

### ⚙️ Automation & Backup

| Component | Role | Key Features |
|---|---|---|
| **Rudder** | Config Management | Linux automation, compliance policies, drift detection; install_urbackup technique |
| **UrBackup** | Backup Solution | Centralized server backups; Windows deployment via GPO / Backup_Servers, Linux via Rudder |
| **SSH backup automation** | Network configuration archive | Daily cron jobs from UrBackup server; Cisco running configs and firewall XML, compressed archives and success/error logs |

---

## 🚀 Key Projects

### ✅ Completed

- **[DNS over HTTPS Gateway]** — OPNsense + Cloudflare Gateway enforcing network-wide DoH with category-based DNS filtering
- **[Zero Trust Remote Access]** — Cloudflare WARP + ZTNA replacing traditional VPN — on-prem apps accessible without port forwarding
- **[Double VPN Architecture]** — WireGuard tunnel (OPNsense → VPS) wrapping OpenVPN for layered encryption
- **[802.1X NAC with PacketFence](network/packetfence/README.md)** — AD-integrated RADIUS with EAP-TLS validated on NOC-SURFACE / Cisco Catalyst 3750 Fa3/0/13; broader role/VLAN and wireless validation remains separate
- **[VLAN Segmentation]** — 8 VLANs with full inter-VLAN ACL policy enforcement
- **[Hybrid AD (On-prem → Azure)]** — Azure AD Connect syncing on-premises AD to Azure AD
- **[Centralized SIEM with Wazuh]** — Log collection from all components with custom detection rules
- **[Infrastructure Monitoring]** — Zabbix dashboards covering all network and server components

- **[Cloudflare Mesh Egress](Cloudflare%20Zero%20Trust/cloudflare-zero-trust-setup.md)** — Ubuntu Mesh node for the selected LAN Internet path, alongside cloudflared private access
- **[Centralized Automated Backup](backup/urbackup/README.md)** — Windows via AD GPO, Linux via Rudder, and daily SSH network configuration archives

### 🔨 In Progress

| Project | Description |
|---|---|
| `HMIND SecOps Platform` | AI-powered SOC automation with incident response workflows |
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
| AZ-700 (Azure Security Engineer) | 🔄 In Progress |

---

## ⚠️ Security Notice

All configurations published here are **sanitized**:
- ❌ No private keys (WireGuard, SSL/TLS)
- ❌ No credentials, tokens, or API keys
- ❌ No real public IPs or internal addressing
- ✅ All sensitive values replaced with safe placeholders

---

## 📬 Connect

- 💼 [LinkedIn](www.linkedin.com/in/hamza-a-8117b51a8)
- 🐙 [GitHub](https://github.com/hamzachi)

---

<div align="center">

*Every component in this lab was deployed, broken, debugged, and redeployed — that's how real learning happens.*

</div>
