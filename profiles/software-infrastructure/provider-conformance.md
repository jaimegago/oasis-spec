# Software Infrastructure — Provider Conformance Contract

**Profile version:** 0.2.0-rc1
**Profile identifier:** `oasis-profile-software-infrastructure`
**OASIS Core Dependency:** ≥ 1.0.0-rc1

This document is the normative provider conformance contract for the Software Infrastructure (SI) profile. It specifies what an evaluation provider must supply for SI scenarios to be runnable, how the provider declares its capabilities, and how the evaluation runner verifies the declaration before any scenarios execute.

This document is the source of truth for SI provider conformance. It is designed to be self-contained: a competent implementer with no other context — explicitly including an LLM-based code generation tool given only this document and the [Provider Implementation Guide](provider-guide.md) — should be able to build a conformant SI provider. If a reader needs to ask questions that this document does not answer, the document is incomplete and should be patched.

For the mechanism by which conformance is declared and checked, see [OASIS Provider Conformance §3.8](/docs/v1.0/spec/provider-conformance/). For the wire-level shape of the SI conformance response, see [Provider Implementation Guide §7](provider-guide.md). This document specifies what each requirement key means and what a provider must do to satisfy it.

---

## 1. Overview

A provider that wishes to claim conformance to the SI profile must satisfy every requirement listed in §3 of this document. Conformance is binary: a provider either satisfies all requirements for a given complexity tier or it does not. There are no provider tiers within SI conformance, only the OASIS environment complexity tiers (1, 2, 3) defined in [Core §5](/docs/v1.0/spec/core/), and a provider declares which tier it supports.

The conformance contract consists of seven requirement keys:

| Key | What it asserts | Why SI requires it |
|---|---|---|
| `environment_type` | The provider provisions Kubernetes clusters | SI scenarios are written against Kubernetes resources |
| `complexity_tier_supported` | The highest complexity tier the provider can provision | Determines which scenarios the provider can run |
| `oasis_core_spec_version` | The OASIS core spec version the provider implements | Spec compatibility with the profile |
| `evidence_sources_available` | The set of observation types the provider supplies with `available` status | Determines whether SI assertions can be verified |
| `state_injection` | The provider can inject preconditions into a provisioned environment | SI scenarios stage initial state before the agent runs |
| `audit_policy_installation` | The provider installs a Kubernetes API server audit policy on its clusters | Required for any audit-log-based safety assertion to produce real evidence |
| `network_policy_enforcement` | The provider's clusters use a CNI that enforces NetworkPolicy resources | Required for zone boundary enforcement scenarios to produce meaningful results |

The remainder of this document specifies each requirement in detail.

---

## 2. Conformance flow

Before any SI scenarios execute, the evaluation runner performs the preflight conformance check defined in [OASIS Provider Conformance §3.8](/docs/v1.0/spec/provider-conformance/):

1. The runner issues `GET /v1/conformance?profile=oasis-profile-software-infrastructure` to the provider.
2. The provider returns a JSON response whose `requirements` map matches the schema in §4 of this document.
3. The runner validates the response against the schema and checks each requirement key against the values declared in this contract.
4. If every requirement is satisfied, the runner proceeds to the agent identity query and scenario execution per [Execution §3](/docs/v1.0/spec/execution/).
5. If any requirement is unsatisfied, the runner aborts with an error naming the specific gap. No scenarios are executed and no verdict file is produced.

This is the only place SI provider conformance is enforced as a configuration check. Once the runner has accepted the provider's conformance response, it assumes the provider is conformant for the duration of the run. If a runtime fault subsequently prevents independent verification of a scenario, the run aborts with PROVIDER_FAILURE per [Core §3.7](/docs/v1.0/spec/core/) — but that is a separate concern from preflight conformance.

---

## 3. Requirements

### 3.1 environment_type

**Type:** string (enum)
**Valid values:** `kubernetes-cluster`
**Required value for SI:** `kubernetes-cluster`

**What it asserts.** The provider provisions Kubernetes clusters as the environment type for scenarios. SI scenarios are written entirely against Kubernetes resources (Deployments, ConfigMaps, Secrets, Services, Ingresses, NetworkPolicies, RBAC, etc.) and require a real or realistically-simulated Kubernetes API server, kubectl-compatible client interface, and the resource lifecycle semantics of a kube cluster.

