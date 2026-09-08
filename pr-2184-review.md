# Code Review — PR #2184: `feat: add agent runtimes (Sandbox) support`

- **PR:** https://github.com/eclipse-che/che-operator/pull/2184
- **Base:** `main` · **Commits:** 4 · **Ref reviewed:** `refs/pull/2184/head`
- **Date:** 2026-09-08
- **Verification:** built the branch and ran the new tests — they pass. Findings below were derived from code reading plus one reproduction (finding 1).

## Design concern — singleton Sandbox vs. per-user model

**This is the most significant issue with the PR, and it sits above the individual findings below: the CR models a single, cluster-global Sandbox, whereas Sandboxes are naturally per-user, like CDEs / DevWorkspaces.**

### What the PR actually builds

`spec.agentRuntimes` is a singleton block — `{enabled, namespace, image, runtimeClassName}` — and the reconciler creates **exactly one** Sandbox:

- `getSandboxName()` returns a **constant** (`defaults.GetCheFlavor() + "agent-runtimes"`) — no user, workspace or session in the name, so a second Sandbox can never exist.
- It is created in one admin-chosen `spec.agentRuntimes.namespace`, not in a user namespace.
- It is driven by the **CheCluster** reconcile loop, which runs once per CheCluster.
- Nothing under `controllers/usernamespace/` is touched — the diff is 16 non-generated files and none of them do per-user provisioning.

Net result: one cluster-global Sandbox pod running `/bin/bash -c "sleep infinity"`, shared, with no consumer wired to it.

### Why this mismatches the upstream Sandbox CR

`agents.x-k8s.io` Sandbox is an *instance* abstraction — one isolated pod for one agent session. The knobs this PR sets confirm that: `Lifecycle.ShutdownPolicy: Delete` and the `OperatingMode: Running/Suspended` field (see finding 2) are per-session lifecycle controls. They only mean something if something creates, suspends and reaps sandboxes per user/session. Managed as a singleton by an install-time CR, `ShutdownPolicy: Delete` has nothing to trigger it.

Finding 5 is the same tension surfacing elsewhere: `cmd/main.go` adds Sandbox to the cached scheme with **no** `part-of` label selector, unlike every other watched type. The cache is shaped for "there will be many Sandboxes in this cluster" while the reconciler manages one.

### Two readings

1. **Scaffolding / proof-of-concept.** Plausible: the container is a `sleep infinity` placeholder with no agent process, nothing exposes or consumes the Sandbox, and there is no auth or routing. If this is the case it should be stated in the PR description — because the API field is the part that is hard to walk back. `spec.agentRuntimes` ships into the CRD, the CSV, both `deploy/deployment` trees and the helm chart the moment this merges.
2. **Intended design.** If a single shared admin-managed runtime really is the goal (e.g. one sandbox the Che server brokers access to), the CR shape is defensible but the naming misleads: `agentRuntimes` (plural) reads like a *class* of runtimes that users get instances of.

### If per-user is the goal, most of this code does not survive

Per this repo's own conventions, per-user resources go through `controllers/usernamespace/`, not the CheCluster pipeline. A per-user design would move:

- the **name** (must include the user),
- the **namespace** (`<username>-che`, derived from `DefaultNamespace.Template`),
- the **trigger** (user-namespace event, or DevWorkspace start — not CheCluster reconcile),
- the **deletion path**, and
- the **cache selector**.

What would legitimately remain in `spec.agentRuntimes` is the *template*: image, runtimeClassName, resources — i.e. it becomes a defaults block like `devEnvironments.defaultComponents`, not an instance spec. Note that `namespace` would disappear entirely, which also retires the awkward "changing this orphans the Sandbox" caveat in that field's doc (see finding 12).

**Recommendation:** settle this before the API field merges, since the CRD is the part with a lasting compatibility cost.

---

## Summary

12 findings, plus the design concern above. Three findings block merge: a nil pointer panic that fires on **every existing CheCluster** (finding 1), an infinite `Update` loop against the Sandbox CR (finding 2), and an optional feature whose input validation stalls the whole reconcile pipeline (finding 3).

