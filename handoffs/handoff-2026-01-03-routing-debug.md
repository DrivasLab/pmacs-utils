# Session Handoff — 2026-01-03T07:10

## Summary

VPN tunnel fully established and working (keepalives flow). DNS resolution via VPN DNS servers fails - packets not reaching DNS servers through tunnel.

## What Works

- **Full auth flow**: prelogin → login → MFA (DUO push) → getconfig → tunnel
- **TUN device**: wintun created, IP assigned (10.156.56.x)
- **SSL tunnel**: START_TUNNEL received, bidirectional keepalives working
- **Route addition**: Routes added with interface index (`route add ... IF 86`)

```
TUN device created: wintun
SSL tunnel established
Starting tunnel event loop
Sending keepalive
Read 76 bytes from TUN       <- TUN receiving packets
Received keepalive           <- Gateway responding
```

## What Doesn't Work

### DNS Resolution via VPN DNS
```
Route added: 128.91.22.200 -> 10.156.56.32 IF 86
Resolving prometheus.pmacs.upenn.edu via VPN DNS
DNS query to 128.91.22.200 failed: send failed: host unreachable (os error 10065)
```

The UDP DNS query packet isn't reaching the VPN DNS server through the tunnel.

## Key Observations

1. **Tunnel works for its own traffic** - keepalives flow bidirectionally
2. **Routes are added correctly** - `route add ... mask ... gateway IF <wintun_index>`
3. **UDP packets don't flow** - "host unreachable" suggests routing issue

## Hypothesis

The Windows route command might not be binding UDP sockets correctly to the TUN interface. When we do:
```rust
let socket = UdpSocket::bind("0.0.0.0:0")?;
socket.send_to(query, "128.91.22.200:53")?;
```

Windows may be:
1. Not using the TUN interface despite the route
2. The route metric/priority isn't high enough
3. Something about wintun's routing table integration

## Files Changed This Session

| File | Change |
|------|--------|
| `src/vpn/routing.rs` | Added VPN DNS resolution, interface-aware routing |
| `src/platform/windows.rs` | Added interface index lookup, `IF <index>` in route command |
| `src/platform/mod.rs` | Added `get_routing_manager_for_interface()` |
| `src/main.rs` | Start tunnel before adding routes, use interface-aware router |

## Next Steps to Investigate

### 1. Verify Route in Windows
While VPN is connected, check:
```powershell
route print | findstr "128.91"
Get-NetRoute -DestinationPrefix "128.91.22.200/32"
```

### 2. Check if TUN Receives DNS Packets
Add debug logging to see if DNS UDP packets arrive at TUN:
```rust
// In tunnel.rs, log all TUN reads
debug!("TUN packet: {:02x?}", &tun_buf[..n.min(64)]);
```

### 3. Try Binding UDP to TUN IP
Instead of binding to 0.0.0.0, try binding to the TUN IP:
```rust
let socket = UdpSocket::bind((tunnel_ip, 0))?;
```

### 4. Check Windows Firewall
Windows Firewall might block wintun traffic:
```powershell
Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*wintun*"}
```

### 5. Compare with Working VPN Clients
Use Wireshark to capture what the official GlobalProtect client does for DNS.

### 6. Alternative: Use ESP/IPsec Instead of SSL
The getconfig response includes IPsec keys. The official client might use ESP mode instead of SSL tunnel mode.

## Test Command

```powershell
# Run as Administrator
.\target\release\pmacs-vpn.exe -v connect -u yjk
```

## VPN Config Reference

```xml
<ip-address>10.156.56.32</ip-address>
<dns>
    <member>128.91.22.200</member>
    <member>172.16.50.10</member>
</dns>
<access-routes>
    <member>0.0.0.0/0</member>
    <member>128.91.22.200/32</member>
    <member>172.16.50.10/32</member>
</access-routes>
```

## Architecture

```
Current packet flow (broken):
  App → UDP socket → Windows routing → TUN? → SSL tunnel → Gateway

Expected:
  App → UDP socket → route matches 128.91.22.200 IF 86 →
  TUN device → SSL tunnel → Gateway → VPN network → DNS server

Working (keepalives):
  Tunnel code → TUN device → SSL tunnel → Gateway (and back)
```

## Code Locations

- DNS resolution: `src/vpn/routing.rs:163-220` (`build_dns_query`, `query_dns_server`)
- Route addition: `src/platform/windows.rs:42-86`
- Tunnel loop: `src/gp/tunnel.rs:156-213`
- Main flow: `src/main.rs:178-284`
