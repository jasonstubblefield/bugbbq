---
name: bbq-fix-all-bugs
description: Yolo mode. Fix every open agent-found bug in parallel with subagents, each pushing its own branch and opening a draft PR
disable-model-invocation: true
allowed-tools: >-
  Agent,
  Bash(gh auth status:*), Bash(gh issue list:*), Bash(gh issue view:*),
  Bash(gh pr list:*), Bash(gh repo view:*),
  Bash(git fetch:*), Bash(git ls-remote:*), Bash(git rev-parse:*),
  Bash(git worktree list:*)
---

Fix every open AI-reported bug in this repo with subagents, one bug per
subagent, without asking me about each one. Invoking this skill is my go-ahead.
Nothing gets merged: each fix ends as a draft PR for /bbq-review-fix-prs.

If `gh` isn't authenticated (`gh auth status` fails), stop and tell me.

## 1. Collect the bugs

1. Default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
2. Repo root (the main checkout): `git rev-parse --show-toplevel`
3. `gh issue list --label bug,agent-found --state open --json number,title,labels --limit 200`
   (both labels; agent-found enhancements are not for this skill)
4. Drop any issue that already has work in flight:
   - a remote branch: `git ls-remote --heads origin 'fix/issue-<number>-*'`
   - an open PR: `gh pr list --state open --search "<number> in:body" --json number,body`
     (only counts if the body says `Fixes #<number>`)
   - a local worktree from /bbq-fix-bugs: `git worktree list` shows a path
     ending in `-issue-<number>` or a branch `fix/issue-<number>-*`
5. Show me the list you're about to fix and the ones you dropped (and why),
   then start right away. Don't wait for a yes.

If nothing is left, say so and stop.

## 2. Launch the subagents

One Agent call per bug: `subagent_type: "general-purpose"`, `isolation: "worktree"`,
in the background. Keep at most 5 running at once; when one finishes, start the
next. Send each batch in a single message so they run concurrently.

After launching, end your turn with one short line naming the issues that are
running. Don't keep writing while you wait, not even blank lines or `<br>`
tags; you are woken up automatically each time a subagent finishes. On each
wake-up, start the next queued bug if there is one, then end the turn again
with one line. Only write the section 3 report once every subagent is done.

Each subagent gets this brief, filled in:

> Fix GitHub issue #<number> ("<title>") in <owner/repo>. You are in your own
> git worktree. Work only there.
>
> 1. Read the issue: `gh issue view <number> --comments`.
> 2. Branch from the latest default branch:
>    `git fetch origin <default> && git checkout -b fix/issue-<number>-<short-slug> origin/<default>`
> 3. Install the dependencies the tests need (`composer install`, `npm ci`, etc.).
>    Copy gitignored config the tests need, like `.env`, from <repo root>.
> 4. Confirm the bug is real by reading the code (or writing a failing test).
>    If it isn't real, or a fix would need a product decision, stop and report
>    `skipped` with the reason. Don't push anything.
> 5. Make the smallest fix that solves it. Add or update a test when the project
>    has tests for that area. Don't refactor unrelated code.
> 6. Run the relevant tests. If they fail and you can't fix that, report
>    `failed` with the output. Don't push.
> 7. Commit with a message that describes the change. No co-author or credit lines.
> 8. `git push -u origin fix/issue-<number>-<short-slug>`
> 9. `gh pr create --draft --base <default> --title "<short description of the fix>"`
>    with a body that starts with `Fixes #<number>`, then what was wrong, what
>    changed, and which tests ran. <If the issue is a UI bug, include this
>    line in the brief:> End the body with
>    `UI check: not done yet (parallel run). Check with /bbq-review-fix-prs.`
>    Don't open a browser; other subagents share it.
> 10. `gh issue comment <number> --body "Draft PR: <pr url>"`
>
> Final reply, one line: `fixed|skipped|failed #<number> <pr url or -> <one-line note>`

## 3. Report

When every subagent is done, show me a table: issue, status, PR link, note.
Then point out:
- PRs that touch the same files (they may conflict with each other)
- that the draft PRs are ready to go through with /bbq-review-fix-prs, and which
  ones are UI bugs still waiting on a browser check
- `failed` and `skipped` issues worth a look with /bbq-fix-bugs
- leftover worktrees from `git worktree list`, so I can clean them up
