# NHI Remediation AI Agent System

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4)](https://dotnet.microsoft.com/)
[![React](https://img.shields.io/badge/React-18-61DAFB)](https://reactjs.org/)

> Autonomous multi-agent system for investigating and remediating orphaned/stale Non-Human Identity accounts across enterprise

## 🎯 Overview

**Problem:** IAM teams manually investigate 2,000+ service/generic accounts to keep the organization compliant. With growing AI agents in the company adds overhead on top of this.
**Solution:** AI agents automate 80% of investigations and the system can be handy with 'n' number of increase of AI agents in the future.  
**Impact:** 99% cost reduction • 95% time savings 

---

## 🏗️ System Architecture
```
┌──────────────────────────────────────────────────────┐
│         WEB INTERFACE (React + .NET 8)               │
│  Select Application → Review → Approve/Reject        │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│              COORDINATOR AGENT                       │
│  • Batch Processing: 500 accounts → 5×100            │
│  • Event-Driven Orchestration (Service Bus)          │
└──────────────────────────────────────────────────────┘
        ↓              ↓              ↓
   ┌────────┐    ┌────────┐    ┌────────┐
   │AGENT A │ →  │AGENT B │ →  │AGENT C │
   │Enrich  │    │Risk    │    │Outreach│
   └────────┘    └────────┘    └────────┘
        ↓              ↓              ↓
   ┌─────────────────────────────────────┐
   │    Azure SQL + Service Bus + SIEM   │
   └─────────────────────────────────────┘
```

---

## 📋 Requirements

### Functional

#### Agent A: Enrichment Agent
- **Task:** Determine account activity & ownership
- **Process:**
  - Categorize: Orphaned (owner left) vs Stale (appears unused)
  - Check activity: PAM → SIEM → Direct Query → SailPoint (tiered fallback)
  - Output: Activity evidence + ownership context
- **LLM:** Phi-4 (local) OR GPT-4o (cloud)

#### Agent B: Risk Evaluation Agent  
- **Task:** Assess risk using compliance policies
- **Process:**
  - RAG: Query Vector Store (HIPAA, SOX, GDPR policies)
  - LLM reasoning: Risk score + recommended action
  - Output: Disable / Transfer / Keep / Escalate
- **LLM:** GPT-4o/Claude (complex reasoning)

#### Agent C: Stakeholder Outreach Agent
- **Task:** Contact stakeholders, gather approvals
- **Process:**
  - Craft personalized Teams/Slack messages
  - Track responses, handle escalations (7-day timeout)
  - Update web interface with approval status
- **LLM:** Phi-4 (local) OR GPT-4o

#### Workflow
- **Batch Processing:** 100 accounts/batch, 5 parallel instances
- **Partial Success:** Failed accounts retry 3× (exponential backoff: 30s, 60s, 120s)
- **DLQ:** Permanently failed → Dead Letter Queue → Manual review
- **IGA Integration:** SailPoint workflow → SNOW ticket (audit trail)

### Non-Functional

| Requirement | Implementation |
|-------------|----------------|
| **Consistency** | CAP theorem: Consistency > Availability (ACID via Azure SQL) |
| **Scale** | 1,000+ concurrent investigations, horizontal auto-scaling |
| **Failover** | Exponential backoff retry, circuit breakers |
| **Security** | Managed identities (no secrets), PII redaction before LLM |
| **RBAC** | Read-only vs Elevated permissions |
| **Observability** | Application Insights, health probes |
| **Concurrency** | Optimistic locking (version field in DB) |
| **Orchestration** | Event-driven pub/sub (Azure Service Bus) |

---

## 📊 Data Sources

### Required (Always Available)
- **SailPoint IGA:** Account inventory (237 connectors)
- **HR System:** Employee status, termination dates

### Optional (Tiered Fallback)

| Source | Coverage | Confidence | Use Case |
|--------|----------|------------|----------|
| **PAM**  | 10-20% | ⭐⭐⭐⭐⭐ | Privileged accounts |
| **SIEM** (Splunk/Sentinel) | 40-50% | ⭐⭐⭐⭐ | AD, Entra, UNIX logs |
| **Direct Query** (SSH/PowerShell) | Variable | ⭐⭐⭐⭐ | Critical accounts |
| **SailPoint + Context** | 40-50% | ⭐⭐⭐ | Legacy apps, trust IGA data |

### Endpoint Support
- **Windows:** AD, Entra ID (EventID 4624, 4768, 4769)
- **UNIX/Linux:** SSH logs, lastlog, /var/log/auth.log
- **Applications:** SAP, Workday, Oracle, custom apps (via SailPoint)
- **Cloud:** AWS IAM, Azure IAM, GCP IAM (CloudTrail, Activity Logs)

---

## 🔄 Data Flow
```
1. IAM team: "Investigate SAP - 500 accounts"
                    ↓
2. Coordinator: Divide into 5 batches × 100
                    ↓
3. Agent A (5 instances): Enrich accounts
   - Check PAM/SIEM/Direct/SailPoint
   - Categorize: Orphaned vs Stale
   - 93 succeed, 7 fail → Retry queue
                    ↓
4. Agent A: Publish "agent_a_complete" events (×5)
                    ↓
5. Coordinator: Verify 465 complete, 35 in retry
   - Retry failed accounts (3× with backoff)
   - 2 accounts → DLQ (permanent failure)
                    ↓
6. Coordinator: Trigger Agent B with 493 accounts
                    ↓
7. Agent B (5 instances): Risk evaluation
   - RAG: Query Vector Store for policies
   - LLM: Risk score + recommendation
                    ↓
8. Agent B: Publish "agent_b_complete" events
                    ↓
9. Coordinator: Trigger Agent C
                    ↓
10. Agent C (5 instances): Stakeholder outreach
    - Send Teams messages with findings
    - Track approvals: 350 approved, 80 pending
                    ↓
11. After 7 days: Escalate 80 pending to managers
                    ↓
12. IAM team: Approve 350 → Trigger IGA workflow
                    ↓
13. SailPoint: Disable accounts + Create SNOW tickets
```

---

## 🛠️ Technology Stack

| Layer | Technology |
|-------|-----------|
| **Orchestration** | Microsoft Agent Framework (A2A protocol) |
| **Backend** | C# .NET 8 |
| **Frontend** | React 18 + Tailwind CSS |
| **LLM** | Phi-4 (local), GPT-4o, Claude Sonnet |
| **Vector Store** | Azure AI Search (RAG) |
| **State** | Azure SQL (Serverless) |
| **Events** | Azure Service Bus |
| **Compute** | Azure Kubernetes Service (Spot instances) |
| **LLM Compute** | Azure Container Instances (GPU, on-demand) |
| **Monitoring** | Application Insights |

---

## 💰 Cost Analysis

### Monthly Cost (2,000 accounts)

| Component | Cost |
|-----------|------|
| **AKS (Spot Instances)** | $315 |
| **Azure SQL (Serverless)** | $150 |
| **Agent A (Phi-4 on-demand)** | $16 |
| **Agent B (GPT-4o API)** | $85 |
| **Agent C (Phi-4 on-demand)** | $8 |
| **Vector Store (Azure AI Search)** | $75 |
| **Service Bus** | $10 |
| **Storage & Monitoring** | $5 |
| **Total** | **$664/month** |


---

## 📄 License

MIT License - See [LICENSE](LICENSE)

---

**Author:** Nikesh  
**Last Updated:** February 4, 2026  
**Version:** 1.0.0

