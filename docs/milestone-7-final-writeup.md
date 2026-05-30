# Milestone 7: Final Write-Up

This document is for revision and interview preparation. Milestone 7 turns the
project into a clear portfolio case study.

The goal is not just to say:

```text
I built a crawler.
```

The goal is to show:

```text
I can design, build, operate, measure, optimize, and explain a real data system.
```

## Goal

Milestone 7 should make the repository easy to understand for:

- a recruiter scanning quickly
- an engineer reading the architecture
- an interviewer asking design questions
- your future self revising the concepts

The final write-up should explain:

- problem
- motivation
- architecture
- schema
- queue state machine
- crawler politeness
- operational readiness
- profiling
- Rust hot path
- analysis results
- tradeoffs and limitations
- what you would do next

In one sentence:

> Milestone 7 turns the project from a collection of code into a polished
> engineering story backed by implementation details and data.

## Why This Milestone Matters

A strong project is not only built. It is explained.

Interviewers often cannot read every line of code. The README and docs must
quickly answer:

```text
What did you build?
Why did you build it?
What technical problems did you solve?
What tradeoffs did you make?
What did you learn?
Why is this relevant to the role?
```

For MixRank, this matters because the role values curiosity, systems thinking,
crawling, Python, PostgreSQL, Rust, Linux, and data analysis.

Milestone 7 connects those signals into one coherent narrative.

Interview phrase:

> I wrote the final README as a technical case study, not just setup
> instructions, so a reviewer can understand the system design, tradeoffs, and
> results without reverse-engineering the repository.

## Target Reader

Write for a technical reviewer who has limited time.

They should understand the project in layers:

```text
30 seconds: what this is and why it is interesting
3 minutes: architecture and milestones
15 minutes: implementation details and tradeoffs
30+ minutes: code, queries, tests, and results
```

The README should support all three reading speeds.

## Final README Structure

Recommended structure:

```text
# npm Dependency Graph Crawler

## One-paragraph summary
## Why this project exists
## What it does
## Architecture
## Data model
## Crawl loop and queue state machine
## Politeness and failure handling
## Profiling and Rust hot path
## Operating the crawler
## Analysis results
## Tests
## What I learned
## What I would improve next
## Reproducing a small demo
```

Keep the README readable. Put deeper explanations in `docs/`.

## Concept 1: The Opening Summary

The opening should be concrete.

Good:

```text
A resumable crawler for the npm registry dependency graph. It stores package,
version, dependency, and raw registry data in PostgreSQL, uses the database as a
durable work queue, rate-limits registry requests, and analyzes the resulting
graph to find dependency depth and load-bearing packages.
```

Weak:

```text
This is a Python project that crawls npm.
```

The summary should name the core technical ideas:

- resumable crawler
- npm dependency graph
- PostgreSQL work queue
- rate limiting
- Rust hot path
- graph analysis

## Concept 2: Technical Story Arc

The final write-up should tell a progression:

```text
Milestone 1: prove one package can be fetched and stored
Milestone 2: make the crawler resumable
Milestone 3: add politeness and throughput controls
Milestone 4: profile before porting hot path to Rust
Milestone 5: make longer crawls operable
Milestone 6: analyze the graph
Milestone 7: write the case study
```

This shows engineering maturity:

```text
correctness before scale
measurement before optimization
operations before huge crawls
analysis before conclusions
```

## Concept 3: Architecture Explanation

The architecture section should explain ownership clearly:

```text
Python owns orchestration.
PostgreSQL owns durable state.
Rust owns the measured hot path.
SQL owns analysis.
```

Recommended diagram:

```text
CLI
 |
 v
Python crawl loop
 |
 +--> PostgreSQL crawl_frontier queue
 |
 +--> fetch backend
        |
        +--> Python httpx fetch
        |
        +--> Rust fetch_many hot path
 |
 v
PostgreSQL packages / versions / dependencies / raw_documents
 |
 v
SQL analysis queries
```

Keep diagrams text-based unless an image truly helps.

## Concept 4: Schema Explanation

The write-up should explain the schema in human terms:

```text
packages: one row per npm package
package_versions: one row per published version
dependencies: graph edges
raw_documents: full registry JSON for re-parsing
crawl_frontier: durable work queue
```

Important phrase:

```text
The dependencies table is the graph.
```

Explain why raw JSON is stored:

```text
It lets me re-parse data later without re-fetching from npm.
```

Explain why `crawl_frontier` is in PostgreSQL:

```text
It makes crawl progress durable and restart-safe.
```

## Concept 5: Queue State Machine

Explain states:

```text
pending -> in_progress -> done
pending -> in_progress -> pending
pending -> in_progress -> failed
```

Explain leases:

```text
If a worker crashes while a package is in_progress, the lease eventually
expires and another worker can reclaim it.
```

Explain atomic claiming:

