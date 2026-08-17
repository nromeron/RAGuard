# D2 — Target Architecture and MVP Scope

**Project:** RAGuard
**Status:** Draft v2 — architecture + scope table (ADRs and CI/CD pipeline design pending)

---

## 1. Target Architecture

RAGuard is organized into the four layers defined in the project brief:

1. **Entry & security** — the boundary between the public internet and the system: Nginx (TLS
   termination, IP allowlisting), rate limiting, and inbound PII redaction.
2. **Cloud-native services** — a Pipeline Orchestrator coordinating the RAG pipeline and its
   guardrails: retrieval, a sufficiency check, a legacy-data-necessity check, a legacy data client,
   prompt-injection defense, generation, output validation, the storage/cache layer, and an offline
   ingestion pipeline that populates the corpus before the API serves traffic.
3. **Legacy abstraction** — the layer that decouples RAGuard from the banking core: the Legacy Data
   Client and the Legacy Mock Service that stands in for the real core in this MVP.
4. **Legacy core** — the real OmniBank banking platform. Simulated, not built.

Three C4 diagrams document this at increasing levels of detail:

- `c4-context.mermaid` — RAGuard as a black box, and the external actors/systems it talks to.
- `c4-container.mermaid` — the deployable pieces inside RAGuard's boundary (Nginx, the App, the
  Legacy Mock, Postgres, Redis) and how they communicate.
- `c4-component-app.mermaid` — the internal pipeline of the App container itself: the sequence every
  request passes through, guardrail by guardrail.

---

## 2. MVP Scope

Every component named across D1 and the three diagrams, mapped to whether it's in this MVP. Each row
points to the decision that already justifies it — nothing here is re-argued, only compiled.

