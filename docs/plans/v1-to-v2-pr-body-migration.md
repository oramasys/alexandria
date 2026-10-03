> **Alexandria ingest:** 2026-09-25 · docs-only plan copy · portable-memory scrubbed.

# /PLAN: v1 → v2 PR-Body Reporting Migration

**Status:** plan, not yet executed.
**v1 source (mine from):** `diazMelgarejo/orama-system` — `scripts/cursor/append-pr-body.sh`,
`scripts/cursor/pr-body-grant-lib.py`, `bin/orama-system/skills/git-history-surgery/references/text-metadata-integrity-and-review-completeness-reference-card.md`,
`bin/orama-system/cidf/references/remote-content-integrity-reference-card.md`.
**v2 target:** `oramasys/oramasys` (confirmed via direct search: no reporting
tooling exists there yet — this is genuinely greenfield, not a duplicate).

## 0. A naming correction, stated plainly rather than guessed around

`oramasys/kungfu` had **no GitHub remote** when this plan was authored (org repo list and broader GitHub search both empty). A **local Wave 0** tree does exist under the operator's `oramasys/kungfu` workspace clone (contracts, decision ledger, fail-closed source-manifest validator); that local presence must not be read as proof a published remote exists. The real v2
org today is `perpetua-core`, `oramasys`, `agate`, `telos`, `phylax`, plus
two very new, still-mostly-empty repos: `alexandria` (docs hub, untouched
since 2026-09-01) and `anamnesis` (epistemic memory / provenance ledger,
actively touched as recently as 2026-09-16). If "kungfu" names something
real under a different spelling or a repo not yet created, this plan can't
route to it — say which repo you mean and I'll retarget. Everything below
proceeds against the confirmed five-repo v2 map instead of guessing.

`anamnesis`'s description ("provenance/audit ledger, controlled promotion")
is the one genuine candidate for a *new* invariant this migration should
account for, flagged in §3.

## 1. What v1 actually built (the mining inventory)

Four real, tested assets exist in `orama-system` today, all from this
session's own work, not assumed:

| Asset | What it does | Reusable as-is? |
| --- | --- | --- |
| `pr-body-grant-lib.py` | HMAC-bound operator grant, nonce replay-state machine (`minted → reserved → remote_applied → consumed`), crash reconciliation | Logic yes, location no — see §2 |
| `append-pr-body.sh` | Three sequential integrity guards (`STALE_ON_REREAD`, `STALE_PREWRITE`, `POSTWRITE_MISMATCH`), each with a stable `PR_BODY_E_*` code, `guard_trace` test seam | Logic yes, guard set changes — see §3 |
| `text-metadata-integrity-and-review-completeness-reference-card.md` | 7 rules: representation-aware hashing, TDD completeness, review-resolution gating, stale-review scoping, CI-fixture-dependency boundary, relayed-report verification, honest race-bound documentation | Directly portable — these are process rules, not code |
| `test_append_pr_body_guard_checkpoints.py` pattern | Ledgered fake transport + explicit manifest + bidirectional structural test (catches guard addition *and* deletion) | Directly portable as a testing pattern |

## 2. The invariant that changes the shape of the port: ownership

v1's two files live together in one repo with no cross-repo authority
question. v2 doesn't have that luxury — this session's own established,
repeatedly-verified invariant (first found on `perpetua-core`, later reused
for `agate`) is:

> Core never imports upward from policy/application repositories. A CI-only
> dependency must not be published through any package metadata, not even a
> named optional extra — install it from a separate requirements file.

Applied here: **the reporting tool belongs in `oramasys/oramasys`**, the
orchestration/application layer — not `perpetua-core` (the scheduler has no
business knowing about PR bodies) and not `telos` (endpoint authority, not
reporting). This isn't a new decision; it's the same ownership split this
session already used for `route_node`'s agate-authority migration, applied
to a different capability.

**GitHub API access is itself an egress concern.** v1's scripts call `gh`
directly with no transport-authority layer, because v1 has none. v2 has
Telos. Per Telos's own boundary docs (read while wiring §1 above): Telos
owns *IdP and purpose-scoped HTTP*, explicitly **not** arbitrary CLI
subprocess invocation, and the `gh` CLI's own auth (its stored token,
`GH_TOKEN` env var) is a different trust surface than an HTTP purpose Telos
would authorize. Recommendation: **do not** route `gh` calls through Telos
`request()` — wrong layer, `gh` is a local trusted tool invocation, not a
purpose-scoped HTTP dial. State this explicitly in the ported code's own
docstring so a future reader doesn't "fix" it by force-fitting Telos where
it doesn't belong.

## 3. Guard-set changes for v2

Not a 1:1 port. Two of v1's three guards don't survive the port unchanged;
one gets stronger because v2 has a real dependency graph to lean on.

