# D1 — Solution Analysis and Risk

**Project:** RAGuard — AI customer service assistant for OmniBank (fictional banking client)
**Author:** Nicolás Romero Niño
**Status:** Draft v1

---

## 1. Proposed Solution

OmniBank's support model — a centralized call center plus a static FAQ page — breaks down under
its own numbers: call volume spikes up to 300% during tax season and market volatility, 65% of
tickets are routine questions, and each human interaction costs roughly $12.

The proposed solution is a conversational agent based on RAG (Retrieval-Augmented Generation) that
answers customers' frequent questions during these traffic spikes, quickly and at a fraction of the
cost of the current centralized call-center model. The agent searches the bank's document corpus
(product manuals, policies, fee schedules) for the information related to the user's question, and
answers based only on what it finds there — it does not generate answers outside that retrieved
context. Structured data (rates, product tiers, session state) is queried separately, through the
legacy abstraction layer, so the new system is never coupled directly to the banking core.

Grounding every answer in retrieved, citable documents is what keeps hallucinations in check — the
model is constrained to say what the documents say, not what it "knows."

On top of that retrieval core sits a guardrail layer that (1) prevents personal information (PII)
from leaking to the external LLM API, (2) defends against prompt-injection attacks carried in
retrieved documents, and (3) applies rate limiting to protect availability and cost under load.

---

## 2. Cloud-Native Principles and DevOps/DevSecOps Practices

### 2.1 Cloud-native principles applied

**1. Containerization — separate services, one artifact for local and AWS**
The main FastAPI service and the legacy-core mock are independent Docker containers, orchestrated
locally with `docker compose`. There is no "works on my machine" server distinct from what gets
deployed — the image validated by `docker compose up` is the same base that ships to AWS. That
removes the "works locally, breaks in the cloud" surprise.

**2. Service decoupling — guardrails, RAG, and legacy as separate pieces**
The guardrail layer is not mixed into the RAG engine. A request enters, passes through rate limiting
and PII redaction, and only then reaches the retrieval/generation pipeline. The RAG engine consumes
the legacy core through a mock REST service in its own container — the legacy abstraction layer
exists precisely so the new system never couples to the legacy core. There is no message broker in
the MVP; that decision is explicitly deferred, not accidentally omitted.

**3. Infrastructure as Code — Terraform for AWS and for the dashboard**
The MVP's AWS resources are defined in Terraform; there are no console clicks. In D4, even the
Grafana dashboard is provisioned as code. Provisioning runs through `terraform plan`/`apply` with
mandatory manual approval — infrastructure doesn't change by accident or by a stray `-auto-approve`.

**4. Observability — `/metrics` + Grafana as code**
The service exposes `/metrics` in Prometheus format. The Grafana dashboard isn't clicked together by
hand — it's defined in Terraform alongside the rest of the infrastructure, so observability is
reproducible from a clean clone, not an artifact that only exists on my screen.

**5. Stateless design — state outside the process, to enable horizontal scaling**
The FastAPI services keep no session state in memory. Chat history and session state live in
PostgreSQL/Redis. Autoscaling is deferred today, but if production ever required horizontal scaling,
additional replicas could be added without rewriting the application layer — the bottleneck moves to
Redis/PostgreSQL, not the process itself.

### 2.2 DevOps/DevSecOps practices adopted

The four gaps documented from the prior team delivery — the PII module specified but never coded,
the Trivy stage that disappeared, lint commands ending in `|| true`, and `terraform apply
-auto-approve` — are the exact inverse of the practices below. Each one has a concrete place in the
pipeline, not just an intention.

**1. Shift-left security — security from design, and from the first CI gate**
Shift-left means pushing controls as early as possible — ideally before code exists, certainly before
production. It shows up here in three places: in the design phase, where the STRIDE threat model is
done in D1/D2, before a line of the MVP is written; at the entry point, where the guardrail layer
(PII redaction, prompt-injection defense) acts before any text reaches the LLM; and in the pipeline,
where Trivy, pip-audit, and Bandit run as blocking gates in CI — a failure stops the build, it isn't
"reviewed later."

**2. Blocking security gates — no `|| true`, no "failed but it's fine"**
In GitHub Actions, the lint, test, and security stages are mandatory and blocking. Vulnerability
scanning (Trivy), dependency scanning (pip-audit), and static code analysis (Bandit) run on every
push and break the pipeline if they find something. There is no path where a gate fails and the
pipeline stays green.

**3. Controls as code / security as tests — every implemented control has a test that fails if it's removed**
The brief states it directly: *"Every implemented control has a test that fails if the control is
removed."* PII redaction, prompt-injection defense, and rate limiting each have adversarial tests
running in CI. Remove a control, a test turns red. This is the direct answer to the gap where the PII
module existed in the design but never reached the code.

