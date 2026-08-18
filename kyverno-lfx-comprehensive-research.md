# Kyverno LFX Mentorship — Comprehensive Research, Mapping & Architecture Specification

**Target Repository:** `kyverno/kyverno` (`main`, Aug 2026)  
**Document Purpose:** Unified ground-truth research, Go package-to-Chainsaw conformance suite mapping, AI CI/CD threat models, and architectural specification for the LFX Mentorship proposal.

---

## 0. Executive Summary & Keystone Thesis

### The Core Problem
Kyverno maintainers spend significant recurring effort on low-judgment, repetitive tasks: triaging issues, managing PR conflicts/rebases, bisecting CI failures, and tracking down regressions.

A ground-truth audit of the repository reveals the core bottleneck:
> **Kyverno’s end-to-end conformance test suite (`tests-conformance.yaml`) does NOT run on Pull Requests.**

* `.github/workflows/check-tests.yaml` triggers on `push:` to `main` / `release-*` only. No `pull_request` trigger.
* On PRs, conformance is **opt-in** via a `/conformance` comment (`comment-conformance.yaml`).
* **Why:** `tests-conformance.yaml` contains **54 jobs**, most with a 3-way Kubernetes version matrix (`v1.33.7`, `v1.34.3`, `v1.35.1`), several sharded 3–12 ways, alongside `tests-conformance-policy-library.yaml` (shard-count 12) and `tests-k6.yaml`. That equals **150+ runner allocations per full run**. Across **300+ open PRs**, running full conformance per-PR is computationally and financially unfeasible.

### The Downstream Consequence
* Regressions land on `main`, then get discovered by the post-merge run.
* `.github/actions/workflow/failure-issue` auto-files an issue on failure and auto-closes it on the next green run. There are **95+ issues** with the `workflow-failure` label, overwhelmingly titled *"Workflow failed: Tests (refs/heads/main)"*.
* Maintainers must manually bisect regressions on `main` under release pressure.

### Keystone Framing
> **Kyverno's CI is structurally *post-merge* because *pre-merge* is too expensive. Scoped test selection is what makes pre-merge affordable. Everything else in this project is downstream of that.**

---

## 1. Ground-Truth Package → Chainsaw Conformance Suite Mapping

### Task A — CRD Kinds per Chainsaw Conformance Suite
*Location:* `test/conformance/chainsaw/<suite>` (52 top-level test suites + shared templates).

