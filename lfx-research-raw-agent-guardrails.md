# Raw research — Repo structure & agent guardrails (monorepo, AGENTS.md, PR metadata, skills delegation)

Supporting notes for the Repo Structure & Agent Guardrails section of `proposal.md`. Module-graph and churn analysis verified directly against `kyverno/kyverno` (see `kyverno-lfx-research.md` §8.1, §8.3 for the full evidence this section summarizes).

## 1. Open questions for maintainers (Repo Structure & Agent Guardrails)

1. `pkg/cel/` is the highest-churn hand-written package (910 touches/12mo) but has the least `AGENTS.md` coverage — agree it should be the first per-directory stub, ahead of `pkg/engine`/`pkg/webhooks` as the mentor issue assumed?
2. Is expressing the agent's safety boundary as an actual Kyverno `ClusterPolicy` (validated via `kubectl-kyverno` in CI) something you'd want as a real Phase-1 deliverable, or is that a stretch idea better left for later once `.github/ai-maintainer.yaml` itself is trusted?
3. Who owns `.github/ai-maintainer.yaml` once it exists — is CODEOWNERS-gating it enough, or does changing an agent's autonomy need a higher bar (e.g. two maintainer approvals)?
