---
name: bbq-find-ui-bugs
description: Scan a running site in Playwright, Chrome or Safari for real UI/frontend bugs and file them as GitHub issues, one at a time with confirmation
argument-hint: "[url and pages/flows to scan] [--browser playwright|chrome|safari]"
disable-model-invocation: true
allowed-tools: >-
  Bash(gh auth status:*), Bash(gh label list:*), Bash(gh label create:*),
  Bash(gh issue list:*), Bash(gh issue view:*), Bash(gh issue create:*),
  mcp__playwright__browser_navigate, mcp__playwright__browser_navigate_back,
  mcp__playwright__browser_snapshot, mcp__playwright__browser_take_screenshot,
  mcp__playwright__browser_console_messages, mcp__playwright__browser_network_requests,
  mcp__playwright__browser_network_request, mcp__playwright__browser_click,
  mcp__playwright__browser_hover, mcp__playwright__browser_type,
  mcp__playwright__browser_fill_form, mcp__playwright__browser_select_option,
  mcp__playwright__browser_press_key, mcp__playwright__browser_find,
  mcp__playwright__browser_wait_for, mcp__playwright__browser_resize,
  mcp__playwright__browser_emulate_media, mcp__playwright__browser_evaluate,
  mcp__playwright__browser_handle_dialog, mcp__playwright__browser_tabs,
  mcp__playwright__browser_close,
  mcp__claude-in-chrome__tabs_context_mcp, mcp__claude-in-chrome__tabs_create_mcp,
  mcp__claude-in-chrome__tabs_close_mcp, mcp__claude-in-chrome__navigate,
  mcp__claude-in-chrome__read_page, mcp__claude-in-chrome__find,
  mcp__claude-in-chrome__get_page_text, mcp__claude-in-chrome__computer,
  mcp__claude-in-chrome__form_input, mcp__claude-in-chrome__javascript_tool,
  mcp__claude-in-chrome__read_console_messages, mcp__claude-in-chrome__read_network_requests,
  mcp__claude-in-chrome__resize_window,
  mcp__safari-mcp__create_tab, mcp__safari-mcp__close_tab, mcp__safari-mcp__list_tabs,
  mcp__safari-mcp__switch_tab, mcp__safari-mcp__navigate_to_url,
  mcp__safari-mcp__wait_for_navigation, mcp__safari-mcp__page_info,
  mcp__safari-mcp__get_page_content, mcp__safari-mcp__page_interactions,
  mcp__safari-mcp__screenshot, mcp__safari-mcp__evaluate_javascript,
  mcp__safari-mcp__browser_console_messages, mcp__safari-mcp__list_network_requests,
  mcp__safari-mcp__get_network_request, mcp__safari-mcp__set_viewport_size
---

Scan $ARGUMENTS for real UI/frontend bugs. Ignore style nits, minor spacing,
and cosmetic polish. Report only genuine defects.
If $ARGUMENTS is empty, ask me which URL (and which pages/flows) to scan before starting.

## Pick a browser

If $ARGUMENTS has `--browser playwright|chrome|safari`, use that one. Otherwise
use the first one available, in this order:
1. **Playwright** (`mcp__playwright__*`): a clean, separate browser, and the only
   one that can emulate dark mode
2. **Chrome** (`mcp__claude-in-chrome__*`): my real Chrome, already logged in
3. **Safari** (`mcp__safari-mcp__*`): my real Safari, already logged in

A tool can be listed as deferred rather than loaded. Load what you need with
ToolSearch in one call, and treat a backend as available only if its tools
load and its first call works (Chrome's extension or Safari's automation can be
off). If one fails, fall back to the next and tell me which one you're using.

If Playwright isn't available:
- **Chrome or Safari is available**: start the scan with it, but first print
  this note, then carry on without waiting:

  > For better results, install Playwright MCP. It runs a clean, separate
  > browser that doesn't act as your logged-in account, and it's the only one
  > that can test dark mode:
  >
  > `claude mcp add -s user playwright -- npx -y @playwright/mcp@latest --browser chrome --headless`
  >
  > Then start a new session (or reconnect with `/mcp`). On a machine without
  > Google Chrome, leave out `--browser chrome` and run
  > `npx playwright install chromium` once.

- **Neither is available**: stop. Tell me which of the three you tried and why
  each failed, give me the same install instructions, and ask me to install
  Playwright and rerun the scan. Don't install it yourself.

Read `browsers.md` in this skill's folder for the tool names for each step and
for each backend's limits.

## Stop early and tell me if

- `gh auth status` fails
- none of the three browsers is available (see "Pick a browser" for what to say)
- the site needs a login, I'm not already logged in, and the macOS Keychain
  has none for this host

## Logins

If the site asks for basic auth or a login, look it up in the login keychain:
- username: `security find-internet-password -s <host> | awk -F'"' '/"acct"/{print $4}'`
- password: `security find-internet-password -s <host> -w`
- if there's no internet password, try `security find-generic-password -s <host> -w`

Use the password only to log in. Never repeat it in chat, screenshots or issue bodies.

## What counts as a real UI bug

- JS console errors or unhandled promise rejections on page load or interaction
- Network requests returning 4xx/5xx for assets the page needs (images, CSS, JS,
  XHR/fetch), and broken `<img>` tags that fall through to error handlers
- Dead or wrong-target links (404, or navigates to an unintended route, e.g.
  singular/plural path mismatches)
- Forms that submit but silently fail, or that surface validation errors with
  no visible feedback
- Layout breakage that makes content unreachable (overlap that hides a button,
  z-index traps, off-screen content with no scroll, unclickable targets)
- Accessibility blockers (missing alt on content images, form inputs with no
  label association, focus traps, keyboard-inaccessible controls)
- Dark-mode regressions where text becomes unreadable against the background

## What does NOT count

- Off-by-a-few-pixels alignment
- Minor copy/grammar issues
- Subjective design preferences
- Warnings (not errors) in the console unless they indicate real failures

## Scan approach

1. Open a new tab, go to the target URL, read the page, and check console errors
   and failed network requests.
2. Exercise the main navigation: header/footer links, primary CTAs, search box.
   For each page, repeat the console/network check.
3. If I gave specific flows, prioritize those.
4. Don't submit forms that send email, take payment, or create real records on a
   production site without asking me first. In Chrome or Safari you're acting
   as me, logged in as me.
5. Dark mode: turn it on (or use the fallback in browsers.md), then revisit the
   home page and the main pages you covered. Look for unreadable text, invisible
   icons and borders, and hard-coded white backgrounds. Switch back to light after.
6. Phone width: resize to 390x844, revisit the same pages, and open the mobile
   menu if there is one. Look for horizontal scroll, overlapping or unreachable
   controls, and a nav that won't open or close. Resize back after.
7. For suspected layout issues, take a screenshot to confirm before reporting.
8. For suspected link bugs, click the link and verify the resulting URL/status.
9. Close the tabs you opened.

## Filing

Follow `filing.md` in the `bbq-find-bugs` skill folder (installed next to this
one, e.g. `../bbq-find-bugs/filing.md`), with the extra label `ui` so the fix
skills know to check the fix in a browser. Titles and the Summary block follow
filing.md; use the page or flow as the area when no product area fits better
(e.g. "Checkout: ...", "Mobile: ..."). After the Summary, the issue body
contains:
- the page URL, and the viewport and color scheme if they matter
- which browser it was found in (Chromium via Playwright, Chrome, or Safari)
- reproduction steps (what I click, what breaks)
- observed vs. expected
- any console/network evidence you captured

At the end, also say which browser you used and any pass you couldn't do
fully (dark mode, phone width).
