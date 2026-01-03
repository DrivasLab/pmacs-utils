# Session Handoff — Mac Development

Created: 2026-01-02 from Windows session

## Context

This repo was reset for a fresh start. The previous Docker-based approach was scrapped in favor of native OpenConnect + vpn-slice for lower resource usage.

## First Steps on Mac

### 1. Wipe and sync with remote

```bash
git clone git@github.com:DrivasLab/pmacs-utils.git
cd pmacs-utils
```

Or if already cloned:
```bash
git fetch origin
git reset --hard origin/master
```

### 2. Test manual VPN connection

Before writing any scripts, verify the manual flow works:

```bash
# Install
brew install openconnect
sudo pip3 install vpn-slice

# Test vpn-slice
sudo vpn-slice --self-test

# Connect
sudo openconnect psomvpn.uphs.upenn.edu --protocol=gp -u YOUR_USER \
  -s 'vpn-slice prometheus.pmacs.upenn.edu'
```

When prompted:
- Enter PMACS password
- Enter "push" for passcode (triggers DUO)
- Approve DUO on phone quickly (timeout is short)

### 3. Test SSH (in another terminal)

```bash
ssh prometheus.pmacs.upenn.edu -l YOUR_USER
```

You'll get another DUO push for SSH.

### 4. Disconnect

Ctrl+C in the openconnect terminal. vpn-slice cleans up routes automatically.

## Implementation Plan

### `scripts/setup.sh`

One-time setup wizard:

1. Check for Homebrew, prompt to install if missing
2. `brew install openconnect`
3. `sudo pip3 install vpn-slice`
4. Prompt for PMACS username
5. Save to `~/.config/pmacs/config`
6. Check for SSH key (`~/.ssh/id_ed25519`), generate if missing
7. Configure `~/.ssh/config` with prometheus host
8. Optionally: start VPN, SSH in with password, upload public key

### `scripts/connect.sh`

Daily connection script:

1. Load username from `~/.config/pmacs/config`
2. Prompt for password (or retrieve from Keychain if implemented)
3. Run openconnect with vpn-slice
4. User approves DUO
5. Script exits when VPN is connected (openconnect stays running)

Tricky part: openconnect runs in foreground. Options:
- Run in background with `&` and log to file
- Keep terminal open (simpler, user sees status)
- Use `--background` flag if available

### `scripts/disconnect.sh`

Stop VPN:

1. Find openconnect process: `pgrep openconnect`
2. Send SIGTERM: `sudo kill <pid>`
3. vpn-slice handles cleanup automatically

## SSH Key Flow

1. VPN must be running first
2. Generate: `ssh-keygen -t ed25519 -C "user@pmacs"`
3. First SSH uses password (prompts for password + DUO)
4. Append key: `ssh-copy-id prometheus` or manual append
5. Future logins: key + DUO (no password)

Note: DUO is required even with SSH keys.

## Open Questions to Resolve

- **What PMACS subnets need routing?**
  - Start with just `prometheus.pmacs.upenn.edu`
  - May need to add IP ranges (10.x.x.x/16 or similar) if other hosts are needed

- **Password storage?**
  - Start simple: prompt each time
  - Later: macOS Keychain integration if desired

- **Background vs foreground?**
  - Test both approaches, see what feels better

## Files to Implement

| File | Status | Notes |
|------|--------|-------|
| `scripts/setup.sh` | Placeholder | Implement first |
| `scripts/connect.sh` | Placeholder | After setup works |
| `scripts/disconnect.sh` | Placeholder | Simple, do last |
| `docs/MAC.md` | Has manual steps | Update after scripts work |

## When Done

After Mac implementation is working:
1. Commit and push
2. Return to Windows session
3. Implement Windows support (WSL-based or Docker fallback)

## Git Notes

Force pushing is expected during development:
```bash
git add -A && git commit -m "message"
git push --force origin master
```

Once stable, we'll protect master and use normal PRs.
