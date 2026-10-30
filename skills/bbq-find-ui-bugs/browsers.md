# Browser backends for bbq-find-ui-bugs

Use whichever backend was picked in SKILL.md. Each row maps a scan step to that
backend's tool. Tool names are shown without their prefix:
Playwright `mcp__playwright__`, Chrome `mcp__claude-in-chrome__`, Safari `mcp__safari-mcp__`.

| Step | Playwright | Chrome (Claude in Chrome) | Safari MCP |
|---|---|---|---|
| Open a tab | `browser_navigate` (own browser) | `tabs_context_mcp`, then `tabs_create_mcp` | `create_tab` |
| Go to URL | `browser_navigate` | `navigate` | `navigate_to_url` |
| Read the page | `browser_snapshot` | `read_page` (`filter: "interactive"` for controls), `find` | `get_page_content` |
| Click, type, hover | `browser_click`, `browser_type`, `browser_hover`, `browser_fill_form` | `computer` (by `ref`), `form_input` | `page_interactions` (batch them) |
| Console errors | `browser_console_messages` | `read_console_messages` with `onlyErrors: true`, `pattern: "."` | `browser_console_messages` with `level_filter: ["error"]` |
| Failed requests | `browser_network_requests` | `read_network_requests` (look for status >= 400) | `list_network_requests` with `filter: {status_min: 400}` |
| Screenshot | `browser_take_screenshot` | `computer` with `action: "screenshot"` | `screenshot` |
| Run JS | `browser_evaluate` | `javascript_tool` | `evaluate_javascript` |
| Phone width | `browser_resize` 390x844 | `resize_window` 390x844 (see below) | `set_viewport_size` 390x844 |
| Dark mode | `browser_emulate_media` `colorScheme: "dark"` | not supported (see below) | not supported (see below) |
| Clean up | `browser_close` | `tabs_close_mcp` on tabs you opened | `close_tab` on tabs you opened |

## Differences that matter

**Playwright** runs its own browser with no saved sessions. Logins come from the
Keychain (see SKILL.md). It's the only backend that can emulate dark mode.

**Chrome** is my real Chrome profile, so I'm probably already logged in. Try the
page before looking up Keychain logins. Because it's my real account, anything
you submit is done as me: follow the form rule in SKILL.md strictly. Load the
`anthropic-skills:chrome-browser` skill before the first Chrome call.
- Phone width: `resize_window` resizes the whole window, and Chrome usually won't
  go below about 500px. After resizing, check `window.innerWidth` with
  `javascript_tool`. If it's over 430, say phone width wasn't fully covered.
- Dark mode: no emulation. See the dark-mode fallback below.

**Safari** is my real Safari, so the same login and form notes as Chrome apply.
Console and network data come from Web Inspector, so they only cover requests
made after the tab was opened. Reload the page before reading them.
- Dark mode: `set_emulated_media` only switches screen/print, not color scheme.
  See the fallback below.

## Dark-mode fallback (Chrome and Safari)

Don't change the macOS appearance setting. Instead, check whether the site has
its own theme switch:
- a visible theme toggle: use it
- a class or attribute on `<html>` (e.g. Tailwind `dark`, `data-theme="dark"`):
  set it with JS, then check the pages

If neither exists, skip the dark-mode pass and say so in the final summary.
Switch the site back to light afterwards.