| Suite Name | Dominant Kyverno & K8s Policy CRD Kinds (Counts) | Execution Notes / Constraints |
| :--- | :--- | :--- |
| `_step-templates` | `ValidatingPolicy` (2), `MutatingPolicy` (2), `GeneratingPolicy` (2), `Policy` (1), `ImageValidatingPolicy` (1), Namespaced variants (3) | **Shared infrastructure (14 files), not a standalone test suite.** Referenced 1,768 times across 854 test files. Global blast radius. |
| `assert` | (none) | Contains only `assert/.chainsaw.yaml` (`chainsaw Configuration`); no standalone tests. |
| `autogen` | `ClusterPolicy` (33) | Tests pod controller rule auto-generation. |
| `background-only` | `Policy` (6), `ClusterPolicy` (6), `EphemeralReport` (4) | Background controller scan operations. |
| `cel` | `NamespacedValidatingPolicy` (1) | Only `cel/http`; includes Service/ConfigMap json-server. |
| `cleanup` | `ClusterCleanupPolicy` (8), `CleanupPolicy` (3) | Cron-based cleanup jobs in `cleanup-controller`. |
| `cli` | `ValidatingPolicy` (3), `ImageValidatingPolicy` (3), `ClusterPolicy` (3), `PolicyException` (2), `MutatingAdmissionPolicy` (+Binding) (2), `GeneratingPolicy` (2), `ValidatingAdmissionPolicy` (1) | Validates CLI execution against diverse CRDs. |
| `configs` | `Policy` (4) | Kyverno `ConfigMap` asserts (×6). |
| `custom-sigstore` | `ClusterPolicy` (2) | Custom Sigstore verification setups. |
| `deferred` | `ClusterPolicy` (6) | Deferred policy loading/evaluation. |
| `deleting-policies` | `DeletingPolicy` (10), `GlobalContextEntry` (1) | Deletion policies in `cleanup-controller`. |
| `events` | `ClusterPolicy` (11), `Policy` (8), `MutatingPolicy` (2) | K8s event generation upon policy application. |
| `exceptions` | `PolicyException` (32), `ClusterPolicy` (27) | `kyverno.io` v2 PolicyException matching. |
| `filter` | `ClusterPolicy` (12) | Resource filtering rules. |
| `flags` | `Policy` (3) | Controller feature-flag assertions. |
| `force-failure-policy-ignore` | `ClusterPolicy` (4), `ValidatingPolicy` (1), `MutatingPolicy` (1), `ImageValidatingPolicy` (1) | Requires `kyverno-configs: standard,force-failure-policy-ignore`. |
| `generate` | `ClusterPolicy` (203), `Policy` (92), `UpdateRequest` (2), `ClusterCleanupPolicy` (1) | Legacy UpdateRequest-driven generation engine. |
| `generate-mutating-admission-policy` | `MutatingPolicy` (31), `MutatingAdmissionPolicy` (7) + `Binding` (7), `PolicyException` (3) | Requires `kyverno-configs: standard,generate-mutating-admission-policy`. |
| `generate-mutating-admission-policy-alpha` | `MutatingPolicy` (31), `MAP` (7) + `Binding` (7), `PolicyException` (3) | Alpha version converter tests. |
| `generate-mutating-admission-policy-v1` | `MutatingPolicy` (31), `MAP` (7) + `Binding` (7), `PolicyException` (3) | V1 version converter tests. |
| `generate-validating-admission-policy` | `ClusterPolicy` (70), `VAP` (47) + `VAPBinding` (47), `ValidatingPolicy` (37), `PolicyException` (12) | Native K8s ValidatingAdmissionPolicy generator. |
| `generating-policies` | `GeneratingPolicy` (104), `ClusterPolicyReport` (9), `PolicyException` (6), `PolicyReport` (1), `GlobalContextEntry` (1) | CEL GeneratingPolicy engine (`pkg/cel/policies/gpol`). |
| `globalcontext` | `GlobalContextEntry` (10), `ClusterPolicy` (7) | Global context cache & external API entries. |
| `image-validating-policies` | `ImageValidatingPolicy` (53), `PolicyException` (5), `PolicyReport` (4), `GlobalContextEntry` (2) | CEL ImageValidatingPolicy engine (`pkg/cel/policies/ivpol`). |
| `lease` | `Lease` (4), `Pod` (8) | Leader election per controller (admission, background, cleanup, reports). |
| `mutate` | `ClusterPolicy` (74), `Policy` (13), `MutatingPolicy` (2) | Legacy mutation engine (`pkg/engine/mutate`). |
| `mutating-admission-policy-reports` | `MutatingAdmissionPolicy` (3) + `Binding` (3), `PolicyReport` (2) | Reports generated from MAP evaluations. |
| `mutating-admission-policy-reports-alpha` | `MAP` (3) + `Binding` (3), `PolicyReport` (2) | Alpha report variants. |
| `mutating-admission-policy-reports-v1` | `MAP` (3) + `Binding` (3), `PolicyReport` (2) | V1 report variants. |
| `mutating-policies` | `MutatingPolicy` (59), `PolicyReport` (10), `PolicyException` (6), `NamespacedMutatingPolicy` (1), `GlobalContextEntry` (1) | CEL MutatingPolicy engine (`pkg/cel/policies/mpol`). |
| `namespaced-deleting-policies` | `NamespacedDeletingPolicy` (3) | Namespaced DeletingPolicy evaluation. |
| `namespaced-generating-policies`| `NamespacedGeneratingPolicy` (3) | Namespaced GeneratingPolicy evaluation. |
| `namespaced-image-validating-policies` | `NamespacedImageValidatingPolicy` (9), `PolicyReport` (2) | Namespaced ImageValidatingPolicy evaluation. |
| `namespaced-mutating-policies` | `NamespacedMutatingPolicy` (3) | Namespaced MutatingPolicy evaluation. |
| `namespaced-validating-policies` | `NamespacedValidatingPolicy` (6) | Namespaced ValidatingPolicy evaluation. |
| `openreports` | `Report` (`openreports.io`) (1), `Policy` (1) | Requires `kyverno-configs: openreports` + `install-openreports: true`. |
| `policy-exceptions-disabled` | `ValidatingPolicy` (2), `PolicyException` (2), `ImageValidatingPolicy` (2) | Requires `kyverno-configs: default` (exceptions disabled). |
| `policy-validation` | `ClusterPolicy` (34), `Policy` (11), `PolicyException` (4) | Webhook admission validation of policy schemas. |
| `rangeoperators` | `ClusterPolicy` (1) | Range operator evaluations in JMESPath. |
| `rbac` | `ClusterCleanupPolicy` (2), `ClusterPolicy` (1), `ClusterRole` (7) | Tests under `[standard, default, force-failure-policy-ignore]`. |
| `reports` | `ClusterPolicy` (20), `PolicyReport` (18), `ClusterPolicyReport` (12), `PolicyException` (5), `Policy` (2), `VAP` (1), `EphemeralReport` (1) | Aggregated policy reporting engine. |
| `reports-exclude-result` | `Policy` (1) | Requires `kyverno-configs: exclude-result`. |
| `sigstore-custom-tuf` | `ClusterPolicy` (1) | Requires `kyverno-configs: standard,sigstore-custom-tuf`. |
| `tls-certificates` | `ClusterPolicy` (4) | Key algorithm matrix over `[ECDSA, RSA, Ed25519]`. |
| `ttl` | `Pod` (14), `Job` (3), `ConfigMap` (2) (`cleanup.kyverno.io/ttl`) | Requires `kyverno-configs: standard,ttl`. |
| `validate` | `ClusterPolicy` (188), `PolicyReport` (6), `PolicyException` (4), `Policy` (4) | Legacy validation engine (`pkg/engine/validate`). |
| `validating-admission-policy-reports` | `ValidatingAdmissionPolicy` (9) + `Binding` (4), `PolicyReport` (8), `ValidatingPolicy` (2), `ImageValidatingPolicy` (2) | Reports for native K8s VAPs. |
| `validating-policies` | `ValidatingPolicy` (91), `PolicyReport` (12), `PolicyException` (12), `GlobalContextEntry` (3) | CEL ValidatingPolicy engine (`pkg/cel/policies/vpol`). |
| `verify-images` | `ClusterPolicy` (49), `PolicyReport` (3) | Legacy image verification (`pkg/engine/image_verify`). |
| `verify-manifests` | `ClusterPolicy` (4) | Manifest verification via Cosign signatures. |
| `webhook-configurations` | `ClusterPolicy` (7), `PolicyReport` (4) | Dynamic webhook configuration controller. |
| `webhooks` | `ClusterPolicy` (26), `Policy` (8) | Admission webhook server routing and TLS. |

