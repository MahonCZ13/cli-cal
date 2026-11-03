# tcal — Project Plan

_Terminal calendar for macOS + tmux. Created 2026-11-03._

## 1. Project brief

Build a terminal-only calendar (working name **tcal**) that feels like calcurse but connects
to online accounts the way macOS Calendar does, and pushes upcoming events into the tmux
status line.

### Goals

- Full TUI calendar: month, week, day and agenda views, keyboard-driven (vim-style keys).
- Add online accounts from inside the app: iCloud, Google, and any generic CalDAV server.
- Two-way sync with a local cache, so the app is instant and works offline.
- Reminders and "next event" in the tmux status line, with optional macOS notifications.
- Credentials kept in the macOS Keychain, never in plain text.

### Non-goals (for now)

- No GUI, web UI or mobile app.
- No Exchange/Outlook (EWS / Microsoft Graph) in v1.
- No tasks/to-dos (VTODO) until the calendar is solid.
- No server component: it runs on one Mac, for one person.

**Target platform:** macOS (Apple Silicon), tmux, any modern terminal. Linux stays possible
by isolating the Keychain and launchd code behind interfaces.

## 2. Architecture

One Go binary with three roles. **Only the daemon talks to the network**; everything else
reads and writes the local cache.

```
            ┌──────────────────────────────────────────────┐
            │ Online accounts: iCloud · Google · CalDAV    │
            └──────────────────────┬───────────────────────┘
                                   │ HTTPS only
┌──────────────────────────────────┴──────┐      ┌──────────────────────┐
│ tcal daemon (launchd agent)             │─────▶│ macOS Keychain       │
│  - sync loop (ETags + sync-tokens)      │      │  tokens, app pwds    │
│  - reminder scheduler                   │      └──────────────────────┘
│  - pushes next event to tmux            │
└───────────┬─────────────────────┬───────┘
            │                     │ tmux set -g @tcal_next "…"
┌───────────┴───────────┐   ┌─────┴────────────────┐
│ Local cache (SQLite)  │◀─▶│ TUI (`tcal`)         │   tmux status-right: #{@tcal_next}
│ events, calendars,    │   │ month/week/day/agenda│
│ pending changes       │   └──────────────────────┘
└───────────────────────┘
```

### Stack

| Concern | Choice | Why |
| --- | --- | --- |
| Language | Go 1.23+ | Single static binary, fast startup, easy concurrency |
| TUI | Bubble Tea + Lip Gloss + Bubbles | Mature Elm-style TUI, easy theming |
| CalDAV client | `github.com/emersion/go-webdav` | Discovery, REPORT queries, sync-collection |
| iCalendar | `github.com/emersion/go-ical` | Reads/writes VEVENT, VALARM |
| Recurrence | `github.com/teambition/rrule-go` | RRULE, EXDATE, RDATE expansion |
| Google | `golang.org/x/oauth2` (+ Calendar API, optional path) | OAuth 2.0 with PKCE, loopback redirect |
| Storage | `modernc.org/sqlite` | Pure Go, no cgo |
| Secrets | `github.com/zalando/go-keyring` | macOS Keychain, Linux Secret Service |
| Config | TOML at `~/.config/tcal/config.toml` | Human-editable, no secrets inside |
| Background | launchd agent in `~/Library/LaunchAgents` | Starts at login, restarts on crash |

### Commands

- `tcal` — open the TUI.
- `tcal daemon` — sync + reminder loop (normally started by launchd).
- `tcal next`, `tcal today`, `tcal agenda --days 7` — plain text for scripts and tmux.
- `tcal account add|list|remove` — manage accounts without the TUI.
- `tcal sync` — force a sync now.
- `tcal install-agent` / `tcal uninstall-agent` — manage the launchd plist.

### Project layout

```
cli-cal/
  cmd/tcal/            main.go, subcommand wiring
  internal/config/     TOML config loading
  internal/secrets/    Keychain wrapper (interface + macOS impl)
  internal/provider/   caldav/, google/ — one interface, two impls
  internal/store/      SQLite schema, migrations, queries
  internal/sync/       two-way sync engine, conflict handling
  internal/recur/      recurrence expansion, time zones
  internal/notify/     tmux + macOS notification sinks
  internal/tui/        views: month, week, day, agenda, editors
  docs/                PROJECT_PLAN.md, SECURITY.md, STARTER_PROMPT.md
  testdata/            .ics fixtures, recorded CalDAV responses
```

### Key design decisions

- **Recurrence is expanded at read time**, not stored as copies. The cache stores the master
  event + overrides (RECURRENCE-ID); `internal/recur` expands a date window on demand.
- **All times stored in UTC + original TZID.** Display in local time. DST is tested explicitly
  (Europe/Prague last Sunday of March / October).