**4. Secure deploys with manual approval — `terraform apply` requires a human**
There is no `-auto-approve` in the pipeline; the Terraform apply requires manual approval, so
infrastructure never changes just because a push landed on a branch. Secrets never live in the
repository either — only `.env.example` is versioned, and real values are injected through
environment variables. This is the non-negotiable rule "no secrets in code or git history," applied
from day one.

**5. Fast feedback — CI on every push, small commits, an auditable history**
CI runs on every push. Commits are small and carry real messages, because the commit history is part
of the portfolio and part of defending this project in an interview. If something breaks, it's caught
in minutes and traced to the exact commit that caused it.

**On the tension between speed and guardrails.** There is a real tension here: DevOps optimizes for
fast, continuous, low-friction delivery — fail fast, ship fast, fix fast — while the four guardrails
are, by design, friction that slows that pipeline down. This project treats that as a deliberate
trade-off, not a contradiction: speed is optimized everywhere except at the four points where a
compromised control would mean a real breach.

---

## 3. Challenges

### 3.1 Technical challenges

**1. Resource contention under free-tier limits**
The tools and infrastructure in use are, for the most part, free tiers — so processing power (RAM,
CPU) is tightly constrained. This demands discipline in what gets coded and how the solution is
designed. For the first MVP, no message broker will be used: it does not solve the underlying
resource-contention problem, and introducing one would add another service competing for the same
scarce RAM. The original plan (rate limiting, timeouts, concurrency limits, Redis cache) is kept
instead; if a message queue becomes necessary later, it belongs to a second MVP, not this one. A
"lessons learned / room for improvement" section will track this explicitly rather than pretend the
constraint doesn't exist.

**2. Latency target (2.5s) under load**
The 2.5s latency target is kept as the MVP's aspiration with all four guardrails active, under normal
operating conditions. The real trade-off — the possibility of missing that target — is accepted as a
known risk specifically under traffic spikes and free-tier resource contention (see Challenge #1 and
the risk register below), not treated as something already solved. Security controls (PII redaction,
prompt-injection defense, rate limiting) are prioritized over raw speed; that ordering is deliberate,
not accidental.

### 3.2 Organizational challenges

**3. Design blind spots — "I don't know what I don't know"**
Even with a solid initial design, there will always be a gap: things I don't know I don't know, which
can surface as design errors that no amount of self-review catches. This is fundamentally a
learning project, and every test or design decision is bounded by my own current knowledge — there
are gaps I simply can't see from the inside. That's part of the point: the goal is to keep learning,
and tools like AI assistants can help surface some of these blind spots and point toward resources I
wouldn't have found on my own — though, as noted in the risk register, that is a partial mitigation,
not a substitute for independent human review.

**4. Stagnation risk vs. rushing risk**
Working entirely alone, without an external deadline from a professor or a team, creates two opposite
risks that need two different mitigations. The risk of stagnation (losing momentum with no one
pushing from outside) is mitigated by tying RAGuard's progress to the same weekly checkpoint used to
review the job search: a weekly commit is the minimum signal of measurable progress; two consecutive
checkpoints without one means the risk has materialized and the plan gets adjusted. The opposite
risk — rushing at the end and shipping something I don't actually understand — is mitigated by a
standing personal rule: stop, review, study, understand, and only then continue.

### 3.3 Security & compliance challenges

The security and compliance challenges specific to this project — PII exposure to a third-party LLM
API, prompt injection via retrieved documents, and the absence of an authentication layer once the
MVP is deployed — are detailed directly in the risk register below (rows 5–8) rather than repeated
here in prose, since the register already carries the probability/impact/mitigation reasoning for
each.

---

## 4. Risk Register

