# bun-native-effect

Research workspace for porting [Effect-TS v4](https://github.com/Effect-TS/effect-smol)
semantics into [Bun](https://github.com/oven-sh/bun)'s native Rust runtime layer.

## Goal

Implement Effect's runtime primitives (fiber scheduler, interruption, structured concurrency)
as native Bun extensions — preserving exact Effect semantics, verified mechanically via a
differential oracle against the TypeScript reference implementation.

## Layout

| Path | Contents |
|---|---|
| `bun/` | Pinned `oven-sh/bun` checkout (Rust-port era) |
| `effect-v4/` | Pinned `Effect-TS/effect-smol` checkout — the executable specification |
| `docs/` | Design docs and ADRs |
| `PLAN.md` | Current plan of record |
| `SHAS.md` | Pinned SHAs for reference checkouts |

> `zio-reference/` (ZIO Scala source) is a local reading reference only and is not committed.

## Status

Active research. See [PLAN.md](PLAN.md) for current direction.

## License

MIT — see [LICENSE](LICENSE).
