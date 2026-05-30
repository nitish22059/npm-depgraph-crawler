# Milestone 1: Schema + Single Fetch

This document is for revision and interview preparation. It explains the first
working slice of the project: fetch one npm package, normalize its data, and
store it in PostgreSQL.

## Goal

Milestone 1 proves the crawler can handle one package end to end.

The system should:

- fetch one package document from the npm registry
- parse the JSON response into a Python dictionary
- normalize messy real-world fields
- store package, version, dependency, and raw JSON data in PostgreSQL
- seed discovered dependency names into the crawl frontier

In one sentence:

> Milestone 1 turns one npm registry document into relational database rows that
> can later be crawled and analyzed.

## Architecture

The main files are:

- `crawler/cli.py`: command-line entrypoint
- `crawler/fetch.py`: HTTP request logic
- `crawler/store.py`: JSON normalization and database writes
- `crawler/db.py`: PostgreSQL connection helper
- `sql/schema.sql`: database schema
- `tests/test_store.py`: storage behavior tests

The single-fetch data flow is:

```text
depcrawler fetch-one react
   |
   v
cli.py parses command
   |
   v
fetch.fetch_package("react")
   |
   v
npm registry JSON document
   |
   v
db.connect()
   |
   v
store.store_package(conn, doc)
   |
   v
PostgreSQL rows + discovered dependency names
```

## Concept 1: CLI As The Front Door

The command is:

```powershell
depcrawler fetch-one react
```

In `crawler/cli.py`, this branch handles it:

```python
if args.cmd == "fetch-one":
    doc = fetch.fetch_package(args.name)
    with db.connect() as conn:
        discovered = store.store_package(conn, doc)
```

Plain English:

```text
Fetch the package document.
Open a database connection.
Store the document.
Print a summary.
```

This is a clean entrypoint because `cli.py` does not know SQL details and does
not know npm parsing details. It coordinates other modules.

## Concept 2: Fetching From npm

`crawler/fetch.py` builds registry URLs and sends HTTP requests.

For an unscoped package:

```text
react -> https://registry.npmjs.org/react
```

For a scoped package:

```text
@types/node -> https://registry.npmjs.org/@types%2Fnode
```

The slash in `@types/node` must be URL-encoded as `%2F`. That is why the helper
uses:

```python
quote(name, safe="@")
```

The fetch function:

```python
fetch_package(name)
```

returns a Python `dict` created from the npm JSON response.

Interview phrase:

> I isolated npm URL construction in a helper so scoped package names are
> encoded correctly in one place.

## Concept 3: PostgreSQL Schema

The database schema stores both normalized rows and the raw registry document.

Important tables:

```text
packages
package_versions
dependencies
raw_documents
crawl_frontier
```

### `packages`

One row per package name.

Examples:

```text
react
lodash
@types/node
```

Stores package-level metadata:

- description
- homepage
- license
- repository URL
- latest version
- first/last publish time
- fetched time

### `package_versions`

One row per published version.

Example:

```text
react@18.2.0
react@19.0.0
```

The primary key is:

```text
(name, version)
```

### `dependencies`

One row per dependency edge.

Example:

```text
react@18.2.0 -> loose-envify
```

The table stores:

- dependent package name
- dependent version
- dependency package name
- dependency version range
- dependency type

Dependency type is normalized into:

```text
runtime
dev
peer
optional
```

This table is the graph.

```text
packages = nodes
dependencies = edges
```

### `raw_documents`

Stores the full npm JSON document as `JSONB`.

Why keep raw JSON?

- The parser can improve later without refetching data.
- New fields can be extracted later.
- Raw data helps debug mistakes in normalization.

### `crawl_frontier`

This table becomes more important in Milestone 2. In Milestone 1,
`fetch-one` marks the fetched package as `done` and inserts discovered
dependencies as `pending`.

## Concept 4: Normalization

npm registry data is real-world data. It can be messy.

Normalization means:

```text
Convert messy input into predictable database values.
```

Examples from `crawler/store.py`:

```python
_normalize_homepage(value)
_normalize_license(value)
_normalize_repository(value)
_parse_ts(value)
```

Homepage is simple:

```python
return value if isinstance(value, str) else None
```

Plain English:

