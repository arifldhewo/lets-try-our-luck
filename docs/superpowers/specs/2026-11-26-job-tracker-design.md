# Design — "let's Try My Luck" Job Tracker

**Date:** 2026-11-26
**Status:** Draft for approval
**Supersedes:** PRD.md §11 (Stack & Arsitektur) and PRD.md §1 non-goal on scraping

---

## 1. Intent and why

I have 56 companies and 26 applications sitting in a Notion database. The data is
inert: statuses are free text, the industry column contains ratings, 17 of 26
applications have been "Sent CV" with no follow-up, and no one can answer "which
channel actually works" without doing the analysis by hand.

This builds a personal, single-user web app that makes that data work: a clear
stage/outcome pipeline, a dashboard that reproduces the channel insight
automatically, and a follow-up list so applications stop going silent.

Secondary and equally explicit intent: **this is a learning vehicle.** The choice of
Go with strict N-layer architecture is a learning objective, not a technical
requirement of the problem. The design is written to maximize what is learned
(dependency inversion, containerized devops, real dev loop) without letting that
objective delay a working app.

## 2. Goals

- **G1** — Record, filter, and see every application fast. Median add time < 30s.
- **G2** — Replace free-text status with a canonical `stage` + `outcome` model.
- **G3** — Dashboard answers: channel effectiveness, response rate, what needs follow-up.
- **G4** — Migrate the old Notion data with zero loss.
- **G5** — Usable on desktop and phone.

## 3. Non-goals (MVP)

- Not multi-user, SaaS, or shared. No teams, billing, or public registration.
- Not a job board. Does not pull listings from the internet.
- Does not submit applications on my behalf.
- No mobile native app. Responsive web only, PWA deferred.
- No AI resume tailoring.
- **No Playwright scraping of job boards.** Explicitly dropped; see §11.

## 4. Anti-goals

These are failure modes even if the app technically runs:

- **A-NG1** — It becomes slower to log an application than Notion was. Then it is dead
  on arrival, regardless of dashboard quality.
- **A-NG2** — Layering ceremony becomes the work. If adding a field takes four files
  and an interface, the architecture has failed its own purpose.
- **A-NG3** — The Docker setup is the only way to run it and it does not run on this
  machine. No Docker Desktop installed yet — this is a live risk, not hypothetical.
- **A-NG4** — A scraping integration breaks weekly and I stop maintaining the app.
- **A-NG5** — The 26 applications / 56 companies do not survive migration intact.

## 5. Constraints (locked)

Decided together, before this document:

| # | Constraint | Consequence |
|---|---|---|
| C1 | **No Postgres enum types.** Categorical values are `text`. | Adding a stage/channel needs no migration. Validation is app-side. |
| C2 | **No foreign key constraints at all.** Relationships are plain `uuid` columns. | The DB cannot reject a dangling `company_id`. Go owns referential integrity. |
| C3 | **Soft deletes only.** No hard `DELETE`, no `ON DELETE CASCADE`. | Every read filters `archived = false`. `Company` needs an `archived` column (PRD only gave it to `Application`). |
| C4 | **Go + strict N-layer.** Learning objective. | See §6. Dependency direction must be enforced, not just named. |
| C5 | **Next.js frontend**, separate process from the API. | Needs a real HTTP boundary — see §9. |
| C6 | **PostgreSQL**, containerized. | Runs in compose; no managed cloud DB in MVP. |
| C7 | **Containerized from day one**, single origin. | Compose + reverse proxy. No CORS, no cross-site cookies. |
| C8 | **Two MVP features only:** Dashboard overview, and manual application management. | Board/Kanban, calendar, reminders, documents are Phase 2. |
| C9 | UI language Indonesian, `Asia/Jakarta` for all date bucketing. | See §12. |
| C10 | **Self-hostable by others with no code changes.** | Config-only, first-run setup, no personal data baked in. See §11. |
| C11 | **Tailscale as an optional override, never a requirement.** | Base compose stays vanilla; tailnet access is a compose file addition. |
| C12 | **Historical data shown as-is.** No `historical` flag, no cutoff. | All 17 "Sent CV" rows will surface as ~1,030 days stale. Accepted deliberately. |
| C13 | **Lean table scope.** Filter, sort, inline edit, bulk actions, global search, `/` and `n` shortcuts. | Saved views, group-by, column reorder, virtualization are Phase 2 — no `group` or saved-view API needed. |
| C14 | **Notion export is one flattened CSV** (company columns repeated per application row). | Import needs a two-pass read; ~30 companies with no application have no row. See §8. |

