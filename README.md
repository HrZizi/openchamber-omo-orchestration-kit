# OpenChamber OMO Orchestration Kit

Reusable project-local orchestration for **OpenChamber Desktop**, **OpenCode 2.x**, and **oh-my-opencode-slim (OMO) 3.0.2**.

## Included

- `SETUP_GUIDE.md` — macOS + Windows setup.
- `REMOTE_ORCHESTRATOR_PROMPT.md` — prompt for a separate web/remote orchestrator supervising the local stack.
- `config/opencode.json` — project-local OpenCode plugin registration.
- `config/oh-my-opencode-slim.json` — specialist agents and model routing.
- `config/oh-my-opencode-slim/orchestrator_append.md` — severity, safety, cost, verification, and completion policy.

## Architecture

```text
OpenChamber Desktop
  └─ OpenCode 2.x
      └─ project-local OMO
          ├─ free exploration / implementation / verification
          ├─ serious planning / review / audit
          ├─ CRITICAL Opus gates
          └─ optional paid DeepSeek escalation
```

No external OpenCode server is required.

## Routing philosophy

- LOW work stays cheap.
- MEDIUM/HIGH work gets stronger planning and audit.
- HIGH work gets an independent review challenge.
- CRITICAL work gets Opus-level planning and final audit.
- Implementation remains bounded and preferably free.
- Verification is independent where practical.
- Paid escalation must be explicit.
- Parallel edits require clear ownership.
- Existing repository work must be preserved.
