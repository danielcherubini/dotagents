# SQL / PostgreSQL Code Review Guide

> SQL and PostgreSQL code review guide covering security, performance optimization, query patterns, indexing strategies, and PostgreSQL 17+ features.

## Table of Contents

- [SQL Injection Prevention](#sql-injection-prevention)
- [Query Performance](#query-performance)
- [Indexing Strategies](#indexing-strategies)
- [Transaction & Concurrency](#transaction--concurrency)
- [Schema Design](#schema-design)
- [PostgreSQL 17+ Features](#postgresql-17-features)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

---

## SQL Injection Prevention

### Always Use Parameterized Queries

```python
# ❌ VULNERABLE: String concatenation (SQL injection!)
cursor.execute(f"SELECT * FROM users WHERE email = '{user_input}'")

# ✅ SAFE: Parameterized query
cursor.execute("SELECT * FROM users WHERE email = %s", (user_input,))
```

```javascript
// ❌ VULNERABLE: Template literal interpolation
const query = `SELECT * FROM products WHERE name LIKE '%${searchTerm}%'`;
connection.query(query);

// ✅ SAFE: Parameterized query
connection.query(
  "SELECT * FROM products WHERE name ILIKE '%' || $1 || '%'",
  [searchTerm]
);
```

```java
// ❌ VULNERABLE: Statement concatenation
String sql = "SELECT * FROM orders WHERE status = '" + status + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);

// ✅ SAFE: PreparedStatement
PreparedStatement pstmt = conn.prepareStatement(
  "SELECT * FROM orders WHERE status = ?"
);
pstmt.setString(1, status);
ResultSet rs = pstmt.executeQuery();
```

### ORM Parameterization

```python
# ❌ VULNERABLE: Raw SQL in ORM
User.objects.raw(f"SELECT * FROM users WHERE name = '{name}'")

# ✅ SAFE: ORM parameterized query
User.objects.filter(name=name)

# ✅ SAFE: Raw with parameters
User.objects.raw("SELECT * FROM users WHERE name = %s", [name])
```

### Review Checklist for SQL Security

- [ ] All user input uses parameterized queries (never string concatenation)
- [ ] ORM queries use built-in methods (not raw SQL)
- [ ] No dynamic table/column names from user input
- [ ] Database user has minimal privileges (no superuser in app)
- [ ] Connection strings not hardcoded (use env vars/secret manager)

---

## Query Performance

### Always Use EXPLAIN ANALYZE for Slow Queries

```sql
-- ✅ Analyze query plan and actual execution times
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.*, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
ORDER BY o.total DESC
LIMIT 100;

-- Look for:
-- - Seq Scan on large tables (should use Index Scan)
-- - High actual rows vs estimated rows (stats outdated?)
-- - Sort/Hash operations on large datasets
-- - High execution time in "Planning Time" or "Execution Time"
```

### Common Performance Anti-Patterns

```sql
-- ❌ SELECT * retrieves unnecessary columns and prevents index-only scans
SELECT * FROM users WHERE email = 'test@example.com';

-- ✅ Select only needed columns
SELECT id, name, email FROM users WHERE email = 'test@example.com';
```

```sql
-- ❌ Function on indexed column prevents index usage
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- ✅ Use range query for index-friendly filtering
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

```sql
-- ❌ LIKE with leading wildcard prevents index usage
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Use pg_trgm extension for text search
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_gin ON products USING gin (name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%phone%';
```

```sql
-- ❌ N+1 query pattern in application code
-- Fetch users first
users = SELECT * FROM users;
-- Then fetch orders for each user (N queries!)
for user in users:
    orders = SELECT * FROM orders WHERE user_id = %s, [user.id]

-- ✅ Use JOIN or array aggregation
SELECT u.*, ARRAY_AGG(o.id) as order_ids
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

### Subqueries vs JOINs

```sql
-- ❌ Correlated subquery (executes once per outer row)
SELECT u.name,
       (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) as order_count
FROM users u;

-- ✅ JOIN with GROUP BY (single pass)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

---

## Indexing Strategies

### When to Create Indexes

| Scenario | Index Type | Example |
|----------|-----------|---------|
| Equality lookups | B-tree (default) | `WHERE status = 'active'` |
| Range queries | B-tree | `WHERE created_at > '2024-01-01'` |
| Composite filters | Composite index | `WHERE status = ? AND created_at > ?` |
| Full-text search | GIN (pg_trgm/tsvector) | `WHERE name ILIKE '%phone%'` |
| JSONB queries | GIN | `WHERE data->>'key' = 'value'` |
| Unique constraints | Unique index | `UNIQUE (email)` |

### Composite Index Design

```sql
-- ✅ Order matters: equality columns first, then range
-- Query: WHERE status = 'active' AND created_at > '2024-01-01'
CREATE INDEX idx_orders_status_created ON orders (status, created_at);

-- ❌ Wrong order: range column first makes status index unusable
CREATE INDEX idx_orders_created_status ON orders (created_at, status);
```

### Partial Indexes for Common Queries

```sql
-- ✅ Only index active records (saves space, faster queries)
CREATE INDEX idx_active_orders ON orders (user_id)
WHERE status = 'active';

-- ✅ Index only recent data
CREATE INDEX idx_recent_logs ON logs (created_at DESC)
WHERE created_at > NOW() - INTERVAL '30 days';
```

### Index Maintenance Checklist

- [ ] No duplicate indexes (same columns in same order)
- [ ] Indexes don't exceed ~20% of table size
- [ ] Running `ANALYZE` after large data changes
- [ ] Monitoring index usage: `SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;`

---

## Transaction & Concurrency

### Isolation Levels

```sql
-- ✅ Default: READ COMMITTED (safe for most cases)
BEGIN;
SELECT * FROM accounts WHERE id = 1; -- Snapshot at query start
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- ⚠️ SERIALIZABLE: Prevents all anomalies but low throughput
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
-- Complex multi-table operations
COMMIT;
```

### Avoid Long-Running Transactions

```sql
-- ❌ Bad: Transaction holds locks for too long
BEGIN;
SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;
-- User takes 30 seconds to confirm purchase...
UPDATE inventory SET count = count - 1 WHERE product_id = 1;
COMMIT; -- Now other transactions blocked for 30s!

-- ✅ Good: Minimize work inside transaction
BEGIN;
SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;
UPDATE inventory SET count = count - 1 WHERE product_id = 1;
COMMIT; -- Commit quickly, then do external work
```

### Deadlock Prevention

| Rule | Reason |
|------|--------|
| Always access tables in same order | Prevents circular lock dependencies |
| Keep transactions short | Reduces lock contention window |
| Use `NOWAIT` or `SET lock_timeout` | Fail fast instead of waiting |
| Retry on deadlock errors | PostgreSQL error code `40P01` |

---

## Schema Design

### Data Type Selection

| Use Case | Recommended Type | Why |
|----------|-----------------|-----|
| IDs | `BIGSERIAL` or `UUID` | Auto-increment or distributed |
| Email | `VARCHAR(255)` | Standard length, indexed |
| Price/Money | `NUMERIC(10,2)` | Exact decimal (no float errors) |
| Timestamps | `TIMESTAMPTZ` | Timezone-aware storage |
| Boolean flags | `BOOLEAN` | Clear intent, efficient storage |
| JSON data | `JSONB` | Binary format, indexed, fast queries |
| Text content | `TEXT` | No length limit, efficient |

### Normalization vs Performance

```sql
-- ✅ Normalized: Referential integrity, no duplication
CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE orders (id SERIAL PRIMARY KEY, user_id INT REFERENCES users(id));

-- ⚠️ Denormalized: Faster reads, but sync complexity
-- Only use when read performance is critical and data changes infrequently
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_name TEXT,  -- Duplicated from users table
    user_email TEXT  -- Duplicated from users table
);
```

### Constraints Are Your Friend

```sql
-- ✅ Always use constraints for data integrity
CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    balance NUMERIC(12,2) NOT NULL CHECK (balance >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ✅ Add constraints for business rules
ALTER TABLE orders ADD CONSTRAINT chk_future_date
    CHECK (expected_delivery > order_date);
```

---

## PostgreSQL 17+ Features

### New Memory Management for VACUUM

```sql
-- PG17: Reduced memory consumption during VACUUM
-- No config change needed — automatic improvement
VACUUM (VERBOSE, ANALYZE) users;
```

### Improved Parallel Query Execution

```sql
-- PG17: Better parallel query planning for complex queries
SET max_parallel_workers_per_gather = 4;
SELECT COUNT(*), AVG(amount)
FROM large_transactions
WHERE created_at > NOW() - INTERVAL '1 year';
```

### JSONB Enhancements

```sql
-- ✅ PG17: Improved JSONB operators and indexing
SELECT data->>'key' FROM jsonb_table WHERE data ? 'key';

-- GIN index on JSONB paths (PG13+)
CREATE INDEX idx_jsonb_data ON table USING gin ((data->'nested'));
```

### Logical Replication Improvements

```sql
-- PG17: Better logical replication performance
-- Subscribe to specific tables with filters
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (copy_data = false, create_slot = false);
```

---

## Common Mistakes

### Type Coercion Issues

```sql
-- ❌ Implicit type conversion prevents index usage
WHERE user_id = '123'  -- String compared to integer

-- ✅ Explicit type matching
WHERE user_id = 123  -- Integer compared to integer
```

### NULL Handling Pitfalls

```sql
-- ❌ NULL != NULL (both return UNKNOWN)
SELECT * FROM users WHERE email != 'test@example.com';
-- Misses rows where email IS NULL

-- ✅ Explicit NULL handling
SELECT * FROM users
WHERE email != 'test@example.com' OR email IS NULL;

-- ✅ Use COALESCE for consistent comparisons
SELECT * FROM users
WHERE COALESCE(email, '') != 'test@example.com';
```

### Pagination Gotchas

```sql
-- ❌ Offset pagination doesn't scale (slow on large offsets)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 100000;

-- ✅ Keyset pagination (consistent performance)
SELECT * FROM products
WHERE id > last_seen_id
ORDER BY id
LIMIT 20;
```

### Array and Set Operations

```sql
-- ❌ Using arrays for many-to-many relationships
CREATE TABLE posts (id INT, tags TEXT[]);

-- ✅ Use junction table (normalization)
CREATE TABLE post_tags (post_id INT REFERENCES posts(id), tag_id INT REFERENCES tags(id));
```

---

## Review Checklist

### Security
- [ ] All queries use parameterized statements (no string concatenation)
- [ ] Database user has minimal privileges (no superuser/DROP/ALTER)
- [ ] No sensitive data in logs or error messages
- [ ] Connection strings stored securely (env vars, not code)

### Performance
- [ ] EXPLAIN ANALYZE run on slow queries (>100ms)
- [ ] SELECT * avoided in production code
- [ ] Indexes exist for WHERE/JOIN columns
- [ ] No N+1 query patterns in application code
- [ ] Large result sets paginated (not loaded entirely)

### Schema Design
- [ ] Appropriate data types used (TIMESTAMPTZ, NUMERIC for money)
- [ ] NOT NULL constraints where appropriate
- [ ] CHECK constraints for business rules
- [ ] Foreign keys defined for referential integrity
- [ ] Indexes not excessive (<20% of table size)

### Concurrency
- [ ] Transactions kept short (no user interaction inside)
- [ ] Consistent table access order to prevent deadlocks
- [ ] Appropriate isolation level for transaction type
- [ ] Retry logic for deadlock errors (40P01)

### PostgreSQL Specific
- [ ] Using `TIMESTAMPTZ` not `TIMESTAMP`
- [ ] JSONB used for semi-structured data (not TEXT)
- [ ] pg_trgm extension for full-text search needs
- [ ] VACUUM running regularly (pg_autovacuum enabled)
- [ ] Statistics up-to-date (ANALYZE after bulk loads)

---

## Reference Resources

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [SQL Injection Prevention Cheat Sheet - OWASP](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [PostgreSQL Performance Tips](https://www.postgresql.org/docs/16/performance-tips.html)
- [EXPLAIN ANALYZE Guide](https://www.enterprisedb.com/blog/postgresql-query-optimization-performance-tuning-with-explain-analyze)
- [PgHero](https://github.com/jeremyevans/pghero) - PostgreSQL performance dashboard
