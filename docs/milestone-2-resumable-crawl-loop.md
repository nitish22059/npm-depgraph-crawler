# Milestone 2: Resumable Crawl Loop

This document is for revision and interview preparation. It explains how the
project moved from fetching one package to running a restart-safe crawl loop.

## Goal

Milestone 2 turns the project into a real crawler.

The system should:

- keep crawl work in PostgreSQL
- claim one package at a time
- mark claimed work as `in_progress`
- recover abandoned work after a lease expires
- store successful package data
- retry temporary failures
- mark permanent failures
- enqueue newly discovered dependencies

In one sentence:

> Milestone 2 makes the crawler resumable by using PostgreSQL as a durable work
> queue.

## Architecture

The main files are:

- `crawler/seed.py`: inserts starting packages into the frontier
- `crawler/loop.py`: claim, fetch, success/failure finalization, crawl loop
- `crawler/errors.py`: classifies errors as retriable or fatal
- `crawler/store.py`: stores package data without owning frontier transitions
- `sql/schema.sql`: frontier state and lease columns
- `tests/test_loop_claim.py`: claim/lease behavior tests
- `tests/test_loop_finalize.py`: success/failure transition tests
- `tests/test_errors.py`: error classification tests

The crawl loop data flow is:

```text
seed packages
   |
   v
crawl_frontier rows in pending state
   |
   v
claim(conn)
   |
   v
one package becomes in_progress with a lease
   |
   v
fetch package document
   |
   +--> success: store data, mark done, enqueue dependencies
   |
   +--> failure: classify error, retry or mark failed
```

## Concept 1: The Frontier

The frontier is the queue of packages to crawl.

In this project, the frontier is a PostgreSQL table:

```text
crawl_frontier
```

Important columns:

```text
name
state
attempts
last_attempted_at
last_error
discovered_at
claim_expires_at
```

The important idea:

```text
The database is both storage and work queue.
```

That makes the crawl resumable. If the program stops, the queue is still in
PostgreSQL.

Interview phrase:

> I used Postgres as a durable queue so crawl progress survives process crashes
> and restarts.

## Concept 2: State Machine

Each package in the frontier has a state.

Current states:

```text
pending
in_progress
done
failed
```

Meaning:

```text
pending      -> waiting to be crawled
in_progress  -> claimed by a worker
done         -> fetched and stored successfully
failed       -> permanently failed or retries exhausted
```

Normal success flow:

```text
pending -> in_progress -> done
```

Temporary failure flow:

```text
pending -> in_progress -> pending
```

Permanent failure flow:

```text
pending -> in_progress -> failed
```

This is called a state machine.

## Concept 3: Seeding

Before the crawler can run, the frontier needs starting packages.

`crawler/seed.py` provides:

```python
seed_default(conn)
seed_from_file(conn, path)
```

Default seeds include common packages like:

```text
react
vue
express
lodash
typescript
eslint
```

The insert uses:

```sql
ON CONFLICT DO NOTHING
```

So seeding is idempotent.

Running the seed command twice does not duplicate rows.

## Concept 4: Claiming Work

The `claim(conn)` function chooses one package to crawl.

It looks for:

```text
pending packages
OR in_progress packages whose lease expired
```

Then it marks one row:

```text
state = in_progress
attempts = attempts + 1
claim_expires_at = now() + 60 seconds
last_attempted_at = now()
```

It returns:

```python
(name, attempts)
```

or:

```python
None
```

if no work is available.

## Concept 5: `FOR UPDATE SKIP LOCKED`

The claim query uses:

```sql
FOR UPDATE SKIP LOCKED
```

This is a key production pattern.

Meaning:

```text
Lock the row I am claiming.
If another worker already locked a row, skip it.
```

Why this matters:

- two workers should not process the same package at the same time
- workers can run concurrently
- locked rows do not block other workers unnecessarily

Interview phrase:

> I used `SELECT ... FOR UPDATE SKIP LOCKED` so multiple workers could claim
> crawl jobs concurrently without double-processing the same package.

## Concept 6: Leases And Crash Recovery

A worker can crash after claiming a package.

Without leases, the package might stay stuck forever:

```text
in_progress forever
```

Milestone 2 adds:

```text
claim_expires_at
```

When a package is claimed:

```text
claim_expires_at = now() + 60 seconds
```

If the worker dies, another worker can later reclaim it after the lease expires.

The claim condition includes:

```sql
state = 'in_progress' AND claim_expires_at < now()
```

Plain English:

```text
If work was claimed but the lease expired, it is safe to retry.
```

This is what makes the crawl crash-resilient.

