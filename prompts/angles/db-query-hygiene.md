# Angle: DB query hygiene

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: catch queries that will fail at scale, or already do — unbounded selects, N+1, missing indexes, `SELECT *` on hot paths, unnecessary lock cost.

Assumes Postgres. Adapt patterns for MySQL / SQLite / cloud DB as needed.

## Look for

1. **`SELECT *` on hot paths** where only a few columns are needed. Expands network + memory. Especially bad when the row has jsonb/text columns the caller doesn't read.

2. **N+1 in a loop.** Any code path that fires a query per item in an array. Check RSC data-fetches, sitemap builders, admin dashboards, batch jobs.

3. **Missing indexes.** Fields used in `WHERE`, `ORDER BY`, or `JOIN` that don't have an obvious index. Grep migrations for `CREATE INDEX` and cross-reference against high-traffic query patterns.

4. **`LIKE '%foo%'` / ILIKE without a trigram or GIN index.** Full-table scan on every search keystroke. Especially bad on Cmd+K style admin search.

5. **`ORDER BY <col>` without index** on large tables. Sort in memory.

6. **Unbounded queries.** `SELECT ... FROM analytics` without a `LIMIT` or a date range. Analytics rows accumulate.

7. **Wrong pagination semantics.** `LIMIT N OFFSET M` with large M is O(M). Keyset pagination is O(1). Deep pagination on posts / analytics / notifications hurts.

8. **Missing transactions on multi-statement writes.** Any INSERT + UPDATE + DELETE combo that could leave a half-state on error.

9. **`SELECT COUNT(*)` on unbounded tables.** Needed for pagination UI but expensive. Consider `pg_class.reltuples` for approximate counts on huge tables where the number is cosmetic.

10. **`ORDER BY random()`.** Full sort of every row.

11. **`LEFT JOIN` where `INNER JOIN` would be correct + faster.**

12. **Cast in WHERE.** `WHERE created_at::date = '2026-01-01'` defeats an index on `created_at`. Should be a range.

13. **Idle-in-transaction risk.** `BEGIN` followed by an `await fetch(...)` before `COMMIT` — the network call inside a transaction holds locks and a pool connection.

14. **Nested queries where a JOIN would win.** `WHERE product_id IN (SELECT ...)` when a JOIN could hit an index.

15. **JSONB queries without GIN.** `WHERE metadata->>'key' = 'value'` without an appropriate GIN index.

16. **Missing NOT NULL / UNIQUE constraints** where the app assumes them.

17. **`FOR UPDATE` locks held across a network call.**

18. **Missing `updated_at` triggers.** Every write path has to remember to bump it manually — one forgotten one and downstream cache signals lie.

19. **Prepared-statement thrash.** Dynamic SQL with a variable number of placeholders defeats the pg driver's statement cache.

20. **Row-lock cost from unnecessary UPDATE.** `UPDATE ... SET foo = $1` when `WHERE foo IS DISTINCT FROM $1` would skip no-op writes.

## Files to prioritize

- Central DB helpers (`src/lib/db/`, `src/db/`, `prisma/`, wherever your queries live)
- API route handlers that touch the DB — grep `pool.query`, `client.query`, `prisma.`, `db.`
- Sitemap builders (`app/sitemap.ts`, etc.) — always fan out
- Admin dashboards (they aggregate)
- Migrations — read for `CREATE INDEX` statements and match against WHERE patterns

## Grep starters

```
SELECT \*                          # find SELECT star
pool\.query\(|client\.query\(     # find query call sites
CREATE INDEX                      # inventory existing indexes
ILIKE|LIKE '%                     # find prefix-wildcard patterns
OFFSET                            # find deep pagination
```

## What NOT to report

- "Add caching" — vague, needs a specific caller + specific TTL
- "Consider read replicas" — infra decision, not a code bug
- "This query is complex" — complexity alone isn't a bug
- Findings that would require a schema migration if the audit is meant to code-only (but flag them explicitly as "handoff item — needs migration" so the reviewer can decide)
