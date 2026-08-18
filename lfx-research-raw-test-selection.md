# Raw research — Scoped test selection (path→suite mapping, unit-test selection, conformance gaps)

Supporting notes for the Scoped Test Selection section of `proposal.md`. Numbers below come from the live repo audit in `kyverno-lfx-research.md` §9 (conformance job graph, path→suite design, unit-test selection) — see the proposal section itself for the primary-source file links; this file exists just to hold what doesn't belong inline in the proposal.

## 1. Open questions for maintainers (Scoped Test Selection)

Raised rather than assumed — several of these depend on maintainer preference or on constraints only they know.

1. Given a full push-triggered run of `tests-conformance.yaml` expands to several hundred runner-jobs, what's an acceptable pre-merge runner-minute budget per PR? That's the number a scoped subset needs to fit inside.
2. Should a `go.mod` bump of `github.com/kyverno/api` (or any other `github.com/kyverno/*` module) always force the full CEL-policy conformance surface, or is there a finer-grained signal worth extracting from that dependency's own diff?
3. `tests-k6.yaml` (perf regression) and `tests-conformance-policy-library.yaml` (external `kyverno/policies` compatibility) are only reachable from the push-triggered orchestrator today — should a scoped pre-merge run include a lightweight slice of these two as well, or is post-merge acceptable specifically for this pair?
4. Is `.github/labels.yml` the right place for new `suites:`/`unit:` keys, or would maintainers prefer a separate file to keep PR-labelling and test-selection concerns decoupled, even at the cost of duplicating the same path globs?
5. `check-framework.yaml` has no `failure-issue` wiring, unlike the other test workflows — deliberate, or a gap worth closing alongside this?

## 2. Unit-test selection via Go's build graph — full verified pipeline

Run live against the repo, not estimated. Because the whole repo is a single Go module, a plain `go list -deps` reverse-dependency walk works without any per-module stitching:

```bash
go list -deps -json ./pkg/... > deps.json                         # 2m32s, 59MB, ~1500 packages
jq -r 'select(.Deps) | .ImportPath as $p |
       .Deps[] | select(startswith("github.com/kyverno/kyverno")) |
       "\($p)\t\(.)"' deps.json > edges.tsv                        # 18,060 edges (single pass — .Deps is already transitive)
awk -F'\t' -v pkg="$CHANGED" '$2==pkg {print $1}' edges.tsv         # reverse lookup
```

Verified results: `pkg/engine` → 45 transitive dependents (41 with `_test.go` files, out of 137 total testable packages under `./pkg/...`); `pkg/cel/policies/vpol` → exactly 1 dependent (`pkg/webhooks/policy`, confirming it's the sole caller in the tree); `pkg/webhooks/handlers` → 12 dependents; `pkg/image/verification` → 0 (a leaf package, only imported from outside `./pkg/...`). The same `go.mod`-diff caveat from the proposal's §4.3 applies here too — this pipeline needs the identical special-case rule for external-module bumps.
