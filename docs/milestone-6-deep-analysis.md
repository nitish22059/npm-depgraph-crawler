# Milestone 6: Deep Analysis

This document is for revision and interview preparation. Milestone 6 turns the
crawl output into answers.

The project began with a question:

```text
How deep does the npm dependency graph go, and which packages are load-bearing
nodes that much of the ecosystem depends on?
```

Milestone 6 is where the project stops being only a crawler and becomes a data
analysis project.

## Goal

Milestone 6 should produce SQL-backed findings from the crawled npm dependency
graph.

The system should:

- analyze direct dependency popularity
- estimate transitive reach
- measure dependency depth from seed packages
- identify load-bearing packages
- compare runtime/dev/peer/optional dependency shapes
- use `EXPLAIN` to understand expensive queries
- export small result tables for the final write-up

In one sentence:

> Milestone 6 uses PostgreSQL and graph-style SQL queries to explain the shape
> of the npm dependency graph.

## Why This Milestone Matters

The crawler collects data, but collected data is not useful by itself.

Milestone 6 answers:

```text
What did we learn?
```

Without analysis, the project proves only that packages can be fetched.

With analysis, the project shows:

- database skill
- graph thinking
- SQL query design
- ability to turn raw data into insight
- ability to reason about query performance

This is especially relevant for MixRank because their work is about large-scale
data collection and turning crawled data into useful products.

Interview phrase:

> I did not stop at crawling. I used the stored graph to ask analytical
> questions about direct popularity, transitive reach, dependency depth, and
> load-bearing packages, then inspected query plans for the expensive parts.

## Data Model For Analysis

The important tables are:

```text
packages
package_versions
dependencies
crawl_frontier
```

The graph lives mostly in:

```text
dependencies
```

Each dependency row means:

```text
dependent_name@dependent_version depends on dependency_name
```

Example:

```text
react@18.2.0 -> loose-envify
```

Graph vocabulary:

```text
node = package
edge = dependency relationship
incoming edges = packages depending on this package
outgoing edges = packages this package depends on
```

## Concept 1: Direct Dependency Popularity

Question:

```text
Which packages are directly depended on the most?
```

Basic SQL idea:

```sql
SELECT dependency_name, count(*)
FROM dependencies
GROUP BY dependency_name
ORDER BY count(*) DESC;
```

This counts dependency edges.

But edge count and package count are different:

```text
direct_edges = total dependency rows pointing at a package
dependent_packages = unique packages that depend on it
```

Why both matter:

- one package with many versions can create many edges
- unique dependent packages may better represent ecosystem reach

Interview phrase:

> I separated edge count from distinct dependent package count so one package
> with many versions would not completely distort the popularity result.

## Concept 2: Dependency Types

npm dependencies are not all the same.

The schema stores:

```text
runtime
dev
peer
optional
```

Questions:

- Are most edges runtime dependencies?
- Which packages dominate dev dependencies?
- Are peer dependencies structurally different?
- Should analysis include dev dependencies or only runtime dependencies?

Why this matters:

```text
A runtime dependency affects installed production code differently than a dev
tooling dependency.
```

Milestone 6 should report results both:

- across all dependency types
- filtered to runtime dependencies where useful

## Concept 3: Transitive Dependencies

A direct dependency is one step:

```text
A -> B
```

A transitive dependency is reached through a path:

```text
A -> B -> C -> D
```

In this example:

```text
A transitively depends on C and D
```

Question:

```text
Which packages are reached by many dependency paths?
```

This helps identify load-bearing packages.

## Concept 4: Recursive CTEs

PostgreSQL supports recursive queries:

```sql
WITH RECURSIVE ...
```

Recursive CTEs let SQL walk graph paths.

Mental model:

```text
start with seed rows
repeatedly join to dependencies
stop at a max depth or when no new rows appear
```

Why use a max depth?

- prevents runaway queries
- avoids infinite loops in cyclic graphs
- makes analysis bounded and explainable

Example max depth:

```text
12
```

## Concept 5: Cycles

Graphs can contain cycles.

Example:

```text
A -> B -> C -> A
```

If a recursive query does not guard against cycles, it can loop forever or
explode in size.

A common guard is path tracking:

```text
Do not visit a package already in the current path.
```

Conceptually:

```sql
AND NOT d.dependency_name = ANY(path)
```