---

### Task B — Go Source Directory Tree & Module Boundaries

#### 1. The `api/` Directory vs. External `github.com/kyverno/api`
* `api/kyverno/{v1, v1beta1, v2, v2alpha1, v2beta1}`: Legacy `ClusterPolicy`, `Policy`, `UpdateRequest`, `CleanupPolicy`, `PolicyException`, `GlobalContextEntry`.
* `api/policyreport/v1alpha2`: `PolicyReport`, `ClusterPolicyReport`.
* `api/reports/v1`: `EphemeralReport`, `ClusterEphemeralReport`.
* **Critical Finding:** `ValidatingPolicy`, `MutatingPolicy`, `ImageValidatingPolicy`, `GeneratingPolicy`, `DeletingPolicy`, and all `Namespaced*Policy` types live in the external module `github.com/kyverno/api` (`policies.kyverno.io/v1alpha1`, `v1beta1`).
* **Codegen Impact:** Upstream changes are imported via `go.mod` and regenerated into `config/crds/policies.kyverno.io/` and `pkg/client/` using `make codegen-api-bump`.

#### 2. Core `pkg/` Packages
* `pkg/engine/`: Evaluates `kyverno.io/v1` `ClusterPolicy` / `Policy` only (883 references to `kyvernov1.`). Contains rule handlers for `validate`, `mutate`, `generate`, `image_verify`. **Zero CEL policy types.**
* `pkg/cel/policies/`: Evaluates CEL policies:
  * `vpol/`: `ValidatingPolicy` (`compiler`, `engine`, `autogen`)
  * `mpol/`: `MutatingPolicy` (`compiler`, `engine`, `autogen`)
  * `ivpol/`: `ImageValidatingPolicy` (`engine`, `autogen`)
  * `gpol/`: `GeneratingPolicy` (`compiler`, `engine`, `template`)
  * `dpol/`: `DeletingPolicy` (`compiler`, `engine`)
