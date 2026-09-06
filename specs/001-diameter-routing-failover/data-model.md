# Data Model: Resilient Diameter Routing and Peer-Failure Demo

## Value Types

### RunIdentity

- `scenario_id`: Stable built-in scenario name.
- `seed`: Unsigned 64-bit deterministic input.
- `run_id`: Synthetic value derived from scenario and seed.

The same scenario and seed produce the same run identity.

### DiameterHeader

- `version`: Unsigned 8-bit value; supported value is 1.
- `message_length`: Unsigned 24-bit encoded length, including padded AVPs.
- `flags`: R, P, E, T, and four reserved bits.
- `command_code`: Unsigned 24-bit command; supported application command is 272.
- `application_id`: Unsigned 32-bit value; supported application value is 4.
- `hop_by_hop_id`: Unsigned 32-bit active-attempt identity.
- `end_to_end_id`: Unsigned 32-bit operation identity preserved across retry.

Validation follows RFC 6733 section 3 before any route or transaction allocation.

### Avp

- `code`: Unsigned 32-bit identifier.
- `flags`: V, M, P, and five reserved bits.
- `length`: Unsigned 24-bit header-plus-data length, excluding padding.
- `vendor_id`: Optional unsigned 32-bit value, present exactly when V is set.
- `value`: Typed immutable value or opaque synthetic bytes for the optional demo payload.
- `padding_length`: Derived zero-byte count needed for 32-bit alignment.

Supported base AVPs are Session-Id, Origin-Host, Origin-Realm, Destination-Host,
Destination-Realm, Result-Code, and Failed-AVP. The stateless RFC 8506 envelope also validates
Auth-Application-Id, Service-Context-Id, CC-Request-Type, CC-Request-Number, and Requested-Action.
Unknown mandatory or 3GPP vendor-specific AVPs are rejected.

### ValidatedMessage

- `header`: Validated immutable DiameterHeader.
- `avps`: Ordered immutable AVPs.
- `session_id`: Exactly one synthetic session identity.
- `origin_host`, `origin_realm`: Exactly one valid DiameterIdentity each.
- `destination_host`: Optional DiameterIdentity for requests only.
- `destination_realm`: Exactly one DiameterIdentity for supported requests.
- `cc_request_type`: `EVENT_REQUEST` only.
- `cc_request_number`: Zero only.
- `requested_action`: One valid RFC 8506 one-time action.

Construction is possible only after all header, AVP, cardinality, and envelope checks pass.

## Configuration

### CapacityPolicy

- `queue_limit`: Positive count; default 16.
- `in_flight_limit`: Positive count; default 8.
- `timer_limit`: Positive count; default 16.
- `retained_result_limit`: Positive count; default 64.
- `event_buffer_limit`: Positive count; default 2048.

Zero values are configuration errors. Queue order is FIFO. No insertion may make a count exceed its
limit.

### RetryPolicy

- `maximum_attempts`: Two total attempts.
- `backoff_ms`: Fixed 100 simulated milliseconds.
- `retryable_causes`: Timeout, disconnect-after-send, `DIAMETER_UNABLE_TO_DELIVER`, and
  `DIAMETER_TOO_BUSY` only.

Retry additionally requires a non-fixed destination and an unused eligible alternate peer.

### ShutdownPolicy

- `drain_ms`: 1000 simulated milliseconds.
- `deadline_result`: `SHUTDOWN_DEADLINE`.

### Route

- `kind`: Host, realm, or default.
- `destination`: Host or realm identity; absent for default.
- `application_id`: Required for host and realm keys; application ID 4 for this feature.
- `candidates`: One primary and optional secondary PeerId in declaration order.

Host and realm route keys must be unique, at most one default is allowed, and candidate PeerIds must
be unique within a route. Configuration is immutable after validation.

### Peer

- `peer_id`: Unique synthetic identity.
- `role`: Primary or secondary within a route.
- `state`: CLOSED, OPEN, SUSPECT, or UNAVAILABLE.
- `outcomes`: Ordered deterministic simulated outcomes keyed by operation and attempt.

Only OPEN is eligible for new sends. These are simulator eligibility states, not RFC 6733 transport
peer states.

#### Peer State Transitions

| From | Allowed To | Effect |
|------|------------|--------|
| CLOSED | OPEN | Peer becomes eligible |
| OPEN | SUSPECT, UNAVAILABLE, CLOSED | Peer stops accepting new sends |
| SUSPECT | OPEN, UNAVAILABLE, CLOSED | Recovery or failure is recorded |
| UNAVAILABLE | OPEN, CLOSED | Recovery or reset is recorded |

All other transitions, including self-transitions, return `INVALID_PEER_TRANSITION` and preserve the
current state.

## Transaction Model

### OperationIdentity

- `run_id`: Owning run.
- `sequence`: Monotonically generated per-run admission sequence.
- `end_to_end_id`: Stable unsigned 32-bit value.
- `origin_host`, `session_id`, `cc_request_type`, `cc_request_number`: Stable correlation fields.

