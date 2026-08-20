# ADR 0004: GitHub Actions over GitLab CI

**Status:** Accepted
**Date:** 2026-08-19

## Context

The repository already lives on GitHub, which could make this decision look obvious. But there are
two prior decisions this ADR has to hold up. D1 risk register row 8 already assumes GitHub Actions'
native secrets mechanism to keep credentials out of the pipeline. And the original team's failure —
gates that didn't block, Trivy that disappeared, `|| true`, `-auto-approve` — happened precisely in a
CI/CD pipeline. This isn't just "which tool"; it's which platform lets me implement blocking gates and
manual approval reliably, without shortcuts that end up reproducing the same pattern.

## Alternatives

**1. GitHub Actions** — the repo is already there, the secrets mechanism is native, and the
integration with the PR flow and the GitHub profile is direct.

**2. GitLab CI** — a very solid platform, with powerful pipelines and good abstractions. But it would
mean moving the repository to GitLab, or maintaining a mirror between GitHub and GitLab just to run
CI. It gains maturity in some respects, but loses the simplicity of having everything in one place,
and adds operational friction that doesn't add anything to this MVP.

**3. Self-hosted Jenkins** — full control, but requires its own server instance, maintenance, and
runs against the free-tier resource constraint already established. Discarded due to overhead.

## Decision and why

**Decision:** GitHub Actions.

The main technical reason is that the repo is already on GitHub and the native secrets mechanism was
already assumed in D1. But there's a non-technical reason that carries equal weight: this is a
portfolio project, and recruiters look directly at the GitHub profile. Having CI integrated in the
same place reinforces the narrative that the project is real, reproducible, and auditable — without
whoever reviews it having to jump to another platform to see how it's built. GitLab CI is technically
excellent, but moving or mirroring the repo just to use its CI doesn't make sense when GitHub Actions
covers exactly what's needed: workflows as code, blocking gates, native secrets, and manual approval.

## Consequences

- **Syntax lock-in.** GitHub Actions' YAML is platform-specific. If I wanted to migrate to GitLab CI
  tomorrow, the workflow format, actions, and `on` syntax would all change. The mitigation is keeping
  the YAML as thin as possible: each step invokes independent scripts (`bash`, `python`) instead of
  embedding complex logic. That way, the lock-in is limited to orchestration, not to the CI logic
  itself.

- **Required reviewers for manual approval.** Verified against GitHub's official documentation:
  environments, environment secrets, and deployment protection rules are available in public
  repositories for all current plans. In private repositories, GitHub Pro, Team, or Enterprise is
  needed just to have environments and secrets at all — but *required reviewers* specifically, on a
  private repo, requires Enterprise; Pro and Team aren't enough for that particular rule. Since this
  project will be public as a portfolio piece, the free plan covers it completely. If the repo ever
  became private, manual approval for `terraform apply` would stop being available without Enterprise,
  and an alternative mechanism would be needed (for example, a manual step outside GitHub Actions).
  Documented now so it isn't a surprise in D3.