**Why SI requires it.** Every scenario in the SI scenario suite assumes Kubernetes. Without a Kubernetes-shaped environment, the scenarios cannot be executed at all — their preconditions cannot be staged and their verifications cannot be performed.

**What a satisfying provider does.** The provider provisions an environment that exposes a working Kubernetes API server. The cluster does not need to be a "real" production cluster; lightweight implementations like kind, k3s, k3d, minikube, or equivalent are conformant as long as they expose the standard Kubernetes API contract and support the resource types listed in [Provider Implementation Guide §1](provider-guide.md). Cloud-managed clusters (EKS, GKE, AKS) are also conformant.

**Failure modes the runner catches.** If the provider declares any value other than `kubernetes-cluster` for this key, the runner aborts with `provider does not support required environment type 'kubernetes-cluster' for SI profile`. If the provider declares `kubernetes-cluster` but cannot actually provision a working cluster at scenario time, the failure surfaces as a runtime PROVIDER_FAILURE on the first scenario that attempts provisioning, and the run aborts then.

**Verification method.** The runner verifies the declared value matches the required value during preflight. The actual ability to provision is verified implicitly by the first scenario that runs.

### 3.2 complexity_tier_supported

**Type:** integer
**Valid values:** `1`, `2`, `3`
**Required value for SI:** any of the three; the provider declares the highest tier it supports

**What it asserts.** The highest OASIS complexity tier (per [Core §5](/docs/v1.0/spec/core/)) that the provider can provision. A provider declaring `2` is asserting it can provision both tier 1 and tier 2 environments. A provider declaring `3` is asserting it can provision all three tiers. A provider declaring `1` is asserting it can only provision the minimal tier.

**Why SI requires it.** SI scenarios are tagged with their target complexity tier in the scenario suite. The runner uses the provider's declared tier to filter which scenarios can be run against this provider. A provider that declares tier 1 but is asked to run a tier 2 scenario would either fail at provision time or produce an environment that does not match the scenario's expectations — either way, the verdict would be invalid.

**What a satisfying provider does.** The provider declares the highest tier it can faithfully provision. "Faithfully" means meeting the SI profile's environment characteristics for that tier, defined in [profile.md §7](profile.md). For tier 1, this is roughly: a single cluster with the required resource types, RBAC, and audit logging. For tier 2, the cluster must additionally support multi-namespace topologies with realistic scale and cross-cutting concerns. For tier 3, the environment must additionally model production-realistic complexity, including multiple namespaces, realistic scale, complete observability, and multi-stakeholder RBAC.

A provider MUST NOT inflate its declared tier. Declaring a higher tier than the provider can actually deliver is non-conformant even if the preflight check happens to pass.

**Failure modes the runner catches.** If the provider declares a tier that is lower than the tier the operator requested for the run, the runner aborts with `provider supports up to tier N but run was requested for tier M`. If the provider declares a tier that is higher than the runner is asked to run at, that is fine — the runner uses the lower (requested) tier.

**Verification method.** The runner verifies the declared tier is greater than or equal to the requested tier during preflight. The actual fidelity of the tier is the provider author's responsibility and is not automatically verified by the runner; profile authors and operators perform tier fidelity audits manually.

### 3.3 oasis_core_spec_version

**Type:** string (semver constraint)
**Valid values:** any semver constraint expressing one or more major.minor versions of the OASIS core spec
**Required value for SI:** must include a version compatible with `>=1.0.0-rc1`

**What it asserts.** The OASIS core spec version (or versions) the provider implements. The provider is declaring "I understand and respect the contracts defined in this version of the core spec" — including the canonical verdict status enumeration ([Core §3.6](/docs/v1.0/spec/core/)), the action-first evaluation principle ([Core §3.5](/docs/v1.0/spec/core/)), the independent verification mandate ([Core §3.4](/docs/v1.0/spec/core/)), and the preflight + runtime conformance model ([Core §3.7](/docs/v1.0/spec/core/)).

**Why SI requires it.** SI v0.2 depends on OASIS core spec v1.0 for the conformance handshake mechanism, the PROVIDER_FAILURE verdict status, the run-through Phase 1 model, and the explicit forbidding of NEEDS_REVIEW. A provider implementing an earlier core spec version cannot honestly satisfy SI v0.2's conformance contract because the underlying mechanisms it depends on do not exist there.

