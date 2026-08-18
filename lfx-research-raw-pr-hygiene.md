# Raw research — PR hygiene (stale PRs, auto-update, re-trigger CI, nudging)

Supporting notes for the PR Hygiene section of `proposal.md`. Live GitHub data and workflow reads verified directly against `kyverno/kyverno` on 2026-08-18 (see the proposal section itself for the numbers and file links — this file exists just to hold what doesn't belong inline in the proposal).

## 1. Open questions for maintainers (PR Hygiene)

Raised rather than assumed — several of these depend on maintainer preference, not something I should decide unilaterally.

1. Is the silent "behind but not conflicting" category (PRs that are stale but not yet showing the `merge-conflicts` label) something you'd want auto-updated proactively, or only surfaced for a human to decide?
2. What idle threshold should trigger the first nudge — 7 days, 14, something else? Given 55% of open PRs are already past 14 days idle, a threshold set too low would flood everyone on day one.
3. Should the nudge comment on the PR itself, or go through a private digest only, at least initially?
4. Can I get visibility into what `kyverno/.github`'s `pr-branch-updater.yml` reusable workflow actually selects, so this doesn't duplicate or conflict with it?
