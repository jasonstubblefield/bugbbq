# Bug BBQ skills

Each skill is a folder with a `SKILL.md` that the agent follows when you type
its slash command. The text is written from the point of view of the person
running it ("show me", "ask me"), so "I" and "me" in a skill mean you.

Contents

- [The loop](#the-loop)
- [Conventions every skill shares](#conventions-every-skill-shares)
- [bbq-find-bugs](#bbq-find-bugs)
- [bbq-find-ui-bugs](#bbq-find-ui-bugs)
- [bbq-fix-bugs](#bbq-fix-bugs)
- [bbq-fix-all-bugs](#bbq-fix-all-bugs)
- [bbq-review-fix-prs](#bbq-review-fix-prs)
- [bbq-convert-to-jira](#bbq-convert-to-jira)
- [Merging and deploying](#merging-and-deploying)
- [Codex notes](#codex-notes)
- [Customizing](#customizing)
- [Troubleshooting](#troubleshooting)

## The loop

1. **Find.** `/bbq-find-bugs` reads code; `/bbq-find-ui-bugs` drives a browser.
   Both verify each candidate, search for duplicates, and ask you before filing
   each issue.
2. **Fix.** `/bbq-fix-bugs` fixes one bug with you watching. `/bbq-fix-all-bugs`
   fixes every open agent-found bug in parallel and leaves draft PRs.
3. **Ship.** `/bbq-review-fix-prs` goes through the PRs one at a time. You pick
   merge and deploy, fix further, close or skip.
4. **Or hand off.** `/bbq-convert-to-jira` moves issues to a team's Jira
   backlog instead.

Every skill stops first if `gh auth status` fails.

## Conventions every skill shares

**Labels.** Issues are filed with `bug` and `agent-found`; UI bugs also get
`ui`. The find skills create missing labels on first run. `agent-found` is how
the fix skills know which issues are theirs.

**Issue titles.** `<Area>: <what goes wrong, in plain words>`, under about 80
characters, no class or file names, no fix. The area is a product area as a
user would name it (Search, Billing, Mobile, Homepage), reusing names already
in your issue list. One bug per issue.

**Issue bodies** start with a Summary block:

```
## Summary

Two or three plain sentences: who is affected, what they see or lose, what it costs.

**Severity:** high | medium | low, one-line reason
**How often:** every visit | common flow | rare edge case, one-line reason
**Found by:** bbq-find-bugs on 2026-10-30, app/Services
```

then the technical sections: file:line references, trigger, expected vs actual
(code bugs) or page URL, browser, reproduction steps, observed vs expected,
console and network evidence (UI bugs).

**Branches and PRs.** Fix branches are `fix/issue-<number>-<short-slug>`, cut
from the latest default branch, each in its own git worktree beside your
checkout (`../<repo>-issue-<number>`). PR bodies start with `Fixes #<number>`
so merging closes the issue. UI fixes carry a `UI check (<browser>): verified |
still broken | couldn't check` line in the PR body or a comment.

**Work in flight.** Before fixing, the fix skills look for an existing
`fix/issue-<number>-*` remote branch, an open PR whose body says
`Fixes #<number>`, or a local worktree for the issue, and don't start over.

**Commits.** Messages describe the change. No co-author or credit lines.

## bbq-find-bugs

`/bbq-find-bugs <directory or files>`

1. Runs Claude Code's built-in `code-review` skill at `high` on the target to
   collect candidates, keeping only correctness findings. Falls back to
   reviewing the code itself if that skill isn't available.
2. Re-reads the code around each candidate, including callers and upstream
   guards, and names the concrete input or state that produces the wrong
   result. Candidates it can't do that for are dropped.
3. Checks how your open issues are titled, searches for duplicates, and asks
   about each bug with the exact title, the Summary and the severity. Options:
   file it, skip, or edit first.

Writes: GitHub issues (one at a time, each after a yes) and labels.

## bbq-find-ui-bugs

`/bbq-find-ui-bugs <url and pages or flows> [--browser playwright|chrome|safari]`

Picks a browser backend: Playwright MCP first (clean browser, dark-mode
emulation), then Claude in Chrome, then a Safari MCP server. Chrome and Safari
are your real browsers, logged in as you, so the skill won't submit forms that
send email, take payment or create records on a production site without
asking. `browsers.md` in the skill folder maps each scan step to each
backend's tool and lists their limits.

The scan covers: console errors and failed requests on every page visited,
main navigation and primary calls to action, flows you named, a dark-mode pass,
a 390px phone-width pass, screenshots to confirm layout bugs, and clicking
suspected dead links. Style nits, pixel alignment and copy don't count.

**Logins.** The agent never reads or types your credentials: no Keychain, no
password manager, no `.env`, no asking you to paste a password or 2FA code.
When a page needs a login in Chrome or Safari, it leaves the tab on the login
page and asks you to sign in there yourself, then continues on your word.
Password managers, passkeys and 2FA all work because the login happens in
your own browser. Playwright's browser is headless and can't be logged into,
so for pages behind a login the skill switches to Chrome or Safari if one is
available, or scans the public pages and lists what it skipped. If you know
the pages need a login, say so when you invoke the skill and it starts with
Chrome.

Writes: GitHub issues with the `ui` label, same confirmation flow as
`bbq-find-bugs`.

## bbq-fix-bugs

`/bbq-fix-bugs`

1. Lists open `bug` issues with age, labels and in-flight status, recommends up
   to three with a complexity guess and risk notes, and asks you to pick.
2. Creates a worktree and branch, installs dependencies, copies gitignored
   config such as `.env` from your main checkout, and comments on the issue.
3. Fixes, runs the relevant tests, and for UI bugs checks the fix in a browser
   (`bbq-find-ui-bugs/verify-ui-fix.md`).
4. Shows you the diff, test results and UI check, then asks: **Merge and
   deploy**, **Open PR only**, or **Change something**. Nothing is pushed
   before you pick one of the first two.
5. Opens the PR. On Merge and deploy it merges, deletes the remote branch,
   watches the deploy run and reports what broke if anything did.

Writes: issue comment, branch, PR; on your say-so a merge and a deploy.

## bbq-fix-all-bugs

`/bbq-fix-all-bugs`

Invoking it is the go-ahead. It lists open issues labeled both `bug` and
`agent-found`, drops any with work in flight (and tells you why), then starts
one subagent per bug, at most five at a time, each in its own worktree. Each
subagent confirms the bug is real, makes the smallest fix, adds or updates a
test where the project has them, runs tests, pushes `fix/issue-<n>-<slug>`,
opens a **draft** PR and comments on the issue. Subagents that can't confirm
the bug, hit a product decision or can't get tests green report `skipped` or
`failed` and push nothing.

UI bugs get the line `UI check: not done yet (parallel run)` in the PR, since
subagents don't share a browser. `bbq-review-fix-prs` does that check later.

The final report lists each issue's outcome, PRs that touch the same files,
and leftover worktrees to remove.

Writes: branches and draft PRs without asking. Never merges.

## bbq-review-fix-prs

`/bbq-review-fix-prs [PR numbers]`

Lists open PRs on `fix/issue-*` branches with issue, CI state, size, `ui`
flag and file overlaps, plus one line on how the repo deploys. Then, one PR at
a time: reads the issue and diff, tells you whether it fixes the cause or the
symptom, anything risky, whether a test covers it and what it conflicts with;
runs the browser check for UI bugs that don't have one yet; and asks:

- **Merge and deploy**: marks a draft ready, merges now without waiting for the
  PR's CI, deletes the remote branch, moves on. If the PR's CI already failed,
  it says so and offers Fix it now first.
- **Fix it now**: you say what to change; it edits in the PR's worktree, runs
  tests, shows the diff, pushes on your yes, then asks about the PR again.
- **Close**: closes with your one-line reason. Branch kept, issue left open.
- **Skip**: leaves it as is.

After the last PR it watches the newest deploy run on the default branch,
reports a failure with the most likely culprit PR and an offer to fix or
revert, then prints a summary table and offers to remove finished worktrees.

Writes: PR comments, merges, closes, remote branch deletions, each after your
pick.

## bbq-convert-to-jira

`/bbq-convert-to-jira [JIRA-KEY]`

Needs the Atlassian MCP connector. Lists open `agent-found` issues with their
type, lets you pick by number, range or `all`, then picks the Jira project
(from the argument, a saved default for this repo in
`~/.config/bugbbq/jira-projects.json`, or by asking). Bugs become `Bug`
tickets; `enhancement` issues become `Task` (or `Story`). The ticket
description is `Moved from GitHub: <url>` plus the issue body. It searches
Jira for the GitHub URL first so reruns don't duplicate, then closes the
GitHub issue as "not planned" with a link to the ticket. A failed ticket
leaves the GitHub issue open.

## Merging and deploying

**Merge and deploy** in `bbq-fix-bugs` and `bbq-review-fix-prs` runs
`gh pr merge --merge` immediately. It does not wait for the PR's checks. The
reasoning: when you're clearing a batch of small fixes, waiting on CI per PR
is the slow part, and most repos that deploy on push run their test job on
the default branch before the deploy job. That job is your gate.

Check that this holds for your repo before using it:

- Your deploy workflow's deploy job should `needs:` the test job, so a bad
  merge fails the run instead of shipping.
- If several PRs merge in a row, each starts a run. A `concurrency` group with
  `cancel-in-progress: false` queues them so the last one ships everything;
  the skill watches the newest run.
- A red default branch blocks every later deploy, so the skill treats a failed
  run as something to fix right away, not a note for later.

If your repo has branch protection that requires passing checks, the merge is
refused and the skill tells you. If your repo doesn't deploy on push, "deploy"
means nothing more than the merge.

If you'd rather never merge from the skill, delete the Merge and deploy option
from the two `SKILL.md` files. The rest works unchanged.

## Codex notes

The skills are plain `SKILL.md` files, which Codex CLI loads from
`~/.codex/skills/`. Some parts are written for Claude Code and Codex will
approximate them:

- `allowed-tools` and `disable-model-invocation` in the frontmatter are Claude
  Code settings. Codex ignores them and uses its own approval mode.
- `AskUserQuestion` is Claude Code's option dialog. Codex asks in plain text;
  answer with the option name.
- `bbq-fix-all-bugs` uses Claude Code's `Agent` tool with worktree isolation.
  In Codex, run `bbq-fix-bugs` per issue instead.
- `bbq-find-bugs` starts with Claude Code's `code-review` skill and falls back
  to reviewing the code itself, which is what happens in Codex.
- Browser backends are MCP servers; Playwright MCP works in both agents.

## Customizing

Edit the `SKILL.md` files; they're instructions, not code. Common changes:

- **Different labels.** Change `bug,agent-found` in `bbq-find-bugs/filing.md`
  and in the `gh issue list` lines of the fix skills together.
- **Different branch naming.** `fix/issue-<number>-<short-slug>` appears in
  the fix and review skills and in their `allowed-tools` rules.
- **Issue title areas.** The example list in `filing.md` (Search, Billing,
  Mobile, ...) is a starting point; the skill reuses whatever your open issues
  already use.
- **Severity as labels.** The Summary block records severity in the body. If
  you want it in the issue list, add label creation for `sev:high` and friends
  to `filing.md` the same way `agent-found` is created.
- **Merge method.** The skills use a merge commit unless the repo's
  `CLAUDE.md` names another.

## Troubleshooting

- **"gh isn't authenticated."** `gh auth login`, then rerun.
- **`gh issue create` fails on a label.** The find skills create `bug`,
  `agent-found` and `ui` first; if your account can't create labels, ask a
  repo admin to add them.
- **No browser available.** Install Playwright MCP (command in the README) and
  start a new session. `/bbq-find-ui-bugs` prints the same instructions.
- **Dark mode or phone width "not fully covered."** Chrome won't resize below
  about 500px and neither Chrome nor Safari can emulate dark mode; the skill
  falls back to the site's own theme switch if it has one. Use Playwright for
  full coverage.
- **A fix is dropped as "in flight" but nothing is happening.** A closed PR
  keeps its branch, and a remote `fix/issue-<n>-*` branch counts as in flight.
  Delete the branch (`git push origin --delete <branch>`) to free the issue.
- **Leftover worktrees.** `git worktree list`, then `git worktree remove <path>`.
  The report at the end of each fix skill lists them.
- **The agent keeps printing blank lines while subagents run.** That was a bug
  in an earlier version of `bbq-fix-all-bugs`; update the skills.
