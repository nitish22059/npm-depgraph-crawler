# Milestone 5 Implementation Plan

This is the execution plan for Milestone 5. The companion learning document is
`docs/milestone-5-full-registry-readiness.md`.

## Objective

Make the crawler safe and understandable to run for longer bounded crawls.

Milestone 5 is complete when:

- bounded crawl controls are available
- `status` reports useful progress
- failure summaries are inspectable
- runbook explains safe operation
- database growth checks are documented
- tests cover status and stop conditions

## Non-Goals

Do not:

- run an uncontrolled full npm crawl
- optimize indexes without a query reason
- move analysis work into the crawler loop
- introduce distributed workers yet
- hide failures behind vague log messages

## Phase 1: Cleanly Define Operational Commands

Goal:

```text
Make the CLI usable for safe crawl operation.
```

Commands:

```powershell
depcrawler seed
depcrawler crawl --max-packages 25 --requests-per-second 2
depcrawler crawl --max-seconds 300 --requests-per-second 2
depcrawler status
```

Tasks:

- Confirm `crawl` supports `--max-packages`.
- Confirm `crawl` supports `--max-seconds`.
- Confirm crawl summary includes stop reason.
- Confirm `status` command exists.
- Keep command help text beginner-friendly.

Acceptance criteria:

- `depcrawler crawl --help` shows bounded crawl options.
- `depcrawler status --help` works.
- Crawl output includes `reason=...`.

## Phase 2: Strengthen Status Output

Goal:

```text
Give the operator enough information to understand crawl health.
```

Current status should include:

- pending count
- in-progress count
- done count
- failed count
- packages stored
- versions stored
- dependency edges stored
- top failure reasons

Future useful additions:

- oldest pending item age
- oldest in-progress lease
- recent completion rate
- failed percentage
- database row counts by table

Acceptance criteria:

- Status output is deterministic enough to test.
- Missing states display as zero.
- No failures displays `failure reasons: none`.
- Failure grouping is sorted by count descending.

## Phase 3: Normalize Failure Reasons

Goal:

```text
Make failures easier to inspect.
```

Current code stores raw `str(error)` in `last_error`.

Improvement:

```text
Store normalized reason in a separate field or make status group by a stable
derived reason.
```

Candidate normalized reasons:

```text
timeout
connect_error
network_error
json_decode_error
http_404
http_429
http_500
unknown
```

Implementation options:

- add a `last_error_reason` column
- or derive reason at finalization time and store it in `last_error`
- or keep raw error and add a later migration

Recommended:

```text
Add `last_error_reason` in a small schema migration when ready.
```

Acceptance criteria:

- Status can group failures by stable reason.
- Raw error message is still available somewhere for debugging.
- Existing failure tests are updated.

## Phase 4: Add Database Growth Checks

Goal:

```text
Help the operator understand storage growth.
```

Add documentation or a command for:

```sql
SELECT count(*) FROM packages;
SELECT count(*) FROM package_versions;
SELECT count(*) FROM dependencies;
SELECT state, count(*) FROM crawl_frontier GROUP BY state;
```

Optional later checks:

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

Acceptance criteria:

- Runbook includes row-count and size-check guidance.
- Status command includes enough counts for normal small crawls.

## Phase 5: Add Safer Runbook Steps

Goal:

```text
Make the project operable by following docs.
```

Update `docs/runbook.md` with:

- setup checklist
- env var checklist
- seed command
- small crawl command
- time-boxed crawl command
- status command
- stop/resume steps
- failure interpretation
- recommended scaling path

Scaling path:

```text
1 package via fetch-one
25 package crawl
100 package crawl
5 minute crawl
1,000 package crawl
longer crawl
```

Acceptance criteria:

- A beginner can follow the runbook without guessing command order.
- The runbook warns against starting with an unbounded crawl.

## Phase 6: Test Operational Behavior

Goal:

```text
Protect the operational features.
```

Tests to add or maintain:

- status formatting with failures
- status formatting without failures
- status query returns expected counts using DB fixture
- CLI exposes `status`
- CLI exposes `--max-seconds`
- `run(max_seconds=0)` exits with reason `max_seconds`
- empty queue exits with reason `queue_empty`
- max package limit exits with reason `max_packages`

Acceptance criteria:

- Non-DB status formatting tests pass.
- DB-backed status tests pass when `TEST_DATABASE_URL` is set.
- Full suite remains green or DB tests skip cleanly without test DB.

## Phase 7: Readiness Review

Goal:

```text
Decide whether the crawler is ready for a larger bounded run.
```

Checklist:

- schema applied
- seed command works
- status command works
- small crawl completes
- failures are understandable
- queue resumes after stop
- row counts look reasonable
- no runaway retry behavior
- request rate is conservative
- runbook is current

Acceptance criteria:

- Run a bounded crawl and paste summary into `docs/runbook.md` or a notes file.
- Confirm no unbounded crawl is required to demonstrate readiness.

## Test Plan

Fast tests:

```powershell
.\myvenv\Scripts\python.exe -m pytest tests\test_status.py -q
```

Broader non-DB tests:

```powershell
.\myvenv\Scripts\python.exe -m pytest tests\test_errors.py tests\test_rate.py tests\test_status.py -q
```

DB tests when `TEST_DATABASE_URL` exists:

```powershell
.\myvenv\Scripts\python.exe -m pytest
```

Manual CLI checks:

```powershell
.\myvenv\Scripts\python.exe -m crawler.cli crawl --help
.\myvenv\Scripts\python.exe -m crawler.cli status --help
```

## Risk Notes

Risk:

```text
Crawler appears idle but actually has stuck in_progress rows.
```

Mitigation:

```text
Expose in_progress counts and lease-related checks.
```

Risk:

```text
Failures are too noisy to understand.
```

Mitigation:

```text
Normalize failure reasons and group status output.
```

Risk:

```text
Database grows faster than expected.
```

Mitigation:

```text
Track row counts and table sizes during bounded runs.
```

Risk:

```text
User accidentally starts an unbounded crawl.
```

Mitigation:

```text
Runbook recommends bounded commands first; CLI supports max packages/time.
```

## Final Interview Story

The story should be:

> Once the crawler could fetch and resume correctly, I worked on making it
> operable. I added bounded crawl options, a status command, failure summaries,
> and a runbook so I could run progressively larger crawls safely. I treated
> failures and database growth as operational signals, not just bugs, which is
> important for long-running data collection systems.
