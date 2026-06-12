# Slice 0001 — Tracer Bullet: `a |> b` → `b(a)`

Status: **RED** (test written and failing under stock Bun 1.4.0; green blocked on toolchain —
see PLAN.md Milestone 0 STATUS). Branch: `effect/p1-pipeline-operator` in `bun/`.
Spec: ADR-0002 (grammar), ADR-0003 (token gating). All paths relative to `bun/`.

## Red test (exists)

`test/bundler/transpiler/pipeline-operator.test.ts` — confirmed failing with
`BuildMessage: Unexpected >` (option silently ignored by stock Bun). Repo validity rule
satisfied: fails under system Bun, must pass under `bun bd test` once implemented.

## Implementation recipe (verified seams, exact lines at pin bcad9b220e)

1. **Token** — `src/ast/lexer_tables.rs`: add `TBarGreaterThan,` right after `TBarBar` (:34),
   inside the punctuation block only (enum order is load-bearing: `is_assign()` :135 is a
   discriminant range). Add `token_enums[T::TBarGreaterThan as usize] = b"\"|>\"";` near :324
   (unset entries default to `b""` → empty error messages). No token codegen exists;
   `defines_table.*` is unrelated; never touch `.zig` siblings.
2. **Lexer** — `src/js_parser/lexer.rs`, the `0x7C` arm (:1771-1801): add inner arm
   `0x3E => { self.step_with(contents); self.token = T::TBarGreaterThan; }` before the
   `_ => TBar` fallback; update the arm comment. (`||>` still lexes as `||`+`>`; `|>=` becomes
   `|>`+`=` — both remain SyntaxErrors flag-off.)
3. **Precedence** — `src/ast/op.rs`: insert `Pipe` in `Level` (:155) **between `Conditional`
   and `NullishCoalescing`**, and rewrite `Level::from_raw` (:214-245) in the same edit — it is
   a hand-maintained numeric match that compiles-but-misdecodes after insertion (reachable via
   `add_f`/`sub`). No `Op::Code` variant, no `TABLE` entry, no printer change. Beware: `bun_ast`
   has a second, unrelated logging `Level` at `lib.rs:1572`.
4. **Parse arm** — `src/js_parser/parse/parse_suffix.rs`: add
   `T::TBarGreaterThan => Self::sfx_t_bar_greater_than(p, level, left)` to the dispatch (~:1499),
   modeled on `sfx_t_question_question` (:1088-1105): if flag off → `p.lexer.unexpected()`;
   guard `level.gte(Level::Pipe) → Done`; `p.lexer.next()`; `right = p.parse_expr(Level::Pipe)`
   (same-level RHS = left-assoc); build
   `p.new_expr(E::Call { target: right, args: ExprNodeList::init_one(prev), ..Default::default() }, loc)`
   (synthetic-call idiom: parse_suffix.rs:387, p.rs:1166; printer guards `close_paren_loc`, so
   `Loc::EMPTY` is sourcemap-safe). Also add `TBarGreaterThan` consciously to the
   `typescript.rs:79-87` token-class list decision (no default-arm fall-through).
5. **Flag** — `experimental_pipeline_operator: bool`:
   - `Runtime::Features` struct `src/js_parser/parser.rs:200` + `Default` (:300) = false;
   - compile-enforced touch point: `clone_for_lazy_export` (`parse/parse_entry.rs:179`);
   - NOT compile-enforced trap: append to the `[bool; 17]` in `hash_for_runtime_transpiler`
     (`parser.rs:387`) — Bun.Transpiler path has no cache, but do it now anyway;
   - `api::TransformOptions` field (`src/options_types/schema.rs:140`);
   - decode `experimentalPipelineOperator` in `Config::from_js`
     (`src/runtime/api/JSTranspiler.rs` ~:409, `get_boolean_loose` pattern);
   - thread through `BundleOptions::from_api` (`src/bundler/options.rs:1712`) into the
     ParserOptions assembly (`src/bundler/transpiler.rs:1606-1671`). (Second assembly site
     `bundler/ParseTask.rs:~2504` = `bun build` path — deliberately NOT wired in this slice.)
6. **Verify** — `cargo check -p bun_ast -p bun_js_parser`, then `bun bd test
   test/bundler/transpiler/pipeline-operator.test.ts`; then full flag-off suite
   `bun bd test test/bundler/transpiler/`.

## Going green on the Mac (verified against scripts/build/*.ts at the pin)

The red test travels as a patch; the Mac never needs this Windows machine's git state.

```bash
# 1. Toolchain — Homebrew only, no sudo, no bootstrap.sh
brew install cmake ninja pkg-config libtool ccache llvm@21 rustup-init
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh   # nightly pin auto-installs on first build
export PATH="$(brew --prefix llvm@21)/bin:$PATH"                  # clang-21 must win; add to ~/.zprofile
# Skip go/ruby/icu4c/gnu-sed from CONTRIBUTING's list — unused for prebuilt-WebKit debug builds.

# 2. Workspace at the pin + the branch
git clone <this-workspace-repo> bun-native-effect && cd bun-native-effect
git submodule update --init bun effect-v4        # bun lands on pin bcad9b220e
cd bun && git checkout -b effect/p1-pipeline-operator
git am ../patches/0001-*.patch                   # the red test, commit fd7b6437a3

# 3. Milestone 0 proper: unmodified-ish debug build + red confirmed under the debug build
bun bd                                           # downloads prebuilt WebKit into ~/.bun/build-cache
bun bd test test/bundler/transpiler/pipeline-operator.test.ts   # expect: still RED ("Unexpected >")

# 4. Implement the recipe above → re-run → GREEN, then the flag-off suite:
bun bd test test/bundler/transpiler/
```

Notes: ASAN is on by default in macOS debug builds (≈2× build time) — keep it; the native
phases need it anyway (`bun bd --asan=off` exists if iteration speed hurts). WebKit is never
compiled locally unless you pass `--webkit=local`. If `clang-21` isn't found, the brew llvm@21
PATH export is missing — the build enforces LLVM 21.1.x exactly.

## TDD cycle backlog (one red→green per cycle, in order — do NOT write these tests in bulk)

1. ✅(red) `a |> b` → `b(a);\n` with flag on — THE tracer.
2. Left-assoc chain: `a |> f |> g` → `g(f(a))`.
3. Flag-off guard: `a |> b` still a parse error (pin the new `Unexpected "|>"` message —
   corpus grep confirmed nothing asserts the old message for `|>` inputs).
4. Arrow-RHS rule (ADR-0002): `b |> x => x.f()` is a parse error; `b |> (x => x.f())` lowers
   to a call of the parenthesized arrow.
5. Precedence fixtures (ADR-0002 pinned): `a |> b ? c : d` ⇒ `(a |> b) ? c : d`;
   `x = a |> b` ⇒ `x = (a |> b)`; `a |> b ?? c` ⇒ `a |> (b ?? c)`.
6. TS non-interference: type unions (`type A = B | C`), generics error fixtures
   (`f<x> > g<y>` still "Unexpected >"), `.ts` loader pipe usage.
7. ASI: `a\n|> b` behavior fixture.
