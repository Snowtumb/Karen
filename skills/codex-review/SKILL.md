---
name: codex-review
description: >
  Use after creating a PR, completing a feature branch, or when you want a second-AI-perspective
  bug review of current changes. Invokes OpenAI Codex CLI to analyze the diff and codebase for
  bugs, logic errors, security issues, and integration problems. Claude Code evaluates findings,
  filters false positives, auto-fixes confirmed bugs, and runs a verification re-pass.
  Triggers: "codex review", "codex check", "bug check", "second opinion", "cross-model review",
  after PR creation, before merge when extra confidence is needed.
---

# Codex Review Skill

Invoke OpenAI Codex CLI as a second-opinion bug reviewer. Codex analyzes the PR diff with full codebase read access, then Claude Code evaluates each finding, filters false positives, and auto-fixes confirmed bugs. A verification re-pass ensures fixes didn't introduce new issues.

**This skill is rigid** — follow the phases in order. Do not skip the evaluation phase.

---

## Prerequisites Check

Before anything else, verify the environment:

```bash
# 1. Codex CLI installed?
which codex || echo "MISSING"

# 2. Auth configured?
codex login status 2>/dev/null || echo "NO_AUTH"

# 3. Changes exist?
git diff $(git merge-base main HEAD)...HEAD --stat
```

**If Codex not installed:**
> Codex CLI is not installed. Install it with: `npm install -g @openai/codex`
> Then authenticate with: `codex login` (browser) or set `CODEX_API_KEY` env var.

**If no auth:**
> Codex is not authenticated. Run `codex login` to authenticate via browser.

**If no diff:**
> No changes found against the base branch. Nothing to review.

Stop and inform the user if any prerequisite fails. Do not proceed.

---

## Phase 1: Gather Context

Prepare the inputs for Codex:

1. **Detect base branch:**
   ```bash
   # If a PR exists, use its base
   BASE_BRANCH=$(gh pr view --json baseRefName -q '.baseRefName' 2>/dev/null || echo "main")
   ```

2. **Generate diff:**
   ```bash
   MERGE_BASE=$(git merge-base $BASE_BRANCH HEAD)
   git diff $MERGE_BASE...HEAD > /tmp/codex-review-diff.patch
   ```

3. **List changed files:**
   ```bash
   git diff --name-only $MERGE_BASE...HEAD
   ```

4. **Get PR info** (if available):
   ```bash
   PR_TITLE=$(gh pr view --json title -q '.title' 2>/dev/null || echo "N/A")
   PR_DESC=$(gh pr view --json body -q '.body' 2>/dev/null || echo "N/A")
   BRANCH_NAME=$(git branch --show-current)
   ```

5. **Write project context for Codex:**
   Use your existing knowledge of this project from the conversation to write a rich context block. You already know this project — you've been reading files, writing code, and talking with the user. Include:
   - **What the project is** — language, framework, key technologies, architecture
   - **What was built in this PR** — what changed and why (you know this from the conversation)
   - **Key conventions** — anything from CLAUDE.md or patterns you've observed (multi-tenant scoping, validation approach, error handling patterns, etc.)
   - **What Codex should watch for** — based on the stack and conventions, note specific patterns relevant to this project

   This becomes `{PROJECT_CONTEXT}` in the prompt template.

---

## Phase 2: Invoke Codex

Use `codex exec` with `--sandbox read-only` and the custom prompt. The prompt includes the full diff so Codex has complete context.

**Note:** `codex exec review --base` cannot accept a custom prompt (`--base` and `[PROMPT]` are mutually exclusive). We use generic `codex exec` with `--sandbox read-only` instead, which gives Codex full codebase read access plus our custom review instructions.

**Step 1:** Fill the prompt template from `codex-prompt.md` with the gathered context. Include the full diff output (`git diff $MERGE_BASE...HEAD`) and the changed files list in the prompt. Write the filled prompt to a temp file.

```bash
TIMESTAMP=$(date +%s)
PROMPT_FILE="/tmp/codex-review-prompt-${TIMESTAMP}.md"
OUTPUT_FILE="/tmp/codex-review-${TIMESTAMP}.md"

# Write the filled prompt to a temp file (use Write tool)
# The prompt MUST include the full diff content since we're not using --base
```

**Step 2:** Invoke Codex:

```bash
codex exec \
  --sandbox read-only \
  -o "$OUTPUT_FILE" \
  "$(cat $PROMPT_FILE)"
```

**Flags explained:**
- `exec` — non-interactive mode
- `--sandbox read-only` — Codex can read the entire codebase but cannot write files (safe)
- `-o "$OUTPUT_FILE"` — writes final response to file for Claude Code to read
- No `--model` flag — uses the default model from `~/.codex/config.toml`
- Timeout: use the Bash tool's timeout parameter (300000ms) since macOS lacks `timeout` command.

If timeout or error occurs, report to the user and suggest manual review.

**Important:** The custom prompt from `codex-prompt.md` must include the full diff AND the changed files list. Codex can also read surrounding files via `--sandbox read-only` for integration context.

---

## Phase 3: Evaluate Findings

Read the Codex output file and evaluate **every single finding** individually.

**For each finding Codex reports:**

