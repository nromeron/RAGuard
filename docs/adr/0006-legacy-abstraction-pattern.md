# ADR 0006: Standalone Mock Service for the Legacy Abstraction Layer

**Status:** Accepted
**Date:** 2026-08-19

## Context

OmniBank has a legacy, on-premise banking core that can't be touched. RAGuard needs to read
structured data (rates, products, fees) from that core, but calling it directly would couple the new
system to a platform the brief itself defines as the problem: centralized, without autoscaling,
without digital integration. The legacy abstraction layer exists so RAGuard depends on a stable
interface, not on the core itself. In the MVP the core is simulated, but how it's simulated determines
whether the decoupling is real or just cosmetic.

## Alternatives

**1. Direct call to the legacy core from the RAG engine** — discarded: direct coupling, no ability to
evolve or simulate the core without touching pipeline code.

**2. A legacy mock inside the same FastAPI process** — simpler, an endpoint or module returning
hardcoded data. But it mixes the simulation with the application, and doesn't allow testing the
decoupling as an independent piece. It also doesn't reflect the separation of concerns it would have
in production.

**3. A standalone mock service, exposed over REST** — a separate FastAPI container simulating the
legacy core. RAGuard consumes it through the Legacy Data Client with schema validation. It's the option
most faithful to the target architecture: the legacy core is an external system, and the mock
represents it as one.

## Decision and why

**Decision:** a standalone mock service, consumed via REST through the Legacy Data Client.

The main reason is that physical separation forces the decoupling to be real. If the mock lived inside
the app, the temptation to couple to its internal details would be high, and the day it's replaced by
the real core, a rewrite would be needed. Being a separate container with a REST API forces the Legacy
Data Client to treat the legacy system as external: HTTP call, timeout, error handling, schema
validation. That's exactly what will happen in production, and it's what makes the abstraction
credible.

The REST mock also allows testing the Legacy Data Client in isolation: spin up the mock, feed it
controlled responses, and verify the schema validation rejects the unexpected. If the mock were
embedded, that test wouldn't be possible without contaminating the main process.

## Consequences

- **Testability of the decoupling.** The Legacy Data Client is tested against the REST mock with
  controlled responses, including malformed ones or ones with unexpected free-text fields. This
  validates the schema-validation mechanism that protects the LLM from unvalidated data (D1, Challenge
  #5).
- **Operational cost.** One more container in Docker Compose and one additional HTTP call — not on
  every request, but only when the Legacy Data Necessity Check determines structured data is needed
  (D2 §2), which already bounds this cost rather than paying it unconditionally. On a free-tier
  instance, the remaining cost is acceptable and is already accounted for in D1 Risk 1. If that call's
  latency became a problem even with the conditional check, it could be cached in Redis — a deferred
  decision, not assumed here.
- **Explicit limitation: no event bus.** The brief deferred the event bus, and this ADR confirms it:
  communication with the legacy layer is synchronous REST only. If the legacy core ever emitted
  events, a broker would need to be introduced — and that would be an architecture change, not an
  extension.
