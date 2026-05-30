# Milestone 3: Politeness + Throughput

This document is for revision and interview preparation. It explains what
Milestone 3 added, why it matters, and how the code fits together.

## Goal

Milestone 3 makes the crawler more production-friendly.

The crawler should:

- reuse HTTP connections instead of creating a new client for every package
- avoid sending requests too quickly
- respect server backoff instructions like `Retry-After`
- expose CLI controls for safe small crawls
- log enough information to understand what the crawler is doing

In one sentence:

> Milestone 3 improves throughput with connection reuse while keeping the
> crawler polite with rate limiting and server-directed pauses.

## Architecture

The main files are:

- `crawler/fetch.py`: creates HTTP clients and fetches npm package documents
- `crawler/rate.py`: owns rate limiting and `Retry-After` parsing
- `crawler/loop.py`: orchestrates claim, fetch, store, retry, and logging
- `crawler/cli.py`: exposes crawl configuration as command-line options
- `tests/test_rate.py`: tests rate limiter and `Retry-After` behavior

The data flow is:

```text
CLI options
   |
   v
loop.run(...)
   |
   v
RateLimiter(requests_per_second)
   |
   v
with new_client() as client
   |
   v
claim package from crawl_frontier
   |
   v
limiter.wait()
   |
   v
fetch_package(name, client=client)
   |
   +--> success: store data, mark done, enqueue dependencies
   |
   +--> failure: classify error, maybe honor Retry-After, retry or fail
```

## Concept 1: Throughput With Client Reuse

Before Milestone 3, the simple way to fetch a package was:

```python
fetch_package("react")
```

That creates a new `httpx.Client`, sends one request, and closes it.

This is fine for one package, but inefficient for a crawler. A crawler may fetch
thousands or millions of packages. Creating a new client each time loses the
benefit of connection pooling.

Milestone 3 uses one client for the crawl loop:

```python
with new_client() as client:
    doc = fetch_package(name, client=client)
```

This means:

- `loop.py` creates the client
- `loop.py` owns the client
- `fetch_package()` only borrows it
- the client is closed when the `with` block exits

Interview phrase:

> I kept HTTP fetching flexible: `fetch_package()` can create its own client for
> one-off calls, but the crawl loop passes a shared client so connection pooling
> works across many package requests.

## Concept 2: Ownership

Ownership means: who is responsible for cleaning up a resource?

In `fetch_package()`:

```python
owned = client is None
```

If no client is passed, `fetch_package()` creates one and owns it. Because it
created the client, it closes the client.

If a client is passed in, `fetch_package()` only borrows it. It must not close a
client owned by the caller.

Rule:

```text
create it -> own it -> close it
borrow it -> use it -> do not close it
```

This applies to:

- HTTP clients
- database connections
- files
- sockets
- locks

## Concept 3: Rate Limiting

A crawler should not send requests as fast as possible. That can overload a
server or get the crawler blocked.

The rate limiter answers this question:

```text
Is it okay to send the next request now?
```

The default setting is:

```python
DEFAULT_REQUESTS_PER_SECOND = 2.0
```

That means:

```text
2 requests per second = at least 0.5 seconds between requests
```

The calculation is:

```python
self.min_interval = 1.0 / requests_per_second
```

Examples:

```text
1 request/sec  -> 1.0 second interval
2 requests/sec -> 0.5 second interval
4 requests/sec -> 0.25 second interval
10 requests/sec -> 0.1 second interval
```

## Concept 4: The `RateLimiter` Object

`RateLimiter` is a class because it needs memory.

It remembers:

```python
self._next_allowed_at
```

That value means:

```text
The next time a request is allowed.
```

The first request usually does not sleep. After that, the limiter updates the
next allowed time.

Example with 2 requests per second:

```text
min_interval = 0.5
initial next_allowed_at = 0.0

first wait:
    now = 0.0
    no sleep
    next_allowed_at = 0.5

second wait immediately:
    now = 0.0
    sleep 0.5
    next_allowed_at = 1.0
```

The important method is:

```python
limiter.wait()
```

It returns how many seconds it slept. The crawl loop logs that as `throttle`.

## Concept 5: Dependency Injection For Time

`RateLimiter` accepts a clock and sleeper:

```python
clock: Clock = time.monotonic
sleeper: Sleeper = time.sleep
```

In production:

- `clock` is `time.monotonic`
- `sleeper` is `time.sleep`

In tests:

- `clock` is a fake clock
- `sleeper` is a fake sleep function

This keeps tests fast.

Without fake sleep, a test for a one-second delay would actually wait one real
second. With fake sleep, the test records the sleep and moves fake time forward
instantly.

Interview phrase:

> I injected the clock and sleeper so time-dependent behavior could be tested
> without slow real sleeps.

## Concept 6: Retry-After

Sometimes a server tells clients to slow down:

```text
HTTP 429 Too Many Requests
Retry-After: 60
```

That means:

```text
Wait 60 seconds before retrying.
```

The crawler parses that with:

```python
parse_retry_after(value)
```

It supports two valid forms.

First form: seconds

```text
Retry-After: 7
```

Result:

```python
7.0
```

Second form: HTTP date

```text
Retry-After: Sat, 23 May 2026 12:00:05 GMT
```

