# let's Try My Luck

A personal web app for tracking job applications — from finding a listing and sending a CV to knowing the final outcome.

It turns a passive Notion tracker (56 companies, 26 applications) into an active working tool: a clear pipeline, analytics on which channel actually produces results, and a follow-up list so applications stop going silent.

> **Status:** design finalized, implementation not started.
> Product document: [`PRD.md`](./PRD.md) · Technical design: [`docs/superpowers/specs/2026-11-26-job-tracker-design.md`](./docs/superpowers/specs/2026-11-26-job-tracker-design.md)

---

## Problems this solves

1. **No funnel visibility.** Statuses are free text; there is no way to see how many are hanging versus rejected versus progressing.
2. **Channels are unmeasured.** In the legacy data, 7 of 9 Techinasia applications were rejected, while 2 "Approved" results came from company websites. That insight is invisible without manual analysis.
3. **Applications go silent.** 17 of 26 applications are still "Sent CV" with no follow-up and no reminder system.
4. **Messy schema.** The "Industri" column holds ratings, "Overall State" is empty everywhere, and there are blank rows and typos.
5. **Slow input.** Adding an application in Notion takes many clicks and retyping.

---

## MVP features

Two core features, per the agreed scope:

**1. Dashboard (overview)**
- KPIs: total tracked, applied, active, rejected, offers, accepted
- Funnel: wishlist → applied → screening → interview → challenge → offer
- Conversion per channel (response rate and offer rate per `source` / `applied_via`)
- Follow-up list: active applications past the staleness threshold with no new activity
- Weekly activity

**2. Manual application management**
- Application and company CRUD, with single-call quick-add (company auto-created when missing)
- Table: multi-criteria filter, sort, inline edit, bulk actions, global search
- Application detail with activity timeline
- Separate `stage` and `outcome` pipeline
- Quick Add page pre-fillable from query params

Plus: Notion CSV import, CSV/JSON export, category settings, dark mode.

**Not in MVP:** Kanban board, calendar, automated reminders, email ingestion, PWA, multi-user, job-board scraping.

---

## Stack

| Layer | Technology |
|---|---|
| Backend | **Go** — strict N-layer with dependency inversion |
| Database | **PostgreSQL 16** |
| Frontend | **Next.js** (App Router) + TypeScript, Tailwind, shadcn/ui |
| Table | TanStack Table |
| Charts | Recharts |
| Reverse proxy | Caddy — single origin for API and web |
| Devops | Docker Compose + Makefile |
| Migrations | goose, versioned SQL |
| Testing | Go testing (fake repositories), httptest, Playwright (E2E) |

**Why Go + N-layer?** This is a learning project. The goal is understanding dependency direction, dependency inversion, and containerized devops — not copying a folder structure.

---

## Architecture

Dependency rule: every import points inward.

```
cmd/  →  adapter/  →  usecase/  →  domain/
                  →  platform/  (config, logger, clock, db)
```

- `domain/` — entities, pipeline rules, validation, repository **interfaces**. Imports nothing from this project.
- `usecase/` — business rules. Imports `domain` only. Knows neither HTTP nor SQL.
- `adapter/postgres` — repository implementations. `adapter/http` — handlers and DTOs, no SQL.
- `cmd/` — wiring only, manual constructor injection.

```
lets-try-our-luck/
├── docker-compose.yml
├── Caddyfile
├── Makefile
├── api/                      # Go
│   ├── cmd/api/              # wiring
│   ├── cmd/migrate/          # migration runner
│   ├── migrations/
│   └── internal/
│       ├── domain/
│       ├── usecase/
│       ├── adapter/
│       └── platform/
└── web/                      # Next.js
```

Topology — one origin, no CORS:

```
browser → Caddy :8080
            ├─ /api/*  → api:8080    (Go)
            └─ /*      → web:3000    (Next.js)
                        api ← postgres:5432
```

---

## Prerequisites

- **Docker Desktop** (required — not yet installed on the development machine when this design was written)
- Git
- For development without containers: Go ≥ 1.22, Node ≥ 20, PostgreSQL 16

---

## Running it

*(target — not available until M0 completes)*

```bash
git clone git@github.com:arifldhewo/lets-try-our-luck.git
cd lets-try-our-luck
cp .env.example .env
make up
```

