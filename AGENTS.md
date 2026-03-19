# Agent Guidelines

## Project Overview

Ansible playbook for automated installation of [Paperclip](https://github.com/paperclipai/paperclip)
on Debian/Ubuntu Linux. Paperclip is an AI agent orchestration platform (Node.js + React UI)
that manages multiple AI agents as an autonomous organization.

## Key Principles

1. **Localhost only**: Port 3100 binds to 127.0.0.1. Access via Tailscale or SSH tunnel only.
2. **Production mode**: Always install from source (`pnpm build`), run compiled output.
3. **Non-root**: Paperclip runs as the `paperclip` system user with scoped sudo.
4. **Idempotent**: All tasks must be safe to run multiple times.

## Task Order

```
roles/paperclip/tasks/main.yml:
  nodejs.yml   → Install Node.js 22.x + pnpm (system-wide)
  user.yml     → Create paperclip user, sudoers, SSH keys
  install.yml  → git clone/pull, pnpm install, pnpm build, db:migrate
  service.yml  → Deploy + enable systemd service (skipped in ci_test mode)
```

## Critical Notes

### ExecStart in paperclip.service.j2
The service runs `node dist/index.js` from `{{ paperclip_repo_dir }}/server`.
This assumes the server package compiles to `server/dist/index.js` after `pnpm build`.
Verify this path against the actual repo structure before deploying.

### Database Migrations
`pnpm db:migrate` is run on every install/update. Migrations must be idempotent —
this is guaranteed by Paperclip's migration framework (Drizzle ORM).

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
/home/paperclip/paperclip/     # Cloned repo + built app
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
│   └── paperclip.service.j2
├── defaults/
│   └── main.yml
└── handlers/
    └── main.yml
```

## Making Changes

### Updating Paperclip
Re-run the playbook — it will `git pull`, `pnpm install`, `pnpm build`,
run migrations, and restart the service automatically.

### Changing the Port
Override `paperclip_port` variable. The service template and docs reference
this variable — no hardcoded port values in tasks or templates.

### Adding Environment Variables
Add `Environment=` lines to `paperclip.service.j2`. Restart is triggered
automatically via the handler.
