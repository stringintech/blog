---
title: "PostgreSQL Buffer Cache: A Practical Guide"
date: 2024-12-07 13:10:00+0000
image:
tags:
  - Database
  - PostgreSQL
weight: 1
---

## Introduction

Before adding indexes or application-level caching to optimize PostgreSQL performance, it's worth understanding how a relational database like PostgreSQL manages memory. While we'll focus on PostgreSQL's implementation, the concepts discussed here are fundamental to understanding memory management in most relational database systems.

PostgreSQL keeps frequently accessed data in memory, sometimes providing the performance boost we need without additional complexity of introducing more fined-grained caches in application level. Let's try to come up with a basic idea of this mechanism through practical examples to hopefully better inform our optimization decisions.

## Understanding the Fundamentals

Before diving into practical examples, let's clarify some key PostgreSQL concepts:

- **Relation**: Any database object that contains rows. Tables, indexes, and sequences are all relations in PostgreSQL.
- **Page**: The basic unit of storage in PostgreSQL (typically 8KB). Each relation is stored as a collection of pages on disk.
- **Buffer**: When a page is loaded into memory, it becomes a buffer. Think of buffers as the in-memory representation of disk pages.
- **Buffer Cache**: A shared memory area where PostgreSQL keeps frequently accessed pages.

## Our Toolkit: PostgreSQL Buffer Cache Monitoring with `pg_buffercache` and `pg_class`