This keeps one path from revisiting the same package.

## Concept 6: Dependency Depth

Question:

```text
How deep does the dependency graph go from my seed packages?
```

Depth means number of dependency steps.

Example:

```text
react -> loose-envify
```

Depth:

```text
1
```

Longer path:

```text
A -> B -> C -> D
```

Depth from A to D:

```text
3
```

Milestone 6 should compute:

- deepest path seen
- packages at each depth
- packages reached at surprisingly deep levels

## Concept 7: Load-Bearing Packages

A load-bearing package is a package whose failure or compromise could affect a
large part of the ecosystem.

Possible signals:

- many direct dependents
- high transitive reach
- appears in many dependency paths
- appears across many dependency types
- depended on by popular seed packages

No single metric is perfect.

Milestone 6 should explain the metric used and its limitations.

Interview phrase:

> I treated "load-bearing" as an operational definition, not a magic label. I
> looked at direct dependents, transitive reach, and path frequency, then stated
> the limitations of each metric.

## Concept 8: Query Planning With `EXPLAIN`

Large SQL queries can be expensive.

PostgreSQL can explain a query plan:

```sql
EXPLAIN ANALYZE
SELECT ...
```

This shows:

- whether indexes are used
- where time is spent
- row estimates vs actual rows
- joins and scans

Milestone 6 should include `EXPLAIN` notes for expensive queries.

Important index:

```text
dependencies_by_target ON dependencies (dependency_name)
```

This helps reverse lookups:

```text
Who depends on X?
```

## Concept 9: Result Exports

The final write-up should not depend on live database access.

Milestone 6 should export small result tables:

```text
top_direct_dependencies.csv
dependency_type_shape.csv
deepest_paths.csv
load_bearing_candidates.csv
```

These exports can be summarized in the README.

Keep exports small and meaningful:

```text
top 20 or top 100 rows
```

## Concept 10: Honest Limitations

Analysis depends on crawl coverage.

If the crawl only starts from a seed set, the result is:

```text
dependency graph reachable from that seed set
```

not:

```text
the entire npm ecosystem
```

The write-up should say:

- how many packages were crawled
- how many versions were stored
- how many dependency edges were stored
- what seed set was used
- whether dev dependencies were included
- max recursive depth used

## Analysis Queries To Build

Milestone 6 should create SQL files for:

```text
analysis/01_most_depended_on.sql
analysis/02_dependency_type_shape.sql
analysis/03_seed_depth.sql
analysis/04_transitive_reach.sql
analysis/05_load_bearing_candidates.sql
analysis/06_explain_notes.md
```

Each SQL file should start with a short comment explaining:

- question answered
- tables used
- important assumptions
- output meaning

## Tests And Validation

Analysis SQL should be validated against a small known graph.

Example test graph:

```text
A -> B
B -> C
A -> D
D -> C
```

Expected facts:

- C has two incoming paths from A
- max depth from A is 2
- B and D are direct dependencies of A
- C has multiple upstream dependents

Validation prevents writing impressive-looking SQL that gives wrong answers.

## Design Patterns In This Milestone

### Data-to-insight pipeline

Crawler output becomes analytical findings.

### Graph thinking

Packages are nodes and dependencies are edges.

### Bounded recursion

Recursive queries need max depth and cycle protection.

### Metric clarity

Define what each metric means and what it does not mean.

### Query-plan awareness

Use `EXPLAIN` for expensive queries instead of guessing.

### Reproducible exports

Save small outputs for the final write-up.

## Interview Answer

If asked what Milestone 6 is about, say:

> Milestone 6 is where I turn the crawl into analysis. The dependency table is a
> graph, so I use SQL to find direct dependency popularity, dependency type
> distribution, depth from seed packages, transitive reach, and load-bearing
> package candidates. For recursive graph queries, I use bounded recursive CTEs
> with cycle protection and inspect expensive queries with `EXPLAIN`. I also
> document crawl coverage so the results are interpreted honestly.

## What To Review Before Implementing

Make sure you can explain:

- node vs edge
- direct vs transitive dependency
- incoming vs outgoing edges
- why dependency type matters
- how recursive CTEs walk a graph
- why cycle protection is needed
- what `EXPLAIN ANALYZE` tells you
- why analysis results depend on crawl coverage
- how to define "load-bearing" honestly
