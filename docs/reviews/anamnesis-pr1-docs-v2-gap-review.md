> **Alexandria ingest:** 2026-09-25 · docs-only review copy · portable-memory scrubbed.

# Anamnesis PR #1 vs Orama-System v2 Plans

## Recursive Gap Review and Recommended Next Steps

Date: 2026-09-21
Scope:

- PR: `oramasys/anamnesis#1`
- Feature branch: `feat/anamnesis-memory-building-blocks`
- Reference plan: `docs/v2/31-security-harness-excellence-plan.md`
- Reference review: `docs/security-reviews/2026-06-14-security-efficiency-review-v1.md`
- Broader recursive scope: `docs/v2/*`

## Executive Verdict

PR #1 is a clean and useful first slice of the planned memory architecture. It implements three real primitives:

1. append-only provenance/audit ledger
2. Phylax-owned fail-closed pre-write gate protocol
3. controlled candidate -> reviewed -> graduated promotion state machine

Those pieces align directly with the v2 security strategy around memory provenance, fail-closed admission, and evidence-backed promotion.

But PR #1 is not yet a security-harness implementation and should not be treated as fulfillment of the broader v2 security plan.

The largest gap is architectural rather than cosmetic:

> Anamnesis defines strong memory rules in isolation, but it does not yet demonstrate that those rules are enforced at the system boundary where real memory enters, leaves, gets retrieved, or becomes trusted agent context.

The PR therefore looks like a solid foundation PR, not a completed security milestone.

---

# 1. What PR #1 Actually Delivers

The PR adds seven files and 371 lines.

The stated scope is intentionally narrow:

- ledger
- pre-write gate protocol
- promotion pipeline
- boundaries
- tests
- license

The PR explicitly keeps security redaction in Phylax, runtime orchestration in perpetua-core, and transport in telos.

That boundary separation is good.

The tests cover:

- append-only ordering
- UTC timestamps
- fail-closed gate rejection
- gate transformation
- forward-only promotion
- graduation requiring `decision_ref`
- rejection as terminal

This matches the PR's stated intent.

---

# 2. What It Satisfies From the Security Harness Plan

## 2.1 Memory Provenance

### Status: PARTIALLY SATISFIED

The ledger records:

- `run_id`
- `lesson`
- `provenance_ref`
- `source`
- timestamp
- gate identity

That creates a basic provenance trail.

However, the v2 plan calls for richer memory governance, including:

- source
- trust tier
- checksum
- repo SHA
- author/principal
- task scope
- expiry
- retrieval audit data

PR #1 does not model those fields.

### Recommendation

Expand `MemoryEntry` around an explicit provenance object rather than accumulating optional fields directly.

Suggested shape:

```python
@dataclass(frozen=True, slots=True)
class Provenance:
    source: str
    principal: str
    repo: str | None
    repo_sha: str | None
    task_id: str
    trust_tier: str
    checksum: str
    captured_at: str
    expires_at: str | None
```

Then attach that object to the immutable memory record.

---

# 3. Critical Gap: Gate Is Optional

## Status: HIGH PRIORITY

The security plan says memory admission must be policy-mediated and fail-closed.

PR #1 allows:

```python
Ledger(gate=None)
```

and explicitly records entries with:

```text
accepted_by_gate = "none-configured"
```

That is useful for a library primitive.

It is not a secure production default.

The distinction needs to be explicit:

### Library mode

Gate may be omitted for unit-level use.

### Secure runtime mode

Gate must be mandatory.

Otherwise a caller can accidentally instantiate the ledger in insecure mode and still obtain apparently valid memory.

### Recommended change

Introduce two construction paths:

```python
Ledger.secure(gate=...)
Ledger.unrestricted_for_tests()
```

or enforce:

```python
Ledger(require_gate=True)
```

with a narrow test-only escape hatch.

Do not let production code silently default to `gate=None`.

---

# 4. Critical Gap: "Append-Only" Is Memory Semantics, Not Persistence Integrity

## Status: HIGH PRIORITY

The implementation uses:

```python
self._records: list[LedgerRecord]
```

This is an in-memory list.

That means:

