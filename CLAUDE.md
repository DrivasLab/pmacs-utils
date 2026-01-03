# PMACS Utils

Native VPN bypass toolkit for PMACS cluster access.

## Project Goal

Replace PMACS's full-tunnel GlobalProtect VPN with a lightweight split-tunnel approach using OpenConnect + vpn-slice. Only PMACS traffic goes through VPN; everything else stays on normal internet.

## Why This Approach

We evaluated several options:

| Approach | Overhead | Maintenance | Verdict |
|----------|----------|-------------|---------|
| Docker + wazum/openconnect-proxy | ~2GB (Docker Desktop) | Low | Too heavy for 8GB Macs |
| Hyper-V Ubuntu VM | ~4GB | High | Way too heavy |
| Native OpenConnect + vpn-slice | Near zero | Medium | Chosen approach |

Docker Desktop requires WSL2 on Windows and uses significant RAM. Lab members have 8-18GB Macs and do heavy work on the cluster anyway, so we want the lightest possible local footprint.

## How vpn-slice Works

vpn-slice is a Python replacement for OpenConnect's default vpnc-script. Normal VPN behavior routes ALL traffic through the tunnel (full tunnel). vpn-slice does the opposite:

1. Only routes specified hosts/subnets through the VPN
2. Adds entries to `/etc/hosts` for VPN-only hostnames
3. Cleans up routes and hosts entries on disconnect

It's maintained by one of the OpenConnect developers: https://github.com/dlenski/vpn-slice

Install: `sudo pip3 install vpn-slice`
Test: `sudo vpn-slice --self-test`

## Architecture

```
openconnect (VPN client)
    └── vpn-slice (split-tunnel routing)
        └── Routes only PMACS hosts/subnets through VPN
        └── Manages /etc/hosts entries
        └── Cleans up on disconnect
```

## Key Technical Details

### VPN Connection
- Gateway: `psomvpn.uphs.upenn.edu`
- Protocol: GlobalProtect (`--protocol=gp`)
- Auth: Password + DUO push (type "push" when prompted for passcode)
- DUO timeout is quick — users need to approve promptly

### Split Tunneling
vpn-slice replaces OpenConnect's default vpnc-script. Instead of routing all traffic through VPN, it only routes specified hosts/subnets.

Example command:
```bash
sudo openconnect psomvpn.uphs.upenn.edu --protocol=gp -u USERNAME \
  -s 'vpn-slice prometheus.pmacs.upenn.edu'
```

### SSH Access
- Host: `prometheus.pmacs.upenn.edu`
- Requires VPN to be connected
- SSH key + DUO push for each login (even with keys, DUO is required)

## Scripts

| Script | Purpose |
|--------|---------|
| `setup.sh` | Install deps, configure SSH, generate SSH key, test connection |
| `connect.sh` | Start VPN with split tunneling |
| `disconnect.sh` | Stop VPN cleanly |

## Config Storage

User config stored in `~/.config/pmacs/config`:
```
PMACS_USER=username
```

Password is prompted each time (or optionally stored in macOS Keychain).

## Development Notes

- Mac-first development, Windows support via WSL later
- Scripts should work on Linux with minimal changes
- vpn-slice requires Python 3

## Common Issues

- **DUO timeout**: OpenConnect waits for DUO approval; if user is slow, it times out
- **vpn-slice not found**: Needs `sudo pip3 install vpn-slice`
- **Permission denied**: OpenConnect requires sudo for tunnel creation

## Development Workflow

**Before scripting, test the manual flow:**

```bash
# 1. Install dependencies
brew install openconnect
sudo pip3 install vpn-slice

# 2. Test vpn-slice
sudo vpn-slice --self-test

# 3. Connect manually
sudo openconnect psomvpn.uphs.upenn.edu --protocol=gp -u YOUR_USER \
  -s 'vpn-slice prometheus.pmacs.upenn.edu'
# Enter password, then "push" for DUO, approve on phone

# 4. In another terminal, test SSH
ssh prometheus.pmacs.upenn.edu -l YOUR_USER

# 5. Ctrl+C in openconnect terminal to disconnect
```

Once the manual flow works, implement the scripts to automate it.

## Git Workflow

Before starting development, wipe the remote and force push:
```bash
git fetch origin
git reset --hard origin/master
# Make changes
git add -A && git commit -m "message"
git push --force origin master
```

This repo is in active development. Force pushing is expected until stable.

## Platform Roadmap

1. **Mac (now)**: Native OpenConnect + vpn-slice
2. **Windows (later)**: WSL2 with same scripts, or Docker as fallback
3. **Linux**: Should work with Mac scripts, untested
