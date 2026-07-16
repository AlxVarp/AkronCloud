# AkronCloud — Decision Log

> Append-only log of approved directions. When a decision is finalized, add an entry here with date, decision, why, and any links to PRs/issues. Older entries are preserved verbatim.

---

## 2026-07-16 — Repository bootstrap

- **Decision**: stand up two new repos — `AkronCloud` (monorepo) + `AkronCloud-Node` (tenant VPS bootstrap) — to keep the new system work isolated from the legacy `Akron` repo.
- **Why**: the legacy `Akron` repo carries 9 months of operational history (mt5 slot lifecycle, broker-manual flow, AKR-466 fixes, the 2026-07-16 rollback of the Akron Trade planning era, etc.). Continuing to commit to it would entangle the new rebuild with state we don't want to carry. Freezing `Akron` at master HEAD `535d5992` preserves the operational story while letting the new system start clean.
- **SPEC committed at**: `AkronCloud/SPEC.md` at the root, capturing the 13-section canonical specification.

---

## 2026-07-16 — Slot ↔ cerebro addressing uses NetBird peer name, not public DNS

- **Decision**: All slot ↔ cerebro traffic happens over a NetBird mesh. The cerebro addresses each remote node by its NetBird-assigned peer name (e.g. `nodo1`) via Docker context. Slot containers bind their internal ports (`:8001-8003`, `:3000`, etc.) **only inside the container** — never published to the node's host interface.
- **Why**: eliminate the need for public DNS records, public certs, or port forwarding for slots. The only public-facing TLS we own is `*.alxvarp.com` → Vercel (managed by Vercel). Resolves port-443 conflicts on tenant VPSes that already run other apps (Canencio, WordPress, etc.) — we never claim `:443` on a tenant node.
- **Concretely**: a tenant can run our `akron/mt5-base` slot pool on the same VPS as a medical-app stack sharing `:443`. The cerebro -> `docker --context nodo1 exec ...` reaches the remote daemon through NetBird MagicDNS.
- **Refs**: `AkronCloud/SPEC.md` § 4 (Tech stack — NetBird row), § 7 (Node self-hosting — step 5).

---

## Format for new entries

```markdown
## YYYY-MM-DD — short title

- **Decision**: ...
- **Why**: ...
- **Refs**: PR #N / commit <sha> / [discussion link]
```

---

## 2026-07-16 — Domain apex moved to `alxvarp.com`

- **Decision**: canonical apex is now `alxvarp.com` (with `<slug>.alxvarp.com` for tenants), `signup.alxvarp.com`, `*.alxvarp.com` → Vercel. `akrontrade.io` is **not used** — no redirects, no references, no shared certs.
- **Why**: `alxvarp.com` is owned by the team and was already partially in use (legacy `akron.alxvarp.com` redirector in the RULES.md era). Consolidating to one apex reduces DNS sprawl; the `APEX` env var keeps it overridable for future migrations or sister-team instances.
- **Refs**: `AkronCloud/SPEC.md` § 3 (Multi-tenant model — apex), § 6 (Sign-up flow), § 14 (Self-installer).

---

## 2026-07-16 — Magic-link only auth, no passwords

- **Decision**: Supabase Auth is configured with `signInWithPassword=false` and `signInWithOtp=true` (email magic links). Resend is the SMTP provider (the legacy system already had a `RESEND_API_KEY` flow, so it carries over).
- **Why**: removes the entire password-management surface — no breach surface, no reset flow, no hashing/storage of credentials. The trade-off (email-compromise = account-compromise) is acceptable because super-admins are by-invite only and community admins verify their email at signup.
- **Concretely**: super-admin provisioning becomes `auth.admin.inviteUserByEmail(email, { data: { is_super_admin: true }})`. The invite email is a magic link; first click creates the row with the metadata baked in.
- **Refs**: `AkronCloud/SPEC.md` § 5 (Auth model), § 6 (Sign-up flow — step 2 + step 4 magic-link), § 14 (Self-installer — step 6).

---

## 2026-07-16 — One-command self-installer (`install.sh`)

- **Decision**: the first-time setup is captured in `AkronCloud/scripts/setup/install.sh`. An operator (us now, a future sister team, or an enterprise customer wanting their own isolated instance) runs the script with `.env` populated, and gets a fully provisioned platform end-to-end.
- **Why**: makes the platform **forkable + reproducible**. Any operator can run their own AkronCloud without our help, given they have the per-vendor API keys. Replaces the "9 months of operational history" entry barrier that the legacy `Akron` repo currently presents to a new contributor.
- **Scope**: from "I have all the per-provider accounts and API keys" to "platform deployed + my super-admin invite is in my inbox". The accounts themselves (Vercel, Supabase, NetBird, Cloudflare) are still manual per their TOS; the script does the wiring.
- **Idempotency**: every step is `set -euo pipefail` + re-runnable. Re-running on a partially-provisioned Supabase project does not duplicate; it applies pending migrations + re-asserts config.
- **Refs**: `AkronCloud/SPEC.md` § 14 (Self-installer).