* `pkg/webhooks/`: Webhook server and route handlers:
  * Generic handlers (`pkg/webhooks/handlers/`)
  * Policy admission (`pkg/webhooks/policy/`)
  * Exception admission (`pkg/webhooks/exception/`, `celexception/`)
  * Resource dispatchers (`pkg/webhooks/resource/{vpol,mpol,ivpol,gpol}/`)
  * Direct server routes (`pkg/webhooks/server.go`): `/vpol/*`, `/mpol/*`, `/ivpol/*`, `/gpol/*`, `/validate`, `/mutate`.
* `pkg/background/`: Background processing (`update_request_controller.go`, `generate/`, `mutate/`, `gpol/`, `mpol/`).
* `pkg/image/`:
  * `pkg/image/verification/`: Verification cache, evaluator, variables.
  * `pkg/image/verifiers/cpol/{cosign,notary}`: ClusterPolicy image verifiers.
  * `pkg/image/verifiers/ivpol/{cosign,notary}`: ImageValidatingPolicy image verifiers.
  * `pkg/sigstoretuf/`: Custom Sigstore TUF roots.
* `pkg/controllers/`: Core Kubernetes controllers:
  * `cleanup/`: Cron cleanup policies.
  * `ttl/`: TTL label manager (`cleanup.kyverno.io/ttl`).
  * `deleting/`: DeletingPolicy execution.
  * `report/`: Reports aggregation and background scans (`aggregate/`, `background/`, `resource/`).
  * `webhook/`: Validating/Mutating webhook configuration manager.
  * `admissionpolicygenerator/`: Converts ClusterPolicy/ValidatingPolicy into native K8s VAP/MAP.
  * `globalcontext/`, `certmanager/`, `exceptions/`, `metrics/`.

#### 3. Binary Entry Points (`cmd/`)
* `cmd/kyverno/main.go`: Admission Controller (imports `pkg/webhooks`, `pkg/controllers/webhook`, `pkg/cel/policies/{ivpol,mpol,vpol}`, `pkg/controllers/admissionpolicygenerator`).
* `cmd/background-controller/main.go`: Background Controller (imports `pkg/background`, `pkg/background/gpol`, `pkg/policy`, `pkg/cel/policies/{gpol,mpol}`).
* `cmd/reports-controller/main.go`: Reports Controller (imports `pkg/controllers/report/{aggregate,background,resource}`, `pkg/openreports`, `pkg/admissionpolicy`).
* `cmd/cleanup-controller/main.go`: Cleanup Controller (imports `pkg/controllers/{cleanup,deleting,ttl}`, `pkg/cel/policies/dpol`).
* `cmd/cli/kubectl-kyverno/`: Kyverno CLI (`apply`, `test`, `fix`, `jp`, `docs`, `oci`, `create`).

---

### Task C — Concern → Go Directory Mapping

