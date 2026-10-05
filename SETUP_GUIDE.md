# OpenChamber + OMO Orchestration Setup Guide

**Last verified:** 2026-10-05  
**Target:** OpenChamber Desktop + OpenCode 2.x + oh-my-opencode-slim (OMO) 3.0.2  
**Platforms:** macOS and native Windows  
**Architecture:** OpenChamber Desktop uses its own normal embedded OpenCode runtime. OMO is installed **project-locally** under the project repository. No external OpenCode server is required.

## 1. What this setup is

The recommended project setup is:

```text
OpenChamber Desktop
        │
        ▼
OpenCode 2.x runtime
        │
        ▼
<PROJECT>/.opencode/
        ├── opencode.json
        ├── oh-my-opencode-slim.json
        └── oh-my-opencode-slim/
            └── orchestrator_append.md
        │
        ▼
oh-my-opencode-slim 3.0.2
        │
        ├── foreground manager: session-selected model
        │   normal choice: GPT-6.1 Sol / Medium
        │
        ├── free exploration / implementation / verification lanes
        ├── Sol + Claude serious planning/review/audit lanes
        ├── Opus CRITICAL lanes
        └── paid DeepSeek implementation escalation
```

This is deliberately **project-local**. Do not install OMO globally for this project setup and do not depend on `OPENCODE_CONFIG` overrides.

### Important non-goals

Do **not**:

- run `opencode serve`;
- point OpenChamber at an external OpenCode server;
- set `OPENCODE_HOST` or `OPENCODE_SKIP_START`;
- force OpenChamber to use a custom CLI path;
- symlink OMO into global OpenCode plugin folders;
- add ACP wrappers for Claude;
- add experimental test agents to prove Claude connectivity;
- hand-edit OpenChamber's managed configuration;
- use a global OMO config as the source of truth for the project.

Claude is provided through OpenChamber's **Claude Code integration** and appears to OpenCode/OMO as the `claude-code` provider.

---

## 2. Versions this guide targets

At the time this guide was written:

- OpenChamber **2.1.1** is the current release.
- OMO **3.0.2** is the current release and is pinned in this setup.
- The setup was proven with OpenCode **2.0.18**; target OpenCode 2.x.
- Claude Code supplies the Claude model list dynamically from the locally signed-in Claude CLI.

The config pins OMO to `oh-my-opencode-slim@3.0.2` intentionally. OpenCode 2 may check/update unpinned package plugins, so exact pinning keeps the tested behavior stable.

Official references:

- OpenChamber: https://github.com/openchamber/openchamber
- OMO: https://github.com/alvinunreal/oh-my-opencode-slim
- OMO install guide: https://github.com/alvinunreal/oh-my-opencode-slim/blob/master/docs/installation.md
- OMO configuration: https://github.com/alvinunreal/oh-my-opencode-slim/blob/master/docs/configuration.md
- OpenCode 2 config: https://opencode.ai/v2/docs/config
- OpenCode 2 plugins: https://opencode.ai/v2/docs/plugins
- OpenChamber Claude Code provider: https://github.com/openchamber/opencode-claude
- Claude Code: https://github.com/anthropics/claude-code

---

# Part A — macOS

## 3. Prerequisites on macOS

### 3.1 Install OpenChamber Desktop

Download the current OpenChamber Desktop build from:

https://github.com/openchamber/openchamber/releases

Install it normally and launch it once.

OpenChamber Desktop bundles the matching OpenCode runtime. You are **not** going to run or manage an external OpenCode server.

### 3.2 Install a terminal OpenCode CLI

The project-local OMO installer checks for an `opencode` CLI, so install OpenCode 2 in the terminal as a setup/diagnostic dependency.

Homebrew:

```bash
brew install anomalyco/tap/opencode-v2
```

or npm:

```bash
npm install -g @opencode/cli
```

Verify:

```bash
opencode --version
```

The terminal CLI is not used as an external server for OpenChamber.

### 3.3 Install Node.js / npx