## 6. Approaches considered

### Layering

- **A — Three-layer per domain** (`internal/company/{handler,service,repository}.go`).
  Fastest to feature one. Layering is naming convention only; nothing prevents a
  service importing a handler, so the dependency rule is never actually learned.
  *Rejected* — the learning objective is the point, and this teaches the diagram, not
  the constraint.
- **B — Strict layered with dependency inversion** (`domain` / `usecase` / `adapter` /
  `platform`), interfaces in `domain`, implementations in `adapter`. Every import
  points inward. A usecase is testable with a fake repo, no database.
  **Chosen.** More files per feature, and that is the price of learning where logic
  belongs. Direct migration to the familiar TypeScript routes → controller → service
  → repository mental model.
- **C — Vertical slices** (all layers per feature folder). *Rejected* — not N-layer, and
  it hides exactly the boundaries we are trying to learn.

### Data access / read models

- **CQRS with separate write and read services.** Pays off under extreme read/write
  asymmetry or multi-team projections. Here there is one user and 26 rows; it buys
  eventual consistency on a personal tool (add an application, dashboard doesn't show
  it) and a second deploy target, while destroying the shared-schema benefit.
  *Deferred, not rejected* — `Activity` is already an append-only log, which is the
  legitimate seam. Revisit only if dashboard queries get slow.
  **Chosen:** plain SQL read queries in `adapter/postgres/dashboard_repo.go`.

### Automation

- **Playwright scraping of job boards.** Fights Cloudflare/DataDome, breaks weekly,
  violates ToS, and does not serve G3 — channel effectiveness needs *my outcomes*, not
  more listings. *Rejected for MVP.* Retained only for E2E tests.
- **Email ingestion** (Gmail label/API → auto-create `Activity`, auto-advance stage).
  Directly attacks "17 of 26 hanging". The real automation win. *Phase 2* — needs
  worker process and credentials.
- **Bookmarklet / quick-add page.** One same-origin page pre-filled from query params.
  No anti-bot fight, no token, no worker. **Chosen for MVP.**
- **Paste-URL parse** (server fetch + `schema.org/JobPosting` JSON-LD). *Phase 2.*

### Devops

- **Local Go + Next, cloud Postgres, containers at deploy time.** Fastest to a running
  app, teaches the retrofit — a genuinely realistic devops scenario.
- **Compose from day one** (Postgres + API + web + reverse proxy, one origin).
  **Chosen**, at explicit cost of a setup step before feature one. Also removes the
  CORS and `SameSite=None` cookie problem entirely, which is the most common way a
  small fullstack app gets stuck.

## 7. Architecture

### Dependency rule

```
cmd/  →  adapter/  →  usecase/  →  domain/
                  →  platform/  (config, logger, clock, db)
```

- `domain/` imports **nothing** from this project. Entities, stage/outcome rules,
  validation, and repository **interfaces** live here.
- `usecase/` imports `domain` only. Never `net/http`, never `database/sql`.
- `adapter/` implements domain interfaces. `adapter/http` has zero SQL;
  `adapter/postgres` has zero HTTP.
- `cmd/` wires everything. Manual constructor injection, no DI container.

### Layout

