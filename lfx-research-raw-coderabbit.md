# Raw research — CodeRabbit-based PR review & enrichment

Supporting notes for the CodeRabbit section of `proposal.md`. Feature claims verified directly against `docs.coderabbit.ai` on 2026-08-18 (see the proposal section itself for the links — this file exists just to hold what doesn't belong inline in the proposal).

## 1. Open questions for maintainers (CodeRabbit)

Raised rather than assumed — several of these depend on maintainer preference or on details CodeRabbit's own docs don't fully settle.

1. CodeRabbit's OSS rate limit is star-count-dependent (1–10 PR reviews/hour per [docs.coderabbit.ai/management/plans](https://docs.coderabbit.ai/management/plans)) and re-runs on every push. What's `kyverno/kyverno`'s actual observed limit, and does it comfortably cover 304 open PRs plus daily Dependabot pushes, or does it need throttling on our side (e.g. skip re-review on pure rebase pushes)?
2. Multi-repo/knowledge-base linking (cross-repo breaking-change detection) needs a Pro plan for manual linking and Pro+/Enterprise for automatic linking — does the free-for-OSS grant extend that far, or is this a real gap for a project split across `kyverno/kyverno`, `kyverno/policies`, `kyverno/reports-server`, etc.?
3. Should the `@coderabbitai summary` placeholder replace the "Proposed Changes" section of `PULL_REQUEST_TEMPLATE.md`, or just append below the existing template as CodeRabbit does by default?
4. Pre-merge checks — title/description/docstring-coverage/issue-assessment — should start in `warning` mode per CodeRabbit's own rollout advice. Who decides when (and if) to flip any of them to blocking `error` mode?
5. The PR template already has an "AI Usage Policy" self-disclosure checkbox. Does automated slop-detection labelling on top of that read as a useful cross-check to maintainers, or as an adversarial move toward contributors who already disclosed honestly?
6. CodeRabbit's post-merge action can append to `CHANGELOG.md` directly; `release-drafter` (raised separately, for GitHub Release notes at tag time) writes a different artifact. Worth running both, or is one clearly preferred?
