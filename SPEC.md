# AkronCloud — New System Spec

> **Single source of truth** for the new multi-tenant community platform. This document captures every decision taken on 2026-07-16 for the rebuild on top of the legacy Akron deploy infra. If the system changes, update this file in the same PR — don't leave a "v2" appendix.
>
> **Where this lives:** `AkronCloud/SPEC.md` at the root of this repo. The legacy `Akron` repo (still on master HEAD `535d5992`) holds a `docs/handbook/TASKS.md` and a `docs/handbook/BUSINESS_MODEL.md` that redirect here.

---

## 1. Vision

**Sign up → community name → your own instance at `<slug>.<apex>`.**

A free, multi-tenant, **self-hosted-per-tenant** community platform. Every signup creates an isolated tenant (data + auth + routing + node agents) on a unique subdomain of the apex. The platform owner (us) runs the control plane on a small VPS, the panel UI on Vercel, DB + Auth + Storage + Realtime on Supabase; each community brings their own VPS to run their MT5 slot pool under our bootstrap.

---

## 2. Personas

| Persona | How they're created | What they can do |
|---|---|---|
| **Super-admin** | Pre-provisioned manually in `auth.users` (NOT via the public signup form) with `app_metadata.is_super_admin: true`. | Manages the platform: total tenants / slots / aggregate health / suspend tenant / create super-admin. Login at the apex. |
| **Community admin** | Self-signup at `signup.<apex>`; picks the slug + email + password; first user of a tenant. | Manages their own tenant: register a node, set `desired_slot_count`, see health, recycle slots, customize branding. Login at `<slug>.<apex>`. |
| **End user** *(later, not MVP)* | TBD per community. | TBD. Out of scope for the first iteration. |

---

## 3. Multi-tenant model

- **Apex**: `akrontrade.io` (configurable via `APEX` env var on the cerebro). Each community at `<slug>.<apex>`.
- **Wildcard cert** `*.<apex>` for the per-tenant subdomains. Decision DNS-01 via Cloudflare (S3 from the legacy spec).
- **Data isolation**: Postgres RLS with `tenant_id` on every row. JWT carries tenant via the `urn:akron:tenant_memberships` custom claim.
- **Per-tenant isolation at the network level** (NetBird mesh — see §7).
- **Pricing**: free for all tenants, MVP period. Plans/quotas deferred until product-market fit.
- **Branding per tenant**: display name, logo, accent colors. (Custom CSS themes and per-tenant `comunidad.com` domains are out of scope for MVP.)

---

## 4. Tech stack

| Layer | Provider | Why |
|---|---|---|
| Panel (web UI) | **Vercel** (Next.js) | Free tier covers our scale; zero infra ops for the app tier. |
| DB + Auth + Storage + Realtime | **Supabase** | One vendor, free tier; Postgres RLS works exactly the same; JWT/OIDC-compatible auth; built-in realtime for signals. |
| Orchestrator (control plane) | Existing VPS at `45.151.122.104` | Already paid for; already running the slot lifecycle we have. Stays as our single always-on service. |
| MT5 slot pool | Per-tenant VPS, **self-hosted** | Customer brings hardware; we ship a one-line bootstrap script. |
| Tenant-to-cerebro mesh | **NetBird** (free, self-hosted or SaaS) with **MagicDNS** for peer-name addressing | Each tenant gets its own setup_key for isolation. Slot containers are addressed by NetBird peer name + Docker context, not by public DNS or host-published ports. This means a tenant VPS can host other apps (Canencio-style stacks, WordPress, etc.) on `:443` without us needing to coordinate ports. |
| TLS for `*.<apex>` | Wildcard cert, DNS-01 via Cloudflare | S3 decision. |

> **Single source of truth principle**: no duplicate vendor evaluations. Supabase gives us DB + Auth + Storage + Realtime in one place; Vercel gives us the app tier; the VPS gives us long-lived state (orchestrator).

---

## 5. Auth model

- **Super-admin**: pre-provisioned by us in Supabase Auth with `app_metadata.is_super_admin: true`. No signup form. Recognized by the panel (allowed to access `apex` routes for platform-level metrics) and by the orchestrator (allowed to read all-tenant telemetry via a service-role-equivalent call).
- **Community admin**: signs up via the public `signup.<apex>` flow. Auto-provisioned in `auth.users` with `app_metadata.tenant_id` and `app_metadata.is_tenant_admin: true`.
- **JWT claims used by services**:
  - `sub` (Supabase user UUID, stable per user)
  - `email`
  - `app_metadata.tenant_id` (null for super-admin)
  - `app_metadata.is_super_admin` (boolean)
  - `app_metadata.is_tenant_admin` (boolean)
- **Tenant routing**: the orchestrator and signals-bridge look up `tenant_id` from the JWT and only act on rows for that tenant. Postgres RLS provides defense-in-depth at the DB.