| Layer | Component | In MVP? | Reference |
|---|---|---|---|
| Entry & security | Nginx — reverse proxy, TLS termination, IP allowlist/Basic Auth | ✅ Yes | D1 Risk 8 |
| Entry & security | Rate limiting (Redis-backed counter) | ✅ Yes | D1 Risk 1, Risk 12; Component diagram |
| Entry & security | PII redaction (inbound) | ✅ Yes | D1 Risk 5, Risk 6 |
| Entry & security | Full authentication/authorization layer (OIDC) | ❌ Deferred | D1 Risk 8 |
| Entry & security | Secrets vault, automated rotation | ❌ Deferred | D1 Risk 8 |
| Cloud-native services | Pipeline Orchestrator (owns sequence + abort decisions; guardrails as pure functions) | ✅ Yes | Component diagram; ADR pending |
| Cloud-native services | Retrieval Engine + embedded ChromaDB | ✅ Yes | Component diagram |
| Cloud-native services | Retrieval Sufficiency Check | ✅ Yes | D1 Risk 13; Component diagram |
| Cloud-native services | Cache lookup (read path, by redacted query, before retrieval) | ✅ Yes | Component diagram |
| Cloud-native services | Ingestion Pipeline (chunking, embedding, load to ChromaDB, "last reviewed" field) | ✅ Yes — offline script, not part of the request path | D1 Risk 13; §3; Component diagram |
| Cloud-native services | Prompt Injection Defense (retrieved documents) | ✅ Yes | D1 Risk 7; Component diagram |
| Cloud-native services | System-prompt hardening (direct injection in user message) | ✅ Yes — partial | D1 Risk 14 |
| Cloud-native services | Dedicated input-scanning module (direct injection) | ❌ Deferred | D1 Risk 14 |
| Cloud-native services | Generation Engine + LLM/embeddings client | ✅ Yes | Component diagram |
| Cloud-native services | Output Validation (PII scan + grounding check) | ✅ Yes | D1 Risk 9; Component diagram |
| Cloud-native services | Multi-provider fallback / automatic failover | ❌ Deferred — hard-fail-and-be-honest is the MVP behavior | D1 Risk 11 |
| Cloud-native services | Session state | ⚠️ Simplified | D1 Principle 5 |
| Cloud-native services | Chat history (PostgreSQL) | ✅ Yes — persisted by the orchestrator after Output Validation, using the same redaction detector as inbound | D1 Risk 10; Component diagram |
| Cloud-native services | Cache (Redis: responses, query embeddings, rate-limit counters) | ✅ Yes | Component diagram |
| Cloud-native services | Message broker / async queue | ❌ Deferred | D1 Risk 1 |
| Cloud-native services | Autoscaling | ❌ Deferred | Brief; D1 Principle 5 |
| Legacy abstraction | Legacy Data Necessity Check (keyword classifier on redacted query) | ✅ Yes | D1 Risk 2; Component diagram |
| Legacy abstraction | Legacy Data Client (schema validation) | ✅ Yes — called conditionally, only when the Necessity Check says it's needed | Component diagram |
| Legacy abstraction | Legacy Mock Service (REST, no broker) | ✅ Yes — simplified | Brief; Container diagram |
| Legacy abstraction | Event bus | ❌ Deferred | Brief |
| Legacy core | OmniBank Legacy Core (real on-premise platform) | ❌ Simulated | Context diagram |
| Infrastructure & CI/CD | Terraform IaC, manual `apply` approval | ✅ Yes | D1 §2.2; Risk 8 |
| Infrastructure & CI/CD | EBS volume encryption | ✅ Yes | D1 Risk 10 |
| Infrastructure & CI/CD | Least-privilege IAM; encrypted/ignored Terraform state | ✅ Yes | D1 Risk 8 |
| Infrastructure & CI/CD | CI gates: Trivy, pip-audit, Bandit (blocking) | ✅ Yes | D1 §2.2 |
| Infrastructure & CI/CD | AWS budget alerts | ✅ Yes | D1 Risk 12 |
| Infrastructure & CI/CD | `/metrics` + Grafana provisioned as code | ✅ Yes (D4) | Brief |
| Infrastructure & CI/CD | Distributed tracing | ❌ Deferred | Brief |
| Infrastructure & CI/CD | Multi-environment promotion | ❌ Deferred | Brief |
| Infrastructure & CI/CD | Encrypted backups; formal retention/deletion policy | ❌ Deferred | D1 Risk 10 |
| Compliance & governance | Compliance mapping (GDPR/PCI-DSS language, not certification) | ✅ Yes | D1 §3.3 |
| Compliance & governance | DPIA, data-subject rights workflow, full audit logging | ❌ Deferred | D1 §3.3 |
| Compliance & governance | Corpus ingestion integrity controls | ❌ Not applicable now — self-curated corpus is the control | D1 §4.1 scope note |
| Compliance & governance | Document versioning/governance workflow | ❌ Deferred — minimal "last reviewed" field only in MVP | D1 Risk 13 |
| Channels | Web channel | ✅ Yes | Context diagram |
| Channels | Mobile channel | ❌ Deferred | Brief; Context diagram |
| Channels | Third-party SSO / identity provider | ❌ Deferred | Context diagram |

---

## 3. Chunking Strategy (Ingestion Pipeline)

Hierarchical with fallback: documents are split first by logical section — one FAQ entry, one
product, or one policy per section, matching how the self-curated corpus is authored. If a resulting
section exceeds a token-length ceiling, that section is re-split by fixed size with overlap, as a
safety net for the rare document that isn't cleanly structured. Because the corpus is self-curated,
the fallback is expected to trigger rarely — but its presence is what makes the strategy hold up
against documents that don't arrive as neatly segmented as the ones written for this MVP, which is
closer to how real-world document sets actually behave.

---

*Pending for D2: ADRs (pipeline orchestration pattern — chain vs. orchestrator; Cohere vs. local
embeddings; ChromaDB vs. a managed/pgvector store; GitHub Actions vs. GitLab CI; single-LLM-provider
lock-in; the legacy abstraction pattern) and the CI/CD pipeline stage design.*
