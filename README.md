# Paperclip Ansible Installer

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Lint](https://github.com/kinwahlai/paperclip-ansible/actions/workflows/lint.yml/badge.svg)](https://github.com/kinwahlai/paperclip-ansible/actions/workflows/lint.yml)
[![Ansible](https://img.shields.io/badge/Ansible-2.14+-blue.svg)](https://www.ansible.com/)
[![OS](https://img.shields.io/badge/OS-Debian%20%7C%20Ubuntu-orange.svg)](https://www.debian.org/)

Automated installation of [Paperclip](https://github.com/paperclipai/paperclip) on Debian/Ubuntu
Linux. Paperclip is an AI agent orchestration platform — if OpenClaw is an employee, Paperclip is
the company.

## Features

- **One-command install**: Complete setup in minutes
- **Production-ready**: Runs as a hardened systemd service under a dedicated system user
- **Auto-migration**: Database migrations apply automatically on every startup
- **Zero-downtime upgrades**: Re-run the playbook to upgrade — service restarts only when version changes
- **Secure by default**: Port 3100 is localhost-only, access via Tailscale or SSH tunnel
- **Scoped sudo**: `paperclip` user can only manage its own service

## Requirements

- Debian 11+ or Ubuntu 20.04+
- Root / sudo access
- Internet connection
- Ansible 2.14+

## Quick Start

```bash
# Clone the installer
git clone https://github.com/kinwahlai/paperclip-ansible.git
cd paperclip-ansible

# Install Ansible collections
ansible-galaxy collection install -r requirements.yml

# Run installation
ansible-playbook playbook.yml --ask-become-pass
```

## What Gets Installed

- Node.js 22.x
- Paperclip (`npm install -g paperclipai`)
- `paperclip` system user with scoped sudoers
- First-time setup via `paperclip onboard --yes` (config, embedded Postgres, encryption keys)
- Systemd service (`paperclip run`) with auto-start and restart on failure

## Post-Install

After the playbook completes, Paperclip is running at `http://127.0.0.1:3100`.

Access the UI via SSH tunnel:

```bash
ssh -L 3100:127.0.0.1:3100 user@your-server
```

Then open `http://localhost:3100` in your browser.

Or via Tailscale:

```bash
ssh -L 3100:127.0.0.1:3100 user@your-tailscale-hostname
```

Check service status:

```bash
sudo systemctl status paperclip
sudo journalctl -u paperclip -f
```

## Upgrading

Re-run the playbook. It will upgrade the binary, and restart the service only if the version changed.
Database migrations apply automatically on startup.

```bash
ansible-playbook playbook.yml --ask-become-pass
```

## Configuration

Override variables via command line or a vars file.

### Available Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `paperclip_user` | `paperclip` | System user name |
| `paperclip_home` | `/home/paperclip` | User home directory |
| `paperclip_port` | `3100` | API server port (localhost-only) |
| `nodejs_version` | `22.x` | Node.js version to install |
| `paperclip_ssh_keys` | `[]` | SSH public keys for the paperclip user |

### Via command line

```bash
ansible-playbook playbook.yml --ask-become-pass \
  -e "paperclip_ssh_keys=['ssh-ed25519 AAAAC3... user@host']"
```

### Via vars file

```bash
cat > vars.yml << EOF
paperclip_ssh_keys:
  - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGxxxxxxxx user@host"
EOF

ansible-playbook playbook.yml --ask-become-pass -e @vars.yml
```

## Security

- **Port exposure**: Only SSH (22) needs to be open. Port 3100 never leaves localhost.
- **Non-root**: Paperclip runs as an unprivileged system user.
- **Scoped sudo**: Limited to `systemctl start|stop|restart|status paperclip` and journal access.
- **Systemd hardening**: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=strict`.

## Testing

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Dry run
ansible-playbook playbook.yml --check --ask-become-pass

# Docker-based test suite (Ubuntu 24.04)
bash tests/run-tests.sh
```

## Documentation

- [Agent Guidelines](AGENTS.md) - Guidelines for AI agents and contributors
- [Paperclip Docs](https://paperclip.ing/docs) - Official Paperclip documentation
- [Paperclip Repo](https://github.com/paperclipai/paperclip) - Upstream project

## License

MIT
