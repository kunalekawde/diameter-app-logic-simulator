# Diameter Routing Demo Constitution

## Core Principles

### I. Focused Simulation-Only Scope

The application MUST remain an independent, simulation-only C++ demonstration. It MUST model only the header, AVP, route, peer, and transaction subset required by its specification. Ingress, peer egress, answers, faults, time, IDs, and scheduling MUST use deterministic in-process simulators. Sockets, TCP, SCTP, DNS, telecom network functions, Kubernetes, and external observability platforms MUST NOT be runtime dependencies. Unsupported commands, AVPs, transports, and application procedures MUST return typed errors. Every README and human or JSON output MUST state exactly: "Simulation-only educational demo; not a conformant or production Diameter implementation." This boundary keeps the demo reproducible and prevents educational simplifications from being mistaken for production behavior.

### II. RFC-Traceable Diameter Semantics

Requirements and tests for modeled Diameter base behavior MUST cite the applicable RFC 6733 sections. When SCTP streams appear as metadata, documentation MUST cite RFC 9260 and state that SCTP transport is not implemented. Validation MUST occur before routing and cover header version, length, flags, command and application IDs, transaction IDs, and AVP flags, length, padding, and cardinality. Application IDs MUST NOT be described as implementations of Gx, Gy, S6a, or any other 3GPP application. A traceability table MUST classify behavior as exact, simplified, unsupported, or deviating. Conformance claims MUST require external testing. These rules make every deliberate simplification visible and reviewable.

### III. Transaction Correctness

Every admitted request MUST receive exactly one terminal result. Active hop-by-hop IDs MUST be unique, while end-to-end identity MUST remain stable across eligible retries. Answers MUST correlate only to the expected active transaction and attempt; unknown, mismatched, duplicate, and late answers MUST NOT complete another transaction. Routing MUST be deterministic in this order: exact host, realm, then an optional default. Peer state transitions MUST be explicit and atomic, and invalid transitions MUST preserve the last valid state. Uncertainty, timeout, route failure, retry exhaustion, and rejection MUST NOT become success. Operation IDs MUST be deduplicated as IN_PROGRESS, a retained result, or an explicit retention-expired outcome. These invariants prevent ambiguous ownership and false completion.

### IV. Bounded Retry, Backpressure, and Shutdown

Input queues, in-flight transactions, timers, completed results, and event records MUST have configured bounds. Saturation MUST return typed overload; silent drops and unbounded growth are prohibited. Retry MUST be limited to allow-listed transient failures, a configured maximum attempt count, stable end-to-end identity, deterministic backoff, and an eligible peer. Validation failures, malformed answers, route-not-found outcomes, application rejections, and non-retryable results MUST NOT be retried. Timeouts and backoff MUST use a fake monotonic clock, and tests MUST NOT sleep. Shutdown MUST stop admission, drain until a configured simulated deadline, and classify all unfinished work. These limits make overload, recovery, and termination deterministic under every tested load.

### V. C++11 and SOLID

The application MUST compile in C++11 mode and use no language or library facility newer than the project's explicitly approved C++11-to-C++17 toolchain policy. Domain code MUST use RAII, value types, immutable validated messages, explicit ownership, smart pointers where ownership requires them, and typed results. Manual new or delete, owning raw pointers, mutable globals, unchecked narrowing, ignored errors, broad exception catches, and fallback-to-success behavior are prohibited. Components MUST depend on narrow interfaces for clocks, schedulers, peers, ID sources, event sinks, and result stores. Validation, routing, peer state, transactions, retry, capacity, and presentation MUST remain separate responsibilities. Exceptions MUST NOT cross application boundaries; stable error codes MUST represent boundary failures. These constraints keep ownership and failure behavior explicit.

### VI. Test-First Verification

Test-first development is NON-NEGOTIABLE: a behavior test MUST be written and observed failing before its implementation is added. The suite MUST include unit tests for validation, routing, correlation, and retry; simulator contract tests; state-transition tests; end-to-end scenarios; property tests for uniqueness, determinism, and bounds; and decoder fuzzing. It MUST cover healthy routing, fallback, peer failure, late and duplicate answers, retry exhaustion, malformed input, overload, recovery, and shutdown. Tests MUST NOT use wall time, external services, sockets, DNS, or unrecorded random seeds. CI MUST run AddressSanitizer and UndefinedBehaviorSanitizer, plus ThreadSanitizer if multiple threads are introduced. Changed domain logic MUST achieve at least 90% branch coverage; uncovered RFC branches MUST have a recorded justification. Every defect fix MUST add a regression test. This evidence is the acceptance criterion for correctness claims.

### VII. Built-In Observability and Safe Data

Every event MUST include schema version, run or scenario ID, sequence, simulated time, component, operation and transaction IDs, attempt, peer, severity, and safe detail. Validation, routing, selection, queues, sends, peer state, timeout, retry, correlation, overload, and terminal results MUST emit observable events. Human-readable and JSON output MUST derive from one event model. All realms, hosts, IDs, and payloads MUST be synthetic; secrets and real subscriber data MUST NOT be accepted into logs or fixtures. This event contract makes deterministic scenarios diagnosable without exposing sensitive data.

## Technical Constraints

The implementation MUST be an in-process C++ application whose production target compiles in C++11 mode. Build and analysis tooling MAY use C++17 where the implementation plan records the exact tool, purpose, and compatibility boundary. The application MUST expose typed validation, routing, transaction, overload, shutdown, and unsupported-operation outcomes with stable codes. Simulation configuration MUST state queue, transaction, timer, result, event, retry, and shutdown limits. No feature may introduce a real network transport or external service dependency without a constitution amendment.

## Development Workflow and Quality Gates

Clarification MUST resolve applicable RFC sections, identifier semantics, route ties, capacities, retry policy, and shutdown behavior before planning. Planning MUST record the C++17-capable toolchain, C++11 production target, state machines, error model, bounded queues, simulator contracts, and RFC traceability. Tasks MUST order each failing test before its implementation. Analysis MUST show no unresolved requirement, design, or task gap before implementation begins. CI MUST pass formatting, warnings-as-errors compilation, static analysis, tests, required sanitizers, branch coverage, and traceability checks. Completion requires convergence with no unbuilt requirement. Reviews MUST reject changes that lack deterministic evidence for affected transaction, retry, capacity, or shutdown invariants.

## Governance

This constitution supersedes convenience choices and conflicting project guidance. Each amendment MUST document its rationale, impact analysis, affected artifacts, migration steps, and approval in a reviewable change. A deviation MUST have a decision record naming its rationale, risk, compensating control, owner, and removal date; an undocumented deviation is non-compliant.

Versions follow semantic versioning: MAJOR for incompatible governance or principle removals and redefinitions, MINOR for new principles or materially expanded obligations, and PATCH for non-semantic clarifications. Every amendment MUST update the impact report, version, and Last Amended date. Reviewers MUST verify constitution compliance in specifications, plans, tasks, code, and CI evidence. Release readiness MUST include an explicit compliance review, and unresolved violations MUST block release.

**Version**: 1.0.0 | **Ratified**: 2026-09-06 | **Last Amended**: 2026-09-06
