# PRD — "let's Try My Luck" · Personal Job Tracker

**Version:** 1.0 (draft)
**Status:** For review
**Product type:** Personal web app (single-user)
**Owner:** Irfan Prima Putra
**Last updated:** 2026-11-25

---

## 1. Summary

"let's Try My Luck" is a personal web application for tracking job applications — from finding a listing, sending a CV, through to knowing the final outcome. Today the data lives in a single Notion database (`let's Try My Luck`) holding 56 companies and 26 applications (Jan–Feb 2024).

This application turns a passive Notion tracker into an active working tool: a clear pipeline, analytics on which channels actually produce results, and follow-up reminders so applications stop hanging without news.

### Problems solved

1. **No funnel visibility.** Today status is free text; it is hard to tell how many are hanging versus rejected versus progressing.
2. **Channels are unmeasured.** From the legacy data: all 7 rejections came via Techinasia, while 2 "Approved" results came from company websites — but that insight is invisible without manual analysis.
3. **Applications hang.** 17 of 26 applications are still "Sent CV" with no follow-up; there is no reminder system.
4. **Messy data schema.** The "Industri" column has the wrong options, "Overal State" is empty everywhere, there are blank rows and typos.
5. **Slow input.** Adding an application in Notion takes many clicks and retyping.

---

## 2. Goals & Non-Goals

### Goals
- **G1** — One place to record, filter, and view every application quickly (input < 30 seconds per application).
- **G2** — A clear, consistent status pipeline replacing free-text status.
- **G3** — An analytics dashboard answering: *which channel is most effective, what is the response rate, which applications need follow-up.*
- **G4** — Seamless migration from Notion with no data loss.
- **G5** — Fast access on desktop and phone (job hunting often happens on the phone).

### Non-Goals — for MVP
- Not multi-user / SaaS. No teams, sharing, or billing.
- Not a job board. Does not pull job listings from the internet automatically.
- Does not create or submit applications to companies automatically.
- No native mobile app (responsive web + PWA in a later phase is enough).
- No AI resume tailoring in MVP (candidate for phase 3).

---

## 3. Persona & Primary Use Cases

**Persona:** A single job seeker (UI/UX Designer) applying to 10–30 roles in an active period, across channels (LinkedIn, JobStreet, Techinasia, company websites, referral).

**Core use cases:**
- **UC1 — Record a new application:** when a listing looks interesting, save it to the wishlist; when the CV is sent, update it to "Applied" with a date and channel.
- **UC2 — Monitor the pipeline:** open the board, see which are still in process, which need action.
- **UC3 — Follow up:** see applications that are "stale" (>14 days without a response) and follow up.
- **UC4 — Evaluate strategy:** see conversion per channel and per company type, and decide where to focus applications.
- **UC5 — Weekly reflection:** see how many applications this week, how many responses came in.

---

## 4. Assumptions

Because scope has not been discussed, this PRD uses the following assumptions (please correct if inaccurate):

- **A1** — Single-user, personal data; authentication is simple enough (passkey/passphrase), not a complex account system.
- **A2** — Responsive web app (desktop-first for tables, mobile-friendly for input and board).
- **A3** — Legacy data is imported once from a Notion export (CSV/JSON), not a live Notion integration.
- **A4** — Cloud hosting (Vercel + Postgres) so it is reachable from anywhere; a self-host alternative is supported via export/backup.
- **A5** — "Approved" in the legacy data means the application passed to the final stage; "Declined" means rejected.
- **A6** — UI language: Indonesian (with an English option later).

---

## 5. Scope

