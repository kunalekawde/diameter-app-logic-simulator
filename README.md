# Diameter Application Routing Logic Simulator

Simulation-only educational demo; not a conformant or production Diameter implementation.

## Purpose

This project specifies a deterministic simulator for the application logic around Diameter routing: route selection, request/answer correlation, peer failure, bounded retry and failover, overload backpressure, shutdown, and explainable event output.

It is designed for learning and verification. It does not implement a Diameter network stack or a deployable telecom network function.

## Modeled Behavior

- Diameter header and a small validated AVP subset
- Exact destination-host, destination-realm, and optional default routing
- Primary and secondary simulated peers
- Active hop-by-hop ID uniqueness and stable end-to-end identity across retry
- Attempt-specific answer correlation, including late and duplicate answers
- One bounded failover retry after 100 ms of simulated backoff
- Bounded queue, in-flight, timer, retained-result, and event capacity
- Deterministic shutdown with an explicit terminal result
- Human-readable and versioned JSON output from one event model

The RFC 8506 surface is limited to stateless EVENT_REQUEST CCR/CCA envelope validation. Credit rating, quota, account, subscriber, and service procedures are unsupported. 3GPP TS 29.230 is used only to validate assigned identifiers; no 3GPP Diameter application is implemented or advertised.

## Explicitly Out of Scope

- TCP, SCTP, TLS, DNS, sockets, or real peer connectivity
- Capabilities exchange, watchdog, and the complete Diameter peer state machine
- Persistent storage, external services, Kubernetes, or an observability platform
- Gx, Gy, S6a, or other 3GPP application procedures
- Production, interoperability, or conformance claims

## Environment

- Linux x86-64
- C++11 for application, simulation, tests, and fuzz targets
- CMake 3.16+
- GCC 11+ or Clang 14+
- C++ standard library only at runtime
- Catch2 2.13.10 for tests
- Clang/libFuzzer, ASan, UBSan, clang-tidy, and gcovr

The implementation is planned as a single-threaded command-line application with a reusable domain library. Time, scheduling, peer outcomes, and identifiers are injected deterministic simulations; tests never sleep or call external services.

## Default Limits

| Resource | Limit |
|---|---:|
| Queued requests | 16 |
| In-flight transactions | 8 |
| Timers | 16 |
| Retained results | 64 |
| Event records | 2048 |
| Total attempts | 2 |
| Retry backoff | 100 simulated ms |
| Shutdown drain | 1000 simulated ms |

## Architecture

The domain layer owns validation, routing, peer eligibility, transactions, correlation, retry, and capacity rules. Narrow ports isolate clocks, schedulers, peers, ID sources, event sinks, and result stores. Simulation adapters provide deterministic behavior, the application layer controls admission and shutdown, and presentation renders the shared event model.

## Standards Baseline

- RFC 6733, Diameter Base Protocol
- RFC 8506, Diameter Credit-Control Application, limited to the modeled EVENT_REQUEST envelope
- 3GPP TS 29.230 V19.3.0, identifier validation only

Every modeled rule is classified as exact, simplified, unsupported, or deviating in the standards traceability document. Full conformance requires external testing and is not claimed here.

## Documentation

The current repository contains specification and design artifacts; implementation has not started.

- [Feature specification](specs/001-diameter-routing-failover/spec.md)
- [Implementation plan](specs/001-diameter-routing-failover/plan.md)
- [Research decisions](specs/001-diameter-routing-failover/research.md)
- [Data model and state transitions](specs/001-diameter-routing-failover/data-model.md)
- [Validation quickstart](specs/001-diameter-routing-failover/quickstart.md)
- [CLI contract](specs/001-diameter-routing-failover/contracts/cli.md)
- [Stable errors](specs/001-diameter-routing-failover/contracts/error-codes.md)
- [JSON output schema](specs/001-diameter-routing-failover/contracts/event-output.schema.json)
- [Standards traceability](specs/001-diameter-routing-failover/contracts/standards-traceability.md)
- [Project constitution](docs/constitution.md)

## Status

Specification and Phase 1 design are complete. The next workflow step is task decomposition followed by test-first implementation.
