# Implementation Plan: Resilient Diameter Routing and Peer-Failure Demo

**Branch**: `001-diameter-routing-failover` | **Date**: 2026-09-06 |
**Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-diameter-routing-failover/spec.md`

## Summary

Build a deterministic, simulation-only C++11 command-line application that validates the modeled
Diameter and stateless RFC 8506 envelope subset, selects host/realm/default routes, correlates each
answer to one active attempt, performs one bounded failover attempt, rejects overload, and emits one
shared event model as human-readable or JSON output. Use a single-thread event loop, injected fake
clock and ID sources, immutable validated messages, bounded in-memory stores, and narrow interfaces;
do not add networking, external services, persistence, or application charging behavior.

## Technical Context

**Language/Version**: C++11 for application, domain, simulator, tests, and fuzz target

**Primary Dependencies**: CMake 3.16+; C++ standard library only at runtime; Catch2 2.13.10 for
tests; Clang/libFuzzer for decoder fuzzing

**Storage**: Bounded in-memory queue, active transaction maps, timer heap, retained-result FIFO, and
event buffer; no persistent storage

**Testing**: Catch2 unit, contract, state-transition, integration, and deterministic property tests;
Clang libFuzzer; ASan/UBSan; TSan only if threads are later introduced; gcovr branch coverage

**Target Platform**: Linux x86-64 with GCC 11+ or Clang 14+ compiling in strict C++11 mode

**Project Type**: Single command-line application with a reusable domain library

**Performance Goals**: Correctly process 10,000 concurrently active simulated transactions; repeat
each built-in scenario 100 times with byte-identical normalized JSON; keep every configured count at
or below its bound

**Constraints**: Single thread; fake monotonic time only; no sockets, DNS, wall-clock sleeps,
external services, mutable globals, manual ownership, exception leakage, or unbounded containers;
default limits are queue 16, in-flight 8, timers 16, retained results 64, and events 2048

**Scale/Scope**: One process, seven modeled base AVPs plus one opaque optional payload, stateless
RFC 8506 `EVENT_REQUEST` CCR/CCA envelopes only, primary/secondary peers, two total attempts, 100 ms
retry backoff, and a 1-second simulated shutdown drain

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Pre-Research Gate

| Principle | Evidence | Status |
|-----------|----------|--------|
| I. Focused Simulation-Only Scope | In-process CLI and simulators only; unsupported transports and procedures return typed errors | PASS |
| II. RFC-Traceable Diameter Semantics | Validation contracts and a standards traceability table cite exact source clauses and classify every modeled behavior | PASS |
| III. Transaction Correctness | Immutable identity, active hop-by-hop uniqueness, attempt-specific correlation, deterministic routing, and one terminal result are explicit | PASS |
| IV. Bounded Retry, Backpressure, and Shutdown | Every store is bounded; retry is allow-listed and capped; shutdown has a deterministic deadline outcome | PASS |
| V. C++11 and SOLID | Strict C++11, RAII/value types, no runtime dependency, and narrow clock/scheduler/peer/ID/event/result ports | PASS |
| VI. Test-First Verification | Test layers, fuzzing, sanitizers, deterministic properties, and 90% changed-logic branch coverage are planned | PASS |
| VII. Built-In Observability and Safe Data | One bounded event model feeds both formats and accepts synthetic data only | PASS |

**Gate result**: PASS. No constitutional deviation or complexity exception is required.

The current `/speckit-plan` instruction explicitly accepts the recommended actions for the two items
deferred by the preceding clarification run. They are approved planning inputs: duplicate host-route
or realm-route keys and multiple defaults are configuration errors; peer IDs within a route must be
unique. The 32-bit hop-by-hop source wraps and skips active values deterministically, returning
`ID_SPACE_EXHAUSTED` before admission only if its injected ID domain has no free value. Retained
results evict by oldest completion first; generated per-run operation sequences and an eviction
watermark return `RETENTION_EXPIRED` without an unbounded tombstone store.

### Post-Design Gate

| Principle | Phase 1 Evidence | Status |
|-----------|------------------|--------|
| I. Focused Simulation-Only Scope | CLI contract exposes built-in simulations only; data model has no transport or external-service entity | PASS |
| II. RFC-Traceable Diameter Semantics | Standards contract maps DIA-FR-001 through DIA-FR-021 and explicitly lists unsupported behavior | PASS |
| III. Transaction Correctness | Data model defines immutable identities, active-attempt matching, terminal state, and late/duplicate handling | PASS |
| IV. Bounded Retry, Backpressure, and Shutdown | Capacity policy, result eviction, timer ownership, two-attempt retry, and deadline transitions are explicit | PASS |
| V. C++11 and SOLID | Research fixes strict C++11 and the structure separates domain, ports, simulation, app, and presentation | PASS |
| VI. Test-First Verification | Quickstart covers test layers, ASan/UBSan, fuzzing, traceability, and 90% branch coverage | PASS |
| VII. Built-In Observability and Safe Data | JSON Schema and data model define one bounded synthetic event/result model for both formatters | PASS |

**Post-design gate result**: PASS. Phase 1 introduces no constitutional deviation.

## Project Structure

### Documentation (this feature)

```text
specs/001-diameter-routing-failover/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/
│   ├── cli.md
│   ├── error-codes.md
│   ├── event-output.schema.json
│   └── standards-traceability.md
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
CMakeLists.txt
app/
└── main.cpp
include/diameter_demo/
├── app/
├── domain/
├── ports/
├── presentation/
└── simulation/
src/
├── app/
├── domain/
├── presentation/
└── simulation/

tests/
├── contract/
├── integration/
├── property/
├── unit/
└── fuzz/
```

**Structure Decision**: Use one CMake project. `diameter_demo_domain` owns validation, routing,
transactions, retry, and capacity rules; simulation adapters implement the narrow ports; the app
layer orchestrates admission and shutdown; presentation owns CLI and two event formatters. Public
headers are grouped by ownership boundary, while tests mirror behavior rather than source folders.

## Complexity Tracking

No constitution violations require justification.
