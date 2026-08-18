# Raw research — CI architecture, scoped test selection, flaky test lifecycle

This is the unedited output from the research agent covering the keystone deliverable: the exact shape of `tests-conformance.yaml`, orphan suites, path→suite evidence, flaky-test history, and a real build-graph experiment. Synthesized findings from this file are folded into `kyverno-lfx-research.md` §11; this file is the full raw evidence trail.

All paths relative to `e:\kyverno`. Line numbers refer to file state at research time (2026-08-16).

---

## 1. tests-conformance.yaml — full job table (HIGHEST PRIORITY)

File: `.github/workflows/tests-conformance.yaml`, 1105 lines, `on: workflow_call: {}` only (never triggers standalone — only reachable via `check-tests.yaml` / `comment-conformance.yaml`). 54 top-level jobs confirmed.

### 1.1 Composite action mechanics (`.github/actions/tests/conformance/run/action.yaml`)

Inputs: `k8s-version`(req), `kind-config`(default `./scripts/config/kind/default.yaml`), `kyverno-configs`(default `standard`), `token`(req), `tests-path`(default `.`), `chainsaw-tests`(regex, default `''`), `shard-index`(default `0`), `shard-count`(default `0`), `quarantined-tests`(csv, default `''`), `install-cert-manager`/`install-kubectl-evict`/`install-openreports` (bool flags), `explicit-install-settings` (raw `--set` string passed to helm).

Steps: install helm/cosign/yq → install chainsaw → `helm/kind-action` creates kind cluster from `kind-config` → download+load `kyverno.tar` image archive (built once in the calling workflow, shared across all 54 jobs+shards) → conditionally install OpenReports CRDs / cert-manager → **`export USE_CONFIG=${{ kyverno-configs }}; make kind-install-kyverno`** → apply latest CRDs from `./config/crds` server-side → conditionally `go install kubectl-evict@latest` → wait-ready → **quarantine step**: for each name in `quarantined-tests` (comma split), `find ./test/conformance/chainsaw/${tests-path} -type d -name "$DIR_NAME"` then `yq eval '.spec.skip = true' -i chainsaw-test.yaml` on every match (this mutates the checked-out worktree at runtime, not the repo) → run:
```
cd ./test/conformance/chainsaw/${tests-path}
chainsaw test --config .chainsaw.yaml \
  --include-test-regex '^chainsaw$/${chainsaw-tests}' \
  --shard-index ${shard-index} --shard-count ${shard-count}
```
→ on failure, dump logs via `.github/actions/kyverno/logs`.

`USE_CONFIG` fan-out (Makefile:31,1114,1123,1143): comma-separated list, each token becomes `--values ./scripts/config/$(CONFIG)/kyverno.yaml` (and `kyverno-policies.yaml`) appended to the helm install/upgrade command — i.e. `kyverno-configs: standard,ttl` merges `scripts/config/standard/kyverno.yaml` + `scripts/config/ttl/kyverno.yaml`. Confirmed config dirs present: `custom-sigstore, default, default-with-profiling, dev, exclude-result, force-failure-policy-ignore, generate-mutating-admission-policy, generate-validating-admission-policy, kwok, kyverno-cleanup, mutating-admission-policy-reports, openreports, resources, service-monitor, sigstore-custom-tuf, standard, standard-with-profiling, ttl, validating-admission-policy-reports`.

### 1.2 Full job table

