---
name: verify
description: Adversarial code verification with Codex CLI or Claude skeptical-reviewer fallback.
user-invocable: true
argument-hint: [workstream-name]
---

# Brownfield Verify v1.0 — Adversarial Code Verification Orchestrator

You are the Brownfield verification orchestrator. Your job is to get an independent adversarial review of the BUILT CODE from Codex CLI (primary) or the Claude skeptical-reviewer fallback. The plan IS the spec. You run inline in the current conversation.

## Phase 1: State Validation

### 1.1 Read Configuration

Read `.brownfield/config.json`. If missing:
> "Brownfield isn't configured. Run `/brownfield:init` first."

### 1.2 Resolve Workstream

Standard resolution priority (same as other skills).

### 1.3 Validate Phase

Read `state.json`. Phase must be `build-complete` or `verified` (re-verify).

- If earlier: Guide to correct next step
- If `archived`: "This workstream is already complete and archived."
- If `build-complete` or `verified`: proceed

Update state to `verifying`.

## Phase 2: Gather Verification Context

Collect everything the verifier needs:

1. **Plan:** `.brownfield/workstreams/<id>/plan.md` — this is the spec, the single source of truth for what was supposed to be built
2. **Workstream commit range:** read `state.json` and use the `build.first_commit` and `build.last_commit` fields that `execute` recorded in Phase 6.3. Do NOT try to grep `git log` for the workstream ID — execute does not embed it in commit messages.
3. **Repos touched:** read `state.build.repos_touched` (a JSON array of repo paths). For single-repo workspaces this is `["."]`. For multi-repo it lists every repo that received a commit during the build.
4. **Git diff per repo:** for each repo in `repos_touched`, compute the diff between `build.first_commit~1` and `build.last_commit` from inside that repo.
5. **Test results:** run the test suite (command from `plan.md` Validation Plan section) and capture output.

```bash
# Pseudocode — actual execution should use the Read tool to load state.json
FIRST_COMMIT=$(jq -r '.build.first_commit' .brownfield/workstreams/<id>/state.json)
LAST_COMMIT=$(jq -r '.build.last_commit' .brownfield/workstreams/<id>/state.json)

# For each repo in state.build.repos_touched:
for REPO in $(jq -r '.build.repos_touched[]' .brownfield/workstreams/<id>/state.json); do
  ( cd "$REPO" && git diff "${FIRST_COMMIT}~1..${LAST_COMMIT}" )
done

# Run validation tests (commands from plan.md Validation Plan section)
```

If `state.build.first_commit` is missing (e.g., the workstream was built by an older version), fall back to asking the user to specify the commit range manually:
> "I can't find a recorded commit range in state.json. What commit range should I review? (e.g., `HEAD~5..HEAD`)"

**Auto-suggestion for LARGE workstreams:** If the git diff exceeds 2000 lines or touches more than 20 files, warn the user:
> "This is a large workstream. Verification will take longer and may benefit from being split. Proceeding anyway."

## Phase 3: Build Verification Prompt

Assemble the verification prompt:

```markdown
You are performing adversarial code verification.

## Implementation Plan (what was supposed to be built):
$(cat .brownfield/workstreams/<id>/plan.md)

## Actual Code Changes:
$(git diff <first-workstream-commit>~1..HEAD)

## Test Results:
$(<test command from plan>)

## Your Task:
1. **Read the "Feature Request & Clarifications" section at the top of plan.md carefully** — this is the authoritative user intent. Verify the code actually delivers the clarified scope, not just what's literally in the tasks.
2. Read the actual source files, not just the diff
3. Verify each task was implemented correctly
4. Verify NO scope violations — nothing was added that the user marked "out of scope"
5. Verify success criteria from Clarifications section are met
6. Check for security issues not in the plan
7. Check test quality — are tests meaningful?
8. Check for regressions in existing functionality

## Output:
### Verdict: PASS | PASS WITH NOTES | NEEDS FIXES | FAIL
### Scope Adherence: (Did the implementation stay within clarified scope? Any violations?)
### Success Criteria Met: (From Clarifications section — yes/no with evidence)
### Task Verification: (for each task: verified or issue)
### Issues Found: (severity, description, file:line, suggestion)
### Security Check: PASS or CONCERNS with details
```

Write the assembled prompt to `.brownfield/workstreams/<id>/codex-verify-prompt.md`.

## Phase 4: Execute Verification

### Path A: Codex Available

Check if `codex` is on PATH.

```bash
codex exec "$(cat .brownfield/workstreams/<id>/codex-verify-prompt.md)" --full-auto -o .brownfield/workstreams/<id>/verification.md
```

Do NOT set a timeout. Let Codex run as long as it needs. Codex will explore the codebase, read source files, and produce its verdict.

### Path B: Codex Unavailable

If `codex` is not found on PATH, fall back immediately. Warn the user:

> "Codex CLI not available. Falling back to Claude skeptical-reviewer (single-model mode)."

Spawn the skeptical reviewer:

```
Agent(
  subagent_type="brownfield:skeptical-reviewer",
  description="Code Verification: <feature name>",
  prompt=<verification prompt from codex-verify-prompt.md>
)
```

Write result to `.brownfield/workstreams/<id>/verification.md`.

### Path C: Codex Fails or Crashes

If the `codex exec` command exits with a non-zero status or produces no output, fall back to Path B. Warn the user:

> "Codex crashed or failed (exit code: <code>). Falling back to Claude skeptical-reviewer."

Then proceed exactly as Path B.

## Phase 5: Human Gate (Final)

Read `.brownfield/workstreams/<id>/verification.md` and present results:

> "**Verification complete.**
>
> Reviewer: <Codex CLI / Claude Skeptical Reviewer>
> Verdict: **<verdict>**
>
> **Tasks: <N>/<N> verified**
> **Issues found: <N>** <one-line summary of each with severity, if any>
> **Security: <PASS/CONCERNS>**
>
> Full verification: `.brownfield/workstreams/<id>/verification.md`
>
> **What would you like to do?**
> 1. **Ship it** — mark as verified, run retro to capture lessons
> 2. **Fix issues** — address the findings and re-verify
> 3. **Accept as-is** — acknowledge issues but mark as verified anyway"

On **Ship it** or **Accept as-is**:
- Update state to `verified` (NOT `archived` — retro is the one that archives, so it can still find this workstream)
- Append to history with timestamp
- Print:
> "Workstream **<id>** marked as **verified**. All artifacts preserved in `.brownfield/workstreams/<id>/`.
> Next: run `/brownfield:retro` to capture lessons learned. The retro will archive this workstream once it's done."

On **Fix issues**: Keep state at `build-complete`, user fixes and re-runs `/brownfield:verify`.

## Important Notes

- This reviews CODE, not the plan — focus on what was actually built
- The plan.md is the spec — it defines what "correct" means
- The git diff should capture all changes from the workstream, not just the last commit
- Do NOT set a timeout on Codex — let it run to completion
- If Codex's verification is clearly wrong (hallucinated files, etc.), note it to the user
- Archiving preserves all artifacts — workstreams are a permanent record
