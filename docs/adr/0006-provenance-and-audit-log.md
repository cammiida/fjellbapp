# Provenance fields and an append-only audit log

Every aggregate root carries provenance columns — `created_at`, `created_by`,
`updated_at`, `updated_by` (the last two are last-modified-at / last-modified-by) — and
`created_by`/`updated_by` reference `member.id`. Pure join tables
(`task_dependency`, `task_material`, `task_tool`, `task_assignment`, `party_role`,
`attachment_link`, `expense_line`) are exempt. Beyond that snapshot of "who last touched
this," we keep a full change history in a single append-only `audit_log` table:
`(id, entity_type, entity_id, member_id, action, at, changes JSONB)`, where `changes` is a
before/after (or changed-fields-only) diff.

We chose a generic app-layer audit log over the alternatives. **Per-entity `*_history`
tables** and **temporal tables** mean N tables to maintain and migrate — against our
low-ongoing-effort principle. **Postgres triggers** are airtight (a raw `UPDATE` can't
bypass them) but don't naturally know the app user — the "who" — without a per-transaction
session variable, and add DB machinery to maintain. **Event sourcing** is the kind of
"exotic" datastore shape ADR-0002 deliberately rejected. The generic table fits the stack:
all writes already flow through Drizzle/Hono where the acting `member` is in context, and
the audit row is written in the same transaction as the mutation.

**Capture now, view later.** Changes happen from day one, and the earliest records (first
decisions, first big expenses) are the history we will most want — so the table and the
write-path exist from the start even though the audit-viewing UI is deferred. This is the
same discipline as the build log (`taken_at`) and the offline UUIDs in ADR-0004: pay the
near-zero cost of the seam now so retrofitting later isn't a rewrite.

## Consequences

- **Single audited write-path, enforced by lint.** Every insert/update/delete goes through
  one mutation/repository layer that writes the `audit_log` row; an ESLint rule forbids
  calling `db.insert/update/delete` outside that layer. The guarantee comes from the single
  write-path; lint just keeps us on it.
- **Raw SQL stays allowed for reads.** This amends ADR-0002, which permits dropping to raw
  SQL for dependency-graph queries. Those are *reads* (recursive CTEs for readiness/
  blocking) and remain fine. The lint rule targets raw *mutations*, not raw SQL wholesale.
- **Residual gap we accept:** out-of-band writes — manual `psql`, migrations — bypass the
  log and are not audited. For a two-person app where all application writes go through our
  own API, this is an acceptable trade vs. the complexity of triggers.
