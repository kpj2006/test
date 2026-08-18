# Raw research — Smaller automation gaps (DCO, backport, security advisory)

Supporting notes for the Smaller Automation Gaps section of `proposal.md`. Verified directly against `kyverno/kyverno` on 2026-08-18.

## 1. Open questions for maintainers (Smaller Automation Gaps)

1. Should the DCO reactive-guidance comment come from the existing behaviorbot config (`.github/config.yml`) or a separate bot — reusing the existing welcome-bot identity is simpler but mixes concerns.
2. For backport conflicts: should the bot open a draft PR with conflict markers left in for a human to resolve, or just notify and leave the branch alone?
3. Is the "never let an LLM agent read private security advisories" boundary something you'd want written into `.github/ai-maintainer.yaml` explicitly (§8), or is it obvious enough not to need codifying?