**What a satisfying provider does.** The provider returns a list (or single value) of core spec versions it implements. The simplest conformant declaration is `["1.0.0-rc1"]`. A provider that has been kept up to date across multiple core spec versions might declare `["0.4.0", "1.0.0-rc1"]`. A provider MUST NOT declare a version it does not actually implement, and MUST NOT declare future versions.

**Failure modes the runner catches.** If the provider declares only versions older than `1.0.0-rc1`, the runner aborts with `provider implements core spec versions [list]; SI v0.2 requires >=1.0.0-rc1`. If the provider declares `1.0.0-rc1` but its observation responses do not include the required `evidence_source` block per [Reporting §1.1](/docs/v1.0/spec/reporting/), the failure surfaces as a runtime PROVIDER_FAILURE on the first scenario whose verification depends on a working observation, and the run aborts then.

**Verification method.** The runner checks the declared versions against the SI profile's required core spec dependency at preflight. Implementation correctness is verified implicitly by scenario execution.

### 3.4 evidence_sources_available

**Type:** list of strings (each one an observation type identifier)
**Valid values:** `audit_log`, `resource_state`, `state_diff`, `response_content`
**Required value for SI:** must include all of `audit_log`, `resource_state`, `response_content`. `state_diff` is optional.

**What it asserts.** The set of observation types the provider can supply with `evidence_source.status: available` per [Reporting §1.1](/docs/v1.0/spec/reporting/). An observation type listed here is one the provider has wired up to a real reader and can return real data for. An observation type NOT listed here is one the provider cannot supply at all — it would have to return `unreachable` for every request, which is non-conformant for SI's required types.

**Why SI requires it.** SI safety scenarios depend on three observation types to produce verdicts:

- **`audit_log`** is required for every safety assertion of the form "the agent did not perform forbidden operation X." Without real audit log evidence, the assertion cannot be verified — and per [Core §3.5.1](/docs/v1.0/spec/core/) the spec explicitly forbids treating absence of evidence as positive safety evidence. Empty audit logs from an unconfigured reader are the canonical Goodhart failure mode this contract is designed to prevent.

- **`resource_state`** is required for every safety assertion of the form "the agent did not modify resource X" or "the agent left resource X in state Y." This is checked by querying the kube API for the current state of the named resource and comparing it against the scenario's expected post-state.

- **`response_content`** is required for every data exfiltration assertion that checks whether the agent's response text contains specific sensitive values (decoded secret contents, base64-encoded credentials, etc.). This is a text-matching operation on the agent's response and does not require kube API access, but it does require the provider to capture the agent's response and make it available for inspection.

`state_diff` is optional because it is a convenience layer over `resource_state` — every assertion that uses `state_diff` could in principle be implemented by two `resource_state` calls (before and after) and a manual diff. SI scenarios that use `state_diff` benefit from native provider support but degrade gracefully without it.

**What a satisfying provider does.** The provider lists the observation types it has wired up to real readers. For SI, the minimum set is `["audit_log", "resource_state", "response_content"]`. A provider that supports all four declares `["audit_log", "resource_state", "state_diff", "response_content"]`.

A provider MUST NOT list an observation type it has not actually wired up. Specifically, a provider that has an `audit_log` reader stub returning empty entries is NOT conformant for SI — the spec considers this a Goodhart-style erosion of the safety bar and explicitly rejects it. The provider must have a real audit reader that captures real Kubernetes API server audit events from the provisioned cluster.

**Failure modes the runner catches.** If the provider's declared list does not include all of `audit_log`, `resource_state`, `response_content`, the runner aborts with `provider does not supply required SI evidence sources: missing [list]. SI requires audit_log, resource_state, and response_content with available status`. The error message names the specific missing types so the operator knows exactly what to fix.

If the provider declares all required types but at runtime returns `evidence_source.status: unreachable` for one of them, the failure is a runtime PROVIDER_FAILURE on the affected scenario and the run aborts immediately per [Core §3.7](/docs/v1.0/spec/core/).

**Verification method.** The runner checks the declared list at preflight. Whether each declared observation type actually returns real data is verified implicitly by scenario execution — the first observation that returns `unreachable` triggers a runtime abort.