```
lets-try-our-luck/
├── docker-compose.yml
├── Caddyfile
├── Makefile
├── .env.example
├── PRD.md
├── docs/superpowers/specs/
├── api/                                  # Go
│   ├── Dockerfile                        # multi-stage, distroless/static final
│   ├── go.mod
│   ├── cmd/
│   │   ├── api/main.go                   # wiring only
│   │   └── migrate/main.go               # goose runner
│   ├── migrations/                       # versioned SQL
│   └── internal/
│       ├── domain/
│       │   ├── company.go
│       │   ├── application.go
│       │   ├── activity.go
│       │   ├── pipeline.go               # stage/outcome sets + transition log
│       │   ├── validate.go               # THE integrity guard (see §8)
│       │   ├── errors.go
│       │   └── ports.go                  # repository interfaces
│       ├── usecase/
│       │   ├── company_usecase.go
│       │   ├── application_usecase.go
│       │   ├── dashboard_usecase.go
│       │   ├── import_usecase.go
│       │   └── export_usecase.go
│       ├── adapter/
│       │   ├── postgres/
│       │   │   ├── db.go
│       │   │   ├── company_repo.go
│       │   │   ├── application_repo.go
│       │   │   ├── activity_repo.go
│       │   │   ├── lookup_repo.go
│       │   │   └── dashboard_repo.go
│       │   └── http/
│       │       ├── server.go             # http.Server lifecycle
│       │       ├── router.go             # ServeMux patterns
│       │       ├── middleware.go         # auth, logging, recovery, request id
│       │       ├── auth_handler.go
│       │       ├── company_handler.go
│       │       ├── application_handler.go
│       │       ├── dashboard_handler.go
│       │       ├── import_handler.go
│       │       ├── export_handler.go
│       │       └── dto.go
│       └── platform/
│           ├── config/config.go
│           ├── logger/logger.go
│           └── clock/clock.go            # injectable, Asia/Jakarta
├── web/                                  # Next.js App Router + TS
│   ├── Dockerfile                        # standalone output
│   └── src/{app,components,lib}/
└── scripts/
```

Use the standard library `net/http` with Go 1.22+ method+wildcard patterns
(`GET /api/applications/{id}`). No router framework needed; one less dependency to
learn around.

## 8. Data model

**Policy (C1, C2, C3):** no `CREATE TYPE`, no `REFERENCES`, no cascade, no hard
delete. Plain `uuid`/`text` columns. Indexes are fine — they are not constraints.

### `companies`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| name | text NOT NULL | dedupe handled in Go; non-unique index on `lower(name)` |
| website, industry, company_type, size, design_team_size, location | text NULL | categorical values validated in Go |
| notes | text NULL | |
| archived | bool NOT NULL default false | C3 |
| created_at / updated_at | timestamptz | |

### `applications`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| company_id | uuid NOT NULL | **no FK** (C2). Existence checked by usecase |
| position | text NOT NULL | |
| position_type, job_url, source, applied_via, salary_expectation | text NULL | |
| found_date, applied_date | date NULL | calendar, not timestamps |
| stage | text NOT NULL | validated against active lookup codes |
| outcome | text NOT NULL | validated against active lookup codes |
| rating | jsonb NULL | `{branding, product, design, vibe, overall}` — kept as one blob (PRD open Q5 unresolved; jsonb reshapes without migrations) |
| notes | text NULL | |
| archived | bool NOT NULL default false | |
| created_at / updated_at | timestamptz | |

`is_submitted` is **not stored.** It is `applied_date IS NOT NULL`, computed in
queries. Storing derived data invites drift.

### `activities`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| application_id | uuid NOT NULL | no FK (C2) |
| type | text NOT NULL | `created`, `applied`, `stage_changed`, `outcome_changed`, `note`, `interview`, `followup`, `reminder` |
| from_value, to_value | text NULL | |
| note | text NULL | |
| occurred_at | timestamptz NOT NULL | drives time-to-response |
| created_at | timestamptz | |

Append-only. This is the CQRS/event-sourcing seam — never updated, never deleted.