| Job | tests-path | k8s matrix | shard | extra matrix | Total runs | kyverno-configs | Special deps |
|---|---|---|---|---|---|---|---|
| assert | assert | 3 (v1.33.7/34.3/35.1) | – | – | 3 | standard | – |
| autogen | autogen | 3 | – | – | 3 | standard | – |
| background-only | background-only | 3 | – | – | 3 | standard | – |
| cleanup | cleanup | 3 | – | – | 3 | standard | – |
| cel-http | **cel/http** (not `cel`) | 3 | – | – | 3 | standard | – |
| custom-sigstore | custom-sigstore (hand-rolled job, bypasses the composite action) | 2 (v1.33.x/34.x) | – | – | 2 | n/a (manual) | cosign-installer v2.6.1, `sigstore/scaffolding/actions/setup` (Fulcio/Rekor/CTLog/TUF/Knative 1.10.0), crane, `USE_CONFIG=standard,custom-sigstore` |
| deferred | deferred | 3 | – | – | 3 | standard | – |
| deleting-policies | deleting-policies | 3 | 6 | – | 18 | standard | – |
| events | events | 3 | – | – | 3 | standard | – |
| exceptions | exceptions | 3 | 3 | – | 9 | standard | – |
| filter | filter | 3 | – | – | 3 | standard | – |
| force-failure-policy-ignore | force-failure-policy-ignore | 3 | – | – | 3 | standard,force-failure-policy-ignore | – |
| generate | generate | 3 | 12 | – | 36 | standard | install-kubectl-evict |
| generate-mutating-admission-policy-alpha | generate-mutating-admission-policy-alpha | 1 (v1.33.7) | – | – | 1 | standard,generate-mutating-admission-policy | kind-config `vap-v1alpha1.yaml`, quarantine=`applyconfiguration` |
| generate-mutating-admission-policy | generate-mutating-admission-policy | 2 (v1.34.3/35.1) | – | – | 2 | standard,generate-mutating-admission-policy | kind-config `vap-v1beta1.yaml`, quarantine=`applyconfiguration` |
| generate-mutating-admission-policy-v1 | generate-mutating-admission-policy-v1 | 1 (v1.36.1) | – | – | 1 | standard,generate-mutating-admission-policy | quarantine=`applyconfiguration` |
| generate-validating-admission-policy | generate-validating-admission-policy | 3 | – | – | 3 | standard | – |
| generating-policies | generating-policies | 3 | 6 | – | 18 | standard | quarantine=`sync-modify-downstream`, install-kubectl-evict |
| globalcontext | globalcontext | 3 | – | – | 3 | standard | – |
| image-validating-policies | image-validating-policies | 3 | 3 | – | 9 | standard | – |
| lease | lease | 3 | – | – | 3 | standard | – |
| mutate | mutate | 3 | 3 | – | 9 | standard | – |
| mutating-admission-policy-reports-alpha | mutating-admission-policy-reports-alpha | 1 (v1.33.7) | – | – | 1 | standard,mutating-admission-policy-reports | kind-config `vap-v1alpha1.yaml` |
| mutating-admission-policy-reports | mutating-admission-policy-reports | 2 (v1.34.3/35.1) | – | – | 2 | standard,mutating-admission-policy-reports | kind-config `vap-v1beta1.yaml` |
| mutating-admission-policy-reports-v1 | mutating-admission-policy-reports-v1 | 1 (v1.36.1) | – | – | 1 | standard,mutating-admission-policy-reports | – |
| reports-exclude-result | reports-exclude-result | 1 (v1.34.0) | – | – | 1 | exclude-result | – |
| mutating-policies | mutating-policies | 3 | 3 | – | 9 | standard | – |
| namespaced-deleting-policies | namespaced-deleting-policies | 3 | – | – | 3 | standard | – |
| namespaced-generating-policies | namespaced-generating-policies | 3 | 3 | – | 9 | standard | quarantine=`sync-modify-downstream` |
| namespaced-image-validating-policies | namespaced-image-validating-policies | 3 | – | – | 3 | standard | – |
| namespaced-mutating-policies | namespaced-mutating-policies | 3 | – | – | 3 | standard | – |
| namespaced-validating-policies | namespaced-validating-policies | 3 | 3 | – | 9 | standard | – |
| openreports | openreports | 3 | – | – | 3 | openreports | install-openreports |
| policy-exceptions-disabled | policy-exceptions-disabled | 3 | – | – | 3 | default | – |
| policy-validation | policy-validation | 3 | – | – | 3 | standard | – |
| rangeoperators | rangeoperators | 3 | – | – | 3 | standard | – |
| rbac | rbac | 3 | – | kyverno-configs:[standard,default,force-failure-policy-ignore] | 9 | matrix-driven | – |
| reports | reports | 3 | 3 | – | 9 | standard | – |
| sigstore-custom-tuf | sigstore-custom-tuf | 3 | – | – | 3 | standard,sigstore-custom-tuf | kind-config `vap-v1beta1.yaml` |
| ttl | ttl | 3 | – | – | 3 | standard,ttl | – |
| validate | validate | 3 | 8 | – | 24 | standard | – |
| validating-admission-policy-reports | validating-admission-policy-reports | 3 | – | – | 3 | standard | – |
| validating-policies | validating-policies | 3 | 5 | – | 15 | standard | – |
| verify-images | verify-images | 3 | 2 | – | 6 | standard | `permissions: packages: read` (pulls signed/private test images from GHCR) |
| verify-manifests | verify-manifests | 3 | – | – | 3 | standard | – |
| webhook-configurations | webhook-configurations | 3 | – | – | 3 | standard | – |
| webhooks | webhooks | 3 | – | – | 3 | standard | – |
| monitor-helm-secret-size | n/a (not a chainsaw job) | 1 (v1.35.1) | – | – | 1 | n/a | checks Helm release Secret size < 1,030,000 bytes (etcd object size limit guard) |
| check-tests | n/a — hand-rolled, runs `test/conformance/chainsaw/cli` suite directly via downloaded CLI binary | 1 (hardcoded kind v1.33.1) | – | – | 1 | n/a | downloads `kubectl-kyverno` artifact, chmods it into PATH |
| helm-tests | n/a (`make helm-test`) | 3 | – | – | 3 | n/a | – |
| helm-uninstall | n/a | 3 | – | – | 3 | kyverno-cleanup | – |
| helm-upgrade | n/a (helm install from published chart v3.6 then upgrade) | 3 | – | kyverno-version:["3.6"] | 3 | n/a | tests upgrade path from released chart |
| cert-manager-certificates | **tls-certificates/{rsa,ecdsa}-cert-manager** | 1 (v1.33.1) | – | key-algorithm:[rsa,ecdsa] | 2 | standard | install-cert-manager, explicit-install-settings (cert-manager algorithm/size) |
| self-signed-certificates | **tls-certificates/{ecdsa,ed25519}-self-signed** | 1 (v1.33.1, implicit) | – | key-algorithm:[ecdsa,ed25519] | 2 | standard | explicit-install-settings (tlsKeyAlgorithm) |

