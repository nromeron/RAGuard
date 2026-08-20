# ADR 0002: External Embeddings API over Local Embeddings Model

**Status:** Accepted
**Date:** 2026-08-19

## Context

The original team's deploy broke on EC2 free tier because it downloaded the embeddings model's
weights at startup and there wasn't enough disk. That turned what looked like a technical decision —
which model to use — into an architectural constraint: any option that requires downloading or
keeping local weights on the instance carries the same risk that already materialized once. This
isn't "obviously, use whatever's most accurate" — it's deciding with the MVP's real constraint on the
table, and that constraint is operational, not about embedding quality.

## Alternatives

**1. Local model on the same EC2 instance**
Using `sentence-transformers` with a small model like `all-MiniLM-L6-v2`. Advantages: no per-token
cost, no network dependency, full control. Disadvantages: downloads weights at startup, consumes
RAM/CPU on a free-tier instance, and reproduces exactly the disk failure the original team hit.

**2. External embeddings API — Cohere (or equivalent)**
No local weights, no download, no meaningful disk/CPU consumption. Paid per token. Network and vendor
dependency.

**3. A dedicated embeddings endpoint on GPU infrastructure**
Running an embeddings model on a GPU instance or a service like a SageMaker endpoint. Advantages: more
control, no per-token cost, no weights on the app instance. Disadvantages: high fixed cost, doesn't
fit free tier, operational complexity disproportionate to this MVP. Discarded quickly on budget
grounds.

## Decision and why

**Decision:** external embeddings API, Cohere as the concrete provider.

The reason that mattered most was avoiding the disk/startup failure that broke the team's deploy, not
the relative accuracy of the model. Cohere is competitive, but the decision would have been the same
with an equivalent provider. The second factor was that the external API keeps the instance light and
startup fast, which matters more in a free-tier MVP than any marginal difference in embedding quality.
The local option is the most attractive from a control standpoint, but it reintroduces exactly the
risk this project is designed not to repeat.

## Consequences

The most important consequence is already in D1, Challenge #5: redacting PII before the embedding
call takes the external provider out of the regulatory boundary. That argument was written with the
LLM in mind, but it applies exactly the same way to the embeddings provider — it's the same redacted
data traveling to the same kind of external API. Inbound redaction becomes even more critical, because
it now protects two outbound calls, not one.

Other consequences:

- **Network dependency and availability.** If Cohere goes down, retrieval is unusable even if the LLM
  is available. This is already covered by D1 Risk 11 (hard-fail for both APIs) — choosing Cohere is
  what activates that risk in practice, not something new this ADR discovers.
- **Vendor lock-in and pricing changes.** If Cohere changes its pricing or deprecates the model,
  reacting means re-embedding the entire corpus with the new model. That's not trivial. The cheap
  mitigation is abstracting the embeddings client behind an interface — this lives at the code level
  (an interface that Retrieval Engine and Ingestion Pipeline consume), not a new component in the
  architecture diagram — and storing the embeddings model version in the corpus metadata from day one.
  This data is stored in the same step where Ingestion Pipeline already sets "last reviewed" per
  document (Risk 13, D2 §3) — it's not new work, just one more field in the same metadata.
- **Costlier alternative evaluation.** If I want to compare Cohere against a local model tomorrow, I'd
  need to re-ingest the documents with the other model and compare retrieval; it's not an afternoon's
  test. Storing the model and chunking version in the corpus metadata is what makes that comparison
  possible without rewriting half the pipeline.
- **Variable per-token cost.** There's a cost for every embedded query, though rate limiting already
  contains it as a cost control (D1 Risk 12). The embeddings budget is a fraction of the generation
  cost, but it exists.
- **Symmetry note with the LLM client.** ADR 0005 establishes the same abstraction pattern for the LLM
  side (`LLMClient`) and, in its own Consequences, notes that `EmbeddingsClient` needs the equivalent
  hard-fail handling for Risk 11 — catching the failure, applying retries where appropriate, and
  raising the same domain exception the orchestrator translates into "service temporarily
  unavailable" — so Risk 11 is covered symmetrically across both providers, not just one.
