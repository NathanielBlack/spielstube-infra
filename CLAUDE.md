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

Non-web containers (no subdomain, outbound only):
  ├── sababa-bot                         → Sababa Telegram bot
  └── watchtower                         → auto-redeploy on registry push
```

- Shared `proxy` Docker network — all web-facing containers join it
- nginx-proxy routes by `VIRTUAL_HOST` env var, acme-companion handles Let's Encrypt
- Each game has its own docker-compose with its own deps (DB, cache, etc.)
- Images built locally, pushed to registry.spielstube.app, deployed via ansible
- Watchtower auto-redeploys *labeled* containers when a new image is pushed (see "Auto-redeploy" below)

## Structure

```
ansible.cfg              → Config (vault password file, inventory path)
.vault_pass              → Ansible vault password (gitignored)
inventory/
  hosts.yml              → Server connection details
  vault.yml              → Encrypted shared secrets (registry password)
playbooks/
  setup-server.yml       → Full server bootstrap (common + docker + proxy + registry + watchtower)
  deploy-game.yml        → Generic game deploy (pass -e game_name=xxx)
  watchtower.yml         → Deploy/update Watchtower auto-redeploy
roles/
  common/                → UFW, fail2ban, base packages, deploy user
  docker/                → Docker CE, shared proxy network
  proxy/                 → nginx-proxy + acme-companion
  registry/              → Private Docker registry with basic auth + push notifications
  watchtower/            → Watchtower (HTTP-trigger auto-redeploy on push)
  game/                  → Generic game deployer (pull + start)
games/
  digit-commander/       → Digit Commander game
  hamburg1949/           → Hamburg 1949 map game (needs ANTHROPIC_API_KEY, ELEVENLABS_API_KEY)
  journaley/             → Journaley PWA + sync API + Postgres (3 containers, 2 on proxy)
  kenn-dein-hamburg/     → Hamburg OSM quiz
  marapoop/              → Marapoop game
  rps/                   → Rock Paper Scissors
  sababa/                → Sababa Telegram bot + local whisper voice-to-text sidecar (no subdomain)
  wiki-quiz/             → Wikipedia trivia
  Each contains: deploy.yml, docker-compose.yml.j2, env.j2, vars.yml (encrypted)
  journaley/ also has: backup.sh.j2, backup.yml (nightly backup cron)
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

## Auto-redeploy (Watchtower)

Containers can auto-update when a new image is pushed to the registry — no manual deploy needed.

- **How it works:** the registry fires a push notification → Watchtower (`/v1/update`, bearer-token gated) pulls the new image and recreates the container. Event-driven, no polling.
- **Opt-in per container:** add the label `com.centurylinklabs.watchtower.enable=true` to the service in its `docker-compose.yml.j2`. Watchtower runs with `WATCHTOWER_LABEL_ENABLE=true`, so it *only* touches labeled containers — the games and proxy are never affected unless explicitly labeled.
- **Shared secret:** `watchtower_http_token` in `inventory/vault.yml` (the registry sends it as `Authorization: Bearer`; Watchtower validates it). Watchtower is on the `proxy` network only, never exposed to the internet.
- **Registry pull auth:** Watchtower reads root's `/root/.docker/config.json` to pull private images.
- **Deploy/update:** `ansible-playbook playbooks/watchtower.yml` (also runs as part of `setup-server.yml`).
- **First deploy of a labeled app is still manual** (`ansible-playbook games/<name>/deploy.yml`); subsequent pushes auto-redeploy.

Used by **sababa** (Telegram bot): collaborator pushes `registry.spielstube.app/sababa:latest` → bot auto-updates ~immediately.

**Sababa whisper sidecar:** `games/sababa/docker-compose.yml.j2` also runs a `whisper` service (faster-whisper, `small`/int8, CPU-capped at 2 cores / 2 GB) for local voice-to-text. Internal only — the bot reaches it at `http://whisper:9000` (passed in via `WHISPER_URL`). Public image, so it does not Watchtower-auto-redeploy; only the bot image does. ASR API: `POST /asr?task=transcribe&language=de&output=txt` with multipart `audio_file` (handles Telegram OGG/Opus directly).

## Backups

Journaley has nightly backups to a Hetzner storage box (`u421627@u421627.your-storagebox.de`, SSH port 23).

- **What:** pg_dump + blob volume → single timestamped `.tar.gz`
- **When:** Nightly at 3:00 AM via root cron
- **Retention:** 10 days (older archives auto-deleted)
- **Remote path:** `backups/journaley/`
- **VPS SSH key:** `/root/.ssh/storagebox` (ed25519, authorized on storage box)
- **Log:** `/var/log/journaley-backup.log`
- **Deploy/update:** `ansible-playbook games/journaley/backup.yml`

The storage box is also accessible from this Mac: `ssh -i ~/.ssh/hetzner -p23 u421627@u421627.your-storagebox.de`

## Workflow

- **No auto-commits.** Only commit when explicitly asked.
- Vault password is in `.vault_pass` (gitignored), referenced by `ansible.cfg`.
- Always use `--platform=linux/amd64` when building Docker images (server is x86).

## Global Deploy Skill

There is a global Claude Code skill at `~/.claude/commands/deploy-game.md` that documents the full deploy workflow for any Claude instance. **When making changes to the infrastructure (new games, server changes, architecture updates), always keep that skill file updated too.**