---

## 6. Sign-up flow

1. Public signup at `signup.<apex>` (Vercel-hosted).
2. Community admin enters: **slug** (becomes `<slug>.<apex>`), email, password.
3. System validates:
   - Slug is unique, lowercase, alphanumeric + hyphens only.
   - Email is valid.
4. System auto-provisions:
   - New `auth.users` row with `app_metadata.tenant_id: <new uuid>`.
   - New row in `tenants` table with the slug + display name + theme defaults.
   - First user gets `app_metadata.is_tenant_admin: true`.
5. User redirected to `<slug>.<apex>/onboarding` — wizard:
   - Welcome + branded panel preview.
   - **Step 1 — Register your node**: paste a token / run a one-line command.
   - **Step 2 — Set slot count**: how many MT5 slots you want on day 1.
   - **Step 3 — Choose a theme** (colors + logo).
   - **Step 4 — Done** → land in the panel dashboard.
6. Once the node registers, slots appear in the panel automatically.

---

## 7. Node self-hosting

1. Panel generates a per-tenant, time-bound bootstrap token: `<tenant_id>:<tenant_token>:<orchestrator_endpoint>`.
2. User runs on their VPS:

   ```bash
   curl -fsSL https://github.com/AlxVarp/AkronCloud-Node/raw/main/bootstrap.sh | bash -s -- \
     --tenant-id=<uuid> \
     --tenant-token=<token> \
     --orchestrator=<orchestrator-host>:<port>
   ```

