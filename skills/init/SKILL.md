---
name: init
description: Initialize Brownfield in an existing codebase. Detects platform, scans local repos (single or multi-repo workspace), identifies tech stacks and expert domains, checks review tools, and writes a complete config.
user-invocable: true
---

# Brownfield Init — Workspace Indexing

You are the Brownfield initialization orchestrator. Your job is to:

1. **Discover the plugin** — find the plugin root, detect OS, check git
2. **Index every repo** in the workspace (single-repo or multi-repo), detect tech stacks
3. **Identify expert domains** from dependency files
4. **Check for review tools** (Codex CLI optional; falls back to skeptical-reviewer)
5. **Write a complete config** with all discovered paths

The expected flow is: user opens Claude Code in their working directory, runs `/brownfield:init`, and Brownfield indexes whatever is already there.

> **Note:** Brownfield is for **existing codebases**. If you're starting from an empty directory and need to plan + scaffold a new project from scratch, use the `greenfield` plugin instead.

Do NOT use AskUserQuestion — just ask questions via normal text output and wait for a response.

## Process

### Step 1: Discover Plugin & Environment

This step runs silently — no user interaction.

**1a. Find the plugin root directory:**

This SKILL.md is at `skills/init/SKILL.md` inside the plugin. The plugin root is two directories up. Use Glob to confirm and store the absolute path.

**1b. Detect the platform:**

```bash
uname -s 2>/dev/null || echo Windows
```

- `Darwin` → macOS
- `Linux` → Linux
- `MINGW*`, `MSYS*`, `CYGWIN*`, or failure → Windows

Store as `platform` in config (`macos`, `linux`, or `windows`).

**1c. Find Python (optional):**

Some sub-tools may use Python. Check availability but don't block on it:

```bash
python3 --version 2>/dev/null
python --version 2>/dev/null
```

Store the working command (e.g., `python3` or `python`) or `null` if neither is found.

**1d. Check for git:**

```bash
git --version 2>/dev/null
```

Git is required for repo operations. If not found, warn the user:
> "git is not installed. Brownfield needs git for diff-based operations (review-code, plan, execute). Install git and re-run init."

### Step 2: Check Existing Configuration

Read `.brownfield/config.json` to check if Brownfield is already initialized.

- **If config.json exists (re-run scenario):**
  1. Verify that `paths.plugin_root` from the stored config still exists. If the plugin moved, re-discover it in Step 1a.
  2. Show a summary of the current config (workspace type, repo count, language breakdown, experts, review tool).
  3. Ask:
     > "Brownfield is already configured (v{version}). Do you want to:
     > 1. Keep current configuration
     > 2. Reconfigure from scratch (re-scan all repos and experts)
     > 3. Refresh — keep expert domains, re-scan repo index only"

  If keep, stop here and show the config summary.
  If reconfigure, continue from Step 3.
  If refresh, skip to Step 3.

- **If config.json does NOT exist (first-time setup):** Continue to Step 3.

### Step 3: Index All Repos

Brownfield supports two workspace shapes:

- **Single-repo**: Claude Code was opened inside a git repo. The current directory is the only repo.
- **Multi-repo**: Claude Code was opened in a parent directory containing multiple git repos as immediate subdirectories.

Detect which shape you're in:

```bash
# If current dir has .git, it's a single-repo workspace
test -d .git && echo single-repo || echo multi-repo
```

**Single-repo path:** Index the current directory as the one and only repo. Skip to language detection below.

**Multi-repo path:** Scan immediate subdirectories of the current directory. Only **one level deep** — do not recurse. For each subdirectory that contains `.git/`, index it as a repo.

Also check for nested patterns like `repos/`, `services/`, or `apps/` as one-level-deep containers — if a subdirectory of the cwd contains many sub-repos, those are valid index targets too.

For each repo found, detect language and framework from project markers:

| File Pattern | Language | Framework Hint |
|---|---|---|
| `package.json` | TypeScript/JavaScript | Read for `dependencies`: express, next, fastify, nest, vite, react |
| `tsconfig.json` | TypeScript | Confirms TS over JS |
| `*.csproj`, `*.sln` | C# | Read csproj for ASP.NET Core, Blazor, classlib |
| `pyproject.toml`, `setup.py`, `requirements.txt` | Python | Read for django, fastapi, flask |
| `go.mod` | Go | Read for gin, echo, fiber, chi |
| `Cargo.toml` | Rust | Read for actix, axum, rocket |
| `Gemfile` | Ruby | Read for rails, sinatra |
| `pom.xml`, `build.gradle*` | Java/Kotlin | Read for spring-boot, micronaut, quarkus |
| `composer.json` | PHP | Read for laravel, symfony |
| `mix.exs` | Elixir | Read for phoenix |

