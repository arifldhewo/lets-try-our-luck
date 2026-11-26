# AGENTS.md

Instructions for AI agents working in this repository.

---

## Language rules

**Repository standard: English.**

Everything committed to this repository is written in English:

- Documentation — `README.md`, `PRD.md`, anything under `docs/`
- Commit messages and branch names
- Code comments and docstrings
- Identifiers, function names, table and column names
- API error messages and log output
- File and directory names

**Conversation with the owner: Indonesian.**

All chat communication stays in Indonesian — explanations, questions, status reports, proposals, and design discussion.

This split is deliberate and must not be collapsed:

- Do not translate repository artifacts into Indonesian.
- Do not switch the conversation into English.
- When writing a document, write it in English even though the discussion that produced it was in Indonesian.

---

## Product context

`let's Try My Luck` is a single-user job application tracker. It replaces a Notion database
(56 companies, 26 applications) with a pipeline, a channel-effectiveness dashboard, and a
follow-up list.

Authoritative documents:

- `PRD.md` — product requirements. Sections 1–10, 12, and 14 are normative.
- `docs/superpowers/specs/2026-11-26-job-tracker-design.md` — technical design. **This
  supersedes PRD §11 (Stack & Architecture)**, which still describes the earlier
  Next.js-fullstack + Drizzle plan.

---

## Locked decisions — do not reverse without explicit approval

### Database

- **No PostgreSQL enum types.** Categorical values are `text`.
- **No foreign key constraints at all.** Relationships are plain `uuid` columns.
- **Soft deletes only.** `archived = true`; never a hard `DELETE`; never `ON DELETE CASCADE`.

Reason: DB-level constraints block legitimate intermediate states during partial transitions.

Consequence that follows from this: **integrity is enforced in Go**, not by the database.
Every write path must call the validation package. Reads must tolerate orphaned rows
(`LEFT JOIN` + `COALESCE`). This is load-bearing — there is no DB fallback.

### Architecture

- Go backend, strict N-layer with dependency inversion:
  `cmd/` → `adapter/` → `usecase/` → `domain/`.
- Imports point inward only. `domain/` imports nothing from this project.
  `usecase/` never imports `net/http` or `database/sql`.
- Repository interfaces live in `domain/`, implementations in `adapter/postgres/`.
- CQRS is deferred, not adopted.
- `stage` and `outcome` are ordered constants in `domain/pipeline.go`, **not** editable
  in Settings — funnel math depends on fixed ordering.

### Scope

- MVP is exactly two features: **dashboard overview** and **manual application management**.
- No job-board scraping. Playwright is for E2E tests only.
- Board/Kanban, calendar, reminders, email ingestion, PWA are Phase 2.

### Deployment

- Containerized from day one, single origin behind Caddy (no CORS, no cross-site cookies).
- Must be self-hostable by others with **no code changes** — configuration only.
- Tailscale is an optional compose override, never a requirement.

---

## Working conventions

- Documentation lives in English; conversation lives in Indonesian.
- Never commit secrets, personal passphrases, or the owner's Notion data.
- Soft-delete means every read query filters `archived = false`.
- When a decision here conflicts with `PRD.md`, the design document wins; when either
  conflicts with an explicit instruction from the owner, the owner wins.