Use a current Node.js version. Node 22+ is a conservative baseline for the current OpenCode/OpenChamber ecosystem.

Verify:

```bash
node --version
npx --version
```

### 3.4 Install Claude Code

Recommended native installer:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Alternative on macOS:

```bash
brew install --cask claude-code
```

Verify:

```bash
claude --version
claude auth status
```

If not signed in:

```bash
claude auth login --claudeai
```

---

## 4. Prepare the project repository on macOS

Assume the repository is:

```text
/Users/<you>/your-project
```

Enter the repository:

```bash
cd /Users/<you>/your-project
```

Create the project-local OpenCode directory:

```bash
mkdir -p .opencode
```

If a previous experimental `.opencode` setup exists and you want a clean rebuild, move it aside first rather than deleting it:

```bash
mv .opencode "$HOME/project-old-opencode-$(date +%Y%m%d-%H%M%S)"
mkdir -p .opencode
```

---

## 5. Install OMO project-locally on macOS

Run the installer with `OPENCODE_CONFIG_DIR` pointing at **this project's** `.opencode` directory:

```bash
cd /Users/<you>/your-project

OPENCODE_CONFIG_DIR="$PWD/.opencode" \
npx oh-my-opencode-slim@3.0.2 install \
  --no-tui \
  --skills=yes \
  --background-subagents=no \
  --companion=no
```

The important property is not the exact shell syntax; it is that the installer writes to:

```text
<PROJECT>/.opencode/
```

instead of:

```text
~/.config/opencode/
```

Expected generated files include:

```text
.opencode/opencode.json
.opencode/tui.json
.opencode/oh-my-opencode-slim.json
```

Do not move them to the global OpenCode config directory.

---

## 6. Enable OMO background orchestration for OpenChamber on macOS

OMO's background orchestration depends on:

```text
OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true
OPENCODE_ENABLE_EXA=1
```

Because OpenChamber is a GUI app, make them available to the macOS user launch environment:

```bash
launchctl setenv OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS true
launchctl setenv OPENCODE_ENABLE_EXA 1
```

Verify:

```bash
launchctl getenv OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS
launchctl getenv OPENCODE_ENABLE_EXA
```

Expected:

```text
true
1
```

These `launchctl setenv` values belong to the current login session. If a future reboot/login causes background orchestration to disappear, set them again before launching OpenChamber.

For terminal-only diagnostics you may also export them in your shell, but that is separate from the GUI environment.

Do **not** set:

```text
OPENCODE_CONFIG
OPENCODE_CONFIG_CONTENT
OPENCODE_HOST
OPENCODE_SKIP_START
OPENCODE_BINARY
```

for this setup.

---

## 7. Install the final project orchestration config on macOS

This setup bundle contains:

```text
config/opencode.json
config/oh-my-opencode-slim.json
config/oh-my-opencode-slim/orchestrator_append.md
```

From the bundle directory, copy them into the project repository:

```bash
cd /Users/<you>/your-project
mkdir -p .opencode/oh-my-opencode-slim
```

Then copy:

```text
opencode.json
    -> .opencode/opencode.json

oh-my-opencode-slim.json
    -> .opencode/oh-my-opencode-slim.json

orchestrator_append.md
    -> .opencode/oh-my-opencode-slim/orchestrator_append.md
```

Keep the installer's `.opencode/tui.json` if present.

The final structure should be:

```text
your-project/
└── .opencode/
    ├── opencode.json
    ├── tui.json
    ├── oh-my-opencode-slim.json
    └── oh-my-opencode-slim/
        └── orchestrator_append.md
```

---

## 8. Configure providers in OpenChamber on macOS

Open OpenChamber Desktop normally.

### OpenAI

Connect the OpenAI provider/account used for:

```text
openai/gpt-6.1-sol
openai/gpt-6-astra
```

Normal foreground model:

```text
GPT-6.1 Sol
Medium
```

### OpenCode / Zen

Connect the OpenCode provider so the configured free models are available.

