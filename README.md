# 🛡️ Mini-SIEM CY-01 — Attack Sequence Detection via Deterministic Finite Automaton

> **ENSAM Casablanca · Cybersecurity & Cloud Computing · CSCC 2025–2026**

A fully functional mini Security Information and Event Management (SIEM) system that detects multi-step attack sequences in real time using a **Deterministic Finite Automaton (DFA/AFD)**, built with n8n, Google Sheets, Slack, and WhatsApp (Twilio).

---

## 📐 How It Works

The system models attacker behavior as a state machine. Each incoming security log triggers a state transition, and when a full attack chain is detected, simultaneous alerts are sent to both Slack and WhatsApp.

### DFA States

| State | Label | Meaning |
|-------|-------|---------|
| `q0` | Calm | No suspicious activity |
| `q1` | Suspect | Auth failure threshold reached |
| `q2` | Attack_In_Progress | Port scan detected after suspicious phase |
| `q3` | **Compromised** | Privilege escalation — critical alert triggered |

### Transition Table

| | `b` (brute force) | `s` (scan) | `p` (privilege) | `n` (normal fail) | `r` (reset/ack) |
|--|--|--|--|--|--|
| **q0** | q1 | q0 | q0 | q0 | q0 |
| **q1** | q1 | q2 | q1 | q1 | q1 |
| **q2** | q2 | q2 | **q3** | q2 | q2 |
| **q3** | q3 | q3 | q3 | q3 | q0 |

The only way to exit `q3` is an explicit SOC acknowledgment (`alert_ack` event), which resets to `q0`.

### Temporal Parameters

| Parameter | Value | Role |
|-----------|-------|------|
| `Nauth` | 3 failures | Threshold to transition to `q1` |
| `Wauth` | 5 min | Sliding window for `auth_fail` events |
| `Treset` | 15 min | Inactivity timeout → auto-reset to `q0` |

---

## 🏗️ Architecture

```
POST /webhook
     │
     ▼
┌─────────────────────┐
│  n8n Workflow        │
│                     │
│  1. Webhook trigger  │
│  2. Read state       │◄──── Google Sheets (state store)
│     from GSheets    │
│  3. AFD Engine (JS)  │
│  4. Write new state  │────► Google Sheets
│  5. IF alert q3?     │
│     ├─ YES ──────────┼────► Slack #alerts-siem
│     │                │────► WhatsApp (Twilio)
│     └─ NO → Switch   │
│        ├── q1: warn  │
│        ├── q2: log   │
│        └── q0: ok    │
│  6. Webhook Response │
└─────────────────────┘
```

**10-node n8n workflow** deployed on n8n Cloud, processing logs in real time.

---

## 🚀 Setup & Deployment

### Prerequisites