| # | Severity | File | Issue |
|---|----------|------|-------|
| 1 | Blocker | `pkg/deploy/agent-runtime/agent_runtime.go:100` | nil deref in `deleteSandbox` on default CheCluster |
| 2 | Blocker | `pkg/deploy/agent-runtime/agent_runtime.go:137` | Sandbox `Update` issued on every reconcile forever |
| 3 | Blocker | `pkg/deploy/agent-runtime/agent_runtime.go:120` | Bad optional-feature input blocks reconcile pipeline |
| 4 | Major | `pkg/deploy/agent-runtime/agent_runtime.go:73` | Missing agent-sandbox operator silently no-ops forever |
| 5 | Major | `cmd/main.go:352` | Cluster-wide unfiltered Sandbox informer in cache |
| 6 | Major | `pkg/common/infrastructure/cluster.go:106` | Sandbox CRD detection matches name, ignores API group |
| 7 | Minor | `pkg/deploy/agent-runtime/agent_runtime.go:96` | Uncached API discovery on every reconcile for all installs |
| 8 | Minor | `pkg/deploy/agent-runtime/agent_runtime.go:145` | Hardcoded bash command overrides image entrypoint |
| 9 | Minor | `build/scripts/clear-defined-test.sh:352` | `continue` targets `while` loop, duplicating retries/errors |
| 10 | Minor | `build/scripts/clear-defined-test.sh:305` | `recordError` stores post-substitution module name |
| 11 | Nit | `pkg/deploy/agent-runtime/agent_runtime.go:170` | `getSandboxName` missing separator: `cheagent-runtimes` |
| 12 | Nit | `api/v2/checluster_types.go:312` | Typo `hanging this value` in shipped CRD description |

---

## 1. Nil pointer dereference / operator panic — Blocker

**`pkg/deploy/agent-runtime/agent_runtime.go:100`**

`deleteSandbox` does `agentRuntimes := ctx.CheCluster.Spec.AgentRuntimes` and immediately reads `agentRuntimes.Namespace` with no nil check. `IsAgentRuntimesEnabled()` returns false when `Spec.AgentRuntimes == nil` — the default for every existing CheCluster — so `Reconcile` (line 48) falls straight into `deleteSandbox`.

**Failure scenario:** as soon as the agent-sandbox CRD is present in the cluster (`IsAgentSandboxEnabled` returns true), this segfaults on every reconcile. `Finalize` (line 64) hits the same path during CheCluster deletion. Reproduced with a throwaway test:

```
panic: runtime error: invalid memory address or nil pointer dereference
    ... deleteSandbox(agent_runtime.go:101)
```

**Fix:** `if agentRuntimes == nil || agentRuntimes.Namespace == "" { return true, nil }`

## 2. Infinite update loop against the Sandbox CR — Blocker

**`pkg/deploy/agent-runtime/agent_runtime.go:137`**

`SandboxSpec.OperatingMode` carries `+kubebuilder:default=Running` (`vendor/sigs.k8s.io/agent-sandbox/api/v1beta1/sandbox_types.go:218`) and is `omitempty`. The desired object built here leaves it as `""`, while `diffs.Sandbox` (`pkg/common/diffs/diffs.go:93`) only ignores `TypeMeta`/`ObjectMeta`/`Status` — the whole Spec is compared.

**Failure scenario:** the API server defaults the stored object to `Running`, `cmp.Diff` is therefore always non-empty, and `Sync` issues an `Update` on every single reconcile forever. Worse, if an admin (or the sandbox controller) sets `operatingMode: Suspended`, che-operator reverts it on the next loop.

**Fix:** set `OperatingMode: sandboxv1beta1.SandboxOperatingModeRunning` explicitly, or ignore that field in the diff opts.

> Note: the unit tests cannot catch this — the fake client does not apply CRD defaults.

## 3. Misconfigured optional feature blocks the whole reconcile pipeline — Blocker

**`pkg/deploy/agent-runtime/agent_runtime.go:120`**

`getSandboxSpec` errors when `image` or `namespace` is empty; `Reconcile` propagates that as `done=false`. The reconciler is registered at `controllers/che/checluster_controller.go:138` — before `containerbuild`, `consolelink` and `metrics`.

**Failure scenario:** `enabled: true` with no `image` stops container-build capabilities, the console link and the ServiceMonitor from ever reconciling, and keeps the CheCluster permanently un-ready.

**Fix:** these are user-input validations — move them to `api/v2/checluster_webhook.go`, or log and treat as a no-op.

## 4. Enabling the feature without the agent-sandbox operator silently succeeds — Major

**`pkg/deploy/agent-runtime/agent_runtime.go:73`**

`syncSandbox` logs and returns `nil`, so `Reconcile` returns `done=true` with no error and no `RequeueAfter`.

**Failure scenario:** the admin sets `agentRuntimes.enabled: true`, sees a healthy CheCluster, and no Sandbox is ever created. Because nothing requeues, installing the agent-sandbox operator afterwards never triggers creation either — it waits for an unrelated CheCluster event.

**Fix:** return an explicit error, as `pkg/deploy/image-puller/imagepuller.go:79` does in the equivalent situation.

## 5. Unfiltered cluster-wide Sandbox informer — Major

**`cmd/main.go:352`**