The generated sequence supports deterministic deduplication and the retention watermark.

### Transaction

- `operation`: Stable OperationIdentity.
- `message`: ValidatedMessage.
- `state`: QUEUED, ACTIVE, BACKOFF, or TERMINAL.
- `attempts`: Ordered list with at most two entries.
- `terminal_result`: Absent until the sole terminal transition.
- `admitted_at_ms`, `terminal_at_ms`: Simulated times.

#### Transaction State Transitions

| From | Trigger | To | Required Action |
|------|---------|----|-----------------|
| QUEUED | In-flight capacity available | ACTIVE | Allocate first hop-by-hop ID and attempt |
| QUEUED | Shutdown deadline | TERMINAL | Store `SHUTDOWN_DEADLINE` once |
| ACTIVE | Valid expected answer | TERMINAL | Cancel timer and store answer result once |
| ACTIVE | Retryable cause with eligible alternate | BACKOFF | Close attempt and schedule 100 ms retry |
| ACTIVE | Non-retryable cause or exhausted budget | TERMINAL | Store typed failure once |
| BACKOFF | Backoff expires | ACTIVE | Allocate new hop-by-hop ID, set T, send alternate |
| BACKOFF | Shutdown deadline | TERMINAL | Cancel retry and store `SHUTDOWN_DEADLINE` once |

TERMINAL has no outgoing transition. Any later answer is classified without changing the result.

### Attempt

- `number`: 1 or 2.
- `peer_id`: Selected peer.
- `hop_by_hop_id`: Unique while active.
- `t_flag`: False for attempt 1 and true for attempt 2.
- `sent_at_ms`, `deadline_ms`: Simulated times.
- `state`: PLANNED, WAITING, SUCCEEDED, FAILED, or SUPERSEDED.
- `failure`: Optional retryable or terminal code.

An answer completes an attempt only when hop-by-hop ID, command code, application ID, end-to-end ID,
Session-Id, CC-Request-Type, and CC-Request-Number all match the expected active attempt.

## Simulation and Scheduling

### ScheduledEvent

- `due_time_ms`: Simulated monotonic due time.
- `insertion_sequence`: Unique increasing tie-breaker.
- `kind`: Peer delivery, timeout, retry, state change, admission, or shutdown deadline.
- `owner_operation`: Optional OperationIdentity.
- `cancelled`: Logical cancellation flag.

Ordering is ascending `(due_time_ms, insertion_sequence)`. Cancelled entries release timer capacity
when removed and cannot mutate state.

### SimulatedPeerOutcome

- `kind`: Answer, delay, timeout, disconnect-before-send, disconnect-after-send, malformed answer,
  rejection, duplicate answer, or late answer.
- `delay_ms`: Non-negative simulated delay.
- `answer`: Optional message required for answer-bearing outcomes.
- `declaration_sequence`: Same-time ordering key.

Disconnect-before-send selects another eligible candidate without consuming a retry attempt.

## Results, Deduplication, and Events

### TerminalResult

- `operation`: OperationIdentity.
- `code`: Stable success or failure code.
- `completed_at_ms`: Simulated time.
- `attempt_count`: One or two.
- `peer_id`: Optional completing peer.
- `detail`: Synthetic safe explanation.

### ResultStore

- `active`: Lookup of queued, active, and backoff operations.
- `retained`: Operation-to-result lookup bounded by retained_result_limit.
- `completion_order`: FIFO of retained operation identities.
- `eviction_watermark`: Highest generated sequence evicted in the run.

Duplicate lookup order is active (`IN_PROGRESS`), retained (same TerminalResult), then watermark
(`RETENTION_EXPIRED`). Retained entries override the watermark to support out-of-order completion.

### Event

- `schema_version`: 1.
- `run_id`, `scenario_id`: Synthetic run identity.
- `sequence`: Increasing event order.
- `simulated_time_ms`: Monotonic time.
- `component`, `event_type`, `severity`: Stable classifications.
- `operation_id`, `transaction_id`, `attempt`, `peer`: Nullable until assigned.
- `code`, `detail`: Stable safe values.

One Event value is passed to either formatter. The event sink drains incrementally so its in-memory
buffer remains bounded; output failure is explicit and never converted to transaction success.

## Relationships and Invariants

- A Scenario owns one policy set, route table, peer set, scheduler, and ordered admissions.
- A validated Route references existing Peers; no transaction owns a peer.
- One admitted OperationIdentity owns exactly one Transaction and one TerminalResult.
- One Transaction owns one or two Attempts; only one Attempt is expected at a time.
- Each active hop-by-hop ID maps to exactly one expected Attempt.
- Completing or shutting down a Transaction removes all active ID and timer mappings before capacity
  is released.
- Event and result output contain synthetic values only and always include the required disclaimer at
  the run-document level.