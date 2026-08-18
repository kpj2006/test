# Kyverno AI Maintainer Assistant

---

## 0. The one thing to lead your proposal with

**Conformance tests do not run on pull requests.**

- `.github/workflows/check-tests.yaml` triggers on `push:` to `main` / `release-*` only. No `pull_request` trigger.
- On PRs, conformance is **opt-in** via a `/conformance` comment (`comment-conformance.yaml`).
- Why: `tests-conformance.yaml` has **54 jobs**, most with a 3-way k8s matrix (`v1.33.7`, `v1.34.3`, `v1.35.1`), several sharded 3–12 ways. Plus `tests-conformance-policy-library.yaml` (shard-count 12) and `tests-k6.yaml`. That's 150+ runners per full run. Across **305 open PRs**, running this per-PR is financially impossible.

The consequence is the actual toil:

- Regressions land on `main`, then get discovered by the post-merge run.
- `.github/actions/workflow/failure-issue` auto-files an issue on failure and auto-closes it on the next green run. There are **95 issues** with the `workflow-failure` label, overwhelmingly titled *"Workflow failed: Tests (refs/heads/main)"* (e.g. #17135 open, #17107, #17099, #17098, #17081 …).
- Maintainers then bisect after the fact, on `main`, under release pressure.

So the framing is not "let's automate chores." It's:

> Kyverno's CI is structurally *post-merge*, because *pre-merge* is too expensive. Scoped test selection is what makes pre-merge affordable. Everything else in this project is downstream of that.

This reframes "scoped test selection" from Phase 2 nice-to-have into **the keystone deliverable**, and gives you a hard, measurable success metric (runner-minutes per PR; percentage of `main` breakages caught pre-merge; count of `workflow-failure` issues per month). LFX final reports live or die on having a number.

---

## 1. Ground-truth audit — what already exists

This is the section that will most differentiate your application. Half the issue's proposed scope is already shipped. Proposing to build it again signals you didn't read the repo.

| Proposal item | Status in repo today | The actual remaining gap |
|---|---|---|
| Dependency PR handling | `.github/dependabot.yml` — daily, gomod + github-actions, grouped (`kubernetes`, `sigstore`, `otel`), `rebase-strategy: disabled`. Only **1** open Dependabot PR right now. | No auto-merge workflow. `rebase-strategy: disabled` means stale bot PRs rely on `pr-branch-updater`. Grouping already kills most of the volume — **the pain here is smaller than the issue implies.** Verify with maintainers before spending phase time on it. |
| PR hygiene / rebase | **`pr-branch-updater.yml` exists** — calls reusable workflow `kyverno/.github/.github/workflows/pr-branch-updater.yml`, using a GitHub App token specifically so it can update PRs touching workflow files. | No stale-PR nudging (`actions/stale` absent). No automatic CI re-trigger. No thundering-herd protection across 305 PRs. |
| Merge-conflict labelling | **Exists.** `pr-labelling.yaml` → `eps1lon/actions-label-merge-conflict`, `dirtyLabel: merge-conflicts`. | Nothing. This is done. *(Answers your "label-merge-conflict → ?")* |
| Path→label mapping | **Exists.** `.github/labels.yml` (526 lines) holds color + description + `rules:` globs; `pr-labelling.yaml` renders it through `yq`/`jq` into an `actions/labeler` config at runtime. | It maps paths → **labels**, not paths → **test suites**. Extending this single file into the path→suite map is the clean move (one source of truth, existing render pipeline). |
| Scoped test selection | The primitive is **already parameterised**: `.github/actions/tests/conformance/run/action.yaml` accepts `tests-path`, `chainsaw-tests` (regex), `shard-index`, `shard-count`, `quarantined-tests`, `kyverno-configs`. | The mapping layer and the dynamic matrix generator. The runner is ready; nobody has written the "which suites does this diff need" brain. **This is your highest-leverage deliverable.** |
| Codegen/verify gate | **Exists.** `check-codegen.yaml` runs `make codegen-all-code` → `make verify-codegen`, and on failure writes `codegen-code.patch` and uploads it as an artifact with instructions to `git apply` it. | The patch is uploaded for the *human* to download and apply. An agent should commit it straight to the PR branch. Same for `codegen-docs.patch`. High-frequency, zero-judgment, fully reversible — ideal first agent action. |
| Auto-backport | **Exists.** `/cherry-pick <branch>` comment (`comment-cherry-pick.yaml`) + `cherry-pick-on-merge.yaml` (replays queued `/cherry-pick` comments once merged) + `.github/actions/git/cherry-pick`. | Label-driven backports, and conflict handling — currently a failed cherry-pick just posts *"check workflow logs"*. *(Answers your "auto backport labeled PRs → ?")* |
| First-time contributor welcome | **Exists.** `.github/config.yml` configures behaviorbot (`newIssueWelcomeComment`, `newPRWelcomeComment` which already reminds about sign-off, `firstPRMergeComment`). | Nothing meaningful. Strike from scope. |
| Auto-suggest reviewers | `CODEOWNERS` exists — but only **25 lines**, hand-maintained, area-level. | Paths not covered by CODEOWNERS fall through to `@kyverno/kyverno-core-maintainers` (everyone). Git-history-based suggestion for uncovered paths is a real gap. |
| Flaky test detection | **The quarantine mechanism already exists.** `quarantined-tests` input takes comma-separated dir names and does `yq eval '.spec.skip = true'` on the matching `chainsaw-test.yaml`. Currently hardcoded in `tests-conformance.yaml` for `applyconfiguration` (3 places) and `sync-modify-downstream` (2 places). Four more tests carry a committed `skip: true`. | **Detection, lifecycle, and expiry.** Nothing decides *what* to quarantine, nothing tracks *when it goes back in*, nobody owns it. This is a fully-formed gap with an existing hook to plug into. *(Answers your "flag/test detection → ? (needs example)")* |
| Security scanning | `scan-trivy.yaml` (rootfs, on `push` to main/release only), `scan-semgrep.yaml` (daily cron), `scan-scorecard.yaml` (weekly), `sync-trivy-issues.yaml`, `.semgrepignore`, `.trivyignore.rego`, `SECURITY-INSIGHTS.yml`, `VULN-TEMPLATE.md`. | **No PR-time gate.** Everything is post-merge or scheduled. A vulnerable dep can merge and only surface on the next push scan. No secret scanning (no gitleaks). |
| Release notes | PR template mandates an "Explanation" section explicitly for release-note drafting; `semantic.yml` enforces conventional PR titles (`feat/fix/revert/docs/style/refactor/test/build/autogen/security/ci/chore`); `assign-milestone.yaml` assigns milestones from git history at tag time. | No `release-drafter`. All the structured input exists; nothing consumes it. Low-risk, high-visibility win. |
| Issue triage | `.github/ISSUE_TEMPLATE/` has `bug-cli.yaml`, `bug-webhook.yaml`, `bug-other.yaml`, `feature.yaml`, `VULN-TEMPLATE.md` — templates already pre-classify. | No content-based labelling, no missing-info detection, no repro. **282 open issues.** |
| Slash commands | `comment-prow.yaml` via `jpmcb/prow-github-actions`, enabling only `/assign`, `/unassign`, `/lgtm`, `/milestone`. | `/retest`, `/hold`, `/close`, `/area`, `/kind` are supported by that action but not enabled. Cheap win. |
| Other existing infra | `code-freeze.yaml` (FROZEN_BRANCHES repo variable + required check), `pr-rate-limiter.yaml` (Homebrew action, 8-PR cap per non-member), `periodic-cleanup.yaml` (stale branch reaper via `cbrgm/cleanup-stale-branches-action`), `cache.yaml` (warms gomod/gobuild caches on main). | Note `pr-rate-limiter` — the maintainers have *already* signalled they care about contributor-volume control. Frame your automation as extending that philosophy. |

**Repo scale for your proposal's numbers section:** 305 open PRs · 282 open issues · 1,032 `chainsaw-test.yaml` files across 52 top-level suites · 54 conformance jobs · 526-line label map · 25-line CODEOWNERS.

---

## 2. Every question mark in your notes, answered

### Sheet 1

**`digest → diff tool for report → CI flakiness report`**
Build the digest on top of what already emits data: the `failure-issue` action opens/closes an issue per workflow failure on `main`, so the **open→close interval** of `workflow-failure` issues is a free flakiness time-series. A short-lived issue (opens, closes on the very next run, no commit in between fixing it) is almost certainly a flake; a long-lived one is a real regression. That distinction requires no new infrastructure — just the Issues API and the commit log. The "diff tool" is week-over-week delta on that series, posted as a single pinned digest issue that gets **edited in place**, never re-posted.

**`Cache-reaper`**
`cache.yaml` warms `gomod` and `gobuild` caches on every push to `main`, with prefixed keys via `.github/actions/go/{lookup,save,restore}-cache`. GitHub's 10 GB per-repo cache limit evicts LRU, so a repo this size will thrash. A reaper deleting caches for merged/closed PR refs (`gh cache delete` / `DELETE /repos/{o}/{r}/actions/caches`) is a genuine, tiny, uncontroversial win. Good "first PR to prove yourself" material before the mentorship even starts.

**`Secret scanner → OSV-scanner → gitleaks / semgrep`**
These are three different jobs; don't stack them blindly:
- *Secrets*: **gitleaks** — genuinely absent from Kyverno as a standalone job, but CodeRabbit (adoption already confirmed with maintainers) ships gitleaks and runs Pro features free on public repos — this is covered by the CodeRabbit rollout, no separate workflow needed.
- *SAST*: **semgrep** — already there, daily.
- *Dependency CVEs*: **Trivy** — already there. Adding **OSV-Scanner** mostly duplicates Trivy on Go modules; <cite index="24-3">both walk manifests without configuration</cite>, and <cite index="23-1">OSV-Scanner's differentiator is guided remediation suggesting minimum version bumps that fix everything at once</cite>. Propose it only if you can show Trivy is missing Go advisories in practice. Otherwise it reads as tool-stacking.

**`dependency review → ? (in container) → SDK/API`**
Three distinct things, and the confusion here is worth resolving before you write the proposal:
1. **PR-time gate** — `actions/dependency-review-action` diffs the dependency graph between base and head and fails on newly-introduced vulnerable or badly-licensed deps. This is the real gap: <cite index="20-1">the standard pattern is detect with Trivy/OSV, remediate with Dependabot, and gate at PR time with dependency-review so new vulnerable dependencies never get in</cite>. Kyverno has the first two and not the third.
2. **"In container"** — the action reads the GitHub dependency graph, not images. Container scanning is a separate post-build step; Kyverno's Trivy is `scan-type: rootfs` on built binaries, not image layers. Distinct concern, distinct workflow.
3. **SDK/API** — for an agent, don't shell out to the action. Call `GET /repos/{owner}/{repo}/dependency-graph/compare/{basehead}` directly. You get structured JSON (package, versions, vulnerabilities with severity, license, source repo) that an agent can reason over, rather than parsing action output. This is the right answer for an agentic system and worth saying explicitly in the proposal — it shows you distinguish "run a workflow" from "give the agent a capability."

**`label-merge-conflict → ?`** — Already implemented. See table.

---

### Sheet 2

**`json/YAML → manifest → ?`**
Concrete proposal: `.github/agent-manifest.json`, **generated** from the Makefile, not hand-written, and verified in CI. Kyverno's whole culture is `make codegen-* && make verify-codegen` — a hand-maintained manifest will drift within a month and the maintainers know it. So:

```
make codegen-agent-manifest   # emits .github/agent-manifest.json
make verify-codegen            # already fails on any git diff — free drift protection
```

Shape it around what an agent actually needs to decide:

```json
{
  "schema": 1,
  "targets": {
    "test-unit":  { "cmd": "make test-unit",  "needs_cluster": false, "approx_minutes": 12 },
    "test-cli":   { "cmd": "make test-cli",   "needs_cluster": false },
    "codegen":    { "cmd": "make codegen-all-code", "verify": "make verify-codegen", "autofixable": true }
  },
  "generated_paths": ["**/zz_generated.*", "pkg/client/**", "config/crds/**"],
  "never_autonomous": ["api/kyverno/v1/**", "pkg/cosign/**", "pkg/notary/**", ".github/workflows/**"],
  "suites": { "...": "path → chainsaw suite map, see below" }
}
```

The `approx_minutes` and `needs_cluster` fields are what let a scheduling agent make cost decisions. Nobody else will propose that.

**`agents.md → delegation per directory → make it compact → ask for mattpocock SKILLS`**
Read the existing `AGENTS.md` first — it's already ~200 lines and unusually good (full make-target tables, import-alias rules, logging levels, API versioning rules, a DCO+codegen failure-prevention checklist). Your contribution is *not* writing it; it's:
- **Splitting it.** <cite index="2-1">Codex concatenates AGENTS.md root-down and caps total at 32 KiB by default, while Factory walks nearest-first with the closest file winning on conflict</cite> — so per-directory files genuinely change agent behaviour, and shrinking the root file is a real optimisation. Targets: `pkg/engine/`, `pkg/webhooks/`, `pkg/controllers/`, `pkg/cel/`, `api/`, `test/conformance/`.
- **Making it verifiable.** Add a CI check that every make target referenced in `AGENTS.md` actually exists in the `Makefile`. Docs that can't lie.
- On skills: <cite index="8-1">CodeRabbit already auto-detects `**/AGENTS.md`, `**/CLAUDE.md`, `.cursorrules`, `.github/copilot-instructions.md` and similar as coding-guideline sources by default</cite>, so a well-split `AGENTS.md` is simultaneously your CodeRabbit config. Say that in the proposal — it's a two-for-one.

**`for making files agent readable → comments on top of file (common in .py) → .txt`**
Header comments don't survive refactors and nobody updates them. Prefer machine-checkable signals: Go build tags, a `//go:generate` marker, and `DO NOT EDIT` headers (already present on `zz_generated.*`). For human/agent narrative context, per-directory `AGENTS.md` beats per-file comments — one file to update, and every major agent runtime already reads it.

**`safe automation boundary → write in gist → ?`**
Do **not** use a gist. A gist is unversioned, unreviewable, and outside branch protection — for a security project that's a non-starter. Two better options, and the second is your differentiator:

1. `.github/ai-maintainer.yaml` in-repo, protected by CODEOWNERS, so changing the agent's authority requires a maintainer-reviewed PR.
2. **Express the boundary as Kyverno policy and enforce it with the Kyverno CLI.** Kyverno is a policy engine. An AI maintainer for Kyverno whose own permissions are governed by Kyverno policy — validated in CI with `kubectl-kyverno` against the agent's proposed action rendered as a resource — is a story no other applicant will tell. It also gives maintainers something they intuitively trust: the same admission-control mental model they already built.

**`CI/test metadata → ?`**
Extend `.github/labels.yml` rather than adding a parallel file. It already carries `rules:` globs per area, and `pr-labelling.yaml` already renders it through `yq`/`jq`. Add a sibling key:

```yaml
area/engine:
  color: 0052cc
  description: Policy engine core
  rules:
    - changed-files:
        - any-glob-to-any-file: [pkg/engine/**]
  suites: [assert, autogen, mutate, validate]   # ← new
  unit: ["./pkg/engine/..."]                     # ← new
```

One source of truth, one render step, and the existing labeller keeps working untouched. The alternative — `test/conformance/chainsaw/*/OWNERS` files, as the issue suggests — inverts the direction (suite→path instead of path→suite) and needs a full walk of 52 directories to answer one query.

Ship a **coverage report** alongside it: for each of the 52 suites, which paths map to it, and which paths map to nothing. The unmapped set is where you fall back to the full suite, and shrinking it over time is a clean progress metric.

**`PR metadata → why not make this optional, use CodeRabbit`**
Fine, but note the repo already has `semantic.yml` enforcing conventional PR titles and a template with `/kind` labels. The metadata exists; nothing consumes it. Consuming it is more valuable than generating more of it.

**`Auto-draft release notes → CodeRabbit → ? / release-drafter.yml`**
Use `release-drafter` and configure its categories directly from `semantic.yml`'s type list — the mapping is already decided for you. The PR template's "Explanation" field is explicitly written for release-note drafting, so the human input is already being collected and thrown away. This is probably the single easiest visible win in the whole project; consider landing it as a pre-mentorship PR.

**`flag/test detection → ? (needs example)`**
Here's your example, with real artifacts:

- Mechanism already present: `quarantined-tests: applyconfiguration` at `tests-conformance.yaml:291,308,324` and `quarantined-tests: sync-modify-downstream` at `:358,539`. Someone hit a flake, hand-edited the workflow, and moved on. Nothing records *when* or *why*, and nothing will ever take them out.
- Also committed permanently: `spec.skip: true` in `verify-images/clusterpolicy/standard/update-multi-containers`, and three under `generating-policies/clone/sync/` (`sync-delete-policy`, `sync-delete-one-source`, `sync-delete-one-trigger`).
- Historical evidence: #16757 *"[Bug] Flaky CI: Framework tests (ivpol) failing in PR checks"*, #14347 *"[Bug] Investigate flaky tests root cause"*, #14199/#14200 (*"remove flaky test temporary to unblock PRs"* and its revert — the manual version of quarantine/de-quarantine), plus older #8334, #7460, #3756, #2406.
- Live signal: 95 `workflow-failure` issues, most opening and closing within a run or two.

So the deliverable is not "build quarantine" — it's **close the loop**:

1. Ingest per-test results from chainsaw runs on `main` (JUnit/JSON output) into a flat file in a `ci-metrics` branch or GitHub Pages. Zero infra cost, fully auditable, no external DB.
2. Compute flip-rate (pass↔fail transitions on unchanged code). <cite index="31-1">"Flip flakiness" with an optimal distance parameter is the method used on TestGrid data and performs at least on par with TestGrid's own implementations</cite> — cite this, it's directly applicable and CNCF-adjacent.
3. Above threshold → open a quarantine PR editing the `quarantined-tests` input, **with an owner and an expiry date** in the PR body.
4. Nightly, re-run quarantined tests in a separate non-blocking lane. Clean for N days → auto-open the de-quarantine PR. <cite index="33-1">This is exactly Atlassian's Flakinator loop: collect signals for quarantined tests via scheduled runs, compute test health, and reintroduce automatically once healthy for a configured period</cite>.
5. Enforce expiry. <cite index="32-1">The documented failure mode is assigning quarantine to "the team" rather than a named owner with a deadline, at which point quarantine becomes permanent</cite> — those four committed `skip: true` tests are that failure mode already present in the repo.

Precedent to cite: <cite index="27-1">Kubernetes SIG-Testing operates a zero-flake policy — test jobs must not auto-retry on failure — and unfixable tests get tagged `[Flaky]` so they're quarantined to jobs that explicitly run flakes</cite>. Kyverno is CNCF; aligning with SIG-Testing convention is an easy sell.

**`Auto backport labeled PRs → ?`** — Already exists comment-driven. Gap is label/milestone-driven + conflict handling.

**`DCO -s → pr-target → anything else?`**
DCO is enforced by the GitHub DCO App (repo-level, not a workflow — nothing in `.github/` handles it), and `.github/config.yml`'s `newPRWelcomeComment` already reminds first-timers to sign off. `AGENTS.md` even documents the recovery command (`git rebase --signoff <base>`). So the only gap is **reactive guidance**: on DCO failure, comment with the exact `git rebase --signoff` invocation for that PR's base. Careful with `pull_request_target` here — see the security section.

**`First-time contributor → GitHub API tag`** — behaviorbot already does this via `.github/config.yml`. Done.

**`Security template → use github private advisory`** — `VULN-TEMPLATE.md` + `SECURITY.md` + `SECURITY-INSIGHTS.yml` exist. Note carefully: **never let an LLM agent read private security advisories.** That's an exfiltration path. Confine any advisory work to public CVE cross-referencing and say so explicitly — maintainers of a security product will be checking whether you thought about this.

**`Auto suggest reviewers → CODEOWNERS → made it like openrouting toolkit`**
CODEOWNERS covers 24 paths. Everything else falls through to `*  @kyverno/kyverno-core-maintainers`. A `git log --numstat`-based suggester over the last N months of history, restricted to *paths CODEOWNERS doesn't cover*, is a genuine gap and is fully deterministic — no LLM needed, which makes it trustworthy. Bonus: emit a report of "paths with a clear de-facto owner not in CODEOWNERS" as a maintainer digest item.

---

### Sheet 3

**`retrigger CI → ?`**
Three mechanisms, pick deliberately:
- `POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun-failed-jobs` — reruns only failed jobs. Cheapest, right default.
- `POST .../actions/runs/{run_id}/rerun` — full rerun. Only for infra-level failures.
- Comment command — `jpmcb/prow-github-actions` is already wired in `comment-prow.yaml` and supports `/retest`; it's simply not in the enabled `prow-commands` list. Enabling it is a two-line PR.

**Non-obvious but important:** blind auto-retry is *dangerous here*, because Kyverno currently has no per-test flake data. <cite index="27-1">Kubernetes explicitly banned automatic retry (`ginkgo.flakeAttempts=2` removed in 2019, reconfirmed as policy in 2023)</cite> precisely because retries hide flakes. Correct design: retry **once**, and record the retry as a flake signal feeding the detector. Retry-with-telemetry, never retry-and-forget. Saying this in your proposal demonstrates you understand the tradeoff rather than reaching for the easy button.

**`Attempt automated reproduction → ?`**
The environment already exists — `make kind-create-cluster`, `make kind-deploy-kyverno`, `make kind-deploy-all`, `.github/actions/kyverno/wait-ready`, `.github/actions/kyverno/logs`, the `kindest/node` matrix, and `.github/actions/tests/conformance/run` which does the whole cluster-up-deploy-test-collect-logs cycle. You're not building a harness; you're building an **adapter** from an issue body to that harness.

The design that makes this a standout deliverable:

> **The repro artifact should be a chainsaw test, not a log.**

Pipeline: bug issue → extract policy YAML + resource YAML + expected outcome from the (already-structured) issue template → emit a `chainsaw-test.yaml` in the right suite directory → run it on the KinD matrix → post actual vs. expected. If it reproduces, you now have a **regression test ready to commit**, not just a confirmation comment. Bug reports become test coverage automatically. That directly attacks the 282-issue backlog *and* strengthens the suite that the rest of the project depends on.

Hard constraints to state up front (maintainers will ask): untrusted YAML from strangers executes in a disposable KinD cluster on an ephemeral runner, no secrets in the job environment, no registry push credentials, hard timeout, network egress restricted. `pull_request_target` must never be combined with executing issue-supplied content.

**`Slack / Discussion Q&A → need to check about when confidence is low`**
Don't build a confidence score from model self-report; it's unreliable and maintainers will not trust it. Use structural gates instead:
- **Answer only with citations.** Every answer must cite a file path + line range in `kyverno/kyverno`, `kyverno/website`, or `docs/dev/`. No supporting citation → no answer.
- **Escalate on retrieval failure, not on model doubt.** If top-k retrieval similarity is below threshold, or sources disagree, post nothing and apply a `needs-maintainer` label.
- **Silence is the default.** For a first deployment, propose **draft answers to a maintainer-only channel** rather than public auto-reply. Ask for public posting rights only after a measured precision rate. Maintainers are far more likely to accept a bot that asks permission than one that speaks first.
- Phase it last. It's the highest-risk, lowest-reversibility component — a wrong public answer on a security product's Slack is a real cost, unlike a wrong label.

**`Dependabot PR / Renovate PR / Socket bot`**
Kyverno is on Dependabot with grouping and only 1 open bot PR — grouping already solved the volume problem. A Renovate migration is a large, disruptive change with modest payoff here. If you propose it, propose it as an *evaluation deliverable* (write the ADR, don't force the migration). `DEPENDENCY-POLICY.md` exists at the repo root — read it before you write anything about dependency handling; it likely constrains what auto-merge is even permissible.

**`Issue enrichment keys of CodeRabbit`**
CodeRabbit's config schema does expose automatic issue enrichment — <cite index="12-1">the configuration reference documents settings for automatic issue enrichment that analyse and enrich issues with related code, potential solutions and a complexity assessment, defaulting to false</cite>. With CodeRabbit adoption confirmed, enable this toggle directly — it's a config flip, not a custom Triage Agent to build.

**`missing info → Danger.yaml`**
Danger is JS-based and would add a Node toolchain to a Go repo — friction, and unnecessary now that CodeRabbit is confirmed. Kyverno's issue templates are already `.yaml` forms with required fields, so missing-info detection is covered by CodeRabbit directly. Don't add a Node ecosystem for something already covered.

**`We could also give skills to CodeRabbit for expected behaviours`**
Yes — with adoption confirmed, this is where to put the configuration effort. The mechanism matters:
- `.coderabbit.yaml` `path_instructions` gives per-glob review instructions — <cite index="14-1">e.g. distinct instruction blocks for `src/api/**` vs `src/db/**` vs `**/*.test.*`</cite>. Map these onto Kyverno's real risk surface: `api/kyverno/v1/**` (versioning rules from `AGENTS.md`), `pkg/cosign/**` + `pkg/notary/**` (supply chain), `pkg/webhooks/**` (admission path), `**/zz_generated.*` (should be filtered out entirely).
- <cite index="8-1">`AGENTS.md` and `CLAUDE.md` are auto-detected as guideline sources</cite>, so splitting `AGENTS.md` per-directory improves CodeRabbit and your agents simultaneously.
- <cite index="17-1">CodeRabbit Pro features are free on public repositories with no activation step or time limit</cite> — relevant to a CNCF project's zero-budget constraint, and worth stating.
- CodeRabbit "Skills" are a different product: <cite index="9-1">SKILL.md files installed into a local agent that trigger CodeRabbit CLI reviews from natural language</cite>. Those run in a developer's local agent, not in CI — don't conflate the two in your proposal.

---

## 3. Multi-agent architecture

You asked about running multiple agents. The design principle that matters most:

> **Deterministic where possible; LLM only where judgment is genuinely required.**

Maintainers of a *security* project will not accept an LLM in the merge path if a `yq` expression would do. Diff→suite mapping, CODEOWNERS suggestion, flake statistics, and codegen patching are all deterministic. Reserve the model for triage classification, log root-cause summarisation, repro extraction from prose, and doc-gap detection.

### Topology

State lives in GitHub — labels, check runs, issue bodies, a `ci-metrics` branch. **No external database.** Zero hosting cost, fully auditable, and every agent decision is already a reviewable artifact. This also sidesteps the "who pays for the infra after the mentorship ends" question, which kills a lot of otherwise-good CNCF tooling projects.

```
GitHub webhook / schedule
        │
        ▼
   Router  ──── reads .github/ai-maintainer.yaml (kill-switch, per-capability enable)
        │
        ├─ Scope Agent      (deterministic)  diff → suites + unit packages → matrix JSON
        ├─ Flake Agent      (deterministic)  results → flip-rate → quarantine/de-quarantine PR
        ├─ Codegen Agent    (deterministic)  run codegen → commit patch to PR branch
        ├─ Dependency Agent (deterministic)  dep-graph compare → gate / auto-merge / escalate
        ├─ Triage Agent     (LLM)            issue → labels + missing-info + duplicate link
        ├─ Repro Agent      (LLM + sandbox)  issue → chainsaw test → KinD run → result
        ├─ Docs Agent       (LLM)            diff → does kyverno/website need a PR?
        └─ Answer Agent     (LLM + RAG)      question → cited answer or escalation
        │
        ▼
   Ledger  ──── one check run per action: input, decision, rationale, revert command
```

Each agent gets: its own least-privilege GitHub App installation token, its own rate limit, its own kill-switch key, and a required `revert:` field in its ledger entry. If an agent can't describe how to undo what it did, it isn't allowed to do it.

### Trust ladder

Stage each capability rather than shipping all at once — and put the ladder in the proposal, because it's what turns "autonomous agent" from a scary phrase into a governed process:

| Level | Agent may | Example |
|---|---|---|
| 0 | Observe, write to a private log | everything, week 1 |
| 1 | Comment / label (revert = delete) | triage, digest |
| 2 | Open draft PRs (revert = close) | quarantine, codegen, de-quarantine |
| 3 | Push to *contributor* PR branches | codegen fixes, rebase |
| 4 | Merge, under strict predicate | Dependabot patch-only, green CI, no `hold` label |

Nothing reaches level 4 without measured precision at level 3.

---

## 4. Additional capabilities worth proposing

Ranked by *(differentiation × plausibility of landing)*.

**1. PR-time scoped conformance.** The keystone from §0. Metric: median runner-minutes per PR, and `workflow-failure` issues per month before/after.

**2. Repro-to-regression-test pipeline.** §Sheet 3. Turns issue triage into test-coverage growth.

**3. Kyverno-governs-Kyverno.** Express the safe-automation boundary as Kyverno policy, validate agent actions with `kubectl-kyverno` in CI. Perfect thematic fit, and dogfooding is a value CNCF projects reward.

**4. API compatibility linter.** `AGENTS.md` documents hard API rules — *no new resource types in `kyverno.io/v1`; attributes cannot be deleted or modified within a version; deprecate then remove after 3 minor releases; newer versions may reference older stable types but not vice versa*. Today only human review enforces these. A deterministic checker over `api/**` diffs enforces written policy, catches the highest-cost class of mistake, and needs no LLM. Low risk, high maintainer value, and directly relevant to `area/api` + `area/crds`.

**5. Quarantine expiry enforcement.** Every quarantine gets an owner and a date. Expired entries escalate to the maintainer digest. This is the mechanism that stops your own tool from becoming the next four permanently-skipped tests.

**6. Cost telemetry as a first-class output.** Track runner-minutes per workflow per week and publish the delta. Gives your final report a number, gives maintainers a budget argument, and makes every later optimisation self-justifying.

**7. Attested action ledger.** `pkg/cosign` and `pkg/notary` are already in-tree and `images-publish.yaml` handles signing. Signing the agent's action ledger and attaching it as an attestation is thematically perfect for a supply-chain-security project and technically cheap given the existing tooling.

**8. "Why did CI fail" explainer for contributors.** On a failed check, post the minimal actionable fix. For codegen, that's already computed — `check-codegen.yaml` *generates the exact patch* and uploads it as an artifact for the contributor to find, download, and apply. Just apply it. High frequency, zero judgment, instantly reversible.

**9. Website PR link enforcement.** The PR template requires a `kyverno/website` doc PR link for behaviour changes; nothing verifies it. Deterministic check: behaviour-changing paths touched + no website link + no `docs-not-required` label → nudge.

**10. Nightly soak lane for flake discovery.** Run the N most-suspect tests 20× nightly on a spare matrix. Converts quarantine decisions from anecdote to statistics, and `litmuschaos/pod_cpu_hog` exists in-tree as a starting point for resource-contention-induced flakes — a known cause of exactly the timing flakes in #3756 and #7460.

**11. Cache reaper.** §Sheet 1. Small, safe, ship it before the mentorship starts.

---

## 5. Risks maintainers will raise — address these before they do

**Prompt injection is a supply-chain vulnerability here, not a hypothetical.** Every issue body, PR description, and code comment is attacker-controlled text. An agent with merge rights that reads a PR description is a remote-code-execution path into a security product used for Kubernetes admission control. Mandatory posture: untrusted content never enters a context that holds a write-capable token; the deciding agent and the reading agent are separate processes; all model output is validated against a schema before it becomes an API call.

**`pull_request_target` is the classic escalation.** `pr-labelling.yaml`, `pr-rate-limiter.yaml`, and `code-freeze.yaml` already use it — correctly, since none check out fork code. Any agent work must preserve that invariant absolutely: never `actions/checkout` a fork ref inside a `pull_request_target` job.

**Thundering herd.** 305 open PRs. Auto-rebase, auto-retrigger, and auto-update interact badly: one push to `main` can trigger hundreds of CI runs. `pr-branch-updater.yml` already uses `concurrency: pr-branch-updater` with `cancel-in-progress: false` — respect and extend that. Propose an explicit global budget.

**Auto-merge on a security product.** Even patch-level Go deps land in an admission controller in the cluster's critical path. Expect maintainers to be conservative; `DEPENDENCY-POLICY.md` may already constrain it. Propose auto-merge as the *last* capability, gated on the trust ladder, not the first.

**Quarantine becoming permanent.** Already happening. Your expiry mechanism is the answer — lead with it rather than treating it as an afterthought.

**Maintainer trust is the real constraint, not capability.** A bot that is right 90% of the time and wrong loudly gets disabled in a week. Bias every default toward silence, drafts, and reversibility.

---

## 6. Suggested phase restructure

The issue's phases are ordered by implementation convenience. Reorder by value and risk:

- **Phase 0 — Audit & metadata (weeks 1–3).** Configure CodeRabbit (`.coderabbit.yaml` `path_instructions` per the risk-surface mapping in §Sheet 3 — adoption is already confirmed with maintainers, so this is configuration, not evaluation). Extend `labels.yml` with `suites:`/`unit:` and publish the path→suite coverage report. Generated `agent-manifest.json` + `verify-codegen` hook. Split `AGENTS.md` per-directory + the make-target consistency check. Baseline metrics: runner-minutes, `workflow-failure` rate, PR age distribution.
- **Phase 1 — Deterministic wins (weeks 3–6).** Scoped test selection → PR-time conformance. Codegen auto-fix. API compatibility linter. `dependency-review-action` gate. release-drafter. Enable `/retest`. Cache reaper. *All deterministic; none can hallucinate; each independently landable.*
- **Phase 2 — Flake lifecycle (weeks 6–9).** Result ingestion, flip-rate detection, quarantine + de-quarantine PRs with expiry, nightly soak lane, maintainer digest.
- **Phase 3 — Judgment agents (weeks 9–12).** Triage classification, missing-info detection, repro→chainsaw pipeline. Trust ladder level 1→2 only.
- **Phase 4 — Stretch.** Q&A assistant, draft-to-maintainers only. Auto-merge, if and only if earlier precision justifies it.

Note what moved: **scoped test selection went from Phase 2 to Phase 1**, and **auto-merge went from Phase 1 to last**. Both changes are defensible on evidence from the repo, and explaining *why you reordered the mentor's own phases* — respectfully, with file paths — is exactly the signal that gets an applicant selected.

---

## 7. Questions to put to the maintainers

Ask these in the issue thread or Slack before submitting. Asking good questions is itself a selection signal, and Jim Bugwadia is listed as the mentor.

1. Was keeping conformance off PRs a cost decision, or is there another constraint? If scoped selection cut a PR to ~4 suites instead of 52, would you want it as a required check?
2. `quarantined-tests: applyconfiguration` and `sync-modify-downstream` are hardcoded in `tests-conformance.yaml` — is there a record of why, and is anyone tracking their removal?
3. What's the maintainer view on the 95 `workflow-failure` issues? Useful signal, or noise you'd rather see aggregated into one digest?
4. Does `DEPENDENCY-POLICY.md` constrain what dependency automation is permissible? Is auto-merge acceptable in principle for a project in the admission path?
5. What's the appetite for the agent pushing commits directly to contributor PR branches (codegen fixes) versus only opening draft PRs?
6. Should this live in `kyverno/kyverno`, in `kyverno/.github` (where `pr-branch-updater` already lives), or as a standalone repo reusable by other CNCF projects?
7. `CODEOWNERS` is 24 paths — is that deliberate minimalism, or has it just drifted?

Question 6 matters more than it looks: if this is built to be reusable across CNCF projects rather than Kyverno-specific, the project's ceiling — and the visibility you get from it — is considerably higher.

---

## 8. Second pass — verified addenda (Aug 2026, direct repo inspection)

A deeper multi-agent research pass hit a session-limit interruption before most branches reported back (CI/scoped-testing, novel-capabilities, and community/release research did not return — treat those sections above as the current best answer, not yet re-verified against fresh data). What follows is what was independently confirmed by direct repo inspection in the meantime. It sharpens three points above and adds one new, concrete finding: **the docs and CODEOWNERS have already drifted in exactly the way §2 warns they will** — this is live evidence, not a hypothetical, and it's a strong opening anecdote for the "docs that can lie" argument.

### 8.1 Monorepo question — answer is a clean "no, and here's the evidence"

`go.mod` at repo root is the only real module; `hack/api-group-resources/go.mod` and `hack/controller-gen/go.mod` are isolated tool modules (standard Go practice to keep codegen-tool deps out of the main dependency graph — not evidence of a coupling problem). Five `github.com/kyverno/*` org dependencies are pulled as ordinary pseudo-versioned Go modules, no `replace` directives pointing at local paths, no git submodules:

```
github.com/kyverno/api                v0.0.1-alpha.3.0.20260723090831-fb2785727f98
github.com/kyverno/go-jmespath        v0.4.1-0.20231124160150-95e59c162877
github.com/kyverno/kyverno-authz      v0.4.1-0.20260701230957-9101a3ffd44f
github.com/kyverno/playground/backend v0.0.0-20251124111549-b7997c02bca2
github.com/kyverno/sdk                v0.0.0-20260703121625-e0dc6fb8661a
github.com/kyverno/pkg/ext            v0.0.0-20250303002756-48769d003e55
```

This is textbook Go multi-repo hygiene, already working. The only non-Go-module cross-repo coupling in the tree is the one `replace k8s.io/pod-security-admission => github.com/kyverno/pod-security-admission ...` fork-patch, which is a deliberate, narrow override, not a structural problem. **Recommendation: state in the proposal that a monorepo restructuring was evaluated and explicitly rejected** — splitting further would fragment a codebase that's already correctly modularised, and merging the org's satellite repos in would just relocate the coupling rather than remove it. Spending phase-0 time "evaluating monorepo boundaries" as the issue suggests would mostly re-confirm this; better to say so up front and redirect that time into AGENTS.md/manifest work, which has a real gap.

### 8.2 CODEOWNERS has already drifted — with a reproducible example

Sheet 2 asked "is 24 paths deliberate minimalism, or has it drifted?" Direct verification of every path in `CODEOWNERS` (23 entries) against the working tree answers it:

- `/pkg/cosign` and `/pkg/notary` **do not exist**. The code moved to `pkg/image/verifiers/cpol/cosign`, `pkg/image/verifiers/cpol/notary`, `pkg/image/verifiers/ivpol/cosign`, `pkg/image/verifiers/ivpol/notary` (the CEL-based `ivpol`/`cpol` policy split postdates the CODEOWNERS entry). Those two ownership rules currently match **zero files** — GitHub doesn't error on a dead CODEOWNERS path, it just silently drops the rule, so nobody notices.
- `/pkg/controller/report` is a **literal typo** — singular `controller`, when the real directory is `pkg/controllers/report` (plural, confirmed to exist with real content). This rule has presumably never matched anything since it was written.

That's three of twenty-three CODEOWNERS lines silently dead — over 10% rot, undetected, in a file whose entire job is routing security-sensitive review (cosign/notary are the image-verification trust boundary). This is a gift for the proposal: it's a concrete, reproducible instance of exactly the "docs that can lie" failure mode §2 warns about in the abstract, on the CODEOWNERS file specifically, and it directly motivates **Capability: a CI check that verifies every CODEOWNERS path glob matches at least one file in the tree** — cheap, deterministic, no LLM, and demonstrably needed *today* (propose landing it as a pre-mentorship PR alongside the cache reaper).

The same drift pattern shows up in `DEPENDENCY-POLICY.md`, which names four workflow files that don't exist under those names: it says `trivy.yaml` / `trivy-periodic-scan.yaml` / `lint.yaml` / `scorecard.yaml`, but the actual files are `scan-trivy.yaml`, `periodic-trivy.yaml`, `check-golangci-lint.yaml`, and `scan-scorecard.yaml`. Same failure mode, different file — worth citing both as a pair in the proposal ("this isn't a one-off, it's a pattern the repo doesn't currently have any mechanism to catch").

### 8.3 Real 12-month churn ranking (`git log --since="12 months ago" --name-only`, by top-level+second-level dir)

This is the evidence base for *which* directories get a per-directory `AGENTS.md` first, replacing guesswork:

| Rank | Path | Touches (12mo) |
|---|---|---|
| 1 | `test/conformance` | 3996 |
| 2 | `pkg/cel` | 910 |
| 3 | `pkg/client` (generated) | 556 |
| 4 | `cmd/cli` | 499 |
| 5 | `test/cli` | 392 |
| 6 | `charts/kyverno` | 387 |
| 7 | `pkg/controllers` | 303 |
| 8 | `pkg/engine` | 230 |
| 9 | `pkg/clients` | 225 |
| 10 | `charts/kyverno-policies` | 218 |
| 11 | `api/policies.kyverno.io` | 198 |
| 12 | `pkg/image` | 175 |
| 13 | `pkg/webhooks` | 145 |
| 14 | `pkg/background` | 128 |
| 15 | `config/crds` | 99 |

Two things this ranking changes about the proposal:
- **`pkg/cel` is the single highest-churn hand-written package by a wide margin** (910 touches, more than `pkg/engine` + `pkg/webhooks` + `pkg/background` combined) — this is where the CEL-based policy types (`ValidatingPolicy`, `ImageValidatingPolicy`, etc.) are actively being built out. It's already in `CODEOWNERS`, but it should be the **first** per-directory `AGENTS.md`, ahead of `pkg/engine`/`pkg/webhooks` as the issue assumes — the issue's suggested target list is one release-cycle out of date.
- `pkg/client` (556) is 100% generated (`make codegen-client-all`) — high churn there is noise from regeneration commits, not human decision-making, so it should be *excluded* from the "needs an AGENTS.md" list and *included* in the generated-paths / autofixable list instead. Don't let raw churn numbers alone drive the doc-priority list without this filter.

### 8.4 Make targets — real count for the machine-readable manifest

`make help` currently documents **123 targets** via the `target: ## description` convention (grep-verified, not estimated). Categories, with real counts: `build-*`/`ko-build-*`/`ko-publish-*` (~23), `codegen-*` (~24), `test-*`/`test-cli-local-*` (~19, including per-policy-kind CLI test targets: `vpols`, `gpols`, `mpols`, `ivpols`, `dpols`, `vaps`, `maps` — one per CEL policy kind, confirming §8.3's point that this surface is actively multiplying), `kind-*` cluster lifecycle (~19), `dev-lab-*` observability stack (~8), plus `fmt`/`vet`/`lint`/`verify-codegen`/`release-notes`/`help`. This is a clean, already-consistent source to generate `.github/agent-manifest.json` from — every target already self-documents via the `## ` comment convention the proposal's draft schema (§Sheet 2) assumed existed; it does, which de-risks that deliverable considerably. The remaining work is purely: (a) a `make codegen-agent-manifest` script that parses this existing convention into JSON, (b) hand-annotating each target with the two fields the convention doesn't capture — `needs_cluster` (true for every `kind-*`/`test-cli-policies`/`dev-lab-*` target, false for `build-*`/unit tests) and `approx_minutes` (needs a handful of timed CI runs to seed, not guesswork).

### 8.5 What's still open (from the first addendum pass)

The CI/scoped-testing job-level breakdown, the security/auto-merge predicate are now filled in below (§9, §10). The issue/PR toil quantification and the ranked novel-capabilities list (the two research branches that failed on their first attempt and were not re-run) are still at the state described in §0–§7 above.

---

## 9. Third pass — CI job graph, scoped-test design, and flaky-test lifecycle (verified in full)

This closes out the keystone deliverable from §0 with real numbers instead of estimates.

### 9.1 The 55-job conformance matrix, in full

`tests-conformance.yaml` (1104 lines) has 55 top-level jobs. Every standard job runs through the composite action `.github/actions/tests/conformance/run/action.yaml` (install helm/cosign/yq/chainsaw → kind cluster up → load image archive → `USE_CONFIG=<kyverno-configs> make kind-install-kyverno` → apply CRDs → dynamic quarantine patch → `chainsaw test --shard-index --shard-count`). A handful bypass it entirely with hand-rolled steps: `custom-sigstore` (stands up its own Fulcio/Rekor/TUF stack), `check-tests` (runs the actual compiled CLI binary, not the image archive), and the four `helm-*` jobs (no chainsaw at all — Helm install/upgrade/uninstall lifecycle checks).

Highlights that matter for a scoped-selection design:
- Heaviest shards: `generate` (12-way) and `generating-policies` (6-way, plus a quarantine on `sync-modify-downstream`) and `deleting-policies` (6-way).
- `rbac` is matrixed on `kyverno-configs: [standard, default, force-failure-policy-ignore]` × 3 k8s versions = 9 runs from one suite.
- The three `generate/mutating-admission-policy*` families (alpha/beta/v1) are three separate top-level jobs, each pinned to a different single k8s version via a dedicated `kind-config` file, and each independently applies the `applyconfiguration` quarantine.
- Total matrix-expanded runner count for one full firing of this workflow: several hundred jobs (the earlier "~54 jobs / 150+ runners" estimate in §0 was directionally right but low — closer to several hundred once every shard × k8s-version combination is counted).

### 9.2 Two dead conformance suites — a concrete "orphan detector" proof of concept

Of the 52 top-level suite directories, two are **true orphans**, verified two independent ways:
- `test/conformance/chainsaw/configs/` (2 real, schema-valid tests) and `test/conformance/chainsaw/flags/standard/emit-events/` (1 test) have **zero `tests-path:` references anywhere in `.github/`** (`grep -rn "chainsaw/configs" .github` → 0 hits), and
- they are the **only 2 of the 52 suite roots with no `.chainsaw.yaml` file at all**.

Both were last touched only by a schema-lint chore (#14707, "add missing chainsaw test schema directives") — i.e., they look maintained (someone keeps their YAML schema-valid) but haven't actually executed in CI since at least that commit. This is a ready-made, cheap proof-of-concept for the AI-assistant pitch: **a CI check that diffs `find test/conformance/chainsaw -maxdepth 1 -type d` against every `tests-path:` value across all workflow YAML** would have caught these 3 dead test cases immediately, deterministically, no LLM required. Worth landing as a pre-mentorship PR alongside the cache reaper and the CODEOWNERS-path checker (§8.2) — three small, real, already-identified doc/test-drift bugs make a strong "I read the repo, not just the issue" opener.

### 9.3 The PR-time conformance gap is wider than §0 suggested

`comment-conformance.yaml` (the `/conformance` PR opt-in) only rebuilds images+cli and calls `tests-conformance.yaml` — the 52-suite matrix. It **never reaches `tests-k6.yaml`** (perf regression testing, 3 scenarios × 100 VUs × 1000 iterations against Prometheus) **or `tests-conformance-policy-library.yaml`** (the external `kyverno/policies` repo's own chainsaw suite, 3×12 shards). Both are `workflow_call`-only, reachable exclusively from the push-triggered `check-tests.yaml` orchestrator. So even a maintainer who explicitly asks for conformance on a PR gets zero perf-regression signal and zero policy-library-compatibility signal pre-merge — those two categories of breakage are *always* discovered after merge. This sharpens the keystone framing in §0: scoped selection needs to close this gap too, not just make the existing 52-suite opt-in cheaper.

Also notable: `check-framework.yaml` (the 5-way `vpol/mpol/gpol/dpol/ivpol` envtest integration suite) is the only test workflow of the five with a `paths-ignore` filter (skips on docs/charts/`*.md`-only diffs) and the only one with **no `failure-issue` tracking wired up** — envtest breakages on main are invisible to the maintainer digest mechanism entirely.

### 9.4 Path→suite mapping: the one blind spot that matters most

The path→suite evidence table (10 source dirs, each with a directly-cited suite correlation) confirms the mapping is buildable from CRD-`kind` correlation and directory-name matching alone — no coverage instrumentation needed for a first version. **But one structural blind spot needs to be designed around from day one**: the CEL policy CRD types (`ValidatingPolicy`, `MutatingPolicy`, `GeneratingPolicy`, `DeletingPolicy`, `ImageValidatingPolicy`) are **not defined in this repo at all** — they live in the external module `github.com/kyverno/api`, bumped roughly monthly (9 version bumps visible in `git log -p -- go.mod` over 7 months). A pure `git diff --name-only` selector is blind to this: a routine `go.mod` bump of that one dependency should force re-running essentially the entire CEL-policy conformance surface (`validating-policies`, `mutating-policies`, `generating-policies`+namespaced, `deleting-policies`+namespaced, `image-validating-policies`+namespaced, plus every `pkg/cel/...` and `pkg/webhooks/resource/{vpol,mpol,gpol,ivpol}` unit-test package), and a naive selector would instead see "only go.mod changed" and under-select. **Any scoped-test-selection proposal must state this as an explicit special rule**, not an edge case discovered later: `go.mod` diffs touching `github.com/kyverno/api` (or any other `github.com/kyverno/*` dependency) map to a hardcoded broad suite set, distinct from the normal path-based lookup.

For an empirical (not just heuristic) version of the mapping: no `GOCOVERDIR`/coverage-instrumented-binary mechanism exists anywhere in the repo today (confirmed by grep). The proposed net-new approach — build the controller/webhook binaries with `go build -cover`, run each chainsaw suite against the instrumented image, `go tool covdata percent` per suite, and diff-mine the resulting coverage matrix — is a real, buildable Phase-2 deliverable, not currently duplicated by anything.

### 9.5 Unit-test selection via Go's build graph — a verified, working pipeline

Because the whole repo is one Go module (§8.1), a plain `go list -deps` reverse-dependency walk works without any per-module stitching. This was run live against the repo, not estimated:

```bash
go list -deps -json ./pkg/... > deps.json                         # 2m32s, 59MB, ~1500 packages
jq -r 'select(.Deps) | .ImportPath as $p |
       .Deps[] | select(startswith("github.com/kyverno/kyverno")) |
       "\($p)\t\(.)"' deps.json > edges.tsv                        # 18,060 edges (single pass — .Deps is already transitive)
awk -F'\t' -v pkg="$CHANGED" '$2==pkg {print $1}' edges.tsv         # reverse lookup
```

Real, verified numbers from this run: `pkg/engine` → 45 transitive dependents (41 with `_test.go` files, i.e. real `go test` targets out of 137 total testable packages under `./pkg/...`); `pkg/cel/policies/vpol` → exactly **1** dependent, `pkg/webhooks/policy` (confirming it's the sole caller in the tree); `pkg/webhooks/handlers` → 12 dependents; `pkg/image/verification` → 0 (a leaf package, only imported from outside `./pkg/...`). Same `go.mod`-diff caveat as §9.4 applies: this pipeline needs the identical special-case rule for external-module bumps.

### 9.6 Flaky-test lifecycle: quarantine is a runtime patch, not a static flag

The mechanism is more specific than "hardcoded quarantine list" — it's a live YAML mutation. `.github/actions/tests/conformance/run/action.yaml:110-141` takes the workflow-level `quarantined-tests` input (comma-separated directory *basenames*), finds every directory under the job's `tests-path` matching that basename, and runs `yq eval '.spec.skip = true' -i` on each matching `chainsaw-test.yaml` **at CI runtime**, before chainsaw runs. That's distinct from — and additional to — 4 **statically committed** `skip: true` tests found by direct grep, none carrying any explanatory comment or issue reference:
- `verify-images/clusterpolicy/standard/update-multi-containers`
- `generating-policies/clone/sync/{sync-delete-policy, sync-delete-one-trigger, sync-delete-one-source}`

Two more concrete gaps for the quarantine-lifecycle deliverable (§0, §Sheet 2) to close: **Chainsaw itself runs a beta release** (`v0.2.15-beta.3`, pinned via the installer action `kyverno/action-install-chainsaw@...#v0.2.14`) with `failFast: true` and no retry field anywhere in any `.chainsaw.yaml` — a single flaky step fails the whole test immediately, no built-in flake tolerance exists. And **no JUnit/JSON report output is configured anywhere** (`--report-format`/`--report-path` — flags chainsaw already supports upstream — are never passed); failures are visible only as raw stdout/exit-code in the Action log. Adding those two flags to the composite action and uploading the artifact is a small, concrete, low-risk PR that unlocks all downstream flaky-test analytics — currently there is *no* machine-readable per-test result anywhere to build a flip-rate detector on top of.

### 9.7 `failure-issue` — exact lifecycle (confirms and sharpens §0's free-time-series claim)

`.github/actions/workflow/failure-issue/action.yaml` (121 lines): dedup key is `workflow-failure:${workflowName}:${branch}`, embedded as an HTML comment in the issue body; title is stable (`Workflow failed: ${workflowName} (${branch})`, no run-id/commit in it) so repeated failures update the *same* issue rather than spawning new ones; on success, if a matching open issue exists it posts a closing comment and sets `state: closed`, otherwise does nothing. Wired into `check-unit-tests.yaml`, `check-cli-tests.yaml`, and the top-level `check-tests.yaml` — all **push-only**, so PR failures never open tracking issues, only main/release-branch breakages do. `check-framework.yaml` is the one sibling workflow with no such wiring at all (§9.3). This confirms the open→close-interval time-series proposed in §Sheet 1 is real and free, but the dedup granularity is per (workflow, branch) — not per job or per test — so extracting a *test-level* flip-rate needs the JUnit/JSON output from §9.6, not the issue history alone.

---

## 10. Third pass — dependency automation, security scanning, and the auto-merge predicate (verified in full)

### 10.1 Every scanner in the repo runs post-merge or on a schedule — except one

Direct, full read of all seven scan/periodic workflows:

| Workflow | Trigger | Gates PRs? |
|---|---|---|
| `scan-trivy.yaml` | push to main/release-*, manual | No |
| `periodic-trivy.yaml` | daily schedule | No |
| `sync-trivy-issues.yaml` | on Trivy workflow completion | No (issue bookkeeping only) |
| `scan-semgrep.yaml` | daily schedule | No — and runs with `--no-error`, so it structurally *cannot* fail its own job even if triggered on a PR |
| `scan-scorecard.yaml` | weekly schedule + push to main | No |
| `scan-sonarcloud.yaml` | push to main/release-* | No |
| `scan-fossa.yml` | push to main | No |
| `check-sha-pinned-actions.yaml` | **pull_request** + push | **Yes — the only one** |

This is a sharper, fully-verified version of §Sheet 1's "dependency review → PR-time gate is the real gap" point: it's not just dependency-review-action that's missing, it's that **every single vulnerability/license/quality scanner in the repo is structurally incapable of blocking a PR today**, `check-sha-pinned-actions` being the sole exception. A vulnerable or malicious dependency bump merges cleanly and is only caught after the fact, surfacing as an auto-filed `codeql`/`security` issue via `sync-trivy-issues.yaml`. This is the single cleanest "PR-time security gate" pitch in the whole proposal — concrete, quantified, and distinct from the already-known Trivy/Semgrep/Scorecard *existing*.

### 10.2 Dependabot: grouping solved PR count, not review latency — real 90-day data

`dependabot.yml` has exactly 2 ecosystems (`gomod`: root + 2 hack/ dirs; `github-actions`: root + `.github/actions/*/`), both daily, `rebase-strategy: disabled` on both (which is *why* `pr-branch-updater.yml` exists — it's compensating for this specific Dependabot setting), only 3 groups (kubernetes/sigstore/otel) covering a fraction of the ~107 direct deps, no `ignore:` blocks at all.

Live data (2026-08-16): **62 Dependabot PRs merged in the last 90 days, 1,401 all-time, 1 open right now** (#17141). A 12-PR commit-level sample found **zero cases of Dependabot's own commit ever being amended, force-pushed, or fixed up** — CI passed on the first commit in every sampled case. The only extra commits were `Merge branch 'main' into dependabot/...`, i.e. keeping the PR synced while it waits — some of these merges are authored by `kyverno-pr-updater[bot]` (the same reusable `pr-branch-updater.yml` workflow, confirming maintainers already built machinery for exactly this problem). Time-to-merge ranged from ~26 minutes to ~3 days; two bumps (a `grpc` bump and a Homebrew-action bump) needed **5–6 repeated re-merges from `main`** over ~25 hours, purely from `main` moving while the PR waited for a human click — pure latency toil, not a broken-bump problem.

**This meaningfully updates §0's "the pain here is smaller than the issue implies" caveat**: the *count* problem is genuinely small (1 open PR right now), but the *data shows a real, quantified review-latency problem* that grouping does nothing to address — every single sampled bump, however trivial, sits for hours waiting on a human, and some churn through repeated rebases while waiting. That's a stronger, evidence-backed case for auto-merge than the original framing, not a weaker one — reframe accordingly.

### 10.3 Supply-chain output signing is mature; the gap is entirely on the input side

Full read of `images-publish.yaml`, `release.yaml`, and `.github/actions/image/publish/action.yaml`: every one of the 7 published images (built via `ko`, no Dockerfiles exist anywhere in the repo) gets a CycloneDX SBOM (`CycloneDX/gh-gomod-generate-sbom`), keyless OIDC cosign signing, `cosign attach sbom`, and a SLSA3 provenance attestation via `slsa-framework/slsa-github-generator` — plus a pre-publish `trivy-action` filesystem scan of the whole repo, and (on tagged releases) the Helm/Flux install manifest itself gets pushed as an OCI artifact and cosign-signed too. This is genuinely comprehensive supply-chain *output* hardening — there is no meaningful gap to propose on the signing/attestation side. The gap, confirmed by §10.1, is entirely upstream: nothing scans an *incoming* dependency change before merge.

`check-sha-pinned-actions.yaml` (the one PR-gating security check) enforces full-commit-SHA pinning on every `uses:` in every workflow, with a single named exception (`slsa-framework/slsa-github-generator`, which needs a semver tag per the SLSA spec — this is *why* it's the sole allowlist entry, not an oversight). **Concrete implication for any agent that edits workflow files**: it must never write a floating tag (`@v4`, `@main`) — every action reference it touches must resolve to a full SHA with a `# vX.Y.Z` comment, exactly like Dependabot's own `github-actions` ecosystem bumps already do. The clean design is to have the agent defer SHA resolution to Dependabot rather than hand-computing it.

Go dependency scale: 107 direct + 305 indirect (413 total require lines), `go.sum` 1318 lines, `vendor/` confirmed absent, no `GOFLAGS`/`GOPRIVATE`/`GOSUMDB`/`GOPROXY` overrides anywhere — the 5 `github.com/kyverno/*` org dependencies are pulled and checksum-verified exactly like any third-party module, no special trust carve-out.

### 10.4 The auto-merge predicate, and an explicit policy conflict to flag to maintainers

```
function should_auto_merge(pr):
    if pr.author != "dependabot[bot]": return False
    bump = parse_dependabot_metadata(pr)   # dependency-type, update-type, dependency-group from Dependabot's own commit trailer
    if bump.update_type == "semver-major": return False              # always human
    risk_tier = classify(bump.ecosystem, bump.update_type)            # low: gomod/actions patch, actions minor; medium: gomod minor
    if risk_tier is None: return False                                # unclassified shape -> human
    if pr.has_label("hold"|"do-not-merge"|"wip"): return False
    if not all_required_checks_green(pr): return False                 # lint, unit tests, sha-pin check
    if pr.branch_is_stale(main): return False                          # pr-branch-updater.yml should resolve this first
    if risk_tier == "medium" and not vulnerability_scan_clean(bump):    # doesn't exist yet — see 11.1
        return False
    return True
```

**This needs to be surfaced as an explicit open question, not assumed**: `DEPENDENCY-POLICY.md` states, unconditionally, *"All proposed updates are reviewed by maintainers and must pass CI before merging."* A predicate that auto-merges patch-level bumps with zero human click contradicts the literal text of that sentence, even if it may be compatible with its *intent*. The proposal should present this as a decision point for maintainers ("does 'reviewed by maintainers' require a human click, or would a policy-as-code gate satisfy it for a defined low-risk subset?") and note that adopting auto-merge would likely require an accompanying edit to `DEPENDENCY-POLICY.md` — treating this as settled would be presumptuous given a CNCF security project's own written policy says otherwise.

### 10.5 Two more automation gaps invisible to Dependabot entirely

- **No Dockerfiles exist anywhere in the repo** — images build via `ko` (`.ko.yaml`, base image `ghcr.io/wolfi-dev/static:alpine`, pinned by a **floating tag, not a digest**). Because Dependabot's `docker` ecosystem parses `Dockerfile`/`docker-compose.yml`, it structurally cannot see or bump this base image even if added to `dependabot.yml` — there is currently **no automation of any kind** tracking updates to it. Closing this needs either a digest pin + small custom watcher, or accepting the drift.
- **Helm chart dependencies are unmanaged.** `charts/kyverno/Chart.yaml` pins 3 genuinely external chart versions (`kyverno-api`, `openreports`, `reports-server`) — Dependabot has no generic Helm-chart-dependency ecosystem, so these are untouched by any bump automation today, confirmed by the full `dependabot.yml` read (only `gomod` + `github-actions`, nothing else).

---

## 11. Consolidated addendum summary

Sections 8–10 replace the "still open" items from the interrupted first addendum pass. Combined with the already-completed research in §0–§7, the standing gaps in this document are now only: **real issue/PR toil quantification** (open counts, age distribution, response-time — the "Issue triage and repro harness" branch, which failed twice on session limits and was not re-run a third time) and **the ranked novel-capabilities list re-verification** (the "Agent architecture" branch, same fate — §4's existing list stands as the current best draft, not yet independently re-checked). Everything else in this document is now grounded in a live, verified pass over the actual repo and GitHub data as of 2026-08-16.

---

## 12. Fourth pass — new facts not covered by §8-10 (independent cross-check, same date)

A second, independently-run set of agents re-covered the structure/AGENTS.md and dependency/CI ground in §8-10 using a slightly different methodology (e.g. full `./...` vs `./pkg/...`-scoped build-graph, 54 vs 55 conformance jobs, 51 vs 52 counted suite dirs). Where the two passes overlap, **conclusions matched despite different exact intermediate numbers** — e.g. both independently found ~290-range total conformance runner-jobs, both independently found the same 2 orphan suites (`configs/`, `flags/`), both independently found Dependabot's real problem is review-latency not count, both independently found DEPENDENCY-POLICY.md's language conflicts with any auto-merge predicate. Treat the small numeric deltas as measurement noise, not a contradiction, and cite whichever pass's number is closer to what CI shows at proposal-writing time. What follows is only the genuinely **new** material from the second pass, not yet in §8-10.

### 12.1 Real maintainer-domain routing table (OWNERS.md, not MAINTAINERS.md)

`MAINTAINERS.md` and `GOVERNANCE.md` are both one-line pointers to `kyverno/community` — no local content. The actual roster with area ownership lives in **`OWNERS.md`**: 9 Maintainers with named domains (Jim Bugwadia — Validation/CLI/Docs; Shuting Zhao — Engine/Admission/Background/Mutation/Events; Charles-Edouard Brétéché — Engine/CEL/Autogen/Validation/CLI/Helm; Vishal Choudhary — Engine/Image Verification; Mariam Fahmy — Generation/Admission Policy/Policy Exception; Frank Jogeleit — Cleanup/Reporting/Metrics/Testing; Liang Deng — Pod Security Admission; Yugandhar Suthari — Global Context/CLI; Xu Liu — Global Context/CEL Validation), plus 1 Reviewer (Ammar Yasser — CEL, Reporting; reviewer tier, not maintainer/approver). This is a real, two-tier, domain-scoped routing table that already exists and is more precise than CODEOWNERS — a reviewer-suggestion agent should route by this table for CEL/engine/image-verification changes specifically, falling back to CODEOWNERS' generic team ping only outside these named domains.

### 12.2 CODEOWNERS gaps beyond the 3 dead lines already found in §8.2

Cross-referencing all 23 real CODEOWNERS path lines against the churn ranking in §8.3 surfaces **4 more high-churn directories with zero coverage at all** (falling to the bare `*` wildcard, not even a broad-prefix match): `pkg/controllers/` (303 churn — the correctly-spelled plural dir; its only near-match is the dead singular typo from §8.2), `pkg/image/` (175 churn — arguably the single most security-sensitive directory in the repo, image signature verification, with zero named-owner routing), `pkg/clients/` (225 churn, client wrappers — distinct from the generated `pkg/client/` singular, which is expected to be uncovered), and `config/` (config/crds 99 churn). An AI reviewer-router relying on CODEOWNERS alone would silently fall back to the generic core-maintainers team for exactly the areas that most need a named owner.

### 12.3 Generated-code inventory, with precise counts

Full census of tracked generated files: `zz_generated.{deepcopy,register}.go` — 14 files (7 API packages × 2); `pkg/client/**` — **178 files** (clientset/listers/informers/wrappers); `config/crds/**` — 21 manifests; the Helm CRDs subchart (`charts/kyverno/charts/crds/templates/**`) — 11 files, but **only 3 of the 4 API groups are mirrored there** (the CEL-based `policies.kyverno.io` CRDs are absent from this Helm subchart — worth flagging as a possible real gap, not just a doc gap); API reference HTML (`docs/user/crd/*.html`) — 11; CLI API docs (`docs/crd/v1/index.html`) — 1; Helm chart READMEs — 2; CLI command docs (`docs/user/cli/commands/*.md`) — ~25. **Total ≈263 tracked generated files**, none of which AGENTS.md currently states should never be hand-edited outside of `zz_generated.*` and `pkg/client/` — the CRD/doc outputs have the same rule implicitly but it's never written down.

### 12.4 Concrete drafts: `agent-manifest.json` and `ai-maintainer.yaml`

Both were fully drafted against real, grep-verified Makefile targets and Glob-verified paths (not invented): a `.github/agent-manifest.json` shape with `cmd`/`needs_cluster`/`approx_minutes`/`autofixable`/`verify` fields per target (covering build/fmt/codegen/test/kind/ko targets), and a `.github/ai-maintainer.yaml` split into `never_touch` (verified real paths: `pkg/image/verifiers/{cpol,ivpol}/{cosign,notary}/**`, `api/kyverno/v1/**`, `pkg/tls/**`, `pkg/utils/tls/**`, `charts/kyverno/templates/rbac/**`), `human_review_required` (other API versions, `pkg/engine`, `pkg/webhooks`, `pkg/cel`, `pkg/controllers`, `go.mod`/`go.sum`), and `agent_autonomous` (the generated paths from §12.3, dependency-bump PRs marked propose-only per DEPENDENCY-POLICY.md's maintainer-approval requirement). Full text of both drafts is in `lfx-research-raw-structure.md` §4-5 — ready to paste into a proposal or a pre-mentorship PR with minimal editing.

### 12.5 One more governance fact: two-tier structure constrains "agent-autonomous merge" more than DEPENDENCY-POLICY.md alone suggests

Beyond DEPENDENCY-POLICY.md's blanket "reviewed by maintainers" language (already flagged in §10.4 as conflicting with auto-merge), `OWNERS.md`'s Maintainer/Reviewer split (§12.1) means even "maintainer review" isn't uniform — a Reviewer-tier approval may not count as sufficient sign-off for a merge in the same way a Maintainer's does. Any auto-merge or reviewer-routing design should treat this as a second axis of the policy question in §10.4, not fold it into a single "human approved" boolean.
