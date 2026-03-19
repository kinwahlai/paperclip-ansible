# Agent Guidelines

## Project Overview

Ansible playbook for automated installation of [Paperclip](https://github.com/paperclipai/paperclip)
on Debian/Ubuntu Linux. Paperclip is an AI agent orchestration platform (Node.js + React UI)
that manages multiple AI agents as an autonomous organization.

## Key Principles

1. **Localhost only**: Port 3100 binds to 127.0.0.1. Access via Tailscale or SSH tunnel only.
2. **npm install**: Both opencode-ai and paperclipai are installed via `npm install -g`. No git clone or source build.
3. **Non-root**: Paperclip runs as the `paperclip` system user with scoped sudo.
4. **Idempotent**: All tasks must be safe to run multiple times.

## Task Order

```
roles/paperclip/tasks/main.yml:
  nodejs.yml   → Install Node.js 22.x + pnpm (system-wide)
  user.yml     → Create paperclip user, sudoers, SSH keys
  install.yml  → npm install -g opencode-ai, deploy opencode config, npm install -g paperclipai
  service.yml  → Deploy + enable systemd service (skipped in ci_test mode)
```

## Critical Notes

### Binary name
The installed binary is `paperclipai` (not `paperclip`). Its path is resolved dynamically
via `npm prefix -g` and stored in the `paperclip_bin` fact.

### Database Migrations
Migrations run automatically when `paperclipai run` starts (`ensureMigrations()`).
No separate migrate step is needed — do not add `pnpm db:migrate` tasks.

### No UFW changes
Unlike openclaw-ansible, this playbook makes no UFW changes. Port 3100 is
localhost-only by default. No firewall rules are added or removed.

## Code Style

### Ansible
- Use loops instead of repeated tasks
- No `become_user` at the play level — use per-task `become_user` where needed
- Always specify `executable: /bin/bash` on shell tasks
- Use `creates:` to make shell tasks idempotent where possible

## Testing Checklist

```bash
# 1. Syntax check
ansible-playbook playbook.yml --syntax-check

# 2. Dry run
ansible-playbook playbook.yml --check --ask-become-pass

# 3. Full install (on test VM)
ansible-playbook playbook.yml --ask-become-pass

# 4. Verify service
systemctl status paperclip
curl http://127.0.0.1:3100/health

# 5. Run test suite
bash tests/run-tests.sh
```

## File Locations

### Host System
```
/usr/local/bin/paperclipai                        # installed binary
/home/paperclip/.paperclip/                       # config, DB, keys, storage
/home/paperclip/.config/opencode/opencode.json    # opencode config (if api key set)
/etc/systemd/system/paperclip.service
/etc/sudoers.d/paperclip
```

### Repository
```
roles/paperclip/
├── tasks/
│   ├── main.yml
│   ├── nodejs.yml
│   ├── user.yml
│   ├── install.yml
│   └── service.yml
├── templates/
│   ├── paperclip.service.j2
│   └── opencode.json.j2
├── defaults/
│   └── main.yml
└── handlers/
    └── main.yml
```

## Making Changes

### Updating Paperclip
Re-run the playbook — it will reinstall via `npm install -g paperclipai`,
and restart the service only if the version changed. Migrations apply automatically on startup.

### Changing the Port
Override `paperclip_port` variable. The service template and docs reference
this variable — no hardcoded port values in tasks or templates.

### Adding Environment Variables
Add `Environment=` lines to `paperclip.service.j2`. Restart is triggered
automatically via the handler.
