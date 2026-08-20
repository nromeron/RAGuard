# D2 — Target Architecture and MVP Scope

**Project:** RAGuard
**Status:** Complete — architecture, MVP scope, three C4 diagrams, six ADRs, and CI/CD pipeline design

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
| Cloud-native services | Pipeline Orchestrator (owns sequence + abort decisions; guardrails as stateless, single-responsibility components) | ✅ Yes | Component diagram; ADR 0001 |
| Cloud-native services | Retrieval Engine + embedded ChromaDB | ✅ Yes | Component diagram |
| Cloud-native services | Retrieval Sufficiency Check | ✅ Yes | D1 Risk 13; Component diagram |
| Cloud-native services | Cache lookup (read path, by redacted query, before retrieval) | ✅ Yes | Component diagram |
| Cloud-native services | Ingestion Pipeline (chunking, embedding, load to ChromaDB, "last reviewed" field) | ✅ Yes — offline script, not part of the request path | D1 Risk 13; §3; Component diagram |
| Cloud-native services | Prompt Injection Defense (retrieved documents) | ✅ Yes | D1 Risk 7; Component diagram |
| Cloud-native services | System-prompt hardening (direct injection in user message) | ✅ Yes — partial | D1 Risk 14 |
| Cloud-native services | Dedicated input-scanning module (direct injection) | ❌ Deferred | D1 Risk 14 |
| Cloud-native services | Generation Engine + LLM/embeddings client | ✅ Yes | Component diagram; ADR 0002, ADR 0005 |
| Cloud-native services | Output Validation (PII scan + grounding check) | ✅ Yes | D1 Risk 9; Component diagram |
| Cloud-native services | Multi-provider fallback / automatic failover | ❌ Deferred — hard-fail-and-be-honest is the MVP behavior | D1 Risk 11 |
| Cloud-native services | Session state | ⚠️ Simplified | D1 Principle 5 |
| Cloud-native services | Chat history (PostgreSQL) | ✅ Yes — persisted by the orchestrator after Output Validation, using the same redaction detector as inbound | D1 Risk 10; Component diagram |
| Cloud-native services | Cache (Redis: responses, query embeddings, rate-limit counters) | ✅ Yes | Component diagram |
| Cloud-native services | Message broker / async queue | ❌ Deferred | D1 Risk 1 |
| Cloud-native services | Autoscaling | ❌ Deferred | Brief; D1 Principle 5 |
| Legacy abstraction | Legacy Data Necessity Check (keyword classifier on redacted query) | ✅ Yes | D1 Risk 2; Component diagram |
| Legacy abstraction | Legacy Data Client (schema validation) | ✅ Yes — called conditionally, only when the Necessity Check says it's needed | Component diagram; ADR 0006 |
| Legacy abstraction | Legacy Mock Service (REST, no broker) | ✅ Yes — simplified | Brief; Container diagram; ADR 0006 |
| Legacy abstraction | Event bus | ❌ Deferred | Brief; ADR 0006 |
| Legacy core | OmniBank Legacy Core (real on-premise platform) | ❌ Simulated | Context diagram |
| Infrastructure & CI/CD | Terraform IaC, manual `apply` approval | ✅ Yes | D1 §2.2; Risk 8 |
| Infrastructure & CI/CD | OIDC federation for AWS auth in CI (no static credentials) | ✅ Yes | D1 Risk 8; §4; ADR 0004 |
| Infrastructure & CI/CD | App deploy via AWS SSM Run Command (no public SSH) | ✅ Yes | §4 |
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

## 4. CI/CD Pipeline Design

### Guiding principles

1. Every gate blocks. No `|| true`, no failures that don't break the pipeline.
2. Manual approval to apply infrastructure. `terraform apply` never runs just because a push landed
   on `main`.
3. No static AWS credentials. OIDC federation, not `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` as
   secrets — see ADR 0004 and D1 Risk 8, of which this decision is the natural evolution.
4. Reproducible artifacts. Docker images are built once and promoted, not rebuilt at every stage.

### Stages

**1. Lint** — Triggers: push to any branch, PRs. Commands: `ruff check .`, `black --check .`, `mypy`
(at least on the app). Gate: blocking.

**2. Unit tests** — `pytest` with coverage. Includes tests for every guardrail (PII redaction, prompt
injection defense, output validation, rate limiting), orchestrator tests with stubs, and Legacy Data
Client tests against a REST mock spun up within the job itself. Gate: blocking.

