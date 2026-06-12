# F#-style pipe with Babel-fsharp-minimal grammar

Bun's flag-gated `|>` implements F#-style semantics (`a |> b` ⇒ `b(a)`, left-associative,
unary-call lowering) with exactly Babel's fsharp-proposal grammar: the RHS never greedily
swallows a trailing arrow — `b |> x => x.f()` is a parse error; `b |> (x => x.f())` is required
— and there are no topic placeholders and no special `await` step form. This is a deliberate,
permanent divergence from TC39 (the committee's surviving direction is Hack-style; F#-style was
rejected twice and is dead there): Effect's entire API is unary/data-last, which is what
F#-style composes with, and the feature is default-off behind a flag so the divergence never
leaks into standard code paths.

## Precedence interactions (pinned)

"Babel semantics" means concretely: ternary binds looser (`a |> b ? c : d` ⇒ `(a |> b) ? c : d`),
assignment binds looser (`x = a |> b` ⇒ `x = (a |> b)`), and `??`/`||` and everything tighter
bind into the RHS (`a |> b ?? c` ⇒ `a |> (b ?? c)`). The TC39 F# draft's between-assignment-and-
ternary slot would instead give `a |> (b ? c : d)` — explicitly rejected; in Bun's `Level`
ladder, `Pipe` sits between `Conditional` and `NullishCoalescing`.

## Considered Options

- **Hack-style (TC39-aligned)** — rejected: topic placeholders are strictly worse ergonomics
  for Effect's unary combinator chains, which are the only reason this feature exists.
- **Arrow-friendly precedence** (`a |> x => e` parses without parens) — rejected: novel,
  un-stress-tested grammar; ASI/ternary/yield ambiguity becomes our research problem instead of
  inheriting Babel's known-good rule.
- **F#-proposal-faithful** (including `|> await`) — rejected: extra parse rule and semantics
  for fidelity to a permanently stalled draft.
