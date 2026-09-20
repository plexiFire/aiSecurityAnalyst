# Securigation Product & UI Specification

## 1. Product Requirements & Vision

### Problem Statement
Security Operations Center (SOC) analysts spend hours sifting through gigabytes of raw log files to piece together incident timelines. Feeding full log files into Large Language Models (LLMs) causes token window overflow, slow response times (8-15 seconds), and extreme API costs.

### Solution Overview
Securigation provides an AI SOC Command Dashboard powered by **Paritok Context Optimization**. By compressing raw candidate logs by 85–98% before submitting them to the Groq Llama-3.3-70B model, Securigation delivers instant (<1 second), evidence-backed incident answers with verified attack graphs and timelines.

### Functional Core Requirements
1. **Multi-Format Log Support**: Upload and parse Apache Access logs, Linux Syslog (`/var/log/auth.log`), Windows Security EVTX dumps, AWS CloudTrail JSON, and Palo Alto Firewall CSV.
2. **Paritok Context Compression**: Intercept and optimize log evidence packs, generating real-time telemetry metrics (Events, Tokens, Cost, Latency).
3. **Interactive Graph Visualization**: Render interactive node-edge diagrams mapping IPs, Users, Hosts, Files, and C2 endpoints.
4. **Replay Trace Modal**: Animated step-by-step playback showing candidate retrieval, Paritok token compression, and LLM grounding.
5. **1-Click Demo Scenarios**: Pre-load real-world security scenarios in under 2 seconds:
   - **Scenario A**: APT29 SSH Brute Force & Sudo Root Escalation
   - **Scenario B**: Apache WebShell Exploit & Command Execution
   - **Scenario C**: Windows Remote Desktop & Mimikatz LSASS Dump

---

## 2. SOC Command UI/UX Design System

### Design Language & Color Palette
Custom dark mode theme engineered for security command centers:
- **Background Main**: `#090d16` (Deep Charcoal Black)
- **Container Panel**: `#121824` / `#1a2234` (Glassmorphic Dark Panels)
- **Primary Cyber Cyan**: `#00f5c8` (Paritok Highlights & Active Buttons)
- **Threat Critical Red**: `#ff3b5c` (High Severity Alerts & Attacks)
- **Warning Amber**: `#ffb800` (Suspicious Activity)
- **Muted Text**: `#94a3b8` (System Metadata & Telemetry Labels)

### Layout & Component Breakdown

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Header: Securigation SOC Command | Live Telemetry Status | Demo Reset Button    │
├───────────────────────────────────────┬──────────────────────────────────────────┤
│ Left Column: Log Upload & Dataset     │ Right Column: Grounded Investigation     │
│ - Upload Dropzone (EVTX, Syslog, JSON)│ - Paritok Telemetry Hero Card            │
│ - Pre-loaded Incident Scenarios       │ - Grounded AI Reasoning Output           │
│ - Raw Log Stream Preview              │ - Attack Timeline Chronology             │
│                                       │ - Interactive Incident Entity Graph      │
├───────────────────────────────────────┴──────────────────────────────────────────┤
│ Bottom Bar: Multi-turn Chat Console & Query Input                                │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Demonstration & Presentation Walkthrough

### 3-Minute Live Presentation Flow

#### Phase 1: Problem Setup & Pre-loaded Incident (0:00 - 0:45)
- Launch the SOC Command Workspace.
- Click **"Load APT29 SSH Scenario"**. Show the raw log dataset containing 1,500 authentication events.
- Highlight the problem: *Standard LLMs struggle with 80,000+ token payloads, causing delay and budget drain.*

#### Phase 2: Query Execution & Paritok Telemetry (0:45 - 1:45)
- Submit query: *"Show all malicious SSH login spikes and root escalation commands."*
- Watch the **Paritok Context Compression Pipeline** execute in real time.
- Focus attention on the **Paritok Telemetry Hero Card**:
  - Raw Tokens: **76,804** ➔ Optimized Tokens: **767** (99% Token Reduction)
  - Raw Cost: **$0.0038** ➔ Optimized Cost: **$0.000038**
  - Latency: **8.18s** ➔ **0.58s** (<1 Second Response Time)

#### Phase 3: Evidence Grounding & Attack Graph (1:45 - 2:30)
- Highlight the **Evidence Panel**: Every claim in the LLM answer directly cites raw log event IDs.
- Interact with the **Incident Entity Graph**: Click on `192.168.1.105` to highlight connected compromised accounts (`admin` ➔ `root`).
- Open the **Replay Modal**: Play the step-by-step animation showing candidate log filtering and Paritok context optimization.
