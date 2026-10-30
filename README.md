# Bug BBQ

Six agent skills that take a bug from "nobody has noticed yet" to "merged and
deployed", with you approving each step. They run inside Claude Code (and, with
some limits, Codex CLI) and talk to GitHub through the `gh` CLI.

```
 find bugs ──┐
             ├──> GitHub issues ──> fix one   ──> PR ──> review PRs ──> merge + deploy
 find UI bugs┘    (bug, agent-found)  fix all (yolo)                └──> or convert to Jira
```

**Find**

- **`/bbq-find-bugs <path>`**
  Reviews code for real correctness bugs, verifies each one, and files it as a
  GitHub issue after you say yes.
- **`/bbq-find-ui-bugs <url>`**
  Scans a running site in a browser (Playwright, Chrome or Safari) for console
  errors, broken requests, dead links, layout and dark-mode breakage, phone
  width problems and accessibility blockers. Files issues the same way.

**Fix**

- **`/bbq-fix-bugs`**
  Lists open bugs, recommends three, fixes the one you pick in its own git
  worktree, then opens a PR or merges and deploys on your say-so.
- **`/bbq-fix-all-bugs`**
  Yolo mode. Fixes every open agent-found bug in parallel with subagents, one
  draft PR each. Nothing is merged.

**Ship**

- **`/bbq-review-fix-prs`**
  Walks through the fix PRs one at a time with a short assessment. For each
  one you choose: merge and deploy, fix further, close, or skip.

**Hand off**

- **`/bbq-convert-to-jira`**
  Moves agent-found GitHub issues into a Jira project backlog and closes them
  with a link.

Full details for each skill, the conventions they share, and how to customize
them are in **[docs/skills.md](docs/skills.md)**.

## Requirements

- [Claude Code](https://claude.com/claude-code). Codex CLI also loads the
  skills; see [Codex notes](docs/skills.md#codex-notes) for what differs.
- [GitHub CLI](https://cli.github.com/) (`gh`), logged in: `gh auth login`.
- `git`. The fix skills create worktrees next to your checkout.
- For `/bbq-find-ui-bugs` and UI fix checks: a browser MCP server. Playwright
  MCP is best (clean browser, can emulate dark mode):
  `claude mcp add -s user playwright -- npx -y @playwright/mcp@latest --browser chrome --headless`.
  Claude in Chrome or a Safari MCP server work as fallbacks.
- For `/bbq-convert-to-jira`: the Atlassian MCP connector.

## Install

### With the skills CLI (recommended)

```bash
npx skills add jasonstubblefield/bugbbq -g --all
```

This installs all six skills for every agent it finds on your machine
(Claude Code, Codex and others). Update later with `npx skills update`.

### By asking your agent

Paste one of these into a terminal. The agent clones the repo and copies the
skills into place.

Claude Code:

```bash
claude "Install the Bug BBQ skills: clone https://github.com/jasonstubblefield/bugbbq into a temp folder, copy every folder under its skills/ directory into ~/.claude/skills/ (create it if needed), delete the temp folder, then list the bbq-* skills you installed and read me the Requirements section of its README so I know what else to set up."
```

Codex CLI:

```bash
codex "Install the Bug BBQ skills: clone https://github.com/jasonstubblefield/bugbbq into a temp folder, copy every folder under its skills/ directory into ~/.codex/skills/ (create it if needed), delete the temp folder, then list the bbq-* skills you installed and read me the Requirements section of its README so I know what else to set up."
```

Start a new session afterwards so the agent picks up the new skills.

### By hand

```bash
git clone https://github.com/jasonstubblefield/bugbbq.git
cp -R bugbbq/skills/bbq-* ~/.claude/skills/      # Claude Code
cp -R bugbbq/skills/bbq-* ~/.codex/skills/       # Codex CLI
```

Or symlink the folders instead of copying, so `git pull` updates them.

Install all six together. They share files (`bbq-find-bugs/filing.md`,
`bbq-find-ui-bugs/verify-ui-fix.md`) and expect to sit next to each other.

## Quick start

In a repo with a GitHub remote:

```
/bbq-find-bugs app/Services        # file some bugs
/bbq-find-ui-bugs https://your.site   # and some UI bugs
/bbq-fix-bugs                      # fix one, with review
/bbq-fix-all-bugs                  # or fix them all at once as draft PRs
/bbq-review-fix-prs                # go through the PRs, merge and deploy
```

The first run creates the labels it needs (`bug`, `agent-found`, `ui`).

## What these skills will and won't do on their own

- Filing an issue, pushing a branch, opening a PR, merging and closing always
  wait for you to pick an option, with one exception: `/bbq-fix-all-bugs`
  pushes branches and opens **draft** PRs without asking, because invoking it
  is the go-ahead. It never merges.
- **Merge and deploy** merges without waiting for the PR's CI. If your repo
  deploys on push to the default branch, that ships the fix. Your default
  branch's own test job still gates the deploy if the workflow is built that
  way. Read [the merge section](docs/skills.md#merging-and-deploying) before
  using it on something that matters.
- The agent never touches your credentials. When a page needs a login,
  `/bbq-find-ui-bugs` leaves the tab on the login page and asks you to sign in
  yourself in Chrome or Safari, password manager, passkeys and 2FA included.
  Playwright's browser can't be logged into, so pages behind a login go
  through Chrome or Safari, or are skipped and listed.

## License

[MIT](LICENSE).
