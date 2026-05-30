# Milestone 5: Full-Registry Crawl Readiness

This document is for revision and interview preparation. Milestone 5 is about
turning the crawler from a working prototype into something safe to run for a
longer crawl.

Important: "full-registry readiness" does not mean recklessly crawling millions
of packages immediately. It means adding the controls, visibility, and runbook
needed to crawl responsibly.

## Goal

Milestone 5 makes the crawler operational.

The system should:

- run bounded crawls safely
- show current crawl progress
- summarize failures
- support stop/resume behavior
- expose enough metrics to know whether the crawler is healthy
- document how to operate the crawler
- prepare the database for larger query volumes

In one sentence:

> Milestone 5 makes the crawler safe to operate for longer runs by adding
> status visibility, bounded execution, failure inspection, and a practical
> runbook.

## Why This Milestone Matters

Milestones 1-3 prove the crawler works.

Milestone 5 asks a different question:

```text
Can I safely run this for a long time and understand what is happening?
```

Without operational readiness, a crawler can fail quietly:

- queue gets stuck
- failures grow unnoticed
- database fills unexpectedly
- one bad retry policy loops forever
- the process stops and nobody knows where it left off
- performance degrades as tables grow

Milestone 5 solves the "can I operate this?" problem.

This is especially relevant for MixRank because their work involves real
crawling, large datasets, production systems, and long-running data pipelines.

Interview phrase:

> After proving the crawler worked, I focused on operational readiness: bounded
> runs, status inspection, failure summaries, restart behavior, and a runbook.
> That made the project more like a real data pipeline instead of just a script.

## Architecture

The main pieces are:

- `crawler/loop.py`: runs the crawl and supports bounded execution
- `crawler/status.py`: reads an operational snapshot from PostgreSQL
- `crawler/cli.py`: exposes `crawl` and `status` commands
- `docs/runbook.md`: explains how to run, inspect, stop, and resume
- `sql/schema.sql`: stores crawl state and indexes claimable work

Current operational flow:

```text
seed frontier
   |
   v
run bounded crawl
   |
   v
periodically inspect status
   |
   v
review failures
   |
   v
adjust crawl settings or retry policy
   |
   v
resume crawl
```

## Concept 1: Bounded Crawls

A bounded crawl has an explicit stopping condition.

Examples:

```powershell
depcrawler crawl --max-packages 25
```

```powershell
depcrawler crawl --max-seconds 300
```

Why bounds matter:

- safer while testing
- easier to reproduce
- easier to benchmark
- prevents accidental long runs
- lets you inspect database state between runs

This is a beginner-friendly production habit:

```text
Start small, observe, then scale.
```

## Concept 2: Stop Reasons

The crawl summary includes a stop reason.

Examples:

```text
reason=queue_empty
reason=max_packages
reason=max_seconds
```

Why this matters:

```text
"The crawler stopped" is not enough information.
```

You need to know whether it stopped because:

- there was no work left
- it reached the package limit
- it reached the time limit
- later: it hit an error condition or shutdown signal

## Concept 3: Status Command

Milestone 5 needs an operator view.

The command:

```powershell
depcrawler status
```

should summarize:

```text
frontier: pending, in_progress, done, failed
stored: packages, versions, dependencies
failure reasons
```

The current status output is intentionally simple:

```text
crawl status
frontier: pending=3 in_progress=1 done=8 failed=2
stored: packages=8 versions=15 dependencies=42
failure reasons:
  2x timeout
```

This helps answer:

- Is the crawler making progress?
- How much work remains?
- Are packages stuck in progress?
- How much data has been stored?
- What failures are common?

Interview phrase:

> I added a status command because a long-running crawler needs an operator
> interface. It reports queue state, stored row counts, and top failure reasons.

## Concept 4: Failure Inspection

Failures are not just errors. They are data.

Useful failure questions:

- Are most failures 404s?
- Are many requests timing out?
- Are 429s common?
- Did retries exhaust?
- Are failures concentrated on scoped packages?

The current `status` command groups by `last_error`.

Future improvement:

```text
store normalized failure reason separately from raw error message
```

Example normalized reasons:

```text
timeout
connect_error
http_404
http_429
json_decode_error
```

This makes failure analysis cleaner.

## Concept 5: Restart Safety

Milestone 2 added leases. Milestone 5 operationalizes that behavior.

If the process stops:

```text
done rows stay done
pending rows stay pending
in_progress rows can be reclaimed after lease expiry
failed rows stay failed
```

This means:

```text
Ctrl+C is not catastrophic.
```

The runbook should explain:

- how to stop the crawler
- how long to wait for leases to expire
- how to check status before resuming
- how to identify stuck rows

## Concept 6: Database Growth

A longer crawl grows several tables:

```text
packages
package_versions
dependencies
raw_documents
crawl_frontier
```

Growth affects:

- disk usage
- index size
- query speed
- backup/export time
- status query performance

Milestone 5 should add practical checks:

```sql
SELECT count(*) FROM packages;
SELECT count(*) FROM package_versions;
SELECT count(*) FROM dependencies;
SELECT state, count(*) FROM crawl_frontier GROUP BY state;
```

Future improvement:

```text
Add a db-stats command or SQL snippets for table/index size.
```

## Concept 7: Index Health

Indexes make important queries fast.

Current important indexes:

- dependency reverse lookup: `dependencies_by_target`
- frontier claim lookup: `crawl_frontier_claimable`

The claim query needs to quickly find pending or expired work.

The dependency index supports questions like:

```text
Who depends on package X?
```

Milestone 5 should not add random indexes. Add indexes only when there is a
real query pattern or `EXPLAIN` shows a problem.

Interview phrase:

> I treated indexes as part of query design. I added indexes for known access
> patterns, especially frontier claiming and reverse dependency lookup, rather
> than indexing everything blindly.

## Concept 8: Runbook

A runbook is an operations guide.

The current runbook lives at:

```text
docs/runbook.md
```

It should explain:

- database setup
- environment variables
- schema application
- seeding
- bounded crawl commands
- status inspection
- stop/resume behavior
- common failure interpretation

Why this matters:

```text
If you cannot explain how to run the system, the system is not ready.
```

## Concept 9: Scaling Carefully

A safe scaling path:

```text
fetch-one
10 packages
25 packages
100 packages
1,000 packages
longer time-boxed crawl
```

At each step, inspect:

- status output
- failure reasons
- database row counts
- elapsed time
- packages/sec
- disk usage
- whether the queue is growing too fast

Do not jump straight to millions of packages.

## Concept 10: What "Full Registry" Means

npm has millions of packages. Full-registry crawling creates practical issues:

- time
- storage
- retries
- registry politeness
- database maintenance
- query performance
- crash recovery
- result analysis

Milestone 5 does not need to finish a full registry crawl.

Milestone 5 should make the project ready for progressively larger crawls.

## Tests

Milestone 5 tests should cover:

- status formatting
- status query shape
- crawl stop reason for `max_packages`
- crawl stop reason for `max_seconds`
- empty queue stop reason
- failure reason grouping
- CLI exposes `status`
- CLI exposes bounded crawl options

Current lightweight tests:

```powershell
.\myvenv\Scripts\python.exe -m pytest tests\test_status.py -q
```

Full non-DB safety check:

```powershell
.\myvenv\Scripts\python.exe -m pytest tests\test_errors.py tests\test_rate.py tests\test_status.py -q
```

DB-backed tests require:

```text
TEST_DATABASE_URL
```

## Design Patterns In This Milestone

### Operational visibility

Long-running systems need status commands and useful logs.

### Bounded execution

Every risky operation should have a safe small mode.

### Durable state

PostgreSQL stores progress, so the crawler can stop and resume.

### Failure as data

Failures should be summarized, grouped, and investigated.

### Runbooks

A system should be operable by following documented steps.

### Evidence-based indexing

Add indexes for real query patterns, not guesses.

## Interview Answer

If asked what Milestone 5 is about, say:

> Milestone 5 is about operational readiness. Earlier milestones proved the
> crawler could fetch, store, resume, and behave politely. In this milestone I
> make it safer to run for longer periods by adding bounded crawl options,
> status reporting, failure summaries, and a runbook. I also start thinking
> about database growth and query/index health, because a crawler becomes a data
> pipeline once it runs beyond a tiny demo.

## What To Review Before Implementing

Make sure you can explain:

- why bounded crawls are safer than open-ended runs
- what each status count means
- why failure reasons should be grouped
- how leases support stop/resume
- why database row counts matter
- why index design follows query patterns
- what a runbook is
- how to scale from small crawls to larger crawls carefully