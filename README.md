<p align="center">
  <h1 align="center">openvpn-sso</h1>
  <p align="center">
    OpenVPN CLI client for macOS with SAML/SSO authentication
    <br />
    <em>Split-DNS &bull; Multi-tunnel &bull; Background mode &bull; Zero dependencies beyond openvpn</em>
  </p>
</p>

---

## The Problem

You need a **fast, scriptable** OpenVPN client on macOS that supports:

- **SAML/SSO authentication** (browser-based login, MFA)
- **Split-DNS** (only VPN domains go through VPN DNS)
- **Multiple simultaneous tunnels**

But your options are:

| Client | SAML | Split-DNS | Multi-tunnel | Fast | CLI |
|--------|------|-----------|-------------|------|-----|
| OpenVPN Connect | Yes | No | No | Yes | No |
| Viscosity | Yes | Yes | Yes | **Slow** | No |
| `openvpn` CLI | No | No | No | Yes | Yes |
| **openvpn-sso** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** |

## Quick Start

```bash
# Install openvpn
brew install openvpn
echo 'export PATH="/opt/homebrew/opt/openvpn/sbin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Get openvpn-sso
git clone https://github.com/talf/openvpn-sso.git
cd openvpn-sso

# One-time setup (passwordless sudo + symlink to /usr/local/bin)
./openvpn-sso -i

# Copy your configs
cp ~/Downloads/*.ovpn ~/.openvpn-sso/configs/

# Connect
openvpn-sso -c my-vpn
```

Your browser opens for SAML login. After authentication, the tunnel connects in the background.

## Usage

```bash
openvpn-sso -c client              # connect (opens browser for SAML)
openvpn-sso -c client --fg         # connect in foreground
openvpn-sso -l                     # list all configs and status
openvpn-sso -d client              # disconnect one tunnel
openvpn-sso -d                     # disconnect all
openvpn-sso --log client           # show full log
openvpn-sso --log client -f        # follow log (tail -f)
openvpn-sso --log client --tail 100  # last 100 lines
```

All standard `openvpn` flags pass through:

```bash
openvpn-sso -c client --remote 1.2.3.4 1194    # override server
openvpn-sso -c client --proto tcp               # use TCP
openvpn-sso -c client --verb 5                  # verbose logging
```

## How It Works

```
┌─────────────────────────────────────────────────────┐
│ openvpn-sso                                         │
│                                                     │
│  User Process (your shell)                          │
│  ├─ Opens browser for SAML auth (as your user)      │
│  ├─ Polls for connection status                     │
│  └─ Returns to shell when connected                 │
│                                                     │
│  Root Process (via sudo)                            │
│  ├─ Runs openvpn with IV_SSO=webauth               │
│  ├─ Creates tun interface + routes                  │
│  └─ Configures split-DNS via scutil                 │
│                                                     │
│  Split-DNS (macOS scutil)                           │
│  ├─ VPN domains → VPN DNS server                    │
│  └─ Everything else → normal DNS                    │
└─────────────────────────────────────────────────────┘
```

1. Cleans server-only directives (`push`, `cipher`) from the `.ovpn` config
2. Starts `openvpn` with `IV_SSO=webauth` for SAML authentication
3. Detects the SAML URL from openvpn output and opens your default browser
4. After authentication, configures macOS split-DNS using `SupplementalMatchDomains`
5. Cleans up DNS and kills processes on disconnect

## Multi-Tunnel

Each `.ovpn` config creates an independent tunnel with its own:
- PID tracking
- Log file
- DNS namespace

```bash
openvpn-sso -c us-east
openvpn-sso -c eu-west
openvpn-sso -l
```
```
[*] us-east
    Remote:    vpn-us.example.com:1194
    State:     CONNECTED
    PID:       12345
    Interface: utun5
    VPN IP:    10.8.1.50
    DNS:       10.8.0.1
    Since:     10:32:15

[*] eu-west
    Remote:    vpn-eu.example.com:1194
    State:     CONNECTED
    PID:       12400
    Interface: utun6
    VPN IP:    10.9.1.50
    DNS:       10.9.0.1
    Since:     10:33:01
```

Disconnected or dead tunnels are automatically cleaned up on the next connect.

## Config Lookup

Configs are resolved in order:

1. **Exact path**: `openvpn-sso -c ./profile.ovpn`
2. **Current dir**: `openvpn-sso -c client` → `./client.ovpn`
3. **Config dir**: `openvpn-sso -c client` → `~/.openvpn-sso/configs/client.ovpn`

## Commands

| Short | Long | Description |
|-------|------|-------------|
| `-c` | `--config`, `--connect` | Connect to a VPN |
| `-l` | `--list`, `--status` | Show all configs and connection status |
| `-d` | `--disconnect`, `--cleanup` | Disconnect tunnel(s) |
| `-i` | `--install` | One-time setup (sudo + symlink) |
| `-h` | `--help` | Help |
| `-v` | `--version` | Version |
| | `--log [--tail N] [-f]` | View or follow tunnel logs |
| | `--fg` | Run in foreground |
| | `--uninstall` | Remove sudo config + symlink |

## Performance

Applied automatically:

| Optimization | Default | openvpn-sso | Impact |
|-------------|---------|-------------|--------|
| Send buffer | 9 KB | 512 KB | Major throughput improvement |
| Cipher | AES-256-CBC | AES-256-GCM / ChaCha20 | Hardware-accelerated on Apple Silicon |
| DNS | Full tunnel DNS | Split-DNS (SupplementalMatchDomains) | Only VPN domains use VPN DNS |

## Files

```
~/.openvpn-sso/
├── configs/          .ovpn connection profiles
└── sessions/         runtime state (PIDs, logs, cleaned configs)
```

## Requirements

- **macOS** (uses `scutil` for DNS, `open` for browser)
- **openvpn 2.6+** (`brew install openvpn`)
- **OpenVPN server with SAML/SSO** (`IV_SSO=webauth`)
- **Python 3** (ships with macOS, used for CIDR calculations)

## License

MIT
