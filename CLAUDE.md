# PMACS Utils

Toolkit for streamlined PMACS cluster access via split-tunnel VPN.

## Design Principles

1. **UX first** - "Just works" for non-technical users
2. **Cross-platform** - Windows, Mac, Linux from single codebase
3. **Zero dependencies** - Single binary (+ wintun.dll on Windows)
4. **Lightweight for users** - Our code can be robust; user RAM is precious
5. **Best practices** - Error handling, clear messages, defensive coding

## Technical Approach

**Native GlobalProtect implementation** - no OpenConnect dependency.

- Direct GlobalProtect protocol (SSL tunnel mode)
- Cross-platform TUN device via `tun` crate (0.6)
- Split-tunnel routing for specified hosts only
- DUO MFA support (server-side RADIUS, we just send "push")
- TLS via rustls (ring crypto backend, no cmake required)

**Language:** Rust (single binary, cross-compiles)

## Knowledge Base

| Doc | Purpose |
|-----|---------|
| [docs/native-gp-implementation-plan.md](docs/native-gp-implementation-plan.md) | **Implementation spec** |
| [docs/pmacs-environment.md](docs/pmacs-environment.md) | PMACS hosts, VPN details |
| [docs/rust-claude-guide.md](docs/rust-claude-guide.md) | Rust dev practices |
| [docs/vpn-slice-analysis.md](docs/vpn-slice-analysis.md) | Routing/hosts approach |

## Target UX

```bash
# Connect (prompts for password, sends DUO push)
pmacs-vpn connect -u USERNAME

# Check status
pmacs-vpn status

# Disconnect
pmacs-vpn disconnect
```

## Current Status

**Implementation Complete - Ready for Testing**

- [x] Rust project scaffold with CLI
- [x] Platform routing managers (mac/linux/windows)
- [x] Hosts file management
- [x] State persistence
- [x] Native GlobalProtect auth module
- [x] SSL tunnel implementation
- [x] TUN device integration
- [x] CLI wiring (connect/disconnect)
- [x] 54 unit tests passing
- [x] Clippy clean (no warnings)
- [ ] **Integration testing with real VPN** ← NEXT

## Architecture

```
src/
├── main.rs          # CLI
├── config.rs        # TOML config
├── state.rs         # Connection state persistence
├── gp/              # GlobalProtect protocol (NEW)
│   ├── auth.rs      # prelogin → login → getconfig
│   ├── tunnel.rs    # SSL tunnel
│   ├── tun.rs       # TUN device wrapper
│   └── packet.rs    # Packet framing
├── platform/        # OS-specific routing
│   ├── mac.rs
│   ├── linux.rs
│   └── windows.rs
└── vpn/
    ├── routing.rs   # DNS + route management
    └── hosts.rs     # /etc/hosts management
```

## Development

```bash
# Build
cargo build

# Test
cargo test

# Lint
cargo clippy -- -D warnings

# Build release
cargo build --release
```

## Gateway Details

- **URL:** `psomvpn.uphs.upenn.edu`
- **Protocol:** GlobalProtect (SSL tunnel mode)
- **Auth:** Password + DUO push
- **Target hosts:** `prometheus.pmacs.upenn.edu` (and others in config)

## Windows: wintun.dll

The binary embeds `wintun.dll` (~420KB) from [wintun.net](https://www.wintun.net/). On first TUN device creation, it auto-extracts to the executable's directory. No manual installation needed.

## Testing (requires admin/root)

```bash
cargo build --release
sudo ./target/release/pmacs-vpn connect -u YOUR_USERNAME
# Enter password, approve DUO push on phone
# In another terminal: ssh prometheus.pmacs.upenn.edu
# Ctrl+C to disconnect
```
