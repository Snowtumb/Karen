# Evaluation Guide: Filtering Codex Findings

Reference document for Phase 3 (Evaluate Findings). Use this to classify each Codex finding as confirmed bug, false positive, or style/opinion.

---

## Classification Decision Tree

```
For each finding:
  1. Does the referenced file:line exist?
     NO -> False positive (stale/hallucinated reference)
     YES -> continue

  2. Does the code at that location match what Codex describes?
     NO -> False positive (misread the code)
     YES -> continue

  3. Would this cause a runtime failure, security risk, or broken integration?
     YES -> Confirmed bug -- auto-fix
     NO -> continue

  4. Is this an established pattern used elsewhere in the codebase?
     YES -> False positive (project convention)
     NO -> continue

  5. Is this about error handling for an internal function call?
     YES -> False positive (YAGNI -- trust internal code)
     NO -> continue

  6. Is this a style/naming/formatting preference?
     YES -> Style/opinion -- report only
     NO -> continue

  7. Is this about adding features or enhancements not in the PR scope?
     YES -> False positive (out of scope)
     NO -> continue

  8. Is this a deliberate design decision mentioned in the context summary?
     YES -> False positive (intentional)
     NO -> Uncertain -- report for user review (don't auto-fix)
```

---

## False Positive Patterns

These are common false positives from AI code reviewers. When you see these patterns, verify carefully before classifying as a real bug.

### 1. "Missing null check" on TypeScript strict-mode code
**Pattern:** Codex flags `foo.bar` as needing a null check, but `foo` is typed as non-optional.
**Why false:** TypeScript strict mode already enforces this at compile time.
**Exception:** If the value comes from an external source (database, API response, user input) where the type might not match runtime reality — that IS a real concern.

### 2. "Should handle error" on internal function calls
**Pattern:** Codex suggests try/catch around calling an internal utility function.
**Why false:** Internal functions should be trusted. Error handling belongs at system boundaries.
**Exception:** If the internal function does I/O (file system, network, database) — that's a system boundary.

### 3. "Variable could be undefined" when it can't
**Pattern:** Codex flags a variable that's guaranteed to be defined by the control flow.
**Why false:** Codex may not trace the full control flow correctly.
**Verify:** Read the surrounding code to confirm the variable is definitely assigned before use.

### 4. "Should validate input" on already-validated data
**Pattern:** Codex suggests adding schema validation where the data already passed through a validator.
**Why false:** Data that passed schema validation (Zod, Joi, Yup, class-validator, etc.) upstream doesn't need re-validation.
**Verify:** Trace the data flow — is it validated before reaching this point?

### 5. "Magic number should be a constant"
**Pattern:** Codex flags numbers like `0`, `1`, `200`, `404` as magic numbers.
**Why false:** These are universally understood values, not magic numbers.
**Exception:** Domain-specific numbers (tax rates, thresholds, limits) should be named constants.

### 6. "Function is too long / should be split"
**Pattern:** Codex suggests breaking up a function that's under 50 lines.
**Why false:** Style preference, not a bug. Short functions don't need splitting.
**Exception:** If the function genuinely does multiple unrelated things, splitting helps — but that's a refactoring suggestion, not a bug.

### 7. "Unused import / unused variable"
**Pattern:** Codex flags an import or variable as unused.
**Why false positive sometimes:** The import may be used for types only (TypeScript), or the variable may be used in a template/JSX expression that Codex didn't parse correctly.
**When it IS real:** If truly unused after checking — this is a valid cleanup, but low severity.

### 8. "Should use optional chaining"
**Pattern:** Codex suggests `foo?.bar` where `foo` is always defined.
**Why false:** If `foo` is guaranteed non-null, optional chaining adds unnecessary ambiguity.
**When it IS real:** If `foo` actually can be null/undefined at runtime.

---

## Confirmed Bug Patterns

These patterns are almost always real bugs. Fix with high confidence.

### 1. Wrong function argument count or types
Calling a function with the wrong number of arguments or wrong types. Verify by reading the function definition.

### 2. Missing await on async function
Calling an async function without `await` — the promise is created but not awaited, leading to unhandled promise rejections or race conditions.

### 3. Undefined/null in database write
Writing `undefined` or `null` to a database field that doesn't accept it. Behavior varies by database — some throw, some silently drop the field, some store null. Check the project's database conventions.

### 4. Wrong comparison operator
Using `==` instead of `===`, `>` instead of `>=`, or comparing wrong values. Verify the intended logic.

### 5. Missing import
Using a function/component/type that isn't imported. The file won't compile.

### 6. Stale closure in React hook
Using a variable in a `useEffect` or `useCallback` without including it in the dependency array. The closure captures a stale value.

### 7. XSS via unsanitized user input rendering
Rendering user-provided content without sanitization. Always a security bug.

### 8. Missing tenant/scope filter in multi-tenant query
Querying a multi-tenant data source without filtering by the tenant scope field. Data leak across tenants. Check the project's conventions for what the scope field is (e.g., `organizationId`, `tenantId`, `teamId`).

### 9. Using `||` instead of `??` for numeric defaults
`value || defaultValue` treats `0` as falsy. For numeric fields (like `threshold`, `count`, `limit`), use `value ?? defaultValue`.

---

## Severity Guidelines

| Severity | Criteria | Auto-fix? |
|----------|----------|-----------|
| **Critical** | Would cause crash, data loss, security breach, or data leak | Yes — immediate |
| **High** | Would cause incorrect behavior, broken feature, or degraded UX | Yes |
| **Medium** | Edge case failure, minor integration issue, or missing boundary validation | Yes, if confident |
| **Low** | Code smell, minor improvement, or theoretical concern | No — report only |

---

## When in Doubt

If you cannot confidently classify a finding after reading the actual code:
- **Don't auto-fix.** Report it to the user with your analysis.
- Include: what Codex said, what you found when you checked, and why you're uncertain.
- Let the user decide.
