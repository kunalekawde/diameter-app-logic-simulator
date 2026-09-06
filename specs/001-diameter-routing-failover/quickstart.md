# Quickstart Validation Guide

This guide validates the planned feature after implementation. Contract details live in
[contracts/cli.md](contracts/cli.md), [contracts/error-codes.md](contracts/error-codes.md), and
[data-model.md](data-model.md).

## Prerequisites

- Linux x86-64
- CMake 3.16 or newer
- GCC 11 or newer and Clang 14 or newer
- `check-jsonschema` for output contract validation
- `gcovr`, `clang-format`, and `clang-tidy`

## Configure and Build

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --parallel
ctest --test-dir build --output-on-failure
```

The build must compile all project C++ targets in C++11 mode with compiler extensions disabled and
warnings treated as errors.

## Discover Scenarios

```bash
./build/diameter-routing-demo list-scenarios
./build/diameter-routing-demo list-scenarios --format json
```

Both outputs begin with or contain the mandatory simulation-only disclaimer and list the ten stable
scenario names from the CLI contract.

## Validate Core Routing

```bash
./build/diameter-routing-demo run healthy-host --seed 0 --format human
./build/diameter-routing-demo run realm-fallback --seed 0 --format json > build/realm.json
```

Expected outcomes:

- `healthy-host` selects the exact host's primary and completes one request with one answer.
- `realm-fallback` records why direct host routing did not apply and selects the realm route.
- Each admitted operation has one terminal result and no retry occurs.

## Validate Failover and Correlation

```bash
./build/diameter-routing-demo run retry-failover --seed 7 --format json > build/failover.json
./build/diameter-routing-demo run late-duplicate-answer --seed 7 --format human
./build/diameter-routing-demo run retry-exhaustion --seed 7 --format human
```

Expected outcomes:

- Failover has two total attempts separated by 100 simulated milliseconds.
- Attempt two uses a new hop-by-hop ID, stable end-to-end identity, and the T flag.
- Late and duplicate answers are classified but do not alter the terminal result.
- Exhaustion returns `RETRY_EXHAUSTED` once and releases transaction capacity.

## Validate Backpressure and Shutdown

```bash
./build/diameter-routing-demo run queue-full --seed 0 --format human
./build/diameter-routing-demo run recovery --seed 0 --format human
./build/diameter-routing-demo run graceful-shutdown --seed 0 --format json > build/shutdown.json
```

Expected outcomes:

- The seventeenth queued request is rejected with `QUEUE_FULL` before ID allocation when the queue
  limit of 16 is reached.
- New work is admitted after capacity is released.
- Shutdown rejects new admission, drains for 1000 simulated milliseconds, and gives every unfinished
  admission exactly one `SHUTDOWN_DEADLINE` result while cancelling its timers and retries.

## Validate Malformed and Unsupported Input

```bash
./build/diameter-routing-demo run malformed-input --seed 0 --format human
```

The first validation failure names the stable code and offending field or AVP. No transaction ID is
allocated and no peer send occurs. Unsupported CC request types and 3GPP identifiers have the typed
outcomes in the error contract.

## Validate JSON Contract and Reproducibility

```bash
check-jsonschema --schemafile \
  specs/001-diameter-routing-failover/contracts/event-output.schema.json build/realm.json
check-jsonschema --schemafile \
  specs/001-diameter-routing-failover/contracts/event-output.schema.json build/failover.json
check-jsonschema --schemafile \
  specs/001-diameter-routing-failover/contracts/event-output.schema.json build/shutdown.json
./build/diameter-routing-demo run retry-failover --seed 7 --format json > build/replay-a.json
./build/diameter-routing-demo run retry-failover --seed 7 --format json > build/replay-b.json
cmp build/replay-a.json build/replay-b.json
```

Schema validation and `cmp` must succeed.

## Required Quality Gates

```bash
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON -DENABLE_UBSAN=ON
cmake --build build-asan --parallel
ctest --test-dir build-asan --output-on-failure
cmake -S . -B build-coverage -DCMAKE_BUILD_TYPE=Debug -DENABLE_COVERAGE=ON
cmake --build build-coverage --parallel
ctest --test-dir build-coverage --output-on-failure
gcovr --root . --filter 'src/domain/' --branch --fail-under-branch 90 build-coverage
cmake -S . -B build-fuzz -DCMAKE_CXX_COMPILER=clang++ -DENABLE_FUZZING=ON
cmake --build build-fuzz --parallel
./build-fuzz/diameter_decoder_fuzz -max_total_time=60 tests/fuzz/corpus
```

Formatting, warnings-as-errors, clang-tidy, all tests, ASan, UBSan, decoder fuzzing, traceability, and
at least 90% branch coverage for changed domain logic must pass. TSan is intentionally omitted because
the design is single-threaded.