# Multi-tenant data shape from day one, machinery deferred

> **Amended by ADR-0007:** Project is the top-level root *today*, but is no longer the intended
> permanent root. A durable **Property** (the cabin) will sit above Project and own durable data
> (rooms, requirements, owned tools, parties, access, build log); Project will own only
> effort-scoped data. That insertion is a cheap, unambiguous retrofit (one Project → one Property),
> so it too is deferred. Read this ADR with 0007.

Everything in the database is scoped to a top-level **Project** (the renovation of one
cabin): the aggregate-root tables carry a `project_id`, and people are linked to a Project
through a **Membership** that carries a role (owner / editor / viewer). We seed exactly
one Project today.

`project_id` lives on the entities we query directly (Party, Phase, Task, Material, Tool,
Budget Category, Expense, Attachment, Milestone, Note, Membership), NOT on the pure join
tables (`task_dependency`, `task_material`, `task_tool`, `task_assignment`, `party_role`,
`attachment_link`) — those inherit tenancy through their parent, where a redundant `project_id`
would only add drift risk. If we later enable Postgres row-level security (RLS), we can add
`project_id` to the join tables then, since clean RLS policies want it on every table.

We may, far in the future, want other people to track their own cabins. The only part of
"multi-tenancy" that is genuinely expensive to retrofit is this data-model boundary — once
data exists without a tenant key, adding one is a painful migration. So we pay the near-zero
cost of `project_id` + a Membership join table now (we'd want roles for collaborators
anyway) and explicitly DEFER everything else: public signup, billing, email invite flows,
tenant admin UI, and row-level-security hardening. Those are additive and can be built the
day we actually open up.

When that day comes, better-auth's organizations plugin (organizations + members + roles +
invitations) maps onto our Project/Membership model and can become the multi-tenancy layer
rather than us building it by hand.