Open `http://localhost:8080`. On an empty database the app opens a **first-run setup** screen to create a passphrase — no default password ships with the image.

### Commands

| Command | Purpose |
|---|---|
| `make up` | Start the whole stack |
| `make down` | Stop the stack |
| `make logs` / `make ps` | View logs / container status |
| `make migrate` / `make migrate-down` | Apply / roll back migrations |
| `make seed` | Seed initial categories |
| `make test` / `make lint` | Test / lint |
| `make psql` | Open PostgreSQL |
| `make reset` | Wipe data (**destructive**, confirmation-gated) |

---

## Deployment modes

One set of images, three modes — **configuration only, no code changes**. Designed so anyone can run it without a domain.

| Mode | Trigger | TLS | Phone access |
|---|---|---|---|
| **Local / LAN** (default) | `APP_DOMAIN` unset | `tls internal` | Home Wi-Fi via LAN IP |
| **Tailscale** | add `compose.tailscale.yml` | `tailscale serve` (sidecar) | Anywhere on the tailnet |
| **Public** | `APP_DOMAIN=jobs.example.com` | automatic ACME | Anywhere |

Tailscale runs as a sidecar rather than a Caddy plugin, so the base image stays stock and there is no custom build to trust.

> **Security note:** public mode uses a passphrase only, with no rate-limiting or lockout in MVP. Adequate for LAN and tailnet; public exposure needs additional hardening.

---

## Data model

Three core entities: `companies`, `applications`, `activities` (append-only log).

**Schema decisions in force:**

- **No database-level enums.** Categorical values are stored as `text`. Adding a stage or channel needs no migration.
- **No foreign keys.** Relationships are plain `uuid` columns.
- **Soft deletes only.** `archived = true`; no hard `DELETE`, no cascade.

The consequence: **integrity is enforced in Go**, not by the database. A single validation package is used on every write path, and read queries tolerate orphaned rows.

Editable categories (source, industry, company type, etc.) live in a `lookup_values` table. `stage` and `outcome` are deliberately **not** editable because the funnel requires fixed ordering.

### Pipeline

**Stage:** `wishlist` → `applied` → `screening` → `interview` → `challenge` → `offer`

**Outcome** (independent of stage): `active`, `rejected`, `offer`, `accepted`, `closed`, `withdrawn`

Separating the two is what fixes the legacy "Overall State empty" problem.

---

## Roadmap

| Milestone | Contents |
|---|---|
| **M0** | Foundation: Docker, compose (postgres/api/web/caddy), Go layout, migrations, config/logger/clock, healthcheck |
| **M1** | Schema + categories, validation package, entities, repository interfaces |
| **M2** | **Feature 1** — company & application CRUD, activity log, table, detail page, Quick Add |
| **M3** | **Feature 2** — dashboard, funnel, channel conversion, follow-up list |
| **M4** | Notion data migration (CSV), normalization, dedupe, transactional commit |
| **M5** | Export, settings, dark mode, backup, a11y, E2E, docs |

**MVP done** = legacy data fully imported (56 companies + 26 applications verified), both features working, dashboard shows conversion per channel, export works, first-run setup works on an empty database, and `make up` boots the entire stack on a clean machine.

**Phase 2:** Kanban board, calendar, reminders, email ingestion, PWA, document management.

---

## Testing

- **Usecase** tests with fake repositories — no DB, no HTTP. This is the main payoff of the N-layer.
- **Handler** tests via `httptest` against the real router.
- **Adapter** tests against a real PostgreSQL from compose.
- **Playwright** for E2E (quick-add, import, stage change) — Playwright's only MVP role.
- **Import fixture** built from the real Notion export, including dirty rows: blank rows, the `Product Desginer` typo, empty `Overall State`.

Playwright is **not** used for job-board scraping. The reasoning is in §6 of the design document.

---

## Documentation

- [`PRD.md`](./PRD.md) — product document: problems, persona, metrics, user stories
- [`docs/superpowers/specs/2026-11-26-job-tracker-design.md`](./docs/superpowers/specs/2026-11-26-job-tracker-design.md) — technical design: architecture, schema, API, deployment, decisions

> PRD §11 still lists the earlier stack (Next.js fullstack + Drizzle). The design document is authoritative: Go + N-layer + containerized.

---

## License

[GNU AGPL-3.0](./LICENSE)