To explore buffer cache behavior, we'll use two main PostgreSQL tools:
1. **`pg_buffercache` extension**: Provides real-time visibility into shared buffer cache content, allowing us to track which pages are in memory and their current state. See the PostgreSQL [documentation](https://www.postgresql.org/docs/current/pgbuffercache.html) for more details.
2. **`pg_class` system catalog**: Contains metadata about database objects (tables, indexes, etc.)

Let's start by installing the `pg_buffercache` extension:
```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
```

## Practical Exploration

We'll follow these steps to understand buffer cache behavior:

1. Create a test table with predictable data size
2. Observe how data is stored in pages
3. Use `pg_buffercache` to observe how pages are loaded into memory during queries
4. Add an index to see how it affects page loading patterns
5. Track how buffer cache state changes when we modify data
6. See how system processes handle dirty (modified) pages

### Setting Up Our Test Environment

Let's create a test table **without** any indexes:
```sql
CREATE TABLE buffer_test (
    id      SERIAL,
    data    TEXT
);

-- Insert 100k rows with 1KB data each
INSERT INTO buffer_test (data)
SELECT repeat('x', 1024)
FROM generate_series(1, 100000);
```

### Understanding Table Size

First, let's examine our table's size using `pg_class`:
```sql
SELECT relpages, reltuples 
FROM   pg_class 
WHERE  relname = 'buffer_test';
```

| relpages | reltuples |
|----------|-----------|
| 0        | 0         |

The zero values indicate that table statistics haven't been updated. Let's fix that:
```sql
ANALYZE buffer_test;

SELECT relpages, reltuples 
FROM   pg_class 
WHERE  relname = 'buffer_test';
```

| relpages | reltuples |
|----------|-----------|
| 14,286   | 100,000   |
Here:
- `relpages`: Number of disk pages the table uses
- `reltuples`: Estimated number of rows

### Monitoring Cache Behavior

Let's verify our cache is empty:
```sql
SELECT    COUNT(*) 
FROM      pg_buffercache b
JOIN      pg_class c 
  ON      b.relfilenode = c.relfilenode
WHERE     c.relname = 'buffer_test';
```

| count |
|-------|
| 0     |

The `relfilenode` column in `pg_buffercache` helps us identify which buffers belong to our table. Now, let's query a specific row:
```sql
SELECT * FROM buffer_test WHERE id = 70000;

-- Check cache after the query
SELECT    COUNT(*) 
FROM      pg_buffercache b
JOIN      pg_class c 
  ON      b.relfilenode = c.relfilenode
WHERE     c.relname = 'buffer_test';
```

| count |
|-------|
| 32    |
You may have expected to see only one page loaded into memory since the row we queried earlier belongs to one page. But it's not the case. Since we have not introduced any indexes on the id column yet, database cannot efficiently find that one page and it has to do a sequential scan which causes PostgreSQL to read through all table pages from the beginning until it finds our target row. The number of pages loaded depends on various factors including the database's buffer replacement strategy.

### Adding an Index

Let's add an index:
```sql
CREATE INDEX ON buffer_test(id);
```

Now restart PostgreSQL server to clear the cache. As an example, this is how I did it on my installed version on macOS using [pg_ctl](https://www.postgresql.org/docs/current/app-pg-ctl.html):
```bash
sudo -u postgres pg_ctl restart -D /Library/PostgreSQL/17/data
```

After restart, query the same row:
```sql
SELECT    ctid, id 
FROM      buffer_test 
WHERE     id = 70000;
```

| ctid      | id    |
|-----------|-------|
| (9999,7)  | 70000 |
The `ctid` (Tuple ID) is a special system column in PostgreSQL that represents the physical location of a row **version** within its table. Every row in a PostgreSQL table has a Tuple ID that consists of two numbers: the block number (or page number) and the tuple index within that block. Here `ctid` shows our row is on page 9999. Let's check the buffer cache:

```sql
SELECT    bufferid, 
          relblocknumber, 
          isdirty 
FROM      pg_buffercache b  
JOIN      pg_class c 
  ON      b.relfilenode = c.relfilenode  
WHERE     c.relname = 'buffer_test';
```

| bufferid | relblocknumber | isdirty |
|----------|----------------|----------|
| 149      | 9999          | false    |

With the index, PostgreSQL loaded only the needed page. Key columns used here:
- `bufferid`: Unique identifier for the buffer in shared memory
- `relblocknumber`: Page number within the relation
- `isdirty`: Indicates if the page has been modified

### Observing Dirty Pages

Let's modify our row:
```sql
UPDATE    buffer_test 
SET       data = 'modified data' 
WHERE     id = 70000;

SELECT    bufferid, 
          relblocknumber, 
          isdirty
FROM      pg_buffercache b
JOIN      pg_class c 
  ON      b.relfilenode = c.relfilenode
WHERE     c.relname = 'buffer_test';
```

| bufferid | relblocknumber | isdirty |
|----------|----------------|----------|
| 149      | 9999          | true     |
The page is now marked dirty, indicating pending changes.

### Understanding Checkpoints

By calling `CHECKPOINT` we can force writing dirty pages to disk. By default, PostgreSQL runs automatic checkpoints:
1. Every `checkpoint_timeout` seconds (default: 5 minutes)
2. When WAL or [Write-Ahead Logging](https://www.postgresql.org/docs/current/wal-intro.html) reaches `max_wal_size` (default: 1 GB)

Now let's force a checkpoint to write dirty pages to disk:
```sql
CHECKPOINT;

-- Check cache state
SELECT    bufferid, 
          relblocknumber, 
          isdirty
FROM      pg_buffercache b
JOIN      pg_class c 
  ON      b.relfilenode = c.relfilenode
WHERE     c.relname = 'buffer_test';
```

| bufferid | relblocknumber | isdirty |
| -------- | -------------- | ------- |
| 149      | 9999           | false   |
Not dirty anymore but still in cache!

## Conclusion

We explored PostgreSQL's buffer cache through practical examples that demonstrated its basic memory management behavior. By creating a test table and using monitoring tools, we observed how data pages move between disk and memory during different operations. Our experiments showed how queries without indexes lead to sequential scans that load multiple pages into memory, while adding an index allowed PostgreSQL to load only the specific page needed. We also saw how pages get marked as "dirty" when modified and remain in cache even after a checkpoint writes them to disk.

Using PostgreSQL's monitoring tools like `pg_buffercache`, along with system catalogs like `pg_class`, we were able to observe these buffer cache behaviors in real-time. This hands-on exploration provided us with a practical understanding of PostgreSQL's fundamental memory management, which can help inform decisions about when additional optimization techniques might be needed.