| v1 guard | v2 treatment | Why |
| --- | --- | --- |
| `PR_BODY_E_STALE_ON_REREAD` | Ported unchanged | Pure API-race detection, no v1-specific assumption |
| `PR_BODY_E_STALE_PREWRITE` | Ported unchanged | Same |
| `PR_BODY_E_POSTWRITE_MISMATCH` | Ported, canonicalization function reused verbatim | The narrowed (single-trailing-LF) canonicalization from this session's own audit-driven fix is correct wherever this pattern is reused — do not regress to the wider `.rstrip()` |
| *(new)* `PR_BODY_E_ANAMNESIS_UNRECORDED` — **tentative, flagged not decided** | If `anamnesis` (the audit/provenance ledger repo) is live by the time this ships, a v2 report-write plausibly needs a corresponding provenance-ledger entry before it counts as complete — the same "don't claim completion before evidence exists" discipline Rule 3 of the reference card already establishes, extended to a second system of record | `anamnesis` is too new and undocumented (checked directly — mostly empty) to design against confidently. Named here so it isn't silently missed if it matures before this migration ships; not designed further until its own contract exists. |

## 4. The bigger structural change: comment-first, not body-first

Per the honest bound this session already established (Rule 7, verified
directly against a real GitHub bug report: no server-side CAS exists for
PR-body PATCH): the race is real and unclosable by re-reading harder. v1
kept body-replacement as primary and accepted the bound. v2, being new, gets
to choose the smaller-blast-radius default instead of inheriting v1's.

**v2 default: report via PR *comment*, not body edit.** A comment is its
own independent object — no lost-update race exists for it at all, since
there's nothing to overwrite. This doesn't "solve" the race, it **avoids
needing to** for the common case (status updates, findings summaries),
which is most of what `append-pr-body.sh` was actually used for across this
session's real usage.

Body-edit capability is *not* dropped — ported as a secondary path for the
one case that genuinely needs it: the repo-enforced `## Summary` convention
itself (a CI guard elsewhere already requires this heading to exist in the
body specifically, not a comment). Two tools, not one tool doing two jobs
badly:

- `report_pr.py` (new, comment-first, v2-native) — no integrity guards
  needed, since comments have no lost-update race to guard against; state
  this explicitly in its own docstring rather than let a future reader
  wonder why the guard machinery didn't come along.
- `append_pr_summary.py` (ported from `append-pr-body.sh`, body-edit,
  used only for the `## Summary` case) — keeps all three guards, the grant
  model, and the reference card's Rules 1–6 in full.

## 5. Concrete porting steps

1. **New directory**: `oramasys/oramasys/scripts/reporting/` (or the repo's
   existing scripts convention, checked at execution time rather than
   assumed here) — houses both tools from §4, not mixed into an unrelated
   directory.
2. **Port `pr-body-grant-lib.py`**'s nonce/HMAC/replay-state-machine logic
   as a shared module both tools import — the grant model itself doesn't
   change between v1 and v2, only where it's called from.
3. **Port the canonicalization function** (single-trailing-LF strip)
   verbatim into a small shared utility both `report_pr.py` and
   `append_pr_summary.py` use — one function, not two copies that can drift,
   matching this session's own "single source of truth" discipline.
4. **Write `report_pr.py` first** (comment-first, no guards needed) — it's
   the smaller, lower-risk piece and covers the majority use case.
5. **Port `append_pr_summary.py`** with all three guards, using the
   `test_append_pr_body_guard_checkpoints.py` ledgered-transport +
   explicit-manifest + bidirectional-structural-test pattern verbatim —
   this testing pattern is architecture-agnostic and was independently
   audited twice already this session.
6. **Port the reference card**, retitled for v2 and cross-linked from
   `oramasys/oramasys`'s own skill/doc structure (whatever that turns out
   to be — check before assuming a `bin/orama-system`-shaped layout exists
   in v2, since it may not) rather than copied verbatim with v1-specific
   paths intact.
7. **Do not port the `gh`-via-Telos question as unresolved** — §2's
   recommendation (don't route `gh` through Telos) should be a decided,
   documented choice in the ported code, not left as an open question for
   whoever reads it next.
8. **Falsify before trusting**, same as every guard added this session:
   for each ported guard, deliberately break it, confirm the test catches
   the break, restore, confirm green. Do not skip this step because the
   logic is "just a port" — a port can introduce a transcription error as
   easily as new code can introduce a design error.

## 6. What this plan deliberately does not do

- Does not design against `anamnesis` — too new, too undocumented to
  design against confidently (§3's tentative guard is named, not built).
- Does not eliminate `append-pr-body.sh`/body-replacement entirely — the
  `## Summary` convention genuinely needs it; comment-first is the new
  *default*, not the only path.
- Does not attempt to close the underlying PR-body race — that bound is
  accepted, honestly, the same way v1 accepted it (Rule 7).
- Does not assume a specific target directory/module layout in
  `oramasys/oramasys` beyond what's checked at execution time — sketched
  as "the repo's existing scripts convention," not invented here.

---

## Model governance

Model selection for work under this plan follows the provider-agnostic standard in
[`docs/standards/model-governance.md`](../standards/model-governance.md).
The harness selects the provider path: Cursor / Grok Bot (default `cursor-grok-4.6-medium`,
medium effort, fast off) or Anthropic (default Claude Sonnet 5.5 at medium effort).
Escalation needs an escalation token as defined there, and Fable also needs a budget cap.
This section replaced the earlier Claude-only governance text on 2026-10-03.
