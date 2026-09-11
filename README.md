# opsman

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A local-first **meta-agent orchestrator** for Dev and Ops tasks, for
[Claude Code](https://docs.anthropic.com/en/docs/claude-code), Codex, Gemini
CLI, Copilot CLI, and any agent that supports the
[Agent Skills](https://agentskills.io) standard.

Distributed as a single-plugin [Claude Code plugin marketplace](https://docs.anthropic.com/en/docs/claude-code)
and as a standard Agent Skill.

## Main goal

Most agent sessions hold their plan, progress, and conclusions in
conversation memory: close the window and the work evaporates. Opsman moves
all of that into portable artifacts under `.opsman/` in your repository
(gitignored), and puts a POSIX shell kernel between the agent and that state.

The core stance: **agents reason, the shell proves.**

- The agent interprets your task, selects skills, plans, implements, and
  reviews.
- The `opsman` kernel owns discovery, the state machine, evidence capture,
  test execution, budgets, and validation. Every state change is a typed
  event appended to a journal; illegal transitions are refused mechanically.
- Completion is gated twice: a deterministic acceptance suite must pass
  (Layer A), and an independent Oracle role must approve against a scoring
  rubric — and the kernel re-checks the mechanical blockers, so an Oracle
  cannot approve past a failing test.

Because all state lives on disk rather than in context, a run survives
crashes, new sessions, and even switching between Claude Code and Codex
mid-task.

## How a run flows

```text
DISCOVERING → UNDERSTANDING → SELECTING → PLANNING → TEST_DESIGN
→ IMPLEMENTING → VALIDATING → JUDGING → COMPLETED

side states: DIAGNOSING, REPLANNING, WAITING_APPROVAL, BLOCKED, ABANDONED
```

Each state is owned by a role (discoverer, analyst, selector, planner,
implementer, verifier, critic, oracle). The kernel renders each role a
**context packet** containing only what that role is entitled to see.
Red-before-green is enforced by the state machine: you cannot enter
`IMPLEMENTING` until an executable acceptance check exists and a recorded
baseline proves it currently fails.

See [`plugins/opsman/README.md`](plugins/opsman/README.md) for the full
walkthrough — workspace modes, parallel steps, the interview gate, budgets,
and an annotated example run.

---

## Installation

### Method 1 — Claude Code (slash commands, recommended)

**Step 1 — Add the marketplace** (one-time per machine):

```text
/plugin marketplace add glapsfun/opsman
```

This registers the marketplace under the alias **`opsman`** from the `name`
field in `.claude-plugin/marketplace.json`.

**Step 2 — Install the plugin**:

```text
/plugin install opsman@opsman
```

To update after a new version is published:

```text
/plugin marketplace update opsman
```

To remove:

```text
/plugin remove opsman
```

### Method 2 — Claude Code CLI (non-interactive)

```bash
claude plugin marketplace add glapsfun/opsman
claude plugin install opsman@opsman
```

> **Note:** `claude "/plugin ..."` does **not** work — that form passes the
> string as a model prompt, not as a plugin command. Use `claude plugin ...`
> (no leading slash) for non-interactive use.

### Method 3 — Agent Skill (`npx skills`)

Installs the `plugins/opsman/skills/opsman/` folder into the target agent's
global skills directory:

```bash
npx skills add glapsfun/opsman --skill opsman --agent codex --global -y
npx skills add glapsfun/opsman --skill opsman --agent gemini-cli --global -y
npx skills add glapsfun/opsman --skill opsman --agent copilot --global -y
```

Omit `--global` to install into the current project instead. Verify with
`npx skills list -a codex`, and restart the agent afterwards so the new skill
metadata is loaded.

> **Note:** `npx skills` implements the Agent Skills standard and copies
> **only** the skill folder. The plugin-level `commands/` directory is not
> part of that standard, so the `/opsman`, `/opsman-status`, `/opsman-resume`
> and `/opsman-validate` slash commands come only from the plugin install
> (Method 1 or 2). The skill itself is fully functional standalone — the
> kernel and the orchestration protocol are entirely inside it.

### Method 4 — Local / development install

```bash
git clone https://github.com/glapsfun/opsman.git
```

Then, inside Claude Code, using the absolute path to your clone:

```text
/plugin marketplace add /path/to/opsman
/plugin install opsman@opsman
```

The path must point to the repo root (the directory containing
`.claude-plugin/marketplace.json`).

### Requirements

`git` and `jq` on PATH — the kernel fails fast with exit 7 if either is
missing. Runs must start inside a git repository. The optional `opsman board`
live viewer additionally needs `python3`.

---

## Usage

Start a run with the slash command:

```text
/opsman migrate the ingress manifests in ./deploy to Gateway API
```

Check progress, resume after a crash or a tool switch, and re-run the
acceptance checks on demand:

```text
/opsman-status
/opsman-resume
/opsman-validate
```

Every run declares a workspace mode with `--base branch|current|worktree`,
and budgets are set at start and only at start. Full details, including the
live board and past-run browsing, are in
[`plugins/opsman/README.md`](plugins/opsman/README.md).

## Safety model

- Every plan step declares a **risk class R0–R4**; the kernel refuses steps
  above the run's auto-approval ceiling (default R2).
- A deny-pattern policy (`kubectl apply`, `terraform apply`, force-push,
  credential/IAM changes, resource deletion, …) escalates a step's
  *effective* risk regardless of what the plan declared.
- R3/R4 steps park the run in `WAITING_APPROVAL`; the human's approval is
  recorded as a typed `ApprovalGranted` event (who/what/when), so the audit
  trail survives tool switches.
- **Write scope** — plan steps declare `allowed_files` glob patterns; the
  kernel fails straying steps and refuses `ImplementationCompleted` while
  out-of-scope worktree changes exist.
- Implementation is confined to the run's worktree; the main tree is the
  control plane. **Opsman never pushes.**

## Repository layout

```text
.claude-plugin/marketplace.json   # marketplace manifest (one plugin)
.agents/plugins/marketplace.json  # Codex marketplace manifest
plugins/opsman/
├── .claude-plugin/plugin.json    # Claude Code plugin manifest
├── .codex-plugin/plugin.json     # Codex plugin manifest
├── commands/                     # /opsman /opsman-resume /opsman-status
│                                 #   /opsman-validate
└── skills/opsman/
    ├── SKILL.md                  # the orchestration protocol the agent follows
    ├── agents/                   # role prompt templates (analyst … oracle)
    ├── base-skills/              # built-in fallback team: scout, developer,
    │                             #   reviewer, operator
    ├── scripts/                  # POSIX sh kernel: opsman dispatcher, ~20
    │   └── board/                #   scripts, lib/ — and the live-board UI
    ├── schemas/                  # JSON Schemas for state, events, plan,
    │                             #   evidence, verdicts
    ├── references/               # architecture, state machine, safety policy,
    │                             #   artifact contract
    ├── tests/                    # plain-sh unit tests (t-*.sh), no framework
    └── evals/evals.json          # agent-behavior scenarios
scripts/                          # repo tooling: CI checks
```

## Development

```bash
scripts/validate.sh --fast   # structure, marketplace sync, manifest versions,
                             # JSON, YAML, shell syntax
scripts/lint.sh              # shellcheck + shfmt
scripts/test.sh              # eval schema validation + the opsman kernel tests
scripts/security.sh          # gitleaks secret scan
scripts/check.sh --all       # everything above, as CI runs it
```

The kernel's own suite is plain POSIX sh and needs no framework:

```bash
sh plugins/opsman/skills/opsman/tests/run.sh
```

## License

[MIT](LICENSE)
