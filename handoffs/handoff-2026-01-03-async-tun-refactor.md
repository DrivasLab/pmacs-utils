# Session Handoff — 2026-01-03T08:00

## Summary

Refactored tunnel to use async TUN I/O and fixed multiple issues. DNS now sends through correct interface but times out - likely packet framing issue.

## Completed

### 1. Async TUN Refactor (Critical Fix)
- Upgraded `tun` crate 0.6 → 0.8 with `async` feature
- `TunDevice` now wraps `tun::AsyncDevice` with async `read()`/`write()`
- `SslTunnel::run()` uses proper `tokio::select!` with 3 concurrent futures:
  - TUN read (outbound) - **now processed immediately**
  - Gateway read (inbound)
  - Keepalive timer
- **Before**: Outbound packets waited up to 30s (next keepalive tick)
- **After**: Outbound packets processed immediately

### 2. MTU Fix
- Server returns `<mtu>0</mtu>` which caused "Invalid packet size" errors
- Now defaults to 1400 when server returns 0
- Location: `src/gp/auth.rs:559-565`

### 3. Interface Index Lookup Improvements
- Added `netsh` fallback when PowerShell fails
- Now tries: exact name → wildcard → netsh parsing
- Location: `src/platform/windows.rs:104-183`

### 4. DNS Socket Interface Binding
- Added `IP_UNICAST_IF` setsockopt to bind DNS socket to TUN interface
- Uses interface index (e.g., 86) instead of IP address (which failed with error 10049)
- Location: `src/vpn/routing.rs:353-387`

## Current Status

**Tunnel works:**
- Auth flow ✓
- TUN device created ✓
- SSL tunnel established ✓
- Bidirectional packet flow ✓ (TUN reads happening, packets sent to gateway)

**DNS fails with timeout (10060):**
```
DNS query to 172.16.50.10 failed: recv failed: ... (os error 10060)
```

## Suspicious Observation

Every outbound packet triggers a "keepalive" response:
```
TUN read 277 bytes (outbound)
Received keepalive from gateway  ← This should be actual data!
```

The gateway IS responding, but all responses have `len=0` in the GP header, causing them to be interpreted as keepalives.

## Hypothesis: Packet Framing Issue

The `GpPacket::encode()` might be producing incorrectly formatted packets that the gateway:
1. Receives but can't parse
2. Responds with empty acknowledgments (len=0)

Or the inbound parsing might be wrong - real data is being misread as keepalives.

## Files Modified This Session

| File | Changes |
|------|---------|
| `Cargo.toml` | tun 0.6→0.8 async, added Win32_Networking_WinSock |
| `src/gp/tun.rs` | AsyncDevice wrapper, async read/write |
| `src/gp/tunnel.rs` | Proper tokio::select! with 3 futures |
| `src/gp/auth.rs` | MTU 0 → 1400 default |
| `src/platform/windows.rs` | netsh fallback, public get_interface_index |
| `src/platform/mod.rs` | Export get_interface_index |
| `src/vpn/routing.rs` | IP_UNICAST_IF socket binding, interface_index field |

## Next Steps

### 1. Debug Packet Framing (High Priority)
Add hex dump logging to see exactly what's being sent/received:
```rust
// In tunnel.rs, after send_packet:
debug!("Sent GP frame: {:02x?}", &frame[..frame.len().min(64)]);

// After receiving header:
debug!("Recv GP header: {:02x?}", &net_header);
```

Compare with Wireshark capture of official GlobalProtect client.

### 2. Verify GP Packet Format
Check `src/gp/packet.rs`:
- Is the 16-byte header correct?
- Are the magic bytes right?
- Is the length field in the correct position (bytes 6-7)?

### 3. Consider ESP Mode
The getconfig response includes IPsec keys. Official client might use ESP mode, not SSL tunnel. Check if `ipsec-mode: esp-tunnel` means SSL is wrong.

## Test Command

```powershell
.\target\release\pmacs-vpn.exe -v connect -u yjk
```

## Key Locations

- GP packet encoding: `src/gp/packet.rs`
- Tunnel loop: `src/gp/tunnel.rs:163-249`
- DNS query: `src/vpn/routing.rs:249-351`
- Interface binding: `src/vpn/routing.rs:353-387`
