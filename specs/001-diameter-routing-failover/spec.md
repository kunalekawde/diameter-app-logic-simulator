# Feature Specification: Resilient Diameter Routing and Peer-Failure Demo

**Feature Branch**: Not created (no `before_specify` hook configured)

**Created**: 2026-09-06

**Status**: Draft

**Input**: User description: "Use `speckit.specify.diameter.md` as reference; strictly comply
with RFC 6733, RFC 8506, and 3GPP TS 29.230; include only essential behavior."

## Clarifications

### Session 2026-09-06

- Q: When `Destination-Host` names a known but unavailable peer, should routing fail instead of
  falling back to the realm route? → A: Known unavailable host fails; unknown host may use realm
  routing.
- Q: Which RFC 8506 Credit-Control-Request types should the demo accept as modeled routing
  envelopes? → A: Accept only `EVENT_REQUEST`.
- Q: Which default capacity profile should built-in scenarios use? → A: Queue 16, in-flight 8,
  timers 16, retained results 64, and events 2048; wait in FIFO order at the in-flight limit and
  reject new admission when another applicable bound is reached.
- Q: Which retry budget and backoff policy should apply after a peer accepts a send but delivery
  fails? → A: Two total attempts with fixed 100 ms simulated backoff; retry only timeout,
  disconnect-after-send, `DIAMETER_UNABLE_TO_DELIVER`, or `DIAMETER_TOO_BUSY` on an eligible
  non-fixed destination.
- Q: What should happen when the simulated shutdown deadline expires with unfinished admitted work?
  → A: Drain for 1 simulated second, then complete each unfinished admission once with
  `SHUTDOWN_DEADLINE`.

## User Scenarios & Testing *(mandatory)*

### Terms

- **Message header**: Routing and correlation fields carried before message attributes.
- **AVP**: A typed attribute carrying an identity, destination, result, or other message value.
- **Peer**: A simulated next destination that can accept a request or exhibit a scripted failure.
- **Transaction**: The tracked request from admission through its single final result.
- **Attempt**: One send of a transaction to one selected peer.

### User Story 1 - Explain a Successful Route (Priority: P1)

As a learner, I can submit a valid simulated Diameter request and inspect why a route and peer were
selected and how its answer was correlated.

**Why this priority**: Correct routing and correlation are the minimum useful demonstration.

**Independent Test**: Run one healthy-primary scenario and verify that one request produces one
correlated answer, a complete decision trail, and no retry.

**Acceptance Scenarios**:

1. **Given** matching host and realm routes, **When** a valid request names the host, **Then** the
   host route wins and the selected peer's answer completes only that request.
2. **Given** no Destination-Host or no known direct host entry and a matching realm route, **When** a
  valid request is submitted, **Then** the realm route is selected and the fallback reason is
  recorded.
3. **Given** malformed header or AVP data, an unsupported command or application, or a missing
   required routing AVP, **When** the input is submitted, **Then** it receives the applicable typed
   failure before transaction state or peer work is created.

---

### User Story 2 - Survive an Eligible Peer Failure (Priority: P2)

As a learner, I can fail or delay a primary peer and observe standards-traceable failover without
corrupting transaction state.

**Why this priority**: Failure handling is the central correctness risk after basic routing works.

**Independent Test**: Script one retry-eligible primary failure and verify one secondary attempt,
stable end-to-end identity, a new active hop-by-hop identity, and no cross-request completion.

**Acceptance Scenarios**:

1. **Given** the primary is unavailable before send, **When** candidates are selected, **Then** an
   eligible secondary is selected without counting a retransmission.
2. **Given** a sent request receives no answer and policy permits failover, **When** its deterministic
   timeout and backoff expire, **Then** it is retransmitted once to an eligible secondary with the
   T flag set and unchanged end-to-end identity.
3. **Given** the original answer arrives after failover, **When** it is processed, **Then** it is
   classified as late or duplicate and cannot complete any active transaction.
4. **Given** an application rejection, malformed answer, fixed unavailable destination, unsupported
  request type, or exhausted retry budget, **When** failure handling runs, **Then** no ineligible
  retry occurs and exactly one typed terminal failure is produced.

