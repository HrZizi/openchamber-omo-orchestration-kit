# OpenChamber + OMO Balanced-Provider v2 Setup

**Target:** OpenChamber Desktop + OpenCode 2.x + oh-my-opencode-slim 3.0.2  
**Platforms:** macOS and native Windows  
**Architecture:** OpenChamber uses its normal embedded OpenCode runtime. OMO is installed project-locally under the repository. No external OpenCode server is required.

## 1. Files

Copy:

```text
config/opencode.json
config/oh-my-opencode-slim.json
config/oh-my-opencode-slim/orchestrator_append.md
```

to:

```text
<PROJECT>/.opencode/opencode.json
<PROJECT>/.opencode/oh-my-opencode-slim.json
<PROJECT>/.opencode/oh-my-opencode-slim/orchestrator_append.md
```

Keep the installer's `.opencode/tui.json` if present.

## 2. Prerequisites

Install OpenChamber Desktop, Node.js 22+, an OpenCode 2 CLI for setup/diagnostics, Claude Code CLI, and the provider integrations used by the configured chains.

### macOS

```bash
brew install anomalyco/tap/opencode-v2
curl -fsSL https://claude.ai/install.sh | bash
opencode --version
claude --version
claude auth status
```

OpenCode may alternatively be installed with:

```bash
npm install -g @opencode/cli
```

### Windows

```powershell
npm install -g @opencode/cli
irm https://claude.ai/install.ps1 | iex
opencode --version
claude --version
claude auth status
```

Claude Code may alternatively be installed with:

```powershell
winget install Anthropic.ClaudeCode
```

## 3. Install OMO project-locally

### macOS

```bash
cd /path/to/project
mkdir -p .opencode

OPENCODE_CONFIG_DIR="$PWD/.opencode" \
npx oh-my-opencode-slim@3.0.2 install \
  --no-tui \
  --skills=yes \
  --background-subagents=no \
  --companion=no
```

### Windows

```powershell
Set-Location C:\path\to\project
New-Item -ItemType Directory -Force .opencode | Out-Null
$env:OPENCODE_CONFIG_DIR = (Resolve-Path .opencode).Path

npx oh-my-opencode-slim@3.0.2 install `
  --no-tui `
  --skills=yes `
  --background-subagents=no `
  --companion=no

Remove-Item Env:OPENCODE_CONFIG_DIR
```

Then replace the generated project config/policy with the supplied files.

Do not install this setup globally or use an external OpenCode server.

## 4. Enable background orchestration

Required:

```text
OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true
OPENCODE_ENABLE_EXA=1
```

### macOS GUI environment

```bash
launchctl setenv OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS true
launchctl setenv OPENCODE_ENABLE_EXA 1
```

### Windows user environment

```powershell
[Environment]::SetEnvironmentVariable(
  "OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS",
  "true",
  "User"
)
[Environment]::SetEnvironmentVariable(
  "OPENCODE_ENABLE_EXA",
  "1",
  "User"
)
```

Fully restart OpenChamber afterward.

Do not set `OPENCODE_CONFIG`, `OPENCODE_HOST`, `OPENCODE_SKIP_START`, or an external OpenCode server for this setup.

## 5. Provider setup

### OpenAI

The config references:

```text
openai/gpt-6.1-sol
openai/gpt-6-astra
```

Normal foreground choice:

```text
GPT-6.1 Sol
Medium
```

If foreground OpenAI usage is unavailable, manually switch the OpenChamber session to Claude Sonnet 5.5 and continue the existing task.

### Claude Code

In OpenChamber:

```text
Settings
→ Integrations
→ Claude Code
```

The config currently references:

```text
claude-code/claude-sonnet-5-5
claude-code/claude-opus-5-5
```

### OpenCode / Zen

The config references free lanes including:

```text
opencode/muse-spark-1.3-contributor-free
opencode/mimo-v2.6-flash-free
opencode/ling-3.1-flash-free
opencode/nemotron-3-ultra-free
opencode/nemotron-3.5-lightning-free
opencode/space-bunny-free
```

Free catalogs change. Replace unavailable IDs while preserving role semantics.

### DeepSeek

Optional last-resort paid lane:

```text
deepseek/deepseek-flash
```