**3. Security scans (code and dependencies)** — Bandit (static security analysis for Python) and
pip-audit (vulnerable dependencies in `requirements.txt`). Gate: blocking.

**4. Build and push Docker images** — Only `app` and `legacy-mock` are built (Nginx uses the official
image with config mounted via volume — TLS, allowlist — there's no custom image to build). Tagged
with the commit SHA, scanned with Trivy, and pushed to Amazon ECR. Gate: successful build and Trivy
with no critical/high findings, with severity explicitly configured to avoid false positives breaking
the build for no reason.

**5. Terraform plan** — Runs against remote state. On PRs, the plan is automatically commented for
review. On `main`, it's saved as an artifact for the next stage. Gate: the plan must run without
errors.

**6. Terraform apply (with manual approval)** — `main` only. Uses a GitHub Actions environment
(`production`) protected by required reviewers — available on the free plan because the repo is
public (ADR 0004). The job stops until a reviewer approves manually. Gate: mandatory manual approval,
no `-auto-approve`.

**7. Application deploy — AWS SSM Run Command** — `terraform apply` provisions infrastructure (EC2,
security group, EBS volume), but doesn't deploy the app: new images in ECR don't reach the instance on
their own. This stage runs `docker login` to ECR, `docker compose pull`, and `docker compose up -d` on
the instance via SSM Run Command — without opening port 22 or exposing public SSH. The instance
receives commands through the SSM agent already present, authorized by an instance IAM role; the
pipeline has its own role with `ssm:SendCommand` scoped to that specific instance. Both roles are
defined in Terraform, not set up by hand. Gate: blocking — if the deploy fails, the infrastructure is
fine but the app didn't update, and the pipeline flags it as such.

**8. Post-deploy smoke test** — `curl` against the `/metrics` endpoint and a health endpoint, to
confirm the instance is up and the app responds after a deploy with no downtime. Gate: blocking. If it
fails, the commit shows red, but there's no automatic rollback in the MVP — it's a signal for manual
intervention (check logs, diagnose, fix); infrastructure that's already applied doesn't revert itself,
and that's documented as accepted, not as an oversight.

### Full flow

```
Push to a feature branch
  -> Lint
  -> Unit tests
  -> Security scans (Bandit, pip-audit)
  -> Build + Trivy scan (no push)
  -> Terraform plan (commented on PR)

Push to main
  -> Lint -> Unit tests -> Security scans
  -> Build + Trivy scan + push to ECR
  -> Terraform plan (saved)
  -> [Manual approval] -> Terraform apply
  -> Deploy: SSM Run Command (docker compose pull && up -d)
  -> Smoke test
```

### Implementation notes

- **Terraform state:** S3 backend with SSE-S3 encryption. Locking via DynamoDB (free tier includes
  enough capacity).
- **AWS authentication — OIDC federation, no static credentials.** An IAM role with a trust policy
  that lets GitHub Actions' OIDC provider (`token.actions.githubusercontent.com`) assume the role,
  conditioned on the repo and branch (the `sub` claim). In the workflow, the job declares
  `permissions: id-token: write` and uses `aws-actions/configure-aws-credentials` with
  `role-to-assume` and `aws-region` — the action fetches the OIDC token automatically through the
  environment variables GitHub injects; there's no need for, and it isn't part of, `web-identity-token-file`,
  which is the alternative to OIDC, not part of its flow. No `AWS_ACCESS_KEY_ID` or
  `AWS_SECRET_ACCESS_KEY` anywhere. Configured in Terraform: `aws_iam_openid_connect_provider` +
  `aws_iam_role` with the conditioned trust policy.
- **Secrets that do remain GitHub Secrets:** `COHERE_API_KEY` — Cohere doesn't support OIDC, so its
  credential is injected as an environment variable in the jobs that need it, through GitHub Actions'
  native mechanism.
- **No secrets in logs:** no job prints environment variables; commands use safe redirection.
- **Accepted risk of the OIDC migration:** if the setup gets stuck in D3, it can fall back temporarily
  to static credentials — but D3's stated goal is OIDC, documented as a pipeline decision, not a
  deferred improvement.

---

*D2 complete: architecture, MVP scope, three C4 diagrams, six ADRs, and CI/CD pipeline design.*
