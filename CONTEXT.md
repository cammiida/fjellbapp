# Cabin Renovation Tracker

The shared language for an app that two owners use to plan, execute, and document
the renovation of a mountain cabin in Norway — and to maintain it day-to-day afterward.

## Language

### People & ownership

**Project**:
A bounded effort on one cabin. The top-level owner of all data *today* — every other thing
(task, material, expense…) belongs to exactly one Project. There is only one today, but the
model allows more. A Project carries a **`kind`** (`renovation` | `maintenance`): a renovation is
a bounded build with Phases and Milestones; a maintenance Project is a bounded upkeep period
(e.g. a year) on the same cabin, usually phase-less (recurrence — "every 2 years" — is deferred).
_Intended parent (deferred — see [[property]] and ADR-0007):_ a durable **Property** (the cabin
itself) will sit ABOVE Project as the real owner of durable things; Project will then own only the
effort-scoped things. Until that's built, Project stays the top-level root.
**Scoping rule:** `project_id` lives on the aggregate roots you query directly (Party, Phase,
Task, Material, Tool, Budget Category, Expense, Attachment, Milestone, Note, Membership). Pure
join tables (`task_dependency`, `task_material`, `task_tool`, `task_assignment`, `party_role`,
`attachment_link`) omit it and inherit tenancy through their parent; it can be added later only
if Postgres row-level security is enabled.
_Avoid_: Workspace, account, tenant, site.

**Property** (the cabin) — _DEFERRED; intended durable root above Project, see ADR-0007_:
The physical, durable thing that persists across every effort — the land + cabin. It will sit
ABOVE Project: one Property has many Projects (the renovation, later remodels, annual
maintenance). Durable things belong to the Property, not to any one Project — **Rooms,
Requirements (slot bounds), installed fixtures, owned Tools, Parties/relationships, Members &
access, and the build-log Attachments**. Effort-scoped things stay on Project — **Phases, Tasks,
Dependencies, Milestones, Budget Categories, Expenses** (lifetime cabin spend becomes a
cross-project rollup). Deferred because inserting a parent above a single-Project root is a cheap,
unambiguous retrofit (one Project → one Property backfill), UNLIKE adding a tenant key to untagged
data (ADR-0005). Seam-keeper: build the deferred durable concepts (Room, Requirement, inventory)
with `property_id` from birth, never `project_id`.
_Naming:_ chose **Property** over **Site** — "Site" is construction-phase flavored and was on
Project's avoid list, while this level outlives the build; a close call.
_Avoid_: putting durable, cross-project things (a Room, an owned Tool) under Project.

**Party**:
Anyone the Project has a relationship with — stored in ONE `party` table with a `kind`
(`person` | `organization`). `id`, `kind`, and `name` are real columns (queried/sorted);
the rest of the attributes (email, phone, org-number…) live in a `metadata` JSONB. A Party
is the thing work can be assigned to and that supplies goods. This is the single most
identity-shaped (hard-to-reverse) decision in the model.
_Avoid_: separate Person/Organization/Contractor/Supplier tables — those are KINDS or ROLES
of Party, not their own entities.

**Person / Organization**:
The two `kind`s of Party. A Person is an individual; an Organization is a company/shop.
Modelled in one table, not two (they barely differ). A Person works at / is a contact at an
Organization via a **self-referential FK** `employer_party_id → party.id` (one org per
person for now; a join table comes later if a person needs several). "Employer must be an
organization" is enforced in app validation, since one table can't enforce it.

**Role**:
What a Party *does* — `contractor`, `supplier`, `collaborator` — recorded as a tag
(`party_role`), NOT as a separate entity. A single Organization can be both a contractor and
a supplier. Contractor/Supplier are roles, never tables.
_Avoid_: Contractor table, Supplier table.

