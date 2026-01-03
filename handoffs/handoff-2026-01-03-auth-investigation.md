# Session Handoff — 2026-01-03T05:00

## Summary

Tested native GlobalProtect implementation against real VPN (psomvpn.uphs.upenn.edu). Auth flow works through DUO push approval, but server returns **empty 200 response** instead of expected JNLP with auth-cookie.

## Completed

- Fixed two-step MFA challenge flow (password → challenge → passcode)
- Added correct protocol parameters (`ok=Login`, `prot`, `server`, etc.)
- Enabled cookie store for session persistence
- DUO push successfully received and approved
- Documented all findings in `docs/auth-flow-investigation.md`

## The Problem

After successful DUO approval:
```
MFA response status: 200 OK
content-type: application/xml; charset=UTF-8
content-length: 0  ← EMPTY BODY
```

Expected: JNLP XML with auth-cookie. Got: empty body.

Retry login after empty response → gets NEW challenge (session not preserved).

## What We Know

1. Server uses HTML/JS challenge format (not XML):
   ```javascript
   var respStatus = "Challenge";
   var respMsg = "Enter passcode:";
   thisForm.inputStr.value = "691e86260039...";
   ```

2. MFA request with `inputStr` + `passwd=push` triggers DUO successfully

3. Server doesn't set any `Set-Cookie` headers

4. No session state preserved between requests

## Attempted Fixes (All Failed)

1. `cookie_store(true)` - server sets no cookies
2. Retry login after empty response - gets new challenge
3. `ok=Login` parameter - required but no effect
4. Full protocol params (prot, server, clientos, etc.) - no effect

## Next Steps (Recommended Order)

### 1. Packet Capture (BEST OPTION)
Install Wireshark, connect with official GP client, capture traffic:
- See exact endpoint sequence
- See all parameters/headers
- See what response GP client gets after DUO

### 2. Try Portal Endpoints
Maybe this VPN requires portal auth first:
- `/global-protect/prelogin.esp` → `/global-protect/getconfig.esp`
- Get portal-userauthcookie, use for gateway

### 3. Check if RADIUS Async
The empty 200 might be RADIUS saying "push sent, poll for result"
- Try polling the same or different endpoint

## Files Modified

- `src/gp/auth.rs` - MFA challenge handling, params
- `docs/auth-flow-investigation.md` - detailed findings

## Key Code Location

Challenge parsing: `src/gp/auth.rs:195` (`parse_challenge` function)
MFA handling: `src/gp/auth.rs:313-410` (the if-block after login)

## Quick Test Command

```bash
cargo build && C:/drivaslab/pmacs-utils/target/debug/pmacs-vpn.exe -v connect -u yjk
```

## Resources

- [GP Protocol Doc](https://github.com/dlenski/openconnect/blob/master/PAN_GlobalProtect_protocol_doc.md)
- [OpenConnect auth-globalprotect.c](https://github.com/openconnect/openconnect/blob/master/auth-globalprotect.c)
- [DUO GP Integration](https://duo.com/docs/paloalto)