| Concern | Go Source Directories | Chainsaw Conformance Suites |
| :--- | :--- | :--- |
| **CleanupPolicy / ClusterCleanupPolicy** | `pkg/controllers/cleanup/`, `pkg/validation/cleanuppolicy/`, `cmd/cleanup-controller/handlers/admission/policy/` | `cleanup`, `rbac` |
| **DeletingPolicy / NamespacedDeletingPolicy**| `pkg/controllers/deleting/`, `pkg/cel/policies/dpol/{compiler,engine}/` | `deleting-policies`, `namespaced-deleting-policies` |
| **TTL Label (`cleanup.kyverno.io/ttl`)** | `pkg/controllers/ttl/`, `pkg/validation/resource/`, `cmd/cleanup-controller/handlers/admission/resource/` | `ttl` |
| **GlobalContextEntry** | `pkg/controllers/globalcontext/`, `pkg/globalcontext/{store,externalapi,k8sresource,event}/`, `pkg/webhooks/globalcontext/` | `globalcontext` |
| **Report Generation & Aggregation** | `pkg/controllers/report/{aggregate,background,resource}/`, `pkg/openreports/`, `pkg/utils/report/` | `reports`, `reports-exclude-result`, `openreports`, `mutating-admission-policy-reports*`, `validating-admission-policy-reports` |
| **Autogen (Pod Controllers)** | `pkg/autogen/{v1,v2}/` (legacy), `pkg/cel/autogen/`, `pkg/cel/policies/{vpol,mpol,ivpol}/autogen/` | `autogen` |
| **Webhook Configurations** | `pkg/controllers/webhook/` (`controller.go`, `validating.go`, `match_conditions.go`) | `webhook-configurations`, `webhooks`, `lease` |
| **VAP/MAP Generation** | `pkg/controllers/admissionpolicygenerator/`, `pkg/admissionpolicy/` | `generate-validating-admission-policy`, `generate-mutating-admission-policy*` |
| **PolicyException** | `pkg/webhooks/exception/`, `pkg/webhooks/celexception/`, `pkg/validation/exception/`, `pkg/controllers/exceptions/`, `pkg/exceptions/` | `exceptions`, `policy-exceptions-disabled` |
| **Image Verification (Cosign/Notary)** | `pkg/image/verification/`, `pkg/image/verifiers/cpol/`, `pkg/image/verifiers/ivpol/`, `pkg/sigstoretuf/` | `verify-images`, `image-validating-policies`, `custom-sigstore`, `sigstore-custom-tuf`, `verify-manifests` |

---

### Task D — Existing Mapping Artifacts & Audit of Stale Paths

* **`.github/labels.yml` Audit:**
  * Contains 526 lines mapping file globs to labels.
  * **Stale Globs Identified (Do not exist on disk):**
    * `pkg/imageverification/**`, `pkg/imageverifycache/**`, `pkg/images/**` → Replaced by `pkg/image/verification/**`
    * `pkg/cosign/**`, `pkg/notary/**` → Replaced by `pkg/image/verifiers/cpol/{cosign,notary}/**` and `pkg/image/verifiers/ivpol/{cosign,notary}/**`
    * `pkg/registryclient/**` → Replaced by `cmd/internal/registry.go`
    * `cmd/tools/**` → Deleted
  * **`CODEOWNERS` Audit:** Stale references to `/pkg/cosign`, `/pkg/notary`, and a typo `/pkg/controller/report` (actual path: `/pkg/controllers/report`).
* **Actionable Phase 0 Deliverable:** Patch `.github/labels.yml` and `CODEOWNERS`, and extend `labels.yml` with `suites:` and `unit:` properties to serve as the unified source of truth for both PR labeling and test matrix generation.

---

## 2. Real-World AI CI/CD Threat Models & Defensive Architecture

Recent industry incidents (2025–2026) demonstrate that AI agents in CI/CD fail due to **harness configuration, excessive token privileges, and prompt injection**, not model jailbreaks:

```
                  ┌──────────────────────────────────────────────┐
                  │ UNTRUSTED INPUT (Issue / PR Body / Comments) │
                  └──────────────────────┬───────────────────────┘
                                         │
                                         ▼
                  ┌──────────────────────────────────────────────┐
                  │          QUARANTINED PARSER (NO TOKENS)      │
                  │  - Strips invisible Unicode & Camo URLs      │
                  │  - Emits validated JSON schema only          │
                  └──────────────────────┬───────────────────────┘
                                         │
                                         ▼
                  ┌──────────────────────────────────────────────┐
                  │       PRIVILEGED ENGINE (HARNESS)            │
                  │  - Validates action via kubectl-kyverno      │
                  │  - Network egress firewall (Squid/AWF)       │
                  │  - Ephemeral, least-privilege token          │
                  └──────────────────────┬───────────────────────┘
                                         │
                                         ▼
                  ┌──────────────────────────────────────────────┐
                  │ AUDITABLE LEDGER & GITSIGN SIGNED COMMITS    │
                  └──────────────────────────────────────────────┘
```

