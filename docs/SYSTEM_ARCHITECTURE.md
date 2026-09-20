# Securigation System Architecture & Technical Design

## 1. Executive Summary & Core Architectural Principles

Securigation is an enterprise-grade AI Security Incident Response & Log Investigation Platform engineered to process massive, heterogeneous security log streams (Apache, Linux Syslog, Windows EVTX, AWS CloudTrail, Palo Alto Firewall).

The central technical innovation is the **Paritok Context Optimization Layer**, a high-density evidence compression engine placed directly between Candidate Search Retrieval and the Reasoning LLM (Groq Llama-3.3-70B).

```
[Raw Log Ingestion] ─► [Elastic Search / In-Memory Index] ─► [Candidate Evidence Pack (100k+ Tokens)]
                                                                          │
                                                                          ▼
[Grounded Response UI] ◄─ [Groq Llama-3.3-70B] ◄─ [Compressed Context (<2k Tokens)] ◄─ [Paritok Engine]
```

### Core Architecture Principles
1. **Mandatory Context Compression**: No candidate log payload reaches the LLM without passing through Paritok optimization.
2. **Deterministic Grounding**: Every answer, attack timeline item, and entity relationship in the graph is strictly bound to evidence log event IDs.
3. **Sub-Second Investigation Latency**: Context payloads are reduced by 85% to 98%, cutting LLM reasoning latency from ~8-12 seconds down to <1 second.
4. **Decoupled Architecture**: Clean separation between ingestion parsers, vector/structured retrieval, compression layer, reasoning engine, and SOC UI.

---

## 2. End-to-End System Components

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            Next.js 14 SOC Command Dashboard                      │
│   (Telemetry Cards | Interactive Node Graph | Replay Modal | Grounded Console)   │
└────────────────────────────────────────┬─────────────────────────────────────────┘
                                         │ REST API / WebSocket
┌────────────────────────────────────────▼─────────────────────────────────────────┐
│                            FastAPI Backend Application                           │
├──────────────────────────────────────────────────────────────────────────────────┤
│ 1. Log Ingestion & Normalizer  │ Multithreaded Regex & Structured Parsers        │
│ 2. Evidence Retrieval Engine   │ Ranked Hybrid Search (IP, User, Time Range)      │
│ 3. Paritok Optimization Client │ Micro-chunk Token Compression API               │
│ 4. Groq Reasoning Engine       │ Multi-turn SOC Investigation Prompt Orchestrator│
└────────────────────────────────────────┬─────────────────────────────────────────┘
                                         │
               ┌─────────────────────────┴─────────────────────────┐
               ▼                                                   ▼
┌──────────────────────────────┐                   ┌──────────────────────────────┐
│  Elasticsearch / Local Index │                   │      Paritok External API    │
└──────────────────────────────┘                   └──────────────────────────────┘
```

### Backend Layer (FastAPI & Python 3.13)
- **Log Ingestion Engine**: Normalizes multi-format log feeds into uniform `SecurityEvent` JSON schemas with standardized fields (`timestamp`, `source_ip`, `user`, `event_type`, `severity`, `raw_log`).
- **Hybrid Search & Candidate Fetcher**: Performs candidate event retrieval matching query terms, target IPs, users, and timestamp windows.
- **Paritok Client (`paritok_client.py`)**: Intercepts retrieved log candidate arrays, executes evidence-preserving compression, and outputs token/cost telemetry metrics.
- **Groq Reasoning Engine (`groq_reasoning.py`)**: Submits the compressed evidence pack to `llama-3.3-70b-versatile` with structured system prompts enforcing strict JSON responses.

### Frontend Layer (Next.js 14, React 18, Vanilla CSS)
- **SOC Command Workspace**: Dark mode interface designed for security analysts.
- **Paritok Telemetry Hero Card**: Live telemetry showing raw vs optimized token count, cost reduction, latency savings, and compression ratio.
- **Interactive Incident Graph**: SVG/Canvas node-edge graph visualizing attack paths (IP -> Host -> Compromised User -> File/C2).
- **Replay Modal**: Step-by-step trace playback showing how candidate logs were filtered, compressed by Paritok, and turned into grounded findings.

---

## 3. Architecture Decision Records (ADR)

### ADR-001: Paritok Context Optimization Layer as Mandatory Middleware
- **Context**: Raw security log searches yield 50,000 to 100,000+ tokens per incident query. Sending raw logs directly to LLMs causes token limit crashes, extreme latency (>10s), and high costs ($0.05+ per query).
- **Decision**: Insert Paritok as mandatory context compression middleware.
- **Consequences**: Payload size reduced by 85–98% with zero loss of critical threat indicators (IPs, hashes, commands). Query latency drops to <1s.

### ADR-002: Deterministic Keyword/Timestamp Indexing Over Pure Vector RAG
- **Context**: Security investigations require exact matching on IP addresses, port numbers, usernames, and timestamp ranges. Pure vector embeddings frequently miss exact IP matches or numerical port filters.
- **Decision**: Use a hybrid deterministic search index (exact string matching + date range filter) fed into Paritok context compression.
- **Consequences**: Guarantees 100% recall for exact IPs and security event IDs before compression.

### ADR-003: FastAPI + Pydantic v2 Backend with Async Pipeline
- **Context**: Modern SOC dashboards require high-throughput ingestion and non-blocking streaming.
- **Decision**: Build the core backend in FastAPI with Pydantic v2 schemas and async HTTP clients for Paritok and Groq APIs.

---

## 4. Scalability, Performance & Security

### Performance Specifications
| Operation | Target Latency | Throughput / Efficiency |
| :--- | :--- | :--- |
| Log Parsing & Normalization | <50ms per 10k lines | 100,000 lines/sec multithreaded |
| Candidate Retrieval | <100ms | Sub-100ms across 1M+ event index |
| Paritok Context Compression | <300ms | 85% – 98% token payload reduction |
| LLM Evidence Generation | <800ms | Structured JSON output via Groq Llama-3.3 |
| End-to-End Query Response | <1.5 seconds | Total pipeline turn-around |

### Security & Compliance Design
1. **API Key Isolation**: Server-side storage of `PARITOK_API_KEY` and `GROQ_API_KEY`. Keys are never exposed to client browsers.
2. **Local Data Privacy**: Raw logs are indexed locally or within isolated tenant storage before compression.
3. **Input Sanitization**: Query inputs are sanitized against prompt injection attempts aimed at bypassing evidence grounding rules.