**Total parallel runner-jobs for a full conformance run: ~290** (summed the "Total runs" column above). This is the real fan-out cost of one `/conformance` comment or one push to main — a more precise number than the "150+"/"54 jobs" estimate used in the original draft.

### 1.3 Suite coverage / orphans (verified against the 52 top-level dirs under `test/conformance/chainsaw/`)

- `_step-templates/` — not a suite; 14 shared chainsaw step YAML templates (`create-policy.yaml`, `*-ready.yaml` waiters, etc.) referenced via `use: template:` from other suites. Correctly excluded from the 51 real suites.
- `cel/` — only has one subdir `http/`; fully covered by job `cel-http` (`tests-path: cel/http`).
- `configs/` — has 2 subdirs, each with a real `chainsaw-test.yaml` (`dont-emit-success-events-upon-generateSuccessEvents-set-false`, `emit-success-events-upon-generateSuccessEvents-set-true`). **ORPHAN: no job in any workflow references `tests-path: configs` or anything under it.** Verified via `grep -rn "tests-path" .github/workflows | grep -i configs` → zero hits, and `grep -rln "chainsaw/configs" e:/kyverno` → zero hits anywhere in the repo. These tests are dead code from a CI perspective.
- `flags/` — has 1 subdir `standard/emit-events/` with a real `chainsaw-test.yaml`. **ORPHAN, same as above** — zero references to `tests-path: flags` anywhere.
- `tls-certificates/` — has 4 subdirs (`rsa-cert-manager`, `ecdsa-cert-manager`, `ecdsa-self-signed`, `ed25519-self-signed`); all 4 are covered, but only as **subdir slices** split across 2 different jobs (`cert-manager-certificates`, `self-signed-certificates`) — no job runs `tests-path: tls-certificates` as a whole tree.
- `cli/` — not run through the shared composite action at all; run through a bespoke `check-tests` job that installs the CLI binary artifact and does `chainsaw test --config .chainsaw.yaml` directly against `./test/conformance/chainsaw/cli`.
- No suite is double-covered by two jobs running overlapping test sets — the sharded jobs partition their suite via `--shard-index/--shard-count`, and the `tls-certificates` sub-splits are non-overlapping directory paths.

