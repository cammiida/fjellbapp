# Cloudflare Workers + R2 for hosting and files

The app deploys as a single Cloudflare Worker that serves both the built React app
(via Workers static assets) and the Hono API. Photos and documents live in Cloudflare R2.

We considered a single always-on Node service (Railway/Fly/Render), which is the
conceptually simplest runtime with zero edge-runtime constraints. We chose Cloudflare
because one of us already knows the platform — and on a project whose guiding principle
is low ongoing effort, working on a familiar platform beats a marginally simpler but
unfamiliar one. It also keeps the app and the file storage (R2) under one vendor, and a
single Worker serving both React and the API gives us one deploy, one URL, and no CORS.

R2 specifically was chosen over S3/Vercel Blob because it has no egress fees — we browse
a growing photo build-log repeatedly, and we don't want to pay every time we view our own
photos. Uploads go straight from the phone to R2 via signed URLs, not through the backend.

Consequences / day-one care: Workers is a constrained runtime. We use Neon's
serverless/HTTP driver (not a long-lived TCP pool), enable Workers' Node-compat, and
verified better-auth runs on Workers (its sessions are DB-backed, stored in Neon).
