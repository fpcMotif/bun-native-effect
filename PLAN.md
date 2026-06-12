# Native Effect v4 in Bun — Plan of Record

> The key discipline: **port semantics, not files.** `effect-v4/packages/effect/src/internal/effect.ts`
> is the executable specification. Native code in the Bun fork is an optimization layer that must
> *prove* it preserves that specification — mechanically, via a differential oracle, not by review.

---

## 0. Ground truth (verified from this workspace, 2026-06-12)

These facts were read from the pinned checkouts and **override all four research documents**
wherever they conflict.

### 0.1 Bun @ `bcad9b220e` is in its Rust-port era
- `src/` is a Cargo workspace (~200 crates) built as `libbun_rust.a` via `cargo build -p bun_bin`,
  driven by `bun bd`. Per `bun/src/CLAUDE.md`: the `.zig` siblings are **non-compiled porting
  references**; new code goes in Rust.
- Parser crate `bun_js_parser` (`bun/src/js_parser/`): `lexer.rs` (~159 KB), `p.rs` (~426 KB,
  9,709 lines — the Rust parser core), `parser.rs`, `fold.rs`, and `parse/*.rs`
  (`parse_fn.rs`, `parse_suffix.rs`, `parse_prefix.rs`, …). Generator detection already exists:
  `parse/parse_fn.rs:26` (`is_generator = p.lexer.token == T::TAsterisk`); arrow machinery at
  `parse_arrow_body` (`parse_fn.rs:518`). Binary-op precedence levels: `js_ast::Op::Level`
  (re-exported at `parser.rs:596`).
- GC/JSC seam: crate `bun_jsc` (`bun/src/jsc/`) with `Strong`/`Weak` handle types, `CallFrame.rs`,
  promise/value types — this is the rooting and stack-trace seam, replacing the compass doc's
  stale `bindings.cpp`/bindgen/`v8.cpp` instructions.
- Sourcemaps: `bun/src/sourcemap/*.rs`. Transpiler golden tests: `bun/test/bundler/transpiler/`.
- Dev loop: `bun bd` (never set a timeout on it), `bun bd test <file>`, fuzzy match supported.

**Consequence:** every "write it in Zig" instruction in the research docs is dead. The native
runtime is a **Rust crate**. This also retroactively validates step 6 of the bootstrap list
("pure Rust Micro model") and deep-research-report (9)'s `.rs` paths, which the compass doc had
flagged as wrong — they were right for this checkout.

### 0.2 effect-v4 @ `4500fbfe0`: there is no `Micro` module in v4
- `packages/effect/src/Micro.ts` does not exist. The v4 core **is** the consolidated small
  runtime: `internal/effect.ts` (6,427 lines) + `internal/core.ts` (666 lines).
- `FiberImpl` (effect.ts:496): `_stack: Array<Primitive>`, `currentOpCount`,
  `maxOpsBeforeYield` read from the `Scheduler.MaxOpsBeforeYield` reference, `evaluate()` /
  `runLoop()` (effect.ts:585/612), interruption via `_interruptedCause` + `interruptChildren`.
- Dispatch is **not** a v3-style `contOpSuccess` table: each primitive carries a symbol-keyed
  `[evaluate](fiber)` method (`makePrimitive`). The compass doc's caveat "don't assume v3 names"
  was correct.
- **The complete v4 primitive set (17, adversarially verified):** 13 `makePrimitive` tags in
  `effect.ts` — `Sync, Suspend, Yield, Async, AsyncFinalizer, exitPrimitive ("Exit", a
  continuation wrapper — NOT the terminal exits), OnSuccess, OnFailure, OnSuccessAndFailure,
  OnExit, SetInterruptible, While, Iterator` — **plus 4 more `[evaluate]`-dispatched primitives
  in `core.ts`**: `Success` (:509) and `Failure` (:529, the terminal exit values), `WithFiber`
  (:558, the pervasive reader primitive), and `YieldableError` (:571, errors that evaluate to
  `exitFail(this)`).
