# Performance Progress: 2026-03-14

Branch:
- `exp/master-documented-query-baseline`

Scope:
- Clean-branch performance and reliability work after the documented-only reset.
- Production target remains `plugin-url` unless new evidence disproves the transport decision.

## Current Status

### `get_task_counts`

Status:
- Optimized
- Reliability acceptable in the current clean-branch validation runs

Key validation artifacts:
- `docs/benchmarks/suite-post-listtasks-hardening-20260313-221745/get_task_counts/summary.md`
- `docs/benchmarks/task-counts-1h-post-fast-path-20260314-114300/summary.md`

Observed change in the 1-hour comparison:
- plugin p50: `2609ms -> 2108ms`
- plugin p95: `11078ms -> 9524ms`
- plugin errors/timeouts: `0 -> 0`
- parity mismatches: `0`

Conclusion:
- The documented single-pass + narrow fast-path work improved `get_task_counts` enough to treat it as complete for now.
- Remaining latency is mostly OmniFocus/runtime pressure rather than obviously inefficient JS filtering.

### `list_tasks`

Status:
- Reliability hardened
- Shared-path cleanup validated
- No-total-count streaming fast path implemented and directly validated

Reliability validation artifacts:
- Pre-hardening transport A/B baseline:
  - `docs/benchmarks/transport-ab-safe-20260312-210547/plugin-url/list_tasks/summary.md`
- Post-hardening 1-hour run:
  - `docs/benchmarks/list-tasks-1h-post-hardening-20260313-145309/summary.md`

Reliability conclusion:
- plugin timeout rate dropped from `7.41%` (`8/108`) in the old transport A/B baseline to `0.00%` in the post-hardening 1-hour run.
- This is the main reason the product continues to stay on `plugin-url`.

Shared-path performance validation artifact:
- `docs/benchmarks/list-tasks-1h-post-stream-fast-path-20260314-2313/summary.md`

1-hour comparison against the post-hardening baseline:
- `default`
  - plugin p50: `8454ms -> 7204ms`
  - plugin p95: `9355ms -> 8782ms`
- `inbox_only`
  - plugin p50: `644ms -> 550ms`
  - plugin p95: `847ms -> 746ms`
- `available_only`
  - plugin p50: `8443ms -> 7345ms`
  - plugin p95: `9568ms -> 10551ms`
- `completed_after_anchor`
  - plugin p50: `7139ms -> 6090ms`
  - plugin p95: `7729ms -> 8008ms`
- `flagged_only`
  - plugin p50: `6999ms -> 6005ms`
  - plugin p95: `7582ms -> 7685ms`
- errors/timeouts: `0`
- parity mismatches: `0`

Interpretation:
- The shared-path cleanup improved median latency across every benchmarked scenario.
- Tail latency improved for `default` and `inbox_only`, but was mixed for the other benchmarked scenarios.
- This is acceptable because reliability stayed clean and the benchmark harness does not exercise the new early-stop branch.

Important benchmark limitation:
- `BenchmarkListTasksCommand` currently sets `includeTotalCount=true` for every scenario.
- That means the stock benchmark validates the shared path but not the new no-total-count early-stop path.

Direct no-total-count validation:
- Timed direct CLI calls on the same branch and runtime state:
  - `.build/debug/focusrelay list-tasks --available-only true --fields id,name --limit 50`
    - `real 5.97`
  - `.build/debug/focusrelay list-tasks --available-only true --include-total-count --fields id,name --limit 50`
    - `real 8.48`

Conclusion:
- The early-stop path is real and materially faster when callers do not request totals.
- Current benchmark coverage should be extended later with explicit no-total-count scenarios before making stronger claims about that path.

### `get_project_counts`

Status:
- Reliability acceptable
- Semantics stable on the clean branch

Key artifact:
- `docs/benchmarks/project-counts-soak-diag-20260309-132541/summary.md`

Result:
- plugin: `1236/1236` success, `0` errors, `0` timeouts
- jxa: `1236/1236` success, `0` errors, `0` timeouts
- parity mismatches: `0`

Conclusion:
- `get_project_counts` is stable enough for now.
- It remains slower than desired, but it is no longer the highest-priority blocker.

## Architecture Position

Current position remains unchanged:
- Keep `plugin-url` as the production default.
- Do not switch to `plugin-jxa-dispatch`.
- Do not replace the plugin architecture with pure JXA.

Primary rationale:
- query semantics are stable enough to compare
- transport simplification did not beat `plugin-url`
- recent work improved `plugin-url` reliability enough that transport change is no longer the best lever

Reference:
- `docs/transport-decision-2026-03-13.md`

## Next Practical Work

1. If `list_tasks` remains the main performance focus, extend benchmark coverage with explicit no-total-count scenarios.
2. Keep optimization work on `plugin-url`, not transport switching.
3. Revisit lower-priority tools only after the high-value `list_tasks` path is fully benchmarked for both total-count and no-total-count workloads.
