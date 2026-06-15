# Online-only now, light-offline-capable later

The app is a normal online client-server web app today. We are NOT building offline-first
(local DB / sync engine / CRDTs) — the usage pattern is "plan at home with signal, document
as you go," and full offline-first is far more complex than that warrants.

However, the cabin has patchy connectivity, so we may later want "light offline": reading
the task checklist offline and queuing a few writes that sync on reconnect. The two things
that make that retrofit cheap-vs-painful are decided NOW, even though no offline code exists
yet — this is the surprising part a future reader should understand:

1. **Client-generated UUIDs.** Entities get their ID from the client at creation, not from
   the database. An offline-created task already has its real, final ID, so syncing needs
   no temporary-ID reconciliation. Retrofitting this later would mean re-plumbing every
   table and foreign key.
2. **Idempotent writes.** Mutations are safe to replay (upsert semantics / idempotency
   keys), so a sync outbox can retry naively without creating duplicates.

With those two conventions in place, "light offline" later is a focused feature (a service
worker cache + a client-side outbox queue), not a rewrite. Making the app a PWA at all
(installable, caches last screen) is trivial and can be added any time, independent of this.
