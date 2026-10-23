# [Project Name]

> One-line description of what this is.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Repository Structure](#repository-structure)
3. [Technology Stack](#technology-stack)
4. [Environment Strategy](#environment-strategy)
5. [Pre-Flight Checklist](#pre-flight-checklist)
6. [Implementation Stories](#implementation-stories)
7. [Secrets & Config Management](#secrets--config-management)
8. [Definition of Done](#definition-of-done)

---

## Architecture Overview

```
[ Replace with ASCII diagram ]
```

### Key Design Decisions

- **[Decision 1]:** reason
- **[Decision 2]:** reason
- **Extensibility mechanism:** describe how new [features/services/modules] are added without code changes (e.g. config file, plugin interface)

---

## Repository Structure

```
project/
├── README.md
├── docker-compose.yml
├── docker-compose.prod.yml
├── Caddyfile
├── Caddyfile.prod
├── Makefile
├── .env.example               ← committed; .env is gitignored
│
├── frontend/                  ← Next.js
│   ├── Dockerfile
│   ├── public/                ← MUST EXIST for Docker multi-stage build
│   │   └── .gitkeep
│   └── ...
│
├── api/                       ← Go
│   ├── Dockerfile
│   └── ...
│
└── [other services]/
    ├── Dockerfile
    └── ...
```

---

## Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| Frontend | Next.js 14 (App Router) | |
| Styling | Tailwind CSS | |
| Backend | Go 1.22 + chi | |
| Proxy | Caddy 2 | Automatic HTTPS |
| Containers | Docker + Compose | Consistent across envs |
| Hosting | Fly.io | Docker-native, predictable billing |
| DNS | Cloudflare | Free proxy + cheap registrar |
| CI/CD | GitHub Actions | |

---

## Environment Strategy

| | Local | Dev Server | Production |
|---|---|---|---|
| Domain | `localhost` | `dev.yourdomain.com` | `yourdomain.com` |
| TLS | none | Caddy + Let's Encrypt | Caddy + Let's Encrypt |
| Secrets | `.env` file | `.env` file | Fly.io secrets |
| Deploy | `make dev` | `git pull && make dev` | `fly deploy` (GitHub Actions) |

---

## Pre-Flight Checklist

Run before first `docker compose build` and after any environment change:

```bash
# Docker is running
docker info > /dev/null && echo "✓ Docker running" || echo "✗ Docker not running"

# DNS works from Docker (if this fails, restart Docker daemon)
docker run --rm alpine nslookup registry-1.docker.io && echo "✓ Docker DNS ok"

# Required ports are free
lsof -i :80 -i :443 -i :3000 -i :8080 | grep LISTEN && echo "⚠ ports in use" || echo "✓ ports free"

# .env exists
test -f .env && echo "✓ .env found" || echo "✗ copy .env.example to .env"
```

If Docker DNS fails: `sudo systemctl restart docker` then re-run.

> **Ctrl+C not responding?** If `docker compose up` hangs on stop, your services are missing `stop_grace_period`. All services in `docker-compose.yml` should have `stop_grace_period: 5s`. Without it, Docker waits the full 10-second default before force-killing, and a stuck container can make the terminal unresponsive.

---

## Implementation Stories

### Story Template

Each story is one Claude CLI session. Keep them tight.

```
#### Story X.Y — Title

**Context:** What already exists. What this story builds on. (2-3 sentences max)

**Assumptions:**
- List explicit prerequisites — files, env vars, mounted volumes, running services
- If an assumption is wrong, the story will fail; fix the assumption first

**Tasks:**
- Imperative, specific, one action per bullet
- Include file paths
- Call out any SOLID principles or patterns to follow

**Out of Scope:**
- Anything that might tempt scope creep

**Acceptance Criteria:**
- [ ] Component-level: unit tests pass, binary builds, etc.
- [ ] Integration: `docker compose up` — all containers stay running
- [ ] At least one `curl` or browser check against the running stack
- [ ] No secrets committed; `.env` pattern followed
```
