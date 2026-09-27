# Paperclip + OpenClaw VPS Deploy

The production deployment I built for [Paperclip](https://github.com/robertteah/paperclip) (a fork of [paperclipai/paperclip](https://github.com/paperclipai/paperclip)) and OpenClaw, currently running my own startups on a single VPS.

I didn't build Paperclip or OpenClaw — I built the CI/CD and the production hardening around them: how the images get built, how they get exposed to the internet without opening a port, and how a deploy actually happens.

## Architecture

```
GitHub push (production branch)
        │
        ▼
GitHub Actions ──build──▶ ghcr.io/robertteah/paperclip:latest
        │
        ▼ (manual SSH, two commands — see "Deploy")
VPS: docker compose pull && up -d
        │
   ┌────┴─────────────────────────────┐
   │                                   │
   ▼                                   ▼
paperclip (127.0.0.1:3100)      openclaw-gateway (127.0.0.1:18789)
   │                                   │
   ▼                                   │
postgres:17 (healthcheck-gated)        │
                                        │
        Tailscale HTTPS broker (systemd, unprivileged user)
                        │
                 public HTTPS access
```

Both application containers bind to `127.0.0.1` only — neither is reachable from the public internet directly. Public access goes through a Tailscale-based HTTPS broker I run as its own hardened systemd service, not through an open port on the VPS's public interface.

## CI: build and publish

[`.github/workflows/docker-publish.yml`](./.github/workflows/docker-publish.yml) builds the Paperclip image on every push to `production` and pushes it to my own GHCR namespace. Documenting this pipeline is actually what surfaced two live bugs in it: the branch trigger had a corrupted string (a stray newline had gotten baked into `"production"`, so the workflow was silently not firing on push), and the build context pointed at a `./paperclip` subdirectory that doesn't exist in this repo (the repo root *is* the Paperclip app). Both are fixed here. There was also an `openclaw` build step pointed at a `./openclaw` context that has never existed in this repo — OpenClaw is a separate codebase and needs its own publish workflow in its own repo, so I removed that step rather than leave a build step that can't succeed.

Writing the runbook found the bugs the runbook was supposed to document. That's the honest version of this artifact.

## CD: deploy

Deploy is deliberately a manual two-command SSH step, not a fully automated push-to-prod:

```bash
docker compose -f /opt/paperclip/docker-compose.production.yml --env-file /opt/paperclip/.env pull
docker compose -f /opt/paperclip/docker-compose.production.yml --env-file /opt/paperclip/.env up -d
```

`pull` fetches the new image from GHCR; `up -d` recreates the app containers with it. Postgres isn't touched — same container, same volume, `depends_on: db: condition: service_healthy` on the Paperclip service means the app never starts against a database that isn't ready yet. Rolling restart, no downtime on the DB.

## Why the app containers only bind to loopback

[`docker-compose.production.yml`](./docker-compose.production.yml) publishes `paperclip` and `openclaw-gateway` on `127.0.0.1:<port>` — not `0.0.0.0`. Nothing about either service is reachable from outside the VPS at the Docker layer at all. The only way in is through the Tailscale HTTPS broker.

## The broker: a narrowly-scoped systemd service, not an open port

[`systemd/paperclip-tailscale-https-broker.service`](./systemd/paperclip-tailscale-https-broker.service) runs the broker as its own dedicated unprivileged user (`paperclip-tsbroker`), with:

- `NoNewPrivileges=true` — the process can never escalate privileges after start
- `ProtectSystem=strict` + `ProtectHome=true` — the filesystem is read-only to it outside a few explicit paths
- `PrivateTmp=true` — its own isolated `/tmp`
- `ReadWritePaths` scoped to exactly the three directories it needs (runtime, state, logs) — nothing else on the box is writable by it

The tradeoff this buys: the surface a compromise of the broker process could reach is deliberately tiny — one user, one set of directories, no elevated privileges, no direct write access anywhere else on the VPS.

## Secrets

Nothing in this folder contains real secrets — `docker-compose.production.yml` and the systemd unit both reference credentials via environment variables (`${POSTGRES_PASSWORD}`, `${ANTHROPIC_API_KEY}`, etc.) and an `EnvironmentFile=` path, never inline values. The actual `.env` files stay on the VPS and out of git.