---

### User Story 3 - Apply Bounded Backpressure (Priority: P3)

As an operator, I can set small capacity limits and see excess work rejected predictably.

**Why this priority**: Bounded admission prevents overload from invalidating transaction guarantees.

**Independent Test**: Fill queue and in-flight capacity with delayed work and verify that the next
request receives typed overload without identifier allocation or silent loss.

**Acceptance Scenarios**:

1. **Given** a full input queue, **When** another request arrives, **Then** it receives `QUEUE_FULL`
   and no transaction identifiers are allocated.
2. **Given** the in-flight limit is reached, **When** queued work is considered, **Then** configured
   queue and in-flight bounds are preserved.
3. **Given** capacity is released, **When** a new request arrives, **Then** it is accepted without a
   restart.
4. **Given** shutdown has begun, **When** new or unfinished work is considered, **Then** admission is
   rejected and every admitted request is drained or classified at the simulated deadline.

---

### User Story 4 - Reproduce and Inspect Failures (Priority: P4)

As a developer, I can rerun a named scenario and receive equivalent human-readable and JSON evidence.

**Why this priority**: Reproducibility makes the demo useful for learning and defect diagnosis.

**Independent Test**: Run the same named scenario twice and compare its normalized JSON records.

**Acceptance Scenarios**:

1. **Given** the same scenario and seed, **When** it is rerun, **Then** event order, simulated times,
   identifiers, route choices, attempts, and terminal results are byte-identical after normalization.
2. **Given** any terminal result, **When** its operation ID is inspected, **Then** route, peer-state,
   transaction, timer, queue, and correlation events explain that result.
3. **Given** any supplied scenario, **When** output is inspected, **Then** it contains only synthetic
   hosts, realms, identifiers, and payloads.

### Edge Cases

- Hop-by-hop identifier generation wraps or collides with an active identifier.
- An answer has an unknown hop-by-hop identifier or mismatched command, application, end-to-end,
  Session-Id, CC-Request-Type, or CC-Request-Number value.
- A peer becomes unavailable after selection but before simulated send acceptance.
- A peer recovers during deterministic backoff, or two answers share a simulated timestamp.
- A default route has no eligible peer, or a fixed Destination-Host is unavailable.
- Queue or in-flight capacity is zero, completed-result retention expires, or an event limit is hit.
- A malformed or duplicate answer arrives after transaction expiry.
- Shutdown reaches its deadline while timeout or retry work is scheduled.

## Requirements *(mandatory)*

### Functional Requirements

- **DIA-FR-001**: The system MUST state in every human and JSON output: "Simulation-only educational
  demo; not a conformant or production Diameter implementation."
- **DIA-FR-002**: The system MUST validate, before routing, the RFC 6733 section 3 header rules:
  version 1; total length including padded AVPs; legal R/P/E/T and reserved bits; supported command
  code and application ID; and unsigned 32-bit hop-by-hop and end-to-end identifiers.
- **DIA-FR-003**: The system MUST validate the RFC 6733 sections 4 and 4.1 AVP code, V/M/reserved flags,
  optional Vendor-Id presence, minimum and declared length, data type, zero padding, and modeled
  command cardinality. The first detected AVP error MUST identify the offending or missing AVP.
- **DIA-FR-004**: The modeled base AVPs MUST be limited to Session-Id, Origin-Host, Origin-Realm,
  Destination-Host, Destination-Realm, Result-Code, Failed-AVP, and one non-mandatory opaque synthetic
  payload. Diameter identities MUST use valid synthetic ASCII host or realm values.
- **DIA-FR-005**: For the explicitly supported Credit-Control-Request and Credit-Control-Answer
  envelope subset, the system MUST accept only `EVENT_REQUEST` (value 4), require
  CC-Request-Number 0 and a valid Requested-Action, and validate application ID 4, command code 272,
  request/answer flags, required envelope AVPs, AVP occurrence, and matching Session-Id,
  CC-Request-Type, and CC-Request-Number against RFC 8506 sections 3.1, 3.2, 6, 8.2, 8.3, and 10.1.
  Other CC-Request-Type values MUST return an unsupported-operation error before transaction creation.
  Envelope validation MUST NOT be represented as implementation of credit rating, balances, quota,
  subscriber procedures, or the complete Diameter Credit-Control application.
