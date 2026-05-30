# Milestone 4: Profile, Then Port The Hot Path To Rust

This document is for revision and interview preparation. Milestone 4 is about
moving carefully from a working Python crawler to a measured Rust hot path.

Important: this milestone should not start by writing Rust. It should start by
profiling the Python crawler and proving which part is actually slow.

## Goal

Milestone 4 should make Rust a response to evidence, not decoration.

The system should:

- measure Python crawler performance first
- identify the actual bottleneck
- keep Python responsible for orchestration and PostgreSQL writes
- introduce Rust behind a narrow interface
- preserve the Python fallback path
- compare Python and Rust behavior with tests and benchmarks

In one sentence:

> Milestone 4 profiles the Python crawler, then moves only the fetch/parse hot
> path into Rust while keeping queue orchestration and database ownership in
> Python.

## Why Introduce Rust?

Rust should be introduced only if it solves a real problem observed in the
Python crawler.

The problem is not:

```text
Python is bad.
```

The real question is:

```text
Which part of this crawler limits throughput as the crawl grows?
```

At small scale, Python is excellent for this project:

- easy to write
- easy to debug
- strong PostgreSQL libraries
- simple CLI and orchestration
- fast enough for early milestones

But at larger scale, the crawler may spend a lot of time in the repeated
fetch/parse path:

```text
build request
send HTTP request
read response body
decode JSON
shape result
repeat thousands or millions of times
```

That repeated path is a candidate for Rust if profiling shows Python overhead is
meaningful.

Rust can help with:

- high-concurrency network fetching
- lower per-request overhead
- efficient response body handling
- predictable memory usage
- CPU-heavy parsing or transformation work
- moving performance-critical code into compiled code

Rust also gives stronger compile-time guarantees around data types and error
handling, which is useful when building a small, well-defined hot path.

The problem Milestone 4 is trying to solve:

```text
Improve crawler throughput for the repeated fetch/parse path without making
the whole system harder to reason about.
```

The important constraint:

```text
Do not move the whole crawler to Rust.
```

Python is still the better place for:

- orchestration
- CLI behavior
- PostgreSQL queue transitions
- retry policy
- observability
- analysis workflow

Rust should be used like a focused tool:

```text
Python controls the system.
Rust accelerates the hot path.
```

Interview phrase:

> I introduced Rust only after profiling because I wanted it to solve a concrete
> throughput problem. The likely bottleneck is the repeated fetch/parse path, so
> I keep Python as the control plane and use Rust only for the performance
> sensitive data plane.

## What Problem Are We Solving?

The crawler starts with a simple Python loop:

```text
claim one package
fetch one package
parse one JSON document
write one package to Postgres
repeat
```

This is easy to understand, but it can become slow when the number of packages
gets large.

Potential symptoms:

- packages per second stays low even when the database is healthy
- CPU time is spent decoding or shaping JSON
- one-at-a-time fetching wastes time waiting on network latency
- increasing the allowed request rate does not improve throughput much
- the Python process becomes the bottleneck before Postgres or npm does

Milestone 4 investigates whether the fetch/parse part should become:

```text
claim several packages in Python
fetch and parse them concurrently in Rust
return structured results to Python
store/finalize them in Python
```

The goal is not just speed. The goal is controlled speed:

- faster repeated fetch/parse work
- same durable PostgreSQL queue
- same retry/failure behavior
- same crawler politeness controls
- measurable improvement over the Python baseline

This is the kind of tradeoff systems engineers make often:

```text
Keep most code simple.
Move only the proven bottleneck into a faster, more specialized layer.
```

## Architecture

Current architecture before Rust:

```text
loop.py
   |
   v
RateLimiter.wait()
   |
   v
fetch.fetch_package(name, client)
   |
   v
Python httpx request + Python JSON parsing
   |
   v
store_package_data(conn, doc)
   |
   v
PostgreSQL writes
```

Target architecture after the first Rust slice:

```text
loop.py
   |
   v
claim batch of package names
   |
   v
Python chooses fetch backend
   |
   +--> Python fallback: fetch_package(...)
   |
   +--> Rust hot path: fetch_many(...)
   |
   v
Python receives structured results
   |
   v
Python stores successes and finalizes failures
   |
   v
PostgreSQL remains source of truth
```

The key ownership split:

```text
Python owns:
    CLI
    crawl loop
    queue claiming
    retry policy
    rate/politeness config
    PostgreSQL writes

Rust owns:
    concurrent HTTP fetch
    response body handling
    JSON parse into Python-compatible values
    structured fetch result creation
```

## Concept 1: What Is A Hot Path?

A hot path is the part of the program that runs often enough to dominate
performance.

