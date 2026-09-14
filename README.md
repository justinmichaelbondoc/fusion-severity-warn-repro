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

## Finding 2: committed (non-gitignored) `dbt_packages/` causes a path-doubling error

This is a **separate, fully reproducible** bug, found while investigating
whether the customer's project structure — where `dbt_packages/` is *not*
listed in `.gitignore`, meaning package contents get committed directly
into the consuming project's own git repo — could be a contributing
factor. This finding is based purely on directory structure; no telemetry,
warehouse data, or other customer-specific content was used to build it.

### Setup

`dbt_packages/segment/` present as a plain, already-materialized directory
(not a symlink from a local-path dependency, not fetched via `dbt deps`) —
i.e. exactly what you'd get if a package's files were committed straight
into the project's git history instead of being gitignored:

```
.
├── dbt_project.yml
├── profiles.yml
└── dbt_packages/
    └── segment/
        ├── dbt_project.yml
        ├── .gitignore
        ├── tests/
        └── seeds/
            ├── referrer_mapping.csv
            └── seeds.yml
```

### Result: reproduces 100% of the time

```
    Failed [  1.01s] seed  main.referrer_mapping (table)
   Skipped [-------] test  not_null_referrer_mapping_host (dbt_packages/segment/seeds/seeds.yml:11:13)
   Skipped [-------] test  unique_referrer_mapping_host (dbt_packages/segment/seeds/seeds.yml:8:13)

=================== Errors and Warnings ====================
[error] [DbDriverFailed (dbt1308)]: Database Error in seed referrer_mapping (target/run/segment/dbt_packages/segment/seeds/referrer_mapping.sql)
  IO Error: No files found that match the pattern
  ".../dbt_packages/segment/dbt_packages/segment/seeds/referrer_mapping.csv"
```

Fusion doubles the package path (`dbt_packages/segment/` prefixed onto
`dbt_packages/segment/seeds/referrer_mapping.csv`, which is already a
full path from the project root) when the package exists as a plain
on-disk directory rather than one resolved through its normal install
path. Confirmed this isn't name-specific — renaming the package folder
reproduces the identical doubled-path failure.

### How this relates to ticket #136436

**Not a direct match.** The customer's own telemetry showed the `segment`
package's Hub-deprecation warning, meaning it *was* installed normally via
`dbt deps` in their real run, and the seed loaded successfully with the
test actually executing (1,803 real failing rows) — not an IO/path error.
So this exact failure mode doesn't explain their reported symptom by
itself.

It's flagged here as a **plausible contributing factor**, not a confirmed
cause: if a stale, git-committed copy of `dbt_packages/` sits alongside a
freshly-`deps`-installed one, a collision between the two could produce
corrupted or inconsistent state. That hypothesis has not been verified.

## Next steps

- Try a larger multi-threaded build (dozens+ of nodes, `--threads > 1`) to
  see if the invocation-level miscount from Finding 1 appears at scale.
- Check `dbt-labs/dbt-core` (where Fusion `[v2 Bug]`/`engine:v2` issues are
  filed) for prior art — none found as of this writing across ~350 recently
  paginated `engine:v2` issues (GitHub's search API was returning 502s at
  the time, so this was not an exhaustive check).
