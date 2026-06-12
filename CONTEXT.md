# Bun-Native Effect

Porting the Effect v4 fiber runtime semantics into a Bun fork (native Rust runtime + flag-gated
syntax), with the unmodified effect-v4 codebase as the executable specification.

## Language

### Verification

**Oracle**:
The unmodified effect-v4 runtime executing Program Descriptions to produce the reference
behavior every other runtime must reproduce.
_Avoid_: reference implementation, golden runtime

**Program Description**:
A runtime-agnostic JSON document describing an effect program; the only input format the
Oracle and native runtimes execute.
_Avoid_: test case, program AST, fixture

**Event Log**:
The ordered record of observable behavior produced by executing a Program Description.
Equality of Event Logs (plus Exits) is the project's definition of behavioral equivalence.
_Avoid_: trace — "trace" is reserved for stack traces and Effect tracer spans

**Differential Run**:
Executing one Program Description on two runtimes and comparing Exit and Event Log.
_Avoid_: A/B test, comparison test

### Runtime

**MicroCore**:
The thirteen synchronous primitives of the seventeen v4 primitives, implemented first in the
native model: Sync, Suspend, exitPrimitive, Success, Failure, WithFiber, YieldableError,
OnSuccess, OnFailure, OnSuccessAndFailure, OnExit, Yield, While.
_Avoid_: Micro — Effect v4 has no Micro module; "Micro" refers only to the v3 artifact

**JS Fallback**:
The permanent ability to run any surface on the unmodified JS runtime when the native path is
missing or unproven.
_Avoid_: shim, polyfill, escape hatch

### Syntax

**Effect Syntax**:
The family of flag-gated, default-off Bun syntax extensions motivated by Effect (`|>`,
`*() =>`). Each feature has its own per-feature experimental flag (`experimentalPipelineOperator`,
`experimentalArrowGenerators`); there is no umbrella flag.
_Avoid_: dialect, effect mode

### Process

**Tracer Bullet**:
The first vertical slice — minimal `a |> b` → `b(a)` — chosen because it exercises every layer
(lexer, parser, AST, lowering, flag plumbing, golden test, build loop) at the smallest size.
_Avoid_: spike, MVP, proof of concept

**Pin**:
The recorded submodule SHA of a dependency repo. Pins advance only at phase boundaries, and
every advance re-runs the full differential suite.
_Avoid_: version, snapshot
