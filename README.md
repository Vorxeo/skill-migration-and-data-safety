# Migration and Data Safety

Claude Code skill: `migration-and-data-safety`

## What

Safe schema/backfill/RLS changes: expand→migrate→contract across deploys; irreversible sweeps as off/warn/enforced (default warn); assert prod DB role; never certify Postgres controls on SQLite.

## When to use

Any migration, dual-write, backfill, RLS/policy change, or warn→enforced flag flip.

## Install

Copy this folder into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/migration-and-data-safety
cp SKILL.md ~/.claude/skills/migration-and-data-safety/
```

Claude Code loads `SKILL.md` from `~/.claude/skills/<name>/`.

## Sibling skills

`verified-delivery`, `fail-closed-review`, `contract-and-compat`, `adversarial-qa`, `karpathy-method`, `code-that-holds`, `handoff-faber-rigor`

Proposed repos: see [Vorxeo](https://github.com/Vorxeo) `skill-*` packs.

## License

MIT — Copyright (c) 2026 Vorxeo. See [LICENSE](./LICENSE).
