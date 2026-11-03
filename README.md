# tcal

A terminal calendar for macOS and tmux — calcurse-style TUI, with real account sync like
macOS Calendar (iCloud, Google, any CalDAV server) and upcoming events in your tmux
status line.

> Status: early development. See [docs/PROJECT_PLAN.md](docs/PROJECT_PLAN.md) for the roadmap.

## Features

- Month, week, day and agenda views with vim-style keys
- iCloud, Google and generic CalDAV accounts
- Local SQLite cache: instant startup, works offline
- Background daemon (launchd) for sync and reminders
- Next event and alerts in the tmux status line
- Credentials stored in the macOS Keychain — never in config files

## Requirements

- macOS (Apple Silicon or Intel)
- Go 1.23+ (to build)
- tmux 3.2+ (optional, for the status line)

## Install

```sh
git clone <repo-url> cli-cal
cd cli-cal
make build          # produces ./bin/tcal
make install        # copies to ~/.local/bin (or use: go install ./cmd/tcal)
```

## Add an account

### iCloud

1. Create an app-specific password at <https://appleid.apple.com> → Sign-In and Security →
   App-Specific Passwords.
2. Run `tcal account add icloud` and enter your Apple ID and that password.

### Google

1. Run `tcal account add google`.
2. A browser window opens for Google sign-in; approve access.
3. tcal stores only the refresh token in your Keychain.

_(Until a shared client ID is published, see docs/PROJECT_PLAN.md for creating your own
Google Cloud desktop OAuth client.)_

### Other CalDAV servers (Fastmail, Nextcloud, …)

```sh
tcal account add caldav --url https://caldav.example.com --user you@example.com
```

## Usage

```sh
tcal                  # open the calendar
tcal today            # today's events as text
tcal next             # next upcoming event (one line)
tcal agenda --days 7  # agenda for the next week
tcal sync             # force a sync
```

### Keys

| Key | Action |
| --- | --- |
| `h` `j` `k` `l` | Move |
| `m` `w` `d` `a` | Month / week / day / agenda view |
| `t` | Jump to today |
| `n` / `e` / `x` | New / edit / delete event |
| `/` | Search |
| `c` | Toggle calendars |
| `?` | Help |
| `q` | Quit |

## Background sync

```sh
tcal install-agent    # starts the daemon at login via launchd
tcal uninstall-agent
```

Logs: `~/Library/Logs/tcal/`

## tmux integration

With the daemon running (pushes the value, no polling):

```tmux
set -g status-right '#{@tcal_next} | %H:%M'
```

Without the daemon (polls every 60 s):

```tmux
set -g status-interval 60
set -g status-right '#(tcal next) | %H:%M'
```

## Configuration

`~/.config/tcal/config.toml` — no secrets in here.

```toml
week_start = "monday"
time_format = "24h"
sync_interval = "5m"

[tmux]
enabled = true
alert_minutes = 10
```

## Security

Passwords and tokens live only in the macOS Keychain. See [docs/SECURITY.md](docs/SECURITY.md).

## Development

```sh
make test
make lint
make run
```

Working with Claude Code: project instructions live in a local `CLAUDE.md` (git-ignored).

## License

TBD
