---
name: migration-and-data-safety
description: >-
  Use for schema changes, backfills, dual-writes, and RLS/policy migrations.
  Expand→migrate→contract; irreversible sweeps off/warn/enforced (default warn);
  never expand+contract in one deploy; assert prod connection role (superuser
  bypasses RLS even under FORCE); don't trust SQLite suites for Postgres-only
  controls.
---
# Migration and Data Safety

Schema, backfill, dual-write, and RLS changes fail closed in design and open in production if you skip phases or trust the wrong database.

## When this skill applies

- Any migration, backfill, dual-write, or RLS / row-policy change.
- Feature flags that flip from warn → enforced on data paths.
- Changing DB roles, connection strings, or `FORCE ROW LEVEL SECURITY`.
- Before claiming Verified for Postgres-only controls (`verified-delivery`).

## Expand → migrate → contract

Never expand and contract in one deploy.

| Phase | Do | Don't |
|-------|----|-------|
| **Expand** | Add nullable columns, new tables, new dual-write path, new policies in permissive/warn form | Drop columns, tighten NOT NULL, remove old reads |
| **Migrate** | Backfill idempotently; dual-read/dual-write; verify counts/invariants | Assume empty tables; one-shot destructive UPDATE without resume |
| **Contract** | After Verified migrate: drop old path, tighten constraints, enforce RLS | Contract in the same release that expanded |

**Failure mode expand-contract-atomic**: one deploy adds column and drops old column → rollback impossible, readers break.

Rule: **minimum two deploys** (expand+migrate, then contract). Prefer three when backfill is heavy.

## Irreversible sweeps: off / warn / enforced

For sweeps that cannot be undone safely (mass revoke, hard RLS, delete orphans):

1. **off** — code path present, not active.
2. **warn** — compute would-deny / would-change; log; do not block. **Default for new sweeps.**
3. **enforced** — only after Verified warn metrics and rollback plan.

**Failure mode warn-skipped**: ship straight to enforced → production outage or silent mass deny.

Document the flag name, default (`warn`), and owner. Client-visible responses must not claim "blocked" while in warn (`contract-and-compat` — enforcement-lie is P0).

## RLS and roles

- **Superuser / owner bypasses RLS** even under `FORCE ROW LEVEL SECURITY` for table owners in common Postgres setups — treat connection role as part of the security boundary.
- In production: **assert connection role** is the least-privilege app role, not superuser/migrator.
- Migrator role may bypass by design; app runtime must not use migrator credentials.

**Failure mode rls-theater**: FORCE RLS enabled; app connects as owner/superuser → policies never apply; SQLite or unit mocks still green.

### Assert role (pattern)

At startup or health check in prod:

- Query `current_user` / `session_user` (or equivalent).
- Fail closed if role ∉ allowed app roles.
- Log role once at boot (no secrets).

Do not claim "RLS Verified" without: (1) Postgres, (2) correct role, (3) deny-path test that would fail if RLS were off.

## SQLite ≠ Postgres

**Failure mode sqlite-green-lie**: suite on SQLite "proves" RLS / partial indexes / `EXCLUDE` / `LISTEN` / Postgres policy — it does not.

Rules:

- Tag Postgres-only tests; run them against Postgres in CI for merge.
- Document in delivery receipt: `DB under test: postgres|sqlite` (`verified-delivery`).
- Dual-run critical authz migrations against Postgres before contract phase.

## Backfill rules

1. **Idempotent** — re-run safe; use keys / `WHERE missing` / upserts.
2. **Batched** — resumable; checkpoint progress; avoid single transaction wiping the fleet.
3. **Observable** — counts before/after; anomaly alerts; dry-run or warn mode first when destructive.
4. **No silent truncate** — deletes/archives need explicit approval and backup/restore note in the Faber packet (`handoff-faber-rigor`).
5. **Dual-write window** — writers hit old+new until readers cut over; verify lag = 0 before contract.

## Dual-write / dual-read

- Expand: write both; read old (or read new with fallback).
- Migrate: backfill; compare; fix drift.
- Cutover: read new; keep dual-write until stable.
- Contract: remove old writes/reads.

**Failure mode half-dual-write**: only some code paths dual-write → silent split brain. Grep all writers (`contract-and-compat` resolver grep discipline).

## Deploy checklist

- [ ] Phase named: expand | migrate | contract (exactly one primary intent)
- [ ] Not expand+contract in same deploy
- [ ] Sweep flag default = warn (if irreversible)
- [ ] Backfill idempotent + batched + observable
- [ ] Prod role assertion present or ticketed as blocker before enforce
- [ ] Postgres-only controls tested on Postgres
- [ ] Rollback: expand forward-fix; migrate pause; contract only after Verified
- [ ] Delivery receipt: Status Verified with result file (`verified-delivery`)

## Failure modes (named)

| Name | Meaning |
|------|---------|
| **expand-contract-atomic** | Expand and contract one deploy |
| **warn-skipped** | Jump to enforced |
| **rls-theater** | RLS on; wrong role bypasses |
| **sqlite-green-lie** | SQLite suite certifies Postgres control |
| **half-dual-write** | Not all writers updated |
| **non-idempotent-backfill** | Second run corrupts or duplicates |
| **migrator-in-prod** | App uses migrator/superuser credentials |

## Interaction with other skills

- **verified-delivery**: migration claims need result files from the right DB.
- **fail-closed-review**: RLS / role / default-deny findings.
- **contract-and-compat**: flags and API honesty during warn vs enforced.
- **adversarial-qa**: omit optional tenant fields, rollback counters after migrate.
- **karpathy-method**: measure invariants after each phase, not after "feels done".
- **code-that-holds**: invariants encoded as tests that fail closed.

## Anti-patterns

- "We'll tighten in a follow-up" without a warn flag and ticket.
- Dropping a column because "nothing reads it" without a repo-wide grep and a deploy of dual-read removal first.
- Claiming FORCE RLS as the entire authz story while API gates are warn-only.
