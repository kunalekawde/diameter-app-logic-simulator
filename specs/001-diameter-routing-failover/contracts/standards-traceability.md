# Standards Traceability Contract

This table governs only the modeled simulation subset. It does not establish Diameter transport,
security, application, interoperability, or 3GPP application conformance.

| Requirement | Source | Classification | Planned Evidence |
|-------------|--------|----------------|------------------|
| DIA-FR-001 disclaimer | Project governance | Exact | CLI contract tests for both formats |
| DIA-FR-002 header fields and flags | RFC 6733 section 3 | Exact for decoded fields | Header validator unit tests and fuzz target |
| DIA-FR-003 AVP header, length, flags, padding | RFC 6733 sections 4 and 4.1 | Exact for modeled AVPs | AVP validator unit tests and fuzz target |
| DIA-FR-004 base AVP subset | RFC 6733 sections 4.5, 6.3-6.6, 7.1, 7.5, 8.8 | Simplified | Cardinality and unsupported-AVP tests |
| DIA-FR-005 EVENT CCR/CCA envelope | RFC 8506 sections 3.1, 3.2, 6, 8.2, 8.3, 10.1 | Exact for selected envelope fields; other procedures unsupported | CCR/CCA contract tests |
| DIA-FR-006 3GPP identifiers | 3GPP TS 29.230 V19.3.0 clause 4 and Tables 5.1, 6.1, 7.1 | Validation-only; all 3GPP applications and AVPs unsupported | Identifier rejection tests; archive checksum verification |
| DIA-FR-007 unsupported behavior | RFC 6733 sections 4.1 and 7.1.3-7.1.5 | Simplified typed application boundary | Error mapping tests |
| DIA-FR-008 host/realm/default routing | RFC 6733 sections 2.7, 6.1, 6.1.5, 6.1.6; section 5.5.4 for fixed destination failure | Exact decision order; static simulation only | Route precedence and fixed-host tests |
| DIA-FR-009 primary/secondary selection | RFC 6733 sections 5.1 and 5.5.4 | Simplified eligibility model | Candidate ordering tests |
| DIA-FR-010 simulator peer states | RFC 6733 sections 5.5-5.6 | Deviating by design: no transport peer state machine | Simulator transition tests and explicit output label |
| DIA-FR-011 active answer correlation | RFC 6733 sections 3 and 6.2.1 | Exact identity checks plus stricter modeled fields | Unknown, mismatch, duplicate, and late-answer tests |
| DIA-FR-012 retransmission identity and T flag | RFC 6733 sections 3 and 5.5.4 | Exact for modeled retry | Retry identity contract tests |
| DIA-FR-013 one-time event failover | RFC 8506 section 6.5 | Simplified to two attempts and fixed backoff | Allow-list and failover tests |
| DIA-FR-014 non-retryable failures | RFC 6733 section 7; RFC 8506 sections 6.5 and 9 | Exact selected classifications; conservative otherwise | Negative retry matrix |
| DIA-FR-015 duplicate operation handling | RFC 6733 section 3 and Appendix C; RFC 8506 section 6.5 | Simplified bounded retention | In-progress, retained, and expired tests |
| DIA-FR-016 bounded capacity | Project governance | Exact | Boundary and generated property tests |
| DIA-FR-017 simulated time | Project governance | Exact | No-sleep checks and deterministic replay tests |
| DIA-FR-018 shutdown | Project governance | Exact | Drain and deadline integration tests |
| DIA-FR-019 event schema | Project governance | Exact | JSON Schema and human/JSON parity contract tests |
| DIA-FR-020 built-in scenarios | Project specification | Exact | Scenario registry integration test |
| DIA-FR-021 traceability | Project governance | Exact | CI check that every DIA-FR row and evidence target exists |

## Deliberately Unsupported

- TCP, SCTP, TLS, DNS, capabilities exchange, watchdog, and the RFC 6733 transport peer state machine.
- RFC 8506 session request types, rating, quota, balance, charging, subscriber, and service behavior.
- Every 3GPP Diameter application, command, AVP, and procedure, including Gx, Gy, and S6a claims.
- Conformance or interoperability claims without external testing.

## Baseline Verification

- RFC 6733: IETF standards-track publication.
- RFC 8506: IETF standards-track publication replacing RFC 4006.
- 3GPP TS 29.230: official package `29230-j30.zip`, V19.3.0, Release 19, listed 2026-01-06.
- Implementation must record the package SHA-256 when the official binary archive is reachable.