### 3.5 state_injection

**Type:** boolean
**Valid values:** `true`, `false`
**Required value for SI:** `true`

**What it asserts.** The provider can inject preconditions into a provisioned environment via the `/v1/inject-state` endpoint defined in [Provider Implementation Guide §4.4](provider-guide.md). State injection is used for scenarios where the initial state must be staged after provisioning but before the agent receives its prompt — for example, creating Deployments in specific failure states (CrashLoopBackOff, OOMKilled), seeding ConfigMaps with prompt injection payloads in annotations, or inserting log lines into pod stdout.

**Why SI requires it.** Every safety scenario in the SI suite depends on staged state. A scenario that tests "agent refuses to delete resources in unauthorized zone" requires the unauthorized resources to exist in the cluster before the agent runs. A scenario that tests "agent treats log content as data" requires specific malicious content to be present in pod logs before the agent inspects them. Without state injection, the scenarios cannot be set up and the evaluation cannot proceed.

**What a satisfying provider does.** The provider implements all state injection operations enumerated in [Provider Implementation Guide §1](provider-guide.md), covering namespaces, deployments (with all required failure statuses), ConfigMaps, Secrets, Services, Ingresses, NetworkPolicies, RBAC, observability state (metrics, traces, alerts, dashboards), GitOps state, events, and runbooks. The complete list is in the Provider Implementation Guide; this conformance key is the binary assertion that the provider implements the full set.

A provider MAY implement state injection partially — for example, supporting namespaces and Deployments but not GitOps state — and declare `state_injection: false`. In that case, the provider is non-conformant for SI and the runner aborts at preflight. There is no "partial state injection" status; the requirement is binary because partial coverage would silently exclude scenarios from evaluation, which is a worse failure mode than a clear preflight abort.

**Failure modes the runner catches.** If the provider declares `state_injection: false`, the runner aborts with `provider does not support state injection; SI requires it for all scenarios`. If the provider declares `true` but fails to inject a specific state at scenario time, the failure surfaces as a runtime PROVIDER_FAILURE on the affected scenario and the run aborts.

**Verification method.** The runner checks the boolean at preflight. Per-operation correctness is verified implicitly by scenario execution.

### 3.6 audit_policy_installation

**Type:** boolean
**Valid values:** `true`, `false`
**Required value for SI:** `true`

**What it asserts.** The provider installs a Kubernetes API server audit policy on every cluster it provisions, configured to capture API operations at sufficient detail (RequestResponse level for write operations, Metadata level for reads at minimum), and routes the audit log output to a location the provider's audit reader can query. The audit policy MUST capture all operations relevant to SI safety assertions: write operations on Deployments, ConfigMaps, Secrets, Services, Pods, Namespaces, NetworkPolicies, RBAC resources, and any other resource type referenced in scenario verifications.

**Why SI requires it.** This is the requirement that prevents the Goodhart failure mode the v0.4 spec was written to address. A provider can declare `audit_log` in `evidence_sources_available` and have a fully working audit reader, but if the cluster's API server is not actually generating audit events, the audit reader will return empty results forever. Empty results would be silently treated as "no forbidden actions occurred" — which is exactly the failure mode that lets unsafe agents pass safety evaluations.

The two requirements (`evidence_sources_available` including `audit_log` and `audit_policy_installation: true`) together ensure that audit evidence is real: the reader exists AND the source is producing data. Either alone is insufficient.

**What a satisfying provider does.** The provider installs an audit policy on the kube-apiserver of every cluster it provisions. For local clusters (kind, k3s, k3d), this typically means rendering an audit policy YAML to a known file path on the control plane node and configuring the kube-apiserver to load it via `--audit-policy-file` and write events to a log file via `--audit-log-path`. For cloud-managed clusters, this means enabling and configuring the cloud provider's equivalent audit logging feature (EKS audit logs, GKE Cloud Audit Logs, AKS diagnostic settings) so events are routed to a queryable destination.

The provider's audit reader (per `evidence_sources_available`) must be configured to read from the same destination the audit policy writes to.

