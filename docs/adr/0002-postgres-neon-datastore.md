# Postgres (Neon) as the datastore

We're using a standard relational database — Postgres, hosted on Neon — rather than
anything more exotic (document store, graph DB, etc.).

The app is overwhelmingly about filtering and totalling: what's due this week, what's
left in the kitchen, where are we over budget. That's exactly what relational databases
do best, with very little to maintain. The one genuinely "connected" part — task A can't
start until task B is done — is a dependency graph that Postgres handles cleanly (a join
table, recursive CTEs where needed) without reaching for a graph database.

Neon specifically: serverless Postgres with a generous free tier and an HTTP/serverless
driver that is the supported path on Cloudflare Workers (see ADR 0003). Access is via
Drizzle ORM (schema-as-code, generated migrations), which keeps SQL visible and lets us
drop to raw SQL for the dependency-graph queries when needed.

**Amended by ADR-0006:** the "drop to raw SQL when needed" allowance now applies to *read*
queries only (the recursive-CTE dependency-graph reads above). All *mutations* go through a
single audited write-path so they can be recorded in the `audit_log` — a lint rule forbids
`db.insert/update/delete` outside that layer.