1. **Read the actual code** at the referenced file:line using the Read tool
2. **Verify the issue exists** — does the code actually have this problem?
3. **Check against project patterns** — is this an established pattern used elsewhere?
4. **Cross-reference with intent** — does this conflict with the design decision?
5. **Classify** the finding:

| Classification | Criteria | Action |
|---------------|----------|--------|
| **Confirmed bug** | Issue verified in actual code, would cause runtime failure, security risk, or broken integration | Fix now or later (user chooses) |
| **False positive** | Code doesn't match description, established pattern, internal-only error handling, or out of scope | Dismiss with reasoning |
| **Style/opinion** | Naming, formatting, subjective preferences | Report only, don't fix |

**Refer to `evaluation-guide.md`** for detailed false-positive heuristics.

### Confidence Rule

Only auto-fix findings you classify with **high confidence** — meaning you verified the bug by reading the actual code and confirmed it would cause a real problem. If uncertain, report the finding for user review instead of auto-fixing.

---

## Phase 4: Fix Confirmed Bugs (if user chose "fix now")

**This phase only runs if the user chose to fix now in Phase 6c.** If they chose "fix later", skip to cleanup.

For each confirmed bug:

1. **Read the file** containing the bug
2. **Apply the fix** using the Edit tool
3. **Re-read the fixed code** to verify correctness
4. **Record** what was fixed and why (for the report)

Do NOT commit fixes automatically. The user decides when to commit.

---

## Phase 5: Verification Re-run

After all fixes are applied, run Codex again to verify:

1. **Generate updated diff** (now includes fixes):
   ```bash
   git diff $MERGE_BASE...HEAD > /tmp/codex-review-diff-v2.patch
   ```

2. **Update context summary** to mention the fixes applied

3. **Run Codex again** with the updated prompt:
   ```bash
   TIMESTAMP_V2=$(date +%s)
   PROMPT_FILE_V2="/tmp/codex-review-prompt-v2-${TIMESTAMP_V2}.md"
   OUTPUT_FILE_V2="/tmp/codex-review-v2-${TIMESTAMP_V2}.md"

   # Write updated prompt (with new diff + fix notes) to temp file, then:
   codex exec \
     --sandbox read-only \
     -o "$OUTPUT_FILE_V2" \
     "$(cat $PROMPT_FILE_V2)"
   ```

4. **Evaluate new findings** (same Phase 3 process)

5. **If new confirmed bugs found:** Fix them. This is the LAST pass — no more re-runs.

6. **If clean:** Report "verification pass clean"

**Maximum passes: 2** (initial + 1 verification). No infinite loops.

---

## Phase 6: Report & Persist

### 6a. Present summary to the user

Show the results in the terminal:

```markdown
## Codex Review Results

### Confirmed Bugs (N issues)
1. **[severity]** `file:line` — Short description
   Suggested fix: what should be changed

### Dismissed as False Positive (N issues)
1. `file:line` — "Codex said X"
   Dismissed: reasoning why this is not a real issue

### Style Notes (N items)
1. `file:line` — Suggestion (optional, not auto-fixed)

### Codex Overall Assessment
[Codex's summary/risk assessment from the output]
```

### 6b. Save findings to PR comment

After presenting, **always** post the evaluated findings as a GitHub PR comment so they persist and can be acted on later:

```bash
gh pr comment --body "$(cat findings-report.md)"
```

The comment should include:
- The full evaluated report (confirmed bugs, false positives, style notes)
- For each confirmed bug: file:line, description, and suggested fix
- A header like `## Codex Review (evaluated by Claude Code)` so it's easy to find

This way the findings are attached to the PR — you or Claude Code can come back and fix them without re-running Codex.

### 6c. Ask user: fix now or later?

After posting the PR comment, ask the user:

> "Codex found N confirmed bugs (posted to PR). Fix them now, or come back later?"

- **Fix now** → proceed to Phase 4 (Fix Confirmed Bugs), then Phase 5 (Verification Re-run)
- **Fix later** → skip Phases 4-5, clean up temp files, done. The PR comment has everything needed to fix later.

When fixing later (in a new session), Claude Code can read the PR comment with `gh pr view --comments` to pick up where it left off — no need to re-run Codex.

---

## Cleanup

After reporting, clean up temp files:
```bash
rm -f /tmp/codex-review-*.md /tmp/codex-review-*.patch
```

---

## Error Handling

| Condition | Action |
|-----------|--------|
| `codex` not in PATH | Print: "Install Codex CLI: `npm install -g @openai/codex`" — stop |
| Auth failure | Print: "Run `codex login` or set `CODEX_API_KEY`" -- stop |
| No diff against base | Print: "No changes to review" — stop |
| Timeout (>5 min) | Kill process, print: "Codex timed out. Consider manual review" — stop |
| Codex error exit | Print error output, suggest manual review — stop |
| Empty output file | Print: "Codex returned empty response. Re-run or manual review" — stop |
| All findings are false positives | Report: "Codex found N issues, all evaluated as false positives. Code looks clean." |

---

## What This Skill Does NOT Do

- Replace human code review
- Run tests (use your project's test runner for that)
- Check style/formatting (use linters)
- Auto-commit or auto-push
- Run more than 2 passes
- Modify files outside the working tree
