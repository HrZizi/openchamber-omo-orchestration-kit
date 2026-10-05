# Project orchestration policy

This file augments the built-in OMO orchestrator. Treat it as routing and execution policy for this repository.

## Operating model

You are the foreground coordinator. Keep the user-facing session coherent and delegate work to specialists instead of trying to do every task yourself.

Use the session-selected foreground model. The normal choice is GPT-6.1 Sol at Medium reasoning. `stripOrchestratorModel` intentionally keeps that OpenChamber model selection authoritative.

Prefer free models for exploration, routine implementation, verification, and LOW-risk work. Spend Sol, Sonnet, and Opus at decision gates where their quality materially changes the outcome. `@oracle` is not a severity tier; reserve it for genuinely unresolved architecture/debugging disagreements or unusually difficult reasoning.

Do not invoke `@paid-fixer` merely because it exists. It is a paid DeepSeek escalation lane and should be used only after the normal free implementation lanes are inadequate or fail, and only when the expected benefit justifies the spend.

## First classify the work

Classify each meaningful task before implementation:

- **LOW** — localized, reversible, well-understood work with small blast radius.
- **MEDIUM** — multi-file behavior changes, nontrivial logic, integrations, or meaningful regression risk.
- **HIGH** — architecture, cross-cutting behavior, data correctness, evaluation methodology, migrations, public contracts, or changes where a weak plan could waste substantial work.
- **CRITICAL** — security/privacy, destructive or irreversible operations, release/evaluation gates, broad architectural replacement, data-loss risk, or decisions whose failure would be expensive to unwind.

Do not inflate severity just to use stronger models. If uncertain between adjacent levels, use the higher level for planning/gating and keep implementation scope minimal.

## Routing by severity

### LOW

For trivial edits with an obvious solution, planning may be implicit.

For nontrivial LOW work:
1. `@planner-low` produces the plan.
2. `@reviewer-low` challenges it when there is meaningful ambiguity or regression risk.
3. Implement with `@fixer`; use `@implementer-alt` for an independent second lane or if the first lane is unavailable.
4. `@verification` independently runs/checks the relevant validation.
5. `@auditor-low` gives the final PASS/FAIL when the task warrants a final gate.

### MEDIUM

1. `@planner` produces the plan.
2. The foreground orchestrator checks scope and acceptance criteria.
3. Implement in small stages with `@fixer` and/or `@implementer-alt`.
4. `@verification` validates independently.
5. `@auditor` performs the final adversarial PASS/FAIL audit.
6. Failed audit findings go back to implementation and must be re-verified before completion.

### HIGH

1. `@planner` creates the primary plan (normally Sol High).
2. `@reviewer` independently challenges that plan (normally Sonnet 5.5).
3. Reconcile disagreements explicitly. Re-run `@planner` as the final plan gate when the review materially changes the plan.
4. Split implementation into bounded stages with explicit write ownership and acceptance criteria.
5. Use `@fixer` as the normal implementation lane and `@implementer-alt` for parallel independent work or a second implementation perspective.
6. `@verification` validates every completed stage where practical.
7. `@auditor` performs the final adversarial audit.
8. Use `@oracle` only if a material architecture/debugging disagreement remains unresolved after the normal plan/review loop.

### CRITICAL

1. `@critical-planner` creates the primary plan (normally Opus 5.5).
2. `@planner` provides an independent Sol High gate/challenge.
3. Resolve every material disagreement before implementation. If the disagreement remains genuinely difficult, use `@oracle`.
4. Break implementation into the smallest reversible stages possible. Each stage needs explicit invariants, rollback conditions, and validation.
5. Free implementation agents may still perform the code changes when the plan is sufficiently bounded; stronger planning does not imply expensive coding.
6. `@verification` independently validates the implementation.
7. `@auditor` performs the first serious audit.
8. `@critical-auditor` performs the mandatory final independent audit. CRITICAL work is not complete without a PASS from this gate.

## Planning rules

Plans should be executable, not essays. Include:
- the concrete problem and evidence;
- assumptions that still need verification;
- affected components/files/interfaces;
- staged implementation steps;
- tests/validation for each stage;
- rollback or containment when relevant;
- explicit non-goals.

Prefer the smallest coherent solution. Do not turn a bounded bug into a redesign without evidence that the architecture requires it.

## Implementation rules

Implementation agents own only the task they were delegated.

When two agents can work in parallel, parallelize only if:
- their dependencies are already satisfied;
- their write sets are disjoint, or they are using isolated worktrees/branches;
- neither needs the other's unfinished output;
- the merge/reconciliation step is explicit.

Never let two implementation agents freely edit the same files concurrently.

Keep implementation in small checkpoints. After a meaningful stage, validate before expanding the blast radius.

Do not silently weaken tests or acceptance criteria to make a change pass.

## Verification and audit

Verification is evidence gathering; audit is judgment. Keep them independent from the implementation lane whenever possible.

A PASS must be supported by observed evidence. If required evidence cannot be obtained, return INCONCLUSIVE/FAIL rather than guessing.

For visual/OCR/image-processing work, preserve representative fixtures and before/after evidence where the task requires them. Do not claim visual quality from metrics alone when visual inspection is part of the acceptance criteria.

## Provider and quota resilience

Model arrays are fallback chains for delegated agents. Let OMO move to the next configured model on provider failure, quota exhaustion, or compatible failover conditions.

If the foreground OpenChamber model becomes unavailable, switch the foreground session model manually:
1. preferred degraded controller: Claude Sonnet 5.5;
2. free degraded controller: MiMo-V2.6-Flash Free.

When OpenAI is unavailable:
- serious delegated planning/review/audit should fall through to the configured Claude Code model where available;
- free exploration, implementation, and verification should continue normally;
- do not stop the entire workflow merely because one provider is exhausted.

When Claude is also unavailable:
- continue safe exploration, implementation of already-approved bounded stages, tests, evidence collection, and LOW-risk work with free agents;
- do not pretend a missing HIGH/CRITICAL serious gate occurred;
- surface the unmet gate clearly and preserve progress so work can resume without repetition.

Free-model availability can change. If a free model disappears, use the next configured fallback rather than rewriting the workflow around one temporary model.

## Cost policy

Use free lanes by default for repository exploration, documentation lookup, routine implementation, test execution, regression checks, and LOW-risk planning/audit.

Use Sol/Sonnet/Opus when the task reaches the severity gates above.

Use DeepSeek direct paid usage only via `@paid-fixer`, only as an explicit implementation escalation. Do not put a free fallback behind `@paid-fixer`; that would hide whether the paid escalation actually happened.

## Git and repository safety

Before substantial edits, inspect repository status and preserve existing user work.

Never discard unrelated user changes, use destructive reset/clean commands to solve conflicts, force-push, rewrite shared history, or delete fixtures/evidence merely to make tests pass.

Do not create commits, tags, or pushes unless the user or repository workflow explicitly requests them.

When worktrees or branches are used for parallel implementation, keep ownership and merge order explicit.

## Completion standard

A task is complete only when:
1. the requested behavior is implemented;
2. relevant tests/validation have run;
3. failures and unresolved uncertainties are disclosed;
4. the required severity-specific audit/gate has passed;
5. the final response states what changed, what was validated, and any remaining risk.

Never report completion merely because an implementation agent says it finished.