- [n8n Cloud](https://app.n8n.cloud) account (or self-hosted)
- Google account with Google Sheets API enabled
- Slack workspace with Incoming Webhooks app
- Twilio account with WhatsApp Sandbox activated

### 1. Google Sheets — State Store

Create a Google Sheet with these columns (in order):

```
ip_source | event_type | symbol | state_before | state_after | current_label | auth_fail_count | is_alert | timestamp
```

Copy the Sheet ID from its URL and use it as `YOUR_GOOGLE_SHEETS_ID`.

### 2. Slack — Incoming Webhook

1. Go to your Slack workspace → **Apps → Incoming Webhooks**
2. Create a new webhook pointing to a `#alerts-siem` channel
3. Copy the webhook URL → use as `YOUR_SLACK_WEBHOOK_TOKEN`

### 3. Twilio — WhatsApp Sandbox

1. Create an account at [twilio.com](https://www.twilio.com)
2. Go to **Messaging → Try it out → Send a WhatsApp message**
3. Send the join code shown in your Twilio console to `+14155238886` from WhatsApp
4. Collect your **Account SID** and **Auth Token** from the Twilio console

### 4. Import the Workflow into n8n

1. Open n8n → **Workflows → Import from File**
2. Select `workflow.json`
3. Update the following placeholders in the workflow nodes:

| Placeholder | Where | What to replace with |
|-------------|-------|----------------------|
| `YOUR_GOOGLE_SHEETS_ID` | Google Sheets nodes | Your Sheet ID |
| `YOUR_SLACK_WEBHOOK_TOKEN` | HTTP Request (Slack) | Full webhook URL |
| `YOUR_TWILIO_ACCOUNT_SID` | HTTP Request (WhatsApp) URL | Your Twilio Account SID |
| `YOUR_TWILIO_AUTH_TOKEN` | HTTP Basic Auth credentials | Your Twilio Auth Token |
| `YOUR_ANALYST_WHATSAPP_NUMBER` | HTTP Request (WhatsApp) body | Analyst's number in `whatsapp:+XXXXX` format |
| `YOUR_N8N_INSTANCE.app.n8n.cloud` | Webhook node | Your n8n instance URL |

5. Set up Google Sheets credentials in n8n (OAuth2)
6. **Publish** the workflow (required for production mode)

---

## 📡 API Usage

Send security log events as POST requests to your webhook URL:

```bash
# Auth failure
curl -X POST https://YOUR_N8N_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{"ip_source": "192.168.1.100", "event_type": "auth_fail"}'

# Port scan
curl -X POST https://YOUR_N8N_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{"ip_source": "192.168.1.100", "event_type": "port_scan"}'

# Privilege escalation (triggers ALERT if in q2)
curl -X POST https://YOUR_N8N_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{"ip_source": "192.168.1.100", "event_type": "privilege_escalation"}'

# SOC acknowledgment (resets q3 → q0)
curl -X POST https://YOUR_N8N_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{"ip_source": "192.168.1.100", "event_type": "alert_ack"}'
```

### Supported Event Types

| `event_type` | DFA Symbol | Effect |
|---|---|---|
| `auth_fail` | `b` (if ≥3 in 5 min) or `n` | Brute force detection |
| `port_scan`, `nmap_scan` | `s` | Scan detection |
| `privilege_escalation`, `sudo`, `su_root` | `p` | Escalation detection |
| `alert_ack`, `reset` | `r` | SOC acknowledgment |

### Webhook Response

```json
{
  "status": "processed",
  "state": "q3",
  "ip": "192.168.1.100",
  "is_alert": true,
  "label": "Compromis",
  "timestamp": "2026-05-03T13:00:00.373Z"
}
```

---

## 🧪 Testing

The full attack sequence can be reproduced with 5 requests:

```
auth_fail × 3  →  q0 → q0 → q1  (brute force threshold)
port_scan      →  q1 → q2        (scan detected)
priv_escalation→  q2 → q3 🚨    (ALERT: Slack + WhatsApp)
alert_ack      →  q3 → q0        (SOC acknowledgment)
```

We used [Postman](https://www.postman.com) for testing.

---

## 📁 Repository Structure

```
mini-siem-cy01/
├── workflow.json   # n8n workflow (no secrets)
├── README.md                 # This file
├── docs/
│   └── rapport_CY01.pdf      # Full technical report (French)
└── .gitignore
```

---

## ⚙️ Known Limitations

- **Scalability**: Google Sheets slows down beyond a few hundred rows. Redis or PostgreSQL would be preferred in production.
- **Alert deduplication**: Rapid consecutive tests can produce duplicate `q3` alerts. A TTL-based dedup mechanism would help.
- **Twilio Sandbox**: Limited to pre-approved recipients. A Meta-approved WhatsApp Business account is required for real deployment.
- **Multi-IP concurrency**: Google Sheets reads are sequential, creating bottlenecks under load.

---

## 🤝 Collaboration

This project was developed as a team effort by two students from ENSAM Casablanca.

| Name | GitHub |
|------|--------|
| **Arwa Boudarfa** | — |
| **Ihssane El Maataoui** | [@elmaataoui-ihssane](https://github.com/elmaataoui-ihssane) |

## 📄 License

This project was developed as an academic mini-project at **ENSAM Casablanca** for the CSCC 2025–2026 curriculum. Not intended for production use without the improvements noted above.
