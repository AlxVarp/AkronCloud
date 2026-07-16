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
- **Why**: eliminate the need for public DNS records, public certs, or port forwarding for slots. The only public-facing TLS we own is `*.akrontrade.io` → Vercel (managed by Vercel). Resolves port-443 conflicts on tenant VPSes that already run other apps (Canencio, WordPress, etc.) — we never claim `:443` on a tenant node.
- **Concretely**: a tenant can run our `akron/mt5-base` slot pool on the same VPS as a medical-app stack sharing `:443`. The cerebro -> `docker --context nodo1 exec ...` reaches the remote daemon through NetBird MagicDNS.
- **Refs**: `AkronCloud/SPEC.md` § 4 (Tech stack — NetBird row), § 7 (Node self-hosting — step 5).

---

## Format for new entries
