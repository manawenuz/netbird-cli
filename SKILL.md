---
name: netbird-cli
description: Use the NetBird CLI (`netbird`) like a guru. Covers connection lifecycle, status/peer analysis, SSH access, networks/routes, profiles, expose/forwarding, debug/troubleshooting, environment variables, and real-world workflows for both cloud and self-hosted NetBird.
---

# NetBird CLI Skill

## Overview

NetBird is a WireGuard-based overlay network that connects peers with identity-aware access control. The `netbird` binary is both the **daemon** (runs as a system service) and the **CLI client** used to control it.

Use this skill whenever the user:
- Wants to connect, disconnect, or inspect a NetBird peer
- Needs SSH access to a NetBird peer
- Is troubleshooting NetBird connectivity
- Wants to manage routes/networks, profiles, or exposed ports
- Is deploying NetBird headlessly with setup keys

**Binary name:** `netbird`
**Common paths:**
- macOS (Homebrew): `/opt/homebrew/bin/netbird`
- macOS (Intel Homebrew): `/usr/local/bin/netbird`
- Linux: `/usr/bin/netbird` or `/usr/local/bin/netbird`
- Windows: `C:\Program Files\Netbird\netbird.exe`
**Config file:** `/var/lib/netbird/<profile>.json` (default profile is `default`)
**Daemon socket:** `unix:///var/run/netbird.sock` (default on Linux/macOS)
**Log file:** `/var/log/netbird/client.log` (default)

### WireGuard Interface Names

NetBird creates a WireGuard tunnel interface. Default names by platform:

| Platform | Default Interface | Type | Customization |
|----------|-------------------|------|---------------|
| macOS | `utun100` | Userspace (`wireguard-go`) | `--interface-name` |
| Windows | `wt0` | Userspace/Kernel hybrid | `--interface-name` or `NB_INTERFACE_NAME` |
| Linux | `wt0` or `netbird0` | Kernel (preferred) or Userspace | `--interface-name` |
| FreeBSD | `wt0` | Kernel or Userspace | `--interface-name` |

```bash
# Use a custom interface name
netbird up --interface-name netbird0
```

On macOS you can verify with:

```bash
ifconfig | grep utun100
netbird status --json | jq -r '.netbirdIp, .netbirdIpv6'
netstat -nr | grep "$(netbird status --json | jq -r '.netbirdIp | split("/")[0] | split(".")[0:2] | join(".")')"
```

On Linux with kernel mode:

```bash
ip link show wt0
ip route show table 7120   # NetBird custom routing table
netbird status --json | jq -r '.netbirdIp, .netbirdIpv6'
ip route | grep "$(netbird status --json | jq -r '.netbirdIp | split("/")[0] | split(".")[0:2] | join(".")')"
```

### Direct WireGuard Inspection with `wg`

Even though NetBird manages the tunnel, you can usually inspect it with the standard `wg` tool. Read-only usage is safe; manual changes will be overwritten on the next management sync.

Start with `netbird status --json` because it does not require root. If the user specifically needs kernel/WireGuard-level details, ask them to run or approve the narrow read-only command `sudo wg show <interface>` (for example `sudo wg show utun100`). Do not edit sudoers or install privilege helpers unless the user explicitly asks; if they do, prefer a narrowly scoped rule for `wg show` only.

```bash
# Show all NetBird tunnels
sudo wg show

# Show a specific interface
sudo wg show utun100
sudo wg show wt0
```

Useful fields:
- **interface public key** — your peer's WireGuard public key
- **listening port** — local WireGuard UDP port (default 51820)
- **peer public key** — maps to `publicKey` in `netbird status --json`
- **endpoint** — real UDP endpoint of the peer
- **allowed ips** — per-peer NetBird IP and IPv6
- **latest handshake** — should be < 3 minutes for active peers
- **transfer** — bytes sent/received
- **persistent keepalive** — NetBird defaults to 25 seconds

