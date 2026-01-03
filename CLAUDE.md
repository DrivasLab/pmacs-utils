# PMACS Utils

Native VPN bypass toolkit for PMACS cluster access.

## Project Goal

Replace PMACS's full-tunnel GlobalProtect VPN with a lightweight split-tunnel approach using OpenConnect + vpn-slice. Only PMACS traffic goes through VPN; everything else stays on normal internet.

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
