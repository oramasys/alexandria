# alexandria

Central documentation hub for the `oramasys` org — **not yet in active use**.
This repository is scaffolded now so the destination exists, but no content
has been migrated into it. `orama-system/docs/v2/` remains the canonical,
actively-maintained specification tree until a real migration is explicitly
scheduled.

## What this repository is destined to become

A documentation-only, zero-code repository holding the specifications,
threat models, ADRs, and architectural decisions that today live scattered
across:

1. **Perpetua-Tools** (`docs/phase-0-specifications/`) — runtime-adjacent
   specs, correctly placed there for now.
2. **orama-system** (`docs/v2/`) — architectural planning docs (D1–D26+
   decisions, kernel spec, security model, migration plans). This is the
   primary intended source for a future migration into this repo.
3. Various per-repo `docs/v2/` folders across the `oramasys/*` org repos.

## Governance model (once active)

| Repository | Layer | Owns |
| --- | --- | --- |
| `alexandria` (this repo) | Documentation | Specs, threat models, ADRs, design rationale, cross-repo standards |
| `orama-system` | L3 — Methodology | Stateless planning framework, skills, gate criteria (until superseded) |
| `oramasys/oramasys` | Orchestration | Code, config, as-built docs adjacent to the implementation |
| `Perpetua-Tools` | L2 — Runtime | Implementation, orchestration, runtime-adjacent specs |

## Status

- **2026-09-01:** Repository created (public, MIT-licensed, matching the
  other `oramasys/*` repos). No content migrated.
- **2026-09-25:** First curated docs ingest under [`docs/`](docs/) — PT-agent
  migration architecture, governance plans, Anamnesis gap review, and master
  synthesis (docs-only; portable-memory scrubbed). See [`docs/README.md`](docs/README.md).
- **Not scheduled:** the full `docs/v2/` → `alexandria` migration. When it
  happens, `orama-system/docs/v2/` is expected to remain fully intact as
  historical source — this repo receives copies, not a destructive move.

## Background

This repository's purpose was first proposed in
`orama-system/docs/v2/41-alexandria-repository.md` (an ADR that predates
this repo's creation and was never acted on). That document is the fuller
rationale for why a separate documentation hub was considered; treat it as
background context, not as a commitment to the exact directory structure
it proposes — the real structure will be decided when migration actually
starts.