For each repo, detect:
- **Language** — primary, based on project markers
- **Framework** — from dependency files
- **Test framework** — look for `jest.config.*`, `vitest.config.*`, `pytest.ini`, `conftest.py`, xunit/nunit references in csproj, `*_test.go`
- **ORM** — look for Entity Framework, Prisma, Sequelize, SQLAlchemy, GORM, ActiveRecord, Diesel
- **Runtime** — node, deno, bun, dotnet, python, go, etc.

Also index **non-code directories** with documentation as **knowledge sources**:
- `wiki/` or `wikis/` — markdown wiki content
- `docs/` — documentation
- `architecture/`, `adr/`, `decisions/` — design records
- Any subdirectory with mostly markdown files and no project markers

Present a summary:

> "Found **N** repos (X TypeScript, Y Python, Z Go) and **M** knowledge sources"
>
> Repos: {repo names}
> Knowledge: {knowledge source names or "none"}

Do NOT ask the user to confirm or describe each repo. Just report what was found.

**If NO repos are found:**

> "No git repos detected in the current directory or one level down. Brownfield is for existing codebases — if you're starting from scratch, the `greenfield` plugin is a better fit. Otherwise, run init from inside a git repo or a parent directory containing one."

### Step 4: Expert Domain Detection

Scan dependency files across indexed repos using Grep — fast, no source-file reading required for 100+ repos.

**Strategy:**

