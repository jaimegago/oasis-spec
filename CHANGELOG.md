# Changelog

All notable changes to the OASIS specification are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
OASIS versioning during the pre-1.0 phase uses a `MAJOR.MINOR.PATCH-draft`
suffix; the `-draft` suffix is dropped at release.

## [Unreleased] — 0.4.0-draft

### Clarified

- **Core §3.5.4 added: implementation determinism.** Evaluator implementations
  MUST be pure functions of their inputs — given the same scenario, agent
  transcript, and captured observable evidence, the same verdict MUST be
  produced on every invocation. Applies to both safety and capability
  evaluation, and to both action-based and detection-based assertions.
  Verification heuristics that depend on LLM inference with non-zero
  temperature, random sampling, wall-clock time, or unstable iteration order
  are non-conformant for core evaluation. LLM-as-judge is permitted only in
  the Adversarial Verification Extension (§7). Profile authors wanting
  LLM-judged rubric criteria are directed to decompose them into
  deterministically scorable questions or move them to §7.

- **Principle 4 rewritten** from "Deterministic over probabilistic" to
  "Deterministic verdicts from deterministic evaluators" to surface the
  two-sided requirement (verdict determinism + implementation determinism)
  in the top-level principle list.

- **Provider Conformance §6.3 disambiguated.** The cross-provider
  reproducibility non-guarantee applies only to live agent runs, where agent
  non-determinism and environment timing may produce different transcripts.
  It does NOT relax the within-provider implementation determinism
  requirement of Core §3.5.4: replaying the same recorded evidence through
  the same evaluator MUST yield the same verdict.

- **Provider Conformance §3** now cross-references §3.5.4 and explicitly
  flags LLM-as-judge for core assertions as non-conformant.

- **Profiles §3** verification-method requirement tightened to cite both
  §3.5.3 and §3.5.4, require pure-function behavior, and explicitly redirect
  LLM-judged criteria to the Adversarial Verification Extension.

### Rationale

The prior draft established verdict determinism (every assertion must resolve
to PASS/FAIL/PROVIDER_FAILURE, no NEEDS_REVIEW escape hatch) but did not
explicitly require the evaluator implementation itself to be deterministic.
This left a loophole where a provider could use LLM-as-judge with non-zero
temperature, produce different verdicts on reruns of the same recorded
evidence, and point at §6.3 to claim conformance. These changes close the
loophole by distinguishing two separate properties — within-provider
determinism (now mandatory) and cross-provider verdict equivalence on live
runs (not guaranteed in v0.4) — and by naming LLM-as-judge explicitly as
non-conformant for core assertions.

No existing conformant implementation becomes non-conformant as a result;
substring matching, structural parsing, and direct state inspection remain
conformant and are now explicitly named as examples of deterministic
verification methods.
