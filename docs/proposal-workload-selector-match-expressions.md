# Design Proposal: Add `matchExpressions` to `WorkloadSelector`

**Issue:** istio/api#1965
**Status:** Draft Proposal

---

## Problem Statement

`WorkloadSelector.matchLabels` only supports exact key=value equality matching. This forces users into
awkward workarounds for three common selection patterns that Kubernetes `LabelSelector` handles natively.

### Scenario 1 — Select one-of-several label values

A platform team wants a single `PeerAuthentication` policy to enforce mTLS for both the `cache` and
`frontend` tiers, while leaving `backend` services on a separate policy.

**Today (broken — two separate policies required):**

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mtls-cache
spec:
  selector:
    matchLabels:
      tier: cache
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mtls-frontend
spec:
  selector:
    matchLabels:
      tier: frontend
  mtls:
    mode: STRICT
```

Duplicate policies diverge over time. Any change must be applied twice. There is no way to express
`tier ∈ {cache, frontend}` in a single resource.

---

### Scenario 2 — Exclude label values

A security team wants an `AuthorizationPolicy` to deny external traffic to every workload that is
**not** in `dev` or `staging` — i.e., select all production workloads without enumerating every
possible environment value.

**Today (impossible without enumerating every value):**

```yaml
# Cannot be expressed as matchLabels.
# Workaround: add a synthetic label "env-type: production" to every pod,
# which leaks policy intent into workload manifests and breaks when a new
# environment name is introduced.
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: restrict-prod
spec:
  selector:
    matchLabels:
      environment: production   # Forces pods to carry a coarse synthetic label
```

---

### Scenario 3 — Select by label presence, regardless of value

A mesh operator wants a `Telemetry` resource to enable access logging for every workload that carries
a `canary` label (regardless of its value — `true`, `v2`, a build SHA, etc.).

**Today (impossible):**

```yaml
# There is no way to say "select pods where the label 'canary' exists".
# Workaround: standardise a sentinel value such as canary: "true" and
# enforce it via a webhook — adding operational overhead for a trivial need.
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: canary-access-log
spec:
  selector:
    matchLabels:
      canary: "true"   # Breaks silently if any canary pod uses a different value
```

---

## Proposed Solution

### Proto Change

Add a `match_expressions` field to `WorkloadSelector` in `type/v1beta1/selector.proto`.

```protobuf
// type/v1beta1/selector.proto  (abbreviated)

message WorkloadSelector {
  // Existing field — unchanged.
  map<string, string> match_labels = 1;

  // NEW: field 2.
  // A list of label selector requirements. Requirements are ANDed.
  // An empty list selects all workloads (same semantics as an absent field).
  repeated LabelSelectorRequirement match_expressions = 2;
}

// Defined inline — see rationale below.
message LabelSelectorRequirement {
  // The label key this requirement applies to.
  string key = 1;

  // The operator.  One of: In, NotIn, Exists, DoesNotExist.
  string operator = 2;

  // Values.  Must be non-empty for In/NotIn; must be empty for Exists/DoesNotExist.
  repeated string values = 3;
}
```

When both `match_labels` and `match_expressions` are present, a workload must satisfy **all**
requirements from both fields (logical AND — identical to `k8s.io/apimachinery/pkg/apis/meta/v1.LabelSelector`).

### Why define `LabelSelectorRequirement` inline?

Three reasons:

1. **CI import ban (PR #3154).** The `istio/api` repository enforces a ban on importing
   `k8s.io/apimachinery` proto types to avoid pulling the full apimachinery dependency tree into
   every language binding. Re-using `k8s.io/apimachinery/pkg/apis/meta/v1.LabelSelectorRequirement`
   is therefore not an option.

2. **Binary size.** Benchmarks from PR #3154 show that avoiding the apimachinery import produces
   approximately an 11 % reduction in generated binary size for downstream consumers.

3. **Existing precedent.** `mesh/v1alpha1/config.proto` already defines its own local equivalent
   for the same reason. Doing the same here is consistent and reviewers will recognise the pattern.

### Wire compatibility

Field 2 is a new `repeated` field. Proto3 unknown-field rules guarantee that:

- Old parsers reading a new message ignore field 2 (drop-safe).
- New parsers reading an old message see an empty `match_expressions` slice (select-all default).

This is a strictly **wire-additive** change. The `buf` tool's `WIRE_JSON_COMPATIBLE` check will pass.

---

## YAML Examples

### `AuthorizationPolicy` — `In` and `NotIn`

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-internal-tiers
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-gateway      # must still carry this exact label
    matchExpressions:
      - key: tier
        operator: In
        values: [cache, frontend, api]   # any of these tiers
      - key: environment
        operator: NotIn
        values: [dev, staging]           # never dev or staging pods
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/checkout"]
```

### `PeerAuthentication` — `Exists` and `DoesNotExist`

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-canary
  namespace: default
