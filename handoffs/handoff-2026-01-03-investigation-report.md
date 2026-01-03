# Codebase Investigation Report — 2026-01-03

## Overview
A comprehensive review of the `pmacs-vpn` codebase was conducted to identify stability, security, and performance issues. The findings are categorized by severity and confidence.

## 1. Critical Issues (High Confidence)

### 1.1 DNS Socket Binding Error (Root Cause of Current Failure)
*   **Location:** `src/vpn/routing.rs:query_dns_server`
*   **Issue:** The DNS query socket binds to `0.0.0.0:0`. On Windows, the OS fails to route packets from this unbound socket through the point-to-point TUN interface, resulting in `os error 10065` (Unreachable Host).
*   **Impact:** DNS resolution via VPN fails completely.
*   **Fix:** Bind explicitly to the TUN interface's IP address.

### 1.2 Blocking Network I/O in Async Context
*   **Location:** `src/vpn/routing.rs` (`query_dns_server`, `resolve_host`)
*   **Issue:** These functions use synchronous `std::net::UdpSocket` and `std::net::ToSocketAddrs` calls. They are invoked from `main.rs` (an async function).
*   **Impact:**
    *   `resolve_host` performs a system DNS lookup which can block for seconds.
    *   `query_dns_server` has a 5-second timeout.
    *   While blocking, the `tokio` runtime is paused (if single-threaded) or a worker thread is held hostage. This can freeze the CLI UI and potentially starve the tunnel background task if thread pool is exhausted.
*   **Fix:** Use `tokio::net::UdpSocket` and `tokio::net::lookup_host`.

### 1.3 Expensive PowerShell Invocation
*   **Location:** `src/platform/windows.rs:get_interface_index`
*   **Issue:** The code spawns a new PowerShell process (via `Command::new("powershell")`) to find the network adapter index. It attempts this twice (exact match, then wildcard).
*   **Impact:** PowerShell startup is extremely slow (often 1-2 seconds). This adds significant latency to the connection setup phase for every route addition if cached improperly, or at least once during startup.
*   **Fix:** Use `netsh` (faster) or `ipconfig`, or preferably use native Windows APIs (via `ipconfig` parsing or the `windows` crate).

## 2. Moderate Issues (Medium Confidence)

### 2.1 Sensitive Data Logging
*   **Location:** `src/gp/auth.rs`
*   **Issue:** Debug logs print full XML responses: `debug!("Login response: {}", body);`.
*   **Impact:** The login response contains the `auth-cookie` (session token). If a user runs with `-v` and shares logs for debugging, they may compromise their session.
*   **Fix:** Redact sensitive fields (cookies, session IDs) before logging, or truncate the log output.

### 2.2 Potential Panics in CLI Interaction
*   **Location:** `src/main.rs` lines 143, 145
*   **Issue:** `unwrap()` is used on `stdin().read_line()` and `stdout().flush()`.
*   **Impact:** If the application is run in a non-interactive environment (e.g., from a script without piped input), it will panic and crash instead of exiting gracefully.
*   **Fix:** Use `?` operator or `match` to handle I/O errors and print a friendly "non-interactive mode" error.

### 2.3 Partial State on Hosts File Failure
*   **Location:** `src/main.rs` -> `src/vpn/hosts.rs`
*   **Issue:** Hosts file modification requires Administrator privileges. If this fails (e.g. `AccessDenied`), the error is caught, but the VPN connection remains active with routes added but hostname resolution broken.
*   **Impact:** User believes they are connected, but `ssh hostname` fails.
*   **Fix:** If hosts file writing fails, prompt the user or automatically roll back the connection (disconnect).

## 3. Low Severity / Maintenance

### 3.1 Hardcoded DLL Extraction Path
*   **Location:** `src/gp/tun.rs:ensure_wintun_dll`
*   **Issue:** Extracts `wintun.dll` to the executable's directory.
*   **Impact:** If installed in `C:\Program Files\`, this write will fail due to permissions.
*   **Fix:** Extract to `%TEMP%` or `Appdata` if the executable directory is not writable.

### 3.2 Blocking File I/O
*   **Location:** `src/gp/tun.rs`, `src/config.rs`
*   **Issue:** Uses `std::fs` inside async functions.
*   **Impact:** Minor performance hit during startup/shutdown.
*   **Fix:** Use `tokio::fs`.

## Action Plan

1.  **Fix Critical #1 (DNS Binding):** Apply the fix diagnosed in the previous step.
2.  **Fix Critical #3 (PowerShell):** Optimize `get_interface_index` to prioritize `netsh` or cache the result.
3.  **Refactor #2 (Async I/O):** Convert `VpnRouter` to use async networking to prevent runtime blocking.
4.  **Security Hardening:** Redact sensitive data from logs.