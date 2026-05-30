# Profiling Notes

Milestone 4 should begin here before any Rust code is added.

## Question

Which part of the Python crawler is slow enough to justify moving code to Rust?

## Measurement checklist

Run a bounded crawl and record:

- packages stored
- failures
- elapsed seconds
- packages per second
- average versions per package
- average dependencies discovered per package

Then inspect where time goes:

- HTTP request latency
- JSON decoding
- dependency extraction
- PostgreSQL writes
- queue/status queries

## Decision rule

Only move the hot path to Rust if profiling shows Python-side fetch/parse work is
a meaningful bottleneck. Keep PostgreSQL writes in Python until there is evidence
that crossing the Python/Rust boundary for DB work is worth the complexity.

## First Rust boundary

The first Rust interface should be narrow:

```text
fetch_many(package_names, concurrency, user_agent) -> list[result]
```

Each result should contain either:

- package name and registry document, or
- package name, error class, status code if present, and message.
