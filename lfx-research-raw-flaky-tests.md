# Raw research — Flaky test lifecycle & maintainer digest

Supporting notes for the Flaky Test Lifecycle & Maintainer Digest section of `proposal.md`. Git history and workflow reads verified directly against `kyverno/kyverno` on 2026-08-18 (see the proposal section itself for file links and PR numbers — this file exists just to hold what doesn't belong inline).

## 1. Open questions for maintainers (Flaky Test Lifecycle & Maintainer Digest)

1. Chainsaw is pinned to a beta release (`v0.2.15-beta.3`) with `failFast: true` and no retry field anywhere — is upgrading it or adding retry tolerance in scope, or is quarantine-only the preferred fix?
2. What flip-rate threshold and clean-run count should trigger auto-quarantine and auto-de-quarantine — Kubernetes SIG-Testing style precedent exists, but the right numbers for a repo this size are a maintainer call.
3. Should the weekly digest go to a pinned GitHub issue (edited in place), a Slack channel, or both?
4. `check-framework.yaml` is the one test workflow with no `failure-issue` wiring at all — should it feed the same digest, or is envtest breakage intentionally excluded from this signal?
