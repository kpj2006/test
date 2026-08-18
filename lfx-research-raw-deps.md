# Raw research — dependency automation, security scanning, supply chain, auto-merge

This is the unedited output from the research agent covering: Dependabot behavior, security scan gaps, supply-chain signing, and a concrete auto-merge predicate. Synthesized findings from this file are folded into `kyverno-lfx-research.md` §9-10; this file is the full raw evidence trail.

Repo: e:\kyverno (kyverno/kyverno). Research date: 2026-08-16.

---

## 1. Security/scan workflow table (FULL read of every file)

| Workflow (file) | Trigger | Scope / what it does | Fails on / gate | Gates PRs? |
|---|---|---|---|---|
| `scan-trivy.yaml` ("Trivy") | `push` to main/release-*, `workflow_dispatch` | `make build-all` then `trivy-action` rootfs scan of `./cmd/kyverno/kyverno` binary, severity CRITICAL,HIGH, ignore-unfixed, `.trivyignore.rego` policy filter | Uploads SARIF to GH code scanning (`upload-sarif`, category `kyverno`); trivy step itself does not `exit-code` fail the job (no `exit-code:` input set) — it just reports | **No.** Push-only, not on `pull_request` |
| `periodic-trivy.yaml` ("Trivy (periodic scan)") | `schedule` (daily 02:23 UTC), `workflow_dispatch` | Triggers the `trivy.yaml` workflow via `gh workflow run` against 3 refs: main, release-1.17, release-1.16, and watches the run | Same as scan-trivy | No — periodic only |
| `sync-trivy-issues.yaml` ("Trivy (sync issues)") | `workflow_run` completion of "Trivy" workflow, `workflow_dispatch` | `actions/github-script` job that lists open code-scanning alerts (tool_name Trivy) across main + latest release branch, opens/updates/closes GitHub issues (labels `codeql`,`security`) per alert, dedup via `<!-- codeql-id-N -->` marker in body | N/A (issue bookkeeping) | No |
| `scan-semgrep.yaml` ("Semgrep") | `schedule` (daily 04:30 UTC), `workflow_dispatch` | Runs `semgrep/semgrep:1.168.0` container, `p/security-audit` + `p/golang` rulesets, severity=ERROR, `--no-error` (never fails the job) | Uploads SARIF (category `semgrep`) to code scanning only | **No** — not on PR, and `--no-error` means it can't fail CI even if run on PR |
| `scan-scorecard.yaml` ("Scorecard") | `schedule` (weekly, Sat 01:30 UTC), `push` to main | OpenSSF Scorecard analysis, publishes results publicly (`publish_results: true`), uploads SARIF | Advisory only | No |
| `scan-sonarcloud.yaml` ("Sonarcloud") | `push` to main/release-* | SonarCloud scan (skipped if `SONAR_TOKEN` secret empty) | Advisory (SonarCloud quality gate is external, not enforced in this job) | No — push only |
| `scan-fossa.yml` ("FOSSA") | `push` to main | FOSSA license/dependency analysis (skipped if `FOSSA_API_KEY` empty) | Advisory | No |
| `check-sha-pinned-actions.yaml` ("Actions") | `pull_request` (main/release-*) **and** `push` | `zgosalvez/github-actions-ensure-sha-pinned-actions` — verifies every `uses:` in every workflow is pinned to a full commit SHA, with an explicit allowlist exception for `slsa-framework/slsa-github-generator` (which requires semver tags per SLSA spec) | **Yes, hard fail** if any action reference isn't SHA-pinned | **Yes — this is the only one of these that actually gates PRs** |

Every scan workflow (trivy/semgrep/scorecard/sonarcloud/fossa) ends with a `sync-issue` job (`if: always() && github.event_name == 'push'` or `'schedule'`) that calls the reusable `./.github/actions/workflow/failure-issue` action to auto-file a GitHub issue on failure. This is a maintainer-facing safety net, not a PR gate.

**Key structural finding: none of the vulnerability/license/quality scanners run on `pull_request` except the SHA-pin checker.** All of Trivy, Semgrep, Scorecard, SonarCloud, FOSSA run on push-to-main/release or on a schedule — i.e., *after* merge, or independent of PRs entirely. A vulnerable dependency bump can merge and only get caught post-merge (surfacing as an auto-filed `codeql`/`security` issue), not blocked pre-merge. This is the single biggest concrete "PR-time security gap" in the repo.

## 2. `.github/dependabot.yml` (full)

