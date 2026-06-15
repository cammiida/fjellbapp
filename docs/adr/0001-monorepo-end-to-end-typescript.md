# Monorepo with end-to-end TypeScript

We're building this as a single monorepo with a React frontend and a Hono backend,
both in TypeScript, sharing a common types/schema package.

One of us is strongest in Python, the other in Kotlin/TypeScript, so a Python (FastAPI)
or Kotlin backend was genuinely on the table. We chose TypeScript on both ends anyway:
the frontend is React/TS no matter what, and keeping the backend in the same language
lets us define a type once (a `Task`, a `Material`) and use it across the API boundary —
no schema drift, far less boilerplate. For a two-person side project optimising for low
ongoing effort, that single-language productivity win outweighed leveraging the deepest
existing expertise. The cost we accepted: the Python-strong person learns more TypeScript
(mitigated by the fact that both of us already touch TS through React).

The API boundary uses Hono's built-in RPC client (still plain HTTP underneath, just
type-safe), and the database layer is Drizzle, so types flow from the DB schema all the
way to the React components.
