# OpenClaw VPS Setup Guide
## Secure Self-Hosted Personal AI Assistant

This guide walks through setting up OpenClaw on a VPS with security best practices. All personal data has been anonymized.

---

## Prerequisites

- A VPS (Ubuntu 22.04+ recommended)
- SSH access to your VPS
- A domain or static IP (optional)
- Tailscale account (free tier works)
- Node.js 20+ 

---

## Part 1: Initial VPS Setup

### 1.1 Create Non-Root User

```bash
# SSH in as root initially
ssh root@YOUR_VPS_IP

# Create a new user
adduser yourusername
usermod -aG sudo yourusername

# Switch to new user
su - yourusername
```

### 1.2 SSH Key Authentication

On your **local machine**:
```bash
# Generate SSH key if you don't have one
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy to VPS
ssh-copy-id yourusername@YOUR_VPS_IP
```

### 1.3 Harden SSH

On the VPS:
```bash
sudo nano /etc/ssh/sshd_config
```

Set these values:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

Restart SSH:
```bash
sudo systemctl restart sshd
```

> ⚠️ Keep your current SSH session open until you verify you can connect with keys!

---

## Part 2: Tailscale VPN Setup

Tailscale creates a secure mesh network. After setup, you'll access your VPS only through Tailscale — not the public internet.

### 2.1 Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 2.2 Connect to Tailnet

```bash
sudo tailscale up
```

Follow the authentication link to connect your VPS to your Tailscale network.

### 2.3 Get Your Tailscale IP

```bash
tailscale ip -4
```

Note this IP (e.g., `100.x.x.x`) — you'll use it for SSH going forward.

### 2.4 Verify Connectivity

From another device on your Tailnet:
```bash
ssh yourusername@100.x.x.x
```

---

## Part 3: Firewall Configuration (UFW)

We'll lock down the VPS to only accept SSH via Tailscale.

### 3.1 Install and Enable UFW

```bash
sudo apt update
sudo apt install ufw -y
```

### 3.2 Configure Firewall Rules

```bash
# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH only from Tailscale network (100.64.0.0/10)
sudo ufw allow from 100.64.0.0/10 to any port 22 proto tcp

# Enable firewall
sudo ufw enable
```

### 3.3 Verify Rules

```bash
sudo ufw status verbose
```

Expected output:
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       100.64.0.0/10
```

> ⚠️ You've now blocked public SSH access. Only Tailscale-connected devices can reach your VPS.

---

## Part 4: Install Node.js

### 4.1 Install via NodeSource

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### 4.2 Configure npm Global Directory (Avoid sudo for npm)

```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 4.3 Verify Installation

```bash
node --version   # Should show v22.x.x
npm --version    # Should show 10.x.x
```

---

## Part 5: Install OpenClaw

### 5.1 Install Globally

```bash
npm install -g openclaw
```

### 5.2 Initialize Configuration

```bash
openclaw init
```

Follow the prompts to:
- Set your API key (Anthropic/OpenAI)
- Configure your workspace directory
- Set up messaging integrations (Telegram, WhatsApp, etc.)

### 5.3 Workspace Location

Default workspace: `~/.openclaw/workspace`

This is where your skills, memory files, and configurations live.

---

## Part 6: Secure Credential Management

**Never store API keys in files the agent can read.** Use environment variables.

### 6.1 Create Environment File

```bash
nano ~/.openclaw/.env
```

Add your credentials:
```bash
# API Keys (examples - use your own)
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
FUB_API_KEY=your-fub-api-key
OTHER_SERVICE_KEY=your-key-here
```

### 6.2 Secure the File

```bash
chmod 600 ~/.openclaw/.env
```

### 6.3 Load in OpenClaw

OpenClaw automatically loads `~/.openclaw/.env` at startup.

### 6.4 In Scripts — Use Environment Variables

```javascript
// Good - reads from environment
const apiKey = process.env.FUB_API_KEY;

// Bad - never do this
const apiKey = "sk-xxxxxxxxxxxxx";
```

---

## Part 7: Start OpenClaw Gateway

### 7.1 Start the Gateway

```bash
openclaw gateway start
```

### 7.2 Run as Background Service (Recommended)

Create a systemd service:

```bash
sudo nano /etc/systemd/system/openclaw.service
```

```ini
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=yourusername
WorkingDirectory=/home/yourusername
ExecStart=/home/yourusername/.npm-global/bin/openclaw gateway
Restart=on-failure
RestartSec=10
Environment=PATH=/home/yourusername/.npm-global/bin:/usr/local/bin:/usr/bin:/bin

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable openclaw
sudo systemctl start openclaw
```

### 7.3 Check Status

```bash
sudo systemctl status openclaw
# or
openclaw status
```

---

## Part 8: Additional Security Measures

### 8.1 Automatic Security Updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 8.2 Fail2Ban (Optional but Recommended)

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### 8.3 Regular Backups

Back up your workspace and config:
```bash
# Create backup
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw

# Store off-server (rsync to your local machine via Tailscale)
rsync -avz ~/.openclaw/workspace/ yourlocalpath/openclaw-backup/
```

---

## Part 9: Messaging Integration

### 9.1 Telegram Bot Setup

1. Message @BotFather on Telegram
2. Create a new bot: `/newbot`
3. Copy the API token
4. Add to your config:

```bash
openclaw configure
```

Select Telegram and paste your token.

### 9.2 Test Connection

Send a message to your bot. OpenClaw should respond.

---

## Part 10: Docker Installation (Optional)

Docker enables containerized deployments and is useful for running additional services alongside OpenClaw.

### 10.1 Install Docker

```bash
# Add Docker's official GPG key
sudo apt update
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### 10.2 Run Docker Without Sudo

```bash
# Add your user to the docker group
sudo usermod -aG docker $USER

# Apply changes (log out and back in, or run:)
newgrp docker
```

### 10.3 Verify Installation

```bash
docker --version
docker run hello-world
```

### 10.4 Docker Security Best Practices

```bash
# Don't run containers as root
docker run --user 1000:1000 myimage

# Limit container resources
docker run --memory="512m" --cpus="1.0" myimage

# Use read-only filesystem where possible
docker run --read-only myimage

# Don't expose ports publicly (use Tailscale)
docker run -p 127.0.0.1:8080:8080 myimage
```

### 10.5 Docker Compose (Optional)

Docker Compose is included with modern Docker. Verify:
```bash
docker compose version
```

Example `docker-compose.yml` for additional services:
```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "127.0.0.1:5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped

volumes:
  n8n_data:
```

> 💡 Bind to `127.0.0.1` instead of `0.0.0.0` to keep services local. Access them via Tailscale.

---

## Part 11: Skill Security Scanning

Before installing any third-party skills, scan them for security threats using Cisco's Skill Scanner.

### 11.1 Install Skill Scanner

```bash
pip3 install cisco-ai-skill-scanner
```

### 11.2 Scan a Skill Before Installing

```bash
skill-scanner scan /path/to/skill
```

### 11.3 Full Scan with All Engines

```bash
skill-scanner scan /path/to/skill --use-behavioral --use-llm
```

### 11.4 Understanding Results

The scanner checks for:
- **Prompt injection** — Instructions that hijack the agent
- **Data exfiltration** — Code that sends data to external servers
- **Command injection** — Malicious shell commands
- **Tool poisoning** — Hidden payloads in skill files

Severity levels:
- 🔴 **Critical** — Do not install
- 🟠 **High** — Do not install without review
- 🟡 **Medium** — Review before installing
- 🟢 **Low/Info** — Generally safe

### 11.5 Scan Multiple Skills

```bash
skill-scanner scan-all /path/to/skills --recursive
```

### 11.6 CI/CD Integration

For automated scanning in pipelines:
```bash
skill-scanner scan-all ./skills --fail-on-findings --format sarif --output results.sarif
```

> ⚠️ **Rule: Never install a skill without scanning it first.** Malicious skills can exfiltrate data, execute commands, and bypass agent safety guidelines.

More info: https://github.com/cisco-ai-defense/skill-scanner

---

## Security Checklist

- [ ] Non-root user created
- [ ] SSH key authentication only (password disabled)
- [ ] Root login disabled
- [ ] Tailscale VPN installed and connected
- [ ] UFW firewall enabled (SSH via Tailscale only)
- [ ] Public SSH port blocked
- [ ] API keys in environment variables (not files)
- [ ] `.env` file permissions set to 600
- [ ] OpenClaw running as systemd service
- [ ] Unattended security updates enabled
- [ ] Fail2Ban installed (optional)
- [ ] Backup strategy in place
- [ ] Docker installed (optional)
- [ ] User added to docker group (no sudo needed)
- [ ] Cisco Skill Scanner installed
- [ ] All third-party skills scanned before installation

---

## Connecting From Your Devices

1. Install Tailscale on your phone/laptop
2. Sign in with the same account
3. SSH to your VPS using the Tailscale IP:
   ```bash
   ssh yourusername@100.x.x.x
   ```
4. Your VPS is invisible to the public internet but accessible from any of your devices

---

## Troubleshooting

### Can't SSH after firewall changes
- Use VPS provider's web console to regain access
- Run `sudo ufw disable` to temporarily disable firewall
- Fix rules and re-enable

### OpenClaw not starting
```bash
openclaw doctor
journalctl -u openclaw -f
```

### Tailscale not connecting
```bash
sudo tailscale status
sudo systemctl restart tailscaled
```

---

## Maintenance

### Update OpenClaw
```bash
npm update -g openclaw
sudo systemctl restart openclaw
```

### Check for OS updates
```bash
sudo apt update && sudo apt upgrade -y
```

### View OpenClaw logs
```bash
journalctl -u openclaw -f
```

---

## Summary

This setup gives you:

1. **Network Security**: VPS only accessible via Tailscale VPN
2. **Access Control**: SSH key authentication, no passwords
3. **Firewall Protection**: UFW blocks all public access
4. **Credential Security**: API keys in env vars, not readable by agent
5. **Service Reliability**: systemd keeps OpenClaw running
6. **Automatic Updates**: Security patches applied automatically

Your personal AI assistant is now running securely on your own infrastructure.

---

## Attribution

Created by **Jeremy Kritt** with assistance from OpenClaw AI.

- GitHub: [github.com/jmkritt](https://github.com/jmkritt)
- OpenClaw: [openclaw.ai](https://openclaw.ai)

*Guide created: February 2026*
*OpenClaw version: 2026.2.x*

---

*Licensed under MIT — free to use, modify, and share.*
