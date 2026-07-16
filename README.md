# AkronCloud

Multi-tenant community platform. Sign up at `signup.akrontrade.io`, pick a community name (slug), and your community gets its own instance at `<slug>.akrontrade.io` with isolated DB + auth + routing, plus a one-line bootstrap to register your own VPS node.

## What this is

`AkronCloud` is a rebuild of the legacy `Akron` broker-copier platform as a multi-tenant SaaS-style product:

- **Free for all tenants** during MVP.
- **Self-hosted per tenant**: every community brings their own VPS to run their MT5 slot pool under our one-line bootstrap.
- **Single source of truth**: [`SPEC.md`](./SPEC.md) at the root captures every decision taken on 2026-07-16 for the rebuild. Update it on every PR that changes behavior.
- **Sibling repo**: [`AkronCloud-Node`](https://github.com/AlxVarp/AkronCloud-Node) holds the tenant-side `bootstrap.sh` + agent scripts.

## Architecture in one paragraph

`Vercel` hosts the panel (Next.js). `Supabase` is the only backend vendor — DB + Auth + Storage + Realtime in one place, with Postgres RLS for tenant isolation. The legacy VPS at `45.151.122.104` keeps running the **cerebro orchestrator** (slot lifecycle + signals-bridge + tenant routing) as our single always-on long-lived service. Each tenant runs `akron/mt5-base` slot containers on their own VPS, reaching the cerebro via a per-tenant NetBird mesh.

## Repo layout

```
SPEC.md                                # canonical decisions doc — start here
README.md                              # this file
LICENSE                                # MIT
docs/
  decision-log.md                      # append-only log of approved directions
.github/workflows/                     # CI + VPS deploy
apps/
  web/                                 # Vercel Next.js panel
  orchestrator/                        # VPS cerebro service (Node.js)
packages/
  db/                                  # Supabase schema + RLS policies + Drizzle client
  types/                               # shared TypeScript types
scripts/                               # ops automation (certs, provisioning)
```

## Reading order for a new contributor

1. **Start with `SPEC.md`.** It's the single source of truth for the system.
2. `docs/decision-log.md` for the chronologically-ordered decisions.
3. `apps/web/` for the panel (Next.js).
4. `apps/orchestrator/` for the cerebro service (the port of the legacy Node.js orchestrator + signals-bridge).
5. `packages/db/` for the schema and RLS.
6. `packages/types/` for the shared types between web and orchestrator.

## Development status

This repo is the canonical specification for a system that has not yet been implemented. The legacy Akron repo (still on master HEAD `535d5992`) holds the previous iteration's deploy, which stays operational until the migration plan in `SPEC.md § 9` is executed.

The current focus (in priority order):

1. Stand up Supabase project + schema.
2. Stand up the Vercel panel (Next.js, the signup flow + community panel).
3. Port the cerebro orchestrator to support remote tenant nodes + Supabase Realtime.
4. Build the `AkronCloud-Node` bootstrap.
5. Wire the wildcard cert (`*.akrontrade.io`) + tenant subdomain routing.
6. Provision super-admin + first community.

## Contributing

Open PRs against `master`. Match the naming convention in `SPEC.md`: `feat/<scope>-<slug>`, `fix/<scope>-<slug>`, `docs/<scope>-<slug>`. Update `SPEC.md` in the same PR if behavior changes.