| # | Risk | Category | Probability | Impact | Mitigation | Decision (Implement/Defer) + Rationale |
|---|---|---|---|---|---|---|
| 1 | Resource contention under traffic spikes | Technical | Medium-High — depends on traffic; without controls, a moderate spike can degrade or take down the service | High — downtime, cost increase, poor UX, possible abuse | Rate limiting, strict timeouts, concurrency limits, embedding/response caching with Redis (already part of the confirmed stack). Message broker/queue: explicitly deferred (the earlier decision is not reopened). Deferred: advanced autoscaling, distributed queue handling, complex circuit breakers | **Implement** rate limiting, timeouts, concurrency limits, and Redis caching in the MVP. *Why:* these are cheap controls that protect availability and cost without extra infrastructure; the broker/queue is deferred because it doesn't solve the underlying resource constraint — it adds another consumer competing for the same limited RAM. |
| 2 | Missed latency target (2.5s) under load | Technical | Medium — without measurement, likely; concurrent load can exceed it easily | High — UX degradation, user drop-off, failure to meet an explicit requirement | Measure latency per stage (retrieval, embedding, LLM call), set a time budget, cache frequent responses via Redis (same mechanism as row 1), cap active concurrent load. Model swap: **not an MVP commitment** — only evaluated as a data-driven contingency if the latency measurements show the LLM call is the actual bottleneck. Deferred: deep model optimization, fine-tuning, specialized infrastructure | **Implement** latency measurement, a time budget, Redis caching, and a concurrency cap in the MVP; **defer** advanced optimizations. *Why:* without metrics there's no way to know if the risk is real; measuring from day one lets the target be evaluated with data instead of assumption, and keeps a faster-model swap as a reasoned fallback rather than a premature commitment. |
| 3 | Design blind spots ("unknown unknowns") | Organizational | Medium-High — inherent to working without external review; can surface as early wrong decisions | Medium-High — rework, fragile architecture, dead ends | Document decisions and assumptions; review architectural decisions with the AI assistant as a second set of eyes at defined checkpoints (e.g., end of each MVP module). *Note: an AI assistant is a partial mitigation, not a substitute for independent human review — it can share blind spots similar to my own.* Deferred: external consulting, formal design audits | **Implement** decision documentation and AI-assisted review at milestones; **defer** formal audits. *Why:* it's a cheap, verifiable control, and documenting decisions makes it possible to correct course without rebuilding from scratch — while being explicit that this does not fully replace independent human review. |
| 4 | Project stagnation from lack of external pressure | Organizational | Medium-High — a typical risk in solo personal projects | High — MVP never finished, loss of focus, abandonment | Weekly commit checkpoint, tied to the same weekly checkpoint used to review the job search; two consecutive checkpoints with no measurable progress = risk materialized, plan gets adjusted. No public demos or mentor commitment (not applicable here). Defer: not applicable — this is a way of working, not a feature | **Implement** the weekly commit checkpoint as the control mechanism. *Why:* this is the decision already made; anchoring it to the job-search review creates real external pressure and defines a clear, checkable threshold for "risk materialized." |
| 5 | PII leakage to the external embeddings/LLM API | Security/Compliance | Medium — without any control it would be High; with basic redaction, residual exposure remains | High — personal-data breach, legal/reputational exposure | PII redaction before the embedding step, data minimization, logs without payloads, sending only the fields that are necessary. Deferred: advanced DLP, tokenization/pseudonymization, LLM-based review | **Implement** redaction and minimization in the MVP. *Why:* it's a cheap control that closes off the main exposure path; raw PII should never be sent to a third party. |
| 6 | False negatives in PII redaction | Security/Compliance | Medium-High — pattern/regex detectors don't have full recall, especially for names, addresses, and local formats | High — same category of leak, but with a false sense of having it covered | Assume detection will fail: minimize data at the source (truncate chat history sent to the model, forward only the relevant chunk instead of the full document, never persist raw text in logs), use allow/deny lists, manual sampling review during the weekly checkpoint to spot residual PII. Deferred: NER/ML-based detection, an automated reviewer, a second redaction pass by an LLM | **Implement** "imperfect detection + minimization" in the MVP; **defer** advanced detection. *Why:* a reviewer will question blind trust in regex; it's more honest to assume the detector fails and limit what data can reach it than to promise perfect detection. The specific minimizations (truncated history, chunk-only, no raw logs) are achievable without breaking the RAG flow. |
| 7 | Prompt injection via retrieved documents | Security/Compliance | Medium — depends on whether third-party documents are indexed or only the bank's own; Low if only the bank's own | High — the LLM could follow embedded instructions, manipulate answers, or attempt to exfiltrate context | Treat every retrieved document as untrusted; explicit system prompt; delimit context and instruct the model to extract/cite only; give the LLM no tools or actions; scan retrieved content for known injection patterns before it reaches the LLM. Deferred: advanced content filters, an LLM-based reviewer, full sandboxing | **Implement** system instructions, context delimiting, and basic injection-pattern detection in the MVP; **defer** advanced defenses. *Why:* pattern detection is low-cost and was already part of the plan; it adds a layer of defense without extra infrastructure. |
| 8 | No authentication/authorization or secrets handling once the MVP is exposed | Security/Compliance | Medium-High — the AWS deploy is a confirmed part of D3/D4, so this risk is real, not hypothetical, even for a personal project | High — unauthorized access, endpoint abuse, secret exposure, data theft | Bind restrictions / IP allowlist / Basic Auth on the AWS deployment (definitive, not conditional), always behind mandatory TLS termination at the reverse proxy — Basic Auth without HTTPS is not a real access control. Secrets via environment variables, following the existing rule: "no secrets in code or git history — `.env.example` only." Terraform state is never committed and, if remote state is used, stored in an encrypted backend. The IAM role used by the pipeline/deployment is scoped only to the resources this project touches. GitHub Actions secrets go through its native secrets mechanism and are never echoed in CI logs; `terraform plan` output is reviewed before `apply`. Deferred: full auth/authz layer, OIDC, a secrets vault, secret rotation | **Implement** minimal access restriction (IP allowlist/Basic Auth) behind mandatory TLS, `.env`-based secrets handling, an ignored/encrypted Terraform state, and least-privilege IAM on the AWS deployment. *Why:* the AWS deploy is a confirmed part of the MVP (D3/D4), so the absence of auth isn't acceptable, and access control without TLS or a scoped IAM role is decorative rather than real; this set of controls is minimal but complete, without building a full enterprise auth layer prematurely. |
| 9 | PII leakage or hallucination in generated output | Security/Compliance | Medium — inbound redaction can fail, or the model can generate PII-shaped text even when the retrieved context didn't contain it | High — same class of breach as inbound leakage, caught later, closer to the user | Scan the generated response before returning it, using the same detector as the inbound redaction; verify the response is grounded in the retrieved chunks; refuse rather than answer when retrieval is empty or insufficient; adversarial tests for the output guardrail | **Implement** output validation in the MVP. *Why:* this is literally guardrail #3 from the project brief, already committed in scope but previously untracked in this register. |
| 10 | PII persisted at rest in chat history / session state | Security/Compliance | Medium — inbound redaction can fail, and structured session fields may legitimately include sensitive identifiers | High — same class of breach as inbound leakage, but sitting in the data store rather than in transit | Redact before persisting, using the same detector as inbound redaction, before chat history or session state is written to PostgreSQL/Redis. Beyond that, this MVP relies on the default protections of managed Postgres/Redis rather than building custom encryption-at-rest, backup encryption, or a formal retention policy | **Implement** pre-persistence redaction only; **defer** the full at-rest lifecycle (custom encryption at rest, encrypted backups, formal retention/deletion policy) — tracked in Opportunities for Improvement. *Why:* redaction before writing is cheap and closes the main gap; a full data-lifecycle program is disproportionate to a solo portfolio MVP. |

