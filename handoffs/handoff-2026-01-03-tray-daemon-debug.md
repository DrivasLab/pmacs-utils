# Session Handoff — 2026-01-03 Tray/Daemon Debug

## Completed

### Desktop Shortcuts Fixed
- Shortcuts were using PowerShell wrapper scripts that stopped working
- Changed `scripts/create-shortcuts.ps1` to create **direct exe shortcuts** instead
- All shortcuts now call `pmacs-vpn.exe` directly with args (connect, tray, disconnect)
- Added `Set-RunAsAdmin` helper to set admin flag on .lnk files programmatically
- Recreated all shortcuts - they now work

### Tray Crash Fixed
- Tray was crashing immediately with panic: "Initializing the event loop outside of the main thread"
- Root cause: `tao` event loop was being spawned on a non-main thread
- Fix in `src/tray.rs`:
  - Added `use tao::platform::windows::EventLoopBuilderExtWindows;`
  - Changed event loop builder to use `.with_any_thread(true)` on Windows
- Tray icon now appears and menu works

## In Progress / Broken

### Daemon Mode Not Working
**Symptom:** Clicking "Connect" in tray → auth succeeds → DUO push works → but daemon crashes silently → tray shows red/disconnected after timeout

**What we know:**
- Foreground mode works perfectly (`pmacs-vpn connect` from admin terminal)
- Daemon mode spawns a child process but it dies immediately
- State file shows `"pid": null` (daemon never saves its PID)
- Auth token file is created correctly

**Fixes attempted (in current build):**
- Added `cmd.current_dir(cwd)` to set working directory for daemon child
- Added stdio null redirects for Windows (was only on non-Windows before)

**Still need to investigate:**
1. Why daemon crashes - no logs since stdout/stderr go to null
2. Possible causes:
   - Tracing subscriber failing to initialize without console
   - Some panic in early daemon startup
   - Admin privileges not inherited by detached process?
3. Next steps:
   - Add file-based logging for daemon mode
   - Or run daemon without DETACHED_PROCESS temporarily to see console output
   - Check if `CREATE_NO_WINDOW` flag works better than `DETACHED_PROCESS`

## Files Changed This Session

| File | Changes |
|------|---------|
| `src/tray.rs` | Added Windows EventLoopBuilderExtWindows import, `.with_any_thread(true)` |
| `src/main.rs` | Added cwd setting for daemon, stdio null redirects for Windows |
| `scripts/create-shortcuts.ps1` | Rewrote to create direct exe shortcuts instead of PowerShell wrappers |

## Test Commands

```powershell
# Admin PowerShell required for all

# Test foreground (works)
cd C:\drivaslab\pmacs-utils
.\target\release\pmacs-vpn.exe connect

# Test tray (icon works, connect fails)
.\target\release\pmacs-vpn.exe tray

# Check status
.\target\release\pmacs-vpn.exe status

# Check state file
Get-Content "$env:USERPROFILE\.pmacs-vpn\state.json"

# Check auth token
Get-Content "$env:USERPROFILE\.pmacs-vpn\auth-token.json"
```

## Git Status

Uncommitted changes:
- `src/tray.rs` - any_thread fix
- `src/main.rs` - daemon spawn improvements
- `scripts/create-shortcuts.ps1` - direct exe shortcuts
