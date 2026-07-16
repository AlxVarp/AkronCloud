# AkronCloud — Decision Log

> Append-only log of approved directions. When a decision is finalized, add an entry here with date, decision, why, and any links to PRs/issues. Older entries are preserved verbatim.

---

## 2026-07-16 — Repository bootstrap

- **Decision**: stand up two new repos — `AkronCloud` (monorepo) + `AkronCloud-Node` (tenant VPS bootstrap) — to keep the new system work isolated from the legacy `Akron` repo.
- **Why**: the legacy `Akron` repo carries 9 months of operational history (mt5 slot lifecycle, broker-manual flow, AKR-466 fixes, the 2026-07-16 rollback of the Akron Trade planning era, etc.). Continuing to commit to it would entangle the new rebuild with state we don't want to carry. Freezing `Akron` at master HEAD `535d5992` preserves the operational story while letting the new system start clean.
- **SPEC committed at**: `AkronCloud/SPEC.md` at the root, capturing the 13-section canonical specification.

---

## Format for new entries

```markdown
## YYYY-MM-DD — short title

- **Decision**: ...
- **Why**: ...
- **Refs**: PR #N / commit <sha> / [discussion link]
```