**Endpoint interpretation:**
- `endpoint: 203.0.113.5:51820` or IPv6 → direct P2P UDP path
- `endpoint: 127.2.x.x:17473` → peer traffic is being handled by NetBird's local userspace proxy/relay layer; common on macOS Userspace interfaces and for idle/relayed peers

To map `wg` peer public keys to NetBird hostnames, cross-reference with:

```bash
netbird status --json | jq '.peers.details[] | {fqdn, netbirdIp, publicKey}'
```

## Configuration Precedence

NetBird evaluates settings from highest to lowest priority:

1. **Environment variables** (`NB_*` or legacy `WT_*`)
2. **CLI flags**
3. **Profile JSON file** (`/var/lib/netbird/<profile>.json`)

Environment variable rules:
- Prefix is always `NB` (or `WT` for backwards compatibility)
- Flag dashes become underscores, letters become uppercase
- `--management-url` → `NB_MANAGEMENT_URL`
- `--setup-key` → `NB_SETUP_KEY`
- `--disable-profiles` → `NB_DISABLE_PROFILES`
- `--log-level` → `NB_LOG_LEVEL`

```bash
export NB_MANAGEMENT_URL="https://api.self-hosted.com:33073"
export NB_LOG_LEVEL=debug
netbird up
```

## Global Flags

These flags work with almost any command:

| Flag | Env Var | Description |
|------|---------|-------------|
| `-m, --management-url` | `NB_MANAGEMENT_URL` | Management service URL (default `https://api.netbird.io:443`) |
| `--admin-url` | `NB_ADMIN_URL` | Admin panel URL (default `https://app.netbird.io:443`) |
| `-k, --setup-key` | `NB_SETUP_KEY` | Setup key for headless registration |
| `--setup-key-file` | `NB_SETUP_KEY_FILE` | Path to a file containing the setup key |
| `-c, --config` | `NB_CONFIG` | Override profile config path (default `/var/lib/netbird/default.json`) |
| `--daemon-addr` | `NB_DAEMON_ADDR` | Daemon socket address (default `unix:///var/run/netbird.sock`) |
| `-n, --hostname` | `NB_HOSTNAME` | Custom hostname for this peer |
| `-l, --log-level` | `NB_LOG_LEVEL` | Log level: `panic`, `fatal`, `error`, `warn`, `info`, `debug`, `trace` |
| `--log-file` | `NB_LOG_FILE` | Log destination(s): path, `console`, or `syslog` |
| `-A, --anonymize` | `NB_ANONYMIZE` | Anonymize IPs/domains in status/logs |
| `--preshared-key` | `NB_PRESHARED_KEY` | WireGuard pre-shared key |
| `-s, --service` | `NB_SERVICE` | System service name (default `netbird`) |

## Core Lifecycle Commands

### Connect and authenticate

```bash
# Interactive SSO login (opens browser)
netbird up

# Self-hosted management
netbird up --management-url https://api.self-hosted.com:33073

# Headless/server with setup key
netbird up --setup-key AAAA-BBB-CCC-DDDDDD

# Combine both
netbird up --setup-key AAAA-BBB-CCC-DDDDDD --management-url https://api.self-hosted.com:33073

# Foreground mode (useful for debugging)
sudo netbird up --foreground-mode --log-file console
```

Important `up` flags:

| Flag | Description |
|------|-------------|
| `--allow-server-ssh` | Enable the built-in SSH server on this peer |
| `--enable-ssh-local-port-forwarding` | Allow `-L` style forwarding (requires `--allow-server-ssh`) |
| `--enable-ssh-remote-port-forwarding` | Allow `-R` style forwarding (requires `--allow-server-ssh`) |
| `--enable-ssh-sftp` | Enable SFTP subsystem (requires `--allow-server-ssh`) |
| `--enable-ssh-root` | Allow root login (requires `--allow-server-ssh`) |
| `--disable-ssh-auth` | Disable JWT auth; rely only on network ACLs |
| `--disable-auto-connect` | Do not auto-connect when the service starts |
| `--interface-name` | WireGuard interface name (default `utun100`) |
| `--wireguard-port` | WireGuard listen port (default `51820`) |
| `--dns-resolver-address` | Fix local DNS resolver address, e.g. `127.0.0.1:5053` |
| `--extra-dns-labels` | Add DNS labels for round-robin/load balancing |
| `--external-ip-map` | Map external IPs to local IPs/interfaces |
| `--enable-rosenpass` | Experimental post-quantum key exchange |
| `--rosenpass-permissive` | Allow Rosenpass peer to accept non-Rosenpass peers |
| `--ssh-jwt-cache-ttl` | Cache JWT tokens for N seconds (default `0`) |

### Authenticate without connecting

```bash
netbird login
netbird login --management-url https://api.self-hosted.com:33073
netbird login --setup-key AAAA-BBB-CCC-DDDDDD
```

### Disconnect

```bash
netbird down
```

### Log out / deregister

```bash
# These are aliases
netbird logout
netbird deregister

# Remove a specific profile
netbird deregister --profile work
```

This removes the peer from the management server and deletes local configuration. Use with caution.

## Service Management

The daemon runs as a system service. Most service commands need root/admin.

```bash
# Install daemon service
sudo netbird service install

# With extra options
sudo netbird service install --log-level debug --config /opt/netbird/config.json
sudo netbird service install --disable-profiles
sudo netbird service install --disable-update-settings
sudo netbird service install --enable-capture   # needed for `netbird debug capture`

# Lifecycle
sudo netbird service start
sudo netbird service stop
sudo netbird service restart
sudo netbird service status
sudo netbird service reconfigure

# Remove service
sudo netbird service uninstall
```

## Status and Observability

```bash
# Basic status
netbird status

# Detailed peer view
netbird status -d
netbird status --detail

# Machine-readable
netbird status --json
netbird status --yaml

# Just my NetBird IPv4
netbird status --ipv4

# Filter detailed output
netbird status -d --filter-by-status connected
netbird status -d --filter-by-ips 100.64.0.100,100.64.0.200
netbird status -d --filter-by-names peer-a,peer-b.netbird.cloud

# Anonymize when sharing
netbird status -A
netbird status -d -A
```

### Interpreting `netbird status`

```
OS: darwin/arm64
Daemon version: 0.72.4
CLI version: 0.72.4
Profile: default
Management: Connected
Signal: Connected
Relays: 1/4 Available
Nameservers: 1/1 Available
FQDN: macbook-pro.netbird.selfhosted
NetBird IP: 100.120.160.223/16
NetBird IPv6: fd85:ccc7:327e:3cf5:24e4:9a0f:2533:4739/64
Interface type: Userspace
Wireguard port: 51820
Quantum resistance: false
Lazy connection: true
SSH Server: Disabled
Networks: -
Peers count: 5/20 Connected
```

- **Management** must be `Connected` or the peer cannot receive policies/routes.
- **Signal** must be `Connected` for peer coordination.
- **Relays** should show at least one available; `0/N` usually means firewall/NAT issues.
- **Nameservers** reflects configured DNS resolvers.
- **Interface type**: `Kernel` (preferred, lower overhead) or `Userspace` (fallback, common on macOS and when kernel module is unavailable).
- **Lazy connection**: when `true`, the agent only connects to peers on demand.
- **Peers count**: `X/Y` means X connected out of Y known peers.
- **Service status**: on some platforms (especially macOS) `netbird service status` may report `Stopped` even though the daemon is running; trust `netbird status` for connectivity.

In detailed mode, look for:
- **Connection type**: `P2P` (direct) vs `Relayed`
- **Direct**: `true` is preferred; `false` may indicate NAT/firewall issues
- **ICE candidate**: `host/host`, `srflx/srflx`, `relay/relay`
- **Last WireGuard handshake**: should be recent (< 3 minutes)
- **Latency**: high latency or packet loss suggests relay or routing problems

