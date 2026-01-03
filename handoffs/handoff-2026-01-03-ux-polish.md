# Session Handoff — 2026-01-03 UX Polish

## Completed

### Daemon Mode Fix
- Parent now does auth (password + DUO) in foreground before spawning daemon
- Auth token (`AuthToken` struct) saved to `~/.pmacs-vpn/auth-token.json`
- Daemon child reads token, establishes tunnel without prompts
- Token includes: gateway, username, auth_cookie, portal, domain, hosts, keep_alive
- Token auto-expires after 5 minutes for security
- New function: `gp::auth::getconfig_with_cookie()` for daemon use

### Admin Privilege Check
- Added `is_admin()` check at CLI entry point
- Clear error message with suggestions (scripts/connect.ps1 or sudo)
- Applies to: connect, disconnect, tray commands

### Tray Improvements
- Checks for cached password before attempting connect
- Shows clear error if no config or no cached password
- Replaced hardcoded 5s wait with poll-based status check (500ms × 20)
- Better error messages passed to tray status

### Scripts
- `scripts/tray.ps1` (NEW) - Auto-elevates, runs hidden, launches tray
- `scripts/connect.ps1` - Now checks if already connected, clearer messaging
- `scripts/create-shortcuts.ps1` (NEW) - Creates desktop shortcuts

### Desktop Shortcuts Created
- PMACS VPN Tray.lnk
- PMACS VPN Connect.lnk
- PMACS VPN Disconnect.lnk

### Documentation
- README.md rewritten - clearer setup, troubleshooting section
- TODO.md updated with completed items
- CLAUDE.md updated with current status

## Key Files Changed

| File | Changes |
|------|---------|
| `src/main.rs` | Admin check, spawn_daemon auth flow, connect_vpn_with_token |
| `src/state.rs` | AuthToken struct for daemon auth passing |
| `src/gp/auth.rs` | getconfig_with_cookie function |
| `src/lib.rs` | Export AuthToken |
| `scripts/tray.ps1` | NEW - tray launcher |
| `scripts/create-shortcuts.ps1` | NEW - shortcut creator |

## In Progress / Deferred

### macOS Testing (Next Priority)
- Code exists but untested
- Need to test: TUN device (utun), routing, hosts file, Keychain
- Platform code in `src/platform/mac.rs`

### Linux Testing
- Similar to macOS - code exists but untested
- Need to test: TUN device, ip route, hosts file, Secret Service

### Better Error Messages
- Admin check done
- Still TODO: DNS failures, DUO timeout hints, network errors

## Technical Notes

### Auth Token Flow
```
Parent Process (has console):
  1. Load config
  2. Get password (cached or prompt)
  3. Do auth: prelogin → login
  4. Save AuthToken to ~/.pmacs-vpn/auth-token.json
  5. Spawn detached child with --_daemon-pid flag
  6. Exit

Daemon Child (no console):
  1. Read AuthToken from file
  2. Delete token file immediately
  3. Call getconfig_with_cookie
  4. Establish tunnel
  5. Run until killed
```

### Tray Connect Flow
```
1. Check config exists
2. Check username in config
3. Check password cached for username
4. If any missing → show error, don't spawn
5. Call spawn_daemon (async)
6. Poll VpnState every 500ms for up to 10s
7. Update status when daemon running
```

## Test Status
- 70 tests passing
- Clippy clean
- Release build succeeds

## For Next Session
1. Test on macOS - TUN, routes, keychain
2. Test on Linux - similar
3. Consider GUI password prompt for tray (currently requires cached password)
