# Staged-strict behavioral equivalence for Differential Runs

The Oracle's Event Log is the project's definition of "identical behavior" between the JS and
native runtimes. We decided on a staged-strict contract: v1 (MicroCore wave) compares
user-callback invocation order, finalizer order, and canonicalized Exit over *sequential*
programs only; v2 (async wave) adds concurrent programs under a deterministic scheduler and
mirrors v4's op-counting 1:1, so cross-fiber interleaving must match exactly. Rationale:
interleaving is observable to programs (the `maxOpsBeforeYield` budget determines it), so it
must eventually be in the contract — but sequential op-counts have no discriminating power, and
demanding them from day one would delay the first Differential Run and overfit the native
runtime to JS interpreter internals before any opcode exists.

## Considered Options

- **Strict from day one** — rejected: builds scheduler instrumentation before there is anything
  to discriminate.
- **Exit-only (loose)** — rejected: most expected port bugs (finalizer ordering, double-resume,
  lost interrupts) produce identical Exits with divergent ordering; the fuzzer would be blind to
  them.

## Consequences

- The Event Log schema must be versioned; v2 adds event kinds without changing v1 kinds, so v1
  corpora remain valid.
- A legitimate native optimization that changes interleaving is, by definition, a semantics
  break under this contract — relaxations must be made here first, deliberately, not in the
  native runtime.
