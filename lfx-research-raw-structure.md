# Raw research — repo structure, AGENTS.md, task index, safe-automation boundaries

This is the unedited output from the research agent covering: AGENTS.md deepening, machine-readable task index, safe-automation boundaries, and generated-code inventory. Synthesized findings from this file are folded into `kyverno-lfx-research.md` §9-10; this file is the full raw evidence trail for anyone who wants to check the sourcing.

---

## 1. AGENTS.md (246 lines, root) — exact contents

Sections, in order:
1. Title + intro ("context for AI coding agents")
2. **Project Overview** — language Go, module `github.com/kyverno/kyverno`, Apache 2.0
3. **Repository Structure** — annotated tree (api/, cmd/, pkg/, ext/, charts/, config/, test/, docs/, scripts/, hack/)
4. **Build System** — tables of make targets:
   - Key Build Commands (build-all, build-kyverno, build-kyverno-init, build-cli, build-cleanup-controller, build-reports-controller, build-background-controller, install-tools, clean-tools)
   - Formatting & Linting (fmt, vet, imports, fmt-check, imports-check, unused-package-check) + prose on golangci-lint v2, gosec/misspell/paralleltest/unconvert/errname/importas, formatters gci/gofmt/gofumpt/goimports
   - **Pre-commit checklist**: run `make imports fmt`; run `make imports-check fmt-check` and ensure pass
   - **Pre-PR checks**: run `make codegen-all-code` then `make verify-codegen`; run `./.tools/golangci-lint run`
   - Testing table (test-unit, test-cli, test-cli-local, test-clean, helm-test) + prose on unit/CLI/conformance/fuzz test locations
   - Code Generation table (codegen-all, codegen-all-code, codegen-api-all, codegen-client-all, codegen-crds-all, codegen-helm-all, verify-codegen) + generated file patterns list
   - Docker Images (ko) table (ko-build-all, ko-build-kyverno, ko-publish-all)
   - Local Development with KinD table (kind-create-cluster, kind-delete-cluster, kind-load-all, kind-deploy-kyverno, kind-deploy-all) + KIND_IMAGE/KIND_NAME env var note
5. **Architecture** — one paragraph per controller (Admission, Background, Reports, Cleanup, Init, CLI) with cmd/ paths; notes pkg/controllers, pkg/webhooks, pkg/engine locations
6. **API Design Rules** (verbatim, see below)
7. **Coding Conventions** — import aliases (importas), logging (logr/zerologr, levels), feature flags (pkg/toggle), CGO_ENABLED=0, generated-code rule
8. **Pull Request Guidelines** — proof manifests, doc PR on website repo, CLI test manifests, chainsaw conformance tests, verify-codegen
9. **PR Failure-Prevention Checklist (DCO + Codegen + CI)** — 5-point checklist: DCO signoff (`git commit -s`, `git rebase --signoff`), full codegen verification, pre-push checks, keep branch clean, confirm GH checks (`gh pr checks <pr-number> -R kyverno/kyverno`)
10. **Useful References** — links to DEVELOPMENT.md, CONTRIBUTING.md, docs/dev/api, docs/dev/controllers, docs/dev/logging, docs/dev/feature-flags, docs/dev/reports, kyverno.io

### API Design Rules — VERBATIM (lines 176-183)

> - API types live in `api/` with versioned packages
> - API groups: `kyverno.io`, `policies.kyverno.io`, `policyreport.io`, `reports.kyverno.io`
> - New resource types must NOT be added to `kyverno.io/v1`; use `v2alpha1` and promote as they stabilize
> - New attributes can be added without a new version
> - Attributes cannot be deleted or modified in a version; deprecate and remove after 3 minor releases
> - Newer API versions may reference older stable types, but not vice versa

This is the exact rule set an "API-compatibility linter" would encode as machine checks:
- rule A: no new CRD/Kind added under `api/kyverno/v1` (or any version already marked stable) — diff-based check on new `_types.go` files declaring `+kubebuilder:object:root=true` in a stable version dir
- rule B: struct field removal/type-change in an existing version's `_types.go` is a hard fail unless paired with a `// Deprecated:` comment trail spanning ≥3 minor releases
- rule C: import-direction check — a file in an older version package (e.g. `v1`) must not import a newer version package (e.g. `v2alpha1`)

