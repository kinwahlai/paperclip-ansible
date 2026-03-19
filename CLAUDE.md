# Paperclip Ansible — Claude Code Instructions

## What this repo is
Ansible playbook to install [Paperclip](https://github.com/paperclipai/paperclip) (AI agent
orchestration platform) on Debian/Ubuntu. Installs via `npm install -g paperclipai`, runs as a
systemd service under the `paperclip` system user.

## Critical constraints

- **Port 3100 is localhost-only.** Never add UFW rules to expose it externally. Access is via
  Tailscale or SSH tunnel only.
- **`onboard` runs once.** It's guarded by `~/.paperclip/config.json` existence. Never remove
  that guard — re-running onboard is destructive.
- **Migrations are automatic.** `paperclip run` calls `ensureMigrations()` on startup. No
  separate migrate step needed. Do not add `pnpm db:migrate` tasks.
- **No git clone.** Install is via `npm install -g paperclipai`. Do not reintroduce a source
  build approach.

## Task order (must not change)

```
nodejs.yml  →  user.yml  →  install.yml  →  service.yml
```

`service.yml` is skipped in `ci_test` mode. `onboard` is also skipped in `ci_test` mode.

## Upgrade flow

Re-run the playbook. `install.yml` compares pre/post versions and notifies the `Restart paperclip`
handler only when the version changes. The service restart triggers auto-migration on startup.

## Key paths on the target host

```
/usr/local/bin/paperclip            # installed binary
/home/paperclip/.paperclip/         # config, DB, keys, storage (all data lives here)
/etc/systemd/system/paperclip.service
/etc/sudoers.d/paperclip
```

## Before committing any change

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
ansible-lint playbook.yml
```

## Full reference
See [AGENTS.md](./AGENTS.md) for detailed project guidelines, code style, and testing checklist.
