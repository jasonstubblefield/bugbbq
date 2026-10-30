# Checking a UI fix in a browser

Shared by bbq-fix-bugs and bbq-review-fix-prs, which read it from the
bbq-find-ui-bugs skill folder installed next to theirs. Use it for UI bugs: issues labeled `ui`, or older bbq-find-ui-bugs
issues without the label whose body has a page URL, a "found in" browser line
and reproduction steps.

## 1. Serve the fixed code

The URL in the issue runs the unfixed code. You need a URL that serves the
worktree with the fix.

- A worktree starts without dependencies or built assets. Make sure
  dependencies are installed and front-end assets are built if the project
  needs them (e.g. `npm run build`).
- Herd: if the main checkout is served by Herd as `<dir>.test`, the worktree
  next to it is probably served as `<worktree-dir>.test`. Load it and check
  it's really the worktree's code (e.g. the fixed file's change is visible).
  The copied `.env` may need `APP_URL` pointed at that host.
- Otherwise, start the project's dev server from the worktree in the
  background, on a free port.
- If you can't get it running after a couple of tries, ask me for the URL.

## 2. Reproduce

1. Pick a browser the same way bbq-find-ui-bugs does (Playwright, then Chrome, then
   Safari), using `browsers.md` in the `bbq-find-ui-bugs` skill folder for the
   tool names. Follow the "Pick a browser" section of that skill's `SKILL.md`,
   including its Playwright install note. If no browser is available, report the
   UI check as "couldn't check" with the install instructions, instead of
   stopping the whole fix.
2. Take the path from the issue's page URL and open it on the local URL.
3. Follow the issue's reproduction steps. Check console errors and failed
   network requests again.
4. If the bug was about dark mode or phone width, check in that mode.
5. Take a screenshot of the fixed state if it's a layout or dark-mode bug.

## 3. Result

One of:
- **verified**: the reproduction no longer shows the bug, and nothing new broke
  on that page
- **still broken**: what you saw
- **couldn't check**: why (e.g. needs production data, needs a login and I
  wasn't there to do it, the app wouldn't start)

If the page needs a login, follow the Logins section of bbq-find-ui-bugs:
ask me to log in in the Chrome or Safari tab. Never look up or fill in a
password yourself.

Put the result in the PR body or a PR comment: `UI check (<browser>): <result>`.

Stop any dev server you started and close the tabs you opened.
