# Karen

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**She hates your PR and your code.**

Karen is a [Claude Code](https://claude.ai/code) plugin that uses OpenAI Codex CLI to find every bug you and Claude Code messed up. This is not funny when things don't work and are broken, and Karen knows that. She will absolutely pin-point the issues, and Claude Code *will* listen to her. Otherwise, she will talk to a manager (that's you).

Karen doesn't care about your feelings. She cares about your code not crashing in production. She will read every line Codex flags, verify it against the actual source, dismiss the false alarms with a sigh, and fix the real bugs before they embarrass you in front of the team. Two passes, max. She doesn't have all day.

---

## How It Works

Karen runs a **6-phase workflow** every time you invoke her:

1. **Gather Context** -- Collects the diff, branch info, PR details. Claude Code writes a project context summary because Karen needs to know what kind of mess she's walking into.
2. **Invoke Codex** -- Sends everything to Codex CLI via `codex exec --sandbox read-only`. Codex reviews your code with full codebase read access. No hiding.
3. **Evaluate Findings** -- Claude Code reads the actual code at every flagged location. Each finding gets classified: confirmed bug, false positive, or "that's just your opinion." An 8-step decision tree. No vibes-based reviews.
4. **Fix Confirmed Bugs** -- If you choose "fix now", Claude Code applies fixes. Only high-confidence fixes. Karen doesn't guess.
5. **Verification Re-run** -- Codex runs again to make sure the fixes didn't break something else. Max 2 passes. Karen is thorough, not obsessive.
6. **Report & Persist** -- Results shown in terminal and posted as a PR comment. The evidence is now on the record.

## What Karen Catches

| Category | Priority | Examples |
|----------|----------|---------|
| **Bugs** | Critical | Null dereferences, missing imports, off-by-one errors, race conditions |
| **Logic Errors** | High | Wrong comparisons, inverted conditions, stale closures, swallowed errors |
| **Security Issues** | High | XSS, injection, missing auth checks, exposed secrets |
| **Integration Problems** | Medium | Wrong argument types, missing fields, broken contracts |
| **Boundary Errors** | Medium | Missing error handling on network calls, unvalidated user input |

## Smart False Positive Filtering

Not every Codex finding is a real problem. Some are just noise. Karen knows the difference:

- **Reads the actual code** -- verifies the issue exists at the flagged line
- **Checks project patterns** -- if the codebase does it everywhere, it's a convention, not a bug
- **Cross-references design intent** -- what you built on purpose doesn't get flagged
- **8-step decision tree** -- covers hallucinated references, style opinions, internal error handling, and more

Common false positives Karen automatically dismisses:
- "Missing null check" on TypeScript strict-mode typed values
- "Should handle error" on internal function calls
- "Magic number" on `0`, `1`, `200`, `404`
- "Should validate input" on data that already passed schema validation upstream

---

## Prerequisites

You need three tools installed:

| Tool | Install | Purpose |
|------|---------|---------|
| **Claude Code** | [claude.ai/code](https://claude.ai/code) | The AI coding agent that runs Karen |
| **OpenAI Codex CLI** | `npm install -g @openai/codex` | The second-opinion reviewer Karen sends after your code |
| **GitHub CLI** | `brew install gh` or [cli.github.com](https://cli.github.com) | PR info and comment posting |

After installing Codex CLI, authenticate:
```bash
codex login
# or set CODEX_API_KEY environment variable
```

## Installation

```bash
# Step 1: Add the Karen marketplace
claude plugin marketplace add Snowtumb/Karen

# Step 2: Install the plugin
claude plugin install karen
```

## Usage

### Slash Command

```
/codex-review
```

Run this after creating a PR, completing a feature branch, or whenever you want someone to tell you what's wrong with your code.

### Auto-Triggering

Karen also activates automatically when you say things like:
- "codex review"
- "bug check"
- "second opinion"
- "cross-model review"
- "run codex on this"

### When to Use

- **After creating a PR** -- catch bugs before a human reviewer does (less embarrassing)
- **Before merging** -- extra confidence on critical changes
- **After a big feature** -- fresh eyes from a different AI model
- **When you're unsure** -- "does this look right?" with a second perspective

### Fix Timing

After the review, Karen asks:

> "Codex found N confirmed bugs (posted to PR). Fix them now, or come back later?"

- **Fix now** -- Claude Code applies fixes immediately, then runs a verification pass
- **Fix later** -- Findings are saved as a PR comment. In a new session, Claude Code can read them with `gh pr view --comments` and pick up where Karen left off

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

Contributions are welcome. Karen respects people who fix things.

- **Report false positive patterns** -- If Karen consistently misclassifies a finding, open an issue
- **Improve the evaluation guide** -- Add new heuristics for common false positives
- **Expand prompt coverage** -- Suggest review focus areas for specific stacks
- **Fix bugs** -- PRs welcome

### Development

1. Clone: `git clone https://github.com/Snowtumb/Karen.git`
2. Add as local marketplace: `claude plugin marketplace add /path/to/Karen`
3. Install: `claude plugin install karen`
4. Test with `/codex-review` on a project with changes

### Commit Convention

This project uses [Conventional Commits](https://www.conventionalcommits.org/) for automatic versioning:
- `feat:` -- new feature (minor version bump)
- `fix:` -- bug fix (patch version bump)
- `feat!:` or `fix!:` -- breaking change (major version bump)

---

## License

[MIT](LICENSE) -- Karen is free. The bugs she finds are on you.

---

## Keywords

`claude-code-plugin` `code-review` `codex` `openai` `cross-model` `bug-detection` `ai-review` `second-opinion` `static-analysis` `pr-review` `claude-code` `ai-code-review`
