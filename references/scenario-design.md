# Scenario design

Gate 4. How to turn a changed endpoint into a set of calls that prove something.

The unit is the **variant**: one endpoint, one payload, one expected outcome, one receipt.

## The variant matrix

Three are the floor. Two more are conditional.

| Variant | Always? | What it proves | How to build it |
|---------|---------|----------------|-----------------|
| `happy` | yes | The documented contract holds. | A real, valid payload. Every required field. Values that exist in the data. |
| `boundary` | yes | The edges behave. | Empty string, empty array, `null` in an optional field, the maximum length, a unicode name, zero, a date at a period edge. One edge per variant when they interact. |
| `negative` | yes | Bad input is refused correctly. | A missing required field, a wrong type, an id that does not exist, a malformed body. Expect a 4xx. |
| `regression` | on a bug fix | The reported bug is gone. | The exact payload from the bug report. |
| `authz` | on a permission change | The wrong caller is refused. | The same `happy` payload with a token for the wrong role or the wrong tenant. Expect 401 or 403. |

`negative` earns its place because of one specific failure: a 500 where a 400 belongs. The caller cannot tell "you sent something wrong" from "we are broken". A 500 on bad input is a `FAIL`, even when the data is correctly rejected.

For a `boundary` variant, decide the expected status before you fire it. A boundary case that returns 200 when you expected 400 is a finding. A boundary case with no stated expectation proves nothing, because any result looks acceptable after the fact.

## Payload sourcing

Take the first rung that works. Say in the report which rung you reached.

1. **User-supplied.** The user gave you a case, a request body, or a bug report. Best rung — it is the shape production actually sends.
2. **Saved fixture.** A previous run of this skill saved one under `.verify/*/fixtures/`. Reuse it, then confirm the identifiers inside still resolve.
3. **A real row.** Query the database or a read endpoint for a row that satisfies the endpoint's preconditions. Copy the real identifiers. This is the rung that catches six-year-old legacy values a synthesized payload never has.
4. **Synthesized.** You built it from the request type's fields.

Rung 4 proves the least. A synthesized payload has whatever shape you imagined, so it tests your imagination and not the contract. Label it in the report. Never let "verified" rest on rung 4 alone for a `happy` variant.

### Redaction

Fixtures go on disk and into reports. Before writing:

- Replace real names, emails, phone numbers, and addresses with obvious placeholders.
- Keep identifiers that the query needs — a UUID, a class code, an order id. Truncating those breaks the test.
- Never write a token, password, cookie, or API key into a fixture or a receipt. `scripts/lib.sh` redacts on the way out, but do not rely on it as the only guard.

## Assertions

A receipt with a response body and no expectation is a log, not a test. Each variant states its expectation up front.

Assert three things, in this order:

1. **Status.** Exact, not a range. `200`, not `2xx`.
2. **Shape.** The envelope and the fields the caller depends on exist. `.data.items` is an array. `.data.total` is present.
3. **Value.** The one number or string the change was about. This is the assertion that actually tests the fix. The other two would pass on the broken build too.

Skipping 3 is the most common way a scenario suite goes green on a change that did nothing.

## Scenario file format

`scripts/run-scenarios.sh` reads one JSON file per endpoint from `.verify/<run>/scenarios/`:

```json
{
  "endpoint": "GET /api/v1/reports/summary",
  "auth": "required",
  "variants": [
    {
      "name": "happy",
      "method": "GET",
      "path": "/api/v1/reports/summary",
      "query": { "accountId": 15978, "periodId": 145910 },
      "expect": {
        "status": 200,
        "jq": [".data != null", ".data.items | length > 0", ".data.total == 12"]
      },
      "source": "db-row",
      "note": "accountId from an account with 12 active items"
    },
    {
      "name": "negative",
      "method": "GET",
      "path": "/api/v1/reports/summary",
      "query": { "accountId": -1 },
      "expect": { "status": 400 },
      "source": "synthesized"
    },
    {
      "name": "authz-wrong-role",
      "method": "GET",
      "path": "/api/v1/reports/summary",
      "query": { "accountId": 15978 },
      "env": { "APP_USER_ROLE": "VIEWER" },
      "expect": { "status": 403 },
      "source": "db-row"
    }
  ]
}
```

`expect.jq` entries are jq expressions evaluated against the response body. Every one must return `true`.

`env` sets environment variables for that variant only. This is how an `authz` variant calls as a different identity: the adapter builds `VERIFY_TOKEN_SUBJECT` from the same variables, so the token cache key changes with them and the wrong cached token is never reused.

Keep the files. The next run on the same code re-runs the suite in seconds.

## Side effects

A write endpoint needs a second observation. The response says what the service believes. The store says what happened.

For each writing variant, record:

- The state **before** the call. The row, the count, the file, the queue depth.
- The response.
- The state **after** the call.

A 200 with an unchanged row is a `FAIL` that a status-only check reports as a pass.

For a fix that corrected a wrong value, the before and after pair *is* the proof. Put it in the report, not only the receipt.

## Order and cleanup

Run read variants first. They cost nothing and they confirm the token, the routing, and the health of the service before any write happens.

Run `negative` and `authz` before `happy` when the endpoint writes. A rejected call leaves no state to undo.

Record every row you create, in the run directory. Say in the report what test data now exists. Leaving fixtures behind is usually fine on a local or dev environment; leaving them behind silently is not.
