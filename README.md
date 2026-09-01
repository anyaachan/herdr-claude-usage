# herdr-claude-usage

Global Claude Code plan usage inside [Herdr](https://herdr.dev), in one place, not per pane.

```text
✳ 5h 62% ↻8pm · wk 32% · Fable 34% · € 0%
```

## Features

- **Tab-bar summary**, always visible: session window with its reset hour, week, per-model weekly windows, monthly extra-usage credits. A percentage above 80 gets a `!!` suffix; a trailing `⁺` means another signed-in profile is also active.
- **Popup dashboard** on a keybinding: severity-colored bars, reset times in your local timezone, extra-usage spend with amounts, active session count, one compact line per other signed-in profile. Refreshes every 30s; `q` closes.
- **Multi-account aware**: payloads are clustered into per-account profiles; the logged-in account is primary, others stay visible without hijacking the display.
- **No credentials required** for the core data; an optional macOS-only usage-API fetch adds per-model windows and spend.
- **Reversible install**: the statusLine wrapper chains to whatever was installed before and restores it on uninstall.

## Installation

Requirements: Herdr 0.8+, `bash`, `jq`, `curl`. macOS or Linux.

```sh
herdr plugin install anyaachan/herdr-claude-usage   # or: herdr plugin link /path/to/checkout
bash bin/herdr-claude-usage configure               # wraps ~/.claude/settings.json statusLine (backup + chain)
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

Run `herdr server reload-config`. Data appears within a minute of any Claude Code session running.

## Usage

The tab bar updates on its own every 30s. `prefix+u` opens the dashboard:

```text
  ✳ Claude usage   updated 12s ago · 8 session(s)

  Current session
  █████████░░░░░░░░░░░░░░░░░░░░░░░░░  29% used
  Resets today 3pm (Europe/Prague)

  Current week (all models)
  ███████░░░░░░░░░░░░░░░░░░░░░░░░░░░  22% used
  Resets Thu Sep 3, 6pm (Europe/Prague)

  Current week (Fable)
  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  20% used
  Resets Thu Sep 3, 6pm (Europe/Prague)

  Extra usage (€260 of €300 this month)
  █████████████████████████████░░░░░  87% used

  Other profile (1 session(s) · rover): 4% session · 54% week
```

Bars are lavender below 50%, amber from 50%, red from 80%.

## Configuration

Optional `config` file in the plugin config dir (`herdr plugin config-dir herdr-claude-usage`), shell syntax. The same variables work as environment overrides.

```sh
USE_API=1        # 0 disables the OAuth usage API fetch
BAR_HOT=80       # tab-bar '!!' suffix and dashboard red above this
BAR_WARN=50      # dashboard amber from this
STALE_SECS=1800  # how long a profile counts as fresh
```

## How it works

Two independent data sources:

1. **statusLine push (no credentials, all platforms).** Claude Code invokes the configured `statusLine` command with a JSON payload on stdin that includes `rate_limits`. This plugin's `collect` captures it and chains to whatever statusLine command was installed before, passing stdin and stdout through untouched.
2. **OAuth usage API (optional, macOS only).** Per-model weekly windows (e.g. Fable) and extra-usage credits exist only in the official usage endpoint. The plugin reads your own Claude Code OAuth token from the macOS keychain (first use triggers a keychain permission prompt) and fetches at most every 5 minutes, backing off failures for the same window. The token never leaves the process; only the usage JSON is cached. On Linux the plugin runs on statusLine data alone.

**Several accounts.** Sessions signed into different accounts or organizations report different limits, so payloads are clustered into profiles keyed by their weekly reset time. The profile whose weekly reset matches the usage API response is the logged-in account and is shown as primary; without API data, the profile with the most active sessions wins. Profiles quiet for 30 minutes are dropped from the display.

**The 5h reading never disappears.** Claude Code sometimes omits the `five_hour` window from a payload; the previous unexpired window is kept. When the window has genuinely expired, the displays show `5h 0%`.

## Commands

| Command | Purpose |
| --- | --- |
| `collect` | statusLine hook (invoked by Claude Code; captures payload, chains onward) |
| `bar` | one-line tab-bar summary |
| `dashboard` | interactive popup with bars |
| `draw` | one-shot render (debugging) |
| `configure` / `uninstall` | install/remove the statusLine wrapper, reversibly |

## Troubleshooting

- **Tab bar shows `✳ Claude —`**: no Claude Code session has reported for 30 minutes. Open or use any session; data lands within a minute.
- **No Fable / extra-usage entries**: the usage API needs the macOS keychain. Approve the keychain prompt on first fetch, or check `USE_API` is not `0`. On Linux these entries are unavailable.
- **A keychain dialog appeared**: that is the first usage-API fetch reading your own Claude Code OAuth token. "Always Allow" stops it from asking again.
- **Percentages look uncolored in the tab bar**: expected. The tab bar renders no ANSI colors; `!!` above 80 is the only marker there. The dashboard has the colors.

## Uninstall

```sh
bash bin/herdr-claude-usage uninstall   # restores the previous statusLine
herdr plugin unlink herdr-claude-usage
```

Remove the `tab_bar_right` entry and keybinding from `config.toml`.
