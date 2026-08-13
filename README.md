
# RAGuard — Project Brief

**Owner:** Nicolás Romero Niño · **Started:** August 2026
**Purpose:** Portfolio centerpiece + deliberate practice. Solo build, from scratch.
**Scope:** The full OmniBank business case — all four deliverables, architecture included.

---

## The business case

OmniBank is a financial institution serving millions of retail and corporate clients. Support runs
through a centralised call centre and a static FAQ page, and it doesn't hold up:

| | |
|---|---|
| **Scalability** | Call volume spikes up to **300%** during tax season and market volatility |
| **Cost** | **65%** of tickets are routine questions, at ~**$12** per human interaction |
| **Architecture** | Centralised legacy system, no autoscaling, no integration with digital channels |

**Technical requirements from the brief:**
- Containerised agent, able to scale rapidly during peak traffic
- Semantic search over product manuals and policy documents
- Storage for structured rates, product tiers, real-time session state and chat history
- **PCI-DSS and GDPR compliance: PII must be redacted before any query reaches the LLM layer**

*(OmniBank is a fictional institution from a university course business case. I'm rebuilding the
solution solo, from scratch, to own every decision in it.)*

---

## The four deliverables

I'm following the original course structure, because it's a sound architecture progression:
analysis → architecture → MVP → production-ready with metrics.

### D1 — Solution analysis and risk

**Artifacts:** `docs/01-solution-analysis.md` + one diagram

- Proposed solution and key components
- Cloud-native principles applied, and DevOps/DevSecOps practices adopted
- Challenges: technical, security/compliance, organisational
- Risk register: probability, impact, mitigation — and **which mitigations I will actually implement vs. defer**

That last column is the one the team version didn't have, and it's where the delivery went soft.
Every risk marked "will implement" becomes a test later.

### D2 — High-level architecture and stack

**Artifacts:** `docs/02-architecture.md`, `docs/architecture.png` (C4 context + container), `docs/adr/`

- Target architecture, all four layers (see below)
- Legacy abstraction layer enabling gradual cloud migration without disrupting operations
- Technology stack with justification per choice
- CI/CD pipeline design with stages
- **Explicit MVP scope**: what of the target architecture gets built now, what's deferred, why
- ADR per significant decision — including the ones where I chose the less impressive option

### D3 — MVP: CI/CD, IaC, observability

Working system: RAG pipeline, guardrail layer, containerised, deployed to AWS via Terraform,
GitHub Actions pipeline, `/metrics` endpoint.

### D4 — Definitive MVP + metrics dashboard

Hardened version, impact metrics defined and instrumented, dashboard provisioned **as code**
(Terraform, not clicked together in a console).

---

## Target architecture

Four layers. Everything gets designed; the MVP column says what gets built.

| Layer | Components | In MVP? |
|---|---|---|
| **Entry & security** | Reverse proxy, rate limiting, **PII redaction module** | ✅ Yes — this is the point |
| **Cloud-native services** | RAG engine, semantic search, vector store, LLM client, session state, cache | ✅ Core yes; session state simplified |
| **Legacy abstraction** | Mock legacy API + event bus, so the new system reads the core without coupling | ✅ Yes — simplified (REST mock, no broker) |
| **Legacy core** | On-premise banking platform | ❌ Simulated |

**Deferred, and documented as deferred:** authentication/authorization layer, secrets vault,
distributed tracing, multi-environment promotion, autoscaling, mobile channel.

Deferring is fine. Deferring *silently* is what I'm avoiding.

---

## Design constraints (non-negotiable)

These come from reviewing the team delivery against its own design documents. Four gaps showed up:
the PII module was specified but never coded, the Trivy security stage vanished between design and
implementation, lint commands ended in `|| true` so the stage could never fail, and `terraform apply`
ran with `-auto-approve` despite the design calling for manual approval.

So:

1. **If a control is in the architecture diagram, it exists in the code — or the README says it doesn't.**
2. Every security gate blocks. No `|| true` on anything called a gate.
3. Infrastructure apply requires manual approval.
4. No secrets in code or git history. `.env.example` only.
5. Every implemented control has a test that fails if the control is removed.

---

## Stack

