# es-flora — Elder Scrolls Flora ID

> "Who's that Pokémon?" for Tamriel. See a plant, type its name, keep the streak alive.

Live at **https://es-flora.brandonlocke.xyz**

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
9. [Open Questions & Known Risks](#open-questions--known-risks)

---

## Architecture Overview

```
                    ┌──────────────────────────┐
                    │  Cloudflare DNS (proxy)  │
                    │ es-flora.brandonlocke.xyz│
                    └────────────┬─────────────┘
                                 │ HTTPS
                    ┌────────────▼─────────────┐
                    │   Fly.io app (1 machine) │
                    │  ┌────────────────────┐  │
                    │  │ Caddy 2 file_server│  │
                    │  │  serving /dist     │  │
                    │  └────────────────────┘  │
                    └──────────────────────────┘
                                 ▲
                                 │ fly deploy (GitHub Actions on push to main)
                    ┌────────────┴─────────────┐
                    │  Multi-stage Dockerfile  │
                    │  node:22 → vite build    │
                    │  → caddy:alpine + /dist  │
                    └──────────────────────────┘

  Browser runtime (no network calls after load):

   plants.json ──► PlantRepository ──► GameEngine (pure TS) ──► React UI
   assets/*.svg ──► AssetResolver          │                       │
                                            ├─ RoundGenerator      ├─ TitleScreen
                                            ├─ Scoring/Streak      ├─ PlayScreen
                                            └─ Timer               └─ ResultsScreen
                                                                    │
                                                      localStorage ─┘ (best scores only)
```

### Key Design Decisions

- **No backend.** The entire game is data + logic that fits in a bundle. A server would only exist to hold a leaderboard, which is out of scope. Static files behind Caddy on Fly.io — no DB, no secrets, no API surface to defend.
- **Game engine is framework-free TypeScript.** `src/engine/` imports nothing from React. Round generation, scoring, streaks, and the timer are pure functions and a reducer, unit-tested with Vitest without a DOM. React is a rendering layer over engine state.
- **Answer input is a constrained typeahead.** The player types, matching plant names appear, and only a *committed selection* counts. This removes spelling penalties and fuzzy-match tuning entirely — there is no "was `nirnrut` close enough?" decision to get wrong.
- **Art is placeholder-first.** Real illustration is a project of its own and every Bethesda-sourced image is a licensing problem. Stories 2.3–2.4 ship a deterministic procedural SVG generator so every plant has a distinct, stable image on day one. Epic 9 swaps in real art with zero engine changes.
- **A plant is a species, not a game's ingredient.** Nirnroot is Nirnroot whether the player is thinking of Oblivion or Skyrim. Each entry has one canonical `name`, and a `games` array recording which titles it appears in. This also means the canonical name is the *plant*, not the harvested part: Oblivion's `Lavender Sprig`, `Fly Amanita Cap`, and `Foxglove Nectar` are the plants `Lavender`, `Fly Amanita`, and `Foxglove`. The in-game ingredient names are kept as search aliases, so typing "lavender sprig" still finds it.
- **Extensibility mechanism:** adding a plant means adding one object to `src/data/plants.json` and (optionally) dropping `<id>.svg` into `public/assets/plants/`. No code change. The `games` array exists from the first commit, so adding a title is a data change: an existing plant gains an entry in its array, a new one gets a new object. `AssetResolver` falls back to the procedural generator when a file is missing, so art can land plant-by-plant.
- **Walking skeleton first.** Epic 1 ends with a deployed, HTTPS-serving "hello" page at the real domain. Every later epic ships onto working infrastructure instead of discovering deployment problems at the end.

---

## Repository Structure

```
oblivion-flora-id/
├── README.md
├── README.template.md
├── Dockerfile                     ← multi-stage: vite build → caddy:alpine
├── Caddyfile                      ← prod: file_server, SPA fallback, cache headers
├── fly.toml
├── docker-compose.yml             ← local verification of the *production image*
├── Makefile                       ← dev / build / test / image / deploy
├── .dockerignore
├── .gitignore
├── .env.example                   ← committed; .env is gitignored
├── index.html
├── package.json
├── vite.config.ts
├── vitest.config.ts
├── tailwind.config.ts
├── tsconfig.json
│
├── .github/workflows/
│   ├── ci.yml                     ← typecheck, lint, unit tests, build
│   └── deploy.yml                 ← fly deploy on push to main
│
├── public/
│   ├── .gitkeep
│   └── assets/plants/             ← <plant-id>.svg, added over time; optional per plant
│
├── scripts/
│   ├── generate-placeholders.ts   ← writes procedural SVGs for every plant id
│   └── validate-plants.ts         ← schema + uniqueness + asset-presence check
│
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   │
│   ├── data/
│   │   ├── plants.json            ← the entire content of the game
│   │   └── plants.schema.json
│   │
│   ├── engine/                    ← ZERO React imports
│   │   ├── types.ts
│   │   ├── rng.ts                 ← seeded PRNG, deterministic for tests
│   │   ├── roundGenerator.ts
│   │   ├── scoring.ts
│   │   ├── gameReducer.ts
│   │   ├── timer.ts
│   │   └── __tests__/
│   │
│   ├── search/
│   │   ├── plantIndex.ts          ← normalize, prefix + substring match, ranking
│   │   └── __tests__/
│   │
│   ├── repositories/
│   │   ├── PlantRepository.ts
│   │   └── AssetResolver.ts
│   │
│   ├── storage/
│   │   └── localScores.ts         ← versioned, corruption-tolerant
│   │
│   ├── components/
│   │   ├── theme/                 ← Panel, ScrollFrame, Button, Divider, Heading
│   │   ├── PlantCard.tsx
│   │   ├── AnswerInput.tsx        ← the typeahead
│   │   ├── SuggestionList.tsx
│   │   ├── StreakMeter.tsx
│   │   ├── TimerBar.tsx
│   │   └── FeedbackFlash.tsx
│   │
│   ├── screens/
│   │   ├── TitleScreen.tsx
│   │   ├── PlayScreen.tsx
│   │   └── ResultsScreen.tsx
│   │
│   └── styles/
│       ├── tokens.css             ← Oblivion palette + type scale as CSS vars
│       └── index.css
│
└── e2e/
    └── play.spec.ts               ← Playwright: one full endless run
```

---

## Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| Build | Vite 5 + TypeScript (strict) | One-screen SPA; no SSR, no routing ceremony, sub-second HMR |
| UI | React 18 | Familiar component model; engine stays framework-free regardless |
| Styling | Tailwind CSS + CSS custom properties | Utilities for layout, tokens for the Oblivion palette so theming is one file |
| Game logic | Plain TypeScript modules | Unit-testable without a DOM or renderer |
| Unit tests | Vitest | Native Vite integration, no separate build config |
| E2E | Playwright | One smoke spec against the built bundle; also the post-deploy check |
| Lint/format | ESLint + Prettier | |
| Web server | Caddy 2 (`file_server`) | Static serving + SPA fallback; matches the rest of the fleet |
| Containers | Docker (multi-stage) | Build node, ship alpine; final image is bytes + Caddy |
| Hosting | Fly.io | Consistent with existing sites; predictable billing |
| DNS | Cloudflare | Registrar already there; `es-flora` is one CNAME |
| CI/CD | GitHub Actions | Checks on PR, deploy on main |

**Explicitly not used:** Next.js (nothing to server-render), any database, any auth, any state library (a reducer is enough), any icon/font package requiring a license review.

---

## Environment Strategy

| | Local | Local prod-image | Production |
|---|---|---|---|
| Domain | `localhost:5173` | `localhost:8080` | `es-flora.brandonlocke.xyz` |
| Served by | Vite dev server | Caddy in Docker | Caddy on Fly.io |
| TLS | none | none | Fly-terminated + Cloudflare |
| Secrets | none needed | none needed | `FLY_API_TOKEN` in GitHub Actions only |
| Deploy | `make dev` | `make image && docker compose up` | push to `main` → Actions → `fly deploy` |

There is no dev server tier. The app has no state a staging environment would protect, and PR previews via `fly deploy --app es-flora-preview` can be added later if it ever earns one.

---

## Pre-Flight Checklist

Run before first `make image` and after any environment change:

```bash
# Node version matches CI
node -v | grep -q 'v22' && echo "✓ node 22" || echo "✗ use node 22 (nvm use 22)"

# Docker is running
docker info > /dev/null 2>&1 && echo "✓ Docker running" || echo "✗ Docker not running"

# DNS works from Docker (if this fails, restart Docker daemon)
docker run --rm alpine nslookup registry-1.docker.io > /dev/null 2>&1 && echo "✓ Docker DNS ok" || echo "✗ Docker DNS broken"

# Required ports are free
lsof -i :5173 -i :8080 | grep -q LISTEN && echo "⚠ ports in use" || echo "✓ ports free"

# flyctl authenticated
flyctl auth whoami > /dev/null 2>&1 && echo "✓ fly authed" || echo "✗ run: flyctl auth login"

# Plant data is valid before anything is built
npm run validate:plants && echo "✓ plants.json valid"
```

If Docker DNS fails: `sudo systemctl restart docker` then re-run.

> **Ctrl+C not responding?** `docker-compose.yml` services must set `stop_grace_period: 5s`. Without it Docker waits the full 10-second default before force-killing.

---

## Implementation Stories

### Story Template

Each story is one Claude CLI session. Keep them tight.

```
#### Story X.Y — Title

**Context:** What already exists. What this story builds on. (2-3 sentences max)

**Assumptions:**
- List explicit prerequisites — files, env vars, mounted volumes, running services

**Tasks:**
- Imperative, specific, one action per bullet

**Out of Scope:**
- Anything that might tempt scope creep

**Acceptance Criteria:**
- [ ] Verifiable checks
```

---

## Epic 1 — Walking Skeleton: Toolchain, Image, Deploy

**Goal:** an empty but real page served over HTTPS at the production domain, with CI green, before any game code exists.

#### Story 1.1 — Vite + React + TypeScript + Tailwind scaffold

**Context:** Repo contains only `README.md` and `README.template.md`. This story creates the buildable frontend.

**Assumptions:**
- Node 22 and npm available locally
- No existing `package.json`

**Tasks:**
- `npm create vite@latest . -- --template react-ts`, then flatten into repo root
- Set `"strict": true`, `"noUncheckedIndexedAccess": true`, and `@/*` → `src/*` path alias in `tsconfig.json`; mirror the alias in `vite.config.ts`
- Install and configure Tailwind CSS with `src/styles/index.css`; create empty `src/styles/tokens.css` and import it first
- Install and configure ESLint (typescript-eslint, react-hooks) and Prettier; add `.editorconfig`
- Add Vitest with `vitest.config.ts` (environment `jsdom`, setup file for `@testing-library/jest-dom`)
- Write one trivial passing test at `src/engine/__tests__/smoke.test.ts` to prove the runner works
- Replace `App.tsx` with a single centered "Flora of Tamriel" heading — nothing else
- Create `public/.gitkeep` and `public/assets/plants/.gitkeep` (both must exist for the Docker build)
- Add `.gitignore` (node_modules, dist, .env, .DS_Store) and `.env.example` with an explanatory comment that no runtime env vars are required
- Add `Makefile` targets: `dev`, `build`, `test`, `lint`, `typecheck`

**Out of Scope:**
- Any game logic, plant data, styling beyond Tailwind's reset, Docker

**Acceptance Criteria:**
- [ ] `make dev` serves the heading at `localhost:5173`
- [ ] `make typecheck`, `make lint`, `make test`, `make build` all pass
- [ ] `dist/` is produced and gitignored

---

#### Story 1.2 — Production image: multi-stage Dockerfile + Caddy

**Context:** Story 1.1 produces a `dist/` directory. This story turns it into a runnable image.

**Assumptions:**
- `make build` succeeds
- `public/` exists (empty directories break the COPY layer)

**Tasks:**
- Write multi-stage `Dockerfile`: `node:22-alpine` builder (`npm ci`, `npm run build`) → `caddy:2-alpine` runtime copying only `dist/` and `Caddyfile`
- Write `Caddyfile`: listen on `:8080`, `root * /srv`, `file_server`, `try_files {path} /index.html` for SPA fallback, `encode gzip zstd`
- Set cache headers in `Caddyfile`: hashed assets under `/assets/*` get `Cache-Control: public, max-age=31536000, immutable`; `index.html` gets `no-cache`
- Add `.dockerignore` covering `node_modules`, `dist`, `.git`, `e2e`, `*.md`
- Write `docker-compose.yml` with one service building the image, mapping `8080:8080`, `stop_grace_period: 5s`
- Add `Makefile` targets `image` and `serve`

**Out of Scope:**
- Fly.io config, GitHub Actions, HTTPS (Fly terminates TLS in prod)

**Acceptance Criteria:**
- [ ] `make image` builds with no warnings about missing paths
- [ ] `docker compose up` — container stays running, `Ctrl+C` stops it within 5s
- [ ] `curl -s localhost:8080 | grep -q "Flora of Tamriel"`
- [ ] `curl -sI localhost:8080/assets/<hashed>.js | grep -q immutable`
- [ ] `curl -s localhost:8080/some/deep/route` returns `index.html`, not 404
- [ ] Final image is under 60 MB (`docker images`)

---

#### Story 1.3 — Fly.io deploy and DNS

**Context:** A working production image exists locally. This story puts it on the internet at the real hostname.

**Assumptions:**
- `flyctl auth whoami` succeeds
- Cloudflare account manages `brandonlocke.xyz`

**Tasks:**
- `fly launch --no-deploy` to create app `es-flora`; hand-edit `fly.toml`: `internal_port = 8080`, `force_https = true`, `auto_stop_machines = true`, `auto_start_machines = true`, `min_machines_running = 0`, one shared-cpu-1x/256MB machine, primary region matching your other apps
- Add an HTTP health check on `/`
- `fly deploy` from local; confirm the machine boots
- `fly certs add es-flora.brandonlocke.xyz`; add the CNAME (and any required ACME record) in Cloudflare
- Document the exact Cloudflare record values and proxy setting in a `## Deployment` section appended to this README

**Out of Scope:**
- Automated deploys (Story 1.4), any app functionality

**Acceptance Criteria:**
- [ ] `curl -sI https://es-flora.brandonlocke.xyz` returns 200 with a valid certificate
- [ ] `fly status` shows the machine healthy
- [ ] Requesting `http://` redirects to `https://`

---

#### Story 1.4 — GitHub Actions: CI and deploy

**Context:** Deploys are manual. This story automates checks and release.

**Assumptions:**
- Repo has a GitHub remote; `main` is the default branch
- A Fly deploy token exists

**Tasks:**
- Add `.github/workflows/ci.yml`: on pull request and push — checkout, setup-node 22 with npm cache, `npm ci`, then `typecheck`, `lint`, `test`, `build`
- Add `.github/workflows/deploy.yml`: on push to `main` — `superfly/flyctl-actions/setup-flyctl` then `flyctl deploy --remote-only`, using `secrets.FLY_API_TOKEN`
- Create the token with `fly tokens create deploy -x 8760h` and store it as the `FLY_API_TOKEN` repo secret
- Make `deploy.yml` depend on `ci.yml` passing (`workflow_run` or a job-level `needs` in a combined workflow — pick one and note why)
- Add a post-deploy step that curls the live URL and fails the job on non-200

**Out of Scope:**
- Preview environments, E2E in CI (added in Story 7.4)

**Acceptance Criteria:**
- [ ] A PR runs CI and shows all four checks
- [ ] A merge to `main` deploys and the post-deploy curl passes
- [ ] `FLY_API_TOKEN` appears nowhere in the repo; `git log -p | grep -i fly_api_token` finds nothing

---

## Epic 2 — Plant Data & Asset Pipeline

**Goal:** one validated entry per plant — merged across both games, named for the species — each with a distinct, stable image, before any game logic reads it.

#### Story 2.1 — Plant schema and repository

**Context:** The app has no content. This story defines the shape of the only content it will ever have.

**Assumptions:**
- Story 1.1 complete

**Tasks:**
- Define `src/engine/types.ts` → `Plant`: `id` (kebab-case, stable, unique), `name` (the canonical plant name, globally unique — this is the one string the player is asked to produce), `games` (non-empty array of `"oblivion" | "skyrim"`), `ingredientNames` (map of game → the in-game ingredient name, e.g. `{ oblivion: "Lavender Sprig", skyrim: "Lavender" }`), `aliases` (extra search-only strings — punctuation-free forms, common misspellings), `assetId` (defaults to `id`), `difficulty` (`"common" | "uncommon" | "rare"`), and `sourceUrls` (map of game → UESP page)
- The player never sees, and is never asked to distinguish, a game. `games` and `ingredientNames` exist for filtering, search aliases, and later art reference — not for display during play.
- Write `src/data/plants.schema.json` (JSON Schema draft 2020-12) matching it
- Write `scripts/validate-plants.ts`: schema-validate, assert unique `id`, assert **globally** unique `name` (a collision means two entries are the same plant and must be merged), assert every key in `ingredientNames` and `sourceUrls` appears in `games`, assert every `assetId` resolves
- Add a lint rule to the validator: fail if a `name` ends in a harvest-part suffix (`Cap`, `Caps`, `Leaves`, `Nectar`, `Seeds`, `Sprig`, `Frond`, `Root Pulp`, `Pulp`, `Branch`) unless the id is on a short explicit allowlist — this catches ingredient names leaking into the canonical field
- Wire it as `npm run validate:plants` and run it from CI
- Write `src/repositories/PlantRepository.ts`: loads and freezes the dataset, exposes `all()`, `byId()`, `inGame(game)` (entries whose `games` includes it), `count()`. It is the only module allowed to import `plants.json`

**Out of Scope:**
- Populating the dataset (2.2), images (2.3)

**Acceptance Criteria:**
- [ ] `npm run validate:plants` passes against a 3-plant fixture
- [ ] Validator exits non-zero on a duplicate `id`, a duplicate `name`, a bad `games` value, an empty `games` array, and a name ending in `Cap` (prove each with a temp fixture)
- [ ] Unit tests cover `PlantRepository` accessors, including a plant present in both games appearing in both `inGame("oblivion")` and `inGame("skyrim")`

---

#### Story 2.2 — Build the unified plant compendium

**Context:** Schema and validator exist. This story fills them with one entry per *plant*, merged across both games.

**Assumptions:**
- Story 2.1 complete; UESP is reachable

**Tasks:**
- Compile from UESP's alchemy ingredient lists for Oblivion and for Skyrim (base + Dawnguard + Dragonborn), **flora only** — plants, flowers, fungi. Exclude animal parts, food items, and quest items.
- **Normalize each in-game ingredient name to its plant name.** Strip the harvested-part suffix: `Lavender Sprig` → **Lavender**, `Fly Amanita Cap` → **Fly Amanita**, `Foxglove Nectar` → **Foxglove**, `Columbine Root Pulp` → **Columbine**, `Milk Thistle Seeds` → **Milk Thistle**, `Lady's Mantle Leaves` → **Lady's Mantle**, `St. Jahn's Wort Nectar` → **St. Jahn's Wort**, `Wisp Stalk Caps` → **Wisp Stalk**, `Bog Beacon Asco Cap` → **Bog Beacon**. Keep the original string in `ingredientNames[game]`.
- Judgment cases: keep the suffix when removing it destroys the plant's identity or produces a name no one would recognize (`Spiddal Stick`, `Grass Pod`, `Ashen Grass Pod`, `Elves Ear`, `Snowberries`). Add each such decision to the validator's allowlist with a one-line reason.
- **Merge duplicates into one entry.** A plant appearing in both games is one object with `games: ["oblivion", "skyrim"]` and two `ingredientNames`. Known merges after normalization: Nirnroot, Dragon's Tongue, Lavender, Fly Amanita, Imp Stool, Nightshade, Mandrake, Foxglove, Primrose, Thistle. Expect the merged total to land near 65, not the ~78 the two raw lists sum to.
- Oblivion seed names (pre-normalization): Nirnroot, Alkanet Flower, Aloe Vera Leaves, Bergamot Seeds, Bog Beacon Asco Cap, Cairn Bolete Cap, Clouded Funnel Cap, Columbine Root Pulp, Dragon's Tongue, Elf Cup Cap, Fly Amanita Cap, Foxglove Nectar, Ginseng, Green Stain Cup Cap, Harrada, Imp Stool Cap, Lady's Mantle Leaves, Lavender Sprig, Mandrake Root, Milk Thistle Seeds, Monkshood Root Pulp, Motherwort Sprig, Peony Seeds, Primrose Leaves, Redwort Flower, Sacred Lotus Seeds, Somnalius Frond, Spiddal Stick, St. Jahn's Wort Nectar, Steel-Blue Entoloma Cap, Stinkhorn Cap, Summer Bolete Cap, Tiger Lily Nectar, Viper's Bugloss Leaves, Water Hyacinth Nectar, Wisp Stalk Caps
- Skyrim seed names: Blue Mountain Flower, Red Mountain Flower, Purple Mountain Flower, Yellow Mountain Flower, Bleeding Crown, Blisterwort, Canis Root, Creep Cluster, Deathbell, Dragon's Tongue, Elves Ear, Fly Amanita, Giant Lichen, Glowing Mushroom, Grass Pod, Hanging Moss, Imp Stool, Jazbay Grapes, Juniper Berries, Lavender, Mora Tapinella, Namira's Rot, Nightshade, Nirnroot, Crimson Nirnroot, Scaly Pholiota, Snowberries, Spiky Grass, Swamp Fungal Pod, Thistle Branch, Tundra Cotton, White Cap, Gleamblossom, Ash Creep Cluster, Trama Root, Ashen Grass Pod, Emperor Parasol Moss
- **Verify every name against UESP before committing** — the lists above are a starting point, not authoritative. Match the games' exact spelling for `ingredientNames` (apostrophes, hyphens, capitalization); the canonical `name` follows the normalization rule above.
- Populate `aliases` so any reasonable input finds the plant: every in-game ingredient name, the punctuation-free form (`st jahns wort`, `saint jahns wort`, `ladys mantle`), and singular/plural variants
- Note the plants that are *distinct species with similar names* and must NOT be merged — Nirnroot vs. Crimson Nirnroot, the four Mountain Flowers, Grass Pod vs. Ashen Grass Pod — in a comment block at the top of `validate-plants.ts`
- Assign `difficulty` by how visually distinguishable the plant is: `common` for unmistakable silhouettes (Nirnroot, Deathbell, Tundra Cotton), `rare` for the near-identical mushroom caps and the color-variant Mountain Flowers

**Out of Scope:**
- Images, difficulty tuning based on real play data, any per-game presentation of names

**Acceptance Criteria:**
- [ ] `npm run validate:plants` passes on the full dataset
- [ ] 60+ plants, every entry has a non-empty `games`, and both game values appear across the set
- [ ] No canonical `name` contains a harvest-part suffix except allowlisted entries, each with a recorded reason
- [ ] Every plant in both games has exactly one entry with two `ingredientNames` — assert Nirnroot, Lavender, and Fly Amanita specifically in a test
- [ ] Searching an in-game ingredient name (`"Lavender Sprig"`) resolves to the plant via aliases
- [ ] Every `sourceUrls` entry returns 200 (add a `--check-urls` flag to the validator, off by default)

---

#### Story 2.3 — Procedural placeholder SVG generator

**Context:** There is no art and there will not be for a while. Every plant still needs a distinct, recognizable-as-itself image so the game is playable and testable now.

**Assumptions:**
- Story 2.2 complete

**Tasks:**
- Write `scripts/generate-placeholders.ts`: for each plant, seed a PRNG from a hash of `plant.id` and emit `public/assets/plants/<assetId>.svg`
- Compose each SVG from a small parameterized vocabulary: stem count and curvature, leaf shape (lance / round / frond / none), a top form (bloom / cap / cluster / berry / none), and a two-color palette drawn from the Oblivion token set. Same id always produces the identical file.
- Bias the parameters by plant name keywords so images are thematically plausible, not random: names containing "Cap", "Bolete", "Amanita", "Stool" get a mushroom form; "Flower", "Bloom", "Lotus" get a bloom; "Root" gets a root form; "Seeds"/"Berries" get a cluster
- Keep every SVG under 4 KB, `viewBox="0 0 200 200"`, no external references, no embedded raster
- Mark generated files with an `<!-- generated: do not edit -->` comment and add a `npm run generate:placeholders` script
- Commit the generated SVGs (they are deterministic and small; committing them keeps the build hermetic)

**Out of Scope:**
- Real illustration (Epic 9), animation, per-plant hand-tuning

**Acceptance Criteria:**
- [ ] One SVG exists per plant; `npm run validate:plants` reports zero missing assets
- [ ] Re-running the generator produces zero `git diff`
- [ ] A contact sheet page (`scripts/` output or a temporary route) shows all plants at once and no two are visually identical
- [ ] Total `public/assets/plants/` size under 400 KB

---

#### Story 2.4 — AssetResolver with graceful fallback

**Context:** Placeholders exist. Real art will arrive one plant at a time and must not require a code change.

**Assumptions:**
- Stories 2.1–2.3 complete

**Tasks:**
- Write `src/repositories/AssetResolver.ts`: resolve `plant → asset URL`, preferring `public/assets/plants/<assetId>.png|.webp` if present, else `<assetId>.svg`, else a generic "unknown flora" SVG
- Use Vite's `import.meta.glob` (eager, `as: 'url'`) so resolution happens at build time and missing files are caught at build, not runtime
- Add an `onError` path in the consuming component so a broken asset renders the generic fallback rather than a broken-image icon
- Unit-test resolution order with a mocked glob map

**Out of Scope:**
- Image optimization pipeline, responsive `srcset` (Story 7.3)

**Acceptance Criteria:**
- [ ] Tests cover all four resolution branches
- [ ] Dropping a `.webp` next to an existing `.svg` changes the resolved URL with no code edit
- [ ] `make build` fails loudly if the generic fallback asset is missing

---

## Epic 3 — Game Engine

**Goal:** the complete rules of the game as pure TypeScript, fully tested, with no UI.

#### Story 3.1 — Seeded RNG and round generation

**Context:** The dataset exists. The game needs to pick which plant to show and in what order, reproducibly for tests.

**Assumptions:**
- Epic 2 complete

**Tasks:**
- Write `src/engine/rng.ts`: a small deterministic PRNG (mulberry32 or xorshift128) with `nextInt`, `pick`, `shuffle`. No `Math.random` anywhere in `src/engine/`.
- Write `src/engine/roundGenerator.ts` exposing `createRoundSequence(plants, { seed, gameFilter })` returning a lazily-advanced sequence; `gameFilter` of `"all"` uses everything, otherwise keeps plants whose `games` includes it (a plant in both games is eligible under either filter)
- Enforce a no-immediate-repeat rule and a "no plant repeats until at least `min(20, pool/2)` others have been shown" rule
- Weight selection by `difficulty` so early rounds skew `common` and the mix hardens as the streak grows; expose the weighting curve as a named constant

**Out of Scope:**
- Scoring, timing, React

**Acceptance Criteria:**
- [ ] Same seed produces an identical 500-plant sequence across runs
- [ ] Test asserting no plant appears twice inside the cooldown window over 1000 draws
- [ ] Test asserting the first 10 draws of a fresh sequence are majority `common`
- [ ] Test: a plant with `games: ["oblivion", "skyrim"]` is drawn under both filters
- [ ] `grep -r "Math.random" src/engine/` returns nothing

---

#### Story 3.2 — Game reducer and scoring

**Context:** Rounds can be generated. This story defines what happens when the player answers.

**Assumptions:**
- Story 3.1 complete

**Tasks:**
- Define the state machine in `src/engine/gameReducer.ts`: `idle → playing → (feedback) → playing | finished`, with actions `START`, `SUBMIT_ANSWER`, `ADVANCE`, `TICK`, `QUIT`
- Model both modes in one reducer: `endless` (a wrong answer ends the run) and `timeAttack` (60 seconds; a wrong answer costs 3 seconds and play continues)
- Write `src/engine/scoring.ts`: base points per correct answer scaled by `difficulty`, plus a streak multiplier that steps up at 5/10/20/40. Expose `describeScore()` returning the breakdown so the UI never recomputes scoring itself.
- Track per-run stats the results screen needs: correct, wrong, longest streak, per-plant misses, elapsed time
- Keep the reducer pure — no `Date.now()` inside it; time enters only via `TICK` payloads

**Out of Scope:**
- The timer's wall-clock driver (3.3), persistence, UI

**Acceptance Criteria:**
- [ ] Reducer tests cover every action in every state, including illegal transitions (which must be no-ops, not throws)
- [ ] Test: a wrong answer in `endless` moves to `finished`; the same in `timeAttack` deducts 3s and stays `playing`
- [ ] Test: a 41-correct run produces the expected total from the documented multiplier table
- [ ] Reducer file imports nothing but types and `scoring.ts`

---

#### Story 3.3 — Timer driver

**Context:** The reducer consumes `TICK`. Something must produce ticks accurately across tab backgrounding.

**Assumptions:**
- Story 3.2 complete

**Tasks:**
- Write `src/engine/timer.ts` as a class taking an injectable clock (`() => number`) and scheduler, so tests use fake timers
- Drive from `requestAnimationFrame` with wall-clock deltas rather than counting frames, so throttled background tabs do not gain the player time
- Pause on `visibilitychange` hidden and resume on visible; a run paused longer than 60s is abandoned rather than resumed
- Expose `remainingMs` at 100ms granularity for the UI bar without re-rendering every frame

**Out of Scope:**
- Rendering the bar (Story 6.2)

**Acceptance Criteria:**
- [ ] Tests with a fake clock prove 60.0s elapses in exactly 60.0s of injected time regardless of tick count
- [ ] Test: hiding the tab for 5s and returning does not consume the 5s
- [ ] No timer callbacks remain scheduled after `stop()` (assert via the injected scheduler spy)

---

## Epic 4 — Typeahead Answer Input

**Goal:** the interaction the whole game rests on — type, see matches, commit one.

#### Story 4.1 — Search index and matching

**Context:** The engine can ask "is this plant correct?" but nothing turns typed characters into candidate plants.

**Assumptions:**
- Epic 2 complete

**Tasks:**
- Write `src/search/plantIndex.ts`: build once from `PlantRepository`, normalizing names and aliases (lowercase, strip diacritics, strip `.'`-, collapse whitespace)
- Implement `search(query, limit)` ranking: exact normalized match > name prefix > word-boundary prefix > substring, ties broken alphabetically. **No fuzzy/edit-distance matching** — the answer must be selected, so typo tolerance is unnecessary complexity.
- Return at most 8 results; return `[]` for queries under 2 characters
- Search the whole compendium regardless of the active game filter, so the suggestion list never narrows into a hint about which game the current plant is from — mark the matched substring range for highlighting
- Suggestions display the canonical `name` only. Never render a game label, and never render the ingredient name — aliases match silently.

**Out of Scope:**
- Any React, keyboard handling, rendering

**Acceptance Criteria:**
- [ ] `"nirn"` returns both Nirnroot and Crimson Nirnroot, Nirnroot first
- [ ] `"st jahns"` and `"st. jahn's"` both return St. Jahn's Wort Nectar
- [ ] `"cap"` returns word-boundary matches ahead of mid-word ones
- [ ] `"lavender sprig"` and `"lavender"` both return the single Lavender entry, and no result anywhere shows a game name
- [ ] Search of the full dataset benchmarks under 1ms per query

---

#### Story 4.2 — AnswerInput component

**Context:** Matching works as a pure function. This story is the accessible combobox around it.

**Assumptions:**
- Story 4.1 complete; theme components may not exist yet — use unstyled markup with semantic classes

**Tasks:**
- Build `src/components/AnswerInput.tsx` and `SuggestionList.tsx` following the ARIA combobox pattern: `role="combobox"`, `aria-expanded`, `aria-activedescendant`, `aria-controls`, listbox with `role="option"`
- Keyboard: ↑/↓ move the active option, Enter commits the active option, Tab commits and moves on, Escape closes the list without clearing the input, typing reopens it
- **Enter with no active option does nothing** — an answer is only submitted by committing a suggestion. This is the rule that makes scoring unambiguous; make it explicit in a code comment.
- Mouse/touch: hover sets active, click commits, blur closes after a click-safe delay
- Debounce rendering (not matching) at ~60ms; autofocus on round start; clear on advance
- Highlight the matched substring in each suggestion

**Out of Scope:**
- Oblivion styling (Epic 5), wiring to the reducer (Epic 6)

**Acceptance Criteria:**
- [ ] Testing Library tests cover: arrow navigation, Enter-commits, Escape, click-commit, and Enter-with-no-selection doing nothing
- [ ] Axe reports no violations on the component in both open and closed states
- [ ] The whole interaction is completable with keyboard only, never losing focus
- [ ] Component takes `onCommit(plantId)` and holds no game state itself

---

## Epic 5 — Oblivion Theme & UI Shell

**Goal:** it looks like the Elder Scrolls, using only assets we own.

#### Story 5.1 — Design tokens and typography

**Context:** The app is unstyled Tailwind defaults. This story sets the visual vocabulary everything else uses.

**Assumptions:**
- Story 1.1 complete

**Tasks:**
- Define the palette in `src/styles/tokens.css` as CSS custom properties: aged parchment (`--parchment`, `--parchment-shadow`), dark leather-brown frame (`--frame`, `--frame-dark`), Oblivion's muted gold accent (`--gold`, `--gold-dim`), ink text (`--ink`, `--ink-faded`), plus semantic `--correct` (mossy green) and `--wrong` (dried-blood red)
- Map the tokens into `tailwind.config.ts` as named colors so utilities read as `bg-parchment`, `text-ink`
- Load two **openly licensed** fonts from Google Fonts, self-hosted via `@fontsource` so there is no third-party request at runtime: a medieval-flavored display face for headings and a readable serif for body. Verify the OFL license and record both names and the license in a `## Credits` section in this README.
- **Ship no Bethesda font, texture, or UI graphic.** Every border, corner ornament, and divider in this project is CSS or hand-drawn SVG.
- Define the type scale, a `--radius` of near-zero (Oblivion frames are hard-edged), and two elevation shadows
- Set the page background to a CSS-only parchment texture (layered `repeating-linear-gradient` + `radial-gradient` noise), not an image file

**Out of Scope:**
- Components (5.2), animation (7.2)

**Acceptance Criteria:**
- [ ] A `/styleguide` dev-only route renders every token as a swatch with its name
- [ ] Fonts load from the bundle — DevTools Network shows zero requests to `fonts.googleapis.com` or `fonts.gstatic.com`
- [ ] `grep -ri "bethesda\|oblivion.ttf\|kingthings" public/ src/` finds no asset files
- [ ] Contrast of `--ink` on `--parchment` and `--gold` on `--frame` both pass WCAG AA

---

#### Story 5.2 — Theme component kit

**Context:** Tokens exist. This story builds the handful of primitives every screen composes.

**Assumptions:**
- Story 5.1 complete

**Tasks:**
- Build `src/components/theme/`: `Panel` (the parchment sheet with a drawn double-rule border and gold corner marks), `ScrollFrame` (an outer bordered container evoking the Oblivion menu chrome), `Button` (gold-bordered, three states, with a pressed inset), `Heading`, `Divider` (a centered ornamental rule)
- Draw all ornamentation as inline SVG or CSS `border-image` from a hand-authored SVG — no raster
- Keep the kit small and composable; every component takes `className` and forwards refs
- Restyle `AnswerInput` and `SuggestionList` from Story 4.2 to use the kit — the suggestion list is a parchment dropdown with a gold active-row highlight
- Add all components to the `/styleguide` route

**Out of Scope:**
- Screen layout (Epic 6), sound

**Acceptance Criteria:**
- [ ] `/styleguide` renders every component in every state
- [ ] The typeahead tests from Story 4.2 still pass unchanged (styling did not alter behavior)
- [ ] Total CSS in the built bundle is under 30 KB gzipped
- [ ] The UI is recognizably Elder-Scrolls-flavored while containing no Bethesda-derived file

---

## Epic 6 — Screens & Session Flow

**Goal:** the parts become a game you can sit down and play.

#### Story 6.1 — Title screen and mode selection

**Context:** Engine, input, and theme exist but nothing connects them.

**Assumptions:**
- Epics 3–5 complete

**Tasks:**
- Build `src/screens/TitleScreen.tsx`: title, one-line rules, two mode buttons (Endless, Time Attack), and the player's best score per mode
- Add a game filter control (All / Oblivion / Skyrim) that passes `gameFilter` into `createRoundSequence`; default All. This narrows the *pool*, never the naming — a plant is called the same thing under every filter.
- Wire app-level state in `App.tsx` with `useReducer` over the engine reducer; screens are chosen by reducer phase, no router
- Keep every gameplay decision in the engine — the screen dispatches actions and renders state, nothing more

**Out of Scope:**
- Results screen, persistence internals

**Acceptance Criteria:**
- [ ] Choosing a mode transitions to `playing` with a plant on screen
- [ ] Selecting "Skyrim" produces only plants whose `games` includes Skyrim over 50 rounds (test against the engine, not the DOM)
- [ ] No screen renders a game name next to a plant name at any point during a run
- [ ] Best scores render as "—" on a fresh profile without throwing

---

#### Story 6.2 — Play screen

**Context:** A run can start. This story is the screen the player actually spends their time on.

**Assumptions:**
- Story 6.1 complete

**Tasks:**
- Build `src/screens/PlayScreen.tsx`: `PlantCard` (the image in a framed panel) centered, `AnswerInput` beneath, `StreakMeter` and score in the corner, `TimerBar` only in Time Attack
- Build `FeedbackFlash`: on commit, reveal the correct name and flash the frame green or red for ~700ms before auto-advancing; the input is disabled during feedback
- On a wrong answer in Endless, hold the reveal until the player presses a key or the button — do not auto-advance out of a run ending
- Preload the next round's image during the current round so there is no flash of empty frame
- Add a quit affordance (Escape twice, or a small button) that ends the run and goes to results

**Out of Scope:**
- Results screen (6.3), animation polish (7.2)

**Acceptance Criteria:**
- [ ] A full Endless run is playable start to finish with keyboard only
- [ ] Time Attack ends at exactly 0.0s and shows results
- [ ] Playwright spec plays 5 rounds and asserts the score advances
- [ ] No image pop-in between rounds on a throttled "Fast 3G" profile

---

#### Story 6.3 — Results screen and local best scores

**Context:** Runs end but nothing is shown or remembered.

**Assumptions:**
- Story 6.2 complete

**Tasks:**
- Write `src/storage/localScores.ts`: a versioned record (`{ v: 1, endless: {...}, timeAttack: {...} }`) under one key, with try/catch on every read and write and a schema check that discards corrupt or old-version data instead of throwing
- Build `src/screens/ResultsScreen.tsx`: final score, correct/wrong, longest streak, "new best" badge, and a **missed plants** list showing image and name for each one gotten wrong — this is the part that teaches
- Add "Play Again" (same mode and filter) and "Back to Title"
- Persist best score, longest streak, and total runs per mode; never persist anything identifying

**Out of Scope:**
- Any server, sharing, screenshots

**Acceptance Criteria:**
- [ ] Best score survives a reload; a lower score does not overwrite it
- [ ] Writing garbage into the localStorage key and reloading yields a working fresh profile, not a crash
- [ ] Tests cover the private-browsing case where `localStorage` throws on write
- [ ] Missed-plants list is empty and gracefully hidden on a perfect run

---

## Epic 7 — Polish, Accessibility, Performance

#### Story 7.1 — Responsive layout

**Context:** The game is built desktop-first.

**Assumptions:**
- Epic 6 complete

**Tasks:**
- Make the play screen work from 320px to ultrawide: image scales within a max, input stays above the fold with a mobile keyboard open (`dvh` units, not `vh`)
- On touch, the suggestion list must not be covered by the on-screen keyboard — test on a real device or an emulated one
- Increase touch targets to 44px minimum; keep the desktop density unchanged

**Acceptance Criteria:**
- [ ] Playable at 320×568 and 2560×1440 with no horizontal scroll
- [ ] With the iOS keyboard open, at least 3 suggestions remain visible
- [ ] No layout shift when the timer bar appears (CLS 0)

---

#### Story 7.2 — Motion and feedback polish

**Tasks:**
- Add restrained transitions: the plant card fades and lifts in, the streak meter pulses at multiplier steps, the timer bar reddens under 10s
- Gate every animation behind `prefers-reduced-motion: reduce` — reduced motion means instant state changes, never a degraded game
- Optionally add two short self-authored UI sounds (correct/wrong) behind a default-off mute toggle persisted locally; skip entirely if no openly licensed source is found

**Acceptance Criteria:**
- [ ] With reduced motion forced, no `transition` or `animation` runs and the game remains fully playable
- [ ] No animation exceeds 300ms
- [ ] If sound ships, it is off by default and the license is recorded in Credits

---

#### Story 7.3 — Accessibility pass

**Tasks:**
- Run axe across all three screens and the styleguide; fix everything reported
- Give every plant image a meaningful `alt` — but **not the plant's name during play** (it would leak the answer); use `"unidentified plant specimen"` while playing and the real name on reveal and in results
- Announce round outcomes via an `aria-live="polite"` region ("Correct — Nirnroot", "Wrong — that was Deathbell")
- Verify a full keyboard-only run and a screen-reader run of at least 3 rounds

**Acceptance Criteria:**
- [ ] Zero axe violations on every screen
- [ ] The answer is not present in the DOM before the player commits (assert in a test — this is a cheat vector, not just an a11y nicety)
- [ ] A run is completable with VoiceOver or NVDA

---

#### Story 7.4 — Performance budget and E2E in CI

**Tasks:**
- Set and enforce a budget: JS under 150 KB gzipped, LCP under 1.5s on simulated 4G, TTI under 2s
- Lazy-load the results screen; ensure `plants.json` and the SVG map are not duplicated across chunks
- Add the Playwright spec to `ci.yml`, running against `vite preview` of the production build
- Add a bundle-size check that fails CI on regression beyond the budget

**Acceptance Criteria:**
- [ ] Lighthouse scores 95+ on Performance, Accessibility, and Best Practices against the deployed URL
- [ ] E2E runs in CI in under 2 minutes
- [ ] Deliberately importing a large library fails the size check

---

## Epic 8 — Release

#### Story 8.1 — Production hardening

**Tasks:**
- Add security headers in `Caddyfile`: `Content-Security-Policy` (`default-src 'self'`; no `unsafe-inline` — move any inline style to the bundle), `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Permissions-Policy` denying camera/mic/geolocation
- Add `robots.txt`, a favicon drawn from the theme, and Open Graph tags with a self-authored preview image
- Verify Cloudflare caching is not serving a stale `index.html`; purge on deploy or rely on `no-cache`
- Add a `## Deployment` runbook: how to roll back (`fly releases` + `fly deploy --image`), where the token lives, what to check after a deploy

**Acceptance Criteria:**
- [ ] `curl -sI https://es-flora.brandonlocke.xyz` shows every header; CSP violates nothing in the console
- [ ] securityheaders.com grade A or better
- [ ] A rollback is performed once and documented with real output

---

#### Story 8.2 — Content review before launch

**Tasks:**
- Re-verify all plant names against UESP one final time; fix any spelling drift
- Re-check the merge set: no two entries are the same species, and no species is split across two entries
- Confirm the repository contains no Bethesda-derived file of any kind (`git ls-files | xargs file` review of every binary/asset)
- Fill in the `## Credits` section: font names and licenses, UESP as the data source with a clear "unofficial fan project, not affiliated with Bethesda" note
- Play 20 full runs; note any plant whose placeholder is misleading or whose `difficulty` is wrong, and correct the data

**Acceptance Criteria:**
- [ ] Credits section is complete and accurate
- [ ] Every asset in the repo is either generated by `scripts/` or hand-authored here
- [ ] Difficulty tiers adjusted based on the 20 recorded runs

---

## Epic 9 — Deferred (not part of v1)

Scoped now so v1 does not accidentally build for them.

- **9.1 Real art.** Replace placeholders plant-by-plant. No code change required — drop files into `public/assets/plants/`. Decide then between commissioned/hand-drawn SVG and AI-generated-and-curated. One image per plant: where the two games depict a species differently, pick the more recognizable rendering rather than adding a second asset.
- **9.2 Daily challenge.** A date-seeded run everyone gets identically. Works with zero backend (seed from the UTC date); a shared leaderboard does not.
- **9.3 Leaderboard.** The only feature that justifies a server. Would need a small Go API, a DB, rate limiting, and real thought about a client that can trivially lie about its score.
- **9.4 Study mode.** Browse the full compendium with names visible; the missed-plants list is the seed of this.
- **9.5 More games.** Morrowind and ESO flora are data-only additions: a shared plant appends to an existing entry's `games` array, a new one gets a new object.

---

## Secrets & Config Management

There is exactly one secret in this project.

| Name | Where it lives | Used by |
|---|---|---|
| `FLY_API_TOKEN` | GitHub repo secret | `deploy.yml` |

- The app requires **no runtime environment variables**. `.env.example` exists to document that fact.
- No API keys, no analytics, no third-party scripts, no cookies, no personal data. `localStorage` holds only scores.
- `git log -p | grep -iE 'fly_api_token|FlyV1'` must return nothing before any push.

---

## Definition of Done

A story is done when:

- [ ] `make typecheck && make lint && make test && make build` pass
- [ ] New logic has unit tests; new UI has at least one Testing Library test
- [ ] `make image && docker compose up` — container stays running, `curl localhost:8080` returns the app
- [ ] Acceptance criteria in the story are checked off with evidence (command output, screenshot, or test name)
- [ ] No secrets committed; no Bethesda-derived asset added
- [ ] README updated if a decision in it changed

The project is done when:

- [ ] All Epic 1–8 stories are done
- [ ] https://es-flora.brandonlocke.xyz plays both modes end to end on desktop and mobile
- [ ] Lighthouse 95+ on Performance, Accessibility, Best Practices
- [ ] A cold visitor can start playing within 2 seconds of load with no instructions beyond the title screen's one line

---

## Open Questions & Known Risks

| # | Item | Impact | Current call |
|---|---|---|---|
| 1 | Placeholder art may make the game unsatisfying to actually play | High — it is the whole visual premise | Ship it; Epic 2.3 biases shapes by name so images at least read as the right *kind* of plant. Reassess after Story 8.2's 20 runs. |
| 2 | Normalizing ingredient names to plant names is a judgment call | Medium — a name no player recognizes is worse than a clunky one | Rule and allowlist are in Story 2.2; the validator enforces it. Ambiguous cases (`Spiddal Stick`, `Elves Ear`) keep their in-game form. Every in-game ingredient name is a search alias regardless, so no reasonable input fails to match. |
| 3 | Fly.io scale-to-zero cold starts | Low | ~1s on a static Caddy image. Set `min_machines_running = 1` if it ever feels slow. |
| 4 | UESP name accuracy | Medium — wrong answers are worse than missing ones | Two verification passes: Story 2.2 and Story 8.2. |
| 5 | Font licensing | Low | Google Fonts OFL only, self-hosted, recorded in Credits. Any font requiring a purchase is out. |
| 6 | The whole project is a fan work of copyrighted IP | Low but real | No Bethesda files ship. Names are factual references. Add a clear unaffiliated-fan-project notice (Story 8.2). |