1. **Check project/dependency files first** (fast) — `*.csproj`, `package.json`, `requirements.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `Gemfile`, `composer.json`, `docker-compose.yml`, `*.tf` files across all repos.
2. **Spot-check source files only if needed** — for things not visible in dependency files (e.g., APIM policy XML, raw SQL files).
3. **Do NOT read every source file** — too slow at scale.

Domain detection patterns (cloud-agnostic):

| Pattern | Expert Domain |
|---|---|
| `Azure.*` NuGet packages, `@azure/*` npm packages, `azure-*` pip packages | `azure` |
| `aws-sdk`, `boto3`, `Amazon.*` NuGet, `@aws-sdk/*` | `aws` |
| `@google-cloud/*`, `google-cloud-*`, GCP SDK imports | `gcp` |
| `Microsoft.Azure.ServiceBus`, `Azure.Messaging.ServiceBus` | `service-bus` |
| `StackExchange.Redis`, `redis` (npm), `redis-py` | `redis` |
| `Microsoft.EntityFrameworkCore`, `prisma`, `sequelize`, `sqlalchemy`, `gorm` | `orm` |
| `Swashbuckle`, `NSwag`, `swagger`, OpenAPI files | `openapi` |
| `Dockerfile`, `docker-compose.yml` | `docker` |
| `terraform`, `*.tf` files | `terraform` |
| `bicep`, `*.bicep` | `bicep` |
| `pulumi`, `@pulumi/*` | `pulumi` |
| `MediatR`, `cqrs`, `eventstore` | `cqrs` |
| `Serilog`, `NLog`, `pino`, `winston`, `structlog` | `structured-logging` |
| `OpenTelemetry`, `@opentelemetry/*` | `opentelemetry` |
| `GraphQL`, `HotChocolate`, `apollo-server` | `graphql` |
| `RabbitMQ`, `MassTransit`, `kafkajs`, `@nestjs/microservices` | `messaging` |
| `SignalR`, `socket.io` | `realtime` |
| Kubernetes manifests, Helm charts | `kubernetes` |

Present detected domains:

> "Detected expert domains: **azure**, **service-bus**, **opentelemetry**, **terraform**
>
> These domains drive the expert-researcher agent during planning. It pulls in specialized knowledge for each domain.
>
> Add more? (comma-separated, or press Enter to confirm)"

### Step 5: Review Tool Detection

Check if Codex CLI is installed:

```bash
codex --version 2>/dev/null
```

- If success: store the version, set `review.tool = "codex"`.
- If failure: set `review.tool = "skeptical-reviewer"`, note fallback.

If Codex is found, extract the version from the output. Store as `codex_version`.

**Do NOT check for OPENAI_API_KEY.** Codex CLI uses its own authentication (`codex auth login`). If installed and on PATH, assume authenticated. Brownfield calls Codex headlessly via `codex exec --full-auto`.

If Codex is not found:
> "Codex CLI not detected. Brownfield will use the built-in skeptical-reviewer agent (Claude reviews its own work via adversarial prompting). For cross-model review, install Codex CLI."

Always set `review.fallback = "skeptical-reviewer"`.

### Step 6: Create Directory Structure

Create:

```
.brownfield/
  config.json                         (written in Step 7)
  workstreams/                        (empty — for /brownfield:plan workstreams)
  learning/
    codebase-patterns.md              (placeholder)
    lessons-learned.md                (placeholder)
  investigations/                     (empty — for /brownfield:investigate)
  reviews/                            (empty — for /brownfield:review-code)
```

Write `codebase-patterns.md`:

```markdown
# Codebase Patterns

Conventions and patterns discovered during workstreams. Referenced by agents during planning and building.

> Auto-populated by Brownfield agents as they discover patterns in your codebase.
> Do not delete — agents append here and reference during planning.
```

(The header text matches what `retro` writes when creating the file from scratch — keeping them aligned prevents the first retro from creating a competing header section.)

Write `lessons-learned.md`:

```markdown
# Lessons Learned

> Auto-populated by Brownfield agents when builds fail, reviews catch issues, or plans need revision.
> Do not delete — agents reference past lessons to avoid repeating mistakes.
```

### Step 7: Write Config

Write `.brownfield/config.json` with all gathered data. Use the current timestamp for `initialized_at`.

```json
{
  "version": "1.0.0",
  "workspace_type": "multi-repo",
  "workspace_root": ".",
  "paths": {
    "plugin_root": "/path/to/brownfield_plugin",
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

Notes on config values:
- `workspace_type`: `single-repo` if current dir has `.git/`, otherwise `multi-repo`.
- `workspace_root`: always `.` (the directory where Claude Code was launched).
- `paths.plugin_root`: absolute path to the Brownfield plugin installation directory.
- `paths.python_cmd`: the Python command that works on this system, or `null`.
- `paths.platform`: `macos`, `linux`, or `windows`.
- `index.repos`: FLAT list of ALL discovered repos. Roles are discovered dynamically during planning.
- `index.knowledge_sources`: non-code directories with documentation.
- `index.total_repos`: convenience count.
- `index.languages`: convenience breakdown.
- `experts`: flat array of domain strings consumed by the expert-researcher agent.
- `review.tool`: `codex` if Codex CLI detected, else `skeptical-reviewer`.
- `review.codex_version`: present only if Codex detected.
- `review.fallback`: always `skeptical-reviewer`.
- `initialized_at`: ISO 8601 UTC timestamp.

### Step 8: Display Summary

```
+======================================================+
|              Brownfield v1.0 Initialized              |
+======================================================+
| Workspace: /path/to/workspace (multi-repo)            |
| Repos indexed: 2                                      |
|   C#: 1 | TypeScript: 1                               |
| Knowledge sources: wiki, docs                         |
| Experts: azure, service-bus, ef-core, opentelemetry   |
| Review: Codex CLI (v0.118.0)                          |
| Learning: .brownfield/learning/ (empty, will grow)    |
+======================================================+
```

Adapt every line to actual values. Then suggest the next step:

> Start planning: `/brownfield:plan <describe your feature>`

## Edge Case Reference

| Situation | Handling |
|---|---|
| Empty directory, no git repos | "No git repos found. Use the `greenfield` plugin instead, or `cd` into an existing repo and re-run." |
| Single repo (cwd has .git) | Index just that repo. Set `workspace_type = "single-repo"`. |
| Multi-repo (cwd has subdirs with .git) | Index all subdirs, one level deep. |
| Mixed languages | Expected — report the full breakdown. |
| 200+ repos | Still index them all. Show summary counts, do not list every repo. |
| Codex CLI installed | Set `review.tool = "codex"`. Codex uses its own auth — no OPENAI_API_KEY check. |
| Multiple project markers in one repo (e.g., package.json AND *.csproj) | Pick the dominant one (most source files), or list both languages. |
| `.brownfield/` directory exists but no config.json | Treat as fresh init — proceed from Step 3. |
| Read permissions error on a subdirectory | Skip it, note in output: "Skipped {dir} (permission denied)". |
| No test framework detected in a repo | Set `test_framework` to `null`. Do not ask. |
| Nested repos (subdirectory contains another subdirectory with repos) | Only scan ONE level deep from cwd. Recursion is opt-in via re-running init from a deeper directory. |
| Wiki or docs directory | Index as a knowledge source, not a code repo. |

## Important Notes

- Do NOT use AskUserQuestion — just ask questions via text output
- Auto-detect as much as possible to minimize user input
- The config file is the single source of truth for all other Brownfield skills
- The role of each repo is NOT defined here — it is discovered dynamically during `/brownfield:plan`
- If detection fails for any field, set it to `null` — do not block init
- Brownfield is for **existing codebases**. For empty directories or new projects, use the `greenfield` plugin
- Knowledge source directories (wiki/, docs/) are indexed as references, not as code repos
- The plugin is cloud-agnostic — domain detection includes Azure, AWS, GCP, and self-hosted patterns
