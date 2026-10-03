# Alexandria documentation index

First content ingest: **2026-09-25** (task `mig-20260925-alexandria-ingest`).

This tree is documentation-only. It does not execute runtime logic.
`orama-system/docs/v2/` remains historical source authority until a fuller
migration is explicitly scheduled; Alexandria receives curated copies.

## Layout

| Path | Contents |
| --- | --- |
| [`architecture/`](architecture/) | Cross-repo migration architecture & lesson-capture contracts |
| [`plans/`](plans/) | Executable migration plans (not yet necessarily shipped) |
| [`reviews/`](reviews/) | Gap reviews and master synthesis |
| [`standards/`](standards/) | Cross-cutting standards (model governance) |
| [`notes/`](notes/) | Short factual status notes |

## 2026-09-25 ingest set

### Architecture

- [`architecture/2026-09-25-pt-agent-migration-and-lesson-capture.md`](architecture/2026-09-25-pt-agent-migration-and-lesson-capture.md) — PT `.agent` unbundling, agentic-stack overlay, lesson capture → `oramasys/*` v2

### Plans

- [`plans/v1-to-v2-pr-body-migration.md`](plans/v1-to-v2-pr-body-migration.md) — PR reporting tools → `oramasys/oramasys` (comment-first)
- [`plans/perpetua-core-health-probe-telos-migration.md`](plans/perpetua-core-health-probe-telos-migration.md) — SUPERSEDED by perpetua-core #7 (C0 `REJECT_CORE_TELOS_DEP`); historical `health_probe()` plan retained

### Standards

- [`standards/model-governance.md`](standards/model-governance.md) — provider-agnostic model governance: Cursor / Grok Bot path (default `cursor-grok-4.6-medium`), Anthropic path, escalation token (🟡 proposed)

### Reviews

- [`reviews/anamnesis-pr1-docs-v2-gap-review.md`](reviews/anamnesis-pr1-docs-v2-gap-review.md) — Anamnesis PR #1 vs v2 security-plan gaps
- [`reviews/master-review-and-synthesis.md`](reviews/master-review-and-synthesis.md) — unified ecosystem synthesis

### Notes

- [`notes/kungfu-local-wave0-status.md`](notes/kungfu-local-wave0-status.md) — local Wave 0 exists; GitHub remote was absent at pack authoring time

## Portable-memory invariant

Tracked docs name categories only. No personal home-directory paths, device
names, or local topology fragments. See orama-system
`docs/v2/47-portable-memory-local-topology-invariant.md`.
