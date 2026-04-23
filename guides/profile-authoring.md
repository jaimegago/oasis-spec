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