- **Edits are queued locally** (`pending_changes` table) and pushed by the daemon, so the TUI
  never blocks on the network.
- **Conflicts:** server wins by default; the local version is kept as a conflict copy and shown
  in the TUI for manual resolution.

## 3. Security model

See `docs/SECURITY.md` for the full rules. Summary:

- Secrets only in Keychain (service `tcal`); config holds account IDs only.
- iCloud via app-specific password; Google via OAuth PKCE, store only the refresh token.
- Start with read-only scopes; request write access only in phase 3.
- HTTPS only, system CA store; never log tokens or Authorization headers.
- Cache/config files 0600, dirs 0700.

## 4. Phased plan

Each phase ends with a usable app and a git tag.

### Phase 0 — Skeleton (v0.0.1)

- [ ] `go mod init`, folder layout, Makefile (`build`, `test`, `lint`, `run`)
- [ ] Subcommand wiring (cobra or stdlib `flag`), `tcal version`
- [ ] Config loading with defaults; XDG / macOS paths
- [ ] SQLite store with migrations (calendars, events, sync_state, pending_changes)
- [ ] `secrets` interface + Keychain implementation + in-memory fake for tests
- [ ] CI: `go vet`, `staticcheck`, `govulncheck`, `go test ./...`

**Done when:** `make build` produces a binary, tests pass, `tcal version` works.

### Phase 1 — Read-only MVP (v0.1.0)

- [ ] `tcal account add caldav` and `tcal account add icloud` (URL discovery via
      `/.well-known/caldav`, principal → calendar-home-set → calendars)
- [ ] One-way sync down: initial full fetch, then incremental via sync-token / ETags
- [ ] Recurrence expansion for a date window (RRULE, EXDATE, overrides)
- [ ] TUI: agenda view + month view, navigation (h/j/k/l, t = today, q = quit)
- [ ] Event detail pane (title, time, location, notes, calendar)
- [ ] `tcal next` / `tcal today` text output
- [ ] tmux snippet in README using `#(tcal next)` (polling version)

**Done when:** iCloud calendar shows correctly for 3 months ahead, including a recurring
event with one moved occurrence; tmux shows the next event.

### Phase 2 — Accounts & daemon (v0.2.0)

- [ ] Google account: OAuth PKCE loopback flow, CalDAV endpoint or Calendar API
- [ ] Multiple accounts and calendars; per-calendar colour + visibility toggle
- [ ] Account manager screen in the TUI (add / list / remove)
- [ ] `tcal daemon` with sync interval, backoff on errors, offline detection
- [ ] `tcal install-agent` writes launchd plist; logs to `~/Library/Logs/tcal`
- [ ] Daemon pushes `@tcal_next` to tmux (no polling)
- [ ] Week and day views

**Done when:** iCloud + Google calendars show side by side with colours; closing the TUI
keeps tmux updated.

### Phase 3 — Editing & two-way sync (v0.3.0)

- [ ] Create / edit / delete events (form in TUI, quick-add `tcal add "Lunch tomorrow 12:30"`)
- [ ] Recurring edits: "this event" vs "all events"
- [ ] Pending-changes queue, push via PUT with If-Match ETags
- [ ] Conflict detection + conflict view in TUI
- [ ] Request Google write scope only now
- [ ] Search (`/`) across cached events

**Done when:** an event created in tcal appears on iPhone within one sync interval, and an
edit made on both sides produces a visible conflict instead of silent data loss.

### Phase 4 — Reminders & polish (v1.0.0)

- [ ] VALARM support: reminders fired by the daemon
- [ ] tmux alerts via `tmux display-message`; status segment changes colour when < 10 min
- [ ] Optional macOS notifications (`osascript` / `terminal-notifier`)
- [ ] Themes (light/dark, calcurse-like), configurable keybindings
- [ ] Read-only display of invites (attendees, organizer); RSVP later
- [ ] Homebrew tap or `go install` instructions; man page / `tcal help`

**Done when:** you use tcal daily for a week without opening macOS Calendar.

### Later / ideas

- VTODO tasks pane; Obsidian daily-note integration
- Free/busy view, time-zone side-by-side
- ICS import/export, subscribe to read-only webcal URLs (holidays, sports)
- Linux support (Secret Service, systemd user unit)

## 5. Risks

| Risk | Mitigation |
| --- | --- |
| Recurrence/time-zone bugs | Library for RRULE, fixture-based tests incl. DST weeks |
| Google OAuth setup friction | Document GCP project + desktop client ID step by step; CalDAV fallback |
| iCloud CalDAV quirks | Record real responses into `testdata/` and test against them |
| Data loss on two-way sync | Phase 1–2 read-only; ETags + conflict copies in phase 3; never delete without confirm |
| Scope creep | Each phase has a "done when"; ideas go to the Later list |
