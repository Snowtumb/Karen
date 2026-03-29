# Karen

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Cross-model AI code review plugin for [Claude Code](https://claude.ai/code)** — uses OpenAI Codex CLI as a second-opinion bug reviewer.

Karen invokes Codex to analyze your PR diff with full codebase read access, then Claude Code evaluates every finding, filters out false positives, and auto-fixes confirmed bugs. A verification re-pass ensures fixes didn't introduce new issues.

---

## How It Works

Karen runs a **6-phase workflow** every time you invoke it:

1. **Gather Context** — Detects base branch, generates diff, collects PR info. Claude Code writes a rich project context from its session knowledge (your stack, what you built, conventions).
2. **Invoke Codex** — Sends the diff + project context to Codex CLI via `codex exec --sandbox read-only`. Codex reviews with full codebase read access.
3. **Evaluate Findings** — Claude Code reads the actual code at every flagged location. Each finding is classified as confirmed bug, false positive, or style opinion using a decision tree.
4. **Fix Confirmed Bugs** — If you choose "fix now", Claude Code applies fixes. Only high-confidence fixes are auto-applied.
5. **Verification Re-run** — Codex runs again on the updated diff to catch regressions. Max 2 passes total.
6. **Report & Persist** — Results shown in terminal and posted as a PR comment for future reference.

## What It Catches

| Category | Priority | Examples |
|----------|----------|---------|
| **Bugs** | Critical | Null dereferences, missing imports, off-by-one errors, race conditions |
| **Logic Errors** | High | Wrong comparisons, inverted conditions, stale closures, swallowed errors |
| **Security Issues** | High | XSS, injection, missing auth checks, exposed secrets |
| **Integration Problems** | Medium | Wrong argument types, missing fields, broken contracts |
| **Boundary Errors** | Medium | Missing error handling on network calls, unvalidated user input |

## Smart False Positive Filtering

Not all Codex findings are real bugs. Claude Code evaluates each one against:

- **The actual code** — reads the file at the flagged line to verify the issue exists
- **Project patterns** — checks if the flagged pattern is an established convention
- **Design intent** — cross-references with what was deliberately built
- **A decision tree** — 8-step classification covering hallucinated references, style opinions, internal error handling, and more

Common false positives that get automatically filtered:
- "Missing null check" on TypeScript strict-mode typed values
- "Should handle error" on internal function calls
- "Magic number" on universal values like `0`, `1`, `200`, `404`
- "Should validate input" on data that already passed schema validation upstream

---

## Prerequisites

You need three tools installed:

| Tool | Install | Purpose |
|------|---------|---------|
| **Claude Code** | [claude.ai/code](https://claude.ai/code) | The AI coding agent that runs Karen |
| **OpenAI Codex CLI** | `npm install -g @openai/codex` | The second-opinion reviewer |
| **GitHub CLI** | `brew install gh` or [cli.github.com](https://cli.github.com) | PR info and comment posting |

After installing Codex CLI, authenticate:
```bash
codex login
# or set CODEX_API_KEY environment variable
```

## Installation

```bash
claude plugin add https://github.com/Snowtumb/Karen
```

## Usage

### Slash Command

```
/codex-review
```

Run this after creating a PR, completing a feature branch, or whenever you want a second opinion on your changes.

### Auto-Triggering

Karen's skill also activates automatically when you say things like:
- "codex review"
- "bug check"
- "second opinion"
- "cross-model review"
- "run codex on this"

### When to Use

- **After creating a PR** — catch bugs before review
- **Before merging** — extra confidence on critical changes
- **After a big feature** — fresh eyes from a different model
- **When you're unsure** — "does this look right?" with a second AI perspective

### Fix Timing

After the review, Karen asks:

> "Codex found N confirmed bugs (posted to PR). Fix them now, or come back later?"

- **Fix now** — Claude Code applies fixes immediately, then runs a verification pass
- **Fix later** — Findings are saved as a PR comment. In a new session, Claude Code can read them with `gh pr view --comments` and pick up where it left off

## Configuration

### Codex Model

Karen uses whatever model you've configured in Codex CLI. To change it:

```bash
# Edit ~/.codex/config.toml
# Or pass --model flag when configuring codex
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CODEX_API_KEY` | OpenAI API key for Codex (alternative to `codex login`) |

---

## Contributing

Contributions are welcome! Some ways to help:

- **Report false positive patterns** — If Karen consistently misclassifies a finding, open an issue
- **Improve the evaluation guide** — Add new heuristics for common false positives
- **Expand prompt coverage** — Suggest review focus areas for specific stacks
- **Fix bugs** — PRs welcome

### Development

1. Clone: `git clone https://github.com/Snowtumb/Karen.git`
2. Install locally: `claude plugin add /path/to/Karen`
3. Test with `/codex-review` on a project with changes

### Commit Convention

This project uses [Conventional Commits](https://www.conventionalcommits.org/) for automatic versioning:
- `feat:` — new feature (minor version bump)
- `fix:` — bug fix (patch version bump)
- `feat!:` or `fix!:` — breaking change (major version bump)

---

## License

[MIT](LICENSE)

---

## Keywords

`claude-code-plugin` `code-review` `codex` `openai` `cross-model` `bug-detection` `ai-review` `second-opinion` `static-analysis` `pr-review` `claude-code` `ai-code-review`