### 5.1 MVP (Phase 1)
- Application CRUD (create, read, update, archive/delete) + quick-add.
- New data model: `Company`, `Application`, `Activity` (log).
- Separate pipeline status and outcome.
- Full **table view** (filter, sort, group, inline edit, bulk action).
- **Board (Kanban)** per pipeline stage, drag-and-drop.
- **Application detail** (all fields + activity timeline + notes).
- **Dashboard** basics: KPIs, funnel, conversion per channel, follow-up list.
- **Import from Notion** (CSV/JSON) with preview and mapping.
- **Export** to CSV/JSON (data ownership).
- Global search + saved filters (saved views).
- Dark mode, responsive.

### 5.2 Phase 2
- Calendar (apply date, interview schedule, challenge deadline).
- Automatic reminders/follow-up (in-app + email/notification).
- Advanced analytics: time-to-response, time-to-offer, weekly trends, per company type/size.
- Document management (CV versions, portfolio) + attachments per application.
- PWA (installable + offline read).
- Multi-view: timeline and card gallery.

### 5.3 Phase 3 (experimental)
- Automatic JD parsing from a job URL (form pre-fill).
- AI: CV/cover letter tailoring suggestions per JD.
- Browser extension "save job" in one click.
- Optional multi-user / read-only sharing.

### 5.4 Out of scope
Mass job scraping, auto-apply, company ATS integrations, billing.

---

## 6. Data Model

### 6.1 Entities

#### `Company`
| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| name | text | unique (case-insensitive) |
| website | text? | |
| industry | enum? | **correct industry taxonomy** (not a rating) — see 6.3 |
| company_type | enum? | Startup, Corporate, IT Service, Design Studio, Unicorn, Agency |
| size | enum? | `1-100`, `100-1000`, `>1000` |
| design_team_size | enum? | `1-30`, `31-50`, `>50` |
| location | text? | |
| notes | text? | |
| created_at / updated_at | timestamptz | |

#### `Application`
| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| company_id | fk | → Company |
| position | text | e.g. "UI/UX Designer" |
| position_type | enum? | UI, UX, UI/UX, Product, UX Research, Interaction |
| job_url | text? | listing link |
| source | enum? | **where the info was found**: LinkedIn, JobStreet, Techinasia, Glints, Kalibrr, Dealls, Website, Referral, Other |
| applied_via | enum? | **how the application was sent**: Website, Email, LinkedIn Easy Apply, ATS, Referral, Other |
| found_date | date? | date the listing was found |
| applied_date | date? | date the CV was sent |
| stage | enum | pipeline stage — see 6.2 |
| outcome | enum | `active`, `rejected`, `offer`, `accepted`, `closed`, `withdrawn` — see 6.2 |
| is_submitted | bool | derived from `applied_date != null` |
| salary_expectation | enum? | `Below`, `Matches`, `Above` expectation |
| rating_branding | enum? | `Unattractive`, `Attractive`, `Very Attractive` |
| rating_product | enum? | same |
| rating_design | enum? | `Not OK`, `OK`, `Very OK` |
| rating_vibe | enum? | same as `Unattractive/Attractive/Very Attractive` |
| overall_score | enum? | `Unattractive`, `Company OK`, `Good Company`, `Dream Company` |
| notes | text? | |
| archived | bool | default false |
| created_at / updated_at | timestamptz | |

> **Design note:** the rating fields (`branding`, `product`, `design`, `vibe`, `overall`) are **optional** and are not used for core filtering. `overall_score` may be manual or computed (average of ratings) — default manual, matching the legacy data.

#### `Activity` (log)
| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| application_id | fk | |
| type | enum | `created`, `applied`, `stage_changed`, `outcome_changed`, `note`, `interview`, `followup`, `reminder` |
| from_value / to_value | text? | for stage/outcome changes |
| note | text? | |
| occurred_at | timestamptz | used for time-to-response |

### 6.2 Canonical Pipeline

Separating **stage** (position in the process) from **outcome** (final result) resolves the "Overal State empty" problem.

**Stage (ordered):**
1. `wishlist` — tracked, not yet applied
2. `applied` — CV sent
3. `screening` — screening / HRD interview
4. `interview` — user / technical interview
5. `challenge` — take-home / challenge
6. `offer` — received an offer