- **DIA-FR-006**: 3GPP identifiers MUST be validated against 3GPP TS 29.230 clause 4 and Tables 5.1,
  6.1, and 7.1 from official archive package `29230-j30.zip`, the latest package available on
  2026-09-06. Vendor-Id 10415 MUST appear only with an explicitly supported 3GPP vendor-specific AVP;
  no such AVP is supported in this feature.
- **DIA-FR-007**: Unsupported commands, AVPs with the M bit set, application IDs, 3GPP identifiers,
  transports, and application procedures MUST receive a stable typed error and MUST NOT be presented
  as implemented Diameter or 3GPP functionality.
- **DIA-FR-008**: Route selection MUST use an exact Destination-Host route when a direct peer entry
  exists. If that known fixed destination is unavailable, routing MUST return
  `DIAMETER_UNABLE_TO_DELIVER` without realm fallback. If Destination-Host is absent or has no known
  direct peer entry, routing MUST use exact Destination-Realm plus application ID, then the optional
  default route. No eligible route MUST produce `DIAMETER_UNABLE_TO_DELIVER` rather than success.
- **DIA-FR-009**: Candidate selection MUST prefer an eligible primary, then an eligible secondary,
  exclude a peer already used by the current attempt, and record every inclusion and exclusion reason.
- **DIA-FR-010**: Peer eligibility states `CLOSED`, `OPEN`, `SUSPECT`, and `UNAVAILABLE` MUST be
  documented as simulator states, not the RFC 6733 section 5.6 transport peer state machine. State
  changes MUST be atomic; invalid transitions MUST preserve the last valid state.
- **DIA-FR-011**: Each active hop-by-hop identifier MUST be unique. Each answer MUST match the expected
  active attempt by hop-by-hop identifier and all modeled request identity fields; unknown answers
  MUST be discarded and duplicate or late answers MUST NOT alter a terminal result.
- **DIA-FR-012**: Eligible retransmission MUST preserve Origin-Host and end-to-end identity, allocate
  a unique hop-by-hop identifier for the new next hop, and set the T flag. First transmissions and
  all answers MUST clear the T flag, as required by RFC 6733 sections 3 and 5.5.4.
- **DIA-FR-013**: Retry MUST be limited to configured transient delivery failures, a positive maximum
  of two total attempts, fixed 100 ms simulated backoff, and an eligible alternate peer. The complete
  retry allow-list MUST be timeout, disconnect-after-send, `DIAMETER_UNABLE_TO_DELIVER`, and
  `DIAMETER_TOO_BUSY`. RFC 8506 `EVENT_REQUEST` failover MUST follow section 6.5 and MUST NOT create
  credit-control session state.
- **DIA-FR-014**: Validation failure, malformed answer, permanent or application failure, known
  fixed-host delivery failure, route failure, and retry exhaustion MUST NOT be retried, routed by
  realm fallback, or converted to success.
- **DIA-FR-015**: Every admitted operation MUST produce exactly one terminal result. Duplicate
  operation IDs MUST return `IN_PROGRESS`, the retained terminal result, or `RETENTION_EXPIRED`
  without executing duplicate work.
- **DIA-FR-016**: Queue, in-flight transaction, timer, completed-result, and event-record counts MUST
  have positive configured limits and MUST never exceed them. Built-in scenarios MUST default to 16
  queued requests, 8 in-flight transactions, 16 timers, 64 retained results, and 2048 event records.
  Work blocked only by the in-flight limit MUST wait in FIFO queue order. If admission would exceed
  any other applicable bound, it MUST return typed overload before allocating transaction IDs.
  Silent discard of admitted requests is prohibited.
- **DIA-FR-017**: Timeouts, retry backoff, peer outcomes, IDs, and event order MUST derive only from a
  simulated monotonic clock and deterministic scenario inputs; tests MUST NOT sleep.
