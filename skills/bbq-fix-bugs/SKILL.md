---
name: bbq-fix-bugs
description: List open bug issues via gh, recommend a few good candidates to work on, fix the one I pick in its own worktree, and optionally merge and deploy it
disable-model-invocation: true
allowed-tools: >-
  Bash(gh auth status:*), Bash(gh issue list:*), Bash(gh issue view:*),
  Bash(gh issue comment:*), Bash(gh repo view:*), Bash(gh pr list:*),
  Bash(git status:*), Bash(git fetch:*), Bash(git ls-remote:*),
  Bash(git worktree add:*), Bash(git worktree list:*), Bash(git worktree remove:*),
  Bash(git branch:*), Bash(git diff:*), Bash(git log:*),
  Bash(git push -u origin fix/issue-:*), Bash(git push origin --delete fix/issue-:*),
  Bash(gh pr create:*), Bash(gh pr merge:*), Bash(gh pr checks:*),
  Bash(gh run list:*), Bash(gh run view:*), Bash(gh run watch:*)
---

If `gh` isn't authenticated (`gh auth status` fails), stop and tell me.

## 1. Pick

1. `gh issue list --label bug --state open --json number,title,body,labels,updatedAt --limit 50`
2. Find work already in flight, once for all issues:
   - branches: `git ls-remote --heads origin 'fix/issue-*'`
   - PRs: `gh pr list --state open --json number,headRefName,isDraft,body --limit 100`,
     matched to issues by `Fixes #<number>` in the body or a `fix/issue-<number>-` branch
3. Show me a compact table: number, title, age, `agent-found` and `ui` labels,
   and status (`open`, `branch pushed`, `draft PR #n`, `PR #n`).

Then recommend up to 3 that would be good to tackle today, from the ones with
no work in flight. For each, tell me:

- rough complexity (15 min / half day / multi-day)
- whether it touches anything risky (auth, billing, migrations, external APIs)
- whether I'd need context I don't have

Ask me to pick with AskUserQuestion: one option per recommendation, with its
number, title and complexity. I can type a different issue number under Other.

If I pick one that already has a PR, don't start over. Point me to
/bbq-review-fix-prs for that PR, or ask whether to keep working on its branch.

## 2. Set up a worktree

Work in a separate git worktree so my current checkout, and anything
uncommitted in it, is left alone.

1. Get the default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
2. `git fetch origin <default>`
3. If `fix/issue-<number>-<short-slug>` already exists (`git branch --list`,
   `git worktree list`), stop and ask me.
4. `git worktree add -b fix/issue-<number>-<short-slug> ../<repo-dir>-issue-<number> origin/<default>`
5. Do everything below inside that worktree directory.
6. A new worktree has no dependencies or untracked config. Before running tests,
   install what the project needs (`composer install`, `npm ci`, etc.) and copy
   over gitignored files the tests depend on, like `.env`, from the main checkout.
7. Comment on the issue: `gh issue comment <number> --body "Working on this in branch fix/issue-<number>-<short-slug>."`

## 3. Fix

1. Implement the fix.
2. Run the relevant tests.
3. If it's a UI bug, check the fix in a browser following `verify-ui-fix.md` in
   the `bbq-find-ui-bugs` skill folder (installed next to this one). If it's
   still broken, keep fixing.
4. Stop and show me the diff, the test results and the UI check result, then
   ask with AskUserQuestion. Each option's description says in plain words
   what happens, never just the command:
   - **Merge and deploy**: push, open the PR and merge it into the default
     branch right away, which deploys it if the repo deploys on push. Doesn't
     wait for the PR's CI; the default branch's tests still run before the
     deploy if the deploy workflow needs them.
   - **Open PR only**: push and open the PR, and leave merging for later
     (/bbq-review-fix-prs).
   - **Change something**: tell me what to change; fix, rerun the tests and
     ask again.

Do not push or open a PR until I pick one of the first two.

## 4. PR

1. Commit, `git push -u origin <branch>`, and open the PR with `gh pr create`. Put
   `Fixes #<number>` in the PR body so merging closes the issue, plus the UI
   check result if there was one. Show me the PR URL.
2. If I picked **Open PR only**, skip to the worktree question at the end.
3. If I picked **Merge and deploy**:
   1. `gh pr merge <pr> --merge` (or the merge method the repo's CLAUDE.md
      names), without `--auto` and without waiting for CI. Then
      `git push origin --delete <branch>`.
   2. Find the workflow in `.github/workflows/` that deploys on push to the
      default branch. `gh run list --workflow <it> --branch <default> --limit 3`
      and `gh run watch <id>` on the run for the merge commit.
   3. If it fails, `gh run view <id> --log-failed`, tell me what broke, and
      offer to fix it on a new branch or revert the merge commit. A failed test
      job on the default branch blocks every later deploy, so don't leave it.
   4. If the repo's CLAUDE.md says to check the live site after a deploy, do
      that for this fix.
4. Ask whether to remove the worktree (`git worktree remove <path>`). If the PR
   was merged, also offer to delete the local branch (`git branch -D <branch>`);
   otherwise the branch stays.
