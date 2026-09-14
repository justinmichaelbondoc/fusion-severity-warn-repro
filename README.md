# Fusion `severity: warn` invocation error-tally repro

Related support ticket: https://dbtcloud.zendesk.com/agent/tickets/136436

## Reported behavior

A generic `unique` test configured with `severity: warn` fails on a real
dbt Fusion run, and the individual node correctly resolves to a "warn"
outcome — but the **overall invocation still exits as an error**, causing
the job to fail as if the test had no severity override at all.

Evidence pulled from the customer's own run telemetry (OTel parquet export,
`dbt-fusion 2.0.0-preview.218`, `dbt build`, 682 total nodes):

- The failing test's own span resolves to `NODE_OUTCOME_SUCCESS` (severity
  correctly downgrades the *node* outcome), with
  `"node_outcome_detail":{"NodeTestDetail":{"test_outcome":"TEST_OUTCOME_FAILED","failing_rows":1803}}`.
- But the invocation-level summary reports
  `"metrics":{"total_errors":"1","total_warnings":"2","status_counts":{"skipped":"31","success":"650","error":"1"}}`,
  and both the `Phase: RUN` span and the top-level `dbt invocation` span
  close with `status_code: 2` (`"1 nodes errored"` / `"Executed with errors"`).
- The debug log for the same invocation ID ends with:
  ```
  Invocation ... build with 2 warnings 1 error for target default [3143.62s]
       Summary 682 total | 650 success | 1 error | 31 skipped
  dbt command failed
  ```

So the only `TEST_OUTCOME_FAILED` node in the entire run is a `severity: warn`
test, yet it's the one node counted toward `total_errors`/`status_counts.error`,
which flips the job's exit code to failure.

## This repro

This is a **minimal** reproduction attempt: one seed, one `unique` test on a
column with a duplicated value, `severity: warn` configured under `config:`
(required by Fusion — see below).

```bash
dbt build --profiles-dir . --project-dir .
```

### Result on this minimal case: **not reproduced**

On `dbt-fusion 2.0.0-preview.218`, this minimal project behaves *correctly*:

```
 Succeeded [  0.02s] seed  main.referrer_mapping (table)
    Passed [  0.01s] test  not_null_referrer_mapping_host
    Warned [  0.01s] test  unique_referrer_mapping_host (seeds/referrer_mapping.yml:8:13)

==================== Execution Summary =====================
Finished 'build' with 1 warning for target 'dev' [1.1s]
Processed: 2 tests | 1 seed
Summary: 3 total | 2 success | 1 warn
```

Exit code `0`, the test is correctly counted as a warning, not an error.

**This means the bug does not reproduce on a small (3-node), single-threaded
build.** The customer's failing run was 682 nodes with `num_threads: 2`. The
leading (unverified) hypothesis is that this is a concurrency/aggregation
bug in how per-node telemetry is rolled up into the invocation-level
`total_errors`/`status_counts` summary — something that would only surface
at scale or under concurrent test execution, not in a trivial single-test
build. This has **not been confirmed** — it's a hypothesis based on what
did and didn't reproduce here, not a verified root cause.

### Also attempted: test defined inside a vendored (local) package

To mirror the customer's exact layout (`dbt_packages/segment/seeds/seeds.yml`,
i.e. the test lives inside an installed package, not the root project), a
local-path package was set up the same way. This hit an unrelated Fusion
path-resolution issue with locally-symlinked packages (seed CSV path gets
doubled to `dbt_packages/<pkg>/dbt_packages/<pkg>/seeds/...`) that blocked
seed loading entirely before the severity behavior could even be exercised.
That's a separate, currently-unresolved wrinkle in this repro — not the bug
being tracked here — and was dropped rather than worked around, to avoid
conflating two different issues in one report.

## Config note

Fusion rejects `severity` at the top level of a generic test config (Core's
older syntax) with `DbtYamlValidationError (dbt1159)` and requires it nested
under `config:`:

```yaml
data_tests:
  - unique:
      config:
        severity: warn
```

## Next steps

- Try a larger multi-threaded build (dozens+ of nodes, `--threads > 1`) to
  see if the invocation-level miscount appears at scale.
- Check `dbt-labs/dbt-core` (where Fusion `[v2 Bug]`/`engine:v2` issues are
  filed) for prior art — none found as of this writing across ~350 recently
  paginated `engine:v2` issues (GitHub's search API was returning 502s at
  the time, so this was not an exhaustive check).
