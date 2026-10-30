---
name: bbq-review-fix-prs
description: Walk through open bug-fix PRs (fix/issue-* branches) one at a time, check each fix, and merge and deploy it, fix it further, or close it
argument-hint: "[optional PR numbers]"
disable-model-invocation: true
allowed-tools: >-
  Bash(gh auth status:*), Bash(gh repo view:*), Bash(gh pr list:*),
  Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh pr checks:*),
  Bash(gh pr ready:*), Bash(gh pr merge:*), Bash(gh pr close:*),
  Bash(gh pr comment:*), Bash(gh issue view:*),
  Bash(gh run list:*), Bash(gh run view:*), Bash(gh run watch:*),
  Bash(git push origin --delete fix/issue-:*),
  Bash(git fetch:*), Bash(git worktree list:*), Bash(git worktree add:*),
  Bash(git worktree remove:*), Bash(git branch -D fix/issue-:*),
  Bash(git status:*), Bash(git diff:*), Bash(git log:*)
---

Review the PRs that /bbq-fix-bugs and /bbq-fix-all-bugs opened, and merge the ones I
approve. Picking **Merge and deploy** for a PR is my go-ahead to merge it into
the default branch, which deploys it if the repo deploys on push. Never merge a
PR I didn't pick that for.

If `gh` isn't authenticated (`gh auth status` fails), stop and tell me.

## 1. List

1. `gh pr list --state open --json number,title,url,headRefName,isDraft,body,files,additions,deletions --limit 100`
2. Keep PRs whose branch starts with `fix/issue-`. If $ARGUMENTS lists PR
   numbers, keep only those.
3. For each, get the linked issue (`Fixes #<n>` in the body) and its labels,
   and CI status (`gh pr checks <pr>`). Don't wait for checks that are still
   running; show them as pending.
4. Find out how the repo deploys: look in `.github/workflows/` for a workflow
   that runs on push to the default branch and deploys. Note its name and
   whether its deploy job waits for the tests on the default branch
   (`needs: [tests]` or similar).
5. Show a table: PR, issue, title, CI, size (+/-), `ui` if it's a UI bug, and
   which other PRs in the list touch the same files. Under it, one line on
   deploys, e.g. "Merging deploys www via deploy.yml; main's tests run first."

Go through them in this order: CI passing before pending before failing, then
smallest first. If nothing matches, say so and stop.

## 2. Review each PR, one at a time

1. Read the issue (`gh issue view <n>`) and the diff (`gh pr diff <pr>`).
2. Give me a short assessment:
   - does it fix the cause described in the issue, or only the symptom
   - anything risky (auth, billing, migrations, external APIs, data changes)
   - whether a test covers it
   - conflicts with other PRs in the list
3. If it's a UI bug and the PR doesn't already say `UI check (...): verified`,
   check it in a browser following `verify-ui-fix.md` in the `bbq-find-ui-bugs`
   skill folder (installed next to this one).
   Use the PR's existing worktree if `git worktree list` shows one for its
   branch. Otherwise `git fetch origin <branch>` and
   `git worktree add ../<repo-dir>-issue-<n> <branch>`. Post the result with
   `gh pr comment`.
4. Ask with AskUserQuestion. The question has the PR number, title, your
   assessment in a sentence or two, CI status (passing, pending or failing)
   and UI check result. Each option's description says in plain words what
   happens if I pick it, never just the command. Options:
   - **Merge and deploy**: description like "Merge into main now and deploy.
     Doesn't wait for this PR's CI; main's tests still run before the deploy."
     If the PR's CI already failed, say so in the description and put
     **Fix it now** first instead.
   - **Fix it now**: ask me what to change, make the change in the PR's
     worktree, run the tests, show me the diff, and push on my yes. Then ask
     about this PR again.
   - **Close**: ask me for a one-line reason, then
     `gh pr close <pr> --comment "<reason>"`. Keep the branch, and leave the
     issue open so it can be retried with /bbq-fix-bugs.
   - **Skip**: leave the PR open as it is and move on.

One PR at a time. Wait for my answer before the next one.

### Merging

When I pick **Merge and deploy**:

1. If it's a draft, `gh pr ready <pr>` first (GitHub won't merge a draft).
2. `gh pr merge <pr> --merge`, without `--auto` and without waiting for CI.
   Use the merge method the repo's CLAUDE.md asks for if it names one.
3. `git push origin --delete <branch>`. Leave the local branch and worktree for
   the report step.
4. Don't watch the deploy here; go straight to the next PR. Each merge starts a
   deploy run, and the workflow's concurrency setting usually skips the runs
   in between, so the last merge ships everything. Section 3 watches it.
5. If the merge fails because the PR now conflicts with something merged
   earlier in this session, say so and ask again with **Fix it now** first
   (rebase onto the default branch, rerun the tests, push on my yes, then
   merge).

## 3. Deploy and report

If anything was merged:

1. `gh run list --workflow <deploy workflow> --branch <default> --limit 5` and
   `gh run watch <id>` on the newest run until it finishes.
2. If it failed, `gh run view <id> --log-failed`, tell me which job failed and
   which merged PR most likely caused it, and offer to fix it on a new branch
   or revert that PR's merge commit. A failed test job on the default branch
   blocks every later deploy, so don't leave it.
3. If the PRs touched different deploy components (e.g. `crawler/` or `admin/`
   with their own workflows), watch each component's newest run.
4. If the repo's CLAUDE.md says to check the live site after a deploy, do that
   for the merged fixes.

Then a table: PR, what happened (merged / fixed and merged / closed / skipped),
UI check result, deploy result. Then list worktrees for PRs that are now merged
or closed, and ask whether to remove them (`git worktree remove <path>`, plus
`git branch -D <branch>` for merged ones). Closed PRs keep their branches.
