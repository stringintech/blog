---
title: Optimizing PostgreSQL Mass Deletions with Table Partitioning
date: 2024-10-06 00:00:00+0000
image:
tags:
  - Database
weight: 1
---

## Introduction

In database management, handling large-scale data operations efficiently is critical. One common challenge is executing mass deletions on large tables without dragging down overall performance. This article looks at how PostgreSQL's table partitioning feature can significantly speed up the process and help maintain smooth database operations.

## The Challenge of Mass Deletions

Deleting a large number of rows from a PostgreSQL table can be a time-consuming operation. It involves:

1. Scanning through the table to find the rows to delete
2. Removing the rows and updating indexes
3. Vacuuming the table to reclaim space

For tables with millions of rows, this process can lead to long-running transactions and table locks, potentially impacting database responsiveness.

## Enter Table Partitioning

Table partitioning is a technique where a large table is divided into smaller, more manageable pieces called partitions. These partitions are separate tables that share the same schema as the parent table.

## My Benchmark Setup

To quantify the benefits of partitioning, I set up a benchmark with three scenarios using PostgreSQL in a containerized environment:

1. **Simple Table:** A standard, non-partitioned table
2. **Partitioned Table (Row Deletion):** A table partitioned by week, deleting rows from the first week
3. **Partitioned Table (Partition Drop):** Same as #2, but dropping the entire first week's partition

### PostgreSQL Container Specifications

- PostgreSQL Version: 16.4
- Docker Version: 27.0.3
- Resource Limits:
  - CPU Limit: 8 CPUs
  - Memory Limit: 1 GB

### Data Characteristics

- Total Records: 4 million
- Distribution: Evenly distributed over 4 weeks (1 million per week)
- Indexing: Both tables (simple and partitioned) have an index on the `time` column

## Key Findings

| Scenario                      | Deletion Time | Table Size |
|-------------------------------|---------------|------------|
| Simple Table                  | 1.26s         | 728 MB     |
| Partitioned (Delete Rows)     | 734ms         | 908 MB     |
| Partitioned (Drop Partition)  | 6.43ms        | 908 MB     |

1. **Dramatic Speed Improvement:** Dropping a partition is 196 times faster than deleting rows from a simple table.
2. **Storage Trade-off:** Partitioned tables use about 25% more storage due to additional metadata and per-partition indexes.
3. **Minimal Insertion Impact:** Partitioning only slightly increased data population time (by about 2.8%).

## Why It Works

1. **Targeted Operations:** Partitioning allows the database to work with a subset of the data, reducing the scope of operations.
2. **Metadata Operations:** Dropping a partition is primarily a metadata operation, avoiding the need to scan and delete individual rows.
3. **Reduced Lock Contention:** Smaller partitions mean fewer locks, allowing for better concurrency.

## Implementation Highlights

Here's a simplified example of how to set up a partitioned table in PostgreSQL:

```sql
CREATE TABLE records (
    id BIGSERIAL,
    time TIMESTAMPTZ NOT NULL,
    body TEXT
) PARTITION BY RANGE (time);

CREATE TABLE records_week_1 PARTITION OF records
    FOR VALUES FROM ('2023-01-01') TO ('2023-01-08');

-- Create index on the partition
CREATE INDEX idx_records_week_1_time ON records_week_1 (time);

-- To delete a week's worth of data:
ALTER TABLE records DETACH PARTITION records_week_1;
DROP TABLE records_week_1;
```

## Conclusion

For databases dealing with time-series data or any scenario where large-scale deletions are common, implementing table partitioning can lead to significant performance improvements. While there's a small trade-off in storage and insertion speed, the gains in deletion efficiency often far outweigh these costs.

By leveraging partitioning, you can maintain high performance even as your data grows, ensuring your PostgreSQL database remains responsive and efficient.

[Link to the full benchmark code and detailed results](https://github.com/stringintech/db-stuff/tree/main/postgres-partitioning)