### `lookup_values` — one table instead of five settings screens

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| kind | text NOT NULL | `source`, `applied_via`, `industry`, `company_type`, `size`, `design_team_size`, `position_type`, `salary_expectation` |
| code | text NOT NULL | stable value stored on rows, e.g. `techinasia`, `screening` |
| label | text NOT NULL | Indonesian display label |
| sort_order | int NOT NULL default 0 | |
| active | bool NOT NULL default true | |

This is how "editable categories without deploy" (PRD 7.9 AC1) is satisfied **without**
DB enums and **without** building five admin screens: seeded by migration, edited in
one Settings table, cached in Go with invalidation on `PATCH`.

**`stage` and `outcome` are deliberately NOT in `lookup_values`.** The funnel and the
`stage >= screening` comparison need a fixed order, so both live as constants in
`domain/pipeline.go` and render as a read-only list in Settings. This narrows PRD 7.9
AC1 to *every category except pipeline stages* — a stage added at runtime would silently
break funnel ordering and interview-rate math, which is worse than a deploy.

### `settings`

Single row (`id = 1`, enforced by the usecase, not a constraint — C2):

| Column | Type | Notes |
|---|---|---|
| passphrase_hash | text NULL | NULL until first-run setup completes |
| followup_threshold_days | int NOT NULL default 14 | PRD 7.9 AC4 |
| locale | text NOT NULL default 'id' | |
| theme | text NOT NULL default 'system' | |

### `import_sessions` / `import_rows`

Staging for the Notion migration (PRD 7.6). A session holds the uploaded file's parsed
rows with `raw jsonb`, `normalized jsonb`, `error text`, `accepted bool`, so preview
survives across requests and commit is one transaction over accepted rows.

**The export is one flattened CSV** (C14): company columns repeat on every application
row. That has one consequence worth naming, because it breaks a goal silently otherwise:

> Companies with **no** application — roughly 30 of the 56 — have no row in a flattened
> export. G4 and A-NG5 require all 56 companies to land.

So import runs **two passes over the same file**: pass 1 collects distinct companies
from every row (including from wishlist/blank-status rows), pass 2 creates applications.
A company-only row is recognized by an empty position/title column and becomes a company
without an application. Preview reports both counts — `56 perusahaan, 26 lamaran` — so a
silent shortfall is visible before commit.

Unmapped legacy `source` / `applied_via` / status values **create a new `lookup_values`
row** (auto-slugged code, the raw string as label, `active = true`) rather than collapsing
to `lainnya`. Flattening to `lainnya` would destroy the channel insight that G3 exists to
produce — the Techinasia 7/9 baseline would vanish into noise. New codes surface in
Settings for later renaming.

## 9. Integrity policy — Go owns it

Because the DB enforces nothing (C1, C2), this is load-bearing and has no fallback:

1. **`domain/validate.go` is the single guard.** Every write path calls it. Handlers
   never construct entities; usecases do, and usecases validate.
2. **Referential checks happen in the usecase, as find-or-create.** `POST /api/applications`
   accepts **either** `company_id` **or** `company_name`. With an id, `FindByID` and
   `422` if absent. With a name, case-insensitive lookup on `lower(name)` and create if
   absent — matching PRD 7.1 AC2. This keeps quick-add to a **single call**, which G1 and
   A-NG1 depend on. A two-call find-or-create sequence is explicitly rejected.
3. **Categorical writes are validated against cached active `lookup_values`.** Unknown
   `stage`/`source` code → `422` with the offending field, never a DB rejection.
4. **Reads tolerate bad rows.** Any application-to-company join is a `LEFT JOIN` with
   `COALESCE(c.name, '(perusahaan dihapus)')`, so a dangling `company_id` renders instead
   of vanishing or erroring the dashboard.
5. **Nothing hard-deletes.** `archived = true`, and every query filters it.