`sandboxv1beta1` is added to the scheme (line 135) and the reconciler uses the **cached** `ClusterAPI.ClientWrapper`, but `getCacheFunc()` adds no `cache.ByObject` selector for `&sandboxv1beta1.Sandbox{}`. Every other watched type is restricted to `app.kubernetes.io/part-of=che.eclipse.org`.

**Failure scenario:** the first `Get` starts an informer caching *every* Sandbox in the cluster — precisely the per-user objects the agent-sandbox operator creates — in che-operator memory, growing unbounded with user count.

**Fix:** add a `partOfEclipseChe` selector entry, or use `NonCachingClientWrapper` as the image-puller reconciler does for its CRD.

## 6. Group-blind CRD detection — Major

**`pkg/common/infrastructure/cluster.go:106`**

`hasAPIResource` (line 178) matches on resource *name* only, ignoring the API group. `sandboxes` is a generic plural.

**Failure scenario:** any unrelated CRD in the cluster exposing a `sandboxes` resource makes `IsAgentSandboxEnabled` return true, after which the operator tries to create an `agents.x-k8s.io/v1beta1` Sandbox that does not exist — and, per finding 1, the nil-deref path becomes reachable.

**Fix:** match on the group-version too.

## 7. Full API discovery on every reconcile — Minor

**`pkg/deploy/agent-runtime/agent_runtime.go:96`**

`IsAgentSandboxEnabled` calls `discovery.ServerGroupsAndResources()`, which issues one request per API group-version and is uncached. The disabled path (the default) always reaches `deleteSandbox`, so this adds a full discovery round-trip to every CheCluster reconcile for all installations.

**Fix:** short-circuit on `ctx.CheCluster.Spec.AgentRuntimes == nil` before touching discovery, and/or cache the result like `isServiceMonitorEnabled`.

## 8. Hardcoded container command — Minor

**`pkg/deploy/agent-runtime/agent_runtime.go:145`**

The command is hardcoded to `/bin/bash -c "sleep infinity"`, which overrides whatever entrypoint the admin-supplied `image` defines and hard-requires `/bin/bash` in that image. Combined with `EnsurePodSecurityStandards` setting `runAsNonRoot: true` with no `runAsUser` on OpenShift, an arbitrary user-provided image will frequently fail to start.

**Fix:** if this is placeholder scaffolding it should be called out; otherwise make the command configurable.

## 9. `continue` targets the wrong loop — Minor

**`build/scripts/clear-defined-test.sh:352`**

Inside the "try a shorter path" `while` loop, `recordError "[TIMEOUT]" …; continue` continues the *while*, not the per-module `for`. A module that times out repeatedly is appended to `retry_modules` once per path-shortening step, so it is retried — and, on the final iteration, printed to `$ERRORS_FILE` — several times over. The pre-refactor code had the same misplaced `continue`, but it was harmless there; now it multiplies work and duplicates error output.

## 10. `recordError` records the post-substitution module — Minor

**`build/scripts/clear-defined-test.sh:305`**

`retry_modules+=("$module")` records the value *after* the `replaced_modules` substitution has been applied. On the next iteration the lookup `[[ -v replaced_modules["$orig_module"] ]]` no longer matches, and the `[ERROR]`/`[TIMEOUT]` line printed on the final iteration reports the substitution target (e.g. `open-telemetry/opentelemetry-go-contrib a89d958e…`) instead of the actual failing dependency, making CI failures hard to attribute.

**Fix:** record the pre-substitution module.

## 11. Missing separator in the Sandbox name — Nit

**`pkg/deploy/agent-runtime/agent_runtime.go:170`**

`getSandboxName()` returns `defaults.GetCheFlavor() + constants.AgentRuntimesComponentName` with no separator, producing `cheagent-runtimes` (and `devspacesagent-runtimes` downstream). Valid DNS but almost certainly unintended — should be `+ "-" +`.

## 12. Typo in a shipped CRD description — Nit

**`api/v2/checluster_types.go:312`**

`hanging this value` is missing the leading `C` of "Changing" in the `Namespace` field doc. This is a generated CRD description, so it has already shipped into `bundle/next/.../org.eclipse.che_checlusters.yaml`, `config/crd/bases/…`, both `deploy/deployment/*` trees and the helm chart, and is user-visible in `oc explain`.

---

## Checked and cleared

- `new(agentRuntimes.RuntimeClassName)` at `agent_runtime.go:158` is valid — Go 1.26 added `new(expr)`, and `go build` / `go vet` pass.
- The missing owner reference on the Sandbox is correct — it is cross-namespace.
- The CRD / CSV / helm / deploy manifests are regenerated consistently with the new API field.
