# Variant A – Azure-Native Hand-Built Teams Bot with Entra ID SSO and RAG over SharePoint

## Overview

This variant uses **Azure-native services** to build a custom Teams bot from scratch. It provides maximum control over permission enforcement, data residency, and extensibility, at the cost of higher development effort and operational ownership.

- **Chosen business unit**: Water  
- **Core technologies**: Bot Framework SDK, Azure App Service, Azure AI Search (with SharePoint indexer + ACL trimming), Azure OpenAI (GPT‑4o), Cosmos DB, Entra ID  
- **Deployment region**: EU North (e.g., Sweden Central) – satisfies EU data residency requirement  

---

## Motivation Layer (Lean)

| Element | Type | Description |
|---------|------|-------------|
| Legal & compliance | Stakeholder | Enforces data residency and SharePoint permission rules |
| IT operations (Ecolab) | Stakeholder | Owns Azure tenant and deployment |
| Reduce search friction | Driver | SharePoint search fails for natural language queries |
| Honour existing permissions | Driver | Users must never see content they cannot open |
| Multi‑turn conversation | Goal | Bot supports follow‑up questions |
| Citations required | Goal | Every answer includes link to source document |
| Fast time‑to‑pilot | Goal | Pilot group (500 users) can chat within 4 weeks |
| Entra ID SSO | Requirement | No separate login |
| SharePoint permission enforcement | Requirement | ACLs honoured at retrieval time |
| Natural language Q&A | Requirement | Bot answers over 8000 documents (SOPs, safety sheets, playbooks, training) |
| Teams integration | Requirement | Bot runs inside Microsoft Teams |
| Multi‑turn support | Requirement | Bot retains context across follow‑ups |

---

## Business Layer

| Element | Type | Description |
|---------|------|-------------|
| Water frontline colleague | Business actor | Asks questions via Teams |
| Water shift supervisor | Business actor | Reviews answers, provides feedback |
| Water safety officer | Business actor | Uses bot for safety data sheets |
| Answer service | Business service | Bot provides answers to natural language questions |
| Citation service | Business service | Bot provides source links for every answer |

---

## Application Cooperation View

### Components

| Component | Type | Description |
|-----------|------|-------------|
| Teams Bot | Application component | Handles messages, SSO token exchange, turn state |
| RAG Orchestrator | Application component | Coordinates retrieval, prompt construction, citation mapping |
| Azure AI Search | Application component | Index of SharePoint documents with ACL trimming |
| Azure OpenAI | Application component | LLM for answer generation (GPT‑4o) |
| Cosmos DB | Application component | Stores conversation history |
| Conversation state | Data object | JSON document for multi‑turn context |
| ACL trimming service | Application service | Fetches user’s SharePoint groups and applies filter |
| Indexer with ACL pull | Application service | Indexer that reads documents + permissions from SharePoint |
| OAuth2 OBO flow | Application service | Token exchange provided by Entra ID |

### External Applications

| Name | Type | Description |
|------|------|-------------|
| SharePoint Online | External application | Source documents (Water BU libraries) |
| Entra ID | External application | OAuth2 / on‑behalf‑of token service |

### Interfaces

| Interface | Description |
|-----------|-------------|
| Chat interface | Teams messaging endpoint |
| Search interface | Azure AI Search REST API |
| Completion interface | Azure OpenAI API |

### Data Flow (User Query)

1. User sends message via Teams → Chat interface → Teams Bot  
2. Bot calls Entra ID → OAuth2 OBO flow → exchanges user token for bot token  
3. Bot calls SharePoint API → ACL trimming service → fetches user’s group memberships  
4. Bot calls Azure AI Search → Search interface → includes groups as security filter  
5. Search returns relevant chunks (already trimmed by ACL)  
6. Bot loads conversation history from Cosmos DB → Conversation state  
7. Bot calls Azure OpenAI → Completion interface → prompt = chunks + history  
8. Bot saves updated conversation state to Cosmos DB  
9. Bot returns answer + citations to Teams  

---

## Technology Usage View

