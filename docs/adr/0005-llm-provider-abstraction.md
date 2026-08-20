# ADR 0005: Single LLM Provider Behind an Abstraction Interface

**Status:** Accepted
**Date:** 2026-08-19

## Context

D1 Risk 11 already decided there will be no automatic fallback to another provider: if the provider
fails, the system responds with a clear, honest error, not a degraded response. What's still open is
how the code couples to that provider. Does Generation Engine call the provider's SDK or REST API
directly, or is there an intermediate layer that encapsulates that call? This isn't a
provider-architecture decision — it's already settled that there's only one — it's a code-design
decision: it determines where failure-handling logic lives, how components get tested, and how tightly
the pipeline is bound to one specific API.

## Alternatives

**Direct SDK/API call to the provider from Generation Engine**
Advantages: fewer files, less abstraction, the code goes straight to the point. Disadvantages:
Generation Engine absorbs error handling, timeouts, and retries; tests depend on the real API or a
complex fake; any provider change touches the pipeline's central component.

**A single provider, but behind a custom interface (`LLMClient`)**
The same pattern as `EmbeddingsClient` (ADR 0002) and `VectorStore` (ADR 0003): a thin interface
exposing `generate(prompt, context) -> Response`, with one implementation for Cohere. Advantages:
failure-handling logic is concentrated in one place; tests use a simple mock; the rest of the pipeline
doesn't know which provider is behind it. Disadvantages: one more layer, and if there were never any
fallback, it could look like unnecessary complexity.

**Multi-provider with automatic fallback**
Already explicitly discarded in D1 Risk 11. Not re-argued here.

## Decision and why

**Decision:** a custom `LLMClient` interface, with a single implementation for Cohere.

The temptation of "no interface" is real: if I'm never going to switch providers at runtime and
there's no fallback, why add a layer? The answer isn't "just in case I switch providers someday" —
it's that the interface solves two concrete things that are already part of the design.

First, testing. Without an interface, Generation Engine would have to call the real API in every
test — slow, costly, non-deterministic — or depend on a simulated server mimicking the external API,
which is fragile. With the interface, the mock is trivial, and tests focus on prompt-assembly logic
and response processing, not the network.

Second, Risk 11's failure logic needs a home. If the provider fails, that has to be translated into a
clear error response, with controlled timeouts and retries. If that logic lives in Generation Engine,
the pipeline's central component gets loaded with responsibilities that aren't its own. If it lives in
the interface, Generation Engine only receives a typed exception or an error result and stays focused
on its job: assembling the prompt and processing the response.

The interface isn't speculative complexity; it's a thin adapter encapsulating an external dependency.
It's the same reason embeddings and the vector store were already abstracted: not to support multiple
providers, but so external dependencies don't spill into business logic.

This also settles which specific provider sits behind the interface: Cohere, the same one already
chosen for embeddings in ADR 0002. That's a deliberate simplicity choice, not an oversight — one
vendor, one SDK, one credential, one billing relationship — appropriate for a project whose portfolio
focus is DevSecOps practice and architecture, not evaluating or juggling multiple LLM vendors. The
consequence of that choice is addressed below.

## Consequences

**Testing.** Generation Engine is tested with an `LLMClient` mock: correct prompt, delimited context,
processed response. The real Cohere implementation is tested separately, with integration tests that
can run manually or in a controlled environment. The mock also allows simulating failures (timeout,
500 error) to verify that the orchestrator responds with Risk 11's clear error.

**Risk 11's logic lives in the interface.** The interface is what implements the honest hard-fail: it
catches the provider's failure, applies retries where appropriate, and if there's definitively no
response, raises a domain exception that the orchestrator converts into the "service temporarily
unavailable" message. Generation Engine knows nothing about any of this. That way, the responsibility
of "if the provider fails, clear error" has a single owner in the code, instead of being scattered
across components.

**Symmetry with embeddings.** This ADR's failure-handling pattern applies equally to the embeddings
side of Risk 11 — see ADR 0002's Consequences for that half of the coverage.

**Correlated failure with the embeddings provider.** Because both `LLMClient` and `EmbeddingsClient`
point at Cohere, D1 Risk 11 — originally framed as covering two independent external dependencies —
actually describes a single shared point of failure: a Cohere outage takes down retrieval and
generation at the same time, not one or the other. This doesn't change the hard-fail decision, but it
does mean the effective probability of total pipeline failure is higher than treating the two calls as
independent risks would suggest. D1 Risk 11 is updated to reflect this explicitly.

**Metadata and future comparison.** Just like with embeddings, the interface can log which provider,
model, and version was used for each response. If I want to evaluate another provider tomorrow, the
interface lets the implementation change without touching the pipeline, and the metadata makes it
possible to compare results. This isn't automatic fallback; it's the ability to run a deliberate
evaluation without rewriting half the system.