- process restart loses the ledger
- there is no durable audit trail
- there is no tamper-evident storage
- there is no crash recovery
- there is no cross-process consistency
- append-only behavior is only enforced by API convention

The plan calls this a provenance/audit ledger.

A Python list is a useful prototype.

It is not yet an audit ledger.

### Recommended evolution

Make the persistence backend explicit:

```text
Ledger
  |
  +-- MemoryLedgerBackend
  +-- SQLiteLedgerBackend
  +-- PostgresLedgerBackend
```

Start with SQLite if local-first operation is the target.

Require:

- transactional append
- immutable record IDs
- unique provenance IDs
- monotonic sequence
- checksum or hash chain
- durable timestamps

---

# 5. Missing Tamper Evidence

## Status: HIGH PRIORITY

The plan repeatedly emphasizes auditability.

PR #1 records history but does not make history tamper-evident.

A malicious or buggy process with storage access could rewrite records.

Add:

```text
record_hash
previous_record_hash
```

Then derive:

```text
H_n = SHA256(
    canonical_record_n
    || H_(n-1)
)
```

This gives a lightweight append-only hash chain.

For stronger assurance, periodically anchor a digest externally.

Do not confuse this with cryptographic proof of truth. It proves history continuity, not content correctness.

---

# 6. Missing Memory Retrieval Boundary

## Status: CRITICAL ARCHITECTURAL GAP

PR #1 defines capture and promotion.

It does not define retrieval.

That leaves a major unanswered question:

> How does an agent obtain an Anamnesis memory record?

The security plan identifies:

```text
memory ingest
→ retrieval
→ model context
```

as a trust boundary.

PR #1 only implements the first portion.

The v2 architecture therefore needs:

```text
Capture
→ Admission
→ Provenance
→ Promotion
→ Retrieval
→ Context Injection
→ Audit
```

### Next component

Create a read-side policy layer:

```text
MemoryQuery
MemoryAuthorization
MemoryRetriever
RetrievalAudit
```

Every retrieval should produce an audit event.

---

# 7. Missing Trust Tiers

## Status: HIGH

The security plan calls for memory to remain untrusted until policy grants authority.

PR #1 has promotion states:

```text
candidate
reviewed
graduated
rejected
```

Those are lifecycle states.

They are not the same thing as trust levels.

Separate:

### Lifecycle

```text
candidate
reviewed
graduated
rejected
```

from:

### Trust

```text
untrusted
screened
trusted
restricted
expired
```

A memory can be:

```text
graduated + restricted
```

or:

```text
reviewed + untrusted
```

This distinction matters.

---

# 8. Missing Retrieval-Time Policy

Promotion should never become permanent permission.

A graduated memory may become invalid because:

- repository changed
- policy changed
- source was revoked
- model changed
- lesson became obsolete
- security classification changed

Add:

```text
expires_at
policy_version
scope
revoked_at
revocation_reason
```

and re-check policy at retrieval time.

---

# 9. Promotion State Machine Is Good but Incomplete

The current state machine is:

```text
candidate
   ↓
reviewed
   ↓
graduated

candidate/reviewed
   ↓
rejected
```

This is clean.

But the security plan eventually needs:

```text
candidate
   ↓
screened
   ↓
reviewed
   ↓
approved
   ↓
graduated
   ↓
validated
   ↓
expired/revoked
```

Do not implement all of that now.

First preserve the simple state machine.

Then define a versioned state schema so future states do not create a migration mess.

---

# 10. Missing Identity Model

The PR records `reviewer`.

It does not define what a reviewer identity means.

The system needs an explicit principal model:

```text
human
agent
service
policy-engine
system
```

Each decision should capture:

```text
principal_id
principal_type
authority
credential_ref
```

An agent calling itself `reviewer="operator"` is not an authenticated review.

---

# 11. Missing Decision Evidence Schema

The PR requires:

```text
decision_ref
```

This is good.

But a string reference alone does not establish evidence.

Define:

```text
DecisionRecord
```

with:

```text
decision_ref
decision_type
principal
timestamp
evidence_refs[]
policy_version
reason
```

Then `decision_ref` points to a structured artifact.