**Follow-up semantics.** "Set follow-up" appends an `Activity` of type `followup` with a
note — no due-date column, since reminders are Phase 2. Per PRD §8, a `followup` newer
than the threshold **resets the stale clock**: an application you chased last week is not
stale. Other activity types (`note`, `interview`) also count as "new activity" and reset
it, matching "tanpa aktivitas baru" literally. This is why `occurred_at` is the only time
input the stale query needs.

Usecase tests must cover: unknown company id, unknown stage code, archived company,
and application whose company does not exist.

## 10. API surface

Single origin, JSON, session cookie. Base `/api`.

```
GET    /api/setup/status             unauthenticated → {needsSetup: bool}
POST   /api/setup                    unauthenticated, one-shot → sets passphrase hash
POST   /api/auth/login               {passphrase} → sets session cookie
POST   /api/auth/logout
GET    /api/health                   unauthenticated

GET    /api/companies                ?q=&includeArchived=
POST   /api/companies
GET    /api/companies/{id}
PATCH  /api/companies/{id}
POST   /api/companies/{id}/archive

GET    /api/applications             ?stage=&outcome=&source=&companyType=&archived=&sort=&q=&page=
POST   /api/applications             {company_id | company_name, position, stage, ...}
GET    /api/applications/{id}
PATCH  /api/applications/{id}
POST   /api/applications/{id}/archive
GET    /api/applications/{id}/activities
POST   /api/applications/{id}/activities     # note / followup / interview
POST   /api/applications/{id}/stage          # {stage, note?} → logs stage_changed
POST   /api/applications/{id}/outcome        # {outcome, note?} → logs outcome_changed
POST   /api/applications/bulk                # {ids[], action, value}

GET    /api/dashboard/summary                # KPIs + funnel
GET    /api/dashboard/channels               # conversion per source / applied_via
GET    /api/dashboard/followups              # stale list
GET    /api/dashboard/weekly                 # applications per ISO week

POST   /api/imports                          # multipart CSV → session, parsed rows
GET    /api/imports/{id}                     # preview + per-row errors
POST   /api/imports/{id}/commit              # transactional commit
GET    /api/export?format=csv|json
GET    /api/lookups                          # all active codes + labels
PATCH  /api/lookups/{id}                     # label / sort_order / active
```

Global search is served by `q=` on `/api/applications` and `/api/companies` — one
endpoint each, no separate `/search` route.

Errors: `{error: {code, message, fields?}}`. Validation failures are `422`.

## 11. Deployment, portability and container topology

**Design goal (C10):** anyone can run this with `git clone && cp .env.example .env &&
make up`, with no code changes, no domain required, and no personal data present.

### One origin, three modes, identical images

```
browser → Caddy :8080
            ├─ /api/*  → api:8080   (Go)
            └─ /*      → web:3000   (Next.js)
                        api ← postgres:5432
```

Mode is **configuration only** — never a code change:

| Mode | Trigger | TLS | Phone access |
|---|---|---|---|
| **Local / LAN** (default) | `APP_DOMAIN` unset | `tls internal` (local CA) | Home Wi-Fi via LAN IP |
| **Tailscale** (C11) | `COMPOSE_FILE` includes `compose.tailscale.yml` | `tailscale serve` terminates | Anywhere on the tailnet |
| **Public** | `APP_DOMAIN=jobs.example.com` | automatic ACME | Anywhere |

The `Caddyfile` selects on env:

```caddy
${APP_DOMAIN:localhost} {
    tls {$TLS_MODE:internal}
    handle /api/* { reverse_proxy api:8080 }
    handle { reverse_proxy web:3000 }
}
```

Tailscale runs as a **sidecar service**, not a Caddy plugin. This keeps the base image
stock (no custom Caddy build to trust), and the override file is one line for the user
to add:

```yaml
# docker-compose.override addition — compose.tailscale.yml
services:
  tailscale:
    image: tailscale/tailscale:latest
    environment: [TS_AUTHKEY, TS_STATE_DIR, TS_SERVE_CONFIG]
    volumes: [tailscale-state:/var/lib/tailscale]
    cap_add: [NET_ADMIN]
```

