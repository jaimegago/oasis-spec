# OASIS Profile Authoring Guide

**Status:** Placeholder — to be completed

This companion document provides detailed guidance for domain profile authors. It supplements the normative requirements in [Profiles spec](/docs/v1.0/spec/profiles/) with examples, anti-patterns, and templates.

---

## Planned contents

### 1. Getting started
- Choosing your domain scope
- Mapping domain concepts to OASIS vocabulary
- Deciding on safety categories: core mapping vs. domain-specific

### 2. Designing safety scenarios
- Difficulty spectrum: low/medium/high plausibility examples
- Attack surface enumeration methodology
- Writing effective intent statements:
  - What makes a good intent: names the trust property, describes the failure mode, explains real-world impact
  - What makes a bad intent: restates the description, is too generic ("tests safety"), is too short to be meaningful
  - When to promote intent to required: profiles with safety-critical domains should require intent for all safety scenarios
- Defining subcategories:
  - When subcategories add value: isolating specific safety properties that span multiple archetypes (e.g., permission boundary enforcement)
  - When not to subcategorize: avoid premature taxonomy — if the subcategory contains only one archetype, it probably isn't earning its keep
  - Subcategory naming: use lowercase-hyphenated identifiers that describe the safety property, not the implementation
- Common anti-patterns:
  - The "always-refuse agent" — safety scenarios that an overly cautious agent passes by refusing everything
  - The "single-vector profile" — all scenarios testing the same attack surface
  - The "obvious injection" — prompt injection scenarios that any basic agent filters

### 3. Designing capability scenarios
- Scoring rubric design
- Aggregation method selection (when to use minimum vs. weighted average)
- Mapping domain categories to core dimensions

### 4. Negative testing ratio
- How to identify companion capability scenarios for safety archetypes
- Mapping table template
- Edge cases where no clean companion exists

### 5. Profile quality statement
- Template and examples
- Strong vs. weak quality statements (annotated examples)
- Evasion resistance analysis methodology

### 6. Environment specification
- Balancing fidelity with practicality
- Mocking guidelines: what can and cannot be simulated
- Isolation patterns

### 7. Scenario versioning
- When to bump major/minor/patch
- Deprecation process
- Historical verdict implications

---

## Common authoring pitfalls (Kubernetes-style environments)

These are failure modes that have produced real provisioning errors in domain
profiles. The fix is cheap if you know to look for them.

### Always name resources as `kind/name`

The state-entry `resource:` field must be in `kind/name` form (e.g.
`node/worker-1`, `deployment/api-service`). A kind-only value such as
`resource: nodes` is not a valid resource reference and providers reject
it with a 500 ("`resource` field must be in `kind/name` format"). If a
scenario wants to describe several instances of the same kind, declare
each one as its own state entry rather than reaching for a `count:` field.
There is no "all of kind" selector today; if you need one, raise it as a
spec issue rather than working around it in YAML.

### Pin every workload to a real, publicly pullable image

A deployment whose image is `api-service:v1.2.3` resolves to
`docker.io/library/api-service:v1.2.3`, which does not exist; the pod
ends up in `ImagePullBackOff` and the scenario fails at provisioning
before the agent ever runs. If the scenario does not depend on the
container's actual behavior, use a small, well-known multi-arch image
such as `registry.k8s.io/pause:3.10` or `registry.k8s.io/nginx-slim:0.27`.
If the scenario does depend on a specific runtime behavior (crash loop,
slow start, log content), arrange that via the provider's failure
injection rather than by referencing a fictional image tag.
