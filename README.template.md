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

---

### EPIC 1 — Repo Scaffolding & Docker Foundation

**Epic Goal:** `docker compose up` brings up all containers with placeholder responses. No service crashes on startup.

---

#### Story 1.1 — Initialize Repository Structure

**Context:** Starting from scratch.

**Assumptions:**
- Docker and Docker Compose are installed
- Go and Node are installed locally for validation

**Tasks:**
- Create directory structure per [Repository Structure](#repository-structure)
- `frontend/public/.gitkeep` — required for Docker multi-stage build
- `go.mod` in `api/` (`module github.com/<you>/[project]`)
- `package.json` in `frontend/` — Next.js 14, Tailwind, TypeScript
- `.gitignore` — `node_modules/`, `.env`, `*.env.local`, Go binaries, `.next/`
- `docker-compose.yml` — all services with placeholder healthchecks
- `.env.example` with all required vars (no values); `.env` gitignored
- `Makefile` targets: `dev`, `build`, `down`, `logs`, `preflight`

**Out of Scope:** Any application logic, routing, actual service implementation.

**Acceptance Criteria:**
- [ ] `make preflight` passes (Docker running, DNS works, ports free)
- [ ] `docker compose build` completes without error
- [ ] `docker compose up` — all containers start and stay running (no exits)
- [ ] `go build ./...` succeeds in `api/`
- [ ] `npm run build` succeeds in `frontend/`
- [ ] `git status` shows no `.env` file tracked

---

#### Story 1.2 — Caddy Reverse Proxy

**Context:** Story 1.1 complete. All containers running with placeholder responses.

**Assumptions:**
- `docker-compose.yml` has a `caddy` service
- `frontend` service is on port 3000 (internal), `api` on port 8080 (internal)
- No host ports exposed for frontend or api — Caddy is the only entry point

**Tasks:**
- `Caddyfile` for dev: HTTP only, routes `/api/*` → `api:8080`, `/*` → `frontend:3000`
- `Caddyfile.prod`: TLS via Let's Encrypt, same routes
- Mount correct Caddyfile per environment in `docker-compose.yml` and `docker-compose.prod.yml`
- `Makefile` documents how to switch

**Acceptance Criteria:**
- [ ] `curl http://localhost/` → proxied response from frontend container
- [ ] `curl http://localhost/api/health` → proxied response from api container
- [ ] `curl http://localhost:3000` → connection refused (not exposed to host)
- [ ] `curl http://localhost:8080` → connection refused (not exposed to host)

---

### EPIC 1 Integration Gate

Before starting Epic 2, verify manually:

- [ ] `docker compose up` — zero container exits after 30 seconds
- [ ] `curl http://localhost/` → 200
- [ ] `curl http://localhost/api/health` → 200
- [ ] `docker compose ps` → all services "healthy" or "running"
- [ ] `docker compose down && docker compose up` → clean restart works

**If anything fails:** open a Claude session with the failing check output + `docker compose logs [service]` + the story that owns that service.

---

### EPIC 2 — Backend API

**Epic Goal:** Go API is running, serves project metadata from config, and proxies requests to sidecars.

---

#### Story 2.1 — Go Server & Health Endpoint

**Context:** `api/` has `go.mod`. No Go code yet. Caddy is proxying `/api/*` to port 8080.

**Assumptions:**
- `docker-compose.yml` api service exists and is on the internal network
- Port 8080 is not exposed to host

**Tasks:**
- `cmd/server/main.go` — HTTP server on `$PORT` (default 8080)
- `chi` router
- `GET /health` and `GET /api/health` → `{"status":"ok","version":"..."}`
- Structured JSON logging via `log/slog`
- Graceful shutdown on SIGTERM
- Multi-stage Dockerfile — final image `alpine` or `scratch`, under 20MB

**Acceptance Criteria:**
- [ ] `docker compose up api` — container stays running
- [ ] `curl http://localhost/api/health` → 200 JSON (via Caddy)
- [ ] `docker compose stop api` → graceful shutdown logged
- [ ] `docker images | grep api` → image under 20MB

---

#### Story 2.2 — Project Registry & Config

**Context:** Story 2.1 complete. Health endpoint works.

**Assumptions:**
- `projects.yaml` exists at repo root
- `docker-compose.yml` mounts `./projects.yaml:/app/projects.yaml:ro` on the api service
- Go binary's working directory is `/app`

**Tasks:**
- `internal/config/config.go` — loads `projects.yaml` from `$PROJECTS_YAML` (default `/app/projects.yaml`)
- Validate required fields on startup; crash with clear error if invalid
- `internal/projects/registry.go` — `List()` and `Find(slug)` methods
- `GET /api/projects` → JSON array
- `GET /api/projects/:slug` → single project or 404
- Unit tests (table-driven)
- `projects.yaml` with one real entry + one stub

**Out of Scope:** Proxy handler (Story 2.3)

**Acceptance Criteria:**
- [ ] `docker compose up api` — container starts (does not exit with config error)
- [ ] `curl http://localhost/api/projects` → JSON array with at least one entry
- [ ] `curl http://localhost/api/projects/[real-slug]` → 200
- [ ] `curl http://localhost/api/projects/nonexistent` → 404
- [ ] Remove `projects.yaml` mount, restart → container exits with clear error message
- [ ] `go test ./internal/projects/...` → pass

---

#### Story 2.3 — Reverse Proxy Handler

**Context:** Stories 2.1 and 2.2 complete. Registry works.

**Assumptions:**
- Sidecar containers are on the same Docker network as the api container
- Sidecar hostnames match `internal_host` values in `projects.yaml`

**Tasks:**
- `internal/proxy/handler.go` — `GET /api/projects/:slug/proxy` proxies to sidecar
- Strip prefix before forwarding
- WebSocket upgrade support
- `X-Forwarded-For` and `X-Forwarded-Proto` headers
- 404 if slug not in registry; 502 with JSON body if upstream unreachable
- Per-request logging: slug, upstream latency, status code
- Integration test using `httptest` with mock upstream

**Acceptance Criteria:**
- [ ] `curl http://localhost/api/projects/[slug]/proxy/` → response from sidecar
- [ ] Stop the sidecar container → proxy returns 502 JSON (not 500 or timeout)
- [ ] Logs show slug + latency for each proxied request

---

### EPIC 2 Integration Gate

- [ ] `curl http://localhost/api/health` → 200
- [ ] `curl http://localhost/api/projects` → JSON array
- [ ] `curl http://localhost/api/projects/[slug]` → project metadata
- [ ] `curl http://localhost/api/projects/[slug]/proxy/` → sidecar response
- [ ] Kill sidecar → proxy returns 502, not a hang
- [ ] `docker compose restart api` → comes back up, registry reloads

---

### EPIC 3 — Frontend

**Epic Goal:** Next.js app renders homepage, projects listing, and project embed pages. All data is live from the API.

---

#### Story 3.1 — App Shell & Navigation

**Context:** Frontend container is running with default Next.js page.

**Assumptions:**
- `frontend/public/` directory exists (required for Docker build)
- Caddy is proxying `/*` to frontend

**Tasks:**
- `app/layout.tsx` — root shell, Tailwind globals, `<Nav />`
- `Nav.tsx` — site name left, links right ("About" → `/`, "Projects" → `/projects`), responsive hamburger
- Dark mode via `prefers-color-scheme`
- `app/not-found.tsx` — friendly 404

**Acceptance Criteria:**
- [ ] `docker compose build frontend` → succeeds (no missing `/app/public` error)
- [ ] `http://localhost/` → nav renders, no console errors
- [ ] Mobile viewport (375px) → hamburger visible, opens/closes
- [ ] Dark mode → nav styled correctly
- [ ] Lighthouse accessibility ≥ 90

---

#### Story 3.2 — Homepage / About Me

**Context:** Story 3.1 complete. Nav and layout exist.

**Assumptions:**
- `GET /api/projects` is reachable from the Next.js server (internal Docker network)
- `NEXT_PUBLIC_API_URL` or internal API URL is set in environment

**Tasks:**
- `app/page.tsx` — server component, no `"use client"`
- Sections: Hero, About, Featured Projects (fetched from API), Contact/Links
- `components/ProjectCard.tsx` — server component
- Next.js `fetch()` with revalidation for project data

**Acceptance Criteria:**
- [ ] `http://localhost/` → page renders with project cards from live API
- [ ] Disable JS in browser → page still renders (server component)
- [ ] `docker compose stop api` → homepage shows graceful empty state, not a crash
- [ ] Lighthouse performance ≥ 90

---

#### Story 3.3 — Projects Listing Page

**Context:** Story 3.2 complete.

**Assumptions:**
- `/api/projects` returns a `tags` array per project

**Tasks:**
- `app/projects/page.tsx` — server component, fetches all projects
- `components/TagFilter.tsx` — client component island for filtering
- `<Suspense>` + skeleton cards for loading state

**Acceptance Criteria:**
- [ ] `http://localhost/projects` → all projects render
- [ ] Tag filter works without full page reload
- [ ] Skeleton cards visible during fetch (add artificial delay to verify)
- [ ] JS disabled → full unfiltered list renders

---

#### Story 3.4 — Project Embed Page

**Context:** Stories 3.3 and 2.3 complete.

**Assumptions:**
- Go proxy is working for all registered slugs
- `embed_type: iframe` is the default

**Tasks:**
- `app/projects/[slug]/page.tsx` — fetches metadata, renders embed
- `components/ProjectEmbed.tsx` — client component, iframe with loading spinner and "open in new tab"
- `generateStaticParams` for known slugs
- Error state when proxy is unreachable

**Acceptance Criteria:**
- [ ] `http://localhost/projects/[slug]` → sidecar app visible in iframe
- [ ] Loading spinner shows until iframe content loads
- [ ] "Open in new tab" → opens sidecar directly
- [ ] Stop sidecar → embed shows error state, not blank iframe
- [ ] Unknown slug → Next.js 404 page

---

### EPIC 3 Integration Gate

- [ ] Full user journey: `http://localhost/` → click project card → embed loads
- [ ] `http://localhost/projects` → tag filter works
- [ ] `http://localhost/projects/nonexistent` → 404 page
- [ ] `docker compose restart frontend` → site back in under 10s
- [ ] No console errors on any page

---

### EPIC 4 — Project Sidecars

**Epic Goal:** Each project runs in an isolated container, accessible only via the Go proxy.

---

#### Story 4.1 — [Project Name] Sidecar

**Context:** Docker Compose is running. Go proxy expects sidecar at `[internal_host]:[internal_port]`.

**Assumptions:**
- `projects.yaml` has an entry for this project with correct `internal_host` and `internal_port`
- No host port mapping — internal network only

**Tasks:**
- `projects/[name]/Dockerfile`
- Add service to `docker-compose.yml` — no host ports, healthcheck, `restart: unless-stopped`
- Verify `projects.yaml` entry matches service name and port

**Acceptance Criteria:**
- [ ] `docker compose up [service]` → container stays running
- [ ] `curl http://localhost/api/projects/[slug]/proxy/` → response from sidecar
- [ ] Kill container → `docker compose up [service]` restarts it automatically
- [ ] `docker compose ps` → no host port exposed for this service

---

#### Story 4.2 — Sidecar Template & Developer Guide

**Context:** At least one sidecar working.

**Tasks:**
- `projects/_template/` with commented Dockerfile skeletons (Python, Go, Node)
- `projects/_template/README.md` — checklist for wiring up a new sidecar
- `make new-project NAME=x` — copies template
- Update root README "Adding New Projects" section

**Acceptance Criteria:**
- [ ] `make new-project NAME=test` → creates `projects/test/` with template files
- [ ] Following the checklist produces a working hello-world embed

---

### EPIC 4 Integration Gate

- [ ] All sidecars in `projects.yaml` are running
- [ ] Each sidecar accessible via `http://localhost/api/projects/[slug]/proxy/`
- [ ] Each sidecar renders in its embed page
- [ ] `docker compose down && docker compose up` → all sidecars come back

---

### EPIC 5 — Deployment & CI/CD

**Epic Goal:** Pushing to `main` deploys to Fly.io automatically. Pi dev server runs the same stack.

---

#### Story 5.1 — Fly.io Production Config

**Context:** App works end-to-end locally.

**Tasks:**
- `fly launch` + configure `fly.toml`
- `docker-compose.prod.yml` overrides
- `Caddyfile.prod` with TLS
- Set all secrets via `fly secrets set`
- Document rollback: `fly releases list` + `fly deploy --image <old>`

**Acceptance Criteria:**
- [ ] `fly deploy` succeeds
- [ ] `https://yourdomain.com/` → homepage loads
- [ ] `https://yourdomain.com/api/health` → 200
- [ ] All project embeds work in production
- [ ] Fly.io dashboard shows monthly estimate ≤ $6

---

#### Story 5.2 — GitHub Actions CI/CD

**Context:** Story 5.1 complete.

**Tasks:**
- `.github/workflows/deploy.yml` — test job + deploy job (depends on test)
- `.github/workflows/pr-check.yml` — test only on PRs
- Go test cache, npm cache, Docker layer cache
- `FLY_API_TOKEN` in GitHub secrets

**Acceptance Criteria:**
- [ ] Push to `main` → deploy completes within 5 minutes
- [ ] Failing test → deploy job blocked
- [ ] PR → test job runs, no deploy

---

#### Story 5.3 — Dev Server (Raspberry Pi)

**Context:** Pi has Docker installed. Repo cloned.

**Tasks:**
- `scripts/pi-setup.sh`
- `systemd/portfolio.service` — start on boot, stop cleanly
- `Caddyfile.prod` works for `dev.yourdomain.com`
- Cloudflare DNS A record for Pi's external IP

**Acceptance Criteria:**
- [ ] `sudo systemctl start portfolio` → all containers up
- [ ] `https://dev.yourdomain.com/` → accessible from outside LAN
- [ ] Reboot Pi → stack auto-starts
- [ ] TLS certificate valid

---

### EPIC 5 Integration Gate

- [ ] Full deploy from `git push` → live on `https://yourdomain.com` within 5 minutes
- [ ] Failing test blocks deploy (verify by breaking a test temporarily)
- [ ] Pi auto-starts after reboot
- [ ] Rollback procedure tested: deploy old image, verify, re-deploy current

---

### EPIC 6 — Content & Polish

#### Story 6.1 — Real Content Pass

**Context:** All infrastructure complete. Site runs with placeholder content.

**Tasks:**
- Real bio, tech stack, social links in `app/page.tsx`
- Project thumbnails in `frontend/public/thumbnails/`
- Favicon + Open Graph image in `frontend/public/`
- Real `<head>` metadata

**Acceptance Criteria:**
- [ ] No "Lorem ipsum" or "placeholder" text on any page
- [ ] Open Graph preview correct when URL shared on Slack/Twitter
- [ ] All external links resolve and open in new tab
- [ ] Favicon visible in browser tab

---

## Secrets & Config Management

| Secret | Local | Dev Server | Production |
|---|---|---|---|
| `NEXT_PUBLIC_API_URL` | `.env` | `.env` | `fly secrets set` |
| `GO_ENV` | `.env` | `.env` | `fly secrets set` |
| `FLY_API_TOKEN` | not needed | not needed | GitHub Actions secret |

Never commit `.env`. The `.env.example` file is committed with keys but no values.

---

## Definition of Done

A story is complete when:

- [ ] All acceptance criteria pass
- [ ] `docker compose up` still works cleanly after the change
- [ ] New non-trivial logic has unit tests
- [ ] No secrets committed
- [ ] README updated if setup/config steps changed
- [ ] Epic integration gate passes before moving to next epic
