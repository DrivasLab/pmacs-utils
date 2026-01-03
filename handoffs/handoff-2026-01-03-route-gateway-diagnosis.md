# Session Handoff — 2026-01-03T07:55

## Summary

Re-evaluated the routing failure. While the socket binding issue (previous diagnosis) is valid, a more critical flaw exists in the Windows routing implementation that likely renders the route ineffective even if the socket binds correctly.

## Diagnosis: Incorrect Route Gateway for On-Link Routes

### The Contradiction
In `src/platform/windows.rs`, the code attempts to create an "on-link" route (a route that sends traffic directly out of an interface without a next-hop gateway).

**The Comment says:**
```rust
// Use on-link routing (0.0.0.0 gateway) with interface index
```

**The Code does:**
```rust
Command::new("route")
    .args([
        "add",
        destination,
        "mask",
        "255.255.255.255",
        gateway, // <--- ERROR: Uses '10.156.56.36' instead of '0.0.0.0'
        // ...
        "if",
        &if_index.to_string(),
    ])
```

### Why this fails
When `gateway` (which is `10.156.56.36`, the TUN IP) is passed to `route add` alongside `IF <index>`, Windows interprets this as a route *through* `10.156.56.36` as the next hop.

Since the interface is a point-to-point /32 link:
1.  Windows might accept the route (hence `Route added successfully`).
2.  But when a packet is sent, the routing table says "Send to gateway 10.156.56.36".
3.  The OS looks up how to reach `10.156.56.36`. It finds it's the local address.
4.  Traffic loops back or is dropped because "Gateway" implies a *remote* router on the link, not the local interface itself.

For a VPN tunnel interface (especially `wintun` which is L3), the standard way to inject a route is to specify `0.0.0.0` as the gateway. This tells the OS "the destination is directly on this link; just shove the packet into the interface".

### Supporting Evidence
- Previous logs: `Adding route 128.91.22.200 via interface 86 (on-link)`
- Yet the command used `10.156.56.36`.
- Error `10065` (Unreachable Host) is consistent with a route that points to an invalid or unreachable gateway logic, even if the route entry exists.

## Proposed Fixes (Revised)

### 1. Fix Route Gateway (Primary Fix)
Modify `src/platform/windows.rs` to force `0.0.0.0` as the gateway when an interface index is available.

```rust
// src/platform/windows.rs
let gateway_arg = if self.interface_index.is_some() {
    "0.0.0.0"
} else {
    gateway
};
// Use gateway_arg in Command::new("route")...
```

### 2. Explicit Socket Binding (Secondary Fix)
Retain the previous fix (binding to the TUN IP). While `0.0.0.0` binding *might* work if the route is correct (OS source address selection *should* pick the interface IP), explicitly binding to the TUN IP is safer and prevents ambiguity on multi-homed systems.

## Verification Plan

1.  **Apply Fix 1 (Route Gateway):** Change `src/platform/windows.rs` to use `0.0.0.0`.
2.  **Apply Fix 2 (Socket Binding):** Change `src/vpn/routing.rs` to bind to TUN IP.
3.  **Test:** Connect and verify DNS resolution.
