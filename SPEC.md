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

- **Apex**: `alxvarp.com` (configurable via the `APEX` env var on the cerebro and on the Vercel project). Each community at `<slug>.<apex>`. `akrontrade.io` is not used by AkronCloud — no redirects, no references, no shared certs.
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

- **No passwords.** Supabase Auth is configured with `signInWithPassword: false` and `signInWithOtp: true` for email magic links. The only login flow is: user enters email → receives a single-use link → link click signs them in. The Resend SMTP provider (already keyed in the legacy system via `RESEND_API_KEY`) is wired in via `auth.config.smtp.*`; `RESEND_API_KEY` + `RESEND_FROM_EMAIL` + `RESEND_FROM_NAME` live in `.env`, never hard-coded.
- **Super-admin**: never goes through a public form. Provisioned by `auth.admin.inviteUserByEmail(email, { data: { is_super_admin: true }})` — typically called by the self-installer (`§ 14`). The invited email contains a magic link; first click creates the `auth.users` row with `app_metadata.is_super_admin: true` baked in and drops them straight into `<apex>/admin`. Recognized by the panel (allowed to access `apex` routes for platform-level metrics) and by the orchestrator (allowed to read all-tenant telemetry via a service-role-equivalent call).
- **Community admin**: signs up via the public `signup.<apex>` flow. After the form submit the system sends them a magic link to their email; first click creates `auth.users` + the `tenants` row + sets `app_metadata.tenant_id` + `app_metadata.is_tenant_admin: true`. No password is ever created or stored.
- **JWT claims used by services**:
  - `sub` (Supabase user UUID, stable per user)
  - `email`
  - `app_metadata.tenant_id` (null for super-admin)
  - `app_metadata.is_super_admin` (boolean)
  - `app_metadata.is_tenant_admin` (boolean)
- **Tenant routing**: the orchestrator and signals-bridge look up `tenant_id` from the JWT and only act on rows for that tenant. Postgres RLS provides defense-in-depth at the DB.
- **Trade-offs**: email-compromise = account-compromise, accepted because super-admins are by-invite-only and community admins are verified-by-email at signup. No offline login.

---

## 6. Sign-up flow

1. Public signup at `signup.alxvarp.com` (or `signup.${APEX}` if `APEX` env var is overridden on the Vercel project). Vercel-hosted.
2. Community admin enters: **slug** (becomes `<slug>.alxvarp.com`) + email. **No password field** — auth is magic-link only per `§ 5`.
3. System validates:
   - Slug is unique, lowercase, alphanumeric + hyphens only (`^[a-z0-9-]{3,32}$`).
   - Email is valid + not already a tenant admin.
4. System sends a magic link to the email via Resend (`RESEND_API_KEY` etc. all env-driven).
5. User clicks the link → first click creates:
   - `auth.users` row with `app_metadata.tenant_id: <new uuid>` and `app_metadata.is_tenant_admin: true`.
   - `tenants` row with the slug, display name, default theme colors.
6. User redirected to `<slug>.alxvarp.com/onboarding` — wizard:
   - Welcome + branded panel preview.
   - **Step 1 — Register your node**: paste a token / run a one-line command.
   - **Step 2 — Set slot count**: how many MT5 slots you want on day 1.
   - **Step 3 — Choose a theme** (colors + logo).
   - **Step 4 — Done** → land in the panel dashboard.
7. Once the node registers, slots appear in the panel automatically.

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
hooks/                                  # future (post-installer hooks)
tests/
  vm/                                  # Vagrant / Lima config for local bootstrap tests
package.json                           # for shellcheck / shfmt + npm-run
LICENSE
```

### `AkronCloud-Slot` (separate repo) — the slot service

```
SPEC.md                                  # API contract (REST + WS — the canonical source of truth for this repo)
README.md                                # what + repo layout + companion repos
LICENSE                                  # MIT
package.json                             # @akroncloud/slot-service (Fastify + ws + Drizzle + jose + zod + pino)
tsconfig.json
docs/CONNECTORS.md                       # how to add a new broker connector (slot's extension point)
src/
  server.ts                              # Fastify + WS upgrade
  api/{rest.ts,ws.ts}
  ledger.ts                              # truth source (positions/orders/fills) backed by local SQLite
  reconciler.ts                          # ledger ↔ broker periodic sync
  risk.ts                                # pre-trade validator (slot-side)
  auth.ts                                # cerebro-issued JWT validation
  connectors/{base.ts,deriv.ts,...}      # pluggable BrokerConnector implementations
  db/{schema.ts,migrations/...}
