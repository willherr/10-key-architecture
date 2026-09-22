# Architecture

The real shape of the system, without the real infrastructure identifiers or the business logic behind it.

```
┌──────────────────────────────┐
│  Flutter app                 │
│  Android · Windows · Web     │
│  one codebase, one UI layer  │
└───────────────┬───────────────┘
                │  entitlement check / purchase flow
                ▼
┌───────────────────────────────┐        ┌─────────────────────────┐
│  Backend Worker (Cloudflare)  │──────▶ │  Payment processor       │
│  purchase verification,       │        │  (merchant of record,    │
│  entitlement grants, a        │        │  handles tax/VAT)        │
│  scheduled reconciliation job │        └─────────────────────────┘
└───────────────┬───────────────┘
                │  entitlement records
                ▼
┌───────────────────────────────┐
│  D1 (SQLite at the edge)      │
└───────────────────────────────┘

┌───────────────────────────────┐
│  Marketing site (Cloudflare)  │  separate Worker, separate deploy,
│  static pages + the public    │  no shared release cadence with
│  web trial build              │  the app or the backend Worker
└───────────────────────────────┘
```

## Why it's split this way

**The app never trusts itself alone.** Purchase state is client-side first (offline has to keep working, a paid feature can't disappear because a connection dropped), but a server-side Worker layers a real, live check on top, one that can downgrade a *session* without ever touching the locally persisted grant, and never punishes a device-integrity failure alone (a rooted phone with a real purchase must never lose access).

**The backend Worker and the marketing site are two separate deployables, not one.** They have no shared release cadence: a marketing copy change shouldn't need a backend redeploy, and a backend logic change shouldn't need the marketing site rebuilt. Each gets its own CI workflow (see `.github/workflows/`), scoped by path filter so a change to one never runs the other's checks.

**A scheduled job, not just reactive webhooks.** Payment webhooks can fail silently (a dropped request, a transient error), so a cron job on the backend Worker re-checks state periodically and reconciles anything a webhook handler missed, on a cadence loose enough that catching a real bug matters more than reacting to it instantly.

**D1 over a hosted relational database.** The actual entitlement data is small (one row per grant) and doesn't need complex relational queries, SQLite at the edge next to the Worker that reads it beats a separate database service for both cost and latency at this scale.

## What's deliberately not shown

The real purchase-verification and anti-replay logic, the actual referral-program mechanics, and any live infrastructure identifiers (database IDs, product IDs, custom domains beyond what's already public) stay out of this repo. What's here is the shape, the CI/deploy mechanics, and the reasoning, not the implementation.
