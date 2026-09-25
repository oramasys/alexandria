# Kungfu status note (2026-09-25)

**Scope:** factual status for migration routing — not a claim that a published remote exists.

## Findings

| Claim | Status |
| --- | --- |
| GitHub repo `oramasys/kungfu` | **Absent** when the 2026-09-25 governance pack was authored (org list + broader GitHub search returned empty). Do not route PRs or CI to a remote that is not confirmed. |
| Local Wave 0 tree | **Present** as an operator-local workspace clone under `oramasys/kungfu`. Contents described by that tree's own README: portable method/skills contracts, decision ledger, security-boundary invariants, fail-closed source-manifest validator. Draft / Wave 0 only. |

## Routing implication

Until a published `oramasys/kungfu` remote is explicitly created and verified, v2 skill-method work that the pack might have aimed at "kungfu" should either:

1. stay local to the Wave 0 tree without implying org publication, or
2. retarget a confirmed `oramasys/*` repo after human confirmation of the destination.

Alexandria records this note so later agents do not invent a remote or erase the local Wave 0 fact.

## Non-claims

- This note does **not** assert that a GitHub repository currently exists.
- This note does **not** publish local topology, device names, or personal workstation paths.
- This note does **not** migrate kungfu content into Alexandria.
