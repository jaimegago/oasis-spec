# OASIS — Open Assessment Standard for Intelligent Systems

OASIS is an open standard for evaluating AI agents that operate in real-world systems. It defines how to test whether an agent is safe to deploy and how capable it is — in that order.

## The problem

AI agents are gaining access to production infrastructure, financial systems, databases, and physical devices. Existing evaluation frameworks measure capability (can the agent do the task?) but treat safety as an afterthought — one score among many.

OASIS inverts this. Safety is a gate, not a score. An agent that fails any safety scenario receives no capability score. It simply fails.

## How it works

```
┌──────────────────────────────────────────────────────────┐
│                    OASIS Evaluation                      │
│                                                          │
│  Phase 0: Provider Conformance Preflight                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Runner queries provider's /v1/conformance         │  │
│  │  Profile-defined requirements checked              │  │
│  │  Mismatch ─► run aborts before any scenarios       │  │
│  └────────────────────────────────────────────────────┘  │
│           │                                              │
│  Phase 1: Safety Gate (runs ALL scenarios, then aggs)    │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Boundary Enforcement          PASS / FAIL         │  │
│  │  Prompt Injection Resistance   PASS / FAIL         │  │
│  │  Authority Escalation          PASS / FAIL         │  │
│  │  Blast Radius Containment      PASS / FAIL         │  │
│  │  Declared Mode Verification    PASS / FAIL         │  │
│  │  + domain-specific categories  PASS / FAIL         │  │
│  └────────────────────────────────────────────────────┘  │
│           │                              │               │
│        ANY FAIL ──────────►  EVALUATION FAILED           │
│           │                  (no capability score)       │
│        ALL PASS                                          │
│           │                                              │
│  Phase 2: Capability Scoring                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Domain-specific categories scored 0.0 – 1.0       │  │
│  │  Mapped to core dimensions:                        │  │
│  │    Task Completion · Reliability                   │  │
│  │    Reasoning · Auditability                        │  │
│  │  Scores reported alongside complexity tier         │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Optional: Adversarial Verification                      │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Non-deterministic probes targeting archetypes     │  │
│  │  Reserved (unpublished) deterministic scenarios    │  │
│  │  Results reported separately — do not modify       │  │
│  │  core verdict. Raises flags or adds confidence.    │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Output: Structured verdict with full audit trail        │
│         (PASS, FAIL, or PROVIDER_FAILURE)                │
└──────────────────────────────────────────────────────────┘
```

Verdicts use a binary safety gate with an explicit third status for harness faults: PASS, FAIL, or PROVIDER_FAILURE. The third status exists so a misconfigured or runtime-faulted evaluation harness can never be silently confused with a clean pass — see [Core, §3.6](spec/01-core.md). Phase 1 runs every applicable safety scenario before aggregating, so a failing agent gets a complete failure surface in one run instead of forcing iterative re-runs.

## Key principles

**Safety is a gate, not a score.** One failure vetoes everything. Non-negotiable.

**Independent verification.** The evaluation checks system state directly. It never trusts the agent's self-reporting.

**Domain profiles.** The core spec is domain-agnostic. Domain-specific knowledge lives in profiles — versioned, self-contained packages that define what safety and capability mean for a specific class of systems.

**Scores require context.** Capability scores are always reported with the complexity tier (Minimal / Integrated / Production-realistic) of the evaluation environment. Scores from different tiers are not comparable.

**Safety through precision, not paralysis.** Every safety test that checks "agent refuses X" has a companion test that checks "agent correctly does legitimate-X." An agent that passes safety by refusing everything has demonstrated inability, not safety.

**Predictability is a vulnerability.** A deterministic, public test corpus can be gamed. OASIS provides an optional adversarial verification extension — non-deterministic probes and reserved scenarios — to test whether safety properties generalize beyond the known test surface.

## Domain profiles

OASIS is extensible through domain profiles. Each profile defines safety categories, capability categories, archetypes, and scenarios for a specific class of external systems.

| Profile | Status | Covers |
|---------|--------|--------|
| [Software Infrastructure](profiles/software-infrastructure/) | RC (v0.2.0-rc1) | Kubernetes, CI/CD, observability, IaC, GitOps |
| Finance | Planned | Trading platforms, payment systems, ledgers |
| Robotics / Industrial | Planned | SCADA, actuators, digital twins |

## Specification

The core specification is split into focused documents:

| Document | Audience | Contents |
|----------|----------|----------|
| [Motivation](spec/00-motivation.md) | Everyone | Why OASIS exists, the gap it fills, comparison against existing frameworks |
| [Core](spec/01-core.md) | Everyone | Definitions, architecture, safety gate, capability model, complexity tiers |
| [Scenarios](spec/02-scenarios.md) | Profile authors, implementers | Scenario schema, stimulus types, assertions, scoring, suites |
| [Profiles](spec/03-profiles.md) | Profile authors | Profile structure, required components, quality criteria |
| [Execution](spec/04-execution.md) | Implementers | Agent interface, environment model, execution flow |
| [Reporting](spec/05-reporting.md) | Everyone | Verdict format, report structure, conformance |
| [Principles & Context](spec/06-principles.md) | Everyone | Design principles, prior art, open questions |
| [Adversarial Verification](spec/07-adversarial-verification.md) | Implementers, certification bodies | Non-deterministic probes, reserved scenarios, adversarial robustness (optional extension) |
| [Provider Conformance](spec/08-provider-conformance.md) | Implementers, certification bodies | What makes an evaluation provider conformant |

## Status: Release Candidate

OASIS v1.0.0-rc1 is feature-complete and has been validated through end-to-end evaluation of a real AI infrastructure agent against the Software Infrastructure profile. We are seeking external feedback before committing to v1.0.0 stability guarantees.

If you are implementing a provider or evaluator against this spec, please file issues for any ambiguities, gaps, or friction you encounter. Feedback during the RC period directly shapes the v1.0.0 release.

See [CHANGELOG.md](CHANGELOG.md) for the history of changes across draft revisions.

## License

This work is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).You are free to share, adapt, and build implementations against this standard. Attribution is required.
