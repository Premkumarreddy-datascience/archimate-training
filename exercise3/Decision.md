# Exercise 3 – Decision Document

## Problem Statement

A business unit (Water) has ~8000 SharePoint documents (SOPs, safety data sheets, playbooks, training material). Frontline colleagues cannot find answers via SharePoint search alone.

We need a Teams bot that:
- Authenticates with Entra ID SSO (no separate login)
- Honors SharePoint permissions (never show inaccessible content)
- Answers natural‑language questions with citations
- Supports multi‑turn follow‑ups
- Runs inside the Ecolab Azure tenant

Two designs were considered:

- **Variant A**: Azure‑native hand‑built (Bot Framework + Azure AI Search + Azure OpenAI + Cosmos DB)
- **Variant B**: Microsoft Copilot Studio declarative agent grounded on SharePoint via Graph connectors

---

## Options Considered

| Variant | Core Idea | Strengths | Weaknesses |
|---------|-----------|-----------|-------------|
| A – Azure‑native | Full control over RAG pipeline, ACL trimming, and observability | High permission fidelity, EU data residency, extensible | Longer build time, requires skilled engineers |
| B – Copilot Studio | Low‑code declarative agent, fastest launch | Very fast (2 weeks), low skills required | Vendor lock‑in, limited observability, residency risk |

---

## Winner: **Variant A – Azure‑native hand‑built**

### Scored Trade‑off Axes

| Axis | Variant A | Variant B | Rationale |
|------|-----------|-----------|-----------|
| Time‑to‑first-user | Medium (4‑6 weeks) | **High (2 weeks)** | B is faster, but A is acceptable for a critical system. |
| Permission fidelity | **High** | Medium | A implements exact ACL filter; B relies on Graph connector black box. |
| Data residency | **High** | Low/Medium | A uses EU North, private endpoints. B depends on Microsoft’s EU boundary – not fully guaranteed for LLM. |
| Cost per active user/month (€) | Medium (0.15‑0.30) | Medium (0.20‑0.40) | Similar at 5k users, but A scales cheaper at high volume. |
| Extensibility | **High** | Low | A can add any tool (SAP, Snowflake). B limited to Power Platform connectors. |
| Vendor lock‑in | Medium | **High** | A components are replaceable. B requires complete rewrite to migrate. |
| Observability | **High** | Low | A has Application Insights traces. B only audit logs. |
| Skills required | High | Low (citizen dev) | Ecolab has Azure skills, so A is feasible. |

### Trade‑offs Accepted

- **Longer initial development** (4–6 weeks) in exchange for full permission control and EU data residency.
- **Higher operational ownership** – we must monitor search index freshness, OpenAI quotas, and conversation storage.
- **Requires senior Azure engineer** – but the existing team has that expertise.

### What Would Change Your Mind?

- If **time‑to‑first‑user < 2 weeks** were mandatory **and** legal twist dropped → consider Variant B.
- If **Microsoft contractually guarantees** EU‑only processing for Copilot Studio LLM and provides granular ACL logs → re‑evaluate.
- If **budget cap €5k/month** were absolute and user count >40k – A still wins because B’s per‑user license would exceed cap earlier.

### Mid‑Sprint Twists (How Winner Holds)

| Twist | Does A survive? | Notes |
|-------|----------------|-------|
| No content may leave EU | ✅ Yes | Already in EU North. |
| Scale 500 → 40k users | ⚠️ Breaks at OpenAI TPM | Mitigation: quota increase + caching. |
| Custom internal API tool | ✅ Open | Pluggable adapter in orchestrator. |
| Multi‑document comparison | ✅ Works | RAG with larger chunks; can add agentic retrieval. |
| Budget €5k/month | ✅ At 5k users (~€1500). At 40k exceeds | Mitigation: rate limits, cheaper embeddings. |

---

## Final Decision

**Implement Variant A (Azure‑native hand‑built)** with EU North deployment, private endpoints, Application Insights monitoring, and weekly index refresh from SharePoint.

The marginal extra development time is justified by the need for strict permission enforcement, data residency compliance, and future extensibility for custom tools.
