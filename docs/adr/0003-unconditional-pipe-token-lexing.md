# Lex `|>` unconditionally; gate the feature in the parser

Bun's Rust lexer has no runtime options (its only configuration is eight compile-time
const-generic JSON-mode bools), so `experimentalPipelineOperator` cannot gate tokenization.
We lex `|>` as a single token (`TBarGreaterThan`) unconditionally and gate in the parser:
flag-ON parses the pipe; flag-OFF calls `lexer.unexpected()`. Consequence we accept: with the
flag OFF, `a |> b` still fails to parse, but the error message changes from stock Bun's
"Unexpected >" to "Unexpected \"|>\"" — the only observable flag-OFF change, and strictly more
helpful. Precedent: `?.` is lexed unconditionally; TypeScript-isms are gated in the parser, not
the lexer.

## Considered Options

- **Parser-driven token upgrade** (lex `|` as `TBar`; when the flag is ON, upgrade `TBar`+`>`
  into a pipe via the `maybe_expand_equals` pattern) — rejected: zero flag-OFF drift, but more
  moving parts in the hot suffix loop and whitespace-adjacency rules (`a | > b` must not become
  a pipe), all purchased only to preserve an error-message string.
- **A ninth lexer const-generic** — rejected: threads through the `NewLexer` alias, the
  `lexer_impl_header!` macro, and every instantiation, and doubles monomorphizations.

## Consequences

- `|>=` now lexes as `|>` `=` (previously `|` `>=`); still a SyntaxError flag-OFF either way.
- TS type-skipping paths consume `TBar` for unions (`parse_skip_typescript.rs:347,772`); no
  valid TS has `|` immediately followed by `>`, but the new token variant needs conscious arms
  (not default-arm fall-through) in `parse_suffix` dispatch and the `typescript.rs:79-87`
  token-class list, plus TS fixtures proving type positions are undisturbed.
- Before pinning any error-string assertion, grep the existing test corpus for assertions on
  the old "Unexpected >" message for pipe-like inputs.