**Member** (a.k.a. User):
An app *login* — credentials + an app-level role. References the person-Party it belongs to
(`member.party_id`, 1:1), so "a user IS a person" without duplicating identity: Member holds
*access*, Party holds *who they are*. Every Member is a person Party; not every Party is a
Member (most contractors' staff never log in).
_Avoid_: Collaborator (vague), conflating login (Member) with identity (Party).

**Membership**:
The link between a Member and a Project, carrying that member's role on it
(owner / editor / viewer). Generalizes "us + a few collaborators" to "many cabins" later.

**Assignment**:
A Task is assigned to one or more Parties (M2M `task_assignment(task_id, party_id)`). So work
can go to a Member, to a specific contact (a Person), or to a whole shop (an Organization, or
both its people) — your "both are responsible, don't force one" case is assigning the org.

### The build (from the existing plan — to be sharpened as we go)

**Task**:
A single piece of work, with status, dates, and a cost estimate, belonging to one Project and
*optionally* to one Phase (renovation tasks sit in a Phase; maintenance tasks have none — see Phase).
The heart of the app. (Room is deferred — see Room.) May depend on other Tasks (see
Dependency) and may be assigned to one or more Parties — a Member, a contact (Person), or a
whole shop (Organization). See Assignment.

**Task dependency**:
A finish-to-start link between Tasks: "B can't start until A is done" (A blocks B), stored in
`task_dependency(blocking_task_id, blocked_task_id)`. The links form a DAG. When a Task's dates
change the app *highlights* the affected downstream Tasks — but the owners adjust dates
themselves. The app does NOT auto-reschedule (no date math / critical-path engine).
_Naming_: shares the `task_*` prefix with `task_material` / `task_tool` (the other things a Task
needs). It keeps the name "dependency" rather than `task_task` because that reads clearly.

**Task requirement** (`task_material`, `task_tool`):
What a Task needs to proceed, beyond prerequisite tasks. `task_material(task_id, material_id,
quantity)` — required materials, with a per-task quantity (12 m² of cladding for this task).
`task_tool(task_id, tool_id)` — required tools (no quantity; a Tool is reused). These also serve
costing and the "cluster jobs into a rented tool's window" goal.

**Readiness**:
A derived signal, not a stored field, with two tenses. *Ready now*: a Task is ready when its
prerequisite Tasks are done AND its required Materials are received AND its required Tools are
available within the Task's date window. *At risk* (forward-looking): a Task is at risk when a
required Material's `expected_arrival` falls after the Task's `start_date`, or a required Tool has
no `tool_availability` window covering it — i.e. it won't be ready in time. Unifies
`task_dependency` + `task_material` + `task_tool` + `tool_availability` into one view. The same
mechanism behind blocked-task highlighting; the at-risk tense is the early-warning the
remote-cabin, low-effort thesis depends on.

**Phase**:
A natural stage of the build (site prep, foundation, framing, rough-in, insulation,
finishing). A Task belongs to *at most one* Phase — **optional**, not mandatory: renovation
tasks sit in a Phase, but a maintenance Project has no phases, and a not-yet-sorted task can sit
phase-less (an "inbox"). Phase partitions the work *within a renovation*; it is not the same as a
Milestone (a Phase is the work's mandatory-when-present home; a Milestone is an optional,
overlapping date-target laid over many tasks — see Milestone).

**Room** (a.k.a. Area) — _DEFERRED, not built initially_:
A physical area of the cabin (kitchen, loft, bathroom, deck). Dropped from the initial model:
the whole cabin is the single Project, and per-room rollups ("what's left / cost in the
kitchen") may never be needed. Cheap to add later as a nullable `room_id` on Task/Material/
Expense, with optional backfill. Until then: a Task relates to a Phase only, and any location
need lives as free text. Do NOT smuggle rooms into Phase (different axis) or task titles
(data-in-a-string).

**Material**:
Something *incorporated once* into the build (it goes in and becomes part of the cabin).
The dividing line from a Tool is "incorporated once" vs "reused across jobs" — NOT literally
"used up" (a window isn't used up, but it's still a Material). Has two kinds:
- **Bulk** — measured by quantity + unit of measure (cladding by m², lumber by length).
  One row, a number.
- **Fixture** — tracked individually, one row per physical item (each window, the stove),
  so a specific item can be identified ("the loft north window") and looked up years later.
Common fields: name, cost, supplier, order status. Lifecycle:
needed → ordered (set `expected_arrival`, the date the supplier quotes) → received (set
`received_at`) → used/installed. `expected_arrival` is stored directly (suppliers quote a date,
not a lead-time) and is null until ordered — it drives the Calendar and the forward-looking
*at-risk* readiness signal: a Task is at risk when a required Material's `expected_arrival` falls
after the Task's `start_date`. _Seam:_ add `lead_time_days` later if pre-order forecasting
("if I order today, when does it arrive?") is ever wanted.
Variable per-item attributes (a window's U-value, dimensions, glazing) live in a `specs`
field validated per category, not as columns.
_Avoid_: Tool (reused, not incorporated — kept deliberately separate).

**Tool**:
Something *reused across many jobs* rather than incorporated into one (saw, excavator).
Carries an **acquisition mode**: owned / rented / borrowed. The Tool row is the stable
*identity* (name, supplier, default mode) — there is exactly one row per physical tool, so the
"tools we don't yet own, buy the saw once" list never double-counts. Buying or renting a Tool
can generate an Expense, the same way a Material purchase does.

**Tool availability** (one Tool, zero-or-many windows):
A hold window on a Tool — `tool_availability(id, tool_id, from, to, mode)`. Owned tools are
always available and have NO window. **Rented/borrowed** tools have one window per hold period,
so the *same* excavator rented in spring AND again in autumn is two windows on one Tool row (NOT
two Tool rows, NOT one stretched window that falsely reads as available all summer). The
clustering goal ("cluster jobs into a rented tool's window so we don't pay twice / don't miss
returning a borrowed tool") and the Calendar both read from `tool_availability`.
_Avoid_: an inline single window on the Tool row (can't represent re-renting).

**Requirement** — _DEFERRED, not built initially_ (concept agreed; build later):
A constraint attached to a *slot/location* in the cabin (the "hole to be filled"), e.g.
"loft north opening: U-value ≤ 1.0, egress-capable". It is the durable **bound** the place
demands, and it OUTLIVES any item that fills it — replace a 0.8 window and the requirement
is still ≤ 1.0, never 0.8. A fixture Material **satisfies** a Requirement; the fixture's
actual `specs` must meet the Requirement's target bound (actual vs. bound are distinct).
_Avoid_: storing the requirement on the fixture itself (it would vanish when the fixture is
replaced).

**Specs vs. target**:
A fixture's `specs` are its *actual* measured attributes (this window IS 0.8). A
Requirement's target is the *bound* (must be ≤ 1.0). Satisfying = actual meets bound.

**Supplier**:
A *Role* a Party plays — where Materials and Tools are bought from. Not its own table:
Material/Tool carry a `supplier_party_id → party`, and that party has the `supplier` role.
The same Party may also be a contractor.
_Avoid_: a Supplier table (it's a Role, see Role).

### Money

**Estimate**:
A *planned* cost figure, attached to a Task or Material (and, for labour, derived from a
Contractor's rate). A forecast only. An Estimate is NEVER counted as money spent — it exists
to compare against actuals (variance).
_Avoid_: treating an estimate as an expense.

**Expense**:
A record of *actual* money out — the single source of truth for "spent." Tagged to ONE Budget
Category, carries the total `amount` and `paid_at`. Created deliberately when money moves (no
auto-generation), which is what prevents double-counting. Carries a `status` that today is always
`actual`; the seam is deliberate so `committed` / `estimated` states can be added later with no
migration.
_Avoid_: Cost (ambiguous — say Estimate for planned, Expense for actual).

**Expense line** (M2M, polymorphic):
What an Expense paid for — `expense_line(expense_id, entity_type, entity_id, amount?)`, mirroring
`attachment_link`. One Expense links to MANY things, so a single supply-run receipt buys 15
Materials in one record (the common case for a remote cabin), and a line's optional `amount`
apportions the total across items or categories. Deliberately a link table, NOT nullable
`task_id`/`material_id`/`tool_id`/`party_id` columns on Expense — that one-of-each pattern is the
single-owner pattern in disguise (the same reason `attachment_link` is polymorphic). A
contractor's labour invoice is an Expense line targeting a Party (and usually a Task).
_Avoid_: one-nullable-FK-per-kind on the Expense row.

**Budget Category**:
A money bucket and the dimension the budget rolls up by (e.g. "Windows & doors",
"Plumbing", "Labour", "Tools", "Permits"). Its OWN dimension — deliberately not the same as
Phase or Room, because cost breakdowns cross both (labour and windows span many phases and
rooms). Every Expense and Estimate is tagged to one Budget Category; Phase and Room stay
separate axes you can also slice spend by.

**Budget**:
A target amount set per Budget Category. "Where are we over?" = sum of actual Expenses vs.
the Budget for that category, with Estimates shown alongside as forecast.

**Labour estimate**:
A Task's planned labour cost — either a *fixed quote* (fastpris) or *rate × expected
hours/days* (timepris, from the Contractor's rate). It is an Estimate (forecast). The actual
labour cost is a Labour-category Expense (the Contractor's invoice) linked to the Task and
Contractor. No hour/timesheet tracking today; a seam is left so optional TimeEntry rows
(hours × rate, incl. own sweat-equity) can be added later without rework.

### Documentation

**Attachment** (photos & documents):
A pointer to a file in storage (R2) plus metadata: `kind` (image / document), caption,
uploader, and a **`taken_at`** date (so images line up chronologically into the build log
regardless of upload time). Photos and documents are ONE mechanism, not two. The bytes live
in storage; the database only records the pointer + metadata.

**Attachment link** (M2M):
One Attachment can be linked to many things — a Task, a Material, a Tool, an Expense
(receipt), a Note — via a single polymorphic link table `(attachment_id, entity_type,
entity_id)`. NOT nullable FK columns on the Attachment (that only allows one-of-each and is
the single-owner pattern in disguise). Deleting an entity removes its links, never the
Attachment itself (the file survives in the record). Seam for later: a link may gain an
optional coordinate + direction onto a floor-plan Attachment ("where this photo was taken
from"), added with no rework.

**Build log**:
A *view*, not a stored thing — all image Attachments across the Project ordered by
`taken_at`. The visual record of how the cabin was actually constructed.

**Calendar**:
A *view*, not a data entity — it aggregates things that already carry dates: Task
start/due dates, Milestones, and tool availability/return windows (rented & borrowed).
In-app only initially. Seams for later (additive, no rework): a read-only iCal/ICS feed,
then optional two-way Google Calendar sync (mapping app entities ↔ events via their stable
UUIDs).

**Milestone**:
A date that really matters (weathertight before winter, move-in) — its own **time-axis target**,
deliberately orthogonal to the Phase/Task work breakdown the same way **Budget Category** is
orthogonal to Phase on the money axis (a milestone like "weathertight" spans several phases:
roof, windows, sealing). NOT modelled as a task-with-no-work (it has no assignee, cost, or
effort). A Milestone is a *marker*, not work.
- **Target date** — the deadline you steer toward (always present).
- **Contributing tasks** — M2M `milestone_task(milestone_id, task_id)`: the work whose completion
  fulfils it. (The motivating milestones span *many* tasks, so this is M2M, not a single FK.)
- **Forecast / on-track** — *derived, never stored*: `forecast = max(due_date)` of contributing
  tasks; *at risk* when `forecast > target`. Same forward-looking machinery as Task [[Readiness]],
  rolled up — this is the "in progress / on-track" signal, computed not flagged.
- **Achieved** — *manual* `achieved_at` (nullable date): null = not reached, set = reached on that
  date (which may differ from target and forecast, and is what the build-log history remembers).
  Human judgment wins — you may mark it reached with a contributing task still open (the UI
  soft-warns, no hard constraint), and external milestones with loose/no task links just get ticked.
Appears on the Calendar as a marker. _Avoid_: a stored status enum that competes with the derived
on-track signal (it would drift from the tasks); modelling a milestone as a Task.
_Distinct from_ the deferred **Permit/Inspection** (a *hard* blocker that gates work) — a Milestone
is a *soft* target you steer toward, it does not block.

**Permit / Inspection** — _supporting piece, add when needed_:
A permit type or inspection with dates and approval status; often *gates* other work (acts
like an external Dependency on Tasks).

**Note**:
A short journal entry capturing a decision and its reasoning, so the "why" survives. Can
carry Attachments, and has a **mandatory polymorphic target** (`entity_type` + `entity_id`,
NOT NULL) so a note always sits next to the thing it explains — the loft-window choice on that
Material, a resequencing call on a Task/Phase, a supplier switch on a Party. A note "about the
whole build" is not a null target; it explicitly targets the **Project** (`entity_type =
project`). A *single* target pair per note (not one-FK-per-kind).
_Note:_ a Note also carries `project_id` for **tenancy** (it's an aggregate root). When the
target IS the Project, the same id appears in both — not redundant: `project_id` = which tenant
owns the row, target = what the note is about. Different roles, same value.
_Seam:_ promote to an M2M `note_link` table if a note ever needs to span several subjects.