### What's missing → belongs in per-directory AGENTS.md files

None of these are covered anywhere in the root AGENTS.md today (it only has repo-wide build/lint/API rules):

- **pkg/cel/** (910 churn, #1 priority) — no mention at all in root AGENTS.md beyond a 1-line tree annotation ("CEL-based policy evaluation"). Missing: CEL expression conventions, which policy kinds live here (ValidatingPolicy/vpol, MutatingPolicy/mpol, ImageValidatingPolicy/ivpol, GeneratingPolicy/gpol, DeletingPolicy/dpol — matches the `test-cli-local-{vpols,gpols,mpols,ivpols,dpols,vaps,maps}` targets), relationship to upstream `k8s.io/apiserver/pkg/cel` and Kubernetes ValidatingAdmissionPolicy/MutatingAdmissionPolicy, how autogen interacts with CEL policies, CEL library registration pattern, test conventions distinct from JMESPath-based v1 policies.
- **pkg/engine/** — root doc only says "policy engine (rule evaluation, matching, context)" once in the tree and once in Architecture. Missing: engine context/variable resolution flow, rule-type dispatch (validate/mutate/generate/verifyImages), how JMESPath vs CEL evaluation paths differ, background vs admission evaluation entry points, image verification hook points (cosign/notary), performance-sensitive hot paths.
- **pkg/webhooks/** — only in tree + architecture prose. Missing: webhook registration/admission flow, resource/policy webhook distinction, TLS cert rotation dependency on pkg/tls, how webhook config is generated (relates to config/webhooks or charts), timeoutSeconds/failurePolicy conventions, testing webhook handlers without a live apiserver.
- **pkg/controllers/** — only named in Architecture prose ("Controller code is primarily in pkg/controllers/"); no dedicated section. Missing: controller-runtime-style patterns used (informers/workqueues), subpackage map (admissionpolicygenerator, certmanager, cleanup, deleting, exceptions, generic, globalcontext, metrics, policycache, report, etc.), event recording conventions, leader-election notes. Also: CODEOWNERS' only line for this area (`/pkg/controller/report`, singular) is a dead typo — an AGENTS.md here should explicitly state real ownership since CODEOWNERS doesn't route reviews here correctly.
- **test/conformance/** (3996 churn, by far the highest churn dir in the repo) — root AGENTS.md mentions chainsaw once ("conformance/e2e tests (chainsaw)") with no structural guidance. Missing: chainsaw test-case anatomy (`chainsaw-test.yaml`, step ordering, assert/error manifests), naming/directory conventions (mirrors feature area, `-deprecated` suffix pattern for pre-autogen-v2 checks seen throughout test/conformance/chainsaw/autogen/*), how to run a single test vs the whole suite, the README.md files scattered per-test-case (many small per-case README.md files exist, e.g. test/conformance/chainsaw/autogen/*/README.md).
- **api/** — root AGENTS.md's "API Design Rules" section is repo-wide/versioning only; missing per-group specifics: which of the 4 API groups (kyverno.io, policies.kyverno.io, policyreport.io via wgpolicyk8s.io, reports.kyverno.io) is CEL-based (`policies.kyverno.io`) vs JMESPath-based (`kyverno.io`), codegen tags required in `_types.go` (kubebuilder markers), where `zz_generated.*` and CRD YAML land per group, deepcopy-gen/register-gen invocation per group.

## 2. Governance / process documents

- **CONTRIBUTING.md** — points to external `kyverno/community` CODE_OF_CONDUCT; locally states: docs live in separate `kyverno/website` repo; PR must include proof manifests, doc issue/PR on website repo for new/changed functionality, CLI test manifest + chainsaw conformance tests for e2e-testable changes, and "indicate which release this PR is triaged for" (maintainer-only step). Release process deferred to kyverno.io/docs/releases/.
- **GOVERNANCE.md** — one line, fully deferred to `github.com/kyverno/community/blob/main/GOVERNANCE.md`. No local governance rules.
- **MAINTAINERS.md** — one line, deferred to `github.com/kyverno/community/blob/main/MAINTAINERS.md`. No local content (surprising: the real maintainer roster lives in OWNERS.md instead, see below).
- **OWNERS.md** — contains the actual local maintainer/reviewer table (not MAINTAINERS.md):
  - 9 Maintainers with named "Domain Ownership": Jim Bugwadia (Validation/CLI/Docs), Shuting Zhao (Engine/Admission/Background/Mutation/Events), Charles-Edouard Brétéché (Engine/CEL/Autogen/Validation/CLI/Helm), Vishal Choudhary (Engine/Image Verification), Mariam Fahmy (Generation/Admission Policy/Policy Exception), Frank Jogeleit (Cleanup/Reporting/Metrics/Testing), Liang Deng (Pod Security Admission), Yugandhar Suthari (Global Context/CLI), Xu Liu (Global Context/CEL Validation)
  - 1 Reviewer: Ammar Yasser (CEL, Reporting) — reviewer, not maintainer/approver tier
  - This is a two-tier structure (Maintainer vs Reviewer) that constrains automation: an automation system proposing PRs should route CEL-touching changes to Brétéché/Suthari/Liu/Yasser per this domain map, distinct from the generic CODEOWNERS `@kyverno/kyverno-core-maintainers` team ping.
- **DEPENDENCY-POLICY.md** (already summarized) — additional review/approval detail beyond what was previously extracted: "New dependencies require maintainer approval during code review" (explicit gate on any go.mod addition); Dependabot updates "reviewed by maintainers and must pass CI before merging" — i.e., dependency bumps are NOT auto-mergeable even though Dependabot is automated, they still need a human maintainer approval + green CI. This matters for the ai-maintainer.yaml "agent-autonomous" bucket: an agent could open the bump PR / rebase it, but merge still requires maintainer sign-off.
- No SLA/branch-protection file found beyond the Vulnerability Remediation SLA table in DEPENDENCY-POLICY.md (Critical 7d / High 14d / Medium 28d / Low next-minor — targets not guarantees).

## 3. Generated-code inventory

| Pattern | Location | Count | Notes |
|---|---|---|---|
| `zz_generated.deepcopy.go` + `zz_generated.register.go` | `api/kyverno/{v1,v1beta1,v2,v2alpha1,v2beta1}`, `api/policyreport/v1alpha2`, `api/reports/v1` | 14 files (7 packages × 2 files) | via `codegen-api-deepcopy` / `codegen-api-register` |
| Generated clientset/listers/informers | `pkg/client/**/*.go` | **178 files** | via `codegen-client-all` (clientset, listers, informers, wrappers) |
| CRD manifests | `config/crds/{kyverno,policies.kyverno.io,policyreport,reports}/*.yaml` | **21 files** | via `codegen-crds-all` |
| Helm CRDs subchart | `charts/kyverno/charts/crds/templates/{kyverno.io,reports.kyverno.io,wgpolicyk8s.io}/*.yaml` | 11 files | via `codegen-helm-crds`; note: only 3 of 4 API groups represented here (policies.kyverno.io CEL CRDs not mirrored into this Helm subchart — worth flagging as a possible gap) |
| API reference docs (HTML) | `docs/user/crd/*.html` | 11 files (index + 10 per-group) | via `codegen-api-docs`: target does `rm -rf docs/user/crd && mkdir -p docs/user/crd`, runs `gen-crd-api-reference-docs`, outputs `docs/user/crd/index.html`, then runs `genref` from `docs/user` |
| CLI API docs (HTML) | `docs/crd/v1/index.html` | 1 file | via `codegen-cli-api-docs` (separate target/output tree from the main API docs) |
| Helm chart docs | `charts/kyverno/README.md`, `charts/kyverno-policies/README.md` | 2 files | via `codegen-helm-docs` ($(HELM_DOCS) tool) |
| CLI command docs | `docs/user/cli/commands/*.md` | ~25 files | via `codegen-cli-docs` (built off `$(CLI_BIN)`, i.e. runs the built CLI's own doc-gen) |

Total machine-generated surface actively tracked in git: roughly **14 + 178 + 21 + 11 + 11 + 1 + 2 + ~25 ≈ 263 files**, none of which should ever be hand-edited (AGENTS.md already states this rule for `zz_generated.*` and `pkg/client/`, but not for the CRD/doc outputs — another AGENTS.md gap).

### check-codegen.yaml (`.github/workflows/check-codegen.yaml`) — exact patch mechanism

Two independent jobs, both triggered on PR + push to `main`/`release-*`:

1. **verify-codegen-code**: runs `make codegen-all-code` → `make verify-codegen`. On failure: `git diff > codegen-code.patch`, writes a message to `$GITHUB_STEP_SUMMARY` ("Codegen is out of date. Download the codegen-code-patch artifact, then run `git apply codegen-code.patch` and commit the result."), uploads artifact named **`codegen-code-patch`** containing `codegen-code.patch`.
2. **verify-codegen-docs**: same pattern but runs `make codegen-all-docs` first, produces **`codegen-docs-patch`** artifact containing `codegen-docs.patch`.
3. **sync-issue**: on push (not PR) and `always()`, opens/updates a failure-tracking GitHub issue via a reusable `./.github/actions/workflow/failure-issue` action if either job failed.

This is precisely the mechanism an AI maintainer agent should short-circuit: instead of a human downloading the artifact and running `git apply`, the agent can run `make codegen-all-code`/`make codegen-all-docs` + `make verify-codegen` itself in the same PR branch and push a fixup commit — the patch-artifact path only exists because no automated committer currently closes that loop.

## 4. `.github/agent-manifest.json` — draft (real Makefile targets only)

All target names below were grep-verified to exist in `Makefile` in this session.

```json
{
  "$schema": "https://kyverno.io/schemas/agent-manifest-v1.json",
  "generated_from": "Makefile",
  "tasks": {
    "build-all": { "cmd": "make build-all", "needs_cluster": false, "approx_minutes": 3, "autofixable": false, "verify": "exit code 0; binaries present under cmd/*/" },
    "build-kyverno": { "cmd": "make build-kyverno", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "cmd/kyverno/kyverno exists" },
    "build-cli": { "cmd": "make build-cli", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "cmd/cli/kubectl-kyverno/kubectl-kyverno exists" },
    "fmt": { "cmd": "make imports fmt", "needs_cluster": false, "approx_minutes": 1, "autofixable": true, "verify": "make imports-check fmt-check exits 0" },
    "fmt-check": { "cmd": "make imports-check fmt-check", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "empty git diff" },
    "vet": { "cmd": "make vet", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "exit code 0" },
    "unused-package-check": { "cmd": "make unused-package-check", "needs_cluster": false, "approx_minutes": 1, "autofixable": true, "verify": "go.mod/go.sum unchanged after go mod tidy" },
    "lint": { "cmd": "./.tools/golangci-lint run", "needs_cluster": false, "approx_minutes": 3, "autofixable": false, "verify": "exit code 0 (install first via make install-tools)" },
    "codegen-all-code": { "cmd": "make codegen-all-code", "needs_cluster": false, "approx_minutes": 4, "autofixable": true, "verify": "make verify-codegen exits 0 with empty diff" },
    "codegen-all-docs": { "cmd": "make codegen-all-docs", "needs_cluster": false, "approx_minutes": 2, "autofixable": true, "verify": "make verify-codegen exits 0 with empty diff" },
    "verify-codegen": { "cmd": "make verify-codegen", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "exit code 0, no git diff" },
    "test-unit": { "cmd": "make test-unit", "needs_cluster": false, "approx_minutes": 8, "autofixable": false, "verify": "coverage.out produced, exit code 0" },
    "test-cli-local": { "cmd": "make test-cli-local", "needs_cluster": false, "approx_minutes": 5, "autofixable": false, "verify": "exit code 0 across all test-cli-local-* sub-targets" },
    "test-cli-local-vpols": { "cmd": "make test-cli-local-vpols", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "exit code 0", "note": "CEL ValidatingPolicy CLI fixtures — pair with pkg/cel changes" },
    "test-cli-local-mpols": { "cmd": "make test-cli-local-mpols", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "exit code 0" },
    "test-cli-local-ivpols": { "cmd": "make test-cli-local-ivpols", "needs_cluster": false, "approx_minutes": 1, "autofixable": false, "verify": "exit code 0" },
    "helm-test": { "cmd": "make helm-test", "needs_cluster": false, "approx_minutes": 2, "autofixable": false, "verify": "exit code 0" },
    "kind-create-cluster": { "cmd": "make kind-create-cluster", "needs_cluster": true, "approx_minutes": 2, "autofixable": false, "verify": "kubectl cluster-info succeeds against KIND_NAME context" },
    "kind-deploy-all": { "cmd": "make kind-deploy-all", "needs_cluster": true, "approx_minutes": 6, "autofixable": false, "verify": "kyverno + kyverno-policies helm releases in Deployed state" },
    "kind-delete-cluster": { "cmd": "make kind-delete-cluster", "needs_cluster": true, "approx_minutes": 1, "autofixable": false, "verify": "kind get clusters no longer lists KIND_NAME" },
    "ko-build-all": { "cmd": "make ko-build-all", "needs_cluster": false, "approx_minutes": 5, "autofixable": false, "verify": "images present in local docker/ko.local" }
  }
}
```

Sample-verified target existence (grepped Makefile directly): `build-all`, `build-kyverno`, `build-cli`, `build-kyverno-init`, `build-cleanup-controller`, `build-reports-controller`, `build-background-controller`, `test-unit`, `test-cli`, `test-cli-local`, `test-cli-local-vpols/gpols/mpols/ivpols/dpols/vaps/maps`, `helm-test`, `kind-create-cluster`, `kind-delete-cluster`, `kind-load-all`, `kind-deploy-kyverno`, `kind-deploy-all`, `dev-lab-*` (7 targets + all), `codegen-api-register/deepcopy/docs/all`, `codegen-client-clientset/listers/informers/wrappers/all`, `codegen-crds-kyverno/policies/policyreport/reports/cli/all`, `codegen-cli-api-group-resources/crds/docs/api-docs/all`, `codegen-helm-crds/docs/all`, `codegen-manifest-install-latest/debug/release/all`, `codegen-fix-tests/policies/all`, `codegen-all-code`, `codegen-all-docs`, `codegen-all`, `codegen-api-bump`, `fmt`, `vet`, `imports`, `fmt-check`, `imports-check`, `unused-package-check`, `verify-codegen`, `release-notes`, `help`. Total 123 `target: ## desc`-documented targets confirmed (grep count).

