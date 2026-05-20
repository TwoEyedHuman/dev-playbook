# Lessons Learned

A living document. Append after every project or epic. Periodically rasterize into skeleton READMEs.

---

## Format

Each entry has:
- **Project/Context** — what you were building
- **Lesson** — what went wrong or what worked well
- **Pattern** — the durable rule to bake into future skeletons

---

## 2026-05 — Personal Portfolio Site (Next.js + Go + Docker + Fly.io)

---

### Lesson 1: `frontend/public/` must exist for Docker multi-stage builds

**Context:** Next.js frontend Dockerfile copies `/app/public` in the runner stage. If the directory doesn't exist in the repo, the build fails with a checksum error even though `npm run build` succeeds.

**Pattern:** Story 1.1 (repo scaffolding) must always include:
```bash
mkdir -p frontend/public
touch frontend/public/.gitkeep
```
And the Docker multi-stage build AC should be part of the scaffolding story, not deferred to later.

---

### Lesson 2: AC must test the integrated stack, not just components in isolation

**Context:** Story AC like "npm run build succeeds" and "go build ./... succeeds" both passed, but `docker compose up` failed because the two had never been run together. Bugs only surfaced during manual integration testing.

**Pattern:** Every epic needs an **integration gate** at the end — a checklist of `curl` and browser checks that verify all services in that epic work together before moving to the next epic. Stories test components; epics test integration.

---

### Lesson 3: Volume mounts and file paths need explicit AC

**Context:** `projects.yaml` was mounted correctly in `docker-compose.yml` but the Go binary's working directory didn't match the mount path, causing a startup crash. Neither the story that introduced `projects.yaml` nor the story that wrote the Dockerfile explicitly verified the two were consistent.

**Pattern:** Any story that introduces a config file read at runtime must include AC: *"service starts successfully via `docker compose up` with the config file in place."* Stories should also explicitly state path assumptions so the next Claude session doesn't have to guess.

---

### Lesson 4: Explicit "assumptions" block in each story

**Context:** Several bugs were caused by implicit dependencies between stories — one story assumed a directory existed, another assumed a mount path, neither stated it. Each Claude session only sees its own story and has no memory of prior sessions.

**Pattern:** Every story should have an **Assumptions** section listing what must already be true for the story to succeed. Example:
```
Assumptions:
- docker-compose.yml exists with caddy, frontend, api, streamlit-app services
- projects.yaml is mounted into the api container at /app/projects.yaml
- frontend/public/ directory exists
```

---

### Lesson 5: Docker DNS can silently fail on first build

**Context:** First `docker compose build` failed with DNS lookup errors for `registry-1.docker.io`. Restarting Docker daemon fixed it. No code change needed.

**Pattern:** Add a "pre-flight" checklist to the repo root or Makefile:
```bash
make preflight  # checks docker is running, DNS works, ports are free
```
Or at minimum document in the README: *"If build fails with DNS errors, restart Docker daemon: `sudo systemctl restart docker`"*

---

### Lesson 6: Design for extensibility from story 1

**Context:** The project registry (`projects.yaml`) was designed so adding a new project requires zero code changes — just a new entry in YAML and a new Docker service. This paid off immediately when thinking about future projects.

**Pattern:** When the README says "design for extensibility," make it concrete in Story 1: define the extension mechanism (config file, plugin interface, etc.) and write a stub second entry so the pattern is proven before it's needed.

---

### Lesson 7: Keep stories scoped to one Claude session

**Context:** Stories that were too broad caused Claude to make implicit decisions that conflicted with other stories. Stories that were tight and had clear inputs/outputs were implemented cleanly.

**Pattern:** A well-scoped story fits this template:
- **Context:** 1-2 sentences on what already exists
- **Tasks:** bulleted, imperative, specific
- **Assumptions:** explicit prerequisites
- **Acceptance Criteria:** testable, includes at least one `docker compose` or `curl` check
- **Out of scope:** anything that might tempt scope creep

---

### Lesson 8: "Rasterize" sessions are valuable forcing functions

**Context:** Lessons accumulate as ad-hoc notes. Without a deliberate step to fold them back into the skeleton, each new project starts from the same flawed baseline.

**Pattern:** After each project (or at minimum after each epic), append to this file. Every 2-3 projects, open a Claude chat session with this file + the current skeleton and say: *"Regenerate the skeleton incorporating these lessons."* Commit the result.