In this crawler, possible hot paths are:

- HTTP requests
- JSON decoding
- dependency extraction
- PostgreSQL inserts and updates
- queue claim queries

Do not guess. Measure.

Interview phrase:

> I treated Rust as an optimization tool, not a badge. I first measured where
> the Python crawler spent time, then chose the smallest boundary that addressed
> the bottleneck.

## Concept 2: Profiling Before Optimization

Profiling means measuring where time goes.

Before writing Rust, run a bounded Python crawl and record:

- packages stored
- failures
- elapsed seconds
- packages per second
- average versions per package
- average dependencies discovered per package
- average fetch time
- average store/finalize time

Useful questions:

```text
Is the crawler waiting on network?
Is Python JSON decoding expensive?
Are PostgreSQL writes the bottleneck?
Are queue count queries too frequent?
Is rate limiting intentionally slowing the crawl?
```

If the crawler is mostly waiting on network or database writes, Rust may not
help much yet.

If Python-side fetch/parse work is large, Rust becomes more justified.

## Concept 3: Baseline Measurement

Before porting, create a baseline.

Example baseline table:

```text
Run date:
Machine:
Python version:
Database:
Package limit:
Requests/sec:

Stored:
Failed:
Elapsed:
Packages/sec:
Average versions/package:
Average deps discovered/package:
Notes:
```

Why this matters:

```text
Without a baseline, you cannot prove Rust improved anything.
```

Milestone 4 should update `docs/profiling.md` with real numbers from a small
crawl before the Rust bridge is added.

## Concept 4: Why Keep PostgreSQL Writes In Python?

The project already has tested Python code for:

- package normalization
- idempotent inserts
- frontier transitions
- retry/failure behavior

Moving all of that into Rust at once would make the boundary too large.

The first Rust version should not own database writes.

Better first boundary:

```text
Rust fetches and parses package documents.
Python stores results and updates the queue.
```

Why this is safer:

- smaller Rust API
- fewer moving parts
- existing DB tests still matter
- rollback to Python fallback is easy
- Python remains the orchestration layer

Interview phrase:

> I intentionally kept database writes in Python because they were already
> correct and tested. The first Rust boundary only covered fetch/parse work,
> which kept the integration small and reversible.

## Concept 5: Python/Rust Boundary

The first Rust function should be narrow.

Target shape:

```text
fetch_many(package_names, concurrency, user_agent) -> list[result]
```

Inputs:

```text
package_names: list of npm package names
concurrency: how many requests Rust may run at once
user_agent: HTTP User-Agent string
```

Output:

```text
list of structured results
```

A success result should contain:

```text
package name
registry document
```

A failure result should contain:

```text
package name
error class
status code if available
message
```

The caller should not have to parse random strings to understand errors.

## Concept 6: Structured Results

Avoid a vague API like:

```text
list[dict]
```

with inconsistent shapes.

Prefer a clear result shape.

Conceptually:

```text
FetchSuccess:
    name
    document

FetchFailure:
    name
    error_kind
    status_code
    message
```

Why this matters:

- Python can finalize success/failure cleanly
- retry policy remains explicit
- tests can assert exact behavior
- errors are not hidden inside logs

## Concept 7: PyO3 And Maturin

PyO3 lets Rust expose functions that Python can import.

Maturin builds the Rust extension so Python can use it.

Mental model:

```text
Rust code
   |
   v
compiled extension module
   |
   v
Python import
```

Python should eventually be able to do something like:

```python
from rust_hotpath import fetch_many
```

Then:

```python
results = fetch_many(names, concurrency=16, user_agent=USER_AGENT)
```

Beginner warning:

Rust extension setup has more packaging details than normal Python code. Keep
the first function simple so debugging stays manageable.

## Concept 8: Fallback Path

The crawler should still work without Rust.

Why?

- easier development
- easier tests
- easier debugging
- safer rollout
- Rust build tools may not exist on every machine yet

Target behavior:

```text
If Rust backend is available and selected:
    use Rust fetch_many
Otherwise:
    use Python fetch_package path
```

Possible CLI shape:

```powershell
depcrawler crawl --fetch-backend python
depcrawler crawl --fetch-backend rust
```

Recommended default during development:

```text
python
```

After Rust is stable and tested:

```text
default can become rust, or remain explicit for demos
```

## Concept 9: Concurrency

The current Python loop fetches one package at a time.

Rust hot path may fetch many packages concurrently.

Concurrency means:

```text
start multiple network requests before all previous ones have finished
```

This can improve throughput when the crawler is waiting on network latency.

But concurrency must still respect politeness.

Important design rule:

```text
Higher concurrency does not mean unlimited request rate.
```

