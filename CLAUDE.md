# CLAUDE.md

## Project Overview

Ansible infrastructure for the Spielstube game hosting platform at spielstu.be. Each game runs in Docker containers behind a shared nginx reverse proxy with auto-SSL.

## Server

- **IP:** 152.53.87.246
- **OS:** Debian 11 Bullseye
- **Specs:** 4 CPU, 8GB RAM, 256GB disk
- **SSH key:** ~/.ssh/danik
- **User:** root (deploy user also available)

## Architecture

```
nginx-proxy + acme-companion (shared, auto-SSL)
  ├── hamburg.spielstu.be  → hamburg game container
  ├── registry.spielstu.be → private Docker registry
  └── <game>.spielstu.be   → future games
```

- Shared `proxy` Docker network — all web-facing containers join it
- nginx-proxy routes by `VIRTUAL_HOST` env var, acme-companion handles Let's Encrypt
- Each game has its own docker-compose with its own deps (DB, cache, etc.)
- Images built locally, pushed to registry.spielstu.be, deployed via ansible

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
  hamburg/               → Hamburg 1949 map game config
    deploy.yml           → ansible-playbook games/hamburg/deploy.yml
    docker-compose.yml.j2
    env.j2
    vars.yml             → Encrypted API keys
```

## Commands

```bash
# Full server setup (first time or to update)
ansible-playbook playbooks/setup-server.yml

# Deploy a specific game
ansible-playbook games/hamburg/deploy.yml

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
4. Add DNS A record for `<name>.spielstu.be` → 152.53.87.246
5. Build and push image: `docker build --platform=linux/amd64 -t registry.spielstu.be/<name>:latest . && docker push ...`
6. Deploy: `ansible-playbook games/<name>/deploy.yml`

## Workflow

- **No auto-commits.** Only commit when explicitly asked.
- Vault password is in `.vault_pass` (gitignored), referenced by `ansible.cfg`.
- Always use `--platform=linux/amd64` when building Docker images (server is x86).

## Global Deploy Skill

There is a global Claude Code skill at `~/.claude/commands/deploy-game.md` that documents the full deploy workflow for any Claude instance. **When making changes to the infrastructure (new games, server changes, architecture updates), always keep that skill file updated too.**