The current config expects:

```text
opencode/mimo-v2.6-flash-free
opencode/muse-spark-1.3-contributor-free
opencode/nemotron-3-ultra-free
opencode/nemotron-3.5-lightning-free
opencode/space-bunny-free
opencode/big-pickle
```

The free catalog can change. If an ID disappears, update the OMO config to a currently available equivalent rather than changing the orchestration policy.

### Claude Code

In OpenChamber:

```text
Settings
→ Integrations
→ Claude Code
```

Install/enable the integration and sign in through Claude Code CLI.

The proven setup exposes:

```text
claude-code/claude-sonnet-5-5
claude-code/claude-opus-5-5
```

The integration obtains its model list from the local `claude` CLI, so future model IDs can change.

### DeepSeek

Configure the direct DeepSeek provider only if you want the paid escalation lane:

```text
deepseek/deepseek-flash
```

`paid-fixer` deliberately has no free fallback. If it runs, that should mean the paid escalation actually happened.

---

# Part B — Windows

## 9. Prerequisites on native Windows

This guide targets **native Windows**, not WSL, because the intended UI is OpenChamber Desktop operating on a normal Windows checkout.

### 9.1 Install OpenChamber Desktop

Download the latest Windows build from:

https://github.com/openchamber/openchamber/releases

Install and launch it normally.

Do not configure an external OpenCode server.

### 9.2 Install Node.js

Install a current Node.js release; Node 22+ is a safe baseline.

Open a **new PowerShell** and verify:

```powershell
node --version
npx --version
```

### 9.3 Install the OpenCode 2 terminal CLI

The OMO installer checks for the CLI.

Use npm:

```powershell
npm install -g @opencode/cli
```

Verify:

```powershell
opencode --version
```

OpenCode also publishes standalone Windows binaries if you prefer not to install via npm.

### 9.4 Install Claude Code

Recommended native PowerShell install:

```powershell
irm https://claude.ai/install.ps1 | iex
```

Alternative:

```powershell
winget install Anthropic.ClaudeCode
```

Close and reopen PowerShell, then verify:

```powershell
claude --version
claude auth status
```

Sign in if required:

```powershell
claude auth login --claudeai
```

---

## 10. Prepare the project repository on Windows

Example repository path:

```text
C:\dev\your-project
```

Open PowerShell:

```powershell
Set-Location C:\dev\your-project
New-Item -ItemType Directory -Force .opencode | Out-Null
```

If rebuilding an existing setup, move the current `.opencode` aside first:

```powershell
$stamp = Get-Date -Format "yyyyMMdd-HHmmss"
Move-Item .opencode "$HOME\project-old-opencode-$stamp"
New-Item -ItemType Directory -Force .opencode | Out-Null
```

---

## 11. Install OMO project-locally on Windows

From the repository:

```powershell
Set-Location C:\dev\your-project
$env:OPENCODE_CONFIG_DIR = (Resolve-Path .opencode).Path

npx oh-my-opencode-slim@3.0.2 install `
  --no-tui `
  --skills=yes `
  --background-subagents=no `
  --companion=no

Remove-Item Env:OPENCODE_CONFIG_DIR
```

Expected files:

```text
.opencode\opencode.json
.opencode\tui.json
.opencode\oh-my-opencode-slim.json
```

This one-shot environment variable is only for the installer. Do not set `OPENCODE_CONFIG_DIR` permanently.

---

## 12. Enable OMO background orchestration on Windows

Set the two variables at **User** scope so GUI apps started afterward can inherit them:

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

Verify:

```powershell
[Environment]::GetEnvironmentVariable(
  "OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS",
  "User"
)

[Environment]::GetEnvironmentVariable(
  "OPENCODE_ENABLE_EXA",
  "User"
)
```

Expected:

```text
true
1
```

Fully exit and reopen OpenChamber after changing user environment variables.

Do not set `OPENCODE_CONFIG`, `OPENCODE_HOST`, `OPENCODE_SKIP_START`, or a custom OpenCode server for this setup.

