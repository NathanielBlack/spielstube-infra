# CLAUDE.md

## Project Overview

Ansible infrastructure for the Spielstube game hosting platform at spielstube.app. Each game runs in Docker containers behind a shared nginx reverse proxy with auto-SSL.

## Server

- **IP:** 152.53.87.246
- **OS:** Debian 11 Bullseye
- **Specs:** 4 CPU, 8GB RAM, 256GB disk
- **SSH key:** ~/.ssh/danik
- **User:** root (deploy user also available)

## Architecture

```
nginx-proxy + acme-companion (shared, auto-SSL)
  ├── hamburg1949.spielstube.app         → Hamburg 1949 map game
  ├── kenn-dein-hamburg.spielstube.app   → Hamburg OSM quiz
  ├── wiki-quiz.spielstube.app           → Wikipedia trivia
  ├── digit-commander.spielstube.app     → Digit Commander
  ├── marapoop.spielstube.app            → Marapoop
  ├── rps.spielstube.app                 → Rock Paper Scissors
  ├── companion.journaley.spielstube.app → Journaley PWA (nginx static)
  ├── api.journaley.spielstube.app       → Journaley sync API (.NET + Postgres)
  ├── registry.spielstube.app            → Private Docker registry
  └── <name>.spielstube.app              → future apps
```

- Shared `proxy` Docker network — all web-facing containers join it
- nginx-proxy routes by `VIRTUAL_HOST` env var, acme-companion handles Let's Encrypt
- Each game has its own docker-compose with its own deps (DB, cache, etc.)
- Images built locally, pushed to registry.spielstube.app, deployed via ansible

## Structure

```
ansible.cfg              → Config (vault password file, inventory path)
.vault_pass              → Ansible vault password (gitignored)
inventory/
  hosts.yml              → Server connection details
  vault.yml              → Encrypted shared secrets (registry password)
playbooks/
  setup-server.yml       → Full server bootstrap (common + docker + proxy + registry)
  deploy-game.yml        → Generic game deploy (pass -e game_name=xxx)
roles/
  common/                → UFW, fail2ban, base packages, deploy user
  docker/                → Docker CE, shared proxy network
  proxy/                 → nginx-proxy + acme-companion
  registry/              → Private Docker registry with basic auth
  game/                  → Generic game deployer (pull + start)
games/
  digit-commander/       → Digit Commander game
  hamburg1949/           → Hamburg 1949 map game (needs ANTHROPIC_API_KEY, ELEVENLABS_API_KEY)
  journaley/             → Journaley PWA + sync API + Postgres (3 containers, 2 on proxy)
  kenn-dein-hamburg/     → Hamburg OSM quiz
  marapoop/              → Marapoop game
  rps/                   → Rock Paper Scissors
  wiki-quiz/             → Wikipedia trivia
  Each contains: deploy.yml, docker-compose.yml.j2, env.j2, vars.yml (encrypted)
```

## Commands

```bash
# Full server setup (first time or to update)
ansible-playbook playbooks/setup-server.yml

# Deploy a specific game
ansible-playbook games/hamburg1949/deploy.yml

# Encrypt a secrets file
ansible-vault encrypt games/<game>/vars.yml

# Edit encrypted secrets
ansible-vault edit games/<game>/vars.yml

# SSH into server
ssh -i ~/.ssh/danik root@152.53.87.246

# Check containers on server
ssh -i ~/.ssh/danik root@152.53.87.246 docker ps
```

## Adding a New Game

1. Create `games/<name>/` with `deploy.yml`, `docker-compose.yml.j2`, `env.j2`, `vars.yml`
2. The docker-compose must set `VIRTUAL_HOST`, `VIRTUAL_PORT`, `LETSENCRYPT_HOST` and join the `proxy` network
3. Encrypt vars: `ansible-vault encrypt games/<name>/vars.yml`
4. DNS: wildcard `*.spielstube.app` covers all subdomains including nested ones (e.g. `companion.journaley.spielstube.app`) — no new records needed
5. Build and push image: `docker build --platform=linux/amd64 -t registry.spielstube.app/<name>:latest . && docker push ...`
6. Deploy: `ansible-playbook games/<name>/deploy.yml`

## Workflow

- **No auto-commits.** Only commit when explicitly asked.
- Vault password is in `.vault_pass` (gitignored), referenced by `ansible.cfg`.
- Always use `--platform=linux/amd64` when building Docker images (server is x86).

## Global Deploy Skill

There is a global Claude Code skill at `~/.claude/commands/deploy-game.md` that documents the full deploy workflow for any Claude instance. **When making changes to the infrastructure (new games, server changes, architecture updates), always keep that skill file updated too.**