```yaml
version: 2
updates:
  - package-ecosystem: gomod
    directories: [/, /hack/controller-gen/, /hack/api-group-resources/]
    schedule: {interval: daily}
    rebase-strategy: disabled
    groups:
      kubernetes: {patterns: [k8s.io/*]}
      sigstore:   {patterns: [github.com/sigstore/sigstore/*]}
      otel:       {patterns: [go.opentelemetry.io/*]}
  - package-ecosystem: github-actions
    directories: [/, /.github/actions/*/]
    schedule: {interval: daily}
    rebase-strategy: disabled
```

Notes:
- Only 2 ecosystems: `gomod` (3 directories) and `github-actions` (2 directory globs). **No `docker` ecosystem, no Helm/`docker` for charts** — confirmed absent (see section 8).
- `rebase-strategy: disabled` on both — Dependabot will NOT auto-rebase a PR when its branch goes stale; a separate custom mechanism (`pr-branch-updater.yml`, section 3 below) was built to compensate.
- No `open-pr-limit` set anywhere → Dependabot's own default (5 per manager) applies per ecosystem/directory combo.
- No `ignore:` blocks at all — nothing is pinned/excluded from bumps.
- Grouping only covers 3 groups (kubernetes, sigstore, otel) out of ~107 direct deps — most direct deps (e.g. cosign, go-git, grpc, cel-go, testify, etc.) are NOT grouped and each produces its own individual PR.

## 3. Real GitHub data (via MCP, live, 2026-08-16)

