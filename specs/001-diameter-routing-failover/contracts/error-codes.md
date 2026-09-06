# Stable Error and Result Codes

Codes never imply success unless the category is `SUCCESS`. The first validation error is returned.

## Input and Validation

| Code | Category | Meaning |
|------|----------|---------|
| `INVALID_ARGUMENT` | INPUT | CLI or scenario value is invalid |
| `UNSUPPORTED_VERSION` | VALIDATION | Diameter version is not 1 |
| `INVALID_MESSAGE_LENGTH` | VALIDATION | Message length or 32-bit alignment is invalid |
| `INVALID_HEADER_FLAGS` | VALIDATION | Header flags are illegal for the command direction |
| `COMMAND_UNSUPPORTED` | UNSUPPORTED | Command code is outside the modeled subset |
| `APPLICATION_UNSUPPORTED` | UNSUPPORTED | Application ID is outside the modeled subset |
| `INVALID_AVP_BITS` | VALIDATION | AVP flags conflict with its definition |
| `INVALID_AVP_LENGTH` | VALIDATION | AVP length, data size, or padding is invalid |
| `MISSING_AVP` | VALIDATION | Required AVP is absent |
| `AVP_OCCURS_TOO_MANY_TIMES` | VALIDATION | Singular AVP occurs more than once |
| `INVALID_AVP_VALUE` | VALIDATION | AVP value violates type or supported enumeration |
| `AVP_UNSUPPORTED` | UNSUPPORTED | Unknown AVP has the M bit set |
| `CC_REQUEST_TYPE_UNSUPPORTED` | UNSUPPORTED | CCR is not `EVENT_REQUEST` |
| `THREE_GPP_IDENTIFIER_UNSUPPORTED` | UNSUPPORTED | 3GPP vendor-specific identifier is not modeled |

## Configuration and Capacity

| Code | Category | Meaning |
|------|----------|---------|
| `DUPLICATE_ROUTE` | CONFIGURATION | Host or realm route key appears more than once |
| `MULTIPLE_DEFAULT_ROUTES` | CONFIGURATION | More than one default route is declared |
| `DUPLICATE_ROUTE_PEER` | CONFIGURATION | Peer appears twice in one route |
| `INVALID_PEER_TRANSITION` | CONFIGURATION | Simulator peer transition is not allowed |
| `QUEUE_FULL` | OVERLOAD | Input queue limit is reached before ID allocation |
| `TIMER_LIMIT` | OVERLOAD | Required timer capacity is unavailable before admission |
| `EVENT_CAPACITY` | OVERLOAD | Event sink cannot accept required output safely |
| `ID_SPACE_EXHAUSTED` | OVERLOAD | Injected hop-by-hop ID domain has no inactive value |
| `SHUTTING_DOWN` | SHUTDOWN | Admission is closed |

## Routing, Correlation, and Completion

| Code | Category | Retryable | Meaning |
|------|----------|-----------|---------|
| `SUCCESS` | SUCCESS | No | Expected answer completed the transaction |
| `DIAMETER_UNABLE_TO_DELIVER` | ROUTING | Conditional | No eligible route/peer; retry only after accepted send and only for non-fixed destination |
| `DIAMETER_TOO_BUSY` | PEER | Conditional | Peer reports transient overload after accepted send |
| `TIMEOUT` | PEER | Conditional | Expected answer deadline elapsed |
| `DISCONNECT_AFTER_SEND` | PEER | Conditional | Peer accepted send before disconnecting |
| `MALFORMED_ANSWER` | CORRELATION | No | Answer validation failed |
| `MISMATCHED_ANSWER` | CORRELATION | No | Answer identity differs from expected attempt |
| `UNKNOWN_ANSWER` | CORRELATION | No | Hop-by-hop ID has no active attempt; answer is discarded |
| `LATE_ANSWER` | CORRELATION | No | Answer targets a superseded or terminal attempt |
| `DUPLICATE_ANSWER` | CORRELATION | No | Answer repeats a completed attempt |
| `RETRY_EXHAUSTED` | RETRY | No | Retryable failure consumed the two-attempt budget |
| `IN_PROGRESS` | DEDUPLICATION | No | Duplicate operation is currently admitted |
| `RETENTION_EXPIRED` | DEDUPLICATION | No | Prior result was evicted; work is not repeated |
| `SHUTDOWN_DEADLINE` | SHUTDOWN | No | Work remained after the 1-second simulated drain |

Only `TIMEOUT`, `DISCONNECT_AFTER_SEND`, `DIAMETER_UNABLE_TO_DELIVER`, and `DIAMETER_TOO_BUSY` enter
retry evaluation. Evaluation still requires a non-fixed destination, unused eligible alternate, and
remaining attempt budget.