---

## 13. Install the final project orchestration config on Windows

Create:

```powershell
Set-Location C:\dev\your-project
New-Item -ItemType Directory -Force .opencode\oh-my-opencode-slim | Out-Null
```

Copy the supplied files:

```text
config\opencode.json
    -> .opencode\opencode.json

config\oh-my-opencode-slim.json
    -> .opencode\oh-my-opencode-slim.json

config\oh-my-opencode-slim\orchestrator_append.md
    -> .opencode\oh-my-opencode-slim\orchestrator_append.md
```

Keep `.opencode\tui.json` from the installer.

Provider setup in OpenChamber is the same as macOS:

- OpenAI;
- OpenCode/Zen;
- Claude Code integration;
- direct DeepSeek if the paid lane is desired.

---

# Part C — What the final project routing does

## 14. Foreground manager

Normal choice in a new OpenChamber session:

```text
GPT-6.1 Sol
Medium
```

`stripOrchestratorModel: true` means the model selected in OpenChamber remains authoritative for the foreground manager.

If the foreground OpenAI model is unavailable, foreground fallback is manual:

1. Claude Sonnet 5.5;
2. MiMo 2.6 Flash Free.

Delegated OMO agents have their own automatic fallback chains.

---

## 15. Agent roster

Expected visible specialists:

```text
auditor
auditor-low
critical-auditor
critical-planner
designer
explorer
fixer
implementer-alt
librarian
observer
oracle
paid-fixer
planner
planner-low
reviewer
reviewer-low
verification
```

The effective policy is:

### LOW

```text
planner-low
→ reviewer-low when useful
→ fixer / implementer-alt
→ verification
→ auditor-low when warranted
```

Primarily free models.

### MEDIUM

```text
planner
→ bounded implementation
→ verification
→ auditor
```

Serious planning/audit prefers Sol and can fall through to Sonnet.

### HIGH

```text
planner (Sol High)
→ reviewer (Sonnet 5.5)
→ reconciliation/final plan gate
→ bounded implementation
→ verification
→ auditor
```

Use `oracle` only for a genuinely unresolved architecture/debugging disagreement.

### CRITICAL

```text
critical-planner (Opus 5.5)
→ planner (Sol High independent gate)
→ smallest reversible implementation stages
→ verification
→ auditor
→ critical-auditor (Opus 5.5 final gate)
```

### Implementation

Normal implementation should stay on free lanes where practical:

```text
fixer:
Muse → MiMo → Nemotron

implementer-alt:
MiMo → Muse → Nemotron
```

Paid DeepSeek is an explicit escalation:

```text
paid-fixer:
DeepSeek Flash only
```

---

# Part D — Verification

## 16. Restart cleanly

After installing or replacing config files:

### macOS

Quit OpenChamber completely with **Cmd+Q**, reopen it, open the project repository, and create a fresh session.

### Windows

Exit OpenChamber completely from the app/tray, reopen it, open the project repository, and create a fresh session.

Select:

```text
GPT-6.1 Sol
Medium
```

---

## 17. Agent-registration smoke test

Send:

```text
Identify your primary agent and list all specialist agents available to you.
Do not modify any files.
```

Expected specialist roster:

```text
auditor
auditor-low
critical-auditor
critical-planner
designer
explorer
fixer
implementer-alt
librarian
observer
oracle
paid-fixer
planner
planner-low
reviewer
reviewer-low
verification
```

The foreground may describe itself as GPT-6.1 Sol/workflow manager. The critical success criterion is that the OMO specialist roster is registered and dispatchable.

---

## 18. Routing smoke test

Use a harmless read-only test:

```text
Inspect this repository and identify one small, genuinely LOW-risk cleanup opportunity.

Do not modify any files.

Use the configured orchestration policy:
1. classify the work;
2. delegate LOW-risk planning to planner-low;
3. have reviewer-low challenge the plan if the proposal has meaningful ambiguity;
4. report the proposed change;
5. report which specialist agents and actual models/fallbacks were used.
```