## 5. `.github/ai-maintainer.yaml` — draft (all paths verified to exist)

```yaml
# .github/ai-maintainer.yaml
# Policy boundaries for an AI maintainer assistant operating on this repo.

never_touch:
  # Security-sensitive: image signature verification. Any change here affects
  # supply-chain trust decisions and must go through a human security reviewer.
  - pkg/image/verifiers/cpol/cosign/**
  - pkg/image/verifiers/cpol/notary/**
  - pkg/image/verifiers/ivpol/cosign/**
  - pkg/image/verifiers/ivpol/notary/**
  # Stable API version: attribute add/remove/modify rules are enforced by
  # policy, not by CI today — treat as never-touch for an autonomous agent.
  - api/kyverno/v1/**
  # RBAC / TLS surfaces
  - charts/kyverno/templates/rbac/**
  - pkg/tls/**
  - pkg/utils/tls/**
  - config/e2e/rbac.yaml

human_review_required:
  - api/**                       # all other API version packages (v1beta1, v2, v2alpha1, v2beta1) + policies.kyverno.io, reports.kyverno.io, policyreport
  - pkg/engine/**
  - pkg/webhooks/**
  - pkg/cel/**
  - pkg/controllers/**
  - pkg/background/**
  - pkg/policy/mutate/**
  - go.mod
  - go.sum                       # new dependency additions require maintainer approval per DEPENDENCY-POLICY.md
  - charts/kyverno/values.yaml
  - .golangci.yml

agent_autonomous:
  # Fully generated, mechanically regenerable, safe to auto-commit if
  # `make verify-codegen` passes clean.
  - "api/**/zz_generated.deepcopy.go"
  - "api/**/zz_generated.register.go"
  - pkg/client/**
  - config/crds/**
  - charts/kyverno/charts/crds/templates/**
  - docs/user/crd/**
  - docs/crd/**
  - docs/user/cli/commands/**
  - charts/kyverno/README.md
  - charts/kyverno-policies/README.md
  # Dependency bumps: agent may open/rebase the PR, but merge still requires
  # a maintainer approval per DEPENDENCY-POLICY.md — mark autonomous only up
  # to "propose", not "merge".
  - .github/dependabot.yml   # config only, not the bumps themselves
```