```

There is **no web UI, no login page, no visual layer**. The slot is a pure
API service: REST + WebSocket, authenticated by short-lived JWTs
signed by the cerebro. Deployed to the tenant VPS via the
`AkronCloud-Node` bootstrap, which pulls
`ghcr.io/alxvarp/akroncloud-slot:<tag>` and runs the image with
`.env`-driven broker credentials + JWT secret.

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

## 14. Self-installer (one-command setup)

The first-time setup is captured in `AkronCloud/scripts/setup/install.sh`. An operator (the platform owner) runs this **once** on a fresh machine to provision everything end-to-end, from "I have all the accounts and API keys" to "fully deployed, ready for community signups".

### 14.1. Inputs (all env, nothing hard-coded)

| Variable | Meaning |
|---|---|
| `SUPABASE_PROJECT_REF` | The Supabase project URL host |
| `SUPABASE_SERVICE_ROLE_KEY` | Service-role key (NOT anon) |
| `VERCEL_TOKEN` | Vercel personal access token |
| `VERCEL_PROJECT_ID` | Vercel project pre-created with `<apex>` domain attached |
| `NETBIRD_MANAGEMENT_KEY` | NetBird management API token |
| `CEREBRO_VPS_HOST` | Where the orchestrator will be deployed (e.g., `45.151.122.104`) |
| `CEREBRO_VPS_SSH_KEY` | Path to SSH key file (the installer `scp`s the orchestrator bundle + `ssh`s to deploy) |
| `SUPER_ADMIN_EMAIL` | The email that gets the magic-link invite after install completes |
| `APEX` | Defaults to `alxvarp.com` |
| `RESEND_API_KEY` / `RESEND_FROM_EMAIL` / `RESEND_FROM_NAME` | Resend creds wired into Supabase Auth SMTP |

Per-vendor secrets (Supabase anon key, Vercel deploy hook URL, NetBird setup_key base, etc.) are either left as installer outputs that the operator pastes into Vercel project settings, or read back via the same API in a follow-up step.

### 14.2. What the installer does

1. **Supabase**: applies schema migrations (idempotent), enables RLS, configures `signInWithOtp = true` + Resend SMTP, sets the JWT custom-claims hook.
2. **NetBird**: creates the `akron-cloud` network if missing, generates the base setup_key, stores it back in `.env` for the orchestrator.
3. **Vercel**: triggers a production deploy of the `AkronCloud` monorepo's `apps/web` panel; reads back the deployment URL.
4. **DNS**: instructs the operator to point `<apex>` + `*.<apex>` to the Vercel deployment; alternatively creates Cloudflare records automatically if `CLOUDFLARE_API_TOKEN` is set.
5. **Cerebro (orchestrator)**: `scp` + `ssh` into `CEREBRO_VPS_HOST`, runs `deploy-cerebro.sh` with the Supabase + NetBird env, restarts the systemd unit, runs the legacy smoke test (a curl to `/v1/internal/nodes/whoami`).
6. **Super-admin**: calls `supabase.auth.admin.inviteUserByEmail(SUPER_ADMIN_EMAIL, { data: { is_super_admin: true }})`. The operator receives a magic link in their inbox, clicks it, and lands in `<apex>/admin` already with `is_super_admin: true` set.
7. **Output**: prints the platform URL, the invited-super-admin status, and a checklist (Cloudflare DNS, Resend domain verification, NetBird magic DNS check).

### 14.3. Idempotency

Every step is `set -euo pipefail`-wrapped and re-runnable. Re-running on an already-provisioned Supabase project does not duplicate; it instead applies pending migrations + re-asserts config.

### 14.4. Out-of-scope (still human)

Per the providers' own TOS, the operator must manually:
- Sign up for Vercel + create the project + attach the `<apex>` domain.
- Sign up for Supabase + create the project + capture `SUPABASE_PROJECT_REF` + `SUPABASE_SERVICE_ROLE_KEY`.
- Sign up for NetBird Cloud (or self-host a management server) and get the `NETBIRD_MANAGEMENT_KEY`.
- Set up a VPS reachable via SSH.
- Get a Cloudflare account + token if the apex is a Cloudflare-managed domain.

The installer assumes these are done; it does the wiring.

### 14.5. Why this is here

"Anyone can fork `AkronCloud` + run `./scripts/setup/install.sh` + own their own platform" — i.e. any operator (the team, a future sister team, an enterprise customer wanting their own isolated instance) can spin up a parallel AkronCloud on their own terms without our help. Every credential lives in `.env`, so the installer is auditable and the same script reproduces the same platform across operators.

---

## 15. Tenant modules

Each tenant's panel lets the admin enable/disable modules. Modules are scoped per `tenant_id`; one tenant's modules don't affect another's.

### MVP module matrix

| Module | Tier | What it does (one line) |
|---|---|---|
| signals | MVP | Signal provider posts trade instructions → cerebro executes in the tenant's slots. |
| copy | MVP | Leader's slot closes a trade → cerebro mirrors orders in follower slots per configured ratio. |
| analytics | MVP | Aggregations over `trade_events` (P&L, drawdown, leaderboard) per tenant. |
| notifications | MVP | Cron-driven dispatcher sends email (Resend) + webhook on trade/health events. |
| risk controls (per-slot) | MVP | Pre-trade check on each slot — rejects + kill-switches if `max_position_size` or `max_daily_loss` would be exceeded. |
| chat | MVP | Per-tenant chat channels with realtime push via Supabase Realtime. |
| calendar | MVP | Economic event feed synced weekly from TradingEconomics API → Supabase. |

### Tier 2 — post-MVP

- **marketplace** — rent strategies/signals inside the community (revenue split).
- **ea-hosting** — run MQL5 EAs as managed services on tenant slots.
- **risk controls (advanced)** — cross-slot correlation, real-time margin, portfolio-level VaR.

### Out of scope

- **academia / tutorials** — high content-creation cost; not differentiating.
- **multi-broker per slot** — 1 broker per slot is sufficient for MVP.
- **news feed** — depends on data-provider contracts; changes cost model.

### Toggle mechanism

Each module's enable/disable flag is one BOOLEAN column on the `tenants` row. Toggling from the panel: `PATCH /v1/tenants/me/modules/{module}` with `{ enabled: bool }`.

Per-module detail (data tables, RLS, orchestrator endpoints, UI surface): see [`docs/MODULES.md`](./MODULES.md).

---

## 16. Slot service (lives in [`AkronCloud-Slot`](https://github.com/AlxVarp/AkronCloud-Slot))

The slot service is **its own repo** with **its own release cadence**. Architectural shape:

- **No web UI.** Pure REST + WebSocket API. Auth via short-lived JWT (HS256, max 1h, per-tenant secret) signed by the cerebro.
- **One broker per slot** via a pluggable `BrokerConnector` (Deriv in MVP). Connector interface defined in `AkronCloud-Slot/SPEC.md § 4`.
- **Internal ledger** (SQLite file in `/var/lib/akron-slot/state.db`) is the truth source for orders/fills/positions; the reconciler keeps it in sync with the broker every 30s.
- **Deployed** as `ghcr.io/alxvarp/akroncloud-slot:<semver>`; the `AkronCloud-Node` bootstrap pulls + runs it.
- **Risk engine** (pre-trade validator) reads `risk_limits` set by the cerebro.
- **Reconciliation** emits drift events to the WS + persisted audit; on >1 lot drift or broker-rejected order, the account is marked `status='error'` and further orders return `503 RECONCILING`.

The cerebro consumes the slot's API only via the NetBird peer link — never through a public endpoint. Modules (`signals`, `copy`, `analytics`, etc.) consume cerebro-mediated slot events, **not broker APIs directly**. The slot is the **single point of contact with the broker**.

**Where to read more**:
- API contract (REST + WS endpoints, auth, error codes, data model, env vars): [`AkronCloud-Slot/SPEC.md`](https://github.com/AlxVarp/AkronCloud-Slot/blob/master/SPEC.md).
- How to add a new broker connector: [`AkronCloud-Slot/docs/CONNECTORS.md`](https://github.com/AlxVarp/AkronCloud-Slot/blob/master/docs/CONNECTORS.md).

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
| 2026-07-16 | Third repo: `AkronCloud-Slot` (slot service — pure REST + WS API). | The slot has its own API contract, its own release cadence, and no UI at all — it earns its own repo so the broker-connector work doesn't entangle the platform monorepo. See `§ 16` for the pointer. `AkronCloud-Node` now pulls `ghcr.io/alxvarp/akroncloud-slot:<tag>` instead of doing the slot work itself. |
| 2026-07-16 | Slot ↔ cerebro addressing via NetBird peer name + Docker context (no public DNS, no host-published ports). | Avoids port-443 collisions on tenant VPSes (e.g., Canencio / WordPress stacks); eliminates per-slot public certs. See `§ 4 Tech stack` row, `§ 7` step 5. |
| 2026-07-16 | Domain apex moved to `alxvarp.com` (with `APEX` env var still overridable). | `alxvarp.com` is owned by the team and was already partially in use; consolidating reduces DNS sprawl. See `§ 3`, `§ 6`, `§ 14.1`. |
| 2026-07-16 | Auth is magic-link only (no passwords). Supabase Auth `signInWithPassword=false`, `signInWithOtp=true`. Resend SMTP. | Removes password-management surface. Trade-off (email-compromise = account-compromise) is acceptable because super-admins are by-invite only and community admins verify email at signup. See `§ 5`, `§ 6`, `§ 14.2`. |
| 2026-07-16 | One-command self-installer in `AkronCloud/scripts/setup/install.sh`. | Makes the platform forkable + reproducible. Any operator with the per-vendor API keys can spin up a parallel AkronCloud without our help. Idempotent. See `§ 14`. |
