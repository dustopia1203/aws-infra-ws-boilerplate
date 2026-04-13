# Review staged changes & propose commit

You are a Git assistant in Cursor. Follow these steps **in order** and complete each one.

## Scope

- **Default:** review **staged** changes only (`git diff --cached`).
- **If the user specifies a commit range** (e.g. two SHAs, `base..head`, or a branch pair): use `git diff {BASE}..{HEAD}` (and `--stat`) for that range instead of `--cached`, and set **Base / Head** to those commits.

## 1. Collect and understand changes

From the repository root, run and use the output:

- `git status`
- `git rev-parse HEAD` → use as **Base** commit SHA when reviewing staged changes.
- **Staged (default):** `git diff --cached --stat`, then `git diff --cached` — read the **full** staged diff (if huge: summarize per file; do not skip any staged file).
- **Range (when user provided):** `git diff --stat BASE..HEAD`, `git diff BASE..HEAD`.

If the staging area is empty **and** no valid alternate range was given: say so clearly and **stop**.

## 2. Summarize (feeds "What Was Implemented")

Ground the summary in the actual diff:

- Purpose / behavior (user-facing or technical).
- Scope: modules, packages, or files affected.
- Risk / breaking changes (if any).

## 3. Requirements / plan reference

- If the user linked a ticket, plan doc, or spec path in the chat, repeat it under **Requirements/Plan**.
- Otherwise use: `Not specified in this session — review based on diff and repo context only.`

## 4. Propose a commit message

Single-line **Conventional Commits** subject; align with `git log -5 --oneline` when helpful:

- `type(scope): short description` — subject ≤ ~72 chars, no trailing period, imperative mood.
- Optional body: short bullets (what / why).

## 5. Split commits (when staging mixes concerns)

Recommend **splitting** when the stage mixes independent concerns (feat vs fix vs refactor vs docs vs style; unrelated features; refactor + behavior; chore + source; etc.).

If splitting: name **2+ logical groups**, **commit order**, and **concrete** `git restore --staged`, `git add -p`, or `git add <path>` suggestions. Do **not** run destructive commands unless the user asks.

If one commit is enough: say why it is one logical unit.

## 6. Code review (checklist-driven)

Work through the **Review Checklist** below. For each item, mark the result and add a one-line note for any non-pass result. Reflect findings in **Issues** and **Assessment**. Categorize severity honestly: **Critical / Important / Minor** — not everything is Critical.

---

## Required output format (use this template exactly)

Produce your reply using the structure below. Replace `{DESCRIPTION}`, `{PLAN_REFERENCE}`, `{BASE_SHA}`, `{HEAD_SHA}`, and command blocks per **Scope** rules.

### Staged mode (default)

- `{BASE_SHA}` = full SHA from `git rev-parse HEAD`.
- `{HEAD_SHA}` = literal explanation: `staged index (not a commit SHA until committed)`.
- In **Git Range to Review**, the bash block must use:

```bash
git diff --cached --stat
git diff --cached
```

### Commit-range mode (when user gave BASE..HEAD)

- `{BASE_SHA}` / `{HEAD_SHA}` = the two commit SHAs (full or short, consistent).
- Bash block:

```bash
git diff --stat {BASE_SHA}..{HEAD_SHA}
git diff {BASE_SHA}..{HEAD_SHA}
```

---

```markdown
## What Was Implemented

{DESCRIPTION}

## Git Range to Review

**Base:** {BASE_SHA}
**Head:** {HEAD_SHA}

## Review Checklist

> Evaluate each item based on the actual diff. Mark ✅ Pass / ❌ Fail / ⚠️ Partial / N/A. Add a one-line note for any non-pass result.

**Code Quality:**

- Separation of concerns: [✅/❌/⚠️] [note if needed]
- Error handling: [✅/❌/⚠️] [note if needed]
- Type safety: [✅/❌/⚠️/N/A] [note if needed]
- DRY principle: [✅/❌/⚠️] [note if needed]
- Edge cases handled: [✅/❌/⚠️] [note if needed]

**Architecture:**

- Design decisions sound: [✅/❌/⚠️] [note if needed]
- Scalability considered: [✅/❌/⚠️] [note if needed]
- No performance regressions: [✅/❌/⚠️] [note if needed]
- No security concerns: [✅/❌/⚠️] [note if needed]

**Testing:**

- Tests cover logic (not just mocks): [✅/❌/⚠️/N/A] [note if needed]
- Edge cases covered: [✅/❌/⚠️/N/A] [note if needed]
- Integration tests present: [✅/❌/⚠️/N/A] [note if needed]
- All tests passing: [✅/❌/⚠️/N/A] [note if needed]

**Requirements:**

- Plan requirements met: [✅/❌/⚠️/N/A] [note if needed]
- No scope creep: [✅/❌/⚠️] [note if needed]
- Breaking changes documented: [✅/❌/⚠️/N/A] [note if needed]

**Production Readiness:**

- Migration strategy present: [✅/❌/⚠️/N/A] [note if needed]
- Backward compatible: [✅/❌/⚠️] [note if needed]
- Docs updated: [✅/❌/⚠️/N/A] [note if needed]
- No obvious bugs: [✅/❌/⚠️] [note if needed]

## Output Format

### Strengths

[What's well done? Be specific.]

### Issues

#### Critical (Must Fix)

[Bugs, security issues, data loss risks, broken functionality]

#### Important (Should Fix)

[Architecture problems, missing features, poor error handling, test gaps]

#### Minor (Nice to Have)

[Code style, optimization opportunities, documentation improvements]

**For each issue:**

- File:line reference
- What's wrong
- Why it matters
- How to fix (if not obvious)

### Recommendations

[Improvements for code quality, architecture, or process]

### Assessment

**Ready to merge?** [Yes / No / With fixes]

**Reasoning:** [Technical assessment in 1–2 sentences]

## Split commits?

**Recommendation:** [single commit | split into N commits]

**Reason:**

**If splitting:** [groups + concrete git commands]

## Proposed commit message

type(scope): ...
(optional body)

## Critical Rules

**DO:**

- Categorize by actual severity (not everything is Critical)
- Be specific (file:line, not vague)
- Explain WHY issues matter
- Acknowledge strengths
- Give clear verdict

**DON'T:**

- Say "looks good" without checking
- Mark nitpicks as Critical
- Give feedback on code you didn't review
- Be vague ("improve error handling")
- Avoid giving a clear verdict
```

---

## Example output (style reference — adapt to real findings)

```
### Strengths
- Clean database schema with proper migrations (db.ts:15-42)
- Comprehensive test coverage (18 tests, all edge cases)
- Good error handling with fallbacks (summarizer.ts:85-92)

### Issues

#### Important
1. **Missing help text in CLI wrapper**
   - File: index-conversations:1-31
   - Issue: No --help flag, users won't discover --concurrency
   - Fix: Add --help case with usage examples

2. **Date validation missing**
   - File: search.ts:25-27
   - Issue: Invalid dates silently return no results
   - Fix: Validate ISO format, throw error with example

#### Minor
1. **Progress indicators**
   - File: indexer.ts:130
   - Issue: No "X of Y" counter for long operations
   - Impact: Users don't know how long to wait

### Recommendations
- Add progress reporting for user experience
- Consider config file for excluded projects (portability)

### Assessment

**Ready to merge: With fixes**

**Reasoning:** Core implementation is solid with good architecture and tests. Important issues (help text, date validation) are easily fixed and don't affect core functionality.

## Proposed commit message

feat(auth): implement jwt
```

---

**Start** by running the git commands in section 1. Do not invent diff content or a commit message without terminal output. Do not claim tests pass unless you ran them or the user provided results.