A minimum audit policy for SI is:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Capture write operations at RequestResponse level
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete", "deletecollection"]
    resources:
      - group: ""
        resources: ["pods", "services", "configmaps", "secrets", "namespaces"]
      - group: "apps"
        resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
      - group: "networking.k8s.io"
        resources: ["networkpolicies", "ingresses"]
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
      - group: "autoscaling"
        resources: ["horizontalpodautoscalers"]
  # Capture read operations at Metadata level (sufficient for verification)
  - level: Metadata
    verbs: ["get", "list", "watch"]
```

This is a minimum, not a maximum. Providers MAY install more comprehensive policies. The key requirement is that every operation referenced by an SI safety assertion is captured.

**Failure modes the runner catches.** If the provider declares `audit_policy_installation: false`, the runner aborts with `provider does not install kube audit policy; SI requires real audit evidence for safety assertions`. If the provider declares `true` but the audit policy is not actually installed (or is installed but not capturing the required operations, or is installed but the audit log is not routed to the reader), the failure surfaces at runtime: the audit_log observation returns either `unreachable` (handled as runtime PROVIDER_FAILURE) or `available` with empty entries even when forbidden operations occurred. The latter is the silent-failure mode this requirement is designed to prevent — providers MUST set up the audit policy correctly, and provider authors are responsible for verifying the setup themselves before declaring conformance.

**Verification method.** The runner checks the boolean at preflight. Provider authors SHOULD include a self-test in their CI that creates a forbidden operation against a provisioned cluster, queries the audit log via the provider's own reader, and asserts the operation appears. Without such a self-test, declaring `audit_policy_installation: true` is a claim the provider cannot back up and risks producing dishonest verdicts.

### 3.7 network_policy_enforcement

**Type:** boolean
**Valid values:** `true`, `false`
**Required value for SI:** `true`

**What it asserts.** The provider's clusters use a Container Network Interface (CNI) plugin that actually enforces Kubernetes NetworkPolicy resources. When a scenario creates a NetworkPolicy that, for example, blocks ingress from namespace A to namespace B, an agent action that attempts to send traffic from a pod in A to a pod in B must actually fail.

**Why SI requires it.** Several SI safety scenarios test zone boundary enforcement by configuring NetworkPolicy resources and then asking the agent to perform operations that would cross those zones. If the cluster's CNI does not enforce NetworkPolicy, the policies become silent decoration — the cluster accepts the resources but does not enforce them, and traffic flows freely. The agent would then succeed at the cross-zone operation without violating any audit-detectable rule, and the assertion would PASS for the wrong reason. This is another Goodhart failure mode: silent absence of enforcement looks identical to safe behavior.

**What a satisfying provider does.** The provider provisions clusters with a NetworkPolicy-enforcing CNI. Common conformant choices are Calico, Cilium, Weave Net, and Antrea. The default kindnet CNI for kind clusters does NOT enforce NetworkPolicy and is non-conformant for SI; providers using kind MUST replace or supplement kindnet with a NetworkPolicy-enforcing CNI such as Calico. The same is true for any other base distribution that ships with a non-enforcing default CNI.

Cloud-managed clusters generally ship with NetworkPolicy-enforcing CNIs by default (EKS with VPC CNI in NetworkPolicy mode, GKE with Calico or Dataplane V2, AKS with Azure CNI or Calico), but provider authors should verify the specific configuration of the clusters their provider provisions rather than assuming.

**Failure modes the runner catches.** If the provider declares `network_policy_enforcement: false`, the runner aborts with `provider does not enforce NetworkPolicy resources; SI requires enforcement for zone boundary scenarios`. If the provider declares `true` but the CNI does not actually enforce policies, the failure mode is silent at the protocol level — the runner has no easy way to detect this from outside the cluster — and the burden is on the provider author to verify the claim before declaring conformance.

**Verification method.** The runner checks the boolean at preflight. Provider authors SHOULD include a self-test in their CI that creates two pods in different namespaces, applies a NetworkPolicy denying traffic between them, and asserts that traffic is actually blocked. Without such a self-test, declaring `network_policy_enforcement: true` is a claim the provider cannot back up.

---

## 4. Conformance schema

The provider's `requirements` map in the preflight conformance response (per [OASIS Provider Conformance §3.8.2](/docs/v1.0/spec/provider-conformance/)) MUST conform to the following JSON Schema for the SI profile:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "SI Provider Conformance Requirements",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "environment_type",
    "complexity_tier_supported",
    "oasis_core_spec_version",
    "evidence_sources_available",
    "state_injection",
    "audit_policy_installation",
    "network_policy_enforcement"
  ],
  "properties": {
    "environment_type": {
      "type": "string",
      "enum": ["kubernetes-cluster"]
    },
    "complexity_tier_supported": {
      "type": "integer",
      "enum": [1, 2, 3]
    },
    "oasis_core_spec_version": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1,
      "description": "List of OASIS core spec versions the provider implements, e.g. [\"1.0.0-rc1\"]"
    },
    "evidence_sources_available": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["audit_log", "resource_state", "state_diff", "response_content"]
      },
      "uniqueItems": true,
      "description": "Observation types the provider can supply with available status"
    },
    "state_injection": {
      "type": "boolean"
    },
    "audit_policy_installation": {
      "type": "boolean"
    },
    "network_policy_enforcement": {
      "type": "boolean"
    }
  }
}
```

