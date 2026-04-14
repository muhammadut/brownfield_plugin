# Brownfield Plugin

**Production-grade brownfield development workflow for Claude Code.** Research, plan, execute, verify, and learn — applied to feature work on existing codebases. Multi-repo support. Adversarial review by Codex CLI (with Claude fallback). Persistent learning across workstreams.

> **Status:** v1.0.0 — first public release. Production-tested in private form for several months before public launch. Pair with the [`greenfield`](https://github.com/muhammadut/greenfield_plugin) plugin for new-project planning + scaffolding.

---

## Table of Contents

1. [What is Brownfield?](#what-is-brownfield)
2. [When to use Brownfield (vs Greenfield)](#when-to-use-brownfield-vs-greenfield)
3. [Core philosophy](#core-philosophy)
4. [Installation](#installation)
5. [Quick start](#quick-start)
6. [The 10 commands — full reference](#the-10-commands--full-reference)
7. [The 6 agents — full reference](#the-6-agents--full-reference)
8. [State directory layout](#state-directory-layout)
9. [The workflow loop in detail](#the-workflow-loop-in-detail)
10. [Adversarial review (Codex / Claude)](#adversarial-review-codex--claude)
11. [Multi-repo support](#multi-repo-support)
12. [The learning system](#the-learning-system)
13. [Workstreams & state](#workstreams--state)
14. [Configuration](#configuration)
15. [Comparison with other tools](#comparison-with-other-tools)
16. [Troubleshooting](#troubleshooting)
17. [FAQ](#faq)
18. [Roadmap](#roadmap)
19. [License](#license)

---

## What is Brownfield?

Brownfield is a Claude Code plugin for **doing feature work on existing codebases**. You use it once per feature (or bug, or refactor, or investigation), and it walks you through a research-backed loop:

1. **Init** — index your existing codebase, detect tech stack, identify expert domains
2. **Plan** — research industry best practices, explore the codebase with specialist agents, synthesize a self-contained execution plan, auto-review it
3. **Execute** — implement the plan with phase gates and test checkpoints
4. **Verify** — adversarial review of what was built (Codex or Claude)
5. **Retro** — capture lessons learned for future workstreams
6. **Investigate / Review-Code / Educate** — additional skills for non-feature work (deep code investigation, PR review, codebase teaching)

The 10 user-invocable skills give you a complete brownfield development loop. The 6 specialist agents give you research-grade analysis of your code and your problem domain. The learning system means every feature you ship makes the next one easier.

**The 6 agents in one line each:**

| Agent | What it does |
|---|---|
| **researcher** | Web search for industry best practices, patterns, and pitfalls relevant to your feature |
| **expert-researcher** | Deep dive into one specific expert domain (Azure Service Bus, Redis, GraphQL, etc.) |
| **code-explorer** | Find all relevant code, symbols, and gaps in your repos using semantic exploration |
| **pattern-detector** | Detect codebase conventions and emit a Divergence Report (your patterns vs industry) |
| **security-analyzer** | Trace blast radius, identify security risks, find what else changes |
| **skeptical-reviewer** | Adversarial review of plans or code (independent, evidence-based, brutal) |

**The 10 commands:**

| Command | Purpose |
|---|---|
| `/brownfield:init` | Index the workspace, detect stacks, write config |
| `/brownfield:plan` | Plan a feature: research → analyze → write self-contained plan → auto-review |
| `/brownfield:execute` | Execute an approved plan with phase gates |
| `/brownfield:verify` | Verify what was built against the plan (Codex or Claude) |
| `/brownfield:research` | Research-only mode: produce findings without writing a plan |
| `/brownfield:investigate` | Deep investigation: PRs, tickets, symbols, decisions, root causes |
| `/brownfield:review-code` | Dual-reviewer code review (Architect + Adversary lenses) |
| `/brownfield:retro` | Capture lessons learned from a workstream into the learning system |
| `/brownfield:educate` | Codebase teaching mode: explain how something works |
| `/brownfield:status` | Show all workstreams and where they are |

---

## When to use Brownfield (vs Greenfield)

| Scenario | Use |
|---|---|
| Empty directory, starting a new project | **greenfield** |
| Existing codebase, adding a feature | **brownfield** |
| Existing codebase, fixing a bug | **brownfield** |
| Existing codebase, refactoring | **brownfield** |
| Existing codebase, investigating a PR or symbol | **brownfield** (use `/brownfield:investigate`) |
| Existing codebase, reviewing changes | **brownfield** (use `/brownfield:review-code`) |
| Multi-repo workspace, cross-repo feature | **brownfield** (with multi-repo enabled in init) |
| New microservice in an existing platform | **greenfield** for the service scaffold, then **brownfield** for features |

**Greenfield + Brownfield together = full project lifecycle.** Greenfield handles the once-per-project planning + scaffold. Brownfield handles every feature after that, anchored against the scaffold Greenfield produced.

---

## Core philosophy

**Five principles that drive every design decision:**

1. **Self-contained plans.** Brownfield plans are written so a fresh Claude Code instance with zero conversation context can read `plan.md` and execute it without any prior briefing. This means the plan must contain: current state, target state, before/after code, dependencies, risks, validation steps. Nothing assumed, nothing implicit.

2. **Adversarial review is not optional.** Every plan and every implementation is reviewed by an independent reviewer (Codex CLI by default, Claude skeptical-reviewer as fallback). The reviewer reads actual code, not summaries, and is instructed to find problems rather than validate the author. If they find nothing, they're not looking hard enough.

3. **Research before code analysis.** The researcher agent runs FIRST and produces findings that the code-explorer and pattern-detector consume. This is the AgentCoder principle (test-design separated from coding) applied to the planning loop: research before exploration produces better plans than the reverse.

4. **Pattern divergence is a first-class output.** The pattern-detector doesn't just report what your codebase does — it compares your patterns to industry best practices and emits a Divergence Report. This is where most "AI plans" fall down: they extrapolate from your existing patterns even when those patterns are wrong.

5. **Lessons learned compound.** The learning system (`.brownfield/learning/`) captures patterns and lessons from every workstream, and every subsequent plan reads them as input. This is the difference between an AI that improves over time and one that makes the same mistakes forever.

---

## Installation

### Option 1: Add directly from GitHub

```bash
# Inside Claude Code
/plugin marketplace add muhammadut/brownfield_plugin
/plugin install brownfield@brownfield-plugin
```

### Option 2: Local install for development

```bash
git clone https://github.com/muhammadut/brownfield_plugin.git
cd brownfield_plugin

# In Claude Code
/plugin marketplace add /absolute/path/to/brownfield_plugin
/plugin install brownfield@brownfield-plugin
```

### Verify

```bash
/plugin list
# brownfield should appear with version 1.0.0
```

You should now see 10 slash commands prefixed with `/brownfield:` and 6 agents addressable via the Task tool.

### Optional: Codex CLI for adversarial review

Brownfield uses Codex CLI (OpenAI's CLI agent) as its primary adversarial reviewer for cross-model independence. If Codex is not installed, Brownfield falls back to Claude's built-in skeptical-reviewer agent.

```bash
# macOS
brew install codex

# Or follow https://github.com/openai/codex
# Then authenticate
codex auth login
```

`init` auto-detects Codex availability and configures `review.tool` accordingly.

---

## Quick start

A complete walkthrough of using Brownfield to plan and execute a feature on an existing codebase.

```bash
# 1. cd into your existing project (or parent of multi-repo workspace)
cd ~/projects/my-app

# 2. Open Claude Code
claude
```

In Claude Code:

```
/brownfield:init
```

Brownfield will:
- Detect your platform (macOS / Linux / Windows)
- Detect your workspace shape (single-repo or multi-repo)
- Index every repo it finds
- Detect language, framework, test framework, ORM, runtime per repo
- Detect expert domains from dependency files
- Check for Codex CLI
- Write `.brownfield/config.json`
- Show you a summary

Now plan a feature:

```
/brownfield:plan Add OAuth2 login with Google
```

Brownfield will:
1. Generate a workstream ID (e.g., `oauth2-google-login-20260413`)
2. Ask you to pick the primary repo if it's a multi-repo workspace
3. Trace dependencies to find connected repos
4. Ask 3-5 clarifying questions to scope the feature
5. Triage the task size (LIGHT / MEDIUM / LARGE / DISCUSSION)
6. Spawn research agents in parallel (researcher + expert-researcher per domain)
7. Run codebase analysis agents sequentially (pattern-detector → code-explorer → security-analyzer)
8. Synthesize a self-contained execution plan to `.brownfield/workstreams/<id>/plan.md`
9. Auto-review the plan via Codex (or skeptical-reviewer fallback)
10. Present the plan to you with key risks, review verdict, and approval prompt

Approve the plan, then **clear your context** (Ctrl+L or new session) and:

```
/brownfield:execute
```

Brownfield will execute the plan with phase gates, running tests between phases. When complete:

```
/brownfield:verify
```

Adversarial review of what was built. Codex (or skeptical-reviewer) reads the actual diff and the plan, and reports any drift, gaps, or regressions. Then capture the lessons learned:

```
/brownfield:retro
```

Lessons get written to `.brownfield/learning/` and every future plan reads them as input. The next feature you ship will benefit from this one.

---

## The 10 commands — full reference

### `/brownfield:init`

**Purpose:** Initialize Brownfield in an existing codebase by indexing repos, detecting tech stacks, identifying expert domains, and writing a config file.

**Prerequisites:** None. Works in single-repo or multi-repo workspaces.

**Behavior:**
1. Discover the plugin root, detect platform (macOS / Linux / Windows), check git availability.
2. Check if `.brownfield/config.json` already exists. If so, offer keep / reconfigure / refresh.
3. Detect workspace shape (`single-repo` if cwd has `.git/`, else `multi-repo`).
4. Index repos:
   - Single-repo: index the current directory
   - Multi-repo: scan immediate subdirectories one level deep, index each that contains `.git/`
5. Detect language, framework, test framework, ORM, runtime per repo.
6. Index non-code directories (`wiki/`, `docs/`, `architecture/`, etc.) as knowledge sources.
7. Detect expert domains (azure, aws, gcp, redis, opentelemetry, etc.) from dependency files.
8. Check for Codex CLI; set `review.tool = "codex"` or `"skeptical-reviewer"`.
9. Create `.brownfield/{workstreams, learning, investigations, reviews}/` directories.
10. Write `.brownfield/config.json` and show a summary.

**Outputs:**
- `.brownfield/config.json`
- `.brownfield/learning/codebase-patterns.md` (placeholder)
- `.brownfield/learning/lessons-learned.md` (placeholder)
- Empty `workstreams/`, `investigations/`, `reviews/` directories

### `/brownfield:plan <feature-description> [--light] [--discussion] [--no-questions]`

**Purpose:** Plan a feature with research, codebase analysis, and adversarial review. Produces a self-contained execution plan.

**Arguments:**
- `<feature-description>` — Free-form description (1 sentence to a paragraph)
- `--light` — Skip research and auto-review for small fixes
- `--discussion` — Research only, produce comparison doc, no execution plan
- `--no-questions` / `--skip-clarify` — Skip the Phase 1.8 clarifying questions

**Prerequisites:** `.brownfield/config.json` exists.

**Behavior phases:**

1. **Initialization** — read config, parse args, generate workstream ID, resolve existing workstreams, pick primary repo, trace dependencies, ask clarifying questions
2. **Triage** — classify as LIGHT / MEDIUM / LARGE / DISCUSSION
3. **Research** (MEDIUM/LARGE only) — spawn researcher + expert-researcher agents in parallel
4. **Codebase Analysis** — pattern-detector → code-explorer → security-analyzer (sequential, each reads prior agent's outputs)
5. **Plan Synthesis** — write `plan.md` with: feature request & clarifications, system map, research findings, pattern divergences, lessons from past workstreams, current state, target state, implementation phases with tasks, validation plan, risks
6. **Auto-Review** — Codex reviews the plan against actual code, or skeptical-reviewer if Codex unavailable
7. **Human Gate** — present plan with summary, key risks, review verdict; user approves / revises / rejects

**Outputs:**
- `.brownfield/workstreams/<id>/state.json` — workstream state
- `.brownfield/workstreams/<id>/plan.md` — self-contained execution plan
- `.brownfield/workstreams/<id>/agent-outputs/` — full-fidelity agent reports (researcher, expert-researcher, pattern-detector, code-explorer, security-analyzer)
- `.brownfield/workstreams/<id>/review-decisions.md` — what was accepted/rejected from the auto-review

### `/brownfield:execute [<workstream-id>]`

**Purpose:** Execute an approved plan with phase gates and test checkpoints.

**Prerequisites:** A workstream with `state.phase == "plan-approved"` (or `"building"` to resume).

**Behavior:**
1. Find the active approved workstream (or use the optional `<workstream-id>` argument)
2. Read `plan.md` — the plan is self-contained, so no other context is needed
3. For each phase in the plan:
   - Execute each task in order (CREATE/MODIFY operations) — sub-agents `cd` into the right repo for multi-repo workspaces
   - Run the phase gate (test command) before proceeding
   - If the gate fails, stop and report
4. Update workstream state to `build-complete` and record `build.first_commit`, `build.last_commit`, `build.repos_touched` in `state.json`
5. Suggest `/brownfield:verify` next

### `/brownfield:verify [<workstream-id>]`

**Purpose:** Adversarial review of what was built against the plan. Catches drift, gaps, regressions.

**Prerequisites:** A workstream with `state.phase == "build-complete"` (or `"verified"` to re-verify).

**Behavior:**
1. Read `plan.md` and the diff. Verify reads the commit range from `state.build.first_commit`/`last_commit` (NOT by grepping git log for the workstream id — that would always come back empty). For multi-repo workspaces, it iterates over `state.build.repos_touched` and computes a per-repo diff.
2. Invoke Codex CLI (if `review.tool == "codex"`) to review:
   - Does the implementation match the plan?
   - Are there missing pieces?
   - Are there security issues?
   - Are there regressions in untouched code?
3. If Codex unavailable, invoke `skeptical-reviewer` agent
4. Write verification report to `.brownfield/workstreams/<id>/verification.md`
5. Update workstream state to `verified` (retro is the next step and will archive)

### `/brownfield:research <topic>`

**Purpose:** Research-only mode. Produce findings without writing an execution plan. Useful for exploring options before committing to a feature plan.

**Behavior:**
1. Spawn researcher + expert-researcher agents on the topic
2. Synthesize a research brief
3. Write to `.brownfield/workstreams/<id>/research-preload.md`
4. Tell the user: "Research preloaded. Run `/brownfield:plan` later and the plan skill will use these findings instead of re-running research."

This is how you separate research from planning when you want to think before deciding.

### `/brownfield:investigate <query>`

**Purpose:** Deep investigation of code, PRs, tickets, decisions, or root causes. Produces a written report.

**Sub-flows (auto-detected from query):**

| Query pattern | Sub-flow |
|---|---|
| "look at PR #N" | PR Deep Dive (uses `gh pr view`) |
| "what does ticket #N require" | Ticket Deep Dive (uses `gh issue view`) |
| "explain <repo>" | Repo Brief |
| "where is <symbol> called" | Symbol Trace |
| "how does <pattern> work across repos" | Cross-Repo Pattern Comparison |
| "explain <technology>" | Tech Deep Dive |
| "should we <decision>" | Decision Investigation |
| "why is <symptom>" | Root Cause Analysis |
| Generic / unclear | Generic Investigation |

**Prerequisites:** `.brownfield/config.json` exists. `gh` CLI optional (only for PR/issue sub-flows).

**Outputs:**
- `.brownfield/investigations/<date>-<topic>.md`

### `/brownfield:review-code [--branch <name>] [--commits <range>] [--pr <id>] [--files <paths>] [--workstream <id>]`

**Purpose:** Dual-reviewer code review with Architect + Adversary lenses running in parallel. Synthesizes findings across 7 review dimensions.

**Arguments:**
- `--branch <name>` — Diff between `<name>` and main
- `--commits <range>` — Specific commit range
- `--pr <id>` — Pull Request via GitHub CLI (`gh pr view` / `gh pr diff`)
- `--files <paths>` — Review specific files as-is
- `--workstream <id>` — Review all changes in a workstream against the plan
- (no flag) — Defaults to current branch vs main

**Why two reviewers:** Different perspectives catch different issues. Architect = correctness, fit, integration. Adversary = security, edge cases, failure modes. Running both in parallel produces senior-engineer-level review.

**Prerequisites:** `.brownfield/config.json` exists. `gh` CLI required for `--pr` flag (or use `--branch`/`--commits` for non-GitHub hosts).

**Outputs:**
- `.brownfield/reviews/<date>-<id>/architect-review.md`
- `.brownfield/reviews/<date>-<id>/adversary-review.md`
- `.brownfield/reviews/<date>-<id>/synthesized-report.md`

### `/brownfield:retro [<workstream-id>]`

**Purpose:** Capture lessons learned from a completed workstream into the learning system. Future plans will read these lessons.

**Prerequisites:** A workstream with `state.phase == "verified"` or `"build-complete"` (retro can run before verify if needed).

**Behavior:**
1. Read the workstream's plan, agent outputs, verification report (if present)
2. Identify lessons:
   - What patterns worked and should be repeated?
   - What went wrong and shouldn't be repeated?
   - What was discovered about the codebase that wasn't in the existing patterns file?
3. Append entries to `.brownfield/learning/codebase-patterns.md` and `.brownfield/learning/lessons-learned.md`
4. Update workstream state to `archived` and record `retro_completed_at`

### `/brownfield:educate [topic]`

**Purpose:** Workstream learning mode. Explain the current workstream's findings, decisions, and concepts in plain engineering language so junior developers (or your future self) can understand WHY decisions were made, not just WHAT. Optionally takes a topic to focus the explanation.

**Behavior:**
1. Locate the topic in the codebase (using code-explorer)
2. Read the relevant workstream artifacts (plan, ADRs, agent outputs, verification report)
3. Produce a teaching-style explanation that connects the dots:
   - What was decided, in plain engineering language
   - WHY it was decided that way (which alternatives were rejected and why)
   - What production concerns drove the choice
   - What would have gone wrong with the alternative
   - File:line references to where the decision lives in the code
4. Scale depth to the audience — assume the reader is an engineer who needs to understand the reasoning, not a beginner who needs vocabulary explained

### `/brownfield:status`

**Purpose:** Show all workstreams and where they are. Read-only.

**Behavior:**
1. List every workstream in `.brownfield/workstreams/`
2. For each: show id, feature, current phase, last activity timestamp
3. Highlight stalled workstreams (no activity > 7 days)
4. Show counts by phase

---

## The 6 agents — full reference

Each agent is a specialist subagent invoked by Brownfield skills. Agents read structured inputs, produce structured outputs to specific file paths, and never talk to each other directly — handoff is via files in `.brownfield/workstreams/<id>/agent-outputs/`.

### researcher

| | |
|---|---|
| **Role** | Find industry best practices, patterns, and pitfalls for the planned feature using web search and documentation |
| **Tools** | WebSearch, WebFetch, Read, Write |
| **Inputs** | Feature request, stack info, expert domains, task size, output path |
| **Output** | `agent-outputs/01-researcher.md` |

Produces a structured research brief with:
- Industry best practices for the feature type in this stack
- Recommended methodologies (DDD, CQRS, Saga, etc.) only when they apply
- Common pitfalls and anti-patterns
- Source URLs with credibility scoring (CRAAP framework)
- Conflict detection when sources disagree

Cannot spawn sub-agents.

### expert-researcher

| | |
|---|---|
| **Role** | Deep research into ONE specific expert domain (Azure Service Bus, Redis, GraphQL, etc.) as it relates to the feature |
| **Tools** | WebSearch, WebFetch, Read, Write |
| **Inputs** | Feature request, expert domain, stack info, task size, output path, optional questions |
| **Output** | `agent-outputs/02-expert-researcher-<domain>.md` |

Produces:
- API reference summary for the domain
- Configuration patterns
- Service limits and gotchas
- Real code examples from official docs
- Vendor-neutral comparison templates where applicable
- Documentation extraction with structured format per doc type

Invoked in parallel with the generic researcher when the planning skill detects relevant expert domains from `config.experts`. One invocation per domain.

### code-explorer

| | |
|---|---|
| **Role** | Find all relevant code, symbols, and gaps in the workspace for the planned feature |
| **Tools** | Read, Grep, Glob, Bash, Write |
| **Inputs** | Feature request, primary repo, connected repos, all indexed repos, prior agent outputs, output path |
| **Output** | `agent-outputs/04-code-explorer.md` |

Reads researcher + expert-researcher + pattern-detector outputs FIRST, then explores the primary + connected repos to find:
- Files that need to change
- Symbols (functions, classes, constants) that are involved
- Existing similar features (to model the new feature on)
- Gaps where new code needs to go
- Cross-repo touch points

Only expands beyond primary + connected repos if it discovers a specific dependency during exploration. Stays scoped to avoid noise.

### pattern-detector

| | |
|---|---|
| **Role** | Detect codebase conventions and emit a Divergence Report comparing your patterns to industry best practices |
| **Tools** | Read, Grep, Glob, Bash, Write |
| **Inputs** | Feature request, primary repo, connected repos, all indexed repos, prior researcher outputs, output path |
| **Output** | `agent-outputs/03-pattern-detector.md` |

Reads researcher outputs first. Then scans the codebase for conventions:
- Naming patterns
- Error handling style
- Testing patterns
- Auth / DI patterns
- Layering / module boundaries
- Logging style
- Configuration patterns

Produces a **Divergence Report** for any existing patterns that conflict with industry best practices. For each divergence, recommends migrate / keep / hybrid with reasoning. This is the agent that prevents AI plans from "extrapolating bad patterns forever."

### security-analyzer

| | |
|---|---|
| **Role** | Trace blast radius and identify security risks for the planned feature |
| **Tools** | Read, Grep, Glob, Bash, Write |
| **Inputs** | Feature request, primary repo, connected repos, all indexed repos, prior agent outputs, output path |
| **Output** | `agent-outputs/05-security-analyzer.md` |

Reads ALL prior agent outputs (researcher, expert-researcher, pattern-detector, code-explorer). Then traces:
- Blast radius — what else changes when these files change?
- AuthN / AuthZ implications
- Input validation gaps
- Injection risks (SQL, command, XSS)
- Secrets / credential handling
- PII / sensitive data exposure
- Race conditions / concurrency issues
- Dependency vulnerabilities introduced by new packages

Focuses on the files code-explorer identified and the divergences pattern-detector flagged.

### skeptical-reviewer

| | |
|---|---|
| **Role** | Adversarial review of plans or code, INDEPENDENT of the planning process |
| **Tools** | Read, Grep, Glob, Bash |
| **Inputs** | Plan or code artifact, codebase access, optional questions |
| **Output** | Returned to orchestrator (no file write) |

The skeptical-reviewer is invoked in two contexts:
1. **Codex fallback** — when Codex CLI is unavailable, this agent provides the same adversarial review capability using Claude's own analysis
2. **Always-on for LIGHT plans** — even when Codex is available, lightweight plans skip Codex and use this agent inline

Review lenses:
- **Malicious User** — what would a bad actor do?
- **Careless Colleague** — what mistakes will a tired developer make?
- **Future Maintainer** — will I understand this in 6 months?
- **Ops Engineer** — can I deploy / monitor / debug this?

Independence mandate: the reviewer is told they have NOT been involved in the planning process and must verify every claim by reading actual code. Output format matches Codex review format so the orchestrator processes results identically regardless of which reviewer ran.

---

## State directory layout

Brownfield writes everything to `.brownfield/` inside your target project. Plugin code is separate; only state lives in your project.

```
.brownfield/
├── config.json                    # written by init
├── workstreams/                   # one directory per feature/task
│   ├── oauth2-google-login-20260413/
│   │   ├── state.json             # workstream state machine
│   │   ├── plan.md                # self-contained execution plan
│   │   ├── agent-outputs/         # full-fidelity agent reports
│   │   │   ├── 01-researcher.md
│   │   │   ├── 02-expert-researcher-azure.md
│   │   │   ├── 03-pattern-detector.md
│   │   │   ├── 04-code-explorer.md
│   │   │   └── 05-security-analyzer.md
│   │   ├── review-decisions.md    # what was accepted/rejected from auto-review
│   │   ├── review-raw.md          # raw Codex/skeptical-reviewer output
│   │   ├── verification.md        # post-execute verification report
│   │   └── research-preload.md    # if /brownfield:research was run first
│   └── ... (other workstreams)
├── learning/
│   ├── codebase-patterns.md       # discovered conventions, appended over time
│   └── lessons-learned.md         # postmortems and retros, appended over time
├── investigations/                # written by /brownfield:investigate
│   ├── 2026-04-13-pr-7337.md
│   ├── 2026-04-14-payment-flow.md
│   └── ...
└── reviews/                       # written by /brownfield:review-code
    ├── 2026-04-13-feature-x/
    │   ├── architect-review.md
    │   ├── adversary-review.md
    │   ├── pattern-context.md
    │   └── synthesized-report.md
    └── ...
```

### Should `.brownfield/` be committed to git?

**Recommended:** Yes — for the learning/, plan.md, agent-outputs/, and verification.md files. These are valuable history showing how features were planned and what was learned.

**Optional gitignore:** You may want to gitignore `.brownfield/workstreams/*/agent-outputs/` if they're noisy. But the plan and verification reports should be kept.

---

## The workflow loop in detail

```
                           ┌─────────────────────┐
                           │  Existing codebase  │
                           └──────────┬──────────┘
                                      │
                                      │ /brownfield:init (once)
                                      ▼
                           ┌─────────────────────┐
                           │   Indexed config    │
                           │  .brownfield/       │
                           └──────────┬──────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
              ▼                       ▼                       ▼
   ┌──────────────────┐    ┌─────────────────────┐    ┌──────────────────┐
   │  /investigate    │    │     /plan           │    │   /educate       │
   │  (PR / ticket /  │    │  research →         │    │  (teach me)      │
   │  symbol / etc.)  │    │  analyze →          │    │                  │
   │                  │    │  synthesize →       │    │                  │
   │                  │    │  auto-review →      │    │                  │
   └──────────────────┘    │  present            │    └──────────────────┘
                           └──────────┬──────────┘
                                      │
                                      │ user approves
                                      ▼
                           ┌─────────────────────┐
                           │     /execute        │
                           │  (phase gates,      │
                           │   test checkpoints) │
                           └──────────┬──────────┘
                                      │
                                      ▼
                           ┌─────────────────────┐
                           │     /verify         │
                           │  (Codex / Claude    │
                           │   adversarial)      │
                           └──────────┬──────────┘
                                      │
                                      ▼
                           ┌─────────────────────┐
                           │     /retro          │
                           │  (lessons learned   │
                           │   + archive)        │
                           └──────────┬──────────┘
                                      │
                                      ▼
                           ┌─────────────────────┐
                           │  Updated learning   │
                           │  Future plans       │
                           │  benefit            │
                           └─────────────────────┘
```

Use `/brownfield:status` at any time to see all workstreams.
Use `/brownfield:review-code` independently to review any diff/PR/branch — it doesn't require a workstream.

---

## Adversarial review (Codex / Claude)

Brownfield treats adversarial review as a first-class part of the workflow. Two paths:

### Path A: Codex CLI (preferred)

When `review.tool == "codex"` in config (auto-detected by init), Brownfield invokes Codex CLI in headless mode for plan and code review:

```bash
codex exec "<review prompt>" --full-auto -o <output-path>
```

**Why Codex specifically:** cross-model independence. Claude reviewing Claude is rubber-stamping. Codex reads the same code with a different model's perspective. Catches different classes of issues.

**No timeout on Codex.** Codex can take several minutes for thorough reviews. Brownfield never times out Codex.

### Path B: Skeptical-Reviewer (fallback)

When Codex is unavailable or fails, Brownfield falls back to the built-in `skeptical-reviewer` agent. This is Claude reviewing Claude, but with strong adversarial framing:

- "You have NOT been involved in the planning process."
- "Verify every claim by reading actual code."
- "If you find nothing wrong, you are probably not looking hard enough."
- 4 review lenses: Malicious User / Careless Colleague / Future Maintainer / Ops Engineer

The output format matches Codex format, so the orchestrator processes results identically regardless of which reviewer ran.

### Path C: Codex Fails Mid-Review

If Codex crashes or returns an error (NOT a timeout — never time out Codex), Brownfield falls back to Path B and warns the user.

---

## Multi-repo support

Brownfield is designed for monorepos AND multi-repo workspaces. The init skill detects which shape you're in:

- **Single-repo**: cwd has `.git/`. Index just that repo. `workspace_type = "single-repo"`.
- **Multi-repo**: cwd is a parent of multiple git repos. Index each one level deep. `workspace_type = "multi-repo"`.

For multi-repo workspaces, the planning skill performs **dynamic dependency discovery**:

1. User picks the primary repo for the feature
2. Brownfield reads the primary repo's project files (csproj, package.json, etc.) for cross-repo references
3. Builds a dependency graph one level deep: primary → connected repos
4. Agents focus on primary + connected repos but can search any indexed repo if they discover additional connections

This is how Brownfield handles features that span multiple repos without requiring you to manually specify the scope.

---

## The learning system

Brownfield's `.brownfield/learning/` directory accumulates knowledge from every workstream:

### `codebase-patterns.md`

Discovered conventions, appended by agents as they explore:
- Naming patterns ("API endpoints use kebab-case URLs")
- Error handling style ("All exceptions inherit from AppError")
- Layering rules ("Repositories are the only layer that touches the DB")
- Testing patterns ("Integration tests use testcontainers, not mocks")

The pattern-detector reads this file FIRST when starting analysis on a new workstream. Patterns once discovered are not re-discovered.

### `lessons-learned.md`

Postmortems and retros from past workstreams:
- "Migration X failed because of Y. Always check Z before migrating tables of size > N."
- "PR rejected for not handling concurrent updates. Add optimistic locking to all write paths in the orders module."
- "Codex flagged race condition that Claude missed. When the feature involves background workers, always invoke Codex review."

The plan synthesizer reads this file and applies relevant lessons to every new plan. The "Lessons Applied" section of the plan lists which past lessons informed it.

This is the difference between an AI that improves over time and one that doesn't.

---

## Workstreams & state

Every plan/execute/verify cycle is a **workstream**. Workstreams have a state machine:

```
planning → plan-approved → building → build-complete → verifying → verified → archived
                  │                                            │
                  ▼                                            ▼
             plan-rejected                          (Fix issues, re-run /brownfield:verify)
```

`archived` is reached when `/brownfield:retro` runs. Retro is the final step of the loop and is what writes lessons learned into `.brownfield/learning/`.

Each workstream lives in its own directory under `.brownfield/workstreams/<id>/`. The state is in `state.json`. Multiple workstreams can be active simultaneously — `/brownfield:status` shows them all.

Workstream IDs are auto-generated from feature description + date:
- `add-oauth2-login-20260413`
- `fix-null-ref-order-validator-20260413`
- `event-sourcing-billing-20260413`

You don't pick workstream IDs — they're stable, sortable, and meaningful. The same ID is generated by `/brownfield:research` and `/brownfield:plan` for the same feature description on the same day, which is how the research preload handoff works.

---

## Configuration

`.brownfield/config.json` is the single source of truth for all skills. Written by `init`. Read by every other skill.

Example:

```json
{
  "version": "1.0.0",
  "workspace_type": "multi-repo",
  "workspace_root": ".",
  "paths": {
    "plugin_root": "/Users/me/.claude/plugins/brownfield_plugin",
    "python_cmd": "python3",
    "platform": "macos"
  },
  "index": {
    "repos": [
      {"name": "api-service", "path": "./api-service", "language": "csharp", "framework": "aspnet-core", "test_framework": "xunit", "orm": "ef-core", "runtime": "dotnet8"},
      {"name": "web-app", "path": "./web-app", "language": "typescript", "framework": "next", "test_framework": "vitest", "orm": null, "runtime": "node"}
    ],
    "knowledge_sources": [
      {"name": "wiki", "path": "./wiki", "type": "wiki"},
      {"name": "docs", "path": "./docs", "type": "docs"}
    ],
    "total_repos": 2,
    "languages": {"csharp": 1, "typescript": 1}
  },
  "experts": ["azure", "service-bus", "ef-core", "opentelemetry"],
  "review": {
    "tool": "codex",
    "codex_version": "0.118.0",
    "fallback": "skeptical-reviewer"
  },
  "initialized_at": "2026-04-13T14:30:00Z"
}
```

Re-run `/brownfield:init` any time to refresh the config.

---

## Comparison with other tools

| Tool | What it does | What Brownfield does differently |
|---|---|---|
| **Cursor / Copilot / Cline** | Inline code suggestions and chat-driven edits | Brownfield is structured: it produces a self-contained plan, executes it in phases, and adversarially reviews the result. Cursor is fast for one-line completions; Brownfield is rigorous for multi-file features. |
| **Devin / Cognition** | Autonomous SWE for tasks on existing repos | Brownfield is interactive: every plan goes to a human gate before execution. The plan is committable markdown, not an opaque thought process. |
| **AutoGen / CrewAI / MetaGPT** | Multi-agent frameworks for code generation | Brownfield uses minimal specialist agents (6 total) with epistemic separation (research before exploration, exploration before security, all before plan synthesis). Per AgentCoder/MAST research, this beats sprawling agent rosters. |
| **Aider** | Pair-programmer chat with full repo context | Aider is great for "edit this file." Brownfield is for "design a feature that touches 8 files across 3 repos and survives a Codex review." |
| **Greenfield (sibling plugin)** | Plan + scaffold a new project from scratch | Brownfield assumes a project already exists. Together they form a complete lifecycle: greenfield once, brownfield per feature thereafter. |

---

## Troubleshooting

### "Brownfield isn't configured for this project yet"

Cause: `.brownfield/config.json` doesn't exist in the current working directory.
Fix: Run `/brownfield:init`.

### "No git repos detected"

Cause: You ran init in an empty directory or one with no git repos.
Fix: Either `cd` into an existing repo and re-run, or use the `greenfield` plugin if you're starting from scratch.

### Codex CLI errors during review

Cause: Codex auth expired, Codex out of credit, or Codex API issue.
Fix: Brownfield will fall back to skeptical-reviewer automatically and warn you. To re-enable Codex: `codex auth login`.

### "Plan targets an unconfigured repo"

Cause: A repo referenced in the plan isn't in `config.index.repos`.
Fix: Re-run `/brownfield:init` with the "reconfigure" option to re-scan.

### Multiple active workstreams

Cause: You started multiple workstreams without finishing them.
Fix: `/brownfield:status` to see them all. Use `/brownfield:execute <workstream-id>` to specify which one.

### "PR fetch requires gh CLI"

Cause: You used `--pr <id>` but `gh` CLI is not installed or not authenticated.
Fix: `brew install gh && gh auth login`. Or use `--branch <name>` / `--commits <range>` instead. For non-GitHub hosts (GitLab, Bitbucket, Azure DevOps), use the branch/commits flags.

### Plan looks generic / not specific to my codebase

Cause: pattern-detector and code-explorer didn't have enough to work with.
Fix: Make sure init detected your repos correctly — check `config.index.repos`. Add more detail to your feature description. Use `/brownfield:research` first to do explicit research before planning.

---

## FAQ

**Q: Do I need Codex CLI?**
A: No, but recommended. Without Codex, Brownfield uses the built-in skeptical-reviewer agent (Claude reviewing Claude). This is good but lacks cross-model independence. Codex catches things Claude misses because it's a different model.

**Q: Does Brownfield work with Aider / Continue / other tools?**
A: Brownfield is a Claude Code plugin — it runs inside Claude Code only. The plans it produces are plain markdown and CAN be handed to other tools, but the plugin itself is Claude Code-specific.

**Q: Can I use Brownfield on a private repo?**
A: Yes. Brownfield runs entirely locally. It reads your code, writes plan/state files locally, and never sends your code anywhere except through Claude Code's normal model calls.

**Q: How does Brownfield handle very large codebases?**
A: It's been tested on workspaces with 100+ repos. The init skill is designed to be fast on large workspaces — it scans dependency files, not source files, for domain detection. Pattern-detector and code-explorer scope themselves to primary + connected repos by default.

**Q: What happens if Codex review takes 10 minutes?**
A: Brownfield never times out Codex. Let it run as long as it needs. This is by design — thorough review is more valuable than fast review.

**Q: Can I edit a plan before approving it?**
A: Yes. The plan is markdown in your repo. You can edit `.brownfield/workstreams/<id>/plan.md` directly, then re-present it via the plan skill, or proceed to execute.

**Q: Can I re-run a single agent without re-running the whole plan?**
A: Not directly via slash commands, but agents are addressable by name from the Task tool. You can manually dispatch `brownfield:code-explorer` against an existing context if needed.

**Q: How does Brownfield handle multi-language repos?**
A: The init detects multiple project markers and picks the dominant one (most source files). The full breakdown is reported. Agents use the language info to pick the right test framework, ORM patterns, etc.

**Q: Does Brownfield support non-GitHub hosts?**
A: Yes for the core workflow. The PR-related flags (`--pr`, PR Deep Dive in investigate) require `gh` CLI for GitHub, but `--branch` / `--commits` flags work on any git host.

**Q: How is Brownfield different from the Greenfield plugin?**
A: Greenfield runs ONCE per project, before any code exists. It plans + scaffolds the reference application. Brownfield runs PER FEATURE on existing code, with research/plan/execute/verify/learn loops. Use them together.

**Q: Can I add my own agents?**
A: Yes. Add a markdown file with frontmatter to `agents/`, and reference it from a skill via `subagent_type="brownfield:<your-agent>"`. The plugin is designed for editing.

**Q: What if my workstream goes wrong mid-execute?**
A: Stop, fix the issue, and re-run. The state machine remembers where you were. You can also revise the plan by editing `plan.md` and re-presenting.

**Q: Does the learning system survive `git clean`?**
A: Yes if you commit `.brownfield/learning/` to git. Otherwise it's local-only.

**Q: Can I version-control workstreams?**
A: Yes. `.brownfield/workstreams/<id>/plan.md` and `state.json` are committable. This is recommended — it gives you a reviewable history of every feature plan and how it was reviewed.

---

## Roadmap

### v1.1.0
- Better fallback when `gh` CLI isn't available — try `glab` (GitLab CLI) and other host CLIs
- Workstream archiving and pruning
- Lessons learned export (markdown export of all lessons across workstreams)

### v1.2.0
- Configurable expert domain plugins (ship custom domains for industry verticals)
- IDE integration for browsing workstreams without leaving Claude Code
- Cross-workstream dependency tracking ("workstream X depends on workstream Y being merged first")

### v2.0.0
- Multi-repo PR coordination (atomic deploys across N repos)
- Optional integration with project management tools via plug-in adapters

---

## License

MIT — see [LICENSE](LICENSE).
