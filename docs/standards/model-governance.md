# Model governance and execution routing (provider-agnostic)

> **Status: 🟡 Proposed.** It becomes 🟢 once agent launches actually comply (see [Compliance](#compliance-and-status)).
> Vendor facts last checked against vendor docs and the Cursor Cloud Agents model catalog on **2026-10-03**.
> This is the one source for model selection rules in the oramasys constellation.
> Plans and reviews link here and do not restate model ids.
> Reconciled with [orama-system PR #381](https://github.com/diazMelgarejo/orama-system/pull/381) on **2026-10-24**: outdated Cursor ids refreshed, the [relationship to repository routing policies](#relationship-to-repository-routing-policies) stated, and [escalation approval regimes](#escalation-approval-regimes) added.

## Precedence

1. The operator's (Seth's) explicit instruction wins. His standing hard rules are invariants.
2. The harness decides the provider path:
   - Cursor (IDE, CLI, web, Cloud Agents, SDK) or Grok Bot → **Cursor / Grok Bot path**.
   - Claude Code, the Claude API or SDK, or claude.ai → **Anthropic path**.
   - Runtime code calling `api.x.ai` directly → **direct-xAI path** (optional).
3. A model picked inside a harness keeps that model's own gate. For example, Claude Fable 5.1 picked inside Cursor still needs the Fable escalation token and budget cap.
4. Agents never escalate, raise effort, turn on fast mode, or switch providers on their own.
5. If the path is unclear, use that harness's default, do not escalate, and report the ambiguity.

## Relationship to repository routing policies

Operator ruling, 2026-10-24: this standard **supplements** each repository's existing routing policy. It never supplants it.

- A repository's own tier policy stays the primary rule for choosing a model tier by complexity, risk, and budget. In orama-system that is [`.cursor/commands/model-route.md`](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/.cursor/commands/model-route.md), [`.cursor/rules/common-performance.mdc`](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/.cursor/rules/common-performance.mdc), the MODE 3 tiering in [`claude-code-workflow-canonical.md`](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/bin/orama-system/references/claude-code-workflow-canonical.md), and the tiers in the [cursor-agent skill](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/bin/orama-system/skills/cursor-agent/SKILL.md).
- This standard adds what those policies do not carry: the **fallback** when the tier logic fails or has no verifiable criteria (the harness default or an allowed alternative below, never a self-escalation), the **alternate paths** for harnesses other than the one a policy was written for, and the **gates** on escalated models and settings.
- When a vendor ships a newer model, downstream policies refresh the outdated ids in place. They do not rewrite the policy text around them.
- Downstream pin: orama-system [`docs/standards/model-governance.md`](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/docs/standards/model-governance.md), with the machine-readable [`config/model-governance.yml`](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/config/model-governance.yml) and checker [`src/orama_system/model_governance.py`](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/src/orama_system/model_governance.py).

This standard governs which model an **agent harness** runs. It does not reorder the runtime frugality ladder (local tiers first, paid tiers last) in [`../reviews/master-review-and-synthesis.md` §6](../reviews/master-review-and-synthesis.md#6-frugality-engine--ai-model-routing). Both apply.

## Cursor / Grok Bot path

**Rule: every Cursor model runs at medium effort with fast off**, unless the operator explicitly asks for something else. Set both explicitly on every launch. Cursor's own defaults differ: Grok models have defaulted to high effort (checked for Grok 4.6), and fast is the default speed tier on Pro and above. Re-check exact launch ids with `cursor-agent models`, because Cursor renames them between releases.

Cursor usage labels (what the usage dashboard shows) are not always launch ids (what a Cloud Agents launch passes). Map them as follows:

| Usage label | Cloud Agents launch id | Launch parameters | Status and gate |
| --- | --- | --- | --- |
| `cursor-grok-4.7-medium` | `grok-4.7` | `{effort: medium, fast: false}` | **Default** for agents and coding. No gate. (Refreshed 2026-10-24 from `grok-4.6`.) |
| `composer-2.5` | `composer-2.5` | `{fast: false}` (no effort parameter exists) | Allowed alternative. No gate. |
| `composer-2.5-fast` | `composer-2.5` | `{fast: true}` | Only on the operator's explicit request. |
| `gpt-5.6-terra` | `gpt-5.6-terra` | `{effort: medium, fast: false}` | Allowed alternative on the operator's explicit pick. No gate. |
| `grok-4.7` | `grok-4.7` | must be set as in the default row | The label has no effort suffix, so the effort is unconfirmed. Review the run. |
| `cursor-grok-4.6-medium` | `grok-4.6` | none | **Outdated.** Superseded by `grok-4.7` on 2026-10-24. Never launch or default to `grok-4.6`. |
| `auto` | `default` or `auto` | none | **Editor only**, for interactive work. Never for agent work. |
| `cursor-grok-4.5-medium` | `grok-4.5` | none | **Banned.** Historical label only. Never launch or default to `grok-4.5`. |
| `grok-bot-default`, `grok-bot-automation` | none | none | Grok Bot usage labels, **not launchable ids**. See the Grok Bot note. |

Notes:

- **Auto.** The router's pool includes Claude Opus and Claude Fable models, and the routed model is hidden by default. That bypasses the gates below, so agents never launch with `default` or `auto`.
- **Claude models inside Cursor** follow the Anthropic-path gates with Cursor's parameter names:
  - `claude-sonnet-5-5` with `{reasoning_effort: medium}` (no fast option). Allowed.
  - `claude-opus-5-5` with `{effort: high, fast: false}`. Escalation token required.
  - `claude-fable-5-1` with `{effort: medium}` unless the approval names another level (no fast option). Escalation token and budget cap required.
- **Other Cursor models.** Only on the operator's explicit pick, at medium effort with fast off.
- **Grok Bot.** Grok Bot's own model is managed by the product and has no picker. Its usage shows `grok-bot-default` and `grok-bot-automation`. These rules govern what Grok Bot launches. Cloud agents it starts use the default row above.

## Anthropic path

| Role | Model (API id) | Effort | Gate |
| --- | --- | --- | --- |
| Default | Claude Sonnet 5.5 (`claude-sonnet-5-5`). `claude-sonnet-5` is allowed as a legacy pin. | `medium`, set explicitly (vendor default is `high`) | none |
| Escalation A | Claude Opus 5.5 (`claude-opus-5-5`) | `high` (vendor default is `medium`) | [Escalation token](#escalation-token) |
| Escalation B | Claude Fable 5.1 (`claude-fable-5-1`) | `medium` unless the approval names another level | [Escalation token](#escalation-token) **and** a budget cap |

- **Claude Code.** `default` resolves to Opus 5.5, so set the Sonnet 5.5 default explicitly. Use the managed settings `availableModels`, `deniedModels`, and `maxEffortLevel`. Keep Fable in `deniedModels` unless an escalation token covers the task.
- **Fable budget cap.**
  - On the API, set a hard `max_tokens` plus `output_config.task_budget`. `task_budget` is a beta (`task-budgets-2026-03-13`) and only advisory, so `max_tokens` is the hard stop.
  - In runtime code, the cost guard fails closed when the cap is hit or cannot be evaluated. There is no silent reroute.
  - On claude.ai and Claude Code plans, Fable draws on the plan's usage credits. Confirm the remaining allowance before starting.
  - Headless `claude -p` and Agent SDK runs bill Fable credits without a consent prompt, so run Fable interactively only.
  - The escalation approval states the cap. A "4x Fable budget" multiplier is a hypothesis to measure, not a default.
- **Also gated.** `max` effort, fast mode, and any model outside this table need an escalation token.

## Direct-xAI path (optional)

Runtime code may call xAI directly with `grok-4.6` and `reasoning_effort: "medium"`, set explicitly (the vendor default is `high`). The Cursor path superseded `grok-4.6` with `grok-4.7` on 2026-10-24. Refresh this pin the same way once the xAI model list and pricing confirm `grok-4.7` for direct calls; absence from a search is not evidence either way.

- The 199k-token cliff guard in the master review §6 applies.
- The runtime frugality ladder's budget approval for paid tiers applies.
- Keys come from the environment or a secret store, never from config or docs.
- `grok-4.5` is banned here too.

## Escalation token

"Escalation token" is OramaSys shorthand, not vendor terminology. An escalation token is **both** of the following, and neither half is enough alone.

1. **Proven control-plane authentication: the S-AuthZ bearer.**
   - The control plane binds loopback-first, and the bearer is still required on every protected call. Loopback is the default bind, not an auth bypass. The insecure-dev skip never counts as proof.
   - Optional identity providers (Google OIDC, X, BUZZ NIP-98) may add identity. They never replace or disable local bearer auth.
2. **An explicit human-in-the-loop (HITL) escalation approval, shown by a fresh proof.**
   - The approval names the model id, effort and fast settings, the task (PR or issue), the budget cap (required for Fable), and an expiry.
   - The proof must be one of:
     - **A GitHub signing sign-off on the operator's device:** a GPG- or SSH-signed commit or tag that GitHub shows as *Verified* and that carries the approval. The git author, committer, or a `Signed-off-by:` line alone proves nothing.
     - **An offline GPG/PGP signature** over the approval text, made with the operator's key. This is sufficient by itself and needs no network identity provider.
     - **Any other online signing mechanism coupled to the control plane**, for example a BUZZ NIP-98 signed event or a passkey (WebAuthn) approval through the security-sentinel orbit once that exists.
   - OIDC is optional. It is never required for model escalation.

### What does not count

- An approval cached from an earlier session or task, or one past its expiry. Expiry is never later than 24 hours, in line with the 24-hour rule in the master review §2.1.
- A config flag such as `human_approved: true`, an environment variable, or an acknowledgement file an agent can write.
- A same-user keychain HMAC grant on its own. That is escalation control, not proof that a human was present.
- A signature made with a key that agent sessions can use without the operator's presence (passphrase or hardware touch).
- Anything an agent issues to itself.

**Record.** Post the approval text and a pointer to its proof (signed commit SHA, signature file, or event id) on the PR or issue. Never post the bearer or any key material.

**Until enforcement lands.** Until the control plane checks escalation tokens automatically, the interim approval in [Escalation approval regimes](#escalation-approval-regimes) applies: agents refuse an escalated model or setting unless that approval is recorded, and they report the refusal.

## Escalation approval regimes

Operator ruling, 2026-10-24. A proof cannot be demanded where nothing can issue or verify it, so the approval an escalation needs depends on whether the **escalation grant mechanism** exists. It exists only when both of these hold, and both can be checked in the repository that launches the agent:

1. The launch gate has a production caller at the launch boundary (in orama-system, `assert_launch_allowed`; today it has none).
2. The control plane verifies the S-AuthZ bearer and the signed HITL approval described in [Escalation token](#escalation-token).

| Regime | When | What approves Opus, Fable, or another escalated setting |
| --- | --- | --- |
| 1, interim | Either condition is unmet (**current state**) | An explicit operator confirmation in the session (for example an `AskUserQuestion` answer), recorded on the PR or issue with the model, effort, fast setting, task, and (for Fable) the budget cap. It is a human decision, not a token, and an agent never supplies it for itself. |
| 2, enforced | Both conditions are met | The escalation token, verified by the control plane. A chat confirmation alone no longer approves. |

The switch between regimes is a code change (a production caller and a verifier), not a config flag, environment variable, or note, so an agent cannot flip it. Neither regime lets an agent escalate on its own. The orama-system pin states the same rule in [Escalation approval regimes](https://github.com/diazMelgarejo/orama-system/blob/160ec03697bbd63e9673a1ba6fe3431e70e56bcf/docs/standards/model-governance.md#escalation-approval-regimes).

## Shared invariants

1. Every run has a hard per-request cap and, where the vendor supports one, a per-task budget. Delegations state their budget (tokens, tool calls, wall time).
2. Report the model and settings that actually served the work. Failover and safety fallbacks can change them.
3. No secrets, keys, account ids, or personal paths in docs, prompts, or agent configs.
4. Model ids come only from vendor docs or the vendor's model-list API, with a last-checked date.
5. Update this one file when a vendor launches, deprecates, or reprices a model.

## Compliance and status

The status moves from 🟡 to 🟢 only when all of the following hold:

- Agent launches from Cursor and Grok Bot pass model and parameters explicitly.
- The usage dashboard shows only allowed labels (`cursor-grok-4.7-medium`, `composer-2.5`, `gpt-5.6-terra` on an operator pick, or labels covered by an approval under the current [regime](#escalation-approval-regimes)).
- Claude Code managed settings carry the defaults and denials above.

## Prior decisions this standard follows

Repository-relative paths:

- Perpetua-Tools `.agent/memory/semantic/DECISIONS.md`:
  - A same-user keychain HMAC is escalation control, not proof of human presence. The human override is an operator grant, not environment exports.
  - Git author identity is not a reliable human-versus-agent signal.
- Perpetua-Tools `docs/HUMAN-IN-LOOP-ACCOUNTABILITY.md` (copy in orama-system `docs/v2/references/`):
  - No cached approvals.
  - A config flag is not approval.
  - Tier 1 uses GPG/PGP signed intent.
  - Tier 2 OIDC is reserved for regulated, high-stakes decisions.
- Perpetua-Tools `.agent/protocols/delegation.md`: every delegation carries a budget.
- orama-system `docs/v2/00-context-and-decisions.md`:
  - Loopback bind and no universal default tokens are the baseline.
  - The cost gate fails closed, and cloud escalation is empty by default.
- orama-system `docs/v2/30-multi-llm-router-caching-batching-decorator.md`: escalation and budget rules live in the cost guard, cloud escalation is default-deny, and the budget gate fails closed.
- orama-system `docs/v2/37-manifest-cost-tiering-pattern.md`: "4x Fable budget" is a hypothesis, not a default ceiling.
- orama-system `docs/v2/49-peer-mesh-auth-tls-v2-plan.md`: the bearer is always registered and always last, and the mesh never stops working because of an auth change.
- orama-system `docs/v2/51-security-sentinel-orbit-passkey-mcp.md`: passkey (WebAuthn) human approval bound to the action.
- orama-system `docs/v2/70-portal-knowledge-hitl-development-ladder.md`: the bearer is required with no loopback exemption.
- orama-system [PR #381](https://github.com/diazMelgarejo/orama-system/pull/381): supplement, never supplant, repository routing policies; refresh outdated ids only; escalation approval regimes (operator rulings, 2026-10-24).
- oramasys/oramasys `README.md` (S-AuthZ section): the bearer is required on protected routes, the default bind is loopback, and optional providers are off by default and never replace local secrets.

## Downstream decisions to ratify here

Recorded in orama-system [PR #381](https://github.com/diazMelgarejo/orama-system/pull/381) and listed for the operator to ratify into this standard. They are not yet part of it.

- **Effort ceiling.** `xhigh` and `max` are refused on every harness path, even with matching approval metadata (orama pin, "Effort ceiling").
- **Planning Opus.** A harness-classified planning launch may use Opus 5.5 at medium or high without a per-use approval; non-planning Opus keeps the approval, with medium as its candidate default (orama pin, "What orama must follow").
- **Role selection.** `plan_launch` returns the requested launch, the immediately permitted fallback, and an approval candidate; it never executes a gated candidate (orama pin, "Role selection contract").
- **Decision record.** Perpetua-Tools `.agent/memory` keeps the reasoning behind these rulings ([PR #461](https://github.com/diazMelgarejo/Perpetua-Tools/pull/461)).

## Vendor sources (checked 2026-10-03)

- Anthropic models overview: <https://docs.anthropic.com/en/docs/about-claude/models/overview>
- Anthropic deprecations: <https://docs.anthropic.com/en/docs/about-claude/model-deprecations>
- Anthropic effort: <https://docs.anthropic.com/en/docs/build-with-claude/effort>
- Anthropic task budgets: <https://docs.anthropic.com/en/docs/build-with-claude/task-budgets>
- Claude Code model configuration: <https://docs.anthropic.com/en/docs/claude-code/model-config>
- Claude Fable plan billing: <https://support.claude.com/en/articles/15424964-claude-fable-models-on-your-plan>
- xAI models: <https://docs.x.ai/docs/models>
- xAI reasoning: <https://docs.x.ai/docs/guides/reasoning>
- Cursor models and pricing: <https://cursor.com/docs/models>
- Cursor Grok 4.6: <https://cursor.com/docs/models/grok-4-6>
- Cursor Composer 2.5: <https://cursor.com/docs/models/cursor-composer-2-5>
- Cursor Router (Auto): <https://cursor.com/docs/cursor-router>
- Cursor Cloud Agents API: <https://cursor.com/docs/cloud-agent/api/overview>
- Grok Bot for teams: <https://docs.x.ai/grok-bot/teams-and-enterprises>
