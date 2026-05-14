# Variant B – Microsoft Copilot Studio / Declarative Agent with SharePoint Grounding

## Overview

This variant uses **Microsoft Copilot Studio** (Power Platform) to create a declarative agent that is grounded on SharePoint via Graph connectors and published directly to Teams. It is the fastest way to launch, requires minimal code, but introduces higher vendor lock‑in and less control over retrieval and permissions.

- **Chosen business unit**: Water  
- **Core technologies**: Copilot Studio declarative agent, Graph connector for SharePoint, Microsoft Search, Teams, Entra ID  
- **Deployment region**: EU data boundary (configurable) – relies on Microsoft’s EU commitments  

---

## Motivation Layer (Lean)

| Element | Type | Description |
|---------|------|-------------|
| Legal & compliance | Stakeholder | Enforces data residency and SharePoint permission rules |
| IT operations (Ecolab) | Stakeholder | Owns Microsoft 365 tenant and Copilot Studio configuration |
| Reduce search friction | Driver | SharePoint search fails for natural language queries |
| Honour existing permissions | Driver | Users must never see content they cannot open |
| Multi‑turn conversation | Goal | Bot supports follow‑up questions |
| Citations required | Goal | Every answer includes link to source document |
| Fast time‑to‑pilot | Goal | Pilot group (500 users) can chat within 2 weeks |
| Entra ID SSO | Requirement | No separate login (inherited from Teams) |
| SharePoint permission enforcement | Requirement | ACLs honoured via Graph connector security trimming |
| Natural language Q&A | Requirement | Bot answers over 8000 documents |
| Teams integration | Requirement | Bot runs inside Microsoft Teams (published as Copilot Studio agent) |
| Multi‑turn support | Requirement | Agent retains context across follow‑ups (built into Copilot Studio) |

---

## Business Layer

| Element | Type | Description |
|---------|------|-------------|
| Water frontline colleague | Business actor | Asks questions via Teams |
| Water shift supervisor | Business actor | Reviews answers, provides feedback |
| Water safety officer | Business actor | Uses bot for safety data sheets |
| Answer service | Business service | Agent provides answers to natural language questions |
| Citation service | Business service | Agent provides source links for every answer |

---

## Application Cooperation View

### Components

| Component | Type | Description |
|-----------|------|-------------|
| Copilot Studio agent | Application component | Declarative agent grounded on SharePoint, published to Teams |
| Graph connector for SharePoint | Application service | Ingests SharePoint documents into Microsoft Graph index |
| Microsoft Search | Application service | Retrieves content with ACL‑aware security trimming |
| Grounding service | Application service | Connects retrieved content to LLM prompt |
| Citation mapper | Application service | Maps answers back to source document URLs |
| Agent conversation state | Data object | Built‑in multi‑turn state managed by Copilot Studio |

### External Applications

| Name | Type | Description |
|------|------|-------------|
| Teams channel | External application | User interface for the agent |
| SharePoint Online | External application | Source documents (Water BU libraries) |
| Entra ID | External application | SSO (inherited from Teams session) |

### Interfaces

| Interface | Description |
|-----------|-------------|
| Chat interface | Teams messaging endpoint for the agent (provided by Copilot Studio) |

### Data Flow (User Query)

1. User asks question in Teams → Chat interface → Copilot Studio agent  
2. Agent (via grounding) calls Microsoft Search with user’s identity – Microsoft Graph automatically applies SharePoint ACLs  
3. Microsoft Search returns relevant chunks from Graph index (already security‑trimmed)  
4. Grounding service sends prompt (chunks + conversation history) to underlying LLM (Microsoft‑hosted)  
5. Citation mapper traces each answer sentence back to source document URL  
6. Agent returns answer + citations to Teams  
7. Conversation state is automatically maintained by Copilot Studio for multi‑turn  

---

## Technology Usage View

