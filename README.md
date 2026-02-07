# OpenClaw VPS Setup Guide

A comprehensive guide for setting up [OpenClaw](https://openclaw.ai) on a VPS with security best practices.

## What's Covered

- ✅ Non-root user setup
- ✅ SSH key authentication & hardening
- ✅ Tailscale VPN installation
- ✅ UFW firewall configuration
- ✅ Node.js & OpenClaw installation
- ✅ Secure credential management
- ✅ Systemd service setup
- ✅ Docker installation & security
- ✅ Skill security scanning (Cisco)
- ✅ Automatic updates & backups
- ✅ Complete security checklist

## Quick Start

Read the full guide: **[GUIDE.md](GUIDE.md)**

## Why This Guide?

OpenClaw is powerful — it can run shell commands, access files, and control your system. That power requires proper security:

1. **Network isolation** — Tailscale VPN, not public internet
2. **Access control** — SSH keys only, no passwords
3. **Credential security** — Environment variables, not files
4. **Skill vetting** — Scan before installing

## Security Tools Used

| Tool | Purpose |
|------|---------|
| [Tailscale](https://tailscale.com) | Zero-config VPN mesh network |
| UFW | Simple firewall management |
| [Cisco Skill Scanner](https://github.com/cisco-ai-defense/skill-scanner) | Detect malicious AI skills |
| Fail2Ban | Brute-force protection |

## License

MIT — free to use, modify, and share.

## Author

Created by **Jeremy Kritt** with assistance from OpenClaw AI.

- GitHub: [@jmkritt](https://github.com/jmkritt)
- OpenClaw: [openclaw.ai](https://openclaw.ai)

---

*Last updated: February 2026*