Every `never_touch` and generated path above was directly verified to exist via `Glob`/`find` in this session (cpol/cosign, cpol/notary, ivpol/cosign, ivpol/notary each contain real .go files; `api/kyverno/v1` contains real type files; `charts/kyverno/templates/rbac`, `pkg/tls`, `pkg/utils/tls`, `config/e2e/rbac.yaml` all exist). Note: no dedicated `tls/` top-level dir exists in this repo — TLS code lives under `pkg/tls/` and `pkg/config/tls.go`; listed the real paths instead of guessing a non-existent `tls/` directory.

## 6. CODEOWNERS (23 lines) — coverage gaps beyond the 3 known-dead lines

Full file has 23 real `path owners` lines (plus the `*` catch-all). Already-known dead lines (3): `/pkg/cosign`, `/pkg/notary` (code moved to `pkg/image/verifiers/{cpol,ivpol}/{cosign,notary}`), `/pkg/controller/report` (typo for plural `pkg/controllers/report`).

Cross-referencing the 12-month churn list against the 23 real path prefixes (`/api`, `/api/kyverno/v1`, `/charts`, `/docs`, `/pkg/autogen`, `/pkg/background`, `/pkg/engine`, `/pkg/event`, `/pkg/globalcontext`, `/pkg/policy/mutate`, `/pkg/pss`, `/pkg/utils`, `/pkg/userinfo`, `/pkg/validation`, `/pkg/webhooks`, `/test`, `/pkg/cosign`(dead), `/pkg/notary`(dead), `/cmd/cli/`, `/pkg/openreports`, `/pkg/controller/report`(dead), `/pkg/cel`, `/pkg/admissionpolicy`):

