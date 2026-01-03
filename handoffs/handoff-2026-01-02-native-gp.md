# Handoff: Native GlobalProtect Implementation

**From:** Opus (architecture/planning)
**To:** Sonnet (implementation)
**Date:** 2026-01-02

---

## Context

We're pivoting from wrapping OpenConnect to implementing GlobalProtect natively. This gives us:
- Single binary distribution (no OpenConnect dependency)
- Better Windows support (just need wintun.dll)
- Simpler UX for non-technical lab members

## Before You Start

**Read these files in order:**

1. `docs/rust-claude-guide.md` - Rust development practices
2. `docs/native-gp-implementation-plan.md` - Full implementation spec
3. `docs/pmacs-environment.md` - PMACS network details
4. `CLAUDE.md` - Project conventions

## Current State

- Rust project compiles on Mac/Windows/Linux
- 41 unit tests passing
- Platform routing managers work (mac.rs, linux.rs, windows.rs)
- Hosts file management works
- State persistence works
- CLI scaffold exists with `connect`, `disconnect`, `status`, `init` commands

**The `connect` and `disconnect` commands are stubs** - that's what you're implementing.

## What to Implement

Create a new `src/gp/` module with:

| File | Purpose | Priority |
|------|---------|----------|
| `mod.rs` | Module exports | P1 |
| `auth.rs` | prelogin, login, getconfig | P1 |
| `packet.rs` | SSL tunnel packet framing | P2 |
| `tun.rs` | TUN device wrapper (tun-rs) | P2 |
| `tunnel.rs` | SSL tunnel + packet I/O | P3 |

Then update `main.rs` to wire it together.

## Dependencies to Add

```toml
# In Cargo.toml [dependencies]
reqwest = { version = "0.12", features = ["rustls-tls", "cookies"] }
quick-xml = { version = "0.37", features = ["serialize"] }
tun-rs = { version = "1.4", features = ["async"] }
tokio-rustls = "0.26"
rustls = "0.23"
webpki-roots = "0.26"
rpassword = "7"
```

## Implementation Order

Follow the phases in `docs/native-gp-implementation-plan.md`:

1. **Auth first** - Can test against real server immediately
2. **Packet framing** - Can unit test without network
3. **TUN wrapper** - Platform abstraction
4. **SSL tunnel** - Ties it together
5. **CLI integration** - Wire into main.rs

## Key Technical Details

### Auth Flow

```
prelogin → login (with "push" for DUO) → getconfig
```

The server handles DUO - we just send `passcode=push` and it triggers the push, then waits for approval before responding.

### SSL Tunnel Packet Format

```
[magic:4][ethertype:2][len:2][type:8][payload:N]
```

- Magic: `0x1a2b3c4d`
- Ethertype: `0x0800` (IPv4) or `0x86dd` (IPv6)
- Len: payload size, big-endian (0 = keepalive)
- Type: 8 zero bytes for data packets

### Gateway

- URL: `psomvpn.uphs.upenn.edu`
- Protocol: GlobalProtect (`--protocol=gp` in OpenConnect terms)
- Endpoints: `/ssl-vpn/prelogin.esp`, `/ssl-vpn/login.esp`, `/ssl-vpn/getconfig.esp`, `/ssl-tunnel-connect.sslvpn`

## Testing

You won't have VPN credentials, so:

1. **Unit test** everything you can (XML parsing, packet framing)
2. **Mock** HTTP responses for auth flow tests
3. **Skip** tests that require actual TUN device (needs root)
4. Leave integration testing for Opus review

## Code Style

- Use `thiserror` for error types (already in project)
- Use `tracing` for logging (already in project)
- Follow existing patterns in `src/config.rs`, `src/state.rs`
- Run `cargo clippy -- -D warnings` before committing

## What NOT to Do

- Don't modify platform/*.rs (routing) - it works
- Don't modify vpn/hosts.rs - it works
- Don't implement ESP/IPsec tunnel - SSL tunnel is sufficient
- Don't implement SAML auth - password + DUO is what PMACS uses

## Definition of Done

- [ ] `cargo build` succeeds on Windows
- [ ] `cargo test` passes (new tests + existing 41)
- [ ] `cargo clippy -- -D warnings` clean
- [ ] Auth module can parse real prelogin/login/getconfig responses
- [ ] Packet framing is unit tested
- [ ] TUN wrapper compiles on all platforms
- [ ] CLI shows helpful messages during connect flow

## Questions?

If you hit ambiguity:
1. Check the plan doc first
2. Check OpenConnect's protocol doc: https://github.com/dlenski/openconnect/blob/master/PAN_GlobalProtect_protocol_doc.md
3. Leave a TODO comment and move on

---

**When done:** Commit your work, then Opus will review and do integration testing with real VPN credentials.
