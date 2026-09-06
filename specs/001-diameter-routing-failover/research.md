# Phase 0 Research: Resilient Diameter Routing and Peer-Failure Demo

## C++ Toolchain

**Decision:** Use CMake 3.16+ with GCC 11+ and Clang 14+, forcing
`CMAKE_CXX_STANDARD=11`, `CMAKE_CXX_STANDARD_REQUIRED=ON`, and extensions off for every C++ target.

**Rationale:** These versions are readily available on Linux and support warnings, sanitizers,
coverage, static analysis, and libFuzzer without requiring newer language features in project code.

**Alternatives considered:** Autotools adds unnecessary project machinery. Bazel is excessive for a
single executable and library. Allowing C++17 in tests risks hiding production compatibility defects.

## Dependencies and Structured Output

**Decision:** Use only the C++ standard library at runtime. Serialize the fixed event document through
a dedicated JSON writer that owns string escaping, integer formatting, field order, and stream error
handling. Accept no external JSON input in this feature. Use Catch2 2.13.10 as the only test framework.

**Rationale:** The external surface is a fixed output schema, not a general JSON document model.
Keeping the writer behind one formatter interface avoids ad hoc serialization while eliminating a
large general-purpose runtime dependency. Catch2 2.13.10 is C++11-compatible and supports all planned
test layers; deterministic property tests can use explicit generated loops with recorded seeds.

**Alternatives considered:** nlohmann/json is correct but unnecessary for a fixed write-only schema.
RapidJSON adds a second template-heavy dependency. Combining GoogleTest and Catch2 duplicates test
infrastructure. Catch2 3.x requires a newer C++ standard and is rejected.

## Deterministic Execution Model

**Decision:** Use one thread, an injected monotonic clock measured in integer milliseconds, and a
bounded scheduler ordered by `(due_time, insertion_sequence)`. Scenario actions and same-time answers
use declaration order. No production path reads system time or sleeps.

**Rationale:** A total event order makes IDs, timing, failover, output, and tests reproducible while
avoiding locking and ThreadSanitizer scope.

**Alternatives considered:** Threads add races without educational value. Wall-clock timers make
timeouts slow and flaky. Ordering equal-time work by address or container iteration is non-portable.

## Routing and Peer Selection

**Decision:** Build immutable route configuration before admission. Reject duplicate
`(Destination-Host, Application-Id)` keys, duplicate `(Destination-Realm, Application-Id)` keys,
multiple defaults, and duplicate peer IDs within one route. Resolve host, then realm and application,
then default. Within one route, choose eligible primary then secondary in declaration order.

**Rationale:** Configuration rejection removes route ties instead of hiding them behind arbitrary
runtime precedence. It preserves the clarified known-host failure rule and deterministic selection.

**Alternatives considered:** First-wins and last-wins silently mask mistakes. Numeric peer weights
add behavior not required by the demo.

## Identifier and Retention Semantics

**Decision:** Generate per-run operation IDs monotonically. Allocate 32-bit hop-by-hop IDs by scanning
from the injected source's next value, wrapping from `UINT32_MAX` to zero and skipping active IDs. If
the injected ID domain is
fully occupied, reject before admission with `ID_SPACE_EXHAUSTED`. Keep end-to-end ID, Origin-Host,
Session-Id, CC-Request-Type, and CC-Request-Number stable across retry; allocate a new hop-by-hop ID.
Evict retained results by oldest completion first. Check active and retained operations before an
eviction watermark; a generated operation ID at or below that watermark returns `RETENTION_EXPIRED`.

**Rationale:** Active collisions can never corrupt answer correlation. A scalar watermark provides an
explicit expired outcome without unbounded tombstones. Retained entries take precedence over the
watermark, so out-of-order completion remains correct.

**Alternatives considered:** Immediate failure on the first collision wastes available IDs. Per-ID
tombstones grow without bound. LRU retention makes reads alter deterministic eviction behavior.

## Transactions, Retry, and Shutdown

**Decision:** Represent one admitted operation as one transaction containing at most two attempts.
Retry only timeout, disconnect-after-send, `DIAMETER_UNABLE_TO_DELIVER`, or `DIAMETER_TOO_BUSY` after
100 simulated milliseconds, and only to an eligible alternate for a non-fixed destination. Shutdown
stops admission, drains for 1 simulated second, then completes unfinished operations once with
`SHUTDOWN_DEADLINE` and cancels their scheduled work.

**Rationale:** This directly implements the clarified budget and preserves exactly one terminal result
under failure and shutdown.

**Alternatives considered:** More attempts increase duplicate exposure and demo complexity. Immediate
cancellation prevents demonstration of draining. Generic retry-by-result-class is broader than the
approved allow-list.

## Interface Contracts

**Decision:** Expose a small CLI with `list-scenarios` and `run <scenario>` commands plus `--seed` and
`--format human|json`. Built-in scenario names are stable. Human and JSON formatters consume the same
bounded event records. JSON is one versioned run document conforming to a checked-in JSON Schema.

**Rationale:** This is sufficient to submit, reproduce, and inspect demonstrations without a network,
configuration language, or application framework.

**Alternatives considered:** Interactive input complicates reproducibility. External scenario files
require an input schema and parser outside current scope. JSON Lines cannot express the mandatory
run-level disclaimer as clearly as one document.

## Verification Tooling

**Decision:** Use Catch2 for unit, simulator contract, state-transition, integration, and generated
property tests. Use Clang libFuzzer for bounded header/AVP decode input. CI runs GCC and Clang warning
builds, clang-format, clang-tidy, ASan/UBSan, and gcovr with 90% branch coverage on changed domain
logic. Add TSan only if a future approved amendment introduces threads.

**Rationale:** Each constitutional quality gate has one focused tool and no duplicate framework.

**Alternatives considered:** AFL requires a separate process workflow. TSan has no value in the
single-thread design. Whole-repository coverage can obscure the required changed-domain threshold.

## Standards Baseline and Traceability

**Decision:** Pin modeled behavior to RFC 6733, RFC 8506, and official 3GPP archive package
`29230-j30.zip` (TS 29.230 V19.3.0, Release 19), listed on 2026-01-06 and latest as of 2026-09-06.
Record exact clauses, classification, requirement IDs, and test evidence in the traceability contract.
Capture and record the package checksum during implementation when the archive is reachable.

**Rationale:** The package name and official index provide a stable baseline. The checksum remains an
explicit verification task because the binary archive was unavailable during planning. References
govern only modeled rows and do not establish full protocol or application conformance.

**Alternatives considered:** An unversioned "latest" reference is not reproducible. Copying the 3GPP
tables into source would add licensing and drift risk. Claiming full conformance violates scope.