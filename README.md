# robert-teah-production-notes

Real artifacts from production infrastructure work — not tutorials, not demos. Each folder is a config or pipeline I've actually run, with a README explaining the operational context and the incident or constraint that shaped it, not just the code.

I'm Robert Teah, a Senior DevOps / Platform / SRE Engineer with 10+ years operating production infrastructure at Luxottica, Home Depot, and FutureMedia, now extending that into AI infrastructure (AutoSRE, Reliai, WebIntell). More at [robertteah.com](https://robertteah.com).

## What's here

- **[terraform-state-backend](./terraform-state-backend)** — S3 + DynamoDB Terraform backend with state locking, and the state-corruption incident that made locking non-negotiable.
- **[github-actions-oidc-deploy](./github-actions-oidc-deploy)** — GitHub Actions authenticating to AWS via OIDC, no stored credentials, no rotation overhead.
- **[paperclip-vps-deploy](./paperclip-vps-deploy)** — The CI/CD pipeline and production hardening behind deploying [Paperclip](https://github.com/robertteah/paperclip) and OpenClaw to a single VPS, currently running my own startups: GHCR image builds, loopback-only containers, and a hardened Tailscale HTTPS broker instead of an open port.

## Why this repo exists

Most DevOps content online is tutorials and certification badges. This is the other kind: what I actually built, what broke first, and what I changed because of it.
