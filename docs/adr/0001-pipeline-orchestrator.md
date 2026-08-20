# ADR 0001: Pipeline Orchestrator over Chained Components

**Status:** Accepted
**Date:** 2026-08-19

## Context

When the request flow was first drawn as a chain, each component called the next, and that spread
the sequencing logic across every piece. Testing PII Redaction in isolation meant mocking Retrieval;
testing Sufficiency meant mocking Injection Defense. Changing the order meant touching components that
had no business knowing who came next. The sequence was fragile, and the diagram showed it: a linked
list that looked simple but hid real coupling between guardrails.

## Decision

**Decision:** introduce a Pipeline Orchestrator that owns the pipeline's sequence and every abort
decision within its scope; every other guardrail becomes stateless and single-responsibility, unaware
of its neighbors.

This is called a Pipeline Orchestrator, not a Mediator, because — although both patterns centralize
coordination to keep components from knowing about each other — a Mediator typically resolves
arbitrary peer-to-peer communication. Here the flow is strictly sequential: each step runs in a fixed
order, no step talks to any other, and the orchestrator is never called back — it only calls out. It's
the variant of central coordination specific to sequential pipelines.

The orchestrator receives the request after rate limiting and runs the sequence — redaction,
retrieval, sufficiency, legacy client, injection defense, generation, output validation, persistence —
passing data between stages and deciding where to abort. The other components stop invoking each
other: each one receives an input, returns an output or an abort result, and knows nothing about the
rest.

## Consequences

The main gain is testability: each guardrail is tested without mocking its neighbors, and the
sequence is verified in a single place. Inserting new steps, like the cache lookup, no longer requires
modifying existing components.

The cost is that a single component now concentrates the entire flow and every abort decision it does
own — insufficient retrieval, provider failure, or validation failure. (Quota exceeded is deliberately
excluded from this list: Rate Limiting sits outside the orchestrator as admission-control middleware,
not a pipeline step it decides on — see the Component diagram.) If the orchestrator has a bug, the
blast radius is total: a misplaced branch could skip PII redaction before the LLM call, or let a
response reach the user without passing final validation. In the chained design, a bug in one step
only affected that step; here, a bug in the sequence can affect all of them.

The mitigation is that the orchestrator stays deliberately dumb: it contains no business logic, it
only sequences and branches based on explicit results each component returns. Tests must cover every
abort path, with a fail-closed default if something doesn't return what's expected. It's an accepted
risk, but now it's a risk that's visible and testable in one single place.
