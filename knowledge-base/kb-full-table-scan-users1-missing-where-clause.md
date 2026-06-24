# KB: Full Table Scan on users1 Due to Missing WHERE Clause

## Introduction

**Problem Statement:**
A `SELECT * FROM users1` query was performing a full table scan across ~997,774 rows with no index being used, causing severe performance degradation. The absence of a WHERE clause meant MySQL had no way to narrow the result set, and no index on commonly filtered columns made targeted queries equally slow.

**Environment:**
- Technology / Service: MySQL 8.x, InnoDB
- Component: Table `users1`
- Severity: P2

---

## Process

### 1. Identifying the Problem
The query `SELECT * FROM users1` was submitted for optimization analysis. With nearly 1 million rows in the table, the query was expected to be slow due to returning all rows without any filter.

### 2. Diagnosis

EXPLAIN was run to inspect the query execution plan:

```sql
EXPLAIN SELECT * FROM users1;
```

Output:

| id | select_type | table  | type | possible_keys | key  | rows   | filtered |
|----|-------------|--------|------|---------------|------|--------|----------|
| 1  | SIMPLE      | users1 | ALL  | NULL          | NULL | 997774 | 100.00   |

`type: ALL` confirmed a full table scan. `possible_keys: NULL` and `key: NULL` showed no index was available or used. The table structure was then inspected:

```sql
SHOW CREATE TABLE users1;
```

Result showed the table had only a `PRIMARY KEY` on `id`, with no secondary indexes on filter-friendly columns like `city`, `age`, or `name`.

### 3. Root Cause

Two issues combined to cause the problem. First, the query had no `WHERE` clause, meaning MySQL had no choice but to scan every row regardless of indexes. Second, the `city` column — a common filter candidate — had no index, meaning even well-formed queries filtering by city would also result in full table scans across ~1M rows.

### 4. Fix Applied

A secondary index was added on the `city` column to support filtered queries:

```sql
ALTER TABLE users1 ADD INDEX idx_city (city);
```

The original query was also rewritten to include a WHERE clause:

```sql
-- Recommended query pattern
SELECT * FROM users1 WHERE city = 'Delhi';

-- Or with a row limit
SELECT * FROM users1 LIMIT 100;

-- Or fetching a specific record
SELECT * FROM users1 WHERE id = 500;
```

### 5. Verification

EXPLAIN was re-run on the filtered query after the index was applied:

```sql
EXPLAIN SELECT * FROM users1 WHERE city = 'Delhi';
```

Output:

| id | select_type | table  | type | possible_keys | key      | rows | filtered |
|----|-------------|--------|------|---------------|----------|------|----------|
| 1  | SIMPLE      | users1 | ref  | idx_city      | idx_city | 1    | 100.00   |

`type` changed from `ALL` to `ref`. Rows scanned dropped from **997,774 to 1**. The index is being used correctly.

---

## Conclusion

**Summary:**
A `SELECT * FROM users1` query was causing a full table scan on a ~1M row table due to the absence of a WHERE clause and no secondary indexes on filter columns. Adding an index on `city` and rewriting the query to include a WHERE clause reduced rows scanned from ~1M to 1, resolving the performance issue.

**Prevention:**
- Always include a WHERE clause when querying large tables — never run `SELECT * FROM <table>` without a filter in production
- Add secondary indexes on columns frequently used in WHERE, JOIN, or ORDER BY clauses
- Run `EXPLAIN` before executing any query on a table with more than 100,000 rows
- Use `LIMIT` as a safeguard during development and debugging to avoid accidental full scans
- Periodically audit tables for missing indexes using `SHOW INDEX FROM <table>` and `information_schema`

---

## Tags

`mysql` `performance` `full-table-scan` `indexing` `slow-query`
