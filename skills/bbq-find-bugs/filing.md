# Filing a bug as a GitHub issue

Shared by bbq-find-bugs and bbq-find-ui-bugs. It lives in the bbq-find-bugs
skill folder; the skills install side by side, so bbq-find-ui-bugs reads it from
`../bbq-find-bugs/filing.md`. The calling skill says what goes in the issue body.

## Once per run, before the first issue

1. Run `gh auth status`. If it fails, stop and tell me.
2. Make sure every label you'll use exists, or `gh issue create` will fail:
   `gh label list --limit 200 --json name -q '.[].name'`
   - missing `bug`: `gh label create bug --color d73a4a --description "Something isn't working"`
   - missing `agent-found`: `gh label create agent-found --color 5319e7 --description "Found by an agent scan"`
   - missing `ui` (bbq-find-ui-bugs only): `gh label create ui --color 1d76db --description "User interface bug"`

3. Look at how open issues are titled (`gh issue list --state open --limit 50
   --json title -q '.[].title'`) and reuse the area names already in use.

## Title and summary

I read the issue list before I read any issue, so the title alone has to tell
me what part of the app is broken, who notices, and what they notice.

Title: `<Area>: <what goes wrong, in plain words>`

- **Area** is the part of the product as a user or I would name it, such as Search,
  Saved recipes, Menus, Shopping lists, Web clipper, Billing, MCP, Homepage,
  Mobile, Accessibility, SEO, Security. Never a class or file name.
- Then say what someone sees or loses, and who, if it isn't everyone. Add the
  trigger when it's narrow ("when saving a recipe twice").
- No class, method, column or config names, and no fix. The code goes in the
  body.
- Aim for under 80 characters. One bug per issue; if it needs "and also", it's
  two issues.

| Instead of | Write |
| --- | --- |
| Saving an index recipe again hides its already-extracted ingredients | Saved recipes: saving a recipe twice drops its ingredient amounts |
| Activity log ignores the configured Mongo database, so prod and dev share browsing history by user id | Activity history: prod users can see a dev account's browsing history |
| Tap targets under 24px at 390px: "Try:" links, "Show more", Save/Details/Add to Menu, 10px connector text | Mobile: search buttons and links are too small to tap |
| Homepage "View All" links point to /recipes with no query, which redirects back to / | Homepage: "View All" links go back to the homepage |

The body starts with these, before the calling skill's technical sections:

```
## Summary

<Two or three plain sentences: who is affected, what they see or lose, and
what it costs (money, data, privacy, a broken flow, confusion). No code.>

**Severity:** high | medium | low, <one-line reason>
**How often:** <every visit | common flow | rare edge case>, <one-line reason>
**Found by:** <calling skill> on <YYYY-MM-DD>, <what was scanned>
```

Severity: **high** is lost or leaked data, money charged wrongly, security, or
a main flow that doesn't work; **medium** is a feature that's wrong or broken
for some users; **low** is cosmetic, minor, or only seen in a rare case.

## For each bug

1. Search for duplicates with `gh issue list --state all --search "<keywords>"`.
   Try two or three keyword sets (the symptom, the file or page, the component).
   If a near-match exists, say so in the next step.
2. Ask with AskUserQuestion, one bug per question. Make the question stand on
   its own, since the dialog may be all I can see: the exact title it will be
   filed with, the Summary paragraph, severity, where it is, and any
   near-match issue numbers. Options:
   - **File it**: file as described
   - **Skip**: move on to the next bug
   - **Edit first**: ask me what to change, then ask again
3. On **File it**, run `gh issue create --label bug,agent-found --title "<title>"`
   (plus any extra label the calling skill names, e.g. `--label bug,agent-found,ui`)
   with the body passed through `--body-file -` and a quoted heredoc so
   backticks and `$` survive. Show me the issue URL it prints.
4. On **Skip**, move on.

One at a time. Wait for my answer before each one. Do not batch.

At the end, list what was filed (number, severity and title) and what was
skipped.
