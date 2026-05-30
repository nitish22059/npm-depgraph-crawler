# Milestone 4 Implementation Plan

This is the execution plan for Milestone 4. The companion learning document is
`docs/milestone-4-profile-rust-hotpath.md`.

## Objective

Profile the Python crawler first, then introduce Rust only for the measured
fetch/parse hot path.

Milestone 4 is complete when:

- Python crawl baseline metrics are recorded
- a Rust extension can be built and imported from Python
- Python still works without Rust
- Rust exposes a narrow fetch/parse API
- Python can choose Python or Rust backend
- success and failure results map cleanly back into the existing crawl loop
- Python-vs-Rust comparison is documented

## Non-Goals

Do not:

- rewrite the whole crawler in Rust
- move PostgreSQL writes into Rust
- remove the Python fetch path
- change the database schema unless profiling proves it is necessary
- claim Rust is faster without benchmark numbers

## Phase 1: Baseline The Python Crawler

Goal:

```text
Measure the current crawler before adding Rust.
```

Tasks:

- Add lightweight timing around fetch and store/finalize phases.
- Run a bounded crawl with fixed settings.
- Record results in `docs/profiling.md`.

Suggested command:

```powershell
depcrawler crawl --max-packages 100 --requests-per-second 2
```

Record:

- package limit
- requests per second
- stored count
- failed count
- elapsed seconds
- packages per second
- average fetch time
- average store/finalize time
- notes about errors or throttling

Acceptance criteria:

- Baseline numbers exist in `docs/profiling.md`.
- The baseline can be repeated with the same command.
- We can explain whether fetch/parse appears worth optimizing.

## Phase 2: Add Rust Scaffolding

Goal:

```text
Create the smallest Rust extension that Python can import.
```

Tasks:

- Add Rust project files under `rust_hotpath`.
- Configure PyO3 and maturin.
- Expose a tiny function first, such as:

```text
version() -> str
```

or:

```text
echo_names(names: list[str]) -> list[str]
```

- Add a Python test that imports the module when available.

Acceptance criteria:

- Rust extension builds locally.
- Python can import it.
- A tiny Rust function can be called from Python.
- If Rust is not built, the Python test skips cleanly or uses fallback.

## Phase 3: Define Backend Interface In Python

Goal:

```text
Make Python able to choose a fetch backend without changing crawl logic.
```

Backend options:

```text
python
rust
```

Recommended CLI shape:

```powershell
depcrawler crawl --fetch-backend python
depcrawler crawl --fetch-backend rust
```

Default during development:

```text
python
```

Tasks:

- Introduce a small backend abstraction in Python.
- Keep current `fetch_package()` behavior as the Python backend.
- Add import handling for optional Rust backend.
- Return a consistent result shape from both backends.

Acceptance criteria:

- Existing Python crawl still works.
- Backend selection is explicit.
- Missing Rust extension gives a clear error or falls back according to chosen policy.

## Phase 4: Implement Structured Fetch Results

Goal:

```text
Represent fetch success and failure in a predictable shape.
```

Conceptual result types:

```text
FetchSuccess:
    name
    document

FetchFailure:
    name
    error_kind
    status_code
    message
    retry_after
```

Rules:

- Do not make Python parse unstructured Rust error strings.
- Preserve enough information for retry/failure policy.
- Keep Python responsible for finalizing database state.

Acceptance criteria:

- Python backend returns the same conceptual result shape as Rust.
- Tests cover success, HTTP error, timeout/network error, and JSON error mapping.

## Phase 5: Rust One-Package Fetch

Goal:

```text
Make Rust fetch and parse one npm package correctly.
```

Tasks:

- Build package URL correctly, including scoped package names.
- Send HTTP request with User-Agent.
- Parse response JSON.
- Return structured success or failure.

Acceptance criteria:

- Rust handles unscoped package names.
- Rust handles scoped package names like `@types/node`.
- Rust returns status code on HTTP failures.
- Rust returns structured JSON parse failures.

## Phase 6: Rust Batch Fetch

Goal:

```text
Expose the real hot path: fetch_many(...).
```

Target interface:

```text
fetch_many(package_names, concurrency, user_agent) -> list[result]
```

Tasks:

- Accept list of package names.
- Fetch multiple packages with bounded concurrency.
- Preserve result association with package names.
- Return structured result list.

Start conservatively:

```text
concurrency = 4 or 8
batch size = 10 or 25
```

Acceptance criteria:

- All requested package names produce exactly one result.
- One package failure does not fail the whole batch.
- Concurrency is bounded by the passed setting.
- Tests cover mixed success/failure results.

## Phase 7: Integrate Rust Backend Into Crawl Loop

Goal:

```text
Use Rust results while preserving existing queue/finalization behavior.
```

Tasks:

- Add or prepare `claim_many(conn, limit)`.
- Send claimed names to selected backend.
- For each success, call `finalize_success`.
- For each failure, call `finalize_failure`.
- Preserve attempts from claimed rows.
- Keep leases and retry behavior unchanged.

Acceptance criteria:

- Python backend path still passes existing tests.
- Rust backend can process a small bounded crawl.
- Success/failure finalization remains in Python.
- Existing DB invariants remain true.

## Phase 8: Benchmark And Document Results

Goal:

```text
Compare Python and Rust fairly.
```

Run with the same:

- machine
- database
- seed set
- package limit
- request rate
- retry settings

Compare:

- elapsed time
- packages/sec
- failures
- average fetch time
- average store/finalize time
- notes about bottlenecks

Update:

- `docs/profiling.md`
- `README.md` Milestone 4 notes
- optionally `docs/milestone-4-profile-rust-hotpath.md`

Acceptance criteria:

- Benchmark numbers are recorded.
- The write-up says whether Rust helped and where.
- If Rust did not help much, the reason is explained honestly.

## Test Plan

Python tests:

- backend selection defaults to Python
- missing Rust backend is handled clearly
- Python backend result shape is correct
- Rust result mapping feeds existing finalize functions

Rust/PyO3 tests:

- extension imports
- tiny function works
- fetch success returns document
- HTTP failure returns status code
- JSON parse failure returns structured error
- batch fetch returns one result per input

Integration tests:

- small crawl with Python backend
- small crawl with Rust backend if available
- mixed success/failure batch

Manual checks:

- run `depcrawler crawl --fetch-backend python --max-packages 10`
- run `depcrawler crawl --fetch-backend rust --max-packages 10`
- compare logs and database counts

## Risk Notes

Risk:

```text
Rust build tooling makes development harder.
```

Mitigation:

```text
Keep Python fallback working.
```

Risk:

```text
Rust speedup is hidden by network or database bottlenecks.
```

Mitigation:

```text
Measure before and after. Explain results honestly.
```

Risk:

```text
Batch fetching complicates queue leases.
```

Mitigation:

```text
Keep batch sizes small first and preserve Python finalization.
```

Risk:

```text
Error behavior changes accidentally.
```

Mitigation:

```text
Use structured results and keep retry policy in Python.
```

## Final Interview Story

The story should be:

> I started with a correct Python crawler. Before adding Rust, I profiled it and
> recorded a baseline. Then I introduced Rust behind a narrow PyO3 interface for
> the fetch/parse hot path only. Python remained responsible for the database
> queue, retries, and finalization, which kept the system easy to reason about.
> I preserved a Python fallback and compared both backends under the same crawl
> settings before claiming a performance improvement.