### Precedent Breakdown & Controls
1. **GitHub MCP Server Indirect Injection (Invariant Labs, May 2025):** Public issue injected instructions to read private repos using user-scoped PATs.  
   * *Mitigation:* Session-scoped, repo-isolated GitHub App tokens with read-only defaults.
2. **VS Code Copilot "YOLO Mode" RCE (CVE-2025-53773):** Injected comments added `"chat.tools.autoApprove": true` to `.vscode/settings.json`.  
   * *Mitigation:* Immutable configuration boundaries; agent cannot modify its own harness or workflow definitions.
3. **CamoLeak (CVE-2025-59145):** Exfiltrated private repo data via pre-signed Camo image proxy URLs.  
   * *Mitigation:* Strip invisible markdown/HTML tags; disable outbound media rendering in agent output.
4. **"Clinejection" (Adnan Khan, Feb 2026):** Issue triage bot with shell permissions ran `npm install` from an attacker fork, poisoned GitHub Actions cache, and hijacked release tokens (`NPM_RELEASE_TOKEN`).  
   * *Mitigation:* Never give shell execution to triage bots; partition Actions cache keys so untrusted workflows never share cache with credentialed release flows.
5. **Claude Code Action OIDC Bypass (RyotaK / GMO Flatt, June 2026):** Unconditionally trusted GitHub App actors, exposing OIDC tokens via `/proc/self/environ`.  
   * *Mitigation:* Enforce `checkHumanActor` validation; scrub environment variables in child processes.
6. **Lethal Trifecta ([Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)):** Combining Private Data + Untrusted Content + External Communication = Instant Compromise.  
   * *Mitigation:* Separate unprivileged triage parser from privileged action executor. Validate all actions with `kubectl-kyverno` before executing.

---

## 3. Detailed Resolution of Handwritten Notes

### Sheet 1: PR Hygiene, Security & Metrics
* **CI Flakiness Time-Series & Digest:** Compute flakiness using the open-to-close lifespan of `workflow-failure` issues. Short-lived issues (opening and auto-closing on the next run without intervening fix commits) provide a free flake metric. Publish a weekly delta report as a single, pinned, in-place edited GitHub Issue.
* **Cache Reaper:** Automated workflow on PR close/merge calling `DELETE /repos/{owner}/{repo}/actions/caches?ref=refs/pull/{pr}/merge` to stay within the 10 GB repository limit.
* **Secret & Vulnerability Scanners:**
  * *Secrets:* Gitleaks (via CodeRabbit on PRs).
  * *SAST:* Semgrep (daily cron).
  * *Container:* Trivy rootfs scan.
  * *PR Dependencies:* GitHub `actions/dependency-review-action` fails newly added vulnerable dependencies at PR time.
* **Dependency Review (SDK/API vs Container):** Call GitHub's Dependency Graph API (`GET /repos/{owner}/{repo}/dependency-graph/compare/{basehead}`) to receive structured JSON for policy decisions rather than scraping CLI output.
* **Merge Conflict Labeling:** Already implemented in `pr-labelling.yaml` via `eps1lon/actions-label-merge-conflict`.

---

### Sheet 2: Metadata, Boundaries & Lifecycle
* **Generated Manifest (`agent-manifest.json`):** Generate `.github/agent-manifest.json` from the `Makefile` via `make codegen-agent-manifest` and verify in CI via `make verify-codegen` to prevent drift.
* **Per-Directory `AGENTS.md`:** Split root `AGENTS.md` into local stubs (`pkg/engine/`, `pkg/webhooks/`, `pkg/controllers/`, `api/`, `test/conformance/`). Compatible with CodeRabbit, Codex, Claude Code, and OpenHands.
* **Safe Automation Boundary via Kyverno Policy:** Dogfood Kyverno by expressing agent permissions in a `ClusterPolicy` and evaluating proposed agent actions using `kubectl-kyverno apply` in CI.
* **CI/Test Metadata:** Extend `.github/labels.yml` with `suites:` and `unit:` fields to serve as the unified source of truth.
* **Release Changelog Drafting:** Adopt `release-drafter/release-drafter` configured against `semantic.yml` conventional commit types.
* **Flake Detection & Quarantine Loop:** Ingest Chainsaw JUnit logs into a `ci-metrics` Git branch. Automatically open quarantine PRs with a mandatory owner and 14-day expiry date. Re-run quarantined tests nightly; auto-open de-quarantine PRs after 7 consecutive green runs.
* **DCO Guidance:** On DCO check failure, post the exact copy-paste command:
  `git rebase --signoff origin/main && git push --force-with-lease`