**Net: of 51 real suites, 49 are exercised by some job; 2 (`configs`, `flags`, 3 chainsaw-test.yaml files total) are never executed by CI.**

### 1.4 Cross-check: `check-tests.yaml` orchestration (the only caller of `tests-conformance.yaml` on push)

`.github/workflows/check-tests.yaml` (push to `main`/`release-*` only): builds `images` (docker-save-image-all → `kyverno.tar` artifact) and `cli` (build-cli → `kubectl-kyverno` artifact) in parallel, then fans out to 3 reusable workflows via `uses:` (`tests-conformance.yaml`, `tests-conformance-policy-library.yaml`, `tests-k6.yaml`), then a `sync-issue` job (`if: always()`) rolls **all** of that into a single GitHub issue via `.github/actions/workflow/failure-issue` (see §4). The `conformance-tests` job passes `permissions: packages: read` down (needed for `verify-images`'s GHCR pulls).

---

## 2. Path → suite mapping, with evidence (10 representative dirs)

| Source dir | Suites it maps to | Evidence |
|---|---|---|
| `pkg/cel` (`autogen`, `compiler`, `engine`, `libs`, `matching`, `policies/{vpol,mpol,gpol,dpol,ivpol}`, `resource`) | `validating-policies`, `mutating-policies`, `generating-policies`, `namespaced-generating-policies`, `deleting-policies`, `namespaced-deleting-policies`, `image-validating-policies`, `namespaced-image-validating-policies`, `cel/http`, `autogen`, `policy-validation` | `pkg/cel/policies/vpol/{validate.go,engine/*,compiler/*}` all import `github.com/kyverno/api/api/policies.kyverno.io/v1beta1` (confirmed: `grep -n "kyverno/api" pkg/cel/policies/vpol/validate.go:4`). **The CEL policy CRD Go types (ValidatingPolicy, MutatingPolicy, GeneratingPolicy, DeletingPolicy, ImageValidatingPolicy) do NOT live in this repo at all** — they're vendored from a separate module `github.com/kyverno/api v0.0.1-alpha.3.0.20260723090831-fb2785727f98` (go.mod:35, a pseudo-version). This means schema/field changes to CEL policy kinds originate in a sibling repo and land here only via a `go.mod` bump — a real cross-repo dependency the assistant should track. `test/conformance/chainsaw/validating-policies/apply-v1/policy.yaml` etc. use `kind: ValidatingPolicy`; `deleting-policies/cel-lib/{globalcontext,http,image-data}-lib/policy.yaml` directly exercise `pkg/cel/libs`. |
| `pkg/webhooks` (`celexception`, `exception`, `globalcontext`, `handlers`, `policy`, `resource`, `updaterequest`) | `webhooks`, `webhook-configurations`, `exceptions`, `globalcontext`, `autogen`, `policy-validation` | Directory names line up 1:1 with suite names (`celexception`/`exception` → `exceptions` suite; `globalcontext` → `globalcontext` suite). `pkg/webhooks/handlers` implements the admission webhook HTTP handlers exercised end-to-end by every suite that submits an admission request, but `webhook-configurations`/`webhooks` are the suites that specifically assert on the generated `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration` objects (autogen'd rules), which is `pkg/webhooks/policy`'s job. |
| `pkg/controllers` (`admissionpolicygenerator`, `certmanager`, `cleanup`, `deleting`, `exceptions`, `generic`, `globalcontext`, `policycache`, `policystatus`, `report`, `ttl`, `webhook`) | `generate-validating-admission-policy`, `generate-mutating-admission-policy*`, `tls-certificates/*-cert-manager`, `cleanup`, `deleting-policies`, `ttl`, `webhook-configurations`, `reports`/`validating-admission-policy-reports`/`mutating-admission-policy-reports*` | Direct name matches: `pkg/controllers/admissionpolicygenerator` → the two `generate-*-admission-policy*` jobs (K8s-native VAP/MAP autogenerated from Kyverno policies); `pkg/controllers/certmanager` → `cert-manager-certificates` job (`install-cert-manager: true`); `pkg/controllers/cleanup`→`cleanup` suite; `pkg/controllers/ttl`→`ttl` suite (note `kyverno-configs: standard,ttl`); `pkg/controllers/deleting`→`deleting-policies`/`namespaced-deleting-policies`; `pkg/controllers/report`→ the `*-reports` family of suites. |
| `api/kyverno/v1` (+ v1beta1/v2/v2alpha1/v2beta1 — the classic `ClusterPolicy`/`Policy` CRD types, in-repo) | `assert`, `autogen`, `policy-validation`, `verify-images`, `verify-manifests`, most non-CEL suites | `test/conformance/chainsaw/policy-validation/cluster-policy/admission-disabled/policy-{generate,mutate,validate}.yaml` and `verify-images/clusterpolicy/**/policy.yaml` use `kind: ClusterPolicy` (defined in-repo). This is the one CRD family that IS versioned in-tree, contrasting directly with `pkg/cel`'s external-module dependency above. |
| `cmd/cli` (`kubectl-kyverno/`) | `cli` suite (`test/conformance/chainsaw/cli`) | `check-tests` job in `tests-conformance.yaml` (lines 880-914) downloads the `kubectl-kyverno` artifact (built by `check-tests.yaml`'s `cli` job via `make build-cli`), installs it into PATH, then runs `chainsaw test` directly against `./test/conformance/chainsaw/cli` — the only suite invoked without the shared composite action. Also backed by the Makefile's per-kind local CLI targets (`test-cli-local-vpols/-gpols/-mpols/-ivpols/-dpols/-vaps/-maps`), which run the CLI against fixtures without a cluster at all — a faster, cluster-less feedback loop exercised by `check-cli-tests.yaml` (`make test-cli`) on every PR (see §5). |
| `pkg/validation` (`cleanuppolicy`, `exception`, `globalcontext`, `policy`, `resource`) | `policy-validation`, `exceptions`, `globalcontext`, `cleanup` | Name-for-name match with suite directories; `pkg/validation/policy` is the admission-time policy-schema validator exercised by `policy-validation/cluster-policy/**` fixtures. |
| `pkg/background` (`generate`, `gpol`, `mpol`, `mutate`) | `generate`, `generating-policies`, `mutating-policies`, `mutate` | `pkg/background/generate` = classic `ClusterPolicy` background generation, exercised by `generate` suite (install-kubectl-evict, 12-way sharded — the largest suite in the repo); `pkg/background/gpol`/`mpol` = CEL `GeneratingPolicy`/`MutatingPolicy` background reconciliation, exercised by `generating-policies`/`mutating-policies`. |
| `pkg/image` (`verification`, `verifiers`) | `verify-images`, `verify-manifests`, `custom-sigstore`, `sigstore-custom-tuf` | `verify-images/clusterpolicy/**/policy.yaml` all set `verifyImages:` rules; `custom-sigstore` and `sigstore-custom-tuf` jobs stand up their own Fulcio/Rekor/TUF infra specifically to exercise this package's cosign/sigstore verification code paths against non-default trust roots. |
| `pkg/policy` (`auth`, `common`, `generate`, `mutate`, `validate`, plus `gpol.go`/`mpol.go`/`policy_controller.go`) | `policy-validation`, `generate`, `mutate`, `validate`, `generating-policies`, `mutating-policies` | `pkg/policy/gpol.go`/`mpol.go` bridge the classic policy controller to CEL gpol/mpol reconciliation — same suites as `pkg/background/{gpol,mpol}` above but at the controller-registration layer rather than the background-scan layer. |
| `pkg/engine` (`adapters`, `apicall`, `handlers`, `mutate`, `validate`, `image_verify.go`, `generation.go`) | Nearly all suites transitively (it's the core admission/report evaluation engine) — most directly: `validate`, `mutate`, `generate`, `verify-images`, `policy-validation`, `exceptions` | This is the highest-fan-in package (confirmed by the build-graph experiment in §6: changing `pkg/engine` transitively affects 60 of 563 Go packages) — essentially every conformance suite that submits an admission request exercises `pkg/engine.Handler` in some form, so it's a poor candidate for path-based test *narrowing* despite being small in isolation. |

---

## 3. Flaky-test lifecycle

### 3.1 Two independent suppression mechanisms co-exist

**A. Static, committed `skip: true`** in the chainsaw-test.yaml itself (4 instances repo-wide, verified via `grep -rln "skip: true" test/conformance/chainsaw`):
- `generating-policies/clone/sync/sync-delete-one-source/chainsaw-test.yaml`
- `generating-policies/clone/sync/sync-delete-one-trigger/chainsaw-test.yaml`
- `generating-policies/clone/sync/sync-delete-policy/chainsaw-test.yaml`
- `verify-images/clusterpolicy/standard/update-multi-containers/chainsaw-test.yaml`

**B. Dynamic, CI-injected `skip: true`** via the `quarantined-tests` composite-action input, which does a runtime `find -type d -name "$DIR_NAME"` + `yq eval '.spec.skip = true' -i` against the checked-out worktree (does not touch git). Currently wired up in exactly 2 places in `tests-conformance.yaml`:
- `quarantined-tests: applyconfiguration` on all 3 `generate-mutating-admission-policy*` jobs.
- `quarantined-tests: sync-modify-downstream` on `generating-policies` and `namespaced-generating-policies`.

### 3.2 A real, dated flaky-test incident chain (git-log-verified) shows mechanism (B) was built in response to the pain of mechanism (A)

1. PR #14199 "remove flaky test temporary to unblock PRs" — a flaky generating-policies sync-delete test was deleted outright to unblock merges.
2. PR #14200 (commit `83de60892`, "Revert 'remove flaky test temporary to unblock PRs'") — reverted the deletion but landed the test back with `spec.skip: true` hardcoded, rather than removing it a second time. This produced the 3 `sync-delete-*` skip:true files listed above.
3. PR #14570 (commit `5a832b4d7`, "introduce test quarantine capabilites through inputs to the run tests action", merged ~1 month before #14200) — added the dynamic `quarantined-tests` mechanism precisely so future flaky tests wouldn't require a source-code edit to disable — they can instead be named in the *workflow* file, easier to grep/revert/audit than scattered `skip: true`.

This shows a live "flaky test governance" tension the AI Maintainer Assistant could address: two mechanisms, no dashboard reconciling them, no automatic expiry/reminder on either.

### 3.3 Chainsaw reporting capability — NOT used

- Chainsaw is pinned via `kyverno/action-install-chainsaw@06560d18...` (v0.2.14 action) installing **chainsaw release `v0.2.15-beta.3`** (`.github/actions/tools/chainsaw/action.yaml`).
- `grep -rn "report-format|report-path|--report|junit" .github Makefile test` → **zero matches anywhere in the repo.** Chainsaw v0.2.x does support `--report-format` (JSON/XML/JUnit-ish) and `--report-path` flags upstream, but Kyverno's CI never passes them. This means: (a) test results only exist as raw CI log text, no machine-readable pass/fail/duration artifact is ever produced or uploaded; (b) there is no historical flakiness data set to mine today — enabling `--report-format=JSON --report-path=...` and uploading it as an artifact is the prerequisite for any flake-detection tooling.

---

## 4. `.github/actions/workflow/failure-issue` — full lifecycle

Single-file composite action (`action.yaml`, ~120 lines) using `actions/github-script`. Inputs: `conclusion` (required), `needs` (JSON-serialized `toJSON(needs)`, default `{}`).

**Dedupe key**: `workflow-failure:${workflowName}:${branch}` — embedded as an HTML comment `<!-- workflow-failure:Tests:refs/heads/main -->` at the very end of the issue body. Lookup: list all OPEN issues labeled `workflow-failure` (paginated, up to 100), then `.find(i => i.body.includes(commentMarker))`. **Granularity is per (workflow name, branch) pair — not per job, not per test.** A single workflow like "Tests" failing on `main` collapses to exactly one tracking issue regardless of whether 1 shard or all 290 conformance runner-jobs failed.

**Title format**: `Workflow failed: ${workflowName} (${branch})` e.g. `Workflow failed: Tests (refs/heads/main)`.

**Body**: a markdown table (Run / Branch / Commit / Triggered by / Conclusion) plus an optional `### Job results` table built from `formatNeedsResultsTable(needs)` — but `needs` is only the **direct** `needs:` of the calling `sync-issue` job. In `check-tests.yaml` that's just `[images, cli, conformance-tests, conformance-policy-library-tests, k6-tests]` — 5 rows — even though `conformance-tests` itself fans out to ~290 runner jobs. **The issue body cannot show you which of the 54 conformance jobs or which shard/suite failed; you must click through to the Actions run.** A concrete enrichment: recursively resolve failed sub-jobs via the Actions API and paste a "drill-down" table naming the actual failing suite/shard/k8s-version.

**Create vs update vs close**:
- `conclusion in {failure, cancelled}`: if a matching open issue exists, `issues.update()` (same issue number — no duplicate spam on repeat failures); else `issues.create()` with `labels: ['workflow-failure']`, with a fallback retry without the label if the repo returns HTTP 422.
- Otherwise (success): if a matching open issue exists, post a comment and `issues.update({state: 'closed'})` — **auto-close on next green run.** If no existing issue, no-op.

**Callers** (20 workflows): `cache.yaml`, `check-ah-lint.yaml`, `check-cli-tests.yaml`, `check-codegen.yaml`, `check-ct-lint.yaml`, `check-devcontainer.yaml`, `check-fmt.yaml`, `check-golangci-lint.yaml`, `check-imports.yaml`, `check-sha-pinned-actions.yaml`, `check-tests.yaml`, `check-unit-tests.yaml`, `check-unused-package.yaml`, `check-vet.yaml`, `periodic-cleanup.yaml`, `periodic-trivy.yaml`, `scan-fossa.yml`, `scan-scorecard.yaml`, `scan-semgrep.yaml`, `scan-sonarcloud.yaml`, `scan-trivy.yaml`.

---

## 5. Other workflows — triggers & cost

| Workflow | Trigger | Gates PRs? | Cost / shape |
|---|---|---|---|
| `check-unit-tests.yaml` | `pull_request` + `push` to `main`/`release-*`; concurrency cancels in-progress on PR | **Yes** | 1 job: `make test-unit` = `go test -v -race -covermode atomic -coverprofile ... ./...` (all 563 packages, no scoping today) → uploads `coverage.out` → codecov → `sync-issue` (push only). |
| `check-cli-tests.yaml` | `pull_request` + `push` | **Yes** | 1 job `check`: `make test-cli` then 3 manual negative-test invocations against `test/cli/test-fail/{missing-policy,missing-rule,missing-resource}`. |
| `check-framework.yaml` ("framework tests (envtest)") | `pull_request` (paths-ignore docs/charts/md) + `push` | **Yes** | Matrix over `policy-type: [vpol, mpol, gpol, dpol, ivpol]` (5 jobs), `sigs.k8s.io/controller-runtime/tools/setup-envtest` v0.24.0 against K8s 1.36, `go test -tags=integration ./test/integration/${policy-type}/...` — real API server via envtest, no kind/docker, much cheaper than full conformance. |
| `tests-k6.yaml` | `workflow_call` only | No (push-only via parent) | Matrix `scenario: [simple, kyverno, kyverno-pss]`; `k6 run --vus 100 --iterations 1000`; performance regression, not correctness. |
| `tests-conformance-policy-library.yaml` | `workflow_call` only | No | Checks out a **second, separate repo** `kyverno/policies`, applies its CRDs, runs chainsaw against it (36 runner-jobs: 3 k8s × 12 shards) — the only cross-repo blast-radius test. |

Combined with `check-tests.yaml` (push-only), the picture: unit tests, CLI tests, and envtest framework tests all gate **every PR**; the ~290-job conformance suite, k6 perf, and cross-repo policy-library suite only run on push-to-main/release or on-demand via `/conformance` comment.

---

## 6. Go build-graph-based test selection — verified, working pipeline

**Current state: none exists.** `Makefile:876-879` (`test-unit`) is a flat `go test ... ./...` — no changed-file scoping today. `go list ./...` on this repo returns **563 packages**.

### 6.1 A concrete, tested pipeline (built and run against this checkout)

Step 1 — build a full reverse-dependency edge list once per run:
```
go list -json ./... | jq -r '.ImportPath as $p | (.Deps // [])[] \
  | select(startswith("github.com/kyverno/kyverno/")) | "\(.)\t\($p)"' > edges.tsv
```
Measured: **22,878 edges** across 563 packages, **~33s** wall-clock (dominated by `go list -json` type-checking every package). In CI this should reuse the `gobuild-cache` already wired up in `.github/actions/go/setup`/`go/save-cache` (used by `check-unit-tests.yaml`).

Step 2 — map changed files to Go package import paths:
```
git diff --name-only origin/main...HEAD -- '*.go' \
  | xargs -n1 dirname | sort -u \
  | sed 's#^#github.com/kyverno/kyverno/#'
```

Step 3 — BFS the reverse-edge graph from each changed package to its full transitive closure of importers (implemented and run as `bfs.py` against the real `edges.tsv`):

| Changed package | Transitively-affected packages (of 563) |
|---|---|
| `pkg/webhooks/handlers` | 18 |
| `pkg/cel` (the bare directory, not its subpackages) | 1 — misleading; almost all real fan-out lives in `pkg/cel/policies/*`, `pkg/cel/libs`, etc. as separate Go packages, so a naive dir-level mapping under-counts unless subpackages are walked too |
| `pkg/utils/conditions` | 66 |
| `pkg/engine` | 60 |
| `api/kyverno/v1` | 457 (81% of the whole repo) |

Step 4 — the actual test-selection command: intersect the affected-package set with `go list -f '{{.ImportPath}}' ./...` filtered to packages that have `_test.go` files, then `go test $(that list)`.

### 6.2 What this tells the proposal

- For genuinely leaf/narrow packages (`pkg/webhooks/handlers`, `pkg/utils/conditions`) this would cut unit-test scope by ~85-97%, a real CI-time win on every PR.
- For foundational packages (`api/kyverno/v1`, and by extension anything in `pkg/engine`, `pkg/cel`) the affected-set is 60-81% of the repo — build-graph selection buys little; a heuristic ("if the changed package's affected-set exceeds N% of all packages, just run everything") is needed to avoid false confidence from an incomplete run.
- The `pkg/cel` bare-directory result (1 affected package) is a trap worth calling out explicitly: Go's package graph is per-directory, so any tool built on this must walk `pkg/cel/...` recursively, not stop at the parent directory string.
- None of this maps onto the *conformance* suite, since conformance tests are chainsaw YAML fixtures with no Go import graph — build-graph selection is a unit-test-only optimization; conformance suite selection needs the path→suite table in §2 instead (fuzzier/heuristic, not derivable from `go list`).

---

## Key files referenced (for follow-up)
- `.github/workflows/tests-conformance.yaml` (1105 lines, 54 jobs)
- `.github/actions/tests/conformance/run/action.yaml` (159 lines, the shared composite)
- `.github/workflows/check-tests.yaml` (101 lines, the only push-triggered caller)
- `.github/actions/workflow/failure-issue/action.yaml` (121 lines)
- `.github/actions/tools/chainsaw/action.yaml` (chainsaw v0.2.15-beta.3 pin)
- `.github/workflows/check-unit-tests.yaml`, `check-cli-tests.yaml`, `check-framework.yaml`, `tests-k6.yaml`, `tests-conformance-policy-library.yaml`
- `Makefile:31,876-879,1114,1123,1143` (USE_CONFIG fan-out, test-unit target)
- `go.mod:35` (`github.com/kyverno/api` external module dependency for CEL policy CRD types)
- Git commits: `5a832b4d7` (#14570, quarantine mechanism introduced), `83de60892` (#14200, revert that landed static skip:true)
