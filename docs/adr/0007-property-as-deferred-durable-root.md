# Property as a deferred durable root above Project

A **Property** (the cabin itself — land + building) is the intended durable root that will sit
*above* Project: one Property has many Projects over its life (the initial renovation, later
remodels, annual maintenance). Durable things belong to the Property — **Rooms, Requirements
(slot bounds), installed fixtures, owned Tools, Parties/relationships, Members & access, and the
build-log Attachments**. Effort-scoped things stay on Project — **Phases, Tasks, Dependencies,
Milestones, Budget Categories, Expenses** (lifetime cabin spend becomes a cross-project rollup).
Project also gains a `kind` (`renovation` | `maintenance`) so upkeep is a bounded maintenance
Project under the same Property.

**We are NOT building this now — we record the shape and defer the build**, the same move as
ADR-0004 (offline) and ADR-0005 (multi-tenancy machinery). The deferred durable concepts (Room,
Requirement, household inventory) were already straining against Project precisely because they
outlive any single effort; Property is the home they were missing. But a second project is years
out, and our guiding principle is to not over-build.

## Why this is cheap to retrofit (the surprising part)

ADR-0005 paid for the `project_id` boundary up front because adding a tenant key to *untagged*
data is expensive — you must decide which tenant each orphan row belongs to. Property is the
**opposite** case and is therefore safe to defer:

1. **The key already exists.** Every row has `project_id`. Inserting Property *above* a root that
   has exactly one child makes `property_id` derivable — all rows map to the single Property
   unambiguously. The backfill is mechanical, not a judgment call.
2. **The most durable concepts aren't built yet.** Room, Requirement, inventory are all deferred;
   when built they get `property_id` from birth — zero migration.
3. **No duplication trap.** Creating "Ola the electrician" separately under two Projects only
   happens once a *second* Project exists — which is the same day Property would be introduced, so
   the duplicates never get created.

The one real cost paid on that future day: re-parenting the durables built now (Party, Tool,
Attachment, Membership) from `project_id` to `property_id` — FK changes plus query updates on busy
core tables. Bounded and unambiguous, not a rewrite.

## Consequences

- **Seam-keeper rule:** any deferred durable concept (Room, Requirement, inventory) is built
  `property_id`-scoped from birth, never `project_id`. This keeps the future migration to only the
  durables built before Property exists.
- **Access:** when built, Membership attaches at the Property level (you're a member of the
  *cabin*, you see all its projects) — which also maps more naturally onto better-auth's
  organizations plugin (org = Property) than the Project-level mapping ADR-0005 sketched.
- **Amends ADR-0005:** Project is the top-level root *today*, but is no longer the intended
  permanent root — Property will own durable data; Project will own only effort-scoped data.
- **Recurrence stays deferred:** "re-stain every 2 years" needs a schedule that generates task
  instances. That is a separate future feature (a Property-level `recurring_task`), independent of
  this boundary; until then a maintenance Project copies last year's checklist.

## Considered options

- **Property** (chosen) — captures the durable owned asset across its whole life, including
  post-build maintenance and future remodels.
- **Site** — construction-native (the build "site"/plot), but construction-phase flavored when
  this level is meant to outlive the build, and already on Project's avoid list as a tenancy-word
  synonym. Close call; the name is cheap to change while nothing is built.
- **Build it now** — rejected: over-builds for a second project that is years away, and the
  retrofit is cheap (see above), so there is no boundary worth paying for early here.
