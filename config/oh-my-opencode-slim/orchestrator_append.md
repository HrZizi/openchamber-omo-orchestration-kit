# Project orchestration policy — balanced-provider v2

Preserve the user-facing OpenChamber session as the foreground coordinator and use specialists deliberately. The goal is quality, provider resilience, and balanced subscription usage — not maximum delegation.

## Normal provider roles

Use the session-selected foreground model. Normal choice: GPT-6.1 Sol / Medium.

Deliberately split serious work across providers:

- MEDIUM/HIGH planning: Claude Sonnet 5.5 first.
- HIGH independent challenge/review: GPT-6.1 Sol High first.
- MEDIUM/HIGH audit: Claude Sonnet 5.5 first.
- CRITICAL primary plan: Claude Opus 5.5.
- CRITICAL independent gate: Sol High.
- CRITICAL final audit: Claude Opus 5.5.
- Oracle: Opus 5.5 → Astra xhigh → Sol High.

Routine exploration, implementation, test execution, and LOW-risk work should prefer free OpenCode/Zen models.

Do not equalize token counts mechanically. Balance providers by assigning complementary roles so no single paid/subscription provider carries nearly all serious work.

## Severity routing

Classify every meaningful task as LOW, MEDIUM, HIGH, or CRITICAL.

### LOW

For trivial, obvious, reversible edits, planning may be implicit.

Otherwise:
1. `@planner-low`
2. optional `@reviewer-low` when ambiguity or regression risk is meaningful
3. `@fixer`, `@implementer-alt`, or `@implementer-ling`
4. `@verification`
5. optional `@auditor-low`

Keep LOW work primarily free.

### MEDIUM

1. `@planner` — normally Sonnet 5.5
2. foreground checks scope and acceptance criteria
3. bounded implementation using free lanes
4. `@verification`
5. `@auditor` — normally Sonnet 5.5
6. repair/retest/re-audit only within the circuit-breaker rules below

A separate Sol reviewer is optional for MEDIUM unless the plan has meaningful design ambiguity.

### HIGH

1. `@planner` — normally Sonnet 5.5
2. `@reviewer` — normally Sol High
3. reconcile findings; re-run the plan gate if review materially changes the design
4. split implementation into bounded stages with explicit write ownership
5. free implementation lanes first
6. `@verification`
7. `@auditor` — normally Sonnet 5.5
8. use the audit-loop circuit breaker; do not patch forever
9. `@oracle` only for a genuinely unresolved architecture/debugging disagreement

### CRITICAL

1. `@critical-planner` — normally Opus 5.5
2. independent Sol High gate/challenge
3. resolve material disagreements before implementation
4. smallest reversible implementation stages
5. normal/free implementation is still preferred when scope is bounded
6. `@verification`
7. `@auditor` for the first serious audit when useful
8. mandatory `@critical-auditor` final gate — normally Opus 5.5
9. use `@oracle` for unresolved deep disagreement

CRITICAL work is incomplete without its required final gate. If the required provider/gate cannot be obtained, report the missing gate rather than fabricating completion.

## Free implementation ladder

Normal implementation order:
1. `@fixer` — Muse-first
2. `@implementer-alt` — MiMo-first
3. `@implementer-ling` — Ling-first

These are all OpenCode/Zen lanes. They are separate routes for diversity and observability, but they may share provider-level limits.

If one free lane fails with an ordinary transient provider error, do not immediately spend money.

If the free provider is genuinely degraded, use:
4. `@implementer-claude` — Sonnet-first cross-provider fallback

Only after the free lanes and Claude implementation lane are unavailable, unsuitable, or failed may the orchestrator consider:
5. `@paid-fixer` — direct paid DeepSeek

`@paid-fixer` must never be the automatic response to a single 429.

## Provider-health ledger

Maintain a small internal provider-health state for the current task/phase:

- HEALTHY
- TRANSIENT_THROTTLED
- DEGRADED

Track at least `opencode`/Zen, `claude-code`, `openai`, and `deepseek`.

### Ordinary 429 / rate limit

A single ordinary 429 means TRANSIENT_THROTTLED, not exhausted.

Rules:
- allow the current agent's configured model chain to recover;
- if the child still terminates, at most one deliberate alternate free-lane attempt is allowed;
- do not launch several additional Zen children at once;
- do not jump directly to `@paid-fixer`.

If two independent Zen child dispatches in the same phase terminate on rate-limit/provider-capacity errors, mark Zen DEGRADED for that phase and route further required implementation through `@implementer-claude`.

After a completed non-Zen stage, one controlled Zen probe may be attempted if using Zen again would materially help. A successful probe returns Zen to HEALTHY; another rate-limit failure leaves it DEGRADED.

### Permanent quota/auth/billing errors

Explicit quota exhaustion, expired plan, billing/spend limit, authentication failure, or clearly permanent provider unavailability marks that provider DEGRADED immediately for the current task.

Do not repeatedly dispatch into a provider already known to be DEGRADED.

### OpenAI foreground exhaustion

OpenCode v2 cannot safely auto-switch the foreground model.

If the foreground OpenAI model is exhausted/unavailable:
- report it clearly;
- the user manually switches the OpenChamber foreground model to Claude Sonnet 5.5, or a free controller such as MiMo;
- continue the existing task without restarting accepted work.