| Node | Technology | Region | Key Configuration |
|------|------------|--------|-------------------|
| Microsoft Copilot Studio | SaaS (Power Platform) | EU data boundary (configurable) | Declarative agent, no code |
| Microsoft Graph | Graph API + Search | EU data boundary | Indexes SharePoint content via connector |
| SharePoint Online tenant (Ecolab) | SharePoint in M365 | EU | Water BU document libraries |
| Microsoft Teams | Teams client + Bot framework | Global (EU data boundary can be enforced) | Channel for agent |
| Entra ID tenant (Ecolab) | Microsoft Entra ID | EU | SSO, inherits user identity |
| Power Platform admin center | Management portal | EU | Monitoring, audit logs, configuration |
| Data boundary (EU) | Microsoft 365 configuration | EU | Ensures data processing stays within EU |
| Audit logs | Power Platform analytics | EU | Activity logs, usage metrics |

---

## Physical / Infrastructure Layer

| Node | Description |
|------|-------------|
| Microsoft EU data centers | Physical locations where Microsoft processes data (configurable to EU) |
| Ecolab Microsoft 365 tenant | The specific tenant owned by Ecolab |

---

## Relationships Summary (Key Flows)

| Source | Target | Type |
|--------|--------|------|
| Copilot Studio agent | Microsoft Search | Flow (grounding) |
| Microsoft Search | Graph connector for SharePoint | Realization |
| Graph connector for SharePoint | SharePoint Online | Flow (ingest) |
| Copilot Studio agent | Grounding service | Realization |
| Grounding service | Citation mapper | Composition |
| Grounding service | Agent conversation state | Access |
| Microsoft Search | Entra ID | Flow (security trimming) |
| Microsoft Copilot Studio | Copilot Studio agent | Realization |
| Microsoft Graph | Microsoft Search | Realization |
| Microsoft EU data centers | Microsoft Copilot Studio | Composition |

---

## Trade-offs (Scored)

| Axis | Score | Justification |
|------|-------|---------------|
| Time‑to‑first-user | **High** (2 weeks) | Declarative agent with no custom code; publish in days |
| Permission fidelity | Medium | Graph connector respects SharePoint ACLs at index time, but retrieval is a black box – cannot add custom filters |
| Data residency | Low/Medium | Relies on Microsoft’s EU data boundary – LLM processing may still leave EU unless explicitly guaranteed |
| Cost per active user/month | Medium (€0.20‑0.40) | Per‑user license + consumption; scales linearly with user count |
| Extensibility | Low | Only Power Platform connectors / custom actions; no fine‑grained orchestration |
| Vendor lock‑in | **High** | Entire bot logic tied to Copilot Studio – migration requires complete rewrite |
| Observability | Low | Only high‑level audit logs; cannot debug “why bad answer” at prompt level |
| Skills required | Low (citizen developer) | Power Platform admin + no coding needed |

---

## Resilience to Mid‑Sprint Twists (Not Designed Up‑Front, but Assessed)

| Twist | Does Variant B survive? | Explanation |
|-------|------------------------|-------------|
| No content may leave EU tenant | ⚠️ Risky | Microsoft’s EU data boundary for Copilot Studio is not fully guaranteed for LLM processing. Legal team would need contractual assurance. |
| Rollout 500 → 40 000 users | ⚠️ Breaks first on license cost | Per‑user licensing becomes very expensive at 40k users. Also, Microsoft Graph search may throttle. |
| Custom tool calling internal API | ❌ Limited | Only possible via Power Automate or custom connector – no native orchestration. Complex multi‑step tools are hard. |
| Multi‑document comparison | ⚠️ Possible but limited | RAG works, but agent cannot easily chain multiple retrieval steps or do agentic comparison without custom code. |
| Budget cap €5k/month | ⚠️ Exceeds at 40k users | At 5000 users (~€2000) it’s fine. At 40k users (~€16,000) it blows the cap. |

---

## Summary

**Variant B** is the fastest path to a working bot, ideal for rapid prototyping or when development resources are scarce. It honours SharePoint permissions reasonably well and integrates seamlessly with Teams. However, it suffers from high vendor lock‑in, limited observability, and uncertain data residency guarantees. It is best suited for internal, non‑critical use cases where time‑to‑market is the top priority and compliance requirements are relaxed.
