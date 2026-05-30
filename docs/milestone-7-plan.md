# Milestone 7 Implementation Plan

This is the execution plan for Milestone 7. The companion learning document is
`docs/milestone-7-final-writeup.md`.

## Objective

Turn the repository into a polished technical case study.

Milestone 7 is complete when:

- README clearly explains the project
- architecture and data model are understandable
- milestone results are summarized honestly
- demo commands are included
- tests and setup are documented
- analysis findings are included if available
- limitations and next steps are clear
- MixRank relevance is technical and natural

## Non-Goals

Do not:

- exaggerate scale or results
- claim Rust speedups without measurements
- claim full npm ecosystem conclusions from partial crawls
- turn the README into a cover letter
- bury setup instructions under too much prose
- include huge query outputs inline

## Phase 1: Gather Evidence

Goal:

```text
Collect the facts the README will cite.
```

Gather:

- current milestone status
- test results
- schema overview
- crawl command examples
- status command output example
- profiling numbers if Milestone 4 is complete
- analysis numbers if Milestone 6 is complete
- known limitations

Commands:

```powershell
.\myvenv\Scripts\python.exe -m pytest
.\myvenv\Scripts\python.exe -m crawler.cli crawl --help
.\myvenv\Scripts\python.exe -m crawler.cli status --help
```

Acceptance criteria:

- README claims can be backed by current repo facts.
- Missing future milestone results are clearly marked as planned or pending.

## Phase 2: Rewrite README Opening

Goal:

```text
Make the first 30 seconds strong.
```

Opening should include:

- npm dependency graph crawler
- resumable PostgreSQL queue
- package/version/dependency storage
- crawler politeness
- Rust hot path if implemented or planned
- SQL analysis goal

Acceptance criteria:

- A reviewer can explain the project after reading the first paragraph.
- The opening does not sound generic.

## Phase 3: Add Architecture Section

Goal:

```text
Explain how the system fits together.
```

Include:

- Python orchestration
- PostgreSQL queue and datastore
- Rust hot path boundary
- SQL analysis

Recommended diagram:

```text
CLI -> Python crawl loop -> PostgreSQL queue
                    |
                    v
              fetch backend
                    |
                    v
             package documents
                    |
                    v
        packages / versions / dependencies
                    |
                    v
              analysis SQL
```

Acceptance criteria:

- Architecture can be understood without reading code first.
- Ownership boundaries are clear.

## Phase 4: Explain The Data Model

Goal:

```text
Make the schema understandable.
```

Explain:

- `packages`
- `package_versions`
- `dependencies`
- `raw_documents`
- `crawl_frontier`

Acceptance criteria:

- Reader understands that `dependencies` is the graph.
- Reader understands why raw JSON is preserved.
- Reader understands why frontier state is in Postgres.

## Phase 5: Explain The Crawl Loop

Goal:

```text
Show the systems design.
```

Include:

- state machine
- atomic claiming
- leases
- success finalization
- failure finalization
- discovered dependency enqueueing

Acceptance criteria:

- `pending`, `in_progress`, `done`, and `failed` are explained.
- `FOR UPDATE SKIP LOCKED` is mentioned with purpose.
- Crash recovery is explained simply.

## Phase 6: Explain Politeness And Operations

Goal:

```text
Show production-minded crawler behavior.
```

Include:

- shared HTTP client
- rate limiter
- Retry-After handling
- bounded crawl options
- status command
- runbook link

Acceptance criteria:

- Reader sees that the crawler is controlled and polite.
- Demo commands are safe and bounded.

## Phase 7: Add Profiling/Rust Section

Goal:

```text
Explain Rust as a measured optimization.
```

If Rust is complete, include:

- baseline
- Rust backend design
- benchmark comparison
- result

If Rust is not complete, include:

- planned hot path
- why profiling comes first
- narrow boundary
- Python fallback

Acceptance criteria:

- No unsupported speedup claims.
- Rust is tied to a concrete problem.

## Phase 8: Add Analysis Results

Goal:

```text
Show what the crawl taught you.
```

Include if available:

- dataset size
- top direct dependencies
- dependency type distribution
- depth findings
- load-bearing candidates

If not available yet:

- describe planned analysis
- link to analysis SQL/docs
- avoid claiming results

Acceptance criteria:

- Results include dataset scope.
- Limitations are visible.

## Phase 9: Add Setup And Demo Commands

Goal:

```text
Make the project reproducible at small scale.
```

Include:

```powershell
depcrawler seed
depcrawler crawl --max-packages 25 --requests-per-second 2
depcrawler status
```

Mention:

- `DATABASE_URL`
- `TEST_DATABASE_URL`
- applying `sql/schema.sql`
- DB-backed tests skip without test DB

Acceptance criteria:

- A reader can run a small demo.
- Commands are bounded by default in docs.

## Phase 10: Add Learning, Limitations, And Next Steps

Goal:

```text
Show judgment and growth.
```

Learning section:

- durable queues
- idempotent writes
- crawler politeness
- profiling before optimization
- SQL graph analysis

Limitations:

- single-machine crawl
- partial crawl coverage unless full run completed
- npm registry politeness constraints
- analysis depends on seed/crawl size

Next steps:

- distributed workers
- better metrics
- Rust batching
- richer analysis
- table size/index review

Acceptance criteria:

- Limitations are honest.
- Next steps follow naturally from the project.

## Phase 11: Final Review

Goal:

```text
Make sure the final repo reads professionally.
```

Checklist:

- README has no stale milestone claims
- docs links work
- commands match actual CLI
- test command is current
- future work is not presented as complete
- MixRank relevance is technical, not salesy
- no giant unstructured walls of text

Acceptance criteria:

- A reviewer can understand the project quickly.
- A technical interviewer can drill into details from docs/code.

## Test Plan

Run:

```powershell
.\myvenv\Scripts\python.exe -m pytest
```

Also check:

```powershell
.\myvenv\Scripts\python.exe -m crawler.cli crawl --help
.\myvenv\Scripts\python.exe -m crawler.cli status --help
```

Manual documentation checks:

- README links to important docs
- milestone docs exist
- setup commands are accurate
- no unimplemented feature is marked complete

## Risk Notes

Risk:

```text
README overclaims unfinished milestones.
```

Mitigation:

```text
Separate completed work from planned work.
```

Risk:

```text
README becomes too long.
```

Mitigation:

```text
Keep README high-level and link to docs.
```

Risk:

```text
MixRank relevance sounds like a cover letter.
```

Mitigation:

```text
Focus on technical overlap: crawling, databases, Rust, SQL, data systems.
```

Risk:

```text
Analysis results are misinterpreted.
```

Mitigation:

```text
Always include dataset size, seed set, and limitations.
```

## Final Interview Story

The story should be:

> I wrote the final README as a case study. It explains the crawler's motivation,
> the PostgreSQL schema, the durable queue, lease-based crash recovery,
> politeness controls, profiling/Rust plan, operational runbook, and graph
> analysis. I kept the claims evidence-based and documented limitations, so the
> project shows not just that I can write code, but that I can reason about a
> data system end to end.
