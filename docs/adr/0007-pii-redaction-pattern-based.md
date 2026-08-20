# ADR 0007: Pattern-Based PII Redaction, Biased Toward Over-Redaction

**Status:** Accepted
**Date:** 2026-08-19

## Context

PII redaction is the control that gives this project its reason to exist, and it isn't a decision
that can be waved off as "obviously, use regex." The point isn't that regex is cheap — it's what kind
of error is acceptable when the detector fails. A false negative — PII that passes through unredacted
— is a personal-data leak; a false positive — harmless text redacted unnecessarily — degrades the
quality of the question, but exposes nothing. This ADR documents why pattern-based detection, with a
deliberate bias toward over-redaction, is the right choice for this MVP.

## Alternatives

**Pattern/regex-based detection**
Explicit rules for ID numbers, account numbers, cards, emails, and phone numbers, run in-process.
Advantages: zero new dependencies, minimal latency, deterministic and auditable behavior.
Disadvantages: limited recall, especially for names and addresses.

**NER / ML model for PII detection**
A trained model recognizing entities like names, addresses, organizations. Advantages: potentially
better recall on free text. Disadvantages: if local, it reintroduces the disk-weights and RAM problem
that already broke the original team's deploy; if an external API, it creates a new outbound call
carrying raw text — which would be absurd: sending PII to a third party to detect PII before sending
it to another third party.

**A managed service like AWS Comprehend PII detection**
A specialized external API. Discarded quickly: per-call cost, added network latency, one more
external dependency. Worse, the raw text would travel to that service before being redacted, which
contradicts the whole point of minimizing exposure.

## Decision and why

**Decision:** pattern/regex-based detection, with Luhn validation for card numbers and typed markers
per category.

**Why patterns and not NER:**
The pattern-based option introduces no new resource. No weights to download, no model to load into
memory, no other external API receiving unredacted text. On a free-tier instance, that's decisive. NER
might have better recall on names, but at the cost of either a heavy dependency or an external call
that doubles the exposure surface. For this MVP, patterns plus data minimization are safer than a NER
that promises more detection but introduces a second potential leak point.

**Why Luhn for card numbers:**
A "16 consecutive digits" regex would produce too many false positives: account numbers, long phone
numbers, internal identifiers. The Luhn algorithm validates the checksum of an actual card number, so
it only redacts sequences that genuinely have a valid card shape. This reduces over-redaction without
losing recall: every real card passes Luhn, and non-card numbers are discarded. It's a way of keeping
the bias toward over-redaction without making the query useless from excessive redaction.

**Why typed markers instead of a generic `[REDACTED]`:**
When something is redacted, the data is replaced with a marker carrying its type and instance number:
`[CARD_1]`, `[NAME_1]`, `[PHONE_1]`. The reason is to preserve the shape of the question. "What's the
rate for card `[CARD_1]`?" lets the LLM understand a card is being discussed; `[REDACTED]` destroys
that information and degrades generation. Typed markers also make auditing and testing possible: it's
possible to know exactly which category was redacted and how many times, without ever seeing the
original data.

## Consequences

**Imperfect recall, accepted.** This decision is the direct implementation of D1 Risk 6: the
detector will fail, and the answer isn't promising perfect detection, it's minimizing how much data
ever reaches the detector in the first place. That's why only relevant chunks are sent, history is
truncated, and manual sampling happens at the weekly checkpoint. This ADR doesn't re-argue that point;
it connects to it as the design decision behind it.

**Bias toward over-redaction.** The detector is designed so that, when in doubt, it redacts. The cost
is occasionally losing context; the benefit is not exposing data. Typed markers offset part of that
cost by preserving the semantic type.

**Maintainability.** All rules live in a single module, not scattered across the pipeline. Each
category has adversarial tests, and the control as a whole has a test that fails if it's removed,
satisfying the brief's rule.

**No new dependencies.** No external call, no weights, no managed service. The added latency is
negligible and doesn't affect the 2.5s target.

**Same detector, reused across the pipeline — not just this component.** This decision isn't scoped
to inbound PII Redaction alone: Output Validation reuses the same detector to scan the generated
response before it reaches the user (Risk 9), and the orchestrator reuses it again before persisting
chat history (Risk 10). One implementation, three points of contact with PII across the request
lifecycle — inbound, outbound, and at rest — rather than three separate detectors that could drift
out of sync with each other.