```text
SELECT ... FOR UPDATE SKIP LOCKED prevents multiple workers from claiming the
same package at the same time.
```

This is one of the strongest systems parts of the project. Do not hide it.

## Concept 6: Politeness And Failure Handling

Explain:

- shared HTTP client
- request rate limit
- Retry-After support
- max attempts
- retriable vs fatal errors
- useful crawl logs

Good summary:

```text
The crawler reuses one HTTP client for connection pooling, waits before each
request according to a configurable rate limit, and honors Retry-After headers
on retriable HTTP responses. Failures are classified so temporary errors can be
retried while permanent failures are marked failed.
```

## Concept 7: Profiling And Rust

This section must be honest.

Rust should be described as:

```text
a response to a measured hot path
```

not:

```text
I rewrote it in Rust because Rust is fast.
```

Explain:

- Python baseline
- measured bottleneck
- narrow Rust boundary
- Python fallback
- benchmark result

If Rust has not been implemented yet, write:

```text
Planned Milestone 4 work
```

Do not pretend it is done.

## Concept 8: Analysis Results

The final write-up should include concrete findings.

Examples:

```text
Top 10 directly depended-on packages
Dependency type distribution
Deepest paths from seed set
Load-bearing package candidates
```

Always include dataset scope:

```text
This analysis is based on N packages, M versions, and K dependency edges
crawled from this seed set.
```

This avoids overclaiming.

## Concept 9: MixRank Relevance

Do not write a generic cover-letter section.

Instead, connect technical overlap naturally:

```text
This project touches the same kinds of problems I would be excited to work on
at a web-scale data company: crawling, durable queues, PostgreSQL-backed data
pipelines, performance measurement, Rust hot paths, and SQL analysis over large
datasets.
```

Keep it short and technical.

Avoid:

```text
I am the perfect candidate for MixRank.
```

Prefer:

```text
I built this because I wanted hands-on practice with the kind of crawling,
storage, and analysis problems that appear in web-scale data systems.
```

## Concept 10: Evidence-Based Claims

Every impressive claim should have evidence.

Instead of:

```text
The crawler is scalable.
```

Say:

```text
The queue is stored in PostgreSQL and claims work atomically with
FOR UPDATE SKIP LOCKED, so the design can support multiple workers even though
the current implementation runs on one machine.
```

Instead of:

```text
Rust made it faster.
```

Say:

```text
On a bounded crawl of N packages with the same request rate, the Rust backend
processed X packages/sec compared to Y packages/sec for Python.
```

If no benchmark exists yet, do not claim a speedup.

## Concept 11: What I Learned

This section is valuable for junior roles.

Possible points:

- how to model messy JSON as relational tables
- why durable queues matter
- how leases recover abandoned work
- how `FOR UPDATE SKIP LOCKED` enables safe claiming
- why crawler politeness matters
- why profiling should come before Rust
- how SQL can analyze graph-shaped data

This helps show growth and curiosity.

## Concept 12: What I Would Improve Next

Good future work:

- distributed multi-worker crawl
- better failure reason normalization
- better metrics and dashboards
- more careful package popularity weighting
- npm change feed support
- deeper graph algorithms
- smarter Rust/Python batching
- table partitioning or archive strategy for raw documents

This shows judgment.

Do not list random features. List improvements that follow from the project.

## Final README Checklist

Before marking Milestone 7 complete:

- README opens with a strong concrete summary
- architecture is understandable in 3 minutes
- schema is explained
- queue state machine is explained
- politeness/failure behavior is explained
- Rust section is honest and evidence-based
- analysis findings include dataset scope
- commands reproduce a small demo
- tests are documented
- limitations are stated
- MixRank relevance is technical and natural

## Design Patterns In This Milestone

### Technical storytelling

Explain the system as a sequence of design decisions.

### Evidence-based writing

Claims should be backed by code, tests, metrics, or SQL results.

### Progressive disclosure

README gives the big picture; docs contain deeper explanations.

### Honest limitations

State what the project proves and what it does not prove.

### Role alignment

Connect the project to crawling, data systems, databases, Rust, and analysis
without sounding like marketing.

## Interview Answer

If asked what the final project demonstrates, say:

> This project demonstrates an end-to-end data systems workflow. I started with
> a single-package fetcher, designed a PostgreSQL schema for package/version
> dependency data, turned the frontier table into a durable queue, added
> lease-based crash recovery, made the crawler polite with rate limiting and
> Retry-After handling, planned a measured Rust hot path, and used SQL to analyze
> the resulting dependency graph. The final write-up explains the architecture,
> tradeoffs, results, and limitations like a small production case study.

## What To Review Before Writing

Make sure you can explain:

- the project in one sentence
- why the project exists
- how data flows through the crawler
- what each database table stores
- why the queue is durable
- how crash recovery works
- how politeness works
- what Rust is supposed to solve
- what analysis questions are answered
- what limitations remain