**Outcome (independent of stage):**
- `active` — still in progress
- `rejected` — rejected (at any point)
- `offer` — offer received
- `accepted` — accepted / agreed
- `closed` — listing closed / expired before processing
- `withdrawn` — candidate withdrew

**Mapping from legacy Notion status:**

| Legacy status | stage | outcome |
|---|---|---|
| Not started | wishlist | active |
| Sent CV | applied | active |
| lowongan ditutup | wishlist/applied | closed |
| Proses - Interview HRD | screening | active |
| Proses - Interview User | interview | active |
| Proses - Challange | challenge | active |
| Waiting Interview / Waiting Interview User | interview | active |
| Waiting Challange | challenge | active |
| Done - Interview HRD / Done - Interview User | interview | active |
| Done - Challange | challenge | active |
| Declined | — | rejected |
| Approved | offer | accepted |
| (blank, not submitted) | wishlist | active |

### 6.3 Taxonomy (fixes to the legacy schema)

- **Industry** — a real closed list: `IT Services`, `Software/SaaS`, `E-commerce`, `Fintech`, `Media`, `Education`, `Healthcare`, `Manufacturing`, `Retail`, `Consulting`, `Agency/Design`, `Telecom`, `Other`. (Replaces the legacy "Industri" column whose options were wrongly ratings.)
- **Channel** is split into `source` (where the info came from) and `applied_via` (how it was sent), because in the legacy data "From" and "Send To" overlapped.
- All enums are stored as stable codes (`it_services`, `applied`, etc.) with display labels; no free text for categorical fields.

---

## 7. Features & User Stories

### 7.1 Quick Add Application
**Story:** As a job seeker, I want to add an application in < 30 seconds.
- **AC1** — "+ Application" button on every page; minimal form: Company (autocomplete), Position, Stage.
- **AC2** — A new company is created automatically when absent; an existing company is reused.
- **AC3** — Advanced fields (channel, ratings, notes) are optional / collapsed.
- **AC4** — Can be saved as `wishlist` first and updated later.

### 7.2 Table View
**Story:** I want to see and edit all applications in one dense table.
- **AC1** — Columns match the schema; show/hide and reorder columns.
- **AC2** — Multi-criteria filtering (stage, outcome, source, company_type, size, date range).
- **AC3** — Sort all columns; group by (stage / company_type / source).
- **AC4** — Inline cell editing (select, date, checkbox).
- **AC5** — Bulk actions: change stage/outcome, archive, delete.
- **AC6** — Saved views (e.g. "Applied", "Needs follow-up").
- **AC7** — Virtualized rows; stays smooth for hundreds of applications.

### 7.3 Board (Kanban)
- **AC1** — One column per `stage`; drag-and-drop changes the stage and records an `Activity`.
- **AC2** — Compact card: company, position, date, outcome badge, application age.
- **AC3** — Active filters apply to the board.

### 7.4 Application Detail
- **AC1** — All fields + inline edit.
- **AC2** — Chronological `Activity` timeline (created → applied → each stage change → notes).
- **AC3** — Links to job_url and the company website.
- **AC4** — Quick action buttons: "Mark rejected", "Move stage", "Add note", "Set follow-up".

### 7.5 Dashboard
- **AC1** — KPIs: total tracked, applied, active, rejected, offers, accepted.
- **AC2** — Funnel: wishlist → applied → screening → interview → challenge → offer.
- **AC3** — **Conversion per channel** (bar chart): response rate and offer rate per `source` / `applied_via`.
- **AC4** — Distribution per company_type and size.
- **AC5** — **Follow-up list**: applications in `applied`/`active` with `applied_date` > 14 days and no newer activity.
- **AC6** — Weekly activity (applications per week).