The goal is to prove routing, not implementation.

---

# Part E — Troubleshooting

## 19. OpenChamber only shows build/plan and no OMO specialists

First check the project files exist:

```text
<PROJECT>/.opencode/opencode.json
<PROJECT>/.opencode/oh-my-opencode-slim.json
```

Do not assume a global install will be inherited.

Run OMO doctor from the repository.

macOS:

```bash
cd /path/to/your-project
OPENCODE_CONFIG_DIR="$PWD/.opencode" \
npx oh-my-opencode-slim@3.0.2 doctor
```

Windows:

```powershell
Set-Location C:\path\to\your-project
$env:OPENCODE_CONFIG_DIR = (Resolve-Path .opencode).Path
npx oh-my-opencode-slim@3.0.2 doctor
Remove-Item Env:OPENCODE_CONFIG_DIR
```

If OMO's config validation fails, the whole plugin may fail to initialize. Fix the first validation error rather than debugging individual agents.

The supplied config intentionally avoids the `displayName` field because an invalid value there previously prevented OMO from registering any specialists.

---

## 20. Background subagents do not work

Check the two environment variables.

macOS:

```bash
launchctl getenv OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS
launchctl getenv OPENCODE_ENABLE_EXA
```

Windows:

```powershell
[Environment]::GetEnvironmentVariable("OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS","User")
[Environment]::GetEnvironmentVariable("OPENCODE_ENABLE_EXA","User")
```

After changing them, fully restart OpenChamber.

---

## 21. Claude models are missing

Check:

```bash
claude auth status
```

Then re-open:

```text
OpenChamber
→ Settings
→ Integrations
→ Claude Code
```

The integration gets its model list from your local Claude Code CLI. If your account no longer exposes the exact configured Sonnet/Opus IDs, update `.opencode/oh-my-opencode-slim.json` to the IDs shown in OpenChamber.

Do not add temporary custom Claude test agents merely to prove the provider works.

---

## 22. A free Zen model disappears

The free catalog changes over time.

Do not redesign the workflow. Replace only the unavailable model ID in the relevant fallback chain with a current free equivalent.

Keep the role semantics intact:

- cheap research;
- free implementation;
- independent free verification;
- serious gates on Sol/Claude.

---

## 23. After an OpenChamber or OpenCode update

Because OpenChamber 2.x and OpenCode 2.x are evolving quickly, re-run the **agent-registration smoke test** after meaningful upgrades.

Do not immediately rewrite the configuration if a regression appears. First establish whether:

1. project config is still loading;
2. OMO 3.0.2 is still initializing;
3. providers/models still exist;
4. the background-subagent environment is still present.

The working configuration is intentionally pinned and project-local so those layers can be diagnosed separately.

---

# Part F — Files to keep under version control

The project repository can keep these project-specific files:

```text
.opencode/opencode.json
.opencode/oh-my-opencode-slim.json
.opencode/oh-my-opencode-slim/orchestrator_append.md
```

Whether `.opencode/tui.json` should be versioned is a project preference; it is not essential to the orchestration policy.

Do not commit credentials, provider tokens, local Claude authentication, or machine-specific secrets.

---

# Part G — Current working design summary

The purpose of this configuration is not to maximize the number of agents. It is to use stronger models only where they change the quality of the decision.

```text
Foreground coordination
    Sol Medium

LOW
    free planning / review / implementation / audit

MEDIUM
    Sol serious plan
    free implementation
    independent verification
    Sol/Sonnet audit fallback

HIGH
    Sol High plan
    Sonnet independent challenge
    free bounded implementation
    independent verification
    serious audit

CRITICAL
    Opus primary plan
    Sol adversarial gate
    bounded/reversible implementation
    independent verification
    Sol first audit
    Opus mandatory final audit

Rare unresolved deep reasoning
    Astra Oracle

Paid implementation escalation
    DeepSeek Flash only
```

That is the setup to reproduce and preserve.