---

# 12. Phylax Boundary Needs an Executable Contract

The protocol abstraction is good:

```python
class PreWriteGate(Protocol):
    def gate(...)
```

But the security boundary is currently social rather than enforced.

The architecture needs a compatibility contract between Anamnesis and Phylax.

At minimum:

```text
Input
Output
Error semantics
Version
Policy identity
Decision metadata
```

Recommended:

```python
@dataclass(frozen=True)
class GateDecision:
    allowed: bool
    transformed_entry: MemoryEntry | None
    policy_id: str
    policy_version: str
    decision_ref: str
    reason_code: str
```

Then the gate is auditable.

---

# 13. Tests: Good Foundation, Too Narrow for the Plan

The PR has five tests and reports 5/5.

That proves the local state machine.

It does not prove the security architecture.

Add tests for:

### Admission

- missing gate
- gate timeout
- gate exception
- gate returns malformed result
- gate transforms content
- gate returns a different provenance ID

### Provenance

- duplicate provenance
- missing repo SHA
- checksum mismatch
- expired entry
- revoked source

### Promotion

- unknown entry
- duplicate submit
- rejected -> promotion
- graduated -> rejection
- missing reviewer
- missing reason
- missing decision reference

### Concurrency

- two simultaneous writes
- two simultaneous promotions
- duplicate provenance races

### Persistence

- restart
- crash during append
- transaction rollback
- corruption detection

---

# 14. Important Code Smell in Promotion

`advance()` contains a fallback construction before checking whether the provenance exists.

That is unnecessarily clever.

Replace with a direct lookup:

```python
record = self._state.get(provenance_ref)

if record is None:
    raise ValueError(...)
```

Then unpack it.

The state machine should be boring.

Boring state transitions are good security code.

---

# 15. Unused Ledger Dependency

`PromotionPipeline` receives a ledger and stores:

```python
self._ledger = ledger
```

but does not use it afterwards.

That creates a misleading dependency.

Either:

- use the ledger as the source of truth
- or remove the stored reference

Do not imply coupling that does not exist.

---

# 16. Thread Safety Is Asymmetric

`Ledger` uses a lock.

`PromotionPipeline` does not.

If concurrent agents can promote the same lesson, state transitions can race.

Add a lock around:

- submit
- advance
- reject
- state_of
- history

Or make promotion persistence transactional at the backend level.

---

# 17. Package and Build Completeness

The PR adds source under:

```text
src/anamnesis
```

but the reviewed diff does not show package metadata such as:

```text
pyproject.toml
```

For a standalone repository, ensure the repo has:

- package metadata
- supported Python version
- test dependencies
- lint configuration
- build configuration
- CI

---

# 18. CI Evidence

The PR reports:

```text
tests 5/5
```

but the architecture requires security controls to become executable gates.

Require CI for:

```text
pytest
ruff
mypy
pip-audit or uv audit
```

Later add:

```text
security regression suite
provenance tests
property tests
concurrency tests
```

---

# 19. Relationship to `31-security-harness-excellence-plan.md`

The plan's one-line objective is:

> policy-mediated agency: every agent action is attributable, bounded, replayable, and deny-by-default.

PR #1 currently covers only a fraction:

| Objective | PR #1 |
|---|---|
| Attributable memory | Partial |
| Bounded memory admission | Partial |
| Replayable history | Not yet |
| Deny-by-default | Gate optional |
| Tool controls | Not applicable |
| Filesystem controls | Not applicable |
| Network controls | Not applicable |
| Identity | Partial |
| Observability | Minimal |
| Supply chain | Not addressed |
| Agent cascade prevention | Not addressed |

The PR should therefore be described as:

> Anamnesis memory foundation

not:

> Security harness implementation complete

---

# 20. Relationship to the 2026-06-14 Security Review

The security review identified:

- default-open auth
- dependency pinning
- rate limiting
- testclient authentication boundary
- cookie security
- adaptive polling
- other system-level findings

PR #1 does not address those findings.

That is not a problem.

They belong to different layers.

But the project needs explicit cross-references so reviewers do not mistake the new memory repository for remediation of the existing security queue.