```text
If homepage is a string, store it.
Otherwise store NULL.
```

License is messier. npm may return:

```python
"MIT"
```

or:

```python
{"type": "MIT"}
```

So the code accepts both shapes.

Interview phrase:

> I normalized optional npm fields at the boundary before writing to Postgres,
> because registry metadata is not always shaped consistently.

## Concept 5: Validation And Guard Clauses

The package name is required:

```python
name = doc.get("name")
if not isinstance(name, str):
    raise ValueError("registry document is missing a string 'name' field")
```

This is validation.

It means:

```text
If the document has no string package name, stop immediately.
```

This is also a guard clause:

```text
handle the bad case early, then keep the normal path readable
```

Why be strict about `name`?

Because every database row depends on the package name. Without it, the package
cannot be stored correctly.

Why be more tolerant with nested version/dependency data?

Because one weird nested field should not always destroy the whole package.

## Concept 6: Idempotent Writes

The storage code uses:

```sql
ON CONFLICT ... DO UPDATE
```

or:

```sql
ON CONFLICT DO NOTHING
```

This makes writes idempotent.

Idempotent means:

```text
Running the same operation multiple times does not create duplicate or broken data.
```

Why this matters:

- crawlers get restarted
- packages may be fetched again
- dependencies may be discovered from many packages
- retries should not corrupt the database

Example:

```sql
INSERT INTO crawl_frontier (name)
VALUES (%s)
ON CONFLICT DO NOTHING
```

Plain English:

```text
Insert this package into the frontier if it is not already there.
If it already exists, do nothing.
```

Interview phrase:

> I used Postgres conflict handling to make storage idempotent, which is
> important for resumable crawlers and retries.

## Concept 7: Dependency Extraction

The code maps npm dependency fields into internal dependency types:

```python
DEP_FIELDS = {
    "dependencies": "runtime",
    "devDependencies": "dev",
    "peerDependencies": "peer",
    "optionalDependencies": "optional",
}
```

The nested loop is:

```text
for each version:
    for each dependency field:
        for each dependency:
            insert dependency edge
```

Example input:

```python
"dependencies": {
    "left-pad": "^1.3.0",
    "debug": "^4.0.0"
}
```

Creates edges:

```text
current-package@version -> left-pad
current-package@version -> debug
```

The function returns:

```python
set[str]
```

That set contains unique dependency package names discovered in the document.

## Concept 8: Transactions

Storage code uses:

```python
with conn.transaction(), conn.cursor() as cur:
```

Transaction means:

```text
Group database writes together.
If something fails, Postgres can roll the group back.
```

This protects the database from half-written package data.

## Tests

Milestone 1 storage tests live in `tests/test_store.py`.

They verify:

- `store_package_data()` stores package data but does not touch the frontier
- `store_package()` preserves the original single-fetch behavior
- discovered dependencies are inserted into `crawl_frontier` as pending

Run storage tests:

```powershell
.\myvenv\Scripts\python.exe -m pytest tests\test_store.py -q
```

These tests require `TEST_DATABASE_URL`.

## Design Patterns In This Milestone

### Separation of concerns

`fetch.py` handles HTTP.

`store.py` handles database writes.

`cli.py` coordinates the command.

### Data normalization

Messy npm JSON is converted into predictable database values.

### Guard clauses

Invalid required input is rejected early.

### Idempotency

`ON CONFLICT` makes repeated writes safe.

### Raw data preservation

The full JSON document is stored so future parsing changes do not require
refetching.

## Interview Answer

If asked what Milestone 1 did, say:

> I built the first end-to-end slice of the crawler. The CLI can fetch one npm
> package, parse its registry JSON, normalize messy metadata fields, and store
> package, version, dependency, and raw document rows in PostgreSQL. I also made
> writes idempotent using `ON CONFLICT`, so repeated fetches and retries do not
> create duplicate data. This gave the project a reliable data model before I
> moved on to the resumable crawl loop.

## What To Review Before Moving On

Make sure you can explain:

- how `fetch-one` flows through `cli.py`, `fetch.py`, and `store.py`
- why scoped npm package names need URL encoding
- what each database table represents
- why raw JSON is stored
- what normalization means
- why package `name` is validated strictly
- what idempotent writes are
- how dependency rows form a graph