`paid-fixer` intentionally has no free fallback so paid escalation remains observable.

## 6. Validate before opening OpenChamber

### macOS

```bash
OPENCODE_CONFIG_DIR="$PWD/.opencode" \
npx oh-my-opencode-slim@3.0.2 doctor
```

### Windows

```powershell
$env:OPENCODE_CONFIG_DIR = (Resolve-Path .opencode).Path
npx oh-my-opencode-slim@3.0.2 doctor
Remove-Item Env:OPENCODE_CONFIG_DIR
```

Do not continue until both user/project config checks are clean.

## 7. Expected specialist roster

```text
auditor
auditor-low
auditor-sol
critical-auditor
critical-planner
designer
explorer
fixer
implementer-alt
implementer-claude
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
```

## 8. Normal routing

### LOW

```text
planner-low
→ reviewer-low when useful
→ free implementation
→ verification
→ auditor-low when warranted
```

### MEDIUM

```text
planner (Sonnet)
→ bounded free implementation
→ verification
→ auditor (Sonnet)
```

### HIGH

```text
planner (Sonnet)
→ reviewer (Sol High)
→ reconciliation
→ bounded free implementation
→ verification
→ auditor (Sonnet)
```

### CRITICAL

```text
critical-planner (Opus)
→ independent Sol High gate
→ smallest reversible implementation stages
→ verification
→ serious audit
→ critical-auditor (Opus)
```

### Oracle

```text
Opus
→ Astra xhigh
→ Sol High
```

## 9. Free implementation ladder

```text
fixer:
Muse → MiMo → Ling → Sonnet

implementer-alt:
MiMo → Ling → Muse → Sonnet

implementer-ling:
Ling → Muse → MiMo → Sonnet
```

If one free lane fails with an ordinary transient provider error, do not immediately use paid capacity.

If two independent Zen child dispatches in the same phase terminate on provider-capacity/rate-limit errors, mark Zen degraded for that phase and route further required implementation through:

```text
implementer-claude
```

before considering:

```text
paid-fixer
```

## 10. Concurrency and provider health

The supplied config uses:

```text
default total background concurrency: 3
opencode:    1
claude-code: 1
openai:      1
deepseek:    1
```

Provider health is tracked conceptually as:

```text
HEALTHY
TRANSIENT_THROTTLED
DEGRADED
```

A single ordinary 429 is transient. Permanent quota/auth/billing failures are degraded immediately.

Do not repeatedly dispatch into a provider already known to be degraded.

## 11. Audit-loop circuit breaker

```text
first same-family audit failure
→ bounded repair

second same-family failure
→ stop implementation
→ re-plan/review the invariant

third same-family failure or three failures in a stage
→ Oracle required
→ invariant-level repair only
```

## 12. Librarian

`librarian` is deliberately read-only: edit, bash, and child-task delegation are denied.

If a permission loop still occurs, terminate the lane and use another read-only research path rather than repeatedly approving/cancelling an inaccessible request.

## 13. Registration smoke test

After fully restarting OpenChamber, send:

```text
Do not modify any files and do not dispatch any specialist yet.

Identify your foreground model.
List every available specialist agent.
For planner, reviewer, auditor, oracle, fixer, implementer-alt,
implementer-ling, implementer-claude, verification, planner-sol,
reviewer-claude and auditor-sol, report the configured model chain.
Report configured background concurrency limits.
Confirm the project orchestration policy is loaded.
Do not infer missing information.
```

Configuration visibility proves registration, not that every model can execute. Runtime availability is proven only by an actual dispatch.

## 14. OpenCode v2 fallback boundary

Model arrays are the first delegated-agent fallback layer, but the policy does not assume they recover every OpenCode v2 failure mode.

If a child still terminates on provider error, stopped-without-terminal-result, stale/inconclusive recovery, or similar failure, use the explicit orchestration-level alternate route.

Foreground automatic switching is not relied upon. Switch the session model manually when necessary.

## 15. Git safety

Recommended files to version:

```text
.opencode/opencode.json
.opencode/oh-my-opencode-slim.json
.opencode/oh-my-opencode-slim/orchestrator_append.md
```

Do not commit provider credentials, local authentication data, or machine-specific secrets.
