# OpenChamber OMO Orchestration Kit

<div align="center">

**A project-local, multi-provider agent orchestration stack for OpenChamber + OpenCode.**

Use strong models where judgment matters, free models where they are sufficient, keep verification independent, and keep working when one provider is throttled or unavailable.

[![OpenCode](https://img.shields.io/badge/OpenCode-2.x-111111)](https://opencode.ai/)
[![OMO](https://img.shields.io/badge/oh--my--opencode--slim-3.0.2-5c6ac4)](https://github.com/alvinunreal/oh-my-opencode-slim)
[![OpenChamber](https://img.shields.io/badge/OpenChamber-Desktop-4b5563)](https://github.com/openchamber/openchamber)
[![Platforms](https://img.shields.io/badge/platforms-macOS%20%7C%20Windows-0a7ea4)](#platforms)
[![Policy](https://img.shields.io/badge/policy-balanced--provider%20v2.1-2ea44f)](#routing-policy)

</div>

---

## Overview

This repository is a reusable orchestration layer for **OpenChamber Desktop**, **OpenCode 2.x**, and **oh-my-opencode-slim (OMO) 3.0.2**.

It is designed for people who want coding agents to behave more like a small engineering team than a single model repeatedly planning, implementing, reviewing, and approving its own work.

The configuration provides:

- severity-aware routing for **LOW / MEDIUM / HIGH / CRITICAL** work;
- independent planning, review, implementation, verification, and audit roles;
- deliberate workload balancing across **OpenAI**, **Claude Code**, **OpenCode/Zen**, and optional **DeepSeek**;
- free-model-first implementation and verification where practical;
- reciprocal Haiku/Luna lightweight subscription fallbacks after free operational agents;
- cross-provider recovery when one provider is throttled or unavailable;
- bounded background concurrency to reduce provider bursts and 429s;
- an audit circuit breaker to stop endless patch → audit → patch loops;
- deterministic, readable subagent chat names;
- project-local configuration that can be versioned with the repository;
- an optional prompt for supervising the local stack from a separate web/remote orchestrator.

The goal is **not maximum delegation**. The goal is to use the right level of reasoning, independence, cost, and redundancy for the task at hand.

> [!IMPORTANT]
> This repository contains orchestration configuration and policy. It does **not** replace OpenChamber, OpenCode, or OMO, and it does not run a separate OpenCode server.

---

## Why use an orchestrator at all?

A single capable coding model can do a lot, but long-running software work exposes several recurring problems.

### Planning and implementation are different jobs

The model that writes code should not always be the model that decides what the code should be. Separating those roles makes it easier to catch scope creep, architectural mistakes, and assumptions before implementation begins.

### Self-review is weaker than independent review

A model often preserves its own assumptions when reviewing its own work. This stack uses separate review and audit agents—often on a different provider—to create a genuinely independent challenge.

### Premium models are expensive capacity

Routine repository exploration, focused implementation, and test execution usually do not need the most expensive model available. The policy keeps ordinary work on free or cheaper lanes and spends premium capacity at decision gates.

### Provider limits are part of the system

Rate limits, temporary throttling, exhausted subscription windows, authentication failures, and provider outages happen. A robust workflow should degrade gracefully instead of collapsing or immediately jumping to a paid fallback.

### Long tasks need control loops

Without explicit policy, a coding agent can get stuck in:

~~~text
implement
→ audit fails
→ patch one symptom
→ audit fails
→ patch another symptom
→ ...
~~~

This stack tracks repeated failure families and forces re-planning or Oracle escalation before the loop becomes unproductive.

### Evidence matters more than agent confidence

A task is not considered complete because an implementation agent says "done". Verification and audit are separate responsibilities, and final conclusions are expected to be grounded in the live tree, tests, and observed evidence.

---

## Why OMO?

[oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) provides the agent and background-orchestration primitives that make this architecture practical inside OpenCode:

- named specialist agents;
- per-agent model chains;
- child/subagent sessions;
- background jobs;
- agent-specific prompts and permissions;
- model fallback arrays;
- project-local configuration;
- tool and session hooks.

This repository adds an opinionated **engineering policy** on top of those primitives.

**OMO provides the machinery. This repository defines how that machinery should be used.**

---

## Architecture

~~~mermaid
flowchart TD
    U[User in OpenChamber] --> F[Foreground coordinator<br/>session-selected model]

    F --> P[Planning]
    F --> I[Implementation]
    F --> V[Verification]
    F --> A[Audit]

    P --> C1[Claude<br/>Sonnet / Opus]
    P --> O1[OpenAI<br/>Sol / Astra]

    I --> Z[OpenCode / Zen<br/>free model lanes]
    Z --> IC[Subscription fallback<br/>Haiku / Luna / Sonnet / Sol]
    IC --> D[DeepSeek paid fallback<br/>last resort]

    V --> ZV[Free verification lane]
    ZV --> VC[Claude verification fallback]

    A --> C2[Claude serious audit]
    A --> O2[OpenAI independent challenge]

    F --> OR[Oracle<br/>Opus → Astra → Sol]
~~~

The foreground session stays under the user's control. OMO specialists are delegated deliberately according to severity, provider health, and task ownership.

There is **no external OpenCode server** in this architecture.

---

## Design principles

### Project-local by default

The source of truth lives inside each project:

~~~text
<PROJECT>/.opencode/
├── opencode.json
├── tui.json
├── oh-my-opencode-slim.json
└── oh-my-opencode-slim/
    └── orchestrator_append.md
~~~

Why project-local?

- the orchestration policy can be versioned with the codebase;
- different repositories can use different routing policies;
- teammates can reproduce the same setup;
- global OpenCode state does not silently become the project's source of truth;
- upgrades and experiments are easier to review and roll back.

### Separate judgment from execution

Strong reasoning models are concentrated around planning, adversarial review, difficult architecture decisions, and final audits. Routine execution is delegated to cheaper/free workers when safe.

### Prefer provider diversity over model-count theater

Three free models from the same provider do not provide the same resilience as two independent providers.

The policy distinguishes:

- **model diversity** — Muse vs MiMo vs Ling;
- **provider diversity** — Zen vs Claude Code vs OpenAI vs direct DeepSeek.

That distinction matters during provider-level throttling.

### Preserve user work

The policy explicitly forbids destructive shortcuts such as resetting unrelated work, force-pushing, rewriting shared history, or deleting evidence just to make a gate pass.

---

## Routing policy

### LOW

For small, reversible work:

~~~text
planner-low
→ reviewer-low when useful
→ free implementation
→ verification
→ auditor-low when warranted
~~~

The goal is to stay cheap and fast without creating ceremony for trivial changes.

### MEDIUM

~~~text
planner (Sonnet 5.5)
→ bounded free implementation
→ verification
→ auditor (Sonnet 5.5)
~~~

A separate Sol reviewer can be added when design ambiguity is meaningful.

### HIGH

~~~text
planner (Sonnet 5.5)
→ reviewer (Sol High)
→ reconciliation
→ bounded implementation
→ verification
→ auditor (Sonnet 5.5)
~~~

HIGH work deliberately uses different providers for plan and challenge.

### CRITICAL

~~~text
critical-planner (Opus 5.5)
→ independent Sol High challenge
→ smallest reversible implementation stages
→ verification
→ serious audit
→ critical-auditor (Opus 5.5)
~~~

CRITICAL work is not complete without its final critical audit.

### Oracle

Oracle is not a normal severity tier. It exists for unresolved architecture/debugging disagreements or repeated same-family failures.

~~~text
Opus 5.5
→ Astra xhigh
→ Sol High
~~~

---

## Agent and model map

The exact model IDs live in [config/oh-my-opencode-slim.json](config/oh-my-opencode-slim.json).

| Agent | Primary route | Purpose |
|---|---|---|
| <code>planner-low</code> | MiMo → Ling → Nemotron Ultra → Sonnet | Cheap planning for LOW-risk work |
| <code>reviewer-low</code> | Nemotron Ultra → Ling → MiMo → Sonnet | Lightweight independent plan challenge |
| <code>planner</code> | Sonnet → Sol High | Normal MEDIUM/HIGH planning |
| <code>planner-sol</code> | Sol High | Explicit OpenAI planning fallback |
| <code>reviewer</code> | Sol High → Sonnet | Independent MEDIUM/HIGH challenge |
| <code>reviewer-claude</code> | Sonnet | Explicit Claude review fallback |
| <code>critical-planner</code> | Opus → Sol High → Astra xhigh | CRITICAL planning |
| <code>explorer</code> | MiMo → Ling → Space Bunny → **Luna → Haiku** → Sonnet | Repository reconnaissance |
| <code>librarian</code> | Ling → MiMo → Space Bunny → **Luna → Haiku** → Sonnet | Read-only research / documentation |
| <code>designer</code> | Muse → MiMo → Ling → **Haiku → Luna** → Sonnet | UI/design-oriented work |
| <code>observer</code> | Muse → MiMo → Ling → **Luna → Haiku** → Sonnet | Image-heavy observation |
| <code>fixer</code> | Muse → MiMo → Ling → **Haiku → Luna** → Sonnet | Primary implementation lane |
| <code>implementer-alt</code> | MiMo → Ling → Muse → **Haiku → Luna** → Sonnet | Alternate free implementation lane |
| <code>implementer-ling</code> | Ling → Muse → MiMo → **Haiku → Luna** → Sonnet | Third free implementation lane |
| <code>subscription-implementer</code> | **Haiku → Luna → Sonnet → Sol Medium** | Provider-neutral subscription implementation fallback |
| <code>verification</code> | Nemotron Lightning → Ling → MiMo → Sonnet | Independent verification |
| <code>verification-claude</code> | Sonnet | Cross-provider verification fallback |
| <code>auditor-low</code> | Nemotron Ultra → Ling → MiMo → Sonnet | LOW-risk final audit |
| <code>auditor</code> | Sonnet → Sol High | MEDIUM/HIGH adversarial audit |
| <code>auditor-sol</code> | Sol High | Explicit OpenAI audit fallback |
| <code>critical-auditor</code> | Opus → Sol High → Astra xhigh | Mandatory CRITICAL final gate |
| <code>oracle</code> | Opus → Astra xhigh → Sol High | Deep unresolved reasoning |
| <code>paid-fixer</code> | DeepSeek Flash | Last-resort paid implementation |

> [!NOTE]
> Model catalogs change quickly. The architecture is intentionally role-based: if a free model disappears, replace it with an equivalent model without changing the policy semantics unless there is evidence that the role itself should change.

---

## Free-first implementation

The normal free implementation lanes are:

~~~text
fixer              Muse → MiMo → Ling
implementer-alt    MiMo → Ling → Muse
implementer-ling   Ling → Muse → MiMo
~~~

They provide **model** diversity, but share Zen provider capacity. If Zen is genuinely degraded or an approved task is unsuitable for the free tier, use the subscription lane:

~~~text
Zen free work
     ↓
subscription-implementer
Haiku → Luna → Sonnet → Sol Medium
     ↓
paid-fixer (direct paid DeepSeek; justified last resort only)
~~~

A single ordinary 429 never justifies immediate direct paid escalation.

---

## Lightweight subscription tier — v2.1

Haiku 5.5 and GPT-6 Luna form an inexpensive cross-provider recovery pair **between free Zen workers and stronger Sonnet/Sol models**.

| Operational agent | Free head | Subscription fallback |
|---|---|---|
| Explorer | MiMo → Ling → Space Bunny | **Luna → Haiku → Sonnet** |
| Librarian (read-only) | Ling → MiMo → Space Bunny | **Luna → Haiku → Sonnet** |
| Observer (visual) | Muse → MiMo → Ling | **Luna → Haiku → Sonnet** |
| Designer | Muse → MiMo → Ling | **Haiku → Luna → Sonnet** |
| Fixer | Muse → MiMo → Ling | **Haiku → Luna → Sonnet** |
| Alternate implementer | MiMo → Ling → Muse | **Haiku → Luna → Sonnet** |
| Ling implementer | Ling → Muse → MiMo | **Haiku → Luna → Sonnet** |
| Subscription implementer | — | **Haiku → Luna → Sonnet → Sol Medium** |

Keep `planner-low`, `reviewer-low`, independent verification, audit, serious/critical gates and Oracle **unchanged**. This is an operational cost/resilience improvement, not an instruction to weaken judgment quality.

The new configured IDs are `claude-code/claude-haiku-5-5` and `openai/gpt-6-luna`. The bare vendor IDs are documented upstream, but actual availability through the installed OpenChamber provider integrations still requires runtime validation.

### Observer visual fallback

The Observer retains `image_routing: "auto"` and its original free models, then tries Luna, Haiku and Sonnet on supported failure modes. Both lightweight subscription models support image input, but the host/provider bridge must be tested with real screenshots. Do not claim that text-only evidence is visual verification.

### Errors are not quality failures

Ordered model arrays are recovery preferences for model/runtime **errors**; they do not recognize incorrect completed code. After a failed verification, the orchestrator must deliberately choose a repair or stronger model and re-run the appropriate gates.

Because free-first arrays contain subscription fallbacks, they can spend subscription capacity after an individual Zen error *before* the orchestrator has formally marked Zen DEGRADED. Conversely, OpenCode v2 may not advance a chain for every type of failure. Track actual model execution and do not imply these policy thresholds are enforced by static arrays.

## Provider-health model

The policy tracks each relevant provider conceptually as:

~~~text
HEALTHY
TRANSIENT_THROTTLED
DEGRADED
~~~

### Transient throttling

A single ordinary 429 means **TRANSIENT_THROTTLED**.

The orchestrator should:

- allow the current model chain to recover;
- make at most one deliberate alternate free-lane attempt if the child terminates;
- avoid launching several more jobs at the same provider;
- avoid paid escalation.

### Degraded provider

If repeated independent jobs in the same phase terminate due to the same provider's capacity/rate-limit condition, the provider is marked **DEGRADED** for that phase.

For Zen implementation work, that means moving to <code>subscription-implementer</code>, subject to the actual health of the remaining providers.

Permanent quota, authentication, billing, or clearly unavailable-provider errors also move a provider directly to DEGRADED.

---

## Concurrency

The v2.1 configuration preserves the same conservative limits:

~~~text
total background jobs: 3

opencode:     1
claude-code:  1
openai:       1
deepseek:     1
~~~

This allows useful cross-provider parallelism while preventing a swarm of simultaneous jobs against one provider.

Parallel implementation is allowed only when:

- dependencies are satisfied;
- write ownership is explicit;
- write sets are disjoint or properly isolated;
- neither task depends on unfinished output from the other;
- reconciliation order is known.

---

## Audit circuit breaker

The stack explicitly prevents infinite micro-patching.

~~~text
Audit FAIL #1
→ bounded repair
→ verify
→ re-audit

Audit FAIL #2 in same subsystem/finding family
→ STOP local patching
→ return to serious planning/review
→ revise invariant-level repair plan

Audit FAIL #3 in same family
or three audit failures in one stage
→ Oracle required
→ only then resume implementation
~~~

This is one of the most important differences between "multiple agents" and an actual orchestration policy.

---

## Verification is not audit

These are intentionally separate concepts.

### Verification asks

- did the tests run?
- does the observed output match the acceptance criteria?
- did a regression appear?
- does the live tree contain what the implementation claims?

### Audit asks

- was the right thing implemented?
- is the design sound?
- are important edge cases missing?
- is the evidence strong enough?
- did the implementation accidentally weaken the original requirements?

The implementation agent is not allowed to self-certify either one.

---

## Readable subagent sessions

Every new specialist session follows:

~~~text
<Readable Agent Name> #<N> — <short task description>
~~~

Examples:

~~~text
Planner #1 — Design migration strategy
Reviewer #1 — Challenge migration strategy
Fixer #2 — Repair parser edge cases
Oracle #1 — Resolve ownership semantics
~~~

Counters are per role inside the parent session. Resuming an existing child keeps its original name and number.

On OpenCode v2, the subagent description becomes the child-session title, so this convention makes long OpenChamber runs much easier to navigate.

---

## Quick start

For the complete installation procedure, use **[SETUP_GUIDE.md](SETUP_GUIDE.md)**.

At a high level:

1. Install OpenChamber Desktop, OpenCode 2.x, Node.js, and Claude Code.
2. Install OMO **project-locally** into your repository's <code>.opencode/</code> directory.
3. Copy this repository's configuration files into that project.
4. Enable OMO background-subagent support for the OpenChamber process.
5. Connect the providers you want to use.
6. Run OMO's <code>doctor</code>.
7. Fully restart OpenChamber.
8. Start a **new session** after orchestration prompt/agent-definition changes.
9. Run the registration smoke test from the setup guide.

The three files you normally version in a target project are:

~~~text
.opencode/opencode.json
.opencode/oh-my-opencode-slim.json
.opencode/oh-my-opencode-slim/orchestrator_append.md
~~~

---

## Platforms

The setup targets:

- **macOS**
- **native Windows**

WSL is not the intended OpenChamber Desktop deployment path for this configuration.

Platform-specific environment setup and commands live in [SETUP_GUIDE.md](SETUP_GUIDE.md).

---

## Project structure

~~~text
.
├── README.md
├── SETUP_GUIDE.md
├── REMOTE_ORCHESTRATOR_PROMPT.md
└── config/
    ├── opencode.json
    ├── oh-my-opencode-slim.json
    └── oh-my-opencode-slim/
        └── orchestrator_append.md
~~~

### config/opencode.json

Registers the pinned OMO plugin and disables OpenCode's generic built-in agents that overlap with this setup.

### config/oh-my-opencode-slim.json

Defines:

- specialist agents;
- model chains and variants;
- fallback agents;
- permissions;
- background concurrency;
- image routing;
- project-level OMO behavior.

### config/oh-my-opencode-slim/orchestrator_append.md

Contains the engineering policy:

- severity classification;
- routing;
- provider health;
- fallback behavior;
- verification and audit rules;
- circuit breaker;
- paid escalation;
- Git safety;
- completion requirements;
- subagent naming.

### REMOTE_ORCHESTRATOR_PROMPT.md

Optional prompt for a separate ChatGPT/web/remote session that supervises the local OpenChamber + OMO execution engine without duplicating its implementation work.

---

## Using a remote/web orchestrator

Some workflows benefit from separating **execution** from **supervision**.

~~~text
Web / remote orchestrator
        ↓
frames task + acceptance criteria
        ↓
OpenChamber foreground coordinator
        ↓
OMO specialist team
        ↓
implementation + verification + audit
        ↓
evidence returned to web orchestrator
        ↓
PASS / targeted follow-up / FAIL
~~~

Use [REMOTE_ORCHESTRATOR_PROMPT.md](REMOTE_ORCHESTRATOR_PROMPT.md) when you want that workflow.

The external orchestrator should supervise the local system, not manually micromanage every subagent call.

---

## Foreground model behavior

The foreground model is selected in OpenChamber. The normal recommendation in this configuration is:

~~~text
GPT-6.1 Sol
Medium
~~~

Because <code>stripOrchestratorModel: true</code> is enabled, the OpenChamber session selection remains authoritative.

### Important OpenCode v2 limitation

Foreground model fallback is **manual**.

If the current foreground OpenAI model becomes unavailable or its usage window is exhausted, switch the OpenChamber session manually to Claude Sonnet 5.5 (or another suitable controller) and continue the existing task.

Delegated specialist agents still have their own configured model chains.

---

## Configuration lifecycle

OpenCode v2 does not treat every config change identically.

Agent definitions and orchestration prompt changes should be treated as **new-session changes**.

After modifying:

~~~text
.opencode/oh-my-opencode-slim.json
.opencode/oh-my-opencode-slim/orchestrator_append.md
~~~

fully restart OpenChamber when appropriate and start a **fresh chat** so the new policy/agent definitions are definitely loaded.

Do not assume an old long-running session has adopted structural orchestration changes.

---

## Validation

Before opening a new OpenChamber session, validate the project config.

### macOS

~~~bash
OPENCODE_CONFIG_DIR="$PWD/.opencode" \
npx oh-my-opencode-slim@3.0.2 doctor
~~~

### Windows PowerShell

~~~powershell
$env:OPENCODE_CONFIG_DIR = (Resolve-Path .opencode).Path
npx oh-my-opencode-slim@3.0.2 doctor
Remove-Item Env:OPENCODE_CONFIG_DIR
~~~

A project-local-only setup may legitimately show:

~~~text
[user] No config file found
[project] <PROJECT>\.opencode\oh-my-opencode-slim.json ✓
~~~

The missing user-level config is informational; the project config is the source of truth for this architecture.

See the setup guide for the full registration smoke test.

---

## Expected specialist roster

A correctly loaded v2 project exposes:

~~~text
auditor
auditor-low
auditor-sol
critical-auditor
critical-planner
designer
explorer
fixer
implementer-alt
subscription-implementer
implementer-ling
librarian
observer
oracle
paid-fixer
planner
planner-low
planner-sol
reviewer
reviewer-claude
reviewer-low
verification
verification-claude
~~~

---

## What this setup deliberately does not do

This repository does **not**:

- run a separate <code>opencode serve</code> instance for OpenChamber;
- require ACP wrappers for Claude;
- make a global OMO configuration the project's source of truth;
- auto-spend paid DeepSeek capacity after one free-model error;
- treat every task as HIGH or CRITICAL;
- launch as many subagents as possible;
- allow multiple agents to edit the same files without ownership;
- allow the implementation agent to declare itself verified;
- automatically force-switch the foreground model on OpenCode v2;
- rewrite Git history or discard unrelated user work.

---

## Customizing the stack

This repository is intentionally opinionated, but the architecture is reusable.

Common safe customizations include:

- replacing unavailable free models while preserving their role;
- changing the preferred foreground model;
- adding a specialist with a clearly defined responsibility;
- changing severity thresholds for your project;
- tightening permissions for read-only agents;
- adjusting concurrency for a provider with known limits;
- changing the final paid escalation provider.

When changing the stack, prefer **role-based changes** over simply appending every new model to every fallback chain.

A new free model is not automatically a better fit for every role.

---

## Safety and repository hygiene

The orchestration policy requires the local stack to preserve existing work.

Agents should not:

- discard unrelated modifications;
- use destructive reset/clean commands to resolve conflicts;
- force-push;
- rewrite shared history;
- delete fixtures, logs, or evidence just to make validation pass;
- commit credentials or provider secrets.

The configuration files in this repository contain no provider credentials. Authentication remains with OpenChamber/OpenCode/provider tooling.

---

## Limitations

This architecture intentionally accepts a few limitations:

- model availability and free-model catalogs change;
- multiple "different" free models may still share the same Zen provider capacity;
- automatic foreground failover is not relied upon on OpenCode v2;
- OMO model-array fallback is useful but is not treated as a guarantee for every failure mode, or as a quality check; subscription capacity may be used before Zen's degraded threshold is established;
- permissions/tool availability can vary across hosts;
- large multi-agent runs still require good task boundaries and acceptance criteria;
- no orchestration policy can replace actual tests and repository-specific engineering judgment.

---

## Troubleshooting

### doctor says [user] No config file found

That is fine for this architecture if the project config is valid:

~~~text
[project] ...\.opencode\oh-my-opencode-slim.json ✓
~~~

The user-level OMO config is optional.

### New agents or policy changes are not showing up

Start a **new OpenChamber session**. For structural config/prompt changes, fully restart OpenChamber first.

### A free implementation model receives one 429

Do not immediately use <code>paid-fixer</code>. One 429 is treated as transient.

### Several Zen workers fail in the same phase

Treat Zen as provider-degraded and use <code>subscription-implementer</code> rather than hammering more Zen models. Its preference order is Haiku → Luna → Sonnet → Sol Medium; record which provider/model actually executed.

### The foreground model runs out of usage

Manually switch the OpenChamber foreground model. Do not rewrite the project configuration just to recover the current task.

### Audit keeps finding new variants of the same defect

Follow the circuit breaker. After the second same-family failure, stop patching and re-plan. After the third, use Oracle before further implementation.

---

## Documentation

| Document | Use it for |
|---|---|
| [README.md](README.md) | Architecture, philosophy, routing, and project overview |
| [SETUP_GUIDE.md](SETUP_GUIDE.md) | Installation and platform-specific setup |
| [REMOTE_ORCHESTRATOR_PROMPT.md](REMOTE_ORCHESTRATOR_PROMPT.md) | Supervising the local stack from an external/web orchestrator |
| [config/oh-my-opencode-slim.json](config/oh-my-opencode-slim.json) | Exact agents, models, fallbacks, permissions, and concurrency |
| [orchestrator_append.md](config/oh-my-opencode-slim/orchestrator_append.md) | Exact orchestration policy |

---

## Upstream projects

This kit builds on:

- [OpenChamber](https://github.com/openchamber/openchamber)
- [OpenCode](https://opencode.ai/)
- [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)
- [Claude Code](https://github.com/anthropics/claude-code)

Use the upstream projects' documentation for their installation, provider support, and product-specific behavior.

---

## Philosophy in one sentence

> Use cheap models for execution, strong models for judgment, different providers for independent challenge, and evidence—not confidence—as the definition of done.