If current time is `12:00:00`, result is:

```python
5.0
```

Invalid or missing values return:

```python
None
```

This distinction matters:

- `None` means no usable instruction exists
- `0.0` means the parsed delay is zero seconds

## Concept 7: Server-Directed Pause

Normal rate limiting is our rule:

```text
Send at most 2 requests per second.
```

`Retry-After` is the server's rule:

```text
Wait this long before continuing.
```

The rate limiter handles server-directed waiting with:

```python
limiter.pause(seconds)
```

It:

- ignores zero or negative pauses
- sleeps for the requested number of seconds
- updates `_next_allowed_at` so normal rate limiting still applies after the pause

In `loop.py`:

```python
retry_after = _retry_after_delay(e, retry_after_max_seconds)
if retry_after is not None:
    limiter.pause(retry_after)
```

The delay is capped:

```python
DEFAULT_RETRY_AFTER_MAX_SECONDS = 300.0
```

This prevents one response from freezing the crawler for a very long time.

## Concept 8: Crawl Loop Integration

Milestone 3 changes the fetch section of the crawl loop.

The important sequence is:

```python
slept = limiter.wait()
print_line("fetching", name, throttle=f"{slept:.2f}s")
doc = fetch_package(name, client=client)
```

Plain English:

```text
Before fetching, wait if needed.
Log how much throttling happened.
Fetch the package using the shared HTTP client.
```

On failure:

```python
retry_after = _retry_after_delay(e, retry_after_max_seconds)
if retry_after is not None:
    limiter.pause(retry_after)
```

Plain English:

```text
If the error includes Retry-After, pause before continuing.
```

## Concept 9: CLI Controls

The crawl command exposes knobs:

```powershell
depcrawler crawl --max-packages 25 --requests-per-second 2
```

Options:

```text
--max-packages
--max-seconds
--requests-per-second
--retry-after-max-seconds
```

Why this matters:

- `--max-packages` makes demo crawls safe
- `--max-seconds` makes time-boxed experiments safe
- `--requests-per-second` controls crawler politeness
- `--retry-after-max-seconds` prevents extremely long sleeps

Interview phrase:

> I exposed operational controls in the CLI so I could safely run bounded
> crawls and tune politeness without editing code.

## Concept 10: Observability

Long-running crawlers need visibility.

Milestone 3 logs events like:

```text
claim
fetching
done
retry
failed
tick
crawl done
```

Examples of useful fields:

```text
attempts=1/3
queue=120
throttle=0.50s
reason=http_429
retry_after=60.00s
```

This helps answer:

- Is the crawler making progress?
- Is it being throttled?
- Is npm returning errors?
- Is the queue growing or shrinking?
- How many packages are done or failed?

## Tests

Milestone 3 tests live in `tests/test_rate.py`.

They verify:

- first request does not sleep
- second immediate request sleeps the right interval
- rate limiting can be disabled with `0`
- `pause()` sleeps for server-directed delay
- numeric `Retry-After` values parse correctly
- HTTP-date `Retry-After` values parse correctly
- invalid values return `None`

Run only Milestone 3 tests:

```powershell
.\myvenv\Scripts\python.exe -m pytest tests\test_rate.py -q
```

Run the full test suite:

```powershell
.\myvenv\Scripts\python.exe -m pytest
```

Expected result currently:

```text
Milestone 3 tests: 6 passed
Full suite: 22 passed, 17 skipped
```

The skipped tests require `TEST_DATABASE_URL`.

## Design Patterns In This Milestone

### Separation of concerns

Rate logic is in `crawler/rate.py`.

Crawl orchestration is in `crawler/loop.py`.

CLI parsing is in `crawler/cli.py`.

This keeps each file focused.

### Ownership

The code that creates a resource closes it.

The crawl loop creates the shared HTTP client, so the crawl loop owns its
lifetime.

### Dependency injection

`RateLimiter` accepts a clock and sleeper so tests can control time.

### Guard clauses

Invalid settings are rejected early:

```python
if requests_per_second < 0:
    raise ValueError(...)
```

### Defensive parsing

`parse_retry_after()` accepts real-world messy input and returns either a clean
number of seconds or `None`.

## Interview Answer

If asked what Milestone 3 did, say:

> I improved the Python crawler's production behavior. First, I reused a single
> `httpx.Client` inside the crawl loop so HTTP connections can be pooled instead
> of recreated for every package. Then I added a small rate limiter to enforce a
> configurable request rate before each fetch. I also parse and honor
> `Retry-After` headers on retriable HTTP errors, with a maximum cap so one
> server response cannot pause the crawler indefinitely. Finally, I exposed the
> controls through CLI options and added tests using a fake clock so the timing
> behavior is deterministic and fast to test.

## What To Review Before Moving On

Make sure you can explain:

- why connection reuse improves throughput
- why a crawler should be rate limited
- how `requests_per_second` becomes `min_interval`
- why `RateLimiter` needs `_next_allowed_at`
- why tests use a fake clock
- what `Retry-After` means
- why `None` and `0.0` mean different things
- how `loop.py` connects the limiter, fetcher, and retry handling

Once those are clear, Milestone 4 can begin with profiling.
