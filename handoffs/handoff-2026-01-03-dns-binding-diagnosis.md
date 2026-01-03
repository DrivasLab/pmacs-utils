# Session Handoff — 2026-01-03T07:45

## Summary

Diagnosed the "Socket operation on unreachable host" (os error 10065) issue preventing DNS resolution through the VPN tunnel. The root cause is improper socket binding in the DNS query logic.

## Diagnosis

### Symptoms
- Tunnel is established and passing traffic (keepalives working).
- Routes to VPN DNS servers (`128.91.22.200`, `172.16.50.10`) are correctly added to the routing table via the `wintun` interface.
- DNS queries fail immediately with `os error 10065` (WSAEHOSTUNREACH).

### Root Cause
The function `query_dns_server` in `src/vpn/routing.rs` binds the UDP socket to `0.0.0.0:0`:

```rust
// src/vpn/routing.rs:165
let socket = UdpSocket::bind("0.0.0.0:0").map_err(|e| format!("bind failed: {}", e))?;
```

On Windows, especially with point-to-point interfaces like `wintun`, the OS requires packets destined for the tunnel to be sourced from the interface's assigned IP address (`10.156.56.36` in the log). When binding to `0.0.0.0`, the OS might select the LAN IP or fail to associate the socket with the correct interface for the route, causing the "unreachable host" error despite the route existing.

### Evidence
- Log shows proper route addition: `Route added successfully: 128.91.22.200 -> 10.156.56.36` via interface 86.
- Log shows failure immediately after: `DNS query to 128.91.22.200 failed: send failed: ... (os error 10065)`
- `VpnRouter` holds the gateway/interface IP in `self.gateway` (initialized from `config.internal_ip`), but this is not used for the DNS query socket binding.

## Proposed Fix

Modify `src/vpn/routing.rs` to explicitly bind the DNS query socket to the TUN interface's IP address.

1.  **Update `query_dns_server` signature**:
    Accept an optional source IP address.
    ```rust
    fn query_dns_server(query: &[u8], server: SocketAddr, bind_addr: Option<IpAddr>) -> Result<Ipv4Addr, String>
    ```

2.  **Update socket binding logic**:
    ```rust
    let bind_ip = bind_addr.unwrap_or_else(|| "0.0.0.0".parse().unwrap());
    let socket = UdpSocket::bind(SocketAddr::new(bind_ip, 0))...
    ```

3.  **Update call site in `resolve_with_dns`**:
    Pass `self.gateway` (parsed as `IpAddr`) to `query_dns_server`.
    ```rust
    // In resolve_with_dns
    let source_ip: Option<IpAddr> = self.gateway.parse().ok();
    // ...
    match query_dns_server(&query, server_addr, source_ip) { ... }
    ```

## Verification Plan

1.  Apply the code changes.
2.  Run the connection command: `pmacs-vpn connect -u yjk`
3.  Observe logs. Successful DNS resolution will show:
    `INFO VPN DNS resolved prometheus.pmacs.upenn.edu -> <IP>`
4.  Verify final connectivity with `ssh prometheus.pmacs.upenn.edu`.