### 7.6 Import from Notion
- **AC1** — Upload a Notion CSV export (and optionally JSON).
- **AC2** — Column mapping screen (auto-detect based on Indonesian headers: Company, Posisi, From, Send To, Status, etc.).
- **AC3** — Preview + validation; show failed rows and the reason.
- **AC4** — Apply normalization: trim whitespace, drop blank rows, fix enum capitalization, map legacy status → stage/outcome (see 6.2), fix common typos (Product Desginer → Product Designer, etc.).
- **AC5** — Idempotent: re-importing does not duplicate (dedupe by company+position+applied_date).

### 7.7 Export & Backup
- **AC1** — Full CSV & JSON export (Company + Application + Activity).
- **AC2** — Automatic daily backup (30-day retention) server-side.

### 7.8 Global Search & Filter
- **AC1** — Search by company/position/notes.
- **AC2** — Keyboard shortcuts (`/` for search, `n` for new).

### 7.9 Settings
- **AC1** — Change the enum lists (channel, industry, type) without deploying.
- **AC2** — Light/dark theme.
- **AC3** — Language (ID default).
- **AC4** — Follow-up threshold configuration (default 14 days).

---

## 8. Analytics — Metric Definitions

So the numbers stay consistent, metrics are defined explicitly:

- **Applied** = number of `Application` rows with `applied_date != null`.
- **Response rate** = (applications with ≥1 inbound activity other than "applied", i.e. screening/interview/challenge/offer/rejected) ÷ Applied.
- **Interview rate** = applications that ever reached stage ≥ `screening` ÷ Applied.
- **Offer rate** = applications with outcome `offer` or `accepted` ÷ Applied.
- **Conversion per channel** = the metrics above, computed per `source` value (and/or `applied_via`).
- **Median time-to-first-response** = median(`occurred_at` of first response − `applied_date`).
- **Stale application** = `outcome = active`, `stage ∈ {applied}`, and `now − applied_date > threshold` (default 14 days) with no newer activity.
- **Weekly cadence** = number of `applied_date` per ISO week.

> Baseline insight from the legacy data: Techinasia → 7/9 rejected; Website → 2 approved out of 7; LinkedIn → 6 applications with no result yet. The dashboard must be able to reproduce insights like these automatically.

---

## 9. UX & Screen Flow

Primary navigation (sidebar): **Dashboard · Applications · Board · Companies · Insights · Settings**.

**Key flow — "Apply now":**
1. Click `+ Application` (or `n`).
2. Type the company name → autocomplete / "create new".
3. Fill position + source + applied_date (defaults to today).
4. Save → toast appears + card enters the "Applied" Board column.
5. Optional: open detail, add job_url and notes.

**Follow-up flow:**
1. Dashboard shows "Needs follow-up (N)".
2. Click → filtered table.
3. Select one → detail → "Add follow-up note" (recorded as an `Activity`).

**UX principles:**
- Keyboard-first for power users; touch-friendly for mobile.
- Informative empty states (including an import button when there is no data).
- Batch operations do not over-confirm, but delete always confirms/undoes.

---

## 10. Non-Functional Requirements

- **Performance:** TTFB < 500ms (P95), 500+ row table smooth (virtualized), dashboard load < 1.5s.
- **Responsive:** mobile (≤640), tablet, desktop breakpoints; board scrolls horizontally on mobile.
- **Accessibility:** WCAG 2.1 AA — contrast, keyboard focus, ARIA labels, full keyboard navigation.
- **Privacy & security:** personal data; access via authentication (passkey/passphrase + httpOnly session cookie); encryption in transit (HTTPS) and at rest in the DB; no third-party tracking.
- **Reliability:** automatic daily backup; transactional import (all-or-nothing per batch).
- **Data portability:** full export at any time (anti-lock-in).
- **i18n-ready:** UI strings separated from code; Indonesian default.
- **Observability:** error logging + basic uptime (optionally Sentry).

---

## 11. Stack & Architecture (proposal)