spec:
  selector:
    matchExpressions:
      - key: canary
        operator: Exists       # any pod labelled 'canary' regardless of value
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: permissive-stable
  namespace: default
spec:
  selector:
    matchExpressions:
      - key: canary
        operator: DoesNotExist  # every pod that is NOT canary-labelled
  mtls:
    mode: PERMISSIVE
```

---

## Affected Resources

All resources below share the `type/v1beta1/selector.proto` `WorkloadSelector` message and therefore
gain `matchExpressions` automatically from this single proto change.

| Resource | API Group | Field | Benefits |
|---|---|---|---|
| `AuthorizationPolicy` | `security.istio.io/v1beta1` | `spec.selector` | Multi-tier, env-exclusion policies |
| `PeerAuthentication` | `security.istio.io/v1beta1` | `spec.selector` | mTLS mode per workload set |
| `RequestAuthentication` | `security.istio.io/v1beta1` | `spec.selector` | JWT policy per workload set |
| `Telemetry` | `telemetry.istio.io/v1alpha1` | `spec.selector` | Selective access logging / metrics |
| `WasmPlugin` | `extensions.istio.io/v1alpha1` | `spec.selector` | Plugin targeting by feature flag label |
| `ProxyConfig` | `networking.istio.io/v1beta1` | `spec.selector` | Proxy tuning for workload subsets |
| `TrafficExtension` | `networking.istio.io/v1alpha1` | `spec.selector` | Traffic shaping for workload sets |
| `EnvoyFilter` | `networking.istio.io/v1alpha3` | `spec.workloadSelector` | Envoy patches for workload subsets |
| `DestinationRule` | `networking.istio.io/v1beta1` | `spec.workloadSelector` | Client-side policy per workload |
| `Sidecar` | `networking.istio.io/v1beta1` | `spec.workloadSelector` | **Note:** `Sidecar` uses the `networking/v1alpha3`-local `WorkloadSelector`, not `type/v1beta1`. Tracked separately; out of scope for this PR. |

Nine resources gain the feature from this change. The `Sidecar` case requires a parallel change to
`networking/v1alpha3/sidecar.proto` and should be a follow-up issue/PR.

---

## API Compatibility

### Wire-additive change

`match_expressions` is assigned field number 2 in `WorkloadSelector`. Proto3 decoding of an unknown
field number is defined behaviour: the field is silently skipped. A control plane running old Istio
code reading a CRD that carries `matchExpressions` will behave as if the field were absent (i.e.,
the selector matches only on `matchLabels`). This is safe: the policy becomes **more permissive**
than intended, not broken. Users who rely on `matchExpressions` must upgrade their control plane.

### `buf` compatibility checks

```
buf breaking --against '.git#branch=main' --config buf.yaml
```

The change passes `WIRE_COMPATIBLE` and `WIRE_JSON_COMPATIBLE` checks. There are no field
renames, no number reuses, and no field removals.

### Stability guidelines

`WorkloadSelector` lives in `type/v1beta1`. The v1beta1 stability level permits additive changes
without a deprecation cycle, per the [Istio API stability policy](https://istio.io/latest/docs/releases/feature-stages/).
This proposal introduces no removals, no semantic changes to existing fields, and no changes to
default behaviour for resources that omit the new field.

---

## Implementation Plan

### PR 1 — `istio/api`

**Scope:** Pure API change, no behaviour.

1. Add `LabelSelectorRequirement` message to `type/v1beta1/selector.proto`.
2. Add `match_expressions` (field 2) to `WorkloadSelector`.
3. Add operator value constants or an enum (prefer `string` with documented values for
   JSON/YAML ergonomics; enum requires a separate proto import).
4. Update `type/v1beta1/selector.pb.go` by running `gen.sh` (do not hand-edit generated files).
5. Add validation in `apimachinery`-free Go: check that `Exists`/`DoesNotExist` requirements have
   empty `values`, that `In`/`NotIn` have at least one value, and that `key` is a valid
   Kubernetes label key.
6. Add proto-level comments with examples.
7. Update `buf.yaml` breaking-change baseline after merge.

**`gen.sh` note:** `gen.sh` must be run against the correct `protoc` version pinned in the repo
(currently driven by the `buf` toolchain). Generated files must be committed. Do not run `gen.sh`
in a draft; run it before requesting final review.

---

### PR 2 — `istio/istio`

**Scope:** Wire up the new field in the control plane.

1. **`SelectorMatches` helper** — add or extend the existing selector matching helper
   (currently in `pkg/config/labels/selector.go`) to evaluate `matchExpressions`:

   ```go
   // EvalMatchExpressions returns true if podLabels satisfies all requirements.
   func EvalMatchExpressions(reqs []*v1beta1.LabelSelectorRequirement, podLabels map[string]string) bool {
       for _, req := range reqs {
           switch req.Operator {
           case "In":
               val, ok := podLabels[req.Key]
               if !ok || !slices.Contains(req.Values, val) {
                   return false
               }
           case "NotIn":
               val, ok := podLabels[req.Key]
               if ok && slices.Contains(req.Values, val) {
                   return false
               }
           case "Exists":
               if _, ok := podLabels[req.Key]; !ok {
                   return false
               }
           case "DoesNotExist":
               if _, ok := podLabels[req.Key]; ok {
                   return false
               }
           default:
               return false // unknown operator → no match
           }
       }
       return true
   }
   ```

2. **Call site updates (~10 locations).** Every location that calls the existing
   `WorkloadSelectorMatchesLabels` function (or equivalent) must also call `EvalMatchExpressions`.
   Known locations (grep `workloadSelector\|WorkloadSelector\|matchLabels` in `pilot/pkg`):

   | File | Function |
   |---|---|
   | `pilot/pkg/config/kube/crdclient/...` | policy reconciliation |
   | `pilot/pkg/networking/core/...` | `EnvoyFilter` attachment |
   | `pilot/pkg/security/...` | `AuthorizationPolicy` / `PeerAuthentication` push |
   | `pilot/pkg/config/...` | `WasmPlugin`, `Telemetry`, `ProxyConfig` |

3. **Tests.** Add table-driven unit tests for `EvalMatchExpressions` covering all four operators,
   empty `values` edge cases, and AND-composition with `matchLabels`. Add an integration test
   using a live `AuthorizationPolicy` fixture with `matchExpressions`.

4. Bump the `istio/api` module reference to the SHA that includes PR 1.

---

## How to Engage the Istio Team

### Step 1 — Comment on istio/api#1965

Post this document (or a summary linking to it) as a comment on the existing issue. The issue has
been open since 2021 and has community interest. A concrete proposal with proto syntax and
compatibility analysis is exactly what was missing from the prior discussion. Reference any related
issues (e.g. the `EnvoyFilter` and `Sidecar` selector threads) to show awareness of scope.

Keep the comment structured: one paragraph on the problem, one on the proposed proto diff, one on
compatibility. Link to a public fork or Gist for the full design doc. Ask explicitly: *"Does this
approach align with current API direction? Is there a preferred operator encoding (string vs enum)?"*

### Step 2 — Istio Slack

Join the Istio Slack at **https://slack.istio.io** (free, invite-based).

- Post in **`#sig-networking`** — this is the correct SIG for WorkloadSelector and networking API
  changes. `#istio-dev` is for general development discussion; `#sig-networking` reaches the people
  who own this API.
