# Security rules

These rules are non-negotiable. Any change that touches accounts, networking, storage or
logging must follow them.

| Area | Rule |
| --- | --- |
| Credentials | Stored only in macOS Keychain under service `tcal`, one item per account. Config holds an account ID, never a password or token. |
| iCloud | Apple ID + app-specific password (appleid.apple.com → Sign-In and Security → App-Specific Passwords). Never the main Apple ID password. |
| Google | OAuth 2.0 installed-app flow with PKCE and a loopback redirect (127.0.0.1, random port). Store only the refresh token. |
| Scopes | Start read-only (`calendar.readonly`). Write scope only from phase 3. |
| Transport | HTTPS only, system CA store, TLS 1.2+. Refuse `http://` CalDAV URLs unless the host is localhost. Never disable certificate verification. |
| Logs | Never log Authorization headers, tokens, passwords or full event bodies. All logging goes through one redacting helper. |
| Files | Cache and config created with mode 0600, directories 0700. Cache in `~/Library/Application Support/tcal`. |
| Input | Treat all server data (iCalendar text, URLs, HTML in descriptions) as untrusted; strip terminal escape sequences before rendering. |
| Dependencies | Pinned in `go.sum`; `govulncheck` runs in CI. |
| Removal | `tcal account remove` deletes the Keychain item, revokes the Google token, and purges that account's cached events. |
| Tests | Tests never use real credentials; use the in-memory secrets fake and recorded fixtures. |
