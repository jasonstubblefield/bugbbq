---
name: bbq-convert-to-jira
description: List open agent-found GitHub issues, let me pick which to move into Jira (bugs as Bug, enhancements as Task) in a team's project backlog, then close the GitHub issues with a link
argument-hint: "[optional Jira project key]"
disable-model-invocation: true
allowed-tools: >-
  Read, Edit(~/.config/bugbbq/jira-projects.json), Write(~/.config/bugbbq/jira-projects.json),
  Bash(gh auth status:*), Bash(gh repo view:*),
  Bash(gh issue list:*), Bash(gh issue view:*), Bash(gh issue close:*)
---

Move AI-reported GitHub bugs into Jira. Uses the Atlassian MCP connector.

Stop and tell me if `gh auth status` fails or the Atlassian tools aren't available.

## 1. Pick the bugs

1. `gh issue list --label agent-found --state open --json number,title,labels,createdAt,url --limit 200`
2. Show a numbered table: number, title, type (`bug` or `enhancement` from its
   labels), age.
3. Ask which to convert. I'll answer with issue numbers, ranges or `all`
   (e.g. `12, 15-18`). If there are 4 or fewer, use AskUserQuestion with
   multiSelect instead.

## 2. Pick the team (Jira project)

Each team has its own Jira project. Saved choices are in
`~/.config/bugbbq/jira-projects.json` (create it if missing), keyed by GitHub repo
(`gh repo view --json nameWithOwner -q .nameWithOwner`):
`{"owner/repo": {"cloudId": "...", "site": "acme.atlassian.net", "project": "WEB"}}`

1. If $ARGUMENTS names a project key, use it.
2. Otherwise, if this repo has a saved project, ask with AskUserQuestion:
   use <project> (Recommended), or pick another.
3. Otherwise, or if I pick another:
   - Get the Atlassian site and cloudId (getAccessibleAtlassianResources). If
     there's more than one site, ask which.
   - List projects (getVisibleJiraProjects) and ask me which team. More than 4,
     show a numbered list and let me type.
   - Ask whether to save it as the default for this repo. On yes, update
     `~/.config/bugbbq/jira-projects.json`.
4. Check the project's issue types (getJiraProjectIssueTypesMetadata). Bugs
   go in as `Bug`; issues labeled `enhancement` go in as `Task` (or `Story` if
   there's no Task). If a type you need is missing, show me the project's
   types and ask.

Tickets go to the project backlog: don't set a sprint.

## 3. Convert, one issue at a time

For each selected issue:

1. `gh issue view <number> --json title,body,url,labels`
2. Check for an existing ticket so a rerun doesn't duplicate:
   searchJiraIssuesUsingJql with `project = <KEY> AND text ~ "\"<issue url>\""`.
   If one exists, skip creating, but still do step 4 with that ticket.
3. Create the ticket (createJiraIssue):
   - issue type: as decided in step 2.4 from the issue's labels
   - summary: the GitHub title
   - description: `Moved from GitHub: <issue url>`, a blank line, then the
     GitHub issue body as-is (keeps file:line refs, repro steps and evidence)
   - labels: `agent-found`
   - leave priority, assignee and sprint unset
4. Close the GitHub issue with a link:
   `gh issue close <number> --reason "not planned" --comment "Moved to Jira: <ticket url>"`
5. If creating the ticket fails, don't close the GitHub issue. Note the error
   and continue with the next one.

## 4. Report

Table: GitHub issue, Jira key and link, status (created / already existed / failed).
