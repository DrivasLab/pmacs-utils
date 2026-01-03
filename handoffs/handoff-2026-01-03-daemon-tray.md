# Session Handoff — 2026-01-03 Daemon & Tray Implementation

## Completed

### Daemon Mode
- `--daemon` flag spawns detached background process
- PID stored in `~/.pmacs-vpn/state.json`
- `disconnect` kills daemon by PID via taskkill/kill
- `status` shows "Running (PID: X)" or "Stopped (stale PID: X)"
- Platform-specific: Windows uses CREATE_NEW_PROCESS_GROUP | DETACHED_PROCESS

### Session Watchdog
- `--keep-alive` flag uses 10s keepalive interval (vs 30s default)
- Session start time tracked in tunnel
- Warnings printed at 15hr, 15hr15, 15hr30, etc until 16hr limit
- Connection info (TUN device, IP, session expiry) shown on connect

### System Tray (`pmacs-vpn tray`)
- Uses `tray-icon` + `tao` crates
- Colored circle icons: gray (disconnected), green (connected), orange (connecting), red (error)
- Right-click menu: Status, Connect, Disconnect, Exit
- Channels for communication between tray thread and tokio VPN handler
- Connect spawns daemon, disconnect kills it

### Other
- Interactive first-run config (prompts for gateway/username/hosts, offers to save)
- README updated with "Why?" section explaining full-tunnel vs split-tunnel
- Desktop shortcuts updated to point to `C:\drivaslab\pmacs-utils\scripts\`
- 70 tests passing, clippy clean

## Key Files Changed

| File | Changes |
|------|---------|
| `src/main.rs` | Added --daemon, --keep-alive, tray command, interactive config |
| `src/tray.rs` | NEW - System tray module |
| `src/state.rs` | Added pid field, is_daemon_running(), kill_daemon() |
| `src/gp/tunnel.rs` | Added session tracking, expiry warnings, aggressive keepalive |
| `Cargo.toml` | Added tray-icon, tao, image dependencies |
| `README.md` | Added "Why?" section, daemon/tray docs |

## Next Steps (from TODO.md)

1. **Better Connect Output** - Progress indicators like "Authenticating... ✓"
2. **Better Error Messages** - "Run as Administrator" instead of raw errors
3. **macOS/Linux testing** - TUN, routing, hosts file
4. **System tray polish** - Double-click to toggle, notifications

## Technical Notes

### Tray Architecture
```
Main Thread (tao event loop):
  └─ TrayIcon with menu
  └─ Receives menu events
  └─ Sends commands via channel

Tokio Thread:
  └─ Receives TrayCommand (Connect/Disconnect/Exit)
  └─ Spawns daemon or kills by PID
  └─ Sends VpnStatus updates back
```

### Daemon PID Flow
1. Parent calls `spawn_daemon()` with `--_daemon-pid=1` marker
2. Child sees marker, runs VPN, saves its own PID to state
3. Parent returns immediately with child PID
4. `disconnect` reads state, calls `taskkill /PID X /F` (Windows)

### Session Tracking
- `session_start: Instant` in SslTunnel
- Check every 5 minutes via tokio interval
- Warn every 15 min after 15hr mark
- Auto-disconnect at 16hr (server would anyway)