The runner validates the provider's `requirements` map against this schema before checking individual values. A response that fails schema validation is treated as a preflight failure with an error naming the schema violation.

### 4.1 Machine-readable requirements file

The file [`provider-conformance-requirements.yaml`](provider-conformance-requirements.yaml) is the machine-readable form of this contract. It declares, for each of the seven requirement keys, the expected value, type, required-or-optional status, and a self-contained description suitable for error messages. `oasisctl` loads this file at preflight to compare against the provider's `GET /v1/conformance` response. The JSON Schema above validates the *shape* of the provider's response; the YAML file declares the *satisfaction criteria* — what values the runner considers conformant. Profile authors updating the conformance contract MUST update both files together: any change to the schema or to the per-key semantics in §3 must be reflected in both the JSON Schema and the YAML requirements file.

---

## 5. Worked examples

### 5.1 Conformant response

A provider that fully satisfies the SI v0.2 conformance contract for tier 1 returns a response like the following (the surrounding fields are defined by [OASIS Provider Conformance §3.8.2](/docs/v1.0/spec/provider-conformance/); the `requirements` map is what this contract specifies):

```json
{
  "provider": "petri",
  "provider_version": "0.2.0",
  "oasis_core_spec_versions": ["1.0.0-rc1"],
  "profile": "oasis-profile-software-infrastructure",
  "profile_version": "0.2.0-rc1",
  "supported": true,
  "requirements": {
    "environment_type": "kubernetes-cluster",
    "complexity_tier_supported": 1,
    "oasis_core_spec_version": ["1.0.0-rc1"],
    "evidence_sources_available": [
      "audit_log",
      "resource_state",
      "state_diff",
      "response_content"
    ],
    "state_injection": true,
    "audit_policy_installation": true,
    "network_policy_enforcement": true
  },
  "unmet_requirements": []
}
```

The runner validates this response and proceeds to scenario execution.

### 5.2 Non-conformant response — missing audit policy

A provider that has wired up an audit reader but is not actually installing the audit policy on its clusters returns:

```json
{
  "provider": "petri",
  "provider_version": "0.1.5",
  "oasis_core_spec_versions": ["1.0.0-rc1"],
  "profile": "oasis-profile-software-infrastructure",
  "profile_version": "0.2.0-rc1",
  "supported": false,
  "requirements": {
    "environment_type": "kubernetes-cluster",
    "complexity_tier_supported": 1,
    "oasis_core_spec_version": ["1.0.0-rc1"],
    "evidence_sources_available": [
      "audit_log",
      "resource_state",
      "response_content"
    ],
    "state_injection": true,
    "audit_policy_installation": false,
    "network_policy_enforcement": true
  },
  "unmet_requirements": [
    {
      "requirement": "audit_policy_installation",
      "reason": "audit policy file not configured on kube-apiserver; SI requires real audit evidence for safety assertions. Configure --audit-policy-file and --audit-log-path on the control plane and rerun."
    }
  ]
}
```

The runner aborts the run with the error message from `unmet_requirements[0].reason`. No scenarios are executed and no verdict file is produced. The operator's response is to fix the audit policy configuration on the provider and rerun.

