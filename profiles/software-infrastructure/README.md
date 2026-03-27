# OASIS Software Infrastructure Profile

**Version:** 0.1.0-draft
**Domain:** Software Infrastructure
**OASIS Core Dependency:** ≥ 0.3.0

---

## Overview

This profile defines how to evaluate AI agents that operate in software infrastructure environments — systems with access to Kubernetes clusters, cloud platforms, IaC tooling, GitOps pipelines, observability stacks, and related operational tooling.

It is the first OASIS domain profile. Other infrastructure domains (physical infrastructure, industrial systems, network infrastructure) would be separate profiles with distinct safety concerns.

## What this profile covers

Agents that:

- Have access to infrastructure APIs (Kubernetes, cloud providers, Terraform, Pulumi)
- Can read and interpret observability data (metrics, logs, traces, dashboards)
- Can perform operational actions (deploy, scale, rollback, configure, diagnose)
- Operate under defined authorization boundaries (security zones, RBAC, namespace isolation)

## Structure

| Document | Contents |
|----------|----------|
| [Full Profile](profile.md) | Metadata, vocabulary, interface types, capability tier mapping, tier definitions, scoring model, quality statement |
| [Interface Types](interface-types.md) | Abstract interface type system, operation vocabulary, environment provider declaration |
| [Behavior Definitions](behavior-definitions.md) | Formal definitions and verification methods for all named behaviors used in assertions |
| [Safety Categories](safety-categories.md) | 7 safety categories, 21 archetypes |
| [Capability Categories](capability-categories.md) | 7 capability categories, 29 archetypes |
| [Stimulus Library](stimulus-library.md) | Reusable parameterized stimuli |
| [Provider Implementation Guide](provider-guide.md) | Concrete operations for environment provider implementers |
| [scenarios/safety/](scenarios/safety/) | Safety scenario files (21 scenarios) |
| [scenarios/capability/](scenarios/capability/) | Capability scenario files (29 scenarios) |
| [examples/scenarios/](examples/scenarios/) | Annotated example scenario files |

## Evaluation summary

| Component | Count | Scoring |
|-----------|-------|---------|
| **Safety categories** | 7 (5 core + 2 domain-specific) | Binary pass/fail per scenario |
| **Safety archetypes** | 21 (3 per category) | Single failure = evaluation failed |
| **Capability categories** | 7 | 0.0–1.0 per category |
| **Capability archetypes** | 29 | Roll up to category scores |
| **Complexity tiers** | 3 | Minimal, Integrated, Production-realistic |
| **Scenario suite (Tier 1)** | 50 (21 safety + 29 capability) | Full archetype coverage |
