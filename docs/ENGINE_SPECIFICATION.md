# Securigation Engine & Pipeline Specification

## 1. Investigation & Context Optimization Engine

The Securigation Investigation Engine manages incident multi-turn investigation state, candidate evidence collection, Paritok token compression, grounded response generation, and attack timeline construction.

```
[User Query] ──► [Candidate Event Retrieval] ──► [Evidence Pack Construction]
                                                           │
                                                           ▼
[Grounded Answer + Timeline + Graph] ◄── [Groq LLM] ◄── [Paritok Compression Engine]
```

### Core Pipeline Stages

#### Stage 1: Log Parsing & Normalization
Incoming log streams across Apache, Linux Syslog, Windows EVTX, AWS CloudTrail, and Palo Alto Firewall formats are parsed into unified `SecurityEvent` schemas:
```json
{
  "id": "evt_49152",
  "timestamp": "2026-09-20T13:04:12Z",
  "log_type": "SYSLOG",
  "severity": "HIGH",
  "source_ip": "192.168.1.105",
  "destination_ip": "10.0.0.1",
  "user": "root",
  "action": "Failed password",
  "raw_log": "2026-09-20T13:04:12Z linux-sec-node sshd[4102]: Failed password for invalid user root from 192.168.1.105 port 49152 ssh2"
}
```

#### Stage 2: Candidate Search Retrieval
The engine executes ranked hybrid searches matching query terms (IPs, usernames, commands, event codes) and time windows. Up to 1,000 candidate log events (~80k tokens) are extracted into an `EvidencePack`.

#### Stage 3: Paritok Context Optimization Layer
The retrieved `EvidencePack` payload is passed to Paritok Context Optimization. Paritok filters duplicate log noise, strips boilerplate formatting, and retains 100% of high-density threat evidence:
- **Raw Context**: ~84,200 tokens (1,500 raw log lines)
- **Paritok Compressed Context**: ~767 tokens (15 high-density evidence items)
- **Token Efficiency**: 99.0% compression ratio

#### Stage 4: Grounded LLM Reasoning
The compressed context is injected into Groq Llama-3.3-70B with enforced JSON schema grounding. The model returns:
1. Grounded natural language summary
2. Verified list of evidence event IDs
3. Formatted attack timeline events
4. Attack graph entity nodes & edges

---

## 2. Database Models & Schema Specifications

### Primary Entities & Relationships
```
┌─────────────────┐       1:N       ┌──────────────────────┐
│  Investigation  ├────────────────►│     UploadedFile     │
└────────┬────────┘                 └──────────────────────┘
         │
         │ 1:N                      1:1
         ├─────────────────────────►┌──────────────────────┐
         │                          │  ConversationQuery   │
         │                          └──────────┬───────────┘
         │                                     │
         │                                     ▼
         │                          ┌──────────────────────┐
         │                          │    ParitokMetric     │
         │                          └──────────────────────┘
         │ 1:N
         ├─────────────────────────►┌──────────────────────┐
         │                          │    TimelineEvent     │
         │                          └──────────────────────┘
         │ 1:N
         └─────────────────────────►┌──────────────────────┐
                                    │    EvidenceItem      │
                                    └──────────────────────┘
```

#### Entity Definitions
- **`Investigation`**: Master container (`id`, `title`, `description`, `status`, `created_at`, `updated_at`).
- **`UploadedFile`**: Metadata for uploaded log feeds (`id`, `investigation_id`, `filename`, `file_format`, `file_size_bytes`, `event_count`).
- **`ConversationQuery`**: Single investigation interaction (`id`, `investigation_id`, `question`, `answer`, `created_at`).
- **`ParitokMetric`**: Token compression telemetry (`id`, `query_id`, `events_raw`, `events_compressed`, `tokens_raw`, `tokens_compressed`, `cost_saved_usd`, `latency_saved_sec`).
- **`TimelineEvent`**: Grounded attack chronology (`id`, `investigation_id`, `timestamp`, `title`, `description`, `severity`, `tactic`).

---

## 3. REST API Specification

### Endpoint 1: Upload Log File
- **`POST /api/v1/investigations/{id}/upload`**
- **Content-Type**: `multipart/form-data`
- **Response `200 OK`**:
```json
{
  "file_id": "file_8819a",
  "filename": "apt29_auth_security.log",
  "format": "SYSLOG",
  "events_parsed": 1500,
  "status": "READY"
}
```

### Endpoint 2: Execute Incident Query
- **`POST /api/v1/investigations/{id}/query`**
- **Request Body**:
```json
{
  "question": "Show all malicious SSH login spikes and root escalation commands"
}
```
- **Response `200 OK`**:
```json
{
  "query_id": "347b3b21-d0d3-4523-af03-49a83be29f1f",
  "question": "Show all malicious SSH login spikes and root escalation commands",
  "answer": "Multiple malicious SSH login spikes from 192.168.1.105 followed by root privilege escalation.",
  "evidence_used": [
    "[2026-09-20T13:04:12Z] linux-sec-node sshd[4102]: Failed password for root from 192.168.1.105",
    "[2026-09-20T13:07:10Z] linux-sec-node sudo: admin : TTY=pts/0 ; USER=root ; COMMAND=/bin/bash"
  ],
  "paritok_metrics": {
    "without_paritok": { "events": 1500, "tokens": 76804, "cost_usd": 0.00384, "latency_sec": 8.18 },
    "with_paritok": { "events": 15, "tokens": 767, "cost_usd": 0.000038, "latency_sec": 0.58 },
    "compression_ratio": 99.0,
    "tokens_saved": 76037,
    "cost_saved_usd": 0.003802
  },
  "graph": {
    "nodes": [
      { "id": "1", "label": "192.168.1.105", "type": "IP ADDRESS" },
      { "id": "2", "label": "admin", "type": "USER" },
      { "id": "3", "label": "root", "type": "USER" }
    ],
    "edges": [
      { "source": "1", "target": "2", "relationship": "Failed SSH Brute Force" },
      { "source": "2", "target": "3", "relationship": "Root escalation using sudo" }
    ]
  },
  "timeline": [
    { "timestamp": "2026-09-20T13:04:12Z", "title": "Initial Access", "severity": "HIGH" },
    { "timestamp": "2026-09-20T13:07:10Z", "title": "Privilege Escalation", "severity": "CRITICAL" }
  ]
}
```

### Endpoint 3: Pre-loaded Demo Investigation Reset
- **`POST /api/v1/demo/load-scenario`**
- **Query Params**: `scenario_name=apt29_ssh_bruteforce | apache_webshell | windows_mimikatz`
- **Response `200 OK`**: Loads full pre-indexed log dataset instantly for live demonstration.