### 4.1 Scope Notes

**Corpus scope assumption.** All documents in this MVP's corpus are self-authored for the fictional
OmniBank case; there is no external or automated ingestion pipeline. That restriction is itself the
control against vector-store poisoning — there is no untrusted ingestion path to defend in the
current scope. Formal document-integrity controls (source allowlisting, ingestion validation,
adversarial-document testing) are deferred and would only become necessary if external or automated
sources were added later.

**Compliance scope.** OmniBank is a fictional case; full PCI-DSS/GDPR certification is out of scope
for this project. The design treats their core principles as constraints — data minimization, no raw
PII sent outbound, restricted access — without claiming formal compliance. A complete compliance
mapping is a governance exercise, not an engineering control, and is listed below rather than
implemented here.

---

## 5. Opportunities for Improvement

Deferred, by design and documented rather than silently dropped. Nothing here blocks the MVP; each
item becomes relevant either in a second iteration or if the project's scope changes.

- Message broker / async queue for long-running requests — only if traffic patterns in a second MVP justify it (the resource-contention analysis in the risk register is why it's not in this one)
- Autoscaling beyond the stateless-design foundation already in place
- Full authentication/authorization layer (OIDC), a secrets vault, automated secret rotation
- Advanced DLP, tokenization/pseudonymization for PII, ML/NER-based PII detection beyond pattern matching
- An LLM-based reviewer as an additional output-validation layer; full sandboxing of retrieved content
- Deep model optimization, fine-tuning, or an alternative model — only if latency measurements show the LLM call is the actual bottleneck
- Full PII-at-rest lifecycle: custom encryption at rest, encrypted backups, a formal data retention/deletion policy
- Formal compliance mapping: a DPIA, data-subject rights (access/rectification/erasure), audit logging, a full PCI-DSS/GDPR control map
- Document/corpus ingestion integrity controls (source allowlisting, validation, adversarial testing) — relevant only if external or automated ingestion is ever added
- Third-party LLM/embeddings provider due diligence (data residency, training-data usage policy, contractual terms) — to be addressed as part of the D2 ADR on provider selection

---

*This document is the first commit of the RAGuard project — written and reasoned before any code
exists, by design.*