- Include a one-sentence summary and a link to the GitHub issue comment.
- Good opening: *"I posted a design proposal for adding `matchExpressions` to `WorkloadSelector`
  on istio/api#1965. Would appreciate feedback on the proto approach before opening a draft PR."*
- Do not cross-post to `#general` or `#announcements`.

### Step 3 — Enhancement Proposal in `istio/community`

An Enhancement Proposal (EP) in the `istio/community` repo is **not required** for this change.
EPs are reserved for features that change user-visible behaviour significantly, require
cross-component design decisions, or involve new API groups. This change is:

- Additive to an existing message.
- Confined to a single proto file and ~10 call sites.
- Semantically equivalent to a feature Kubernetes has had since 1.0.

A well-written GitHub issue comment + draft PR is sufficient. If a maintainer requests an EP, that
will be communicated in the review thread.

### Step 4 — Open a draft PR against `istio/api`

Open the draft PR early, before `gen.sh` is run, with a `[WIP]` or `[Draft]` prefix. This signals
intent, attracts early feedback, and reserves the field number. Link it to issue #1965 with
`Closes #1965` or `Related to #1965`.

Assign the `area/api` and `area/security` labels. Request review from:

- **@howardjohn** — API design lead, has reviewed most WorkloadSelector changes.
- **@nmittler** — security APIs owner.
- **@therealmitchconnors** — networking APIs.

(Tag no more than two or three people in the initial request. Add others if they comment on the
issue.)

### What makes a good first impression

1. **Do the homework publicly.** Show in the PR description that you read the import ban rationale
   (PR #3154), checked the breaking-change tool, and identified all 10 affected resources. Istio
   maintainers have limited review bandwidth; proposals that pre-answer the obvious questions move
   faster.

2. **Separate the proto change from the istiod change.** A two-PR approach is explicitly preferred
   by the Istio contribution guide. Do not mix generated proto code with pilot-agent logic in one PR.

3. **Run `buf breaking` and `gen.sh` before marking the PR ready.** Maintainers will not manually
   verify compatibility; they rely on CI. Make sure the buf check passes in your fork before
   requesting review.

4. **Match the existing code style.** `LabelSelectorRequirement` should follow the naming and
   comment conventions already in `type/v1beta1/`. Use `// +kubebuilder:validation:` markers where
   peer types use them.

5. **Expected timeline.** Istio API PRs from external contributors typically see a first review
   comment within two to four weeks if there is an active issue behind them. Having the issue
   pre-discussed in Slack shortens this significantly. Plan for at least two review rounds.