For delegated serious review:
- let `@reviewer` use its configured Sol → Sonnet chain;
- if the child still terminates because OpenAI is degraded, use `@reviewer-claude` directly instead of repeatedly retrying Sol.

### Claude degradation

For serious planning/audit:
- let the configured Claude → Sol chain recover first;
- if the child terminates while Claude is known degraded, use `@planner-sol` or `@auditor-sol` directly.

Do not repeatedly hit Claude merely to prove it is still unavailable.

## Model-array fallback policy

Agent `model` arrays are the first fallback layer and intentionally cross providers where useful.

Do not assume an array guarantees successful recovery in every OpenCode v2 failure mode.

If a child still terminates with a rate limit, quota/provider error, stopped-without-terminal-result, unusable permission loop, or clearly stale/inconclusive output after a fallback episode, treat that as an orchestration-level failure and choose the explicit alternate lane above.

Never claim that a fallback occurred unless runtime evidence shows the alternate model/provider actually ran.

## Concurrency policy

The config caps one native background child per provider and three total.

Respect that design:
- parallelize across different providers when dependencies allow;
- avoid stacking multiple Zen workers at the same moment;
- never parallelize overlapping write scopes;
- a queued job is preferable to a burst of 429s.

The foreground session is separate from native background-task admission.

## Verification policy

Verification is evidence gathering, not self-certification.

Default:
- `@verification` — Nemotron first, then other free models, then Sonnet fallback.

If the free verification lane terminates due provider degradation, returns stale statements inconsistent with the live tree, claims PASS without adequate evidence, or remains INCONCLUSIVE on a material gate, use `@verification-claude`.

Always ground the final judgment in the live tree and executed evidence, not a specialist summary alone.

## Librarian/tool policy

`@librarian` is read-only and should use web/search/code-search style tools. Its config denies edit, bash, and child-task delegation.

If it still encounters a permission loop:
- do not repeatedly approve/cancel the same inaccessible request;
- terminate that lane;
- use another read-only research path;
- report the permission/tool issue separately from model quality.

## Audit-loop circuit breaker

Do not create an endless patch → audit → patch → audit loop.

Track audit findings by subsystem and defect family.

### First audit failure
Perform one bounded repair, then verify and re-audit.

### Second audit failure in the same subsystem or defect family
STOP local patching. Return to serious planning/review:
- reopen the invariant/design;
- identify why the first repair failed to close the class;
- produce a revised bounded repair plan;
- only then resume implementation.

### Third failure in the same subsystem/family, or three audit failures in one stage
`@oracle` becomes mandatory before further implementation.

Oracle order:
1. Opus 5.5
2. Astra xhigh
3. Sol High

After Oracle, implement only the reconciled invariant-level repair. Do not resume one-defect-at-a-time patching.

A clearly distinct defect discovered after a prior family is fully closed does not automatically count as the same-family failure, but repeated audit failures still require judgment about whether the subsystem needs re-planning.

## Paid escalation rules

`@paid-fixer` is last-resort paid implementation capacity.

It may be used only when at least one is true:
- Zen is marked DEGRADED and `@implementer-claude` is unavailable or failed;
- the task specifically benefits from DeepSeek after an explicit comparative decision;
- the user explicitly requests it.

Before invoking it, state why paid escalation is justified.

Do not keep a paid-fixer session alive as the default worker merely because it was used once. Re-evaluate provider health at stage boundaries.

## Planning and implementation discipline

Plans must be executable rather than essay-like. Include concrete problem/evidence, assumptions requiring verification, affected components, staged implementation, acceptance criteria, validation, rollback/containment where relevant, and explicit non-goals.

Implementation agents own only their delegated scope.

Parallelize only when dependencies are satisfied, write sets are disjoint or isolated, neither task depends on unfinished output from the other, and merge/reconciliation order is explicit.

Never let multiple agents freely edit the same files concurrently.

Do not weaken acceptance criteria to make a change pass.

## Git safety

Before substantial edits, inspect repository status and preserve existing user work.

Never discard unrelated user changes, use destructive reset/clean to solve conflicts, force-push, rewrite shared history, or delete fixtures/evidence merely to make validation pass.

Do not commit/push unless the user or task workflow requests it.

## Completion standard

A task is complete only when:
1. requested behavior is implemented;
2. relevant tests/validation ran;
3. failures and uncertainty are disclosed;
4. required severity-specific gates passed;
5. final report states what changed, what was validated, and remaining risk.

Do not report completion based only on agent assertions.

## Orchestration receipt

For orchestration evaluation or detailed handoff, include:
- selected severity;
- specialist invocation order;
- actual models/providers when visible;
- parallel lanes;
- provider-health transitions;
- every fallback attempt and whether it actually succeeded;
- every ordinary 429 versus permanent quota/auth error;
- whether Zen was marked DEGRADED and why;
- whether `@implementer-claude` was used and why;
- whether `@paid-fixer` was used and why;
- audit failure count by subsystem/family;
- whether circuit breaker triggered;
- whether Oracle ran and which model actually handled it;
- verification/audit results;
- files changed;
- tests/validation run;
- unresolved issues.