The crawler should still have:

- request rate limits
- retry-after handling
- bounded batch sizes
- clear logs

## Concept 10: Batch Claiming

The current claim function returns one package:

```python
(name, attempts)
```

Rust `fetch_many()` is most useful if Python can claim a small batch.

Future shape:

```text
claim_many(conn, limit) -> list[(name, attempts)]
```

The same safety rules apply:

- claim atomically
- use `FOR UPDATE SKIP LOCKED`
- set leases
- increment attempts
- return claimed rows

Do not start with a huge batch. Start with something small:

```text
10 or 25 packages
```

Then measure.

## Concept 11: Error Mapping

Rust errors must map back to Python retry policy.

Python already knows categories like:

```text
timeout
network_error
http_429
http_500
json_decode_error
```

Rust should return enough information for Python to decide:

```text
retriable or fatal?
```

Do not bury this inside a string like:

```text
"something went wrong"
```

Return structured fields.

## Concept 12: Testing The Rust Boundary

Tests should prove compatibility, not just that Rust compiles.

Test cases:

- success result has package name and document
- HTTP 404 maps to fatal-style failure
- HTTP 429 includes status code and can carry retry-after
- timeout/network failure maps to retriable-style failure
- invalid JSON maps to JSON decode failure
- Python fallback still works
- selected backend is respected

Also test that Python finalization behavior does not change:

```text
Rust success -> finalize_success
Rust failure -> finalize_failure
```

## Concept 13: Benchmarking

After Rust is integrated, compare:

```text
Python backend
Rust backend
```

Use the same:

- seed set
- package limit
- request rate
- database
- machine

Compare:

- elapsed time
- packages/sec
- failures
- CPU usage if available
- bottleneck notes

Do not claim Rust is faster without numbers.

Interview phrase:

> I kept a Python baseline and compared it to the Rust backend under the same
> crawl limit and rate settings, so the optimization was measurable.

## Implementation Path

Recommended step-by-step order:

1. Add simple timing around fetch and store phases in Python.
2. Run a bounded crawl and record baseline results in `docs/profiling.md`.
3. Add Rust project scaffolding under `rust_hotpath`.
4. Expose one tiny PyO3 function first, such as `version()` or `echo_names()`.
5. Confirm Python can import the compiled Rust module.
6. Implement Rust URL encoding and one-package fetch.
7. Extend to `fetch_many(names, concurrency, user_agent)`.
8. Return structured success/failure results.
9. Add Python backend selection with fallback.
10. Add tests for backend selection and result mapping.
11. Run Python-vs-Rust benchmark and document results.

This order avoids trying to debug Rust, packaging, HTTP, concurrency, and crawler
logic all at once.

## What Not To Do

Avoid:

- rewriting the whole crawler in Rust
- moving PostgreSQL writes into Rust immediately
- removing the Python fetch path too early
- adding concurrency before the single Rust call works
- claiming a speedup before benchmarking
- making Python parse unstructured Rust error strings

Good engineering is controlled change.

## Design Patterns In This Milestone

### Measure first

Optimization starts with evidence.

### Narrow interface

Rust should expose one clear fetch/parse API.

### Fallback

Python remains available if Rust is missing or broken.

### Structured errors

Rust returns machine-readable failure information.

### Keep ownership clear

Python owns orchestration and database state. Rust owns hot fetch/parse work.

### Incremental integration

Start with a tiny importable Rust function before building the full hot path.

## Interview Answer

If asked what Milestone 4 is about, say:

> Milestone 4 is where I make the Rust part evidence-based. I first profile the
> Python crawler to see whether fetch/parse work is actually the bottleneck.
> Then I introduce Rust behind a narrow PyO3 interface, starting with a
> `fetch_many` function that returns structured success and failure results.
> Python still owns the crawl loop, PostgreSQL queue, retries, and finalization,
> so the Rust boundary stays small and reversible. I keep a Python fallback and
> compare both backends with the same crawl limits before claiming any speedup.

## What To Review Before Implementing

Make sure you can explain:

- what a hot path is
- why profiling comes before optimization
- why Rust should start behind a narrow API
- why PostgreSQL writes stay in Python first
- what PyO3 and maturin do
- what structured success/failure results should contain
- why fallback matters
- how concurrency differs from request rate
- how to benchmark Python vs Rust fairly

## Milestone 4 Readiness Checklist

Before writing real Rust fetch code:

- Python Milestone 3 tests pass.
- A bounded Python crawl can run.
- Baseline metrics are written in `docs/profiling.md`.
- The Rust module can be built and imported from Python.
- The first Rust function is tiny and tested.
- The Python fallback path is still working.