## SSH Access

NetBird can run a built-in identity-aware SSH server on every peer, accessible via `netbird ssh` or standard OpenSSH.

### Enable SSH server on a peer

```bash
netbird down
netbird up --allow-server-ssh

# With all optional features
netbird up --allow-server-ssh \
  --enable-ssh-local-port-forwarding \
  --enable-ssh-remote-port-forwarding \
  --enable-ssh-sftp \
  --enable-ssh-root
```

### Connect via NetBird CLI

```bash
# Interactive shell
netbird ssh user@100.119.230.104

# Execute a command
netbird ssh user@100.119.230.104 "uptime"

# Force TTY
netbird ssh -t user@100.119.230.104

# Local port forwarding
netbird ssh -L 8080:localhost:80 user@100.119.230.104

# Remote port forwarding
netbird ssh -R 8080:localhost:3000 user@100.119.230.104

# Custom SSH port
netbird ssh -p 2222 user@100.119.230.104

# Disable strict host key checking
netbird ssh --strict-host-key-checking=false user@100.119.230.104
```

### Native OpenSSH integration

NetBird writes a drop-in config at `/etc/ssh/ssh_config.d/99-netbird.conf`. Once SSH is enabled, you can usually run:

```bash
ssh user@peer-name.netbird.cloud
sftp user@peer-name.netbird.cloud
scp file.txt user@peer-name.netbird.cloud:/tmp/
```

NetBird SSH uses an internal `22022` endpoint on peers, but access policies should use the `NetBird SSH` protocol for fine-grained SSH access, or TCP port `22` for basic network-level SSH. Modern management servers add the internal `22022` rule automatically when a port-22 policy targets a peer with native SSH enabled.

### SSH troubleshooting checklist

1. SSH enabled on target peer? `netbird up --allow-server-ssh`
2. SSH Access enabled in dashboard for that peer?
3. ACL policy uses protocol `NetBird SSH`, or TCP `22` for basic network-level SSH
4. If using JWT auth, complete the browser OIDC flow when prompted
5. To bypass JWT auth (host-only trust): `netbird up --allow-server-ssh --disable-ssh-auth`
6. Port forwarding requires the matching `--enable-ssh-*-port-forwarding` flag

## Networks and Routes

`netbird networks` replaces the old `netbird routes` command; `routes` is an alias.

```bash
# List available networks/routes
netbird networks list
netbird routes list

# Accept all networks (including future ones)
netbird networks select all

# Select specific networks
netbird networks select route1 route2

# Append to current selection
netbird networks select -a route3

# Disable all network routes
netbird networks deselect all

# Deselect specific networks
netbird networks deselect route1 route2
```

## Expose Local Ports

NetBird can expose a local service through the NetBird reverse proxy.

```bash
# Expose HTTP on port 8080
netbird expose 8080

# Expose with password protection
netbird expose --with-password my-secret 8080

# Expose with 6-digit PIN
netbird expose --with-pin 123456 8080

# Expose TCP (L4)
netbird expose --protocol tcp 5432

# Expose TCP on a different public port
netbird expose --protocol tcp --with-external-port 5433 5432

# Expose with custom domain
netbird expose --protocol tls --with-custom-domain tls.example.com 4443

# Restrict to user groups
netbird expose --with-user-groups devops,Backend 8080

# Name prefix for the generated service
netbird expose --with-name-prefix my-app 8080
```

Supported protocols: `http`, `https`, `tcp`, `udp`, `tls`.

## Forwarding Rules

```bash
# List active forwarding rules
netbird forwarding list
```

## Profiles

Profiles let one client maintain separate accounts/tenants.

```bash
# Add a profile
netbird profile add work

# List profiles
netbird profile list
netbird profile ls

# Switch profile
netbird profile select work

# Remove a profile (must not be active)
netbird profile remove work
```

Use `--profile <name>` with `login`, `up`, and `deregister` to target a specific profile.

