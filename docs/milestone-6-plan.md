# Milestone 6 Implementation Plan

This is the execution plan for Milestone 6. The companion learning document is
`docs/milestone-6-deep-analysis.md`.

## Objective

Use PostgreSQL queries to answer the project's main graph questions.

Milestone 6 is complete when:

- analysis SQL files exist
- queries answer clear questions
- recursive queries are bounded and cycle-safe
- expensive queries have `EXPLAIN` notes
- outputs are exported or summarized
- README/write-up has evidence-backed findings

## Non-Goals

Do not:

- claim full npm-wide conclusions from a tiny crawl
- run unbounded recursive queries
- add indexes without a query reason
- build a dashboard before the SQL is trustworthy
- hide limitations of the crawl sample

## Phase 1: Record Crawl Coverage

Goal:

```text
Know what dataset the analysis is based on.
```

Record:

- crawl date
- seed set
- packages stored
- versions stored
- dependency edges stored
- failed packages
- dependency types included
- max crawl/package/time limit

Suggested SQL:

```sql
SELECT count(*) FROM packages;
SELECT count(*) FROM package_versions;
SELECT count(*) FROM dependencies;
SELECT dep_type, count(*) FROM dependencies GROUP BY dep_type;
SELECT state, count(*) FROM crawl_frontier GROUP BY state;
```

Acceptance criteria:

- Coverage numbers are saved in analysis notes.
- Any final claim can reference the dataset size.

## Phase 2: Direct Dependency Queries

Goal:

```text
Find packages with the most direct incoming dependency edges.
```

Create:

```text
analysis/01_most_depended_on.sql
```

Query should return:

- package name
- direct edge count
- distinct dependent package count

Acceptance criteria:

- Query runs on crawled data.
- Output can be limited to top 100.
- Query comments explain edge count vs distinct dependents.

## Phase 3: Dependency Type Shape

Goal:

```text
Understand runtime/dev/peer/optional edge distribution.
```

Create:

```text
analysis/02_dependency_type_shape.sql
```

Query should return:

- dependency type
- edge count
- distinct dependent packages
- distinct dependency packages

Acceptance criteria:

- Query makes it clear whether dev dependencies dominate the data.
- Results inform later filters.

## Phase 4: Depth From Seed Packages

Goal:

```text
Measure how deep dependency paths go from known seeds.
```

Create:

```text
analysis/03_seed_depth.sql
```

Query requirements:

- use `WITH RECURSIVE`
- start from a small seed list
- track depth
- track path for cycle protection
- use a max depth limit

Acceptance criteria:

- Query cannot recurse forever.
- Output includes package name, shallowest depth, and path count or sample path.
- Seed list is easy to edit.

## Phase 5: Transitive Reach

Goal:

```text
Estimate which packages are reached transitively by many upstream packages.
```

Create:

```text
analysis/04_transitive_reach.sql
```

Possible output:

- package name
- number of distinct roots that reach it
- number of paths seen
- shallowest depth

Acceptance criteria:

- Query uses bounded recursion.
- Query states whether it counts paths or distinct roots.
- Query is tested on a small known graph or manually validated.

## Phase 6: Load-Bearing Candidates

Goal:

```text
Combine signals into a candidate list.
```

Create:

```text
analysis/05_load_bearing_candidates.sql
```

Possible signals:

- direct dependent package count
- transitive reach
- runtime dependency count
- appears across multiple seed roots

Acceptance criteria:

- Query comments define "load-bearing candidate."
- Output is not presented as absolute truth.
- Limitations are documented.

## Phase 7: Query Plan Notes

Goal:

```text
Understand expensive query behavior.
```

Create:

```text
analysis/06_explain_notes.md
```

For expensive queries, record:

- query name
- dataset size
- whether indexes are used
- slowest operation
- any suggested index
- before/after timing if index is added

Useful command:

```sql
EXPLAIN ANALYZE
SELECT ...
```

Acceptance criteria:

- At least one recursive query has an `EXPLAIN` note.
- Index recommendations are tied to actual query plans.

## Phase 8: Export Results

Goal:

```text
Create small tables for the final write-up.
```

Suggested exports:

```text
analysis/results/top_direct_dependencies.csv
analysis/results/dependency_type_shape.csv
analysis/results/load_bearing_candidates.csv
```

Keep exports small:

```text
top 20 or top 100 rows
```

Acceptance criteria:

- Exported results are reproducible from SQL files.
- README can summarize findings without requiring live DB access.

## Phase 9: Write Findings

Goal:

```text
Turn query outputs into a clear story.
```

Write:

- what dataset was analyzed
- top findings
- surprising results
- limitations
- what would improve the analysis

Acceptance criteria:

- Findings are evidence-backed.
- Claims mention crawl size and scope.
- Limitations are honest.

## Test And Validation Plan

Manual validation:

- create a tiny known graph in a test database
- run recursive depth query
- confirm expected depth/path counts

DB tests when useful:

- insert small fixture graph
- assert direct dependency counts
- assert depth results
- assert type shape counts

Non-DB checks:

- SQL files are documented with comments
- queries use `LIMIT` where appropriate
- recursive queries have max depth
- recursive queries include cycle protection

## Risk Notes

Risk:

```text
Recursive queries become too expensive.
```

Mitigation:

```text
Use max depth, seed limits, and EXPLAIN before scaling.
```

Risk:

```text
Results overclaim beyond crawl coverage.
```

Mitigation:

```text
Record dataset size and seed set with every finding.
```

Risk:

```text
Version-level edges distort package-level conclusions.
```

Mitigation:

```text
Report both edge counts and distinct dependent package counts.
```

Risk:

```text
Dev dependencies dominate results.
```

Mitigation:

```text
Analyze dependency types separately and include runtime-only views.
```

## Final Interview Story

The story should be:

> After collecting dependency data, I treated the dependency table as a graph. I
> wrote SQL queries for direct popularity, dependency type distribution,
> recursive depth from seed packages, transitive reach, and load-bearing
> candidates. For recursive queries I used depth limits and cycle protection,
> then inspected expensive plans with `EXPLAIN`. I also documented dataset
> coverage so the analysis did not overclaim beyond what was crawled.