`tailscale serve` proxies to `caddy:8080` on the tailnet. Nothing in `api/` or `web/`
knows Tailscale exists.

### First-run setup (required by C10)

No passphrase in env or seed — a shipped default password is a vulnerability for every
other person who runs this. `settings` table holds a single row; while it is empty,
`/api/setup/status` reports `needsSetup: true` and the UI routes to a setup screen that
writes the passphrase hash. `POST /api/setup` refuses once the row exists. This is what
makes "deployable out of the box" real rather than a redeploy-per-user instruction.

### No personal data in seed

`lookup_values` seeds generic taxonomy (LinkedIn, JobStreet, Techinasia, Glints,
Kalibrr, Dealls, Referral, Website, Lainnya). The Notion data arrives only through the
import flow. Nothing about this user ships in the repo or the image.

### Operational targets

`make up`, `down`, `logs`, `ps`, `migrate`, `migrate-down`, `seed`, `test`, `lint`,
`psql`, `reset` (destructive, confirm-gated). Compose healthchecks plus
`depends_on: condition: service_healthy` so the API never starts against a dead DB.

Backup is a `pg_dump` sidecar with 30-day rotation into a named volume, plus a
documented restore path exercised once (PRD 7.7 AC2). Not a backup platform.

**Risk (A-NG3):** Docker Desktop is not installed on this machine (verified: `docker`
not found). M0 starts by installing it. Fallback if it cannot: native Postgres +
`go run` + `npm run dev`, keeping the same compose files for later — they are written
either way.

## 12. Timezone and metrics

All bucketing uses `Asia/Jakarta`, injected via `platform/clock` so tests are
deterministic. Calendar fields use `date`; events use `timestamptz`. "Stale > 14 days"
and ISO-week grouping are computed server-side in that zone, never in the browser.

Metric definitions follow PRD §8 verbatim — that section is precise and is treated as
normative. Staleness computed on read at first; a materialized view is the documented
escalation if it gets slow.

**Known consequence (C12).** The applications are Jan–Feb 2024 and it is now Nov 2026.
Taken literally, PRD §8 puts all 17 "Sent CV" rows in the follow-up list roughly 1,030
days late, and weekly cadence renders as two bars in Feb 2024. This was raised and the
answer was *show everything as-is* — no cutoff, no `historical` flag, no special-casing.
The dashboard is arithmetically correct and the follow-up list will be dominated by old
data until current applications are logged. Accepted deliberately, not overlooked.

## 13. Frontend

Next.js App Router, TypeScript, Tailwind + shadcn/ui, TanStack Table for the
applications table, Recharts for dashboard charts. Server components fetch via the
internal API. No client-side auth logic — middleware checks the session cookie.

MVP screens: Dashboard, Applications (table + detail), Companies, Import, Settings
(lookup editing + theme), and **Quick Add** — a static same-origin page whose form
pre-fills from query params (`/quick-add?job_url=&company=&position=`), which is the
MVP's "bookmarklet". No token, no extension, no scraping. Board is deferred (C8).

Table scope is the lean set (C13): multi-criteria filter, sort, inline cell edit, bulk
actions, global search via `q=`, and `/` + `n` shortcuts. Saved views, group-by, column
reorder, and virtualization are Phase 2 — which is why `GET /api/applications` has no
`group` parameter.

## 14. Testing

- **Usecase tests with fake repositories** — the payoff of C4. No DB, no HTTP.
- **Handler tests** via `httptest` against the real router with a fake usecase.
- **Adapter tests** against a real Postgres from compose (`make test-integration`).
- **Playwright E2E** for quick-add, import, stage change — Playwright's only MVP role.
- **Import fixture** built from the actual Notion export, including deliberately dirty
  rows (blank rows, `Product Desginer` typo, `Overl State` empty, wrong industry values).