## Debug and Troubleshooting

### Debug bundles and logging

```bash
# Create a debug bundle
netbird debug bundle

# Include system info and anonymize
netbird debug bundle -S -A

# Run trace logging for 5 minutes, then bundle
netbird debug for 5m
netbird debug for 5m -A -S

# Change daemon log level temporarily
netbird debug log level debug
netbird debug log level trace
```

### Packet capture

Requires `--enable-capture` at service install/reconfigure time.

```bash
# Live text capture
netbird debug capture

# Capture to pcap file
netbird debug capture -o capture.pcap

# Capture with BPF-style filter
netbird debug capture host 100.64.0.1 and port 443
netbird debug capture tcp
netbird debug capture icmp

# Stream pcap to tshark/tcpdump
netbird debug capture --pcap | tshark -r -
netbird debug capture --pcap | tcpdump -r - -n
```

### Firewall trace

```bash
# Trace incoming TCP SYN to self
netbird debug trace in 100.64.1.1 self -p tcp --dport 80 --syn

# Trace outgoing UDP DNS
netbird debug trace out 10.10.0.1 8.8.8.8 -p udp --dport 53
```

### Persistence

```bash
# Keep the last sync response in memory
netbird debug persistence on
netbird debug persistence off
```

## State Management

```bash
# List stored daemon states
netbird state list

# Clean states
netbird state clean

# Delete a specific state
netbird state delete <state-id>
```

## Common Workflows

### First connect (NetBird Cloud)

```bash
netbird up
# Complete SSO in the opened browser
netbird status
```

### First connect (self-hosted)

```bash
netbird up --management-url https://api.self-hosted.com:33073
netbird status
```

### Headless server deployment

```bash
# Install and start service
sudo netbird service install
sudo netbird service start

# Connect with setup key
sudo netbird up --setup-key AAAA-BBB-CCC-DDDDDD --management-url https://api.self-hosted.com:33073

# Verify
sudo netbird status
```

### Enable SSH on a server

```bash
netbird down
netbird up --allow-server-ssh \
  --enable-ssh-local-port-forwarding \
  --enable-ssh-remote-port-forwarding \
  --enable-ssh-sftp
netbird status
```

Then in the dashboard enable **SSH Access** for that peer and create an access policy using protocol `NetBird SSH` (preferred for user-aware access) or TCP `22` for basic network-level SSH.

### Connect to a peer via SSH

```bash
# Find peer IP
netbird status --json | jq '.peers.details[] | {fqdn, netbirdIp, status}'

# Connect
netbird ssh alice@100.64.0.42

# Run a command
netbird ssh alice@100.64.0.42 "systemctl status myapp"
```

### Select specific routes

```bash
netbird networks list
netbird networks select route1 route2
netbird status | grep -i routes
```

### Generate a support bundle

```bash
netbird debug for 2m -A -S
# Upload the printed .zip path to support
```

### Switch tenant/profile

```bash
netbird profile add customer-a
netbird profile select customer-a
netbird up --management-url https://customer-a.api.example.com
```

## Version

```bash
netbird version
```

## Agent Usage Tips

1. **Always check the daemon first**: `sudo netbird service status`
2. **Prefer `--json` for programmatic parsing** in scripts and agent workflows
3. **Use `-A` / `--anonymize`** when sharing status or debug output
4. **Verify Management and Signal are Connected** before troubleshooting peer connectivity
5. **Check relays**: `0/N Available` usually indicates firewall, MTU, or outbound UDP issues
6. **Use `netbird status -d --filter-by-status connected`** to focus on live peers
7. **Remember profiles**: if a command behaves unexpectedly, confirm the active profile with `netbird profile list`
8. **Service changes need `reconfigure` or `restart`** to take effect after `service install`
9. **For headless peers**, use `--setup-key` or `--setup-key-file`; do not rely on browser SSO
10. **Packet capture requires `--enable-capture`** at service install/reconfigure time
