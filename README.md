# OpenChamber OMO Orchestration Kit

Reusable project-local multi-agent orchestration for **OpenChamber Desktop**, **OpenCode 2.x**, and **oh-my-opencode-slim (OMO) 3.0.2**.

This repository contains the **balanced-provider v2** configuration. It deliberately spreads serious planning/review/audit work across OpenAI and Claude, keeps routine implementation and verification on free OpenCode/Zen models, and treats direct DeepSeek as a last-resort paid implementation escalation.

## Included

- `SETUP_GUIDE.md` — macOS + Windows installation, provider setup, smoke tests, and recovery behavior.
- `REMOTE_ORCHESTRATOR_PROMPT.md` — prompt for an external/web orchestrator supervising the local stack.
- `config/opencode.json` — project-local OpenCode plugin registration.
- `config/oh-my-opencode-slim.json` — specialist agents, model chains, and provider concurrency limits.
- `config/oh-my-opencode-slim/orchestrator_append.md` — severity routing, provider health, fallback rules, audit circuit breaker, safety, and completion policy.

## Architecture

```text
OpenChamber Desktop
  └─ OpenCode 2.x
      └─ project-local OMO 3.0.2
          ├─ foreground: session-selected model
          ├─ Claude: serious planning / audit / critical gates
          ├─ OpenAI: foreground / independent challenge / oracle fallback
          ├─ Zen free models: exploration / implementation / verification
          └─ DeepSeek: last-resort paid implementation escalation
```

No external OpenCode server is required.

## Balanced-provider v2 routing

Normal HIGH flow:

```text
Sol Medium foreground
→ Sonnet 5.5 planner
→ Sol High independent reviewer
→ free bounded implementation
→ independent verification
→ Sonnet 5.5 auditor
```

Normal CRITICAL flow:

```text
Opus 5.5 critical plan
→ Sol High independent gate
→ bounded implementation
→ verification
→ serious audit
→ Opus 5.5 final critical audit
```

Oracle order:

```text
Opus 5.5
→ Astra xhigh
→ Sol High
```

Free implementation lanes:

```text
fixer              Muse → MiMo → Ling → Sonnet
implementer-alt    MiMo → Ling → Muse → Sonnet
implementer-ling   Ling → Muse → MiMo → Sonnet
```

If the shared Zen provider is genuinely degraded, use `implementer-claude` before `paid-fixer`.

## Provider resilience

The v2 policy distinguishes a single transient 429, repeated provider throttling, and permanent quota/auth/billing errors.

One transient 429 does **not** justify paid escalation. Repeated Zen failures in the same phase mark Zen degraded and route implementation through Claude first.

Background concurrency is deliberately conservative:

```text
total background jobs: 3
opencode:    1
claude-code: 1
openai:      1
deepseek:    1
```

This allows cross-provider parallelism while reducing rate-limit bursts against one provider.

## Audit circuit breaker

The policy prevents endless patch/audit loops:

- first same-family failure → bounded repair;
- second same-family failure → stop patching and re-plan;
- third same-family failure or three failures in a stage → Oracle required before further implementation.

## Foreground fallback

OpenCode v2 does not safely auto-switch the foreground model mid-turn. If the selected foreground OpenAI model is unavailable, switch OpenChamber manually to Claude Sonnet 5.5 (preferred) or a suitable free controller and continue the existing task.

See `SETUP_GUIDE.md` for the full procedure.
