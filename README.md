# NetBird CLI Skill

A comprehensive agent skill for mastering the NetBird CLI (`netbird`).

## What it covers

- Connection lifecycle: `up`, `down`, `login`, `logout`/`deregister`
- Service/daemon management: `service install/start/stop/restart/status/uninstall/reconfigure`
- Status interpretation and peer debugging
- SSH access via `netbird ssh` and native OpenSSH integration
- Networks and routes (`netbird networks`, alias `routes`)
- Port exposure through the NetBird reverse proxy (`netbird expose`)
- Forwarding rules, profiles, state management
- Debug tools: bundles, logs, packet capture, firewall traces
- Environment variables, global flags, configuration precedence
- Real-world workflows for cloud, self-hosted, and headless deployments

## Install

```bash
npx skills add manawenuz/netbird-cli
```

Or install the `SKILL.md` directly:

```bash
npx skills add https://raw.githubusercontent.com/manawenuz/netbird-cli/main/SKILL.md
```

## Usage

Once installed, the agent will automatically reference this skill when you ask NetBird-related questions. Example prompts:

- "How do I connect to my self-hosted NetBird?"
- "Enable SSH on this peer and connect to another"
- "Why is my peer showing relayed instead of P2P?"
- "Create a debug bundle for NetBird support"

## License

MIT