---

# 21. Recursive `docs/v2` Gap

The v2 strategy is broader than the two reference documents.

The security plan itself identifies a larger document chain, including:

```text
31  Security Harness Excellence
32  Agentic Security Controls
33  Source Material
34  Local Model Runtime Profile
39  MAESTRO / OWASP reference
46  Repository Standard
```

Anamnesis currently establishes boundaries compatible with that architecture.

What is still missing is the contract wiring between those layers.

Recommended dependency chain:

```text
Phylax
  ↓ admission/redaction
Anamnesis
  ↓ memory/provenance
Perpetua-core
  ↓ orchestration
Telos
  ↓ transport
Agate
  ↓ hardware/model affinity
AlphaClaw
  ↓ external adapter
Observability
```

Every boundary should have:

- API contract
- identity contract
- failure contract
- audit contract
- version

---

# 22. Recommended Revised Strategy

Do not make PR #1 bigger.

Keep it small.

Make it the first node of a deliberate sequence.

## PR #1 — Anamnesis Foundation

Keep:

- ledger
- promotion state machine
- boundary document
- tests

Add before merge:

- package metadata
- CI
- concurrency tests
- mandatory-vs-test gate distinction
- clearer error semantics

## PR #2 — Durable Provenance

Add:

- SQLite backend
- record IDs
- checksum
- hash chain
- transaction semantics
- restart tests

## PR #3 — Identity + Decision Evidence

Add:

- Principal
- DecisionRecord
- decision_ref resolver
- reviewer authority
- audit metadata

## PR #4 — Memory Retrieval Security

Add:

- MemoryQuery
- authorization
- trust tiers
- retrieval audit
- scope and expiry
- revocation

## PR #5 — Phylax Contract Integration

Implement the actual pre-write gate.

Prove:

```text
untrusted input
→ Phylax
→ Anamnesis
→ persisted record
```

## PR #6 — Agent-System Integration

Prove:

```text
capture
→ gate
→ ledger
→ review
→ promotion
→ retrieval
→ agent context
→ audit
```

## PR #7 — Adversarial Security Harness

Add:

- memory poisoning fixtures
- prompt injection fixtures
- provenance spoofing
- privilege escalation attempts
- concurrent promotion races
- malicious tool output
- stale-memory replay

---

# 23. Definition of Done for Anamnesis v1

```text
[ ] Durable append-only persistence
[ ] Tamper-evident provenance
[ ] Mandatory secure admission in production
[ ] Authenticated principal model
[ ] Structured decision evidence
[ ] Forward-only promotion
[ ] Trust tiers separate from lifecycle state
[ ] Retrieval authorization
[ ] Expiry/revocation
[ ] Retrieval audit
[ ] Concurrency safety
[ ] Crash/restart tests
[ ] CI security gates
[ ] Phylax integration
[ ] Orama/Perpetua integration
[ ] Adversarial memory tests
[ ] Documentation cross-references
```

---

# Final Assessment

PR #1 is a good foundation.

Its strongest design decision is the separation of concerns:

```text
Anamnesis = memory persistence + provenance + promotion
Phylax    = security admission + redaction
Perpetua  = runtime orchestration
Telos     = transport
Agate     = hardware/model affinity
```

Its biggest weakness is that the security properties currently exist mostly as interfaces and intentions.

The next step should not be "add everything."

The next step should be:

> Turn the Anamnesis interfaces into durable, authenticated, auditable, testable system boundaries.

That gives the v2 security program something more valuable than a large security document:

**an executable architecture.**

---

## Review Basis

- `oramasys/anamnesis#1`
- `orama-system/docs/v2/31-security-harness-excellence-plan.md`
- `orama-system/docs/security-reviews/2026-06-14-security-efficiency-review-v1.md`
- `orama-system/docs/v2/32-agentic-security-controls.md`
- `orama-system/docs/v2/33-security-harness-source-material.md`
- `orama-system/docs/v2/34-local-model-runtime-profile.md`
- `orama-system/docs/v2/39-maestro-owasp-genai-reference.md`
- `orama-system/docs/v2/46-repository-standard.md`