---

### Sheet 3: Reproduction, Retries & Triage
* **CI Retrigger & Prow Commands:** Enable `/retest` in `comment-prow.yaml`. Enforce **Retry-with-Telemetry (maximum 1 retry)** to prevent hiding intermittent race conditions.
* **Automated KinD Repro → Chainsaw Test Pipeline:**
  1. Extract YAML and steps from structured issue forms (`bug-cli.yaml`, `bug-webhook.yaml`).
  2. Assemble an ephemeral `chainsaw-test.yaml` using `_step-templates/create-policy.yaml`.
  3. Execute in an isolated KinD container via `.github/actions/tests/conformance/run`.
  4. Post reproduction output to the issue.
  5. If reproduced, generate a ready-to-merge regression test fixture.
* **Slack / Discussions Q&A Assistant:** Enforce **Citation-Mandatory Grounding**. Answers must link directly to specific markdown headers in `docs/dev/` or `kyverno/website`. If search similarity is low, the bot remains silent and applies a `needs-maintainer` label.

---

## 4. Build-vs-Adopt Decision Matrix

| Capability | Adopt Existing Tooling | Build In-Tree (Custom Engine) | Rationale |
| :--- | :--- | :--- | :--- |
| **PR Code Review & Summaries** | **CodeRabbit (Pro free for OSS)** | ❌ *Do not build* | Natively reads `AGENTS.md` and provides path-based review instructions without maintaining LLM review harnesses. |
| **Secret Scanning** | **CodeRabbit / Gitleaks** | ❌ *Do not build* | Integrated into PR review pipeline. |
| **PR Dependency Gating** | **`actions/dependency-review-action`** | ❌ *Do not build* | Standard GitHub native PR check. |
| **Release Changelog Drafting** | **`release-drafter/release-drafter`** | ❌ *Do not build* | Consumes `semantic.yml` types out-of-the-box. |
| **Scoped Chainsaw Selection** | ❌ *Nothing exists* | **✅ Build In-Tree** | Computes diff-to-suite mapping and generates dynamic Chainsaw matrix. |
| **Flake Quarantine & Expiry** | ❌ *Nothing exists* | **✅ Build In-Tree** | Computes flip rates and manages automated quarantine/de-quarantine PR loop. |
| **KinD Repro → Chainsaw Test**| ❌ *Nothing exists* | **✅ Build In-Tree** | Synthesizes bug reports into runnable regression test artifacts. |
| **Codegen Self-Healing** | ❌ *Nothing exists* | **✅ Build In-Tree** | Auto-commits generated patches directly back to contributor branches. |
| **Kyverno-Governed Boundary** | ❌ *Nothing exists* | **✅ Build In-Tree** | Validates agent action plans using `kubectl-kyverno`. |

---

## 5. Phased Implementation Roadmap

```
Weeks:   1   2   3   4   5   6   7   8   9   10  11  12
Phase 0: [=== Audit, Manifest & Metadata ===]
Phase 1:         [=== Deterministic Core (Scoped CI) ===]
Phase 2:                     [=== Flake Lifecycle Engine ===]
Phase 3:                                 [=== Repro & Triage ===]
Phase 4:                                             [== Q&A & Wrap ==]
```

