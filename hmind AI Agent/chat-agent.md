<div align="center">

# 🤖 IT Chat Agent — Technical Deep-Dive

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)](.)
[![FastAPI](https://img.shields.io/badge/FastAPI-Backend-009688?style=flat-square&logo=fastapi&logoColor=white)](.)
[![Groq](https://img.shields.io/badge/Groq-LLM%20API-F55036?style=flat-square)](.)
[![SQLite](https://img.shields.io/badge/SQLite-Database-003B57?style=flat-square&logo=sqlite&logoColor=white)](.)
[![Status](https://img.shields.io/badge/Status-Active%20Development-orange?style=flat-square)](.)

**Part of the [HMIND Platform](../README.md)**

</div>

---

## 📌 Overview

The Chat Agent is an AI assistant purpose-built for IT and security professionals. It has **direct read access to live infrastructure data** — alerts, vulnerabilities, host status — and a layered response architecture designed to minimize API cost while maximizing accuracy.

> 💡 **Key principle:** Not every question needs AI. Most infrastructure queries are answered from the database instantly — zero API cost, zero latency.

---

## 🏗️ Architecture

<div align="center">

![Chat Agent Architecture](../chat_agent_architecture.svg)

</div>

---

## ⚡ 3-Layer Response System

Every message passes through three decision layers **in order**. The system only escalates to the next layer if the current one cannot handle the request.

### 🟢 Layer 1 — Pattern Match
> `0ms · No API call · No DB query`

Detects greetings, simple acknowledgements, and short phrases.

```
Input:  "hi", "thanks", "ok", "yes", "got it"
Output: Instant hardcoded reply
Cost:   $0.00 — zero API calls
```

---

### 🟡 Layer 2 — HSOC DB Query
> `Instant · Always works · Even if Groq is down`

Detects data queries about live infrastructure. Queries SQLite directly.

```
Input:  "show critical alerts", "which hosts are down?", "top CVEs"
Output: Structured answer from live DB
Cost:   $0.00 — zero API calls
```

**Keywords that trigger Layer 2:**

| Keyword group | Examples |
|---|---|
| Alert queries | `alerts`, `critical`, `warning`, `fired` |
| Host queries | `hosts`, `down`, `offline`, `unreachable` |
| Vuln queries | `CVE`, `vulns`, `CVSS`, `patch` |

---

### 🔴 Layer 3 — AI with HSOC Context
> `2–3 seconds · Groq API · Full infrastructure awareness`

Handles complex technical questions. Before calling Groq, the backend **injects live HSOC context** from the database.

```python
# Simplified — chat.py

def build_hsoc_context(db: Session) -> str:
    alerts = db.query(HSOCAlert).filter(
        HSOCAlert.severity.in_(["critical", "high"])
    ).order_by(HSOCAlert.timestamp.desc()).limit(10).all()

    vulns = db.query(HSOCVuln).order_by(
        HSOCVuln.cvss_score.desc()
    ).limit(5).all()

    hosts_down = db.query(HSOCHost).filter(
        HSOCHost.status == "down"
    ).all()

    context = f"""
    Current infrastructure state:
    - Critical/High alerts (last 10): {format_alerts(alerts)}
    - Top CVEs by CVSS: {format_vulns(vulns)}
    - Hosts currently down: {format_hosts(hosts_down)}
    """
    return context
```

> The AI receives this context on every Layer 3 call — so it can answer *"what should I investigate first?"* with actual knowledge of your current environment.

**Layer 3 handles:**
- CVE explanation and impact analysis
- Config writing (firewall rules, VPN, hardening)
- Incident response guidance
- Network and security troubleshooting

---

## 🔄 Groq API — 6-Model Fallback Chain

When a model hits its rate limit, HMIND automatically tries the next — the chat **never returns an error to the user** due to rate limiting.

```
llama-3.1-8b-instant   ← ① Primary (fastest)
        │ rate limited?
        ▼
gemma2-9b-it           ← ②
        │ rate limited?
        ▼
llama3-8b-8192         ← ③
        │ rate limited?
        ▼
llama-3.2-3b-preview   ← ④
        │ rate limited?
        ▼
llama-3.2-1b-preview   ← ⑤
        │ rate limited?
        ▼
gemma-7b-it            ← ⑥ Last resort
```

> Every attempt logs which model was used in the `model_used` field of the `messages` table — full audit trail.

---

## 🎬 Typewriter Animation

The backend returns a **complete response** (non-streaming) to avoid Cloudflare SSE buffering issues. The frontend animates the reveal client-side:

```javascript
// chat.js

async function typewriterRender(text, targetEl) {
    const parsed = marked.parse(text);      // render markdown first
    const chars = [...parsed];
    targetEl.innerHTML = "";

    return new Promise(resolve => {
        let i = 0;
        const interval = setInterval(() => {
            targetEl.innerHTML += chars[i++];
            if (i >= chars.length) {
                clearInterval(interval);
                Prism.highlightAllUnder(targetEl); // syntax highlight code blocks
                resolve();
            }
        }, 8); // 8ms per character
    });
}
```

> **Why non-streaming?** Cloudflare Zero Trust tunnels buffer SSE responses, breaking the real-time stream. Full response + client-side animation achieves the same visual effect without SSE dependency.

---

## 🗄️ Database Schema

```sql
-- Chat sessions
CREATE TABLE chat_sessions (
    id          TEXT PRIMARY KEY,       -- UUID
    created_at  DATETIME,
    title       TEXT                    -- auto-generated from first message
);

-- All messages with full audit data
CREATE TABLE messages (
    id                      INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id              TEXT REFERENCES chat_sessions(id),
    role                    TEXT,       -- "user" | "assistant"
    content                 TEXT,
    model_used              TEXT,       -- which Groq model responded
    layer_used              INTEGER,    -- 1, 2, or 3
    timestamp               DATETIME,
    hsoc_context_snapshot   TEXT        -- JSON snapshot of DB state at query time
);
```

> The `hsoc_context_snapshot` field stores exactly what infrastructure data the AI saw — useful for **auditing AI decisions after an incident**.

---

## 🔌 API Endpoints

| Method | Endpoint | Description |
|:---:|---|---|
| `POST` | `/api/chat/message` | Send message, receive AI response |
| `GET` | `/api/chat/sessions` | List all sessions |
| `GET` | `/api/chat/sessions/{id}` | Full message history |
| `DELETE` | `/api/chat/sessions/{id}` | Delete session and messages |

**Request / Response example:**

```json
POST /api/chat/message
{
  "session_id": "uuid",
  "message": "explain CVE-2024-1234 and its impact on my servers"
}
```

```json
{
  "response": "CVE-2024-1234 is a critical RCE vulnerability...",
  "layer_used": 3,
  "model_used": "llama-3.1-8b-instant",
  "hsoc_context_used": true,
  "response_time_ms": 2340
}
```

---

## 🚀 Planned Enhancements

| Feature | Description | Priority |
|---|---|:---:|
| **RAG** | Index internal runbooks into vector store — AI answers from your own docs | 🔴 High |
| **Command execution** | AI proposes fix, operator approves, HMIND executes via SSH | 🔴 High |
| **Conversation memory** | Persistent context across sessions for long investigations | 🟡 Medium |
| **Network page integration** | Ask about switch ports, VLANs, interface status in natural language | 🟡 Medium |
| **Multi-language** | Arabic interface for MENA market | 🟢 Low |

---

## 🔗 How Chat Agent and HSOC Communicate

The Chat Agent and HSOC are **not two separate applications**. They are two interfaces to the same backend and the same SQLite database. Communication happens through shared data — no API calls between modules.

### Shared Database — The Bridge

```
Chat Agent                          HSOC Module
    │                                   │
    │   READ  hsoc_alerts               │   WRITE hsoc_alerts
    │   READ  hsoc_vulnerabilities      │   WRITE hsoc_vulnerabilities
    │   READ  hsoc_hosts                │   WRITE hsoc_hosts
    │   WRITE messages                  │
    │                                   │
    └──────────────┬────────────────────┘
                   │
            SQLite (single file)
```

### Real Example

```
14:00  hsoc_agent pushes 3 new critical Wazuh alerts to DB
14:01  HSOC dashboard refreshes — analyst sees alerts in red
14:02  Analyst asks chat: "what are the critical alerts right now?"
14:02  Layer 2 queries:
         SELECT * FROM hsoc_alerts
         WHERE severity = 'critical'
         ORDER BY timestamp DESC LIMIT 10
       Returns the SAME 3 alerts — same DB, same data, zero duplication
```

### Shared AI Cache

When HSOC generates an AI explanation (analyst clicks *"Explain"*), the result is **cached in the database**. If the same alert is later asked about in chat, the cached response is returned instantly — **no second API call**.

```python
# ai.py — called by BOTH chat.py and hsoc.py

async def explain_with_groq(prompt: str, cache_key: str) -> str:
    cached = db.query(AICache).filter_by(key=cache_key).first()
    if cached:
        return cached.response          # instant — no API call

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
| Audit trail | Every AI call logged in `messages` regardless of which module triggered it |

---

<div align="center">

*[← Back to Achahib-Lab Platform](../README.md) · [HSOC Documentation →](./hsoc.md)*

</div>