| Node | Technology | Region | Key Configuration |
|------|------------|--------|-------------------|
| Azure App Service | .NET 8 / Python | EU North | Hosts Teams Bot and RAG Orchestrator |
| Azure AI Search service | Standard tier + semantic ranker | EU North | Indexer connects to SharePoint via private endpoint |
| Azure OpenAI service | GPT‑4o (or GPT‑35‑turbo) | EU North (Sweden Central) | Private endpoint, assigned quota |
| Cosmos DB account | NoSQL Core API | EU North | Provisioned throughput, private endpoint |
| Entra ID tenant (Ecolab) | Microsoft Entra ID | EU (Microsoft cloud) | OAuth 2.0, OBO flow enabled |
| SharePoint Online tenant (Ecolab) | SharePoint in M365 | EU | Water BU document libraries, private network integration |
| Private endpoint | Azure Private Link | EU North | Secures traffic to PaaS services |
| Virtual Network | Azure VNet | EU North | Isolated network for all Azure resources |
| Application Insights | Azure Monitor | EU North | Logging, traces, performance metrics |

---

## Physical / Infrastructure Layer

| Node | Description |
|------|-------------|
| Azure EU North region | All Azure services deployed here (App Service, AI Search, OpenAI, Cosmos DB) |
| Ecolab Azure tenant | The specific Azure tenant owned by Ecolab |
| Microsoft backbone network | Internal network connecting EU regions, carries VNet traffic |

---

## Relationships Summary (Key Flows)

| Source | Target | Type |
|--------|--------|------|
| Teams Bot | RAG Orchestrator | Composition |
| RAG Orchestrator | Azure AI Search | Flow (query with ACL filter) |
| RAG Orchestrator | Azure OpenAI | Flow (prompt + chunks) |
| RAG Orchestrator | Cosmos DB | Flow (read/write state) |
| Azure AI Search | SharePoint Online | Flow (indexer pulls docs + ACLs) |
| Teams Bot | Entra ID | Flow (OBO token exchange) |
| ACL trimming service | SharePoint Online | Flow (fetch user groups) |
| Azure App Service | Teams Bot | Realization |
| Azure EU North region | Azure App Service | Composition |

---

## Trade-offs (Scored)

| Axis | Score | Justification |
|------|-------|---------------|
| Time‑to‑first-user | Medium (4‑6 weeks) | Requires building RAG pipeline, but no external dependencies |
| Permission fidelity | **High** | Exact ACL enforcement via group claims + search filter |
| Data residency | **High** | All components in EU North, private endpoints, no outbound data |
| Cost per active user/month | Medium (€0.15‑0.30) | Assumes 10 queries/user/day, 5000 users → ~€1500/month |
| Extensibility | **High** | Can add any tool (SAP, Snowflake) as new orchestrator step |
| Vendor lock‑in | Medium | Replaceable pieces: OpenAI → self‑hosted, Search → Vespa |
| Observability | **High** | Application Insights provides end‑to‑end traces |
| Skills required | High | Needs Azure developer + AI Search expertise |

---

## Resilience to Mid‑Sprint Twists (Not Designed Up‑Front, but Assessed)

| Twist | Does Variant A survive? | Explanation |
|-------|------------------------|-------------|
| No content may leave EU tenant | ✅ Yes | Already deployed in EU North. Private endpoints ensure data stays inside region. |
| Rollout 500 → 40 000 users | ⚠️ Breaks first at OpenAI TPM | Mitigation: request quota increase, add semantic caching, scale search replicas. |
| Custom tool calling internal API | ✅ Open | Design includes pluggable adapter; orchestrator can call any HTTP API. |
| Multi‑document comparison | ✅ Works | Basic RAG works by retrieving more chunks; can extend to agentic retrieval. |
| Budget cap €5k/month | ✅ Survives at 5k users (~€1500). At 40k users exceeds cap | Mitigation: rate limiting, cheaper embedding model, or restrict usage. |

---

## Summary

**Variant A** offers full control, high permission fidelity, and strong data residency guarantees. It requires significant development effort and operational expertise but scales well and remains extensible. Suitable for organisations with existing Azure skills and strict compliance needs.