## Concept 7: Success Finalization

When a fetch succeeds:

```python
finalize_success(conn, name, doc)
```

does three things:

```text
store normalized package data
mark current package as done
insert discovered dependencies as pending
```

The discovered dependencies are inserted with:

```sql
ON CONFLICT DO NOTHING
```

So if the dependency was already known, it is not duplicated.

This is how the graph crawl expands:

```text
fetch react
discover loose-envify
enqueue loose-envify
later fetch loose-envify
```

## Concept 8: Failure Finalization

When a fetch fails:

```python
finalize_failure(conn, name, error, attempts)
```

The error is classified:

```python
classification = classify(error)
```

If the error is retriable and attempts are still below the max:

```text
state = pending
```

Otherwise:

```text
state = failed
```

The current max attempts is:

```python
MAX_ATTEMPTS = 3
```

This prevents infinite retry loops.

## Concept 9: Retriable vs Fatal Errors

`crawler/errors.py` decides whether an error should be retried.

Retriable examples:

```text
HTTP 429
HTTP 500
HTTP 502
HTTP 503
HTTP 504
timeouts
network errors
connection errors
```

Fatal examples:

```text
HTTP 404
HTTP 403
bad JSON
unexpected value errors
```

Why this matters:

```text
temporary infrastructure failure -> retry
package does not exist / bad input -> fail
```

Interview phrase:

> I separated error classification from queue transitions so retry policy is
> explicit and testable.

## Concept 10: The Crawl Loop

The loop repeats:

```text
claim work
if no work: stop
fetch package
if success: finalize_success
if failure: finalize_failure
respect max package limit
```

Simplified:

```python
while True:
    result = claim(conn)
    if result is None:
        break

    try:
        doc = fetch_package(name)
        finalize_success(conn, name, doc)
    except Exception as e:
        finalize_failure(conn, name, e, attempts)
```

This structure is common in crawlers and job workers.

## Concept 11: Observability

The loop prints events:

```text
claim
fetching
done
retry
failed
tick
crawl done
```

Why this matters:

Long-running jobs need visibility. Logs help answer:

- Is the crawler progressing?
- Is the queue growing?
- Are failures temporary or permanent?
- How many attempts has a package used?

## Tests

Milestone 2 tests cover the riskiest behavior.

### Claim tests

`tests/test_loop_claim.py` verifies:

- empty queue returns `None`
- pending row becomes `in_progress`
- attempts increment
- lease is set
- valid leases are skipped
- expired leases can be reclaimed

### Finalization tests

`tests/test_loop_finalize.py` verifies:

- success marks package `done`
- success clears lease and error
- discovered dependencies are seeded
- dependency seeding is idempotent
- retriable errors go back to `pending` under max attempts
- retriable errors become `failed` after max attempts
- fatal errors become `failed`

### Error tests

`tests/test_errors.py` verifies:

- HTTP status classification
- timeout classification
- network error classification
- JSON decode classification
- unknown exception classification

Run Milestone 2 tests:

```powershell
.\myvenv\Scripts\python.exe -m pytest tests\test_loop_claim.py tests\test_loop_finalize.py tests\test_errors.py -q
```

The claim/finalize tests require `TEST_DATABASE_URL`.

## Design Patterns In This Milestone

### Durable queue

Work is stored in PostgreSQL, not memory.

### State machine

Each package moves through explicit states.

### Lease-based recovery

Expired claims can be retried after a crash.

### Atomic claim

`FOR UPDATE SKIP LOCKED` prevents double-claiming.

### Idempotency

Discovered dependencies use conflict-safe inserts.

### Retry policy

Error classification controls whether work is retried or failed.

## Interview Answer

If asked what Milestone 2 did, say:

> I turned the single-package fetcher into a resumable crawler by using
> PostgreSQL as a durable work queue. Each package lives in `crawl_frontier`
> with states like `pending`, `in_progress`, `done`, and `failed`. Workers claim
> rows atomically using `FOR UPDATE SKIP LOCKED`, set a lease, and increment
> attempts. If a worker crashes, the lease expires and another worker can
> reclaim the package. Success stores the package and enqueues discovered
> dependencies; failure uses explicit error classification to decide whether to
> retry or mark the package failed.

## What To Review Before Moving On

Make sure you can explain:

- why the frontier lives in PostgreSQL
- what each crawl state means
- why claiming must be atomic
- what `FOR UPDATE SKIP LOCKED` protects against
- how leases recover abandoned work
- how success expands the dependency graph
- why retriable and fatal errors are handled differently
- why idempotent dependency enqueueing matters
