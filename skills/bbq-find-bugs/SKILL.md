---
name: bbq-find-bugs
description: Review code for real bugs and file them as GitHub issues via gh CLI, one at a time with confirmation
argument-hint: "[directory or files to review]"
disable-model-invocation: true
allowed-tools: >-
  Skill(code-review),
  Bash(gh auth status:*), Bash(gh label list:*), Bash(gh label create:*),
  Bash(gh issue list:*), Bash(gh issue view:*), Bash(gh issue create:*)
---

Review $ARGUMENTS for real bugs, not style nits. Ignore TODOs and cosmetic stuff.
If $ARGUMENTS is empty, ask me which directory or files to review before starting.

## 1. Find candidates

Run the `code-review` skill at `high` on the target (`high $ARGUMENTS`) to
collect candidates. Keep only correctness findings; drop reuse, simplification
and efficiency suggestions. If `code-review` isn't available, review the code
yourself.

## 2. Verify each one before showing it to me

For each candidate, re-read the code around it, plus its callers and any guards
or validation upstream, and write down the concrete input or state that
produces the wrong result or crash. If you can't name one, or a guard elsewhere
already prevents it, drop it. Only verified bugs go on to filing.

## 3. File them

Follow `filing.md` in this skill's folder for the title and the Summary
block. After the Summary, the issue body contains:
- file:line references
- what goes wrong, and the input or state that triggers it
- why it's a bug (expected vs. actual)