**High-churn directories with NO CODEOWNERS coverage at all** (not even via a broader prefix):
- **`pkg/controllers/`** (303 churn, plural, real dir with subpackages `admissionpolicygenerator`, `certmanager`, `cleanup`, `deleting`, `exceptions`, `generic`, `globalcontext`, `metrics`, `policycache`, `report`, etc.) — the only line that looks related, `/pkg/controller/report`, is singular and matches zero files. So this entire high-churn controller tree falls through to the bare `*` catch-all (`@kyverno/kyverno-core-maintainers` team only, no named domain owner), despite `docs/dev/controllers/README.md` clearly being a first-class design area.
- **`pkg/image/`** (175 churn; contains `verification` and `verifiers/{cpol,ivpol}/{cosign,notary}`) — no CODEOWNERS line for `/pkg/image` at all. Image verification is arguably the most security-sensitive code in the repo (supply-chain signature checks) yet has zero explicit review routing beyond the wildcard.
- **`pkg/clients/`** (225 churn; client wrappers with tracing/metrics, subpackages `aggregator`, `apiserver`, `dclient`, `dynamic`, `kube`, `kyverno`, `metadata`) — no line; distinct from the generated `pkg/client/` (singular) which is also uncovered but expected since it's generated.
- **`config/`** (config/crds 99 churn + config/e2e) — no `/config` line at all.

Covered (confirmed, not gaps): `pkg/cel` (line 24, explicit), `pkg/engine` (line 9), `pkg/webhooks` (line 17), `pkg/background` (line 8), `test/conformance` + `test/cli` (both fall under `/test`, line 18), `cmd/cli` (line 21), `charts/kyverno` + `charts/kyverno-policies` (both under `/charts`, line 5), `api/policies.kyverno.io` (falls under general `/api`, line 3, no more specific override exists for this group).

**Practical implication for the proposal**: an AI maintainer that routes review requests by CODEOWNERS alone would silently fall back to the generic core-maintainers team for four of the highest-churn areas in the repo (pkg/controllers, pkg/image, pkg/clients, config/crds) — including the single most security-sensitive one (pkg/image). The `.github/ai-maintainer.yaml` draft above compensates for this by hard-coding `pkg/image/verifiers/**` into `never_touch` regardless of what CODEOWNERS says.
