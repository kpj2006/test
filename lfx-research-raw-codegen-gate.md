# Raw research — Codegen/verify gate

Supporting notes for the Codegen & Verify Gate section of `proposal.md`. Verified directly against `.github/workflows/check-codegen.yaml` on `kyverno/kyverno` on 2026-08-18.

## 1. Open questions for maintainers (Codegen & Verify Gate)

1. Is there a reason the patch is left for a human to `git apply` today — e.g. a trust boundary around what pushes to a PR branch automatically — or is that purely because nobody's built the auto-commit step yet?
2. Should the fixup commit be attributed to the PR author (via a bot with write access to their branch) or land as a separate bot commit — this affects DCO sign-off attribution too.