| Layer | Choice | Why |
|---|---|---|
| API | Python 3.11 + FastAPI | Same stack as my professional work; async, typed, self-documenting |
| Vector store | ChromaDB (embedded) | No extra service to operate at MVP scale |
| Embeddings | External API (Cohere or equivalent) | Avoids downloading model weights at startup — the exact failure that broke the team's EC2 deploy on free-tier disk |
| LLM | One provider, one pinned model | Model selection isn't what this project demonstrates |
| Structured data | PostgreSQL | Rates, product tiers, session state, logs |
| Cache | Redis | Frequent-query cache; latency target <2.5s |
| Legacy mock | FastAPI service, separate container | The abstraction layer, standalone |
| Container | Docker + docker compose | |
| IaC | Terraform → AWS free tier | |
| CI/CD | GitHub Actions | Portfolio consolidation on GitHub; I've used GitLab CI professionally already |
| Security scanning | Trivy, pip-audit, Bandit — blocking | |
| Observability | Prometheus metrics + Grafana dashboard, provisioned via Terraform | D4 |

---

## The guardrail layer

The differentiating component. Four controls:

**1. PII detection and redaction (inbound)** — mask PII before embedding or LLM calls. Colombian and
generic formats: cédula, account numbers, card numbers (with Luhn validation to cut false positives),
emails, phones. Redact to typed placeholders (`[CARD_1]`) so the model still sees question shape.

**2. Prompt injection defense** — retrieved documents are untrusted input. Delimit context explicitly,
instruct the model to treat it as data, detect known injection patterns. Test against an adversarial
corpus: documents containing instructions aimed at the model.

**3. Output validation (outbound)** — scan generated answers for PII leakage before returning.
Verify grounding in retrieved chunks; refuse rather than hallucinate on empty retrieval.

**4. Rate limiting and abuse controls** — per-IP limits. Enumeration against a RAG system over
internal documents is a real threat.

Plus `THREAT_MODEL.md`: STRIDE applied to the RAG pipeline, mapped to MITRE ATLAS and OWASP ML
Security Top 10. 8+ threats, each with its control and where that control lives in the code.

---

## Milestones

| # | Phase | Est. | Done when |
|---|---|---|---|
| 1 | **D1** — analysis, risks, mitigation decisions | 6h | Risk register with implement/defer column |
| 2 | **D2** — architecture, C4 diagrams, stack justification, ADRs, MVP scope | 10h | A stranger could build from the document |
| 3 | Skeleton — FastAPI, health, Docker, pytest, CI green | 6h | CI passes on push |
| 4 | RAG core — ingestion, chunking, embeddings, retrieval, generation | 14h | Grounded answer with cited source |
| 5 | **Guardrail layer** — four controls + adversarial tests | 16h | Removing a control turns a test red |
| 6 | Legacy abstraction — mock service, container, integration | 8h | Cloud side reads legacy data through the layer |
| 7 | **D3** — Terraform, AWS deploy, `/metrics` | 10h | Running on AWS, inside free tier |
| 8 | **D4** — impact metrics, Grafana dashboard as code, hardening | 10h | Dashboard provisioned by `terraform apply` |
| 9 | README, threat model, final docs | 8h | Clone → run → understand |

**~88 hours.** At 10h/week that's roughly 9 weeks. That fits alongside the job search **only if this
is also my coding practice** — which it is. It is not an addition to practice time; it replaces it.

**Priority if time compresses:** phases 1, 2 and 5 are the identity of this project. Cut from 6 and 8
before touching them. A project with strong architecture docs and a real guardrail layer beats a
feature-complete one with neither.

---

## How I work with AI on this

This project exists so I can defend it in a technical interview. If I can't explain a line, it
shouldn't be there.

**I write:**
- The entire guardrail layer (phase 5) — every line, including tests
- RAG chain and retrieval logic
- All architecture documents, ADRs and the threat model — this is my reasoning, not generated text
- Terraform resource definitions

**I can delegate:**
- Boilerplate: Dockerfile scaffolding, pytest fixtures, `.gitignore`, dependency pinning
- Explaining errors and unfamiliar APIs
- Review of what I wrote — "what's wrong with this", "what would an attacker do here"
- Refactoring suggestions, after I have something working

**Rules:**
1. Never paste generated code I haven't read line by line.
2. When stuck, ask for an explanation first, a solution second.
3. After each phase, explain the component out loud in 2 minutes without notes. If I can't, I don't understand it yet.
4. Small, frequent commits with real messages. The history is part of the portfolio.

---

## Definition of done

- [ ] Public GitHub repo, pinned on profile
- [ ] All four deliverables documented in `docs/`
- [ ] README: architecture diagram at top, threat→control→location table, one-command run, honest "what's deferred"
- [ ] `docker compose up` works from a clean clone
- [ ] CI green: lint, tests, security scans — all blocking
- [ ] Deployed on AWS via Terraform, manual-approval apply, inside free tier
- [ ] Grafana dashboard provisioned as code
- [ ] THREAT_MODEL.md, 8+ threats mapped to implemented controls
- [ ] I can explain every architectural decision without notes