- **DIA-FR-018**: Shutdown MUST stop admission, drain admitted work until a configured simulated
  deadline of 1 second, then complete each unfinished admission exactly once with
  `SHUTDOWN_DEADLINE`, cancel its pending timers or retries, and release its capacity.
- **DIA-FR-019**: Each versioned event MUST contain scenario ID, sequence, simulated time, component,
  operation and transaction IDs when assigned, attempt, peer when selected, severity, and safe detail.
  Human and JSON views MUST derive from the same event records.
- **DIA-FR-020**: The system MUST include deterministic scenarios for healthy host routing, realm
  fallback, pre-send failure, retry and failover, late and duplicate answers, retry exhaustion,
  malformed input, queue saturation, recovery, and shutdown.
- **DIA-FR-021**: A standards traceability table MUST map every modeled rule to an exact RFC 6733,
  RFC 8506, or 3GPP TS 29.230 clause and classify it as exact, simplified, unsupported, or deviating.
  References MUST govern only the explicitly modeled rows and MUST NOT support a claim of complete
  protocol, application, transport, security, or interoperability conformance. Any deviation MUST be
  rejected unless approved through the constitution's governance procedure.

### Key Entities *(include if feature involves data)*

- **Diameter Message**: Header plus ordered AVPs and validation status.
- **Route and Peer**: Destination criteria, application ID, ordered peer candidates, simulator state,
  and eligibility.
- **Transaction and Attempt**: Operation identity, protocol identifiers, expected answer identity,
  selected peer, deadline, retry status, and terminal result.
- **Policy**: Explicit queue, in-flight, timer, retained-result, and event capacities; FIFO queueing;
  retry allow-list; maximum attempts; backoff; retention; and shutdown deadline.
- **Scenario and Peer Outcome**: Named deterministic inputs for answers, delay, timeout, rejection,
  malformed answer, disconnect, and duplicate or late delivery.
- **Event and Result**: Bounded evidence and the single typed terminal outcome for an admission.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **DIA-SC-001**: Every built-in scenario completes entirely from supplied simulated inputs and
  scenario-controlled time, with zero dependency on an external service or actual elapsed time.
- **DIA-SC-002**: Across at least 10,000 concurrent deterministic simulated transactions, zero answers
  complete the wrong request and every admitted request has exactly one terminal result.
- **DIA-SC-003**: Queue, in-flight, timer, retained-result, and event counts exceed their configured
  limits in zero runs across all built-in load and boundary scenarios.
- **DIA-SC-004**: Repeating any built-in scenario 100 times produces identical normalized ordered
  records in all 100 runs.
- **DIA-SC-005**: One hundred percent of malformed or unsupported test inputs create no transaction,
  consume no peer send, and return the expected typed classification.
- **DIA-SC-006**: Every modeled standards rule has a reviewed traceability classification and exact
  source clause; zero unclassified rules remain before planning.
- **DIA-SC-007**: A learner can identify the selected route, peer, attempt history, and terminal reason
  for every built-in scenario solely from either the human-readable or machine-readable output.

## Assumptions

- One process and volatile state are sufficient; recovery across process loss is out of scope.
- The simulator is authoritative for peer availability and delivery outcomes.
- The modeled RFC 8506 surface is a stateless `EVENT_REQUEST` CCR/CCA routing and transaction envelope
  only. Session-based request types, credit rating, charging, quota enforcement, account state, and
  actual service delivery are unsupported.
- 3GPP TS 29.230 is used only to validate assigned codes and Vendor-Id ownership. This feature does not
  implement or advertise a 3GPP Diameter application.
- Official archive package `29230-j30.zip`, the latest TS 29.230 package published by 2026-09-06, is
  the normative 3GPP identifier baseline.
- RFC 6733 transport, security, discovery, capabilities exchange, watchdog, and complete peer state
  machine requirements are classified as unsupported because no network transport exists.
- All realms, hosts, identifiers, payloads, and credit-control fields are synthetic and contain no
  subscriber data, credentials, secrets, or monetary account information.
