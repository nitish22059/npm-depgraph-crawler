# Crawl Runbook

This is the beginner-friendly operating checklist for a small, safe crawl.

## 1. Start with the database

Create a PostgreSQL database, set `DATABASE_URL`, then apply `sql/schema.sql`.

Use a separate `TEST_DATABASE_URL` for tests. The test fixture truncates tables.

## 2. Seed the frontier

```powershell
depcrawler seed
```

Or provide a small package list:

```powershell
depcrawler seed --from-file seeds.txt
```

## 3. Run a bounded crawl

Start small. This proves the loop, storage, retries, and logs work together.

```powershell
depcrawler crawl --max-packages 25 --requests-per-second 2
```

For a time-boxed run:

```powershell
depcrawler crawl --max-seconds 300 --requests-per-second 2
```

## 4. Inspect progress

```powershell
depcrawler status
```

Look for:

- `pending`: work still waiting.
- `in_progress`: currently leased work.
- `done`: packages fetched and stored.
- `failed`: packages that exhausted retries or hit fatal errors.

## 5. Stop and resume

Press `Ctrl+C` to stop. Claimed rows have leases, so abandoned work can be
reclaimed after the lease expires.