- The `setInterval` keepAlive cited from effect-smol issue #1404 is **gone** — fixed upstream.
- v4 still clamps `Error.stackTraceLimit` (effect.ts:317–343, 1141–1142) — report (10)'s
  stack-suppression finding carries over and matters for the (optional) traces phase.

### 0.3 ZIO @ `d058fcd2`
Reading reference only: `runLoop` shape, `Cause` model, interruptible regions. Its work-stealing
multithreading does not port (JSC is single-threaded). Gitignored; never committed.

---

## 1. Review of the four documents — what stands, what's corrected, what's new

| Document | Keep | Drop / correct |
|---|---|---|
| **Architectural Blueprint (Fable)** | The four product goals; phase ordering instinct; ZIO as loop-shape reference | **Drop the entire Fable/F# compiler axis** (scope creep; the center of gravity is Bun's own parser). Drop "bypass the JS event loop" (microtask-ordering hazard). Drop `function*(){}.bind(this)` lowering (loses `arguments`/`new.target`). Drop all Zig file paths and "zero-overhead" claims. |
| **Compass spec (the backbone)** | The big architectural decision (keep the trampolined interpreter, move the loop native, **no machine-stack switching** — node-fibers lesson); FFI crossings only for user closures / JS-valued refs / async settle; GC rooting contract (root on push, release on pop, root the live generator); flag-gated everything, default OFF; kill criteria; differential testing; adversarial verify loop; risk table R1–R9 | Zig → **Rust** throughout. bindgen/`v8.cpp` seams → `bun_jsc` crate (`Strong`/`Weak`). "Target `Micro` first" → **target a defined v4-core subset** (Micro doesn't exist in v4; see §2 "MicroCore"). Its unverified-opcode caveat is now resolved — the 13 ops above are from source. |
| **Report (10)** | Effect suppresses stacks via `Error.stackTraceLimit` (re-verified in v4); lazy sourcemap resolution principle; "additive, never a forked Effect dialect" compatibility stance; "double parsing" framing for the perf ceiling | Its Bun-internals paths were partially speculative; superseded by §0.1. |
| **Report (9)** | Tiered-validation instinct; `Effect.fn` / `Effect.gen` as the `*() =>` lowering target; dev/prod tracing split | **Skip the Babel/SWC prototype tier** (we own the fork; oracle + golden tests give the same confidence cheaper). **Reject the Hack-pipe recommendation** — we deliberately ship F#-style `a |> b` → `b(a)` for Effect ergonomics, flag-gated, as an accepted permanent divergence from TC39. Its `.rs` paths turned out to be right. |

### Improvements not in any document

1. **"MicroCore" replaces "Micro".** Since v4 has no Micro module, define the proving-ground as
   a subset of the 17 v4 primitives. Wave 1 (the 13 sync primitives): `Sync, Suspend,
   exitPrimitive, Success, Failure, WithFiber, YieldableError, OnSuccess, OnFailure,
   OnSuccessAndFailure, OnExit, Yield, While` — terminal exits and `WithFiber` included, or no
   program can even finish. Wave 2 (the risk): `Async, AsyncFinalizer, SetInterruptible`.
   Wave 3 (the hardest): `Iterator` (live single-shot generator).
2. **Write the Rust model once, reuse it twice.** The standalone model crate abstracts JS values
   behind a `Value`/`Host` trait so the *same* crate is later compiled into Bun with `JSValue` +
   `Strong` rooting plugged in. No throwaway Rust.
3. **The oracle is the highest-leverage artifact — build it before any Rust.** A seeded random
   generator of effect-program descriptions (small JSON AST), an interpreter that builds the real
   effect-v4 program from a description and runs it on the JS runtime, recording
   `Exit` + an ordered event log (op order, finalizer order, interruption points). Both the Rust
   model (Phase 3) and the native runtime in Bun (Phase 4) are tested *only* by comparing against
   this oracle. This turns "did we port the semantics?" into a mechanical question.
4. **Windows is a named risk, not a footnote.** This machine is Windows 11; `bun bd` needs the
   full toolchain (cargo workspace + LLVM/clang + vendored WebKit artifacts). Milestone 0 is
   literally "unmodified debug build + one passing transpiler test". If it doesn't go green in a
   few focused days, move the build to WSL2 and keep editing on Windows.
5. **Never fork effect-v4.** It stays a pinned, unmodified oracle. All shipped changes live in
   the Bun fork behind default-off flags; the userland swap package is additive.

---

## 2. The plan

### Phase 0 — Workspace & dev loop (the first step)

| Step | What | Done when |
|---|---|---|
| 0.1 | `git init` the workspace repo; `.gitignore` excludes `zio-reference/` (done); `SHAS.md` labeled (done) | repo committed |
| 0.2 | **Build unmodified bun-debug:** `cd bun && bun bd` | build green |
| 0.3 | Prove the test loop: `bun bd test <one transpiler test>` from `test/bundler/transpiler/` | test green |
| 0.4 | Run effect-v4's own test suite with its normal toolchain (oracle health check) | suite green |

**Milestone 0 gate:** edit→build→test cycle works on this machine — this gates *the tracer
bullet going green*, not all work. (See "first hard part" below.)

> **STATUS (2026-06-12): build machine = the Mac.** This Windows machine has no native
> toolchain and stays toolchain-free by decision: it does Phase 1 (oracle + runtime map, stock
> Bun only) and red-test/syntax authoring. Builds and green runs happen on the user's Mac —
> see "Going green on the Mac" in `docs/slices/0001-pipeline-operator.md` (verified path:
> Homebrew `cmake ninja pkg-config libtool ccache llvm@21 rustup-init`, no sudo, no bootstrap
> script; WebKit is a prebuilt download into `~/.bun/build-cache`; ASAN is on by default in
> macOS debug builds — keep it, we want ASAN for the native phases). The branch travels as
> `patches/0001-*.patch` applied onto the pin. Windows-native install and WSL2 are off the
> table.
>
> **⚠ NEVER run `bun/scripts/bootstrap.ps1` on a personal machine** (CI image-bake script:
> unconditionally installs an OpenSSH server with a SYSTEM boot task granting every oven-sh
> GitHub org member SSH access, copies binaries into System32, can destructively prune the
> machine PATH — verified from source, bootstrap.ps1:315/104/512). The macOS `bootstrap.sh`
> has no SSH server but defaults CARGO_HOME/RUSTUP_HOME to `/opt/rust` and pulls CI tooling —
> skip it too; manual brew install is the whole job. Step 0.1 is done — the workspace repo
> exists with `bun/` and `effect-v4/` as proper git submodules.

### Phase 1 — Semantics extraction (no Bun changes; can run parallel to Phase 2)

- **1.1 v4 runtime map** (`docs/v4-runtime-map.md`): for each of the 17 primitives — constructor
  site, fields, exact `[evaluate]` behavior, stack interaction, interrupt behavior, with
  `effect.ts`/`core.ts` line references. Plus: `FiberImpl` lifecycle, `maxOpsBeforeYield` yield protocol,
  scheduler integration, `interruptChildren` semantics, how `Effect.gen` drives `Iterator`.
- **1.2 Oracle harness** (`oracle/`, plain TS run on stock Bun): program-description JSON format,
  seeded random program generator, runner that executes descriptions on the effect-v4 runtime and
  emits `{ exit, events[] }`.

**Gate:** 17/17 primitives documented from source; oracle replays 1,000 seeded programs byte-identically
across runs (determinism is a property of the harness we'll depend on for everything later).

### Phase 2 — P1 syntax behind flags (first user-visible win, ~3–4 weeks)

**Slice 2.1 = THE FIRST SLICE** (see §3). Minimal F#-pipe: token `TBarGreaterThan`
(`ast/lexer_tables.rs` punctuation block — enum order is load-bearing for `is_assign()` range
checks) + one inner `0x3E` arm in the lexer's `|` match (`lexer.rs:1771`); insert `Level::Pipe`
**between `Conditional` and `NullishCoalescing`** in `ast/op.rs:155` AND rewrite the
hand-maintained `Level::from_raw` decoder (`op.rs:214`) in the same edit (it compiles-but-
misdecodes otherwise); a new `sfx_` arm in `parse_suffix.rs` dispatch (~:1499) that guards
`level.gte(Level::Pipe)`, parses the RHS at `Level::Pipe` (left-assoc), and lowers directly to
`E::Call { target: rhs, args: [lhs] }` (no new `Op::Code`, no `TABLE` entry, no printer change —
synthetic calls are the established JSX-lowering idiom); gate on
`features.experimental_pipeline_operator` with `lexer.unexpected()` when off (ADR-0003). Babel
semantics per ADR-0002: ternary binds looser (`a |> b ? c : d` ⇒ `(a |> b) ? c : d`), `??`/`||`
bind tighter (`a |> b ?? c` ⇒ `a |> (b ?? c)`). One golden test in
`test/bundler/transpiler/pipeline-operator.test.ts`. Checklist traps: `clone_for_lazy_export`
(`parse_entry.rs:179`, compile-enforced) and the `[bool; 17]` cache-hash array
(`parser.rs:387`, NOT compile-enforced — required when the CLI surface lands).

**Slice 2.2** `*() =>`: extend the arrow path in `parse/parse_fn.rs` to accept a leading `*`,
set `is_generator` on the arrow, lower to a generator function expression that captures lexical
`this`/`arguments` via hoisted locals (reuse Bun's existing arrow-capture machinery; **never**
`.bind`).

**Slice 2.3** Flag plumbing + edge fixtures: `bunfig.toml [transpiler]` option + CLI flag +
per-file pragma, default OFF. Fixtures: ASI (`a\n|> b`), TS generics (`f<T>() |> g`),
arrow-RHS rule (`b |> (x => x.f())` required), precedence vs `??`/ternary/`await`/`yield`,
decorator interaction, concise arrow-generator bodies.

**Gate:** 100% of existing transpiler tests pass flag-OFF; all golden fixtures pass flag-ON.
Draft the upstreamable PR now — it banks a win even if everything after is killed.

### Phase 3 — Pure-Rust model outside Bun (~6–10 weeks)

Standalone crate `crates/effect-model` in this workspace (not in the bun tree yet):

- `Primitive` enum mirroring the 17 primitives; `Fiber { stack: Vec<ContFrame>, op_count, interrupt
  state, refs }`; `run_loop` mirroring `FiberImpl.runLoop` including the `maxOpsBeforeYield`
  budget and interrupt polling at op boundaries; a `Scheduler` trait (deterministic test
  scheduler first).
- JS values and user closures abstracted behind a `Host` trait — in the model, "closures" are
  table indices into the program description, so the model needs **no** JS engine.
- A CLI that consumes the same program-description JSON the oracle consumes and emits the same
  `{ exit, events[] }` shape.
- **Differential testing:** run oracle (JS) and model (Rust) over seeded corpora; compare exit
  values and full event logs. 3a: Wave-1 ops (sync). 3b: + `Async`/`AsyncFinalizer`/
  `SetInterruptible` + interruption races on the deterministic scheduler. 3c: + `Iterator`
  (modeled as a host-driven resumable sequence — the live-generator stand-in).

**Gate:** 100k seeded differential programs identical per wave; an overnight fuzz run finds zero
divergences. **Kill criterion:** if Wave-2 interruption semantics can't converge with the oracle
after two focused iterations, stop and ship Phases 1–2 only.

### Phase 4 — Bind the native runtime into Bun (~8–12 weeks, gated R&D)

New crate `bun_effect` inside the bun workspace, depending on `effect-model` with the `Host`
trait implemented over `bun_jsc`:

- JS values in continuation frames held as **`Strong` handles: root on push, release on pop**;
  the live `Effect.gen` iterator rooted for the fiber's lifetime; over-root rather than
  under-root (leak-safe beats use-after-free).
- Crossings into JS only for: user closures (`Sync`/`OnSuccess` callbacks, `iterator.next(v)`),
  JS-valued `Context.Reference` reads/writes, async settle/resume. Check the pending-exception
  state after **every** crossing; convert to a `Failure` primitive.
- Resumption is routed through Bun's existing event-loop task scheduling — never out-of-band —
  to preserve microtask ordering against user `await`/`Promise.then`. (The blueprint's "bypass
  the event loop" stays dead.)
- Exposure: a `bun:effect` builtin + an additive userland package that feature-detects the
  native runtime and swaps the v4 `FiberImpl` for a native-fiber handle. **Every unimplemented
  or unproven surface transparently falls back to the JS runtime** — fallback is a permanent
  architecture feature, not scaffolding.

**Sub-gates:** 4a sync-only ops pass the relevant effect-v4 `Effect` tests natively → 4b async +
interruption (interrupt-while-parked runs finalizers exactly once) → 4c `Iterator` with rooted
live generators, GC forced between every op in stress mode → 4d performance: ≥2× nested-flatMap
microbenchmark vs the JS runtime, no regression on the differential suite, leak-clean over 1M
fiber spawn/join cycles.

**Kill criteria (inherited from compass, still correct):** leak-free 1M spawn/join unreachable
after two focused iterations → ship transpiler-only. Vendored-WebKit/JSC rebase breaks rooting
twice in a quarter → freeze native, keep P1.

### Phase 5 — Unbroken traces (optional, last, ~6–8 weeks)

Per-fiber logical span stack captured at `OnSuccess`/`gen` boundaries; stitch into Bun's
V8-style `error.stack` construction (seams: `bun/src/jsc/CallFrame.rs`, `bun/src/sourcemap/`);
account for v4's own `Error.stackTraceLimit` clamping (effect.ts:317, 1141). Bun's sourcemap
remapping has a history of off-by-one bugs — treat P5 as droppable without shame.

---

## 3. The first slice, the first hard part, the big achievement

### First slice (the tracer bullet): `a |> b` end-to-end
One token in `lexer.rs` → one precedence entry in `js_ast::Op::Level` → one parse arm → one
AST rewrite to a call expression → one golden test under `test/bundler/transpiler/` → behind one
flag. It is deliberately the smallest change that exercises **every layer you'll need for
everything else**: the lexer, the parser, the AST, the lowering pipeline, flag plumbing, the
golden-test harness, and the full `bun bd` edit-build-test loop. When this slice is green you
have proven the fork is workable; nothing about it is throwaway.

### First hard part (in order of appearance)
1. **Milestone 0: `bun bd` on Windows.** Unglamorous, but it blocks literally everything:
   ~200-crate cargo workspace, LLVM/clang, vendored WebKit artifacts, codegen steps. Treat it as
   a real milestone with a timebox; fallback is building inside WSL2.
2. **First *engineering* hard part: the GC rooting contract (Phase 4a→4c).** Keeping every
   JS value in a native continuation frame alive (`Strong` on push, release on pop) and — the
   peak — keeping the **live single-shot generator** behind `Effect.gen` rooted and driven
   exactly once across arbitrary async boundaries, while JSC's conservative stack scan cannot
   see heap-allocated Rust structs. Under-rooting is a use-after-free; replaying the generator
   is the O(n²) correctness trap. The whole plan shape exists to de-risk this one cliff: by the
   time you face it, semantics are already proven (Phase 3), so any failure is a *rooting* bug,
   not a semantics bug.

### Big achievement (what done looks like)
An **unmodified effect-v4 program** runs on the fork with its fiber interpreter executing in
Rust inside Bun — identical observable behavior to the JS runtime on a 100k-program differential
suite, ≥2× faster on nested-flatMap hot loops, leak-clean over 1M spawn/join — with `|>` and
`*() =>` ergonomics, all behind default-off flags, with transparent JS fallback.

### Bottom line
The plan is shaped so **every phase ends with something shippable even if the next phase dies**:
- Phase 2 alone → flag-gated `|>` + `*() =>` in a Bun fork, upstreamable.
- Phase 3 → the definitive v4 runtime map + a differential oracle (valuable to upstream Effect
  itself, runtime-independent).
- Phase 4 → the actual prize: native Effect in Bun.
- Phase 5 is a bonus, never a dependency.

Port semantics, not files. The oracle is the referee. Flags keep the exit open.
