<div align="center">

# 🛡️ HSOC — Security Operations Center

[![Wazuh](https://img.shields.io/badge/Wazuh-SIEM%20%7C%20EDR-005571?style=flat-square)](.)
[![Zabbix](https://img.shields.io/badge/Zabbix-Monitoring-CC0000?style=flat-square&logo=zabbix&logoColor=white)](.)
[![OpenVAS](https://img.shields.io/badge/OpenVAS-Vuln%20Scanner-4CAF50?style=flat-square)](.)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)](.)
[![Status](https://img.shields.io/badge/Status-Active%20Development-orange?style=flat-square)](.)

**Part of the [HMIND Platform](../README.md)**

</div>

---

## 📌 Overview

HSOC is the live security monitoring module of HMIND. It aggregates data from **three independent security systems**, normalizes everything into a unified schema, and presents it in a single dashboard with AI-driven analysis per alert, vulnerability, and host.

> 💡 **Key principle:** One dashboard. Three data sources. AI explanation for everything.

---

## 🏗️ Architecture

<div align="center">

![HSOC Architecture](https://raw.githubusercontent.com/hamzachi/Achahib-lab/main/hmind%20AI%20Agent/Architecture/hsoc_architecture.svg)
</div>

---

## 📡 Data Sources

| Source | What It Collects | API | Frequency |
|---|---|:---:|:---:|
| **🔴 Wazuh** | Security alerts · IDS · auth failures · malware · Windows/Linux events | OpenSearch `:9200` | Every 5 min |
| **🟠 Zabbix** | Infrastructure alerts · host availability · network monitoring | JSON-RPC | Every 5 min |
| **🟢 OpenVAS** | CVEs · CVSS scores · affected services · patch solutions | GMP Unix socket | Every 5 min |

---

## ⚙️ hsoc_agent.py — How It Works

A lightweight Python script installed on each security server. It runs every 5 minutes via `cron` or `systemd timer`.

### 🔴 Wazuh — OpenSearch API

```python
def collect_wazuh(config):
    url = f"https://{config['wazuh_host']}:9200/wazuh-alerts-*/_search"
    query = {
        "query": {
            "range": {
                "@timestamp": {
                    "gte": last_checkpoint,   # never re-send old alerts
                    "lte": "now"
                }
            }
        },
        "sort": [{"@timestamp": "desc"}],
        "size": 100
    }
    response = requests.post(url, json=query, auth=HTTPBasicAuth(...), verify=False)
    hits = response.json()["hits"]["hits"]

    alerts = []
    for hit in hits:
        src = hit["_source"]
        alerts.append({
            "source":    "wazuh",
            "severity":  map_severity(src.get("rule", {}).get("level", 0)),
            "title":     src.get("rule", {}).get("description", ""),
            "agent":     src.get("agent", {}).get("name", ""),
            "timestamp": src.get("@timestamp"),
            "raw":       json.dumps(src)
        })
    return alerts
```

### 🟠 Zabbix — JSON-RPC API

```python
def collect_zabbix(config):
    token = zabbix_rpc("user.login", {
        "user": config["zabbix_user"],
        "password": config["zabbix_pass"]
    })
    problems = zabbix_rpc("problem.get", {
        "output": "extend",
        "selectHosts": ["host"],
        "severities": [2, 3, 4, 5],    # warning → disaster
        "time_from": last_checkpoint
    })
    alerts = []
    for p in problems:
        alerts.append({
            "source":    "zabbix",
            "severity":  map_zabbix_severity(p["severity"]),
            "title":     p["name"],
            "agent":     p["hosts"][0]["host"] if p["hosts"] else "unknown",
            "timestamp": datetime.fromtimestamp(int(p["clock"])).isoformat(),
            "raw":       json.dumps(p)
        })
    return alerts
```

### 🟢 OpenVAS — GMP Unix Socket

```python
def collect_openvas(config):
    with gvm.connections.UnixSocketConnection(
        path="/run/gvmd/gvmd.sock"
    ) as conn:
        gmp = Gmp(conn, transform=EtreeCheckCommandTransform())
        gmp.authenticate(config["openvas_user"], config["openvas_pass"])
        results = gmp.get_results(filter_string="rows=100 sort-reverse=severity")

        vulns = []
        for result in results.findall("result"):
            cvss = float(result.findtext("severity", "0"))
            vulns.append({
                "cve":        result.findtext("nvt/refs/ref[@type='cve']/@id", ""),
                "name":       result.findtext("name", ""),
                "cvss_score": cvss,
                "severity":   map_cvss_to_severity(cvss),
                "host":       result.findtext("host/hostname", ""),
                "solution":   result.findtext("nvt/solution", ""),
            })
    return vulns
```

---

## 🔁 Checkpoint Tracking — No Duplicate Data

After every successful push, the agent saves a checkpoint file locally. On the next run, it only pulls data **newer than the checkpoint** — no duplicates ever reach HMIND.

```json
{
  "wazuh_last_timestamp": "2025-01-15T14:32:00Z",
  "zabbix_last_clock": 1736951520,
  "openvas_last_result_id": "a3f9c2b1-..."
}
```

---

## 📊 Severity Normalization

All three sources use **different severity scales**. The agent normalizes everything to a unified 4-level schema before pushing to HMIND.

| HMIND Level | Wazuh Rule Level | Zabbix Severity | OpenVAS CVSS |
|:---:|:---:|:---:|:---:|
| 🟢 `low` | 0–6 | Information / Warning | 0.1–3.9 |
| 🟡 `medium` | 7–10 | Average | 4.0–6.9 |
| 🟠 `high` | 11–13 | High | 7.0–8.9 |
| 🔴 `critical` | 14–15 | Disaster | 9.0–10.0 |

---

## 🗄️ Database Schema

```sql
-- All security alerts from Wazuh and Zabbix
CREATE TABLE hsoc_alerts (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    source      TEXT NOT NULL,          -- "wazuh" | "zabbix"
    severity    TEXT NOT NULL,          -- "low" | "medium" | "high" | "critical"
    title       TEXT NOT NULL,
    agent       TEXT,                   -- hostname of affected machine
    timestamp   DATETIME NOT NULL,
    raw         TEXT,                   -- full original JSON from source API
    ai_analysis TEXT,                   -- cached AI explanation
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(source, title, agent, timestamp)
);

-- CVEs from OpenVAS
CREATE TABLE hsoc_vulnerabilities (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    cve         TEXT,
    name        TEXT NOT NULL,
    cvss_score  REAL,
    severity    TEXT NOT NULL,
    host        TEXT,
    solution    TEXT,
    raw         TEXT,
    ai_analysis TEXT,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(cve, host)
);

-- All monitored hosts with live status
CREATE TABLE hsoc_hosts (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    hostname    TEXT NOT NULL UNIQUE,
    ip_address  TEXT,
    status      TEXT DEFAULT "unknown",  -- "up" | "down" | "unknown"
    source      TEXT,
    last_seen   DATETIME,
    os          TEXT,
    updated_at  DATETIME
);

-- Agent connectivity tracking
CREATE TABLE hsoc_agent_servers (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT NOT NULL UNIQUE,   -- "wazuh" | "zabbix" | "openvas"
    status          TEXT,                   -- "online" | "offline"
    last_contact    DATETIME,
    alerts_received INTEGER DEFAULT 0,
    error_message   TEXT
);
```

> ⚠️ **UNIQUE constraint fix:** Early versions crashed on duplicate records during overlapping agent runs. Fixed by calling `db.flush()` before `db.commit()` — this forces SQLite to raise `IntegrityError` per-record instead of per-batch, so one duplicate does not roll back the entire ingestion.

---

## 📋 Dashboard Panels

| Panel | Data Source | Key Info |
|---|---|---|
| **Overview** | All tables | Total alerts, vulns, hosts — at a glance |
| **Wazuh Alerts** | `hsoc_alerts WHERE source='wazuh'` | Sorted by severity → timestamp |
| **Zabbix Alerts** | `hsoc_alerts WHERE source='zabbix'` | Sorted by severity → timestamp |
| **Vulnerabilities** | `hsoc_vulnerabilities` | Sorted by CVSS score DESC |
| **Hosts** | `hsoc_hosts` | 🟢 green = up · 🔴 red = down |
| **Servers** | `hsoc_agent_servers` | Agent online/offline per source |

---

## 🤖 AI Explanation Flow

When an analyst clicks **"Explain"** on any alert, vuln, or host:

```
Analyst clicks "Explain" on alert #142
              │
              ▼
POST /api/hsoc/explain
{ "type": "alert", "id": 142 }
              │
              ▼
Backend fetches:
  ├── alert raw JSON
  └── last 5 events from same host

              │
              ▼
Prompt built:
  "You are a security analyst. Explain this Wazuh alert.
   What triggered it, what the attacker may have done,
   recommended response.
   Alert: {raw}
   Host history: {last_5_events}"
              │
              ▼
Groq API → response cached in hsoc_alerts.ai_analysis
              │
              ▼
✅ Returned to dashboard · available to Chat Agent from cache
```

---

## 🔌 API Endpoints

| Method | Endpoint | Description |
|:---:|---|---|
| `POST` | `/api/hsoc/ingest` | Receive push from hsoc_agent.py |
| `GET` | `/api/hsoc/alerts` | List alerts with filtering |
| `GET` | `/api/hsoc/vulnerabilities` | List CVEs sorted by CVSS |
| `GET` | `/api/hsoc/hosts` | All hosts with status |
| `GET` | `/api/hsoc/servers` | Agent connectivity |
| `POST` | `/api/hsoc/explain` | AI explanation for alert/vuln/host |
| `GET` | `/api/hsoc/summary` | Overview counts |

---

## 🚀 Planned Enhancements

| Feature | Description | Priority |
|---|---|:---:|
| **Auto-healing** | Critical alert fires → HMIND runs playbook (block IP, restart service, isolate host) | 🔴 High |
| **Correlation engine** | Same IP in Wazuh + Zabbix simultaneously → unified incident | 🔴 High |
| **Threat intel enrichment** | Enrich IPs/CVEs with AbuseIPDB / MISP on ingest | 🟡 Medium |
| **Network page integration** | Map alerts to switch ports and VLANs for faster triage | 🟡 Medium |
| **Email / Telegram alerting** | Push criticals without requiring dashboard access | 🟡 Medium |
| **Retention policies** | Auto-archive alerts older than N days | 🟢 Low |

---

## 🔗 How HSOC and Chat Agent Communicate

HSOC and the Chat Agent are **not two separate applications**. They share the same backend and the same SQLite database — communication happens through shared data, not API calls.

### Shared Database — The Bridge

```
HSOC Module                         Chat Agent
    │                                   │
    │   WRITE hsoc_alerts               │   READ hsoc_alerts
    │   WRITE hsoc_vulnerabilities      │   READ hsoc_vulnerabilities
    │   WRITE hsoc_hosts                │   READ hsoc_hosts
    │   WRITE hsoc_agent_servers        │
    │                                   │
    └──────────────┬────────────────────┘
                   │
            SQLite (single file, shared schema)
```

### Data Flow — End to End

```
Step 1  hsoc_agent.py pulls Wazuh + Zabbix + OpenVAS (every 5 min)
            │
            ▼
Step 2  POST /api/hsoc/ingest → FastAPI writes to SQLite
            │
            ├──► Step 3a  HSOC dashboard reads panels — analyst sees alerts
            │
            └──► Step 3b  Chat Agent reads same table as AI context
                          User asks: "show critical alerts"
                          Layer 2 queries DB directly — same data, zero duplication
```

### Shared AI Cache

When HSOC generates an AI explanation, it is **cached in `hsoc_alerts.ai_analysis`**. If the chat is later asked about the same alert, the cached explanation is returned — **no second API call**.

```python
# ai.py — called by BOTH hsoc.py and chat.py

async def explain_with_groq(prompt: str, cache_key: str) -> str:
    cached = db.query(AICache).filter_by(key=cache_key).first()
    if cached:
        return cached.response      # instant — no Groq call

    response = await call_groq_with_fallback(prompt)
    db.add(AICache(key=cache_key, response=response))
    db.commit()
    return response
```

### Summary

| What | How |
|---|---|
| HSOC writes data | `hsoc_agent.py` → `POST /api/hsoc/ingest` → SQLite |
| Chat reads data | Direct DB query in Layer 2 and context builder |
| Both use AI | Same `explain_with_groq()` — shared cache in DB |
| No duplication | One database, one schema, two interfaces |
| Audit trail | Every AI interaction logged in `messages` regardless of which module triggered it |

---

<div align="center">

*[← Back to Achahib-Lab Platform](../README.md) · [Chat Agent Documentation →](./chat-agent.md)*

</div>