> **Superseded.** This section describes the earlier plan (Next.js fullstack + Drizzle). The
> authoritative stack is in `docs/superpowers/specs/2026-11-26-job-tracker-design.md`:
> Go + strict N-layer backend, Next.js frontend, PostgreSQL, containerized behind Caddy.

- **Frontend/Backend:** Next.js (App Router) + TypeScript.
- **UI:** Tailwind CSS + shadcn/ui; lucide icons.
- **Table:** TanStack Table (virtualized).
- **Kanban:** dnd-kit.
- **Charts:** Recharts.
- **Validation:** Zod (shared client/server schema).
- **DB:** PostgreSQL (Supabase/Neon) + **Drizzle ORM**; versioned migrations.
- **Auth:** single-user — passkey (WebAuthn) or passphrase + session; no public registration.
- **Deploy:** Vercel (app) + managed Postgres; self-host alternative (Docker) with local Postgres.
- **Testing:** Vitest (unit), Playwright (E2E for core flows: quick-add, import, stage drag).
- **Domain structure:** `companies`, `applications`, `activities` + `import`, `analytics` modules.

> Lighter alternative (if a cloud DB is undesirable): SQLite (Turso/libSQL) + Drizzle — suited to single-user, cheaper, but less ideal for serverless Vercel without Turso.

---

## 12. Milestones (no time estimates)

- **M0 — Foundation:** scaffold Next.js + TS + Tailwind + shadcn, DB schema + migrations, single-user auth, CI.
- **M1 — Core CRUD:** Company & Application CRUD, quick-add, detail + Activity log, pipeline enums.
- **M2 — Views:** Table (filter/sort/group/inline edit) + Board (drag-drop) + search.
- **M3 — Import:** Notion CSV import with mapping, normalization, dedupe, preview.
- **M4 — Dashboard & Insights:** KPIs, funnel, conversion per channel, follow-up list.
- **M5 — Polish:** export/backup, settings, dark mode, a11y, E2E tests.
- **M6 — (Phase 2):** calendar, reminders, documents/CV, PWA, advanced analytics.

**Definition of "MVP done":** legacy Notion data 100% imported, CRUD + table & board working, dashboard showing conversion per channel, and data exportable.

---

## 13. Risks & Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| Dirty legacy data (blank rows, typos, combined statuses) | Messy import | Normalization screen + preview + dedupe; documented status mapping (6.2) |
| Over-engineering (AI/scraping too early) | MVP slips | Lock scope to Phase 1; AI/extension features in Phase 3 |
| Single-user auth assumed "safe enough" | Personal data leaks | Passkey/passphrase + httpOnly cookie + HTTPS; no public URL |
| Enums too rigid | Hard to adapt | Editable enums in Settings |
| Notion dependency | Lock-in | One-time import + full export; no live sync |

---

## 14. Open Questions

1. **Hosting:** cloud (Vercel + Postgres) or self-host local/VPS? (affects DB and auth choices)
2. **Multi-user?** Stay personal single-user, or later make it shareable (read-only)?
3. **AI/scraping:** is JD parsing from a URL and CV tailoring needed near MVP, or deferred to Phase 3?
4. **Meaning of "Approved"** in the legacy data: passed screening, received an offer, or hired? (determines the final mapping)
5. **Rating fields** (branding/product/design/vibe/overall): still used for decisions, or archival only?
6. **Primary target platform:** desktop first or mobile-first?

---

## 15. Success Metrics (for this product itself)

- **Fast input:** median time to add one application < 30 seconds.
- **Adoption:** all 56 legacy companies imported with no data loss (100% target).
- **Data consistency:** ≥ 90% of applications have `stage`, `outcome`, and `source` filled.
- **Follow-up coverage:** 100% of stale applications appear in the follow-up list.
- **Insight available:** dashboard can show conversion per channel within < 1 minute of opening the app.
