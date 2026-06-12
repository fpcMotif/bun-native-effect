# Contributing

This is a research workspace for porting Effect-TS v4 semantics into Bun's native runtime layer.

## Repo layout

| Directory | Purpose |
|---|---|
| `bun/` | Pinned checkout of `oven-sh/bun` (Rust-port era) |
| `effect-v4/` | Pinned checkout of `Effect-TS/effect-smol` — the executable specification |
| `zio-reference/` | Local reference only — not committed |
| `docs/` | Design docs, ADRs, and agent-skill configuration |
| `PLAN.md` | Current plan of record |
| `SHAS.md` | Pinned commit SHAs for the reference checkouts |

## How to contribute

1. Check `PLAN.md` for current direction before opening an issue.
2. Open a GitHub Issue for bug reports or implementation tasks on the Bun fork.
3. Use the issue template and apply the appropriate triage label.
4. PRs against `main` are welcome; keep them focused on a single concern.

## Reference checkouts

`bun/` and `effect-v4/` are read-only references pinned at the SHAs in `SHAS.md`.
Do not commit changes to them — work should be upstreamed to their respective repos.
