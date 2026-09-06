# CLI Contract

Executable name: `diameter-routing-demo`

Every command writes the exact disclaimer below as its first human line or as the JSON document's
`disclaimer` field:

> Simulation-only educational demo; not a conformant or production Diameter implementation.

## Commands

### List Scenarios

```text
diameter-routing-demo list-scenarios [--format human|json]
```

Returns these stable built-in names in declaration order:

1. `healthy-host`
2. `realm-fallback`
3. `pre-send-failure`
4. `retry-failover`
5. `late-duplicate-answer`
6. `retry-exhaustion`
7. `malformed-input`
8. `queue-full`
9. `recovery`
10. `graceful-shutdown`

### Run Scenario

```text
diameter-routing-demo run SCENARIO_NAME [--seed UINT64] [--format human|json]
```

- `SCENARIO_NAME` must be one listed above.
- `seed` defaults to 0 and controls every generated identifier.
- `format` defaults to `human`.
- Options may appear once; unknown, missing, or duplicate options are usage errors.
- The command accepts no network address, external scenario file, subscriber identity, or secret.

## Exit Status

| Status | Meaning |
|--------|---------|
| 0 | Command and scenario execution completed, including scenarios whose expected transaction result is failure |
| 2 | Invalid command, option, value, or scenario name |
| 3 | Invalid built-in scenario configuration |
| 4 | Internal invariant or output failure |

Transaction result codes are reported in output and do not become process exit statuses.

## Human Output

After the disclaimer, output contains a run header, ordered event lines, and terminal result lines.
Each event line includes sequence, simulated milliseconds, component, event type, operation/attempt
when assigned, peer when selected, severity, code, and safe detail.

## JSON Output

Output is one JSON document conforming to [event-output.schema.json](event-output.schema.json).
Fields and array order are stable. Strings use JSON escaping; integers use base-10 canonical form;
no insignificant whitespace is emitted in normalized output.

## Determinism

The same executable, scenario, seed, and format produce byte-identical normalized output. Equal-time
events follow scenario declaration order and scheduler insertion sequence.
