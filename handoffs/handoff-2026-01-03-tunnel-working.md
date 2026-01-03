# Session Handoff — 2026-01-03T05:35

## Summary

Native GlobalProtect VPN tunnel **successfully established**. Auth flow fully working with DUO MFA. Tunnel is up and passing keepalives. SSH not yet working due to missing DNS/routing configuration.

## Completed

### Auth Flow (All Fixed)
- [x] Prelogin → Challenge → MFA → JNLP parsing
- [x] Required params: `jnlpReady=jnlpReady`, `ok=Login`, `direct=yes`
- [x] Positional JNLP format parsing (PMACS doesn't use labeled format)
- [x] Getconfig with full params (user, portal, domain, enc-algo, etc.)
- [x] Tunnel request with username in URL

### Infrastructure Fixes
- [x] DNS resolver: replaced trust-dns with std::net (runtime conflict)
- [x] Windows state dir: USERPROFILE fallback (HOME not set on Windows)
- [x] Documentation updated in `docs/auth-flow-investigation.md`

### What's Working
```
START_TUNNEL received
TUN device: wintun
Internal IP: 10.156.56.27
Keepalives: sending/receiving OK
TUN reads: packets flowing
```

## Not Working

### SSH to prometheus
```powershell
ssh prometheus.pmacs.upenn.edu  # fails - DNS can't resolve
ssh 128.91.231.101              # might work if we had the IP
```

**Root cause:** DNS resolution uses system DNS, not VPN DNS (128.91.22.200)

## Next Steps (Priority Order)

### 1. Add Route Without DNS (Quick Test)
Hardcode prometheus IP to verify tunnel actually works:
```rust
// In main.rs, temporarily add direct IP route
router.add_ip_route("128.91.231.101")?;  // or whatever prometheus IP is
```

Or manually in Windows:
```powershell
route add 128.91.231.101 mask 255.255.255.255 10.156.56.27
```

### 2. Configure VPN DNS
The tunnel config provides:
- DNS servers: 128.91.22.200, 172.16.50.10
- DNS suffixes: pmacs.upenn.edu, uphs.upenn.edu

Options:
a. Set Windows adapter DNS to VPN servers
b. Implement DNS relay/proxy
c. Use VPN DNS only for specific domains (split DNS)

### 3. Proper Route Management
Current issue: routes require DNS resolution which fails.
Fix: Either resolve using VPN DNS, or accept IP addresses directly.

### 4. Test Packet Flow
The tunnel shows TUN reads but we need to verify:
- Packets are properly framed (GP packet format)
- Gateway is accepting/responding to data packets
- Routing is correct for VPN traffic

## Files Modified This Session

| File | Change |
|------|--------|
| `src/gp/auth.rs` | +jnlpReady, +direct, +positional JNLP parsing, +getconfig params |
| `src/gp/tunnel.rs` | +username in tunnel request |
| `src/vpn/routing.rs` | std::net DNS instead of trust-dns |
| `src/state.rs` | USERPROFILE fallback for Windows |
| `src/main.rs` | Updated API calls |
| `docs/auth-flow-investigation.md` | Documented all fixes |

## Test Command

```powershell
cargo build && C:/drivaslab/pmacs-utils/target/debug/pmacs-vpn.exe -v connect -u yjk
```

## Key Learnings

1. **GP protocol is picky** - Missing `jnlpReady` = silent failure (empty 200)
2. **JNLP format varies** - Some servers use positional, not labeled
3. **Tunnel request needs user** - Empty user = "Invalid user name"
4. **trust-dns conflicts with Tokio** - Use std::net for sync DNS in async context
5. **Windows lacks HOME** - Use USERPROFILE fallback

## Architecture Reference

```
Auth Flow:
prelogin.esp → login.esp (creds) → login.esp (MFA+inputStr) → getconfig.esp → ssl-tunnel-connect.sslvpn

Tunnel:
TUN device ←→ GP packet framing ←→ TLS stream ←→ Gateway
```

## VPN Config Received

```
IP: 10.156.56.27
DNS: 128.91.22.200, 172.16.50.10
Gateway: 170.212.0.240
Timeout: 7200s (2hr idle)
Lifetime: 57600s (16hr session)
```