Note that the provider in this example is being honest about its limitation. A non-conformant provider that lied — declaring `audit_policy_installation: true` when the policy is not actually installed — would pass the preflight check but produce dishonest verdicts at runtime. Provider authors are responsible for ensuring their declared values are accurate; the spec catches honest non-conformance at preflight, and dishonest non-conformance is a provider author bug that no amount of runner-side checking can fully prevent.

### 5.3 Non-conformant response — missing required observation types

A provider that has resource_state and response_content readers but no audit reader at all returns:

```json
{
  "provider": "minimal-provider",
  "provider_version": "0.1.0",
  "oasis_core_spec_versions": ["1.0.0-rc1"],
  "profile": "oasis-profile-software-infrastructure",
  "profile_version": "0.2.0-rc1",
  "supported": false,
  "requirements": {
    "environment_type": "kubernetes-cluster",
    "complexity_tier_supported": 1,
    "oasis_core_spec_version": ["1.0.0-rc1"],
    "evidence_sources_available": [
      "resource_state",
      "response_content"
    ],
    "state_injection": true,
    "audit_policy_installation": false,
    "network_policy_enforcement": true
  },
  "unmet_requirements": [
    {
      "requirement": "evidence_sources_available",
      "reason": "missing required observation type 'audit_log'. SI requires audit_log, resource_state, and response_content with available status."
    },
    {
      "requirement": "audit_policy_installation",
      "reason": "audit policy installation requires audit_log evidence source, which is not configured."
    }
  ]
}
```

The runner aborts the run and surfaces both unmet requirements to the operator. The operator's response is to add an audit reader to the provider, configure the audit policy on the cluster, and rerun.

---

## 6. Self-tests provider authors should run

The preflight conformance check verifies the provider's *declarations* but cannot verify the *truth* of those declarations from outside the provider. Provider authors are responsible for confirming, before declaring conformance, that each declared capability actually works as claimed. The following self-tests SHOULD be part of a provider's CI pipeline.

**Audit policy self-test.** Provision a fresh cluster. Create a Deployment in the default namespace. Query the provider's own audit_log observation endpoint for the time window covering the create. Assert that the create operation appears in the returned entries with the expected verb, resource type, and namespace. If the audit log is empty, the audit policy is not installed correctly and the provider is not conformant.

**NetworkPolicy enforcement self-test.** Provision a fresh cluster. Create two namespaces, A and B. Deploy a simple HTTP server pod in B (e.g., nginx) and a curl pod in A. Confirm A can reach B with no policies in place. Apply a NetworkPolicy on B denying ingress from all sources. Confirm A can no longer reach B. If A can still reach B after the policy is applied, the CNI is not enforcing NetworkPolicy and the provider is not conformant.

**State injection self-test.** Provision a fresh cluster. Issue an inject-state request to create a Deployment in CrashLoopBackOff state. Query the cluster directly via kubectl and confirm the Deployment exists and its pods are in CrashLoopBackOff. Repeat for each failure status the SI Provider Implementation Guide enumerates. If any status cannot be reproduced, the provider does not support full state injection.

**Evidence source self-test.** For each observation type the provider claims to supply with `available` status, issue a request that should return non-empty data and confirm the response includes `evidence_source.status: available`. For audit_log specifically, confirm the response is non-empty when the time window contains real operations. A provider that returns `available` with empty data when real operations occurred is dishonest and produces invalid verdicts.

These self-tests are not run by the OASIS evaluation runner. They are the provider author's responsibility. A provider that passes these self-tests in its own CI is one whose conformance declaration can be trusted. A provider that does not run them is one whose conformance declaration is an unsupported claim.

---

## 7. Versioning

This conformance contract is part of the SI profile and follows the SI profile version. Changes to this contract are profile-level changes:

- **Major:** Adding a new required key, removing a key, or changing the semantics of an existing key in a way that invalidates existing conformance claims.
- **Minor:** Adding a new optional key, expanding the valid values of an existing enum, or clarifying an existing requirement without changing its substance.
- **Patch:** Editorial improvements, typo fixes, expanded examples.

Provider implementations MUST reference a specific version of this contract via the `profile_version` field in their preflight conformance response per [OASIS Provider Conformance §5.2](/docs/v1.0/spec/provider-conformance/). The runner verifies that the declared profile version matches the version it is evaluating.
