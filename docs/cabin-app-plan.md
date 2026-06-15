# Cabin Tracking App — Where We've Landed

A summary of the planning so far, for the app we'll use to manage building the cabin.

## The idea

A single place to plan the build once and then check in as we go — what needs doing, who's doing it, what we need to buy, where the money's going, and a photo record of how everything was actually put together. The guiding principle throughout has been **low ongoing effort**: set it up thoughtfully now so that keeping it current is quick rather than a chore.

## What it's built on

We're using a standard relational database (Postgres) rather than anything more exotic. The short version: our app is mostly about filtering and totalling things up — what's due this week, what's left in the kitchen, where are we over budget — and that's exactly what this kind of database does best, with very little to maintain. The one genuinely "connected" part of the build (task A can't start until task B is done) is handled cleanly without needing anything fancier.

## What it will track

**The core of it:**

- **Tasks** — the heart of the app. Every piece of work, with its status, dates, cost estimate, and which stage and room it belongs to.
- **Dependencies between tasks** — what blocks what, so we can ask "if the foundation slips a week, what else moves?"
- **Phases** — the natural stages of the build (site prep, foundation, framing, rough-in, insulation, finishing).
- **Rooms / areas** — kitchen, loft, bathroom, deck, and so on, so we can see what's left in each.
- **Contractors** — who's doing what, their rates and contact details.
- **Materials** — the stuff that gets used up in the build (lumber, fixtures, tile), with quantities, costs, and — important for a remote cabin — lead times and order status.
- **Tools** — kept separate from materials, because a tool gets reused across many jobs rather than consumed by one. This lets the app tell us the full list of tools we don't yet own, so we buy the saw *once*, and helps us cluster jobs that need a rented tool into the same window so we're not paying for the excavator twice.
- **Suppliers** — where materials and tools come from.
- **Expenses & budget** — a running money ledger that rolls up against a budget per category, to answer "where are we over?"

**Supporting pieces:**

- **Permits & inspections** — types, dates, and approvals, which often gate other work.
- **Milestones** — the dates that really matter (weathertight before winter, move-in).
- **Notes** — a quick journal of decisions, so in six months we remember *why* we chose what we did.
- **Photos & documents** — see below.

## The photo record

This is the part that pays off later. We'll attach photos to tasks and to rooms as we go — partly to document tools and receipts, but mainly to capture **how each part of the cabin is actually constructed**. Lined up by date, that quietly becomes a visual build log: a reference for understanding how the loft, the walls, the systems all went together, long after the work is hidden behind finished surfaces.

A practical note on this: the photos and documents themselves are stored as ordinary files (in a folder or cloud storage), and the app just keeps track of where they are and what they show. That keeps everything fast and easy to back up. Photos and documents are handled by one simple mechanism rather than two.

## What's next

The plan is to build the core first — tasks, phases, rooms, contractors, materials, tools, suppliers, expenses, and the dependency links — and add the supporting pieces only as we feel we need them, rather than over-building up front.

The next concrete step is to draw the whole thing out as a diagram showing how everything connects, then turn that into the actual database structure.