3. The script:
   - Installs NetBird client + joins the per-tenant mesh using a per-tenant setup_key (generated by the orchestrator on demand).
   - Pulls the `akron/mt5-base` image from the registry baked by the legacy system (`ghcr.io/alxvarp/akron-mt5-base:mt5-preinstalled` or the local `:latest` we're already using on the cerebro VPS).
   - Stands up `desired_slot_count` warm-pool slots, each with a unique `slot_index`, all networked to the orchestrator via the NetBird mesh.
   - Reports back to the orchestrator: "node registered, tenant X, slot count N, all slots healthy".
4. After registration, the orchestrator manages slot lifecycle as it does today (warm pool, recycle, broker-manual flow), but scoped to the tenant — slots from one tenant never see slots from another tenant.
5. **Slot ↔ cerebro addressing.** The cerebro runs `docker --context nodo<peer-name>` (the NetBird-assigned peer name, e.g. `nodo1`). It manages the slot pool via the Docker daemon on the remote node, traversing the NetBird mesh. Slot containers bind their internal ports (`:8001-8003`, `:3000`, etc.) **only inside the container** — never published to the node's host interface. As a result the node can host unrelated apps (Canencio, WordPress, etc.) on `:80`/`:443` without port conflict.

---

## 8. Signals / Trade events

> **Decision**: collapse the legacy `akron-signals-service-1` into the cerebro orchestrator.

- Slot reports trade event to orchestrator (existing flow, no change).
- Orchestrator's **signals-bridge** (a small module inside the same Node.js service) writes the event to a Supabase Postgres table `trade_events` (partitioned by `tenant_id` for cheap RLS).
- Supabase Realtime broadcasts the row change over a per-tenant channel.
- Front-end subscribes via `supabase-js` to the `<tenant_id>:trades` channel; gets the trade push without the orchestrator doing any WebSocket fan-out.
- The legacy `akron-signals-service-1` container is decommissioned.

---

## 9. Migration plan from the legacy `Akron` repo

The legacy `Akron` repo stays on master HEAD `535d5992` and is not modified beyond a final ARCHIVED notice (separate concern; not authored in this repo).

Migration order (when this system is ready for each phase):

1. **Provision Supabase project**: schema for `tenants`, `users`, `node_registrations`, `trade_events` + RLS policies + JWT custom claims hook.
2. **Migrate legacy DB**: one-shot `pg_dump | psql` into Supabase, reconcile with the new schema (or migrate via `supabase db push` if we keep the schema as the source of truth).
3. **Move frontend to Vercel**: stand up Next.js + Vercel functions, point apex DNS to Vercel, set up `*.<apex>` wildcard cert.
4. **Update cerebro orchestrator** to:
   - Manage remote tenant nodes (in addition to local slots).
   - Write trade events to Supabase (signals-bridge).
   - Validate JWTs against Supabase JWKS endpoint (vs Keycloak today).
5. **Provision super-admin** in Supabase Auth with `is_super_admin`.
6. **Bootstrap our own first community** (we ourselves as a tenant) on a separate VPS — the legacy `45.151.122.104` stays as the cerebro.
7. **Smoke test** the public signup with a second tenant on a different VPS.
8. **Decommission the legacy containers** on the cerebro VPS once their job is done: `akron-front-akron-front-1`, `akron-db-1`, `akron-signals-service-1` can all go.

---

## 10. Repo layout (proposed)

### `AkronCloud` (this repo) — monorepo

```
SPEC.md                                # this file
README.md                              # project overview + entry points
LICENSE                                # MIT
.gitignore
apps/
  web/                                 # Vercel panel — Next.js
    app/
      signup/                          # public signup
      tenants/[slug]/                  # community panel
      api/                             # Vercel functions (if we add any)
    components/
    package.json
    next.config.js
  orchestrator/                        # VPS cerebro — Node.js + TypeScript
    src/
      server.ts
      signals-bridge.ts
      tenant-router.ts                 # subdomain → tenant_id
      node-registry.ts                 # remote node management
      slot-pool.ts                     # existing slot lifecycle (port from legacy)
    package.json
    tsconfig.json
packages/
  db/                                  # Supabase schema + Drizzle client + RLS
    schema.ts                          # tenants, users, node_registrations, trade_events
    migrations/                        # SQL migrations as needed
    package.json
  types/                               # shared TypeScript types
    tenant.ts
    slot.ts
    node.ts
    package.json
scripts/
  provision-new-tenant.sh             # super-admin script for ad-hoc provisioning
  render-certs.sh                      # wildcard cert renewal automation
docs/
  decision-log.md                      # one entry per "approved direction" with date + reason
.github/
  workflows/
    ci.yml                             # typecheck + tests on PR
    release-vps.yml                    # deploy orchestrator to cerebro VPS
```

### `AkronCloud-Node` (separate repo) — tenant VPS bootstrap

```
README.md                              # how the bootstrap works + how to test
bootstrap.sh                           # the one-line installer (the active artifact)
src/
  lib/
    netbird.sh                         # NetBird mesh client setup
    mql5-base.sh                       # akron/mt5-base image pull + warm-pool startup
    registry.sh                        # report back to orchestrator
hooks/                                  # future (post-installer hooks)
tests/
  vm/                                  # Vagrant / Lima config for local bootstrap tests
package.json                           # for shellcheck / shfmt + npm-run
LICENSE
```

---

## 11. Out of scope for MVP

- Custom domains per tenant (`comunidad.com` style).
- CSS-level theming — only name + colors + logo for MVP.
- Per-tenant billing / quotas (free for now).
- Email / push notifications.
- Multi-region failover.
- Multi-region DB read replicas.
- Compliance certifications (SOC2, GDPR-DPA, etc.).
- Customer end-user model (only community-admin exists in MVP).
- Mobile native apps.
- Slack/Discord integrations.

---

## 12. Open items

- Brand identity: name + logo + apex domain — TBD with the team.
- Branding depth for MVP: stop at name + colors + logo, or also a preset gallery of themes?
- Slots without a cap (per current decision) — but rate-limit per tenant on node registrations to prevent abuse.
- Migration of legacy tenants: do we keep supporting the `45.151.122.104` integration via a thin adapter?
- Whether the orchestrator runs as one process or splits (signals-bridge vs slot-pool) — early instrumentation will tell.
- Whether to host NetBird ourselves on the cerebro or use NetBird Cloud (free tier).

---

## 13. Decisions log

| Date | Decision | Why |
|---|---|---|
| 2026-07-16 | Architecture split: control plane on Vercel + Supabase; long-lived orchestrator on the existing cerebro VPS; per-tenant MT5 slots on customer hardware. | Save OPEX, scale with the platform, and keep a clear single source of operational truth (the cerebro). |
| 2026-07-16 | Use Supabase for DB + Auth + Storage + Realtime. | Single vendor covers the four back-end primitives we need, has free tier for our scale, supports Postgres + RLS + JWT. |
| 2026-07-16 | Self-hosted-per-tenant (each community brings their own VPS). | Aligns with the product's "you own your instance" framing; OPEX stays bounded as we grow. |
| 2026-07-16 | Free for all tenants (MVP period). | Validate product-market fit before introducing plans/quotas. |
| 2026-07-16 | Super-admin pre-provisioned manually, not via signup form. | Avoids a self-service platform-admin attack surface. |
| 2026-07-16 | Per-tenant NetBird mesh (not shared). | Network isolation is the safest default; cost is one extra setup_key per tenant signup. |
| 2026-07-16 | Collapse the legacy `akron-signals-service-1` into the cerebro orchestrator's signals-bridge. | Removes an entire service from the deploy; Supabase Realtime covers the front-end fan-out. |
| 2026-07-16 | Two new repos: `AkronCloud` (monorepo, this) + `AkronCloud-Node` (tenant bootstrap). | Same pattern as legacy `Akron` + `Akronfront`, but with backend as a monorepo (apps + packages) and the node bootstrap as a focused release-able artifact. |
