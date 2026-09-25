> **Alexandria ingest:** 2026-09-25 · docs-only plan copy · portable-memory scrubbed.

# perpetua-core: Route `health_probe()` Through Telos

**Status:** plan, about to be executed immediately after this document.
**Scope:** `oramasys/perpetua-core` only. No other repo's code changes.
**Repo state confirmed before writing this plan:** cloned and read directly,
not assumed — `perpetua-core` has zero open PRs; `oramasys/oramasys` imports
`BackendRegistry`/`select_backend` from it but never calls the vulnerable
method directly (traced by grep across both repos' actual source).

## 1. The finding this closes

`src/perpetua_core/discovery/probe.py`'s `health_probe()` makes a raw
`httpx.AsyncClient().get(url)` call with no endpoint validation, no SSRF
checks, no DNS-rebinding defense — none of the protections Telos exists to
provide, and the exact class of gap the restoration coordination board
explicitly flagged as needing an individual audit ("remaining direct
curl/urllib/httpx/socket paths").

Two callers:

- `BackendRegistry.autodetect()` — a hardcoded seed list (`localhost:11434`,
  `localhost:1234`, one hardcoded LAN IP). Lower risk today, still
  unprotected in principle.
- `BackendRegistry.register_by_ip(ip, port, ...)` — accepts an **arbitrary
  caller-supplied IP and port**, builds a URL directly, probes with zero
  validation. A textbook SSRF shape. No current caller found in either
  `perpetua-core` or `oramasys/oramasys`'s own code, but it's public API
  waiting for one.

## 2. Design decision, stated explicitly (not left implicit)

Telos's existing authorization model is an **exact allowlist**
(`EndpointAuthorizer.from_exact_rules`) — a fixed, pre-known set of
`(scheme, host, port)` tuples per purpose. That model doesn't fit discovery
directly: backends are found dynamically (`autodetect()`'s seeds can
change; `register_by_ip()` is inherently ad hoc), so there is no fixed
allowlist to check against in advance.

**Decision:** for the `HEALTH_PROBE` purpose specifically (a purpose that
already exists in Telos's taxonomy), semantic authorization is permissive
by design — any endpoint may be *probed* for health. The real security
boundary is **transport safety, not semantic allowlisting**: the transport
policy passed for this call sets `allow_public=False` unconditionally,
`allow_private=True`, `allow_loopback=True`. A health probe can reach
loopback or private/LAN addresses only, never a public/internet address,
regardless of what `ip` a caller supplies to `register_by_ip()`. This
matches Telos's own stated philosophy that transport safety and semantic
authorization are separate, composable decisions neither of which
substitutes for the other — here, transport safety alone is the
sufficient, correct gate for this specific purpose, and semantic
allowlisting would just be redundant ceremony for a "can I check if
something is alive" operation.

This is a genuine design choice with a real alternative (per-call dynamic
allowlists threaded through from the caller) that was more complex for no
added safety here, since the transport-level restriction already prevents
the one thing that matters — reaching outside the local network.

## 3. Implementation

1. **Add `telos` as a `perpetua-core` dependency** in `pyproject.toml`
   (same-language, same-org package — matches how `oramasys/oramasys`
   consumes Telos as a direct Python dependency for its Gateway dialer,
   not the JSONL bridge, which exists for non-Python consumers only).

2. **Rewrite `health_probe()`** to call `telos.transport.request()`
   instead of raw `httpx`:
   - `purpose=EndpointPurpose.HEALTH_PROBE`
   - `authorizer`: a permissive authorizer scoped to this purpose only
     (`EndpointAuthorizer` subclass or a purpose-scoped always-allow rule —
     exact shape decided during implementation, kept as small and
     auditable as the existing `_HostAllowlistAuthorizer` pattern in
     `telos/bridge.py`)
   - `transport_policy=TransportPolicy(allow_public=False,
     allow_private=True, allow_loopback=True, require_https_for_public=True)`
   - Preserve the existing return shape (`ProbeResult` with
     `BackendHealth`/`models`) and the existing offline/degraded
     classification on connection failure or malformed JSON — this is a
     transport-layer swap, not a behavior change to callers.

3. **No changes to `registry.py`, `backend.py`, or any caller** — both
   `autodetect()` and `register_by_ip()` call `health_probe()` unchanged;
   the fix is fully contained in `probe.py`.

4. **Tests**: real regression tests, not just type-checking —
   - A genuinely public address is rejected before any request is sent
     (proves the SSRF gate, matching the pattern already used for
     Telos's own DNS-rebinding and credential-header tests this session).
   - Loopback and private/LAN addresses still succeed (proves the fix
     isn't over-broad — `autodetect()`'s existing seed list must keep
     working).
   - Existing offline/degraded classification behavior is unchanged for
     connection failures and malformed JSON.

5. **Verify**: run the full existing `perpetua-core` test suite (not just
   the new tests) to catch any regression in `discovery/` or elsewhere;
   confirm real CI green on the pushed commit before opening a PR.

## 4. What this does not do

- Does not touch `register_by_ip()`'s own signature or add caller-side
  input validation beyond what the transport-level fix already provides —
  the transport fix is the validation.
- Does not add a new Telos purpose or change Telos's own package at all;
  everything here is `perpetua-core`-side.
- Does not merge — same rule as every other PR this session. Opens a PR,
  reports exact head/CI/review-thread status, waits for explicit
  authorization.

---

## Claude Model Governance & Execution Routing (2026-09-25 Standard)

All execution, evaluation, synthesis, and verification workflows under this plan adhere to the canonical Anthropic Claude routing hierarchy:

- **Baseline / Default Execution:** **Claude Sonnet 5 (Medium effort)** (`claude-sonnet-5`). Standard everyday planning, linting, review, code modification, and automated verification tasks MUST dispatch to Sonnet 5 Medium by default.
- **Escalation Path A — Deep Reasoning / Complex Architecture:** **Claude Opus 5.5 (High effort)** (`claude-opus-5-5`). Permitted **ONLY** under an explicit human-initiated request or an authenticated, human-gated / human-approved agentic escalation token (e.g. following a documented evaluation or confidence failure in Sonnet 5).
- **Escalation Path B — Long-Horizon Agentic Hard Problems:** **Claude Fable 5.1** (`claude-fable-5-1`). Permitted **ONLY** under direct human command or human-approved escalation, and strictly when runtime budget guards allow and account usage credits are enabled.