### Phase 0: Audit, Manifest & Metadata (Weeks 1–3)
* **Deliverable 0.1:** Finalize Build-vs-Adopt ADR (configure `.coderabbit.yaml` rules, review instructions, and Gitleaks).
* **Deliverable 0.2:** Split `AGENTS.md` into per-directory stubs (`pkg/engine/`, `pkg/webhooks/`, `pkg/controllers/`, `api/`, `test/conformance/`) and add Makefile target verification check.
* **Deliverable 0.3:** Fix stale globs in `.github/labels.yml` and `CODEOWNERS`; extend `labels.yml` with `suites:` and `unit:` mappings.
* **Deliverable 0.4:** Implement `make codegen-agent-manifest` and hook into `make verify-codegen`.

### Phase 1: Keystone Deterministic CI Automation (Weeks 3–6)
* **Deliverable 1.1:** **PR-Time Scoped Conformance Selector:** Action that parses changed files against `labels.yml` and dynamically emits a scoped Chainsaw matrix (reducing execution from 52 suites to ~3–5 per PR).
* **Deliverable 1.2:** **Codegen Self-Healing Agent:** Auto-commits `codegen-code.patch` / `codegen-docs.patch` back to contributor PR branches upon `check-codegen.yaml` failure.
* **Deliverable 1.3:** **PR Dependency Gate & Cache Reaper:** Add `dependency-review-action` and automated Actions cache deletion on PR close.
* **Deliverable 1.4:** **Release Drafter & `/retest` Integration:** Configure `release-drafter` and enable `/retest` in `comment-prow.yaml`.

### Phase 2: Flake Lifecycle & Quarantine Engine (Weeks 6–9)
* **Deliverable 2.1:** **JUnit Ingestion & Flake Metric Pipeline:** Ingest test results into a `ci-metrics` branch; compute pass/fail flip-rate statistics.
* **Deliverable 2.2:** **Automated Quarantine Loop:** Generate PRs updating `quarantined-tests` with mandatory owner assignments and 14-day expiry limits.
* **Deliverable 2.3:** **Nightly Soak Lane:** Run quarantined tests in an isolated, non-blocking lane; auto-open de-quarantine PRs after 7 consecutive green runs.
* **Deliverable 2.4:** **Maintainer Weekly Digest:** Generate a weekly summary issue detailing CI stability, flaky test health, and PR backlog metrics.

### Phase 3: Issue Triage & KinD Repro-to-Chainsaw Pipeline (Weeks 9–11)
* **Deliverable 3.1:** **Structured Issue Parser:** Extract policy YAML and resource YAML from GitHub issue forms into validated JSON schemas.
* **Deliverable 3.2:** **Ephemeral KinD Reproduction Harness:** Run extracted test artifacts in an isolated KinD runner and post reproduction output.
* **Deliverable 3.3:** **Chainsaw Test Synthesizer:** Generate a ready-to-merge `chainsaw-test.yaml` regression test fixture when a bug reproduces.

### Phase 4: Grounded Q&A Assistant, Evaluation & Mentorship Wrap-up (Weeks 11–12)
* **Deliverable 4.1:** **Citation-Grounded Q&A Bot:** Implement documentation-grounded search for Slack/Discussions with strict citations and maintainer escalation fallbacks.
* **Deliverable 4.2:** **Project Documentation & Final Report:** Publish comprehensive documentation, benchmark runner-minute savings, and present final metrics to the Kyverno community.

---

## 6. Measurable Success Metrics

| Metric | Baseline (Pre-Mentorship) | Target (Post-Mentorship) | Measurement Method |
| :--- | :--- | :--- | :--- |
| **PR Conformance Coverage** | **0%** (conformance does not run on PRs) | **100% of functional PRs** run relevant scoped suites | GitHub Actions PR run logs |
| **Post-Merge `workflow-failure` Issues**| ~20–30 issues/month | **< 5 issues/month** | GitHub Issues API query on `workflow-failure` |
| **Median PR CI Runner Minutes** | ~150+ runner allocations (full suite) | **< 15 runner allocations (scoped)** | GitHub Actions workflow telemetry |
| **Codegen Contributor Roundtrips** | 1–2 days waiting for manual re-runs | **< 5 minutes** (automated patch commit) | Time from failed codegen check to push |
| **Flaky Test Lifespan** | Indefinite (hardcoded skips in repo) | **< 14 days** (automated expiry / de-quarantine) | `ci-metrics` branch audit trail |