- **Merged Dependabot PRs, last 90 days (since 2026-05-18):** `total_count: 62`.
- **All-time merged Dependabot PRs:** `total_count: 1401`.
- **Currently open Dependabot PRs:** `total_count: 1` — PR #17141, "bump google.golang.org/protobuf ... to 1.36.12", opened 2026-08-14.
- **Sample of 12 recent merged Dependabot PRs — commit-level inspection:**

  | PR | Title | Created → Closed | Time to merge | Extra commits after Dependabot's own commit |
  |---|---|---|---|---|
  | #17066 | bump Homebrew/actions/limit-pull-requests | 08-11 06:34 → 08-12 07:48 | ~25h | 1 merge-from-main (human) |
  | #17065 | bump zgosalvez/...ensure-sha-pinned-actions | 08-11 06:34 → 08-11 08:46 | ~2h | none captured |
  | #17067 | bump cel-go 0.30.0→0.31.0 | 08-11 06:35 → 08-11 08:09 | ~1.5h | 1 merge-from-main (human) |
  | #17008 | bump cosign/v3 3.1.2→3.1.3 | 08-10 06:36 → 08-10 07:51 | ~1.25h | 1 merge-from-main (human) |
  | #17007 | bump otel group (10 updates) | 08-10 06:36 → 08-10 07:14 | ~38m | 1 merge-from-main (by `kyverno-pr-updater[bot]`) |
  | #16966 | bump codeql-action/upload-sarif | 08-07 06:35 → 08-07 15:01 | ~8.5h | 1 merge-from-main (human) |
  | #16957 | bump Homebrew/actions/limit-pull-requests | 08-06 06:35 → 08-06 17:39 | ~11h | none (single commit) |
  | #16912 | bump sigstore-go 1.2.2→1.3.0 | 08-03 06:35 → 08-03 10:16 | ~3.75h | 2 merge-from-main (1 bot, 1 human) |
  | #16834 | bump docker/login-action 4.5.2→4.6.0 | 07-30 06:33 → 07-31 06:44 | ~24h | 2 merge-from-main (bot + human) |
  | #16835 | bump grpc 1.82.1→1.83.0 | 07-30 06:34 → 07-31 07:28 | ~25h | **6 merge-from-main commits** (5 bot, 1 human) over ~25h |
  | #16737 | bump kubernetes group (7 updates, 3 dirs) | 07-24 06:34 → 07-27 09:28 | ~3 days (spanned weekend) | 1 merge-from-main (human) |

  **Finding: in every sampled PR, Dependabot's own commit was never amended, force-pushed, or followed by a fix-up commit from a human.** CI appears to pass on Dependabot's first commit every time in this sample — no evidence of manual intervention to fix broken bumps. The only "extra" commits are `Merge branch 'main' into dependabot/...` — i.e., keeping the PR branch current while it waits in the merge queue for a human to click merge, not fixing failures.
  - Some of these merge-commits are authored by **`kyverno-pr-updater[bot]`** (the reusable `kyverno/.github/.github/workflows/pr-branch-updater.yml` invoked from `pr-branch-updater.yml`, triggered on every push to main).
  - Time-to-merge ranges from ~26 minutes to ~3 days; the 3-day outlier (#16737) spans a weekend, and the two ~24-25h outlier grpc/Homebrew-action bumps needed repeated re-merges from main (5-6 times) purely because `main` kept moving while the PR waited.
  - **Conclusion**: grouping (3 groups) reduces PR *count* for k8s/sigstore/otel, but the *volume problem is really a review-latency problem*, not a raw-count problem — even trivial patch bumps to single GitHub Actions sit for hours to a day waiting for a human click, and some need to be re-synced with main multiple times while waiting.

## 4. PR-time security gaps

- **`dependency-review-action`: absent.** No PR-time check that a Dependabot (or any) PR isn't introducing a package with a known CVE or incompatible license — only caught post-merge via push-triggered Trivy/FOSSA/Semgrep.
- **gitleaks / secret-scanning config: absent.** No repo-level custom gitleaks job; relies entirely on GitHub's platform-level secret scanning.
- **Supply-chain signing/SBOM (real and fairly strong)** — read `images-publish.yaml`, `release.yaml`, and the composite actions they call in full:
  - Both build the same 7 images (kyverno, kyverno-init, background-controller, cleanup-controller, cli, reports-controller, readiness-checker) via `ko` (no Dockerfiles).
  - Each image publish step does, in order: (1) `ko` build+push to `ghcr.io`, (2) generate a **CycloneDX SBOM** via `CycloneDX/gh-gomod-generate-sbom`, (3) upload the SBOM as a workflow artifact, (4) **keyless cosign sign** (OIDC-based, no static key) into a dedicated `ghcr.io/<owner>/signatures` repo, (5) `cosign attach sbom` to push the SBOM into `ghcr.io/<owner>/sbom`.
  - Both workflows also run **`aquasecurity/trivy-action` in `fs` mode over the whole repo** before publishing — a pre-publish scan of the source tree.
  - Each of the 7 images gets a **SLSA3 provenance attestation** via `slsa-framework/slsa-github-generator`'s reusable workflow. This is why that action is the sole allowlist exception in `check-sha-pinned-actions.yaml` (pinned to semver, not SHA, per SLSA spec requirement).
  - `release.yaml` additionally signs the **Helm/Flux install manifest** itself via `flux push artifact` + `cosign sign`.
  - **Net assessment**: supply-chain *output* signing (images + SBOM + SLSA3 provenance + manifest signing, all keyless/OIDC via cosign) is mature and comprehensive. The gap is entirely on the *input* side — nothing scans a dependency bump for known vulnerabilities before it merges.

## 5. `check-sha-pinned-actions.yaml` (full read)

- Triggers on **both** `pull_request` (main/release-*) and `push` — one of the only two workflows in the repo that gate PRs directly (the other being the standard CI test/lint/vet/codegen suite).
- Runs `zgosalvez/github-actions-ensure-sha-pinned-actions@c5fc58b...` (v5.0.7) with a single `allowlist` entry: `slsa-framework/slsa-github-generator`.
- **Enforces:** every `uses:` reference must resolve to a full 40-character commit SHA, not a mutable tag/branch/short SHA.
- **Implication for an AI agent editing workflow files:** the agent MUST NOT write `uses: some-action@v3`; it must always resolve to commit SHA + trailing `# vX.Y.Z` comment, for every action reference it touches or adds. An agent should defer SHA resolution to Dependabot's PRs for the `github-actions` ecosystem rather than hand-editing versions itself.

## 6. Go dependency counts / vendor / env vars

- `go.mod`: **107 direct dependencies** + **305 indirect dependencies**. Total require-line count: 413.
- `go.sum`: 1318 lines.
- `vendor/` directory: **confirmed absent**.
- `GOFLAGS` / `GONOSUMDB` / `GOPRIVATE` / `GOSUMDB` / `GOPROXY` / `GONOSUMCHECK`: **none found** anywhere. Module verification runs against the public default with no repo-specific override. The 5 `github.com/kyverno/*` org dependencies are pulled as fully public, sum-verified pseudo-versioned modules like any third-party dependency — no special-cased trust.
- One `replace` directive: `k8s.io/pod-security-admission => github.com/kyverno/pod-security-admission v0.0.0-...` (a fork), confirmed the only replace in go.mod.

## 7. Auto-merge predicate (proposed pseudocode)

```
function should_auto_merge(pr):
    if pr.author != "dependabot[bot]":
        return False, "not a dependency-bot PR"

    bump = parse_dependabot_metadata(pr)
    ecosystem = bump.ecosystem               # gomod | github-actions

    if bump.update_type == "semver-major":
        return False, "major bump always requires human review"

    if ecosystem == "github-actions" and bump.update_type in {"semver-minor", "semver-patch"}:
        risk_tier = "low"       # CI-only blast radius, not shipped to users
    elif ecosystem == "gomod" and bump.update_type == "semver-patch":
        risk_tier = "low"
    elif ecosystem == "gomod" and bump.update_type == "semver-minor":
        risk_tier = "medium"    # requires green checks + no new advisory
    else:
        return False, "unclassified bump shape, default to human review"

    if pr.has_label("hold") or pr.has_label("do-not-merge") or pr.has_label("wip"):
        return False, "explicit maintainer hold"

    required_checks = [
        "golangci-lint", "unit-tests", "check-sha-pinned-actions",
        # NOTE: none of Trivy/Semgrep/Scorecard/SonarCloud/FOSSA run on pull_request today —
        # true pre-merge vuln scanning would require ADDING a PR-triggered
        # dependency-review-action; until that exists, "green checks" cannot include a vuln gate.
    ]
    if not all(check_passed(pr, name) for name in required_checks):
        return False, "required checks not green"

    if pr.branch_is_stale(base="main"):
        return False, "needs rebase/merge from main first"

    if risk_tier == "medium" and not vulnerability_scan_clean(bump):
        return False, "no PR-time vuln signal available yet — treat as block until dependency-review-action exists"

    return True, f"auto-mergeable: {ecosystem} {bump.update_type} bump, risk_tier={risk_tier}"
```

**Explicit conflict flag for maintainers:** DEPENDENCY-POLICY.md currently states *"All proposed updates are reviewed by maintainers and must pass CI before merging."* A predicate like the one above — which auto-merges patch-level Actions/Go bumps with zero human review — would **contradict the literal text of the current written policy**, even though it doesn't contradict the *spirit*. This should be raised as an open question for maintainers: does "reviewed by maintainers" mean a human must look at every PR, or would maintainers accept an automated policy-as-code gate as satisfying "review" for a defined low-risk subset? Probably needs an accompanying edit to DEPENDENCY-POLICY.md if adopted.

## 8. Docker / Helm dependency gaps

- **No Dockerfiles exist in the repo at all.** Kyverno builds container images with **`ko`** (`.ko.yaml` at repo root). `.ko.yaml` sets `defaultBaseImage: ghcr.io/wolfi-dev/static:alpine` — a **floating tag, not a digest** — shared across all 7 `ko` build targets.
- Because there's no Dockerfile, even if Dependabot's `docker` ecosystem were added, it would not see or bump this base image (Dependabot's docker ecosystem parses `Dockerfile`/`docker-compose.yml`, not `.ko.yaml`). Currently there appears to be **no automation of any kind** tracking updates to this base image.
- **Helm chart dependencies are also uncovered.** `charts/kyverno/Chart.yaml` declares 3 genuinely external chart dependencies: `kyverno-api` (0.0.1-alpha.2), `openreports` (0.1.0), `reports-server` (0.1.6). None of these pinned chart versions are touched by Dependabot.
- **Confirmed: `dependabot.yml` has exactly 2 ecosystems (gomod, github-actions) and nothing else** — docker and Helm chart dependencies are both fully unmanaged by any automated update mechanism today.

## Cross-cutting note (docs-drift pattern)

DEPENDENCY-POLICY.md's own workflow filenames are stale (says `trivy.yaml`/`trivy-periodic-scan.yaml`/`lint.yaml`/`scorecard.yaml`; actual files are `scan-trivy.yaml`/`periodic-trivy.yaml`/`check-golangci-lint.yaml`/`scan-scorecard.yaml`), same pattern as 3 dead CODEOWNERS path rules found in the structure-cluster research. This is a repo-wide pattern, not a one-off.

## 9. Open questions for maintainers (Dependency PR Handling)

Raised rather than assumed, because #1 is a genuine conflict with a written policy, not a judgment call I should make unilaterally.

1. **`DEPENDENCY-POLICY.md` says every update is "reviewed by maintainers and must pass CI before merging"** — with no low-risk carve-out. Auto-merging patch bumps contradicts the literal sentence even if it fits the intent. Does "reviewed" mean a human click, or would a policy-as-code gate satisfy it for a defined subset? If the latter, adopting this should come with a PR editing that doc — not a quiet divergence from it.
2. Start auto-merge on `github-actions` only (lower blast radius, CI-only impact) before touching `gomod`?
3. What CVE severity should hard-block a merge versus just flag it for review?
4. `.ko.yaml`'s base image (`ghcr.io/wolfi-dev/static:alpine`) is on a floating tag today, not a digest (see §8) — is pinning to digest wanted, or is that deliberate?
5. Who may apply the kill-switch `hold` label — any maintainer, or a named role from `OWNERS.md`?