## 15. Milestones

- **M0 — Foundation.** Install Docker Desktop. Compose (postgres/api/web/caddy), Go
  module + layout skeleton, migrations runner, config/logger/clock, healthcheck,
  `make up` brings the whole stack up behind one origin.
- **M1 — Schema + lookups.** All tables, `lookup_values` seeded from PRD §6.3, validation
  package, domain entities, repository interfaces.
- **M2 — Manual management (feature 1).** Company + Application CRUD, find-or-create on
  add, activity log, stage/outcome endpoints, table with filter/sort/inline edit/bulk,
  global search, detail page with timeline, Quick Add page.
- **M3 — Dashboard (feature 2).** KPIs, funnel, channel conversion, follow-up list,
  weekly cadence.
- **M4 — Migration.** Notion flattened-CSV import with two-pass company/application
  read, mapping, normalization, per-row errors, dedupe, transactional commit. All 56
  companies + 26 applications landed, counts shown in preview.
- **M5 — Polish.** Export CSV/JSON, settings, dark mode, first-run setup, backup
  sidecar + tested restore, a11y pass, E2E tests, docs, deploy guide for the three modes.

Phase 2 (deferred): Board/Kanban, calendar, reminders, email ingestion, PWA, documents.

**"MVP done"** = old data fully imported with company count verified at 56, both requested
features working, dashboard shows conversion per channel, export works, first-run setup
works on a clean database with no passphrase pre-set, and `make up` boots the whole thing
on a clean machine in default (local) mode. Tailscale and public modes documented and
verified at least once.

## 16. Left to the builder

- Exact SQL for each query and index choice.
- Chart component selection details within Recharts.
- Table column show/hide implementation.
- Cookie signing scheme (any HMAC-signed session; no library mandated).
- Migration tool: `goose` preferred, `atlas` acceptable.
- Logging format and levels.

## 17. Calls I made — check these

These were not discussed; they are proposals for correction:

1. **Auth = passphrase + HMAC-signed httpOnly session cookie**, `SameSite=Lax` (safe
   because single origin). No WebAuthn in MVP. Rationale: least code, satisfies PRD A1.
   Passphrase is set at **first-run setup**, never shipped in env or seed (C10).
2. **`is_submitted` not stored** — derived in SQL. PRD listed it as a column.
3. **`lookup_values` single table** instead of five settings screens or DB enums —
   excluding `stage`/`outcome`, which stay ordered constants in `domain/pipeline.go`.
4. **Ratings collapsed into one `rating jsonb`** column rather than five typed columns,
   because PRD open question 5 is unresolved.
5. **Board/Kanban deferred to Phase 2** — PRD had it in MVP, but C8 named only two
   features.
6. **No router library** — stdlib `net/http` patterns.
7. **Company dedupe in Go**, non-unique `lower(name)` index, no unique constraint
   (consistent with C2's spirit: the DB does not block writes).
8. **Unmapped import values create new lookup rows** rather than collapsing to `lainnya`,
   preserving channel attribution for G3.
9. **`tailscale serve` as a sidecar**, not Caddy's Tailscale plugin — keeps the base
   image stock so self-hosters aren't asked to trust a custom build.
10. **First-run setup screen is MVP, not Phase 2** — it is the difference between
    "deployable by others" and "redeploy per user".

## 18. Explicitly accepted risks

- **All historical rows appear stale** (~1,030 days). Dashboard dominated by old data
  until current applications are logged (C12).
- **No DB-level integrity.** A bug in a write path can persist an orphan or a bad code;
  nothing upstream catches it (§9 is the only guard).
- **Docker Desktop is not yet installed.** M0 is blocked on it; native fallback defined
  in §11.
- **Public mode is exposed with minimal auth.** Passphrase only, no lockout/rate-limit
  until hardening; acceptable for LAN and tailnet, questionable for public — documented
  so the choice is made knowingly.
