# herdr-claude-usage

Global Claude Code plan usage inside [Herdr](https://herdr.dev) — in one place, not per pane.

```text
  ✳ Claude usage   updated 12s ago

  Current session
  ███████░░░░░░░░░░░░░░░░░░░░░░░░░░  23% used
  Resets tomorrow 2am (Europe/Prague)

  Current week (all models)
  █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  17% used
  Resets Thu Sep 3, 6pm (Europe/Prague)
```

Two surfaces:

- **Tab bar** (always visible): `✳ 5h ▃29% ↻2am · wk ▂18% · Fable ▂15% · extra €▇87%` — mini gauges, the 5h reset hour, per-model weekly windows, and monthly extra-usage credits. A trailing `⁺` means another signed-in profile is also active.
- **Popup dashboard** (on a keybinding): full bars, severity-colored (lavender < 50% < amber < 80% < red), reset times in your local timezone, extra-usage spend (`€260 of €300 this month`), and a compact line per other active profile. Refreshes every 30s; `q` closes.

## How it gets the data

Two sources, gracefully independent:

1. **statusLine push (no credentials)** — Claude Code invokes the configured `statusLine` command with a JSON payload on stdin that includes `rate_limits`. This plugin's `collect` captures it and **chains to whatever statusLine command was installed before**, passing stdin/stdout through untouched.
2. **OAuth usage API (optional, `USE_API=0` disables)** — per-model weekly windows (e.g. Fable) and extra-usage credits only exist in the official usage endpoint. The plugin reads your own Claude Code OAuth token from the macOS keychain (first use triggers a keychain permission prompt) and fetches at most every 5 minutes. The token never leaves the process; only the usage JSON is cached.

Sessions signed into different accounts/organizations report different limits, so payloads are clustered into **profiles** keyed by their weekly reset time. The profile matching the API response is the logged-in account and is shown as primary; other active profiles appear as a compact line in the dashboard and as `⁺` in the tab bar. If nothing has reported for 30 minutes the tab bar shows `✳ Claude —` and the dashboard flags the data as stale.

## Install

```sh
herdr plugin link /path/to/herdr-claude-usage
bash bin/herdr-claude-usage configure     # wraps ~/.claude/settings.json statusLine (backup + chain)
```

Then in `~/.config/herdr/config.toml`:

```toml
[ui]
tab_bar_right = [
  { type = "command", command = "bash /path/to/herdr-claude-usage/bin/herdr-claude-usage bar", interval_seconds = 30, timeout_seconds = 5 },
]

[[keys.command]]
key = "prefix+u"
type = "popup"
command = "bash /path/to/herdr-claude-usage/bin/herdr-claude-usage dashboard"
width = 64
height = 20
```

`herdr server reload-config`, and data appears within a minute of any Claude Code session running.

## Uninstall

```sh
bash bin/herdr-claude-usage uninstall   # restores the previous statusLine
herdr plugin unlink herdr-claude-usage
```

Remove the `tab_bar_right` entry and keybinding from `config.toml`.

## Commands

| Command | Purpose |
| --- | --- |
| `collect` | statusLine hook (invoked by Claude Code; captures payload, chains onward) |
| `bar` | one-line tab-bar summary |
| `dashboard` | interactive popup with bars |
| `draw` | one-shot render (debugging) |
| `configure` / `uninstall` | install/remove the statusLine wrapper, reversibly |
