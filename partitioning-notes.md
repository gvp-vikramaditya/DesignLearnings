# Partitioning: the Same Word, Three Different Jobs

Partitioning splits data into groups, but the reason matters. These are three common uses, not three mutually exclusive types:

| Where | Main goal |
| --- | --- |
| Relational tables, such as SQL Server or Oracle | Manage retention and maintenance; skip irrelevant partitions |
| Data lakes queried by Spark | Read less data |
| Distributed databases, such as Cassandra or DynamoDB | Route requests and spread storage and traffic |

## 1. Relational tables: make old data easy to manage

Suppose we retain six years of data, with a partition for every six months. Most everyday queries touch recent data, while older partitions remain available for occasional access.

The table is still one logical table. Partitioning does not automatically make every query faster: queries benefit from **partition elimination** when their filters let the database skip irrelevant partitions. Indexes still matter.

The big operational benefit is retention. Instead of a large `DELETE`, retire a whole expired partition. Oracle supports dropping partitions; SQL Server commonly uses partition switching or truncation. Align boundaries with the retention policy, and remove a partition only when all its data has expired.

## 2. Data lakes: avoid reading what the query does not need

For time-based queries, grouping files by date lets Spark skip unrelated data through **partition pruning**.

These object-storage layouts both represent daily partitions:

```text
year=2026/month=10/day=07/
date=2026-10-07/
```

More folder levels do not automatically mean finer granularity. An additional hour level would make these hourly partitions.

With ordinary Hive-style folders, queries generally need a filter on the partition columns. A filter on `event_time` alone may not prune a separate `date` column. Iceberg can derive partition filters from its declared timestamp transforms.

### Choose granularity from queries and data volume

Hourly partitions may help when queries usually read the last few hours, but only if there is enough data per hour. Daily or monthly partitions may work better for lower volumes or wider query ranges.

Monthly partitioning is not necessarily useless for an hourly query: file and Parquet row-group statistics may still skip data. It simply provides less precise partition pruning.

**Partition size and file size are separate decisions.** One daily partition can contain many files, and many workers can read it. A storage partition is also not the same thing as a Spark execution or shuffle partition.

### Why small files are expensive

Reading thousands of tiny files adds work beyond reading their bytes:

- Discovering files, listing directories or object keys, and tracking metadata.
- Opening each file, making storage requests, and reading file headers or footers.
- Planning and scheduling work, sometimes spending more time on overhead than useful processing.

Spark can group small files into a task, so one file does not necessarily mean one task. That still does not eliminate per-file overhead.

### Batch writes, streaming, and compaction

A daily ETL job can write several reasonably sized files into one daily partition. It need not produce exactly one file. A few hundred MB per file is a useful starting point for many analytics workloads, not a universal target.

Streaming prioritizes freshness. Frequent micro-batches and parallel writers can produce small files even inside a daily partition. Writers do not need to hold an entire target-sized file in memory; buffering, batch size, and flush behavior depend on the implementation.

**Compaction** rewrites small files into larger ones, usually within each partition. Run it periodically or use supported automatic compaction. It improves the file layout without changing the logical rows.

## 3. Distributed databases: avoid concentrating all the traffic

A partition key helps route requests to the responsible storage shards or replicas. It is not a physical node ID.

In DynamoDB, an item lookup needs the full primary key: the partition key and, if defined, the sort key. Cassandra similarly uses partition-key columns to locate data and clustering columns to organize rows within a partition.

Consider billing events keyed like this:

```text
Partition key: date
Sort key:      timestamp + event ID
```

This makes "read today's events" convenient, but all new writes target today's key. Recent-data queries also concentrate there. The rest of the database may have spare capacity while that key becomes a **hot partition**.

A date-only key is not always wrong; it can work at modest volumes. At high volumes, consider:

- **Entity plus time bucket**, such as customer ID plus month, when queries are customer-specific. A very busy customer can still be hot.
- **Time bucket plus shard**, such as date plus a stable hash-derived shard number, to spread a day's traffic across multiple keys.

Sharding trades write distribution for read complexity. A time-range query must query the relevant buckets and shards, filter by the sort key, and combine the results. Choose enough shards for expected traffic without creating unnecessary fan-out.

This differs from a lake: Spark can spread one date's files across many workers, while a database shard has serving limits. Some databases also separate compute and storage, so physical co-location is not the deciding factor. **What matters is how traffic is distributed and where the limits are.**

## 4. What Delta Lake and Iceberg add

A data lake is the storage environment. **Delta Lake and Apache Iceberg are table formats**, commonly used over Parquet files.

They track which files belong to the table through metadata and committed versions, rather than treating every file in a folder as valid data. They provide:

- **Atomic commits and consistent reads.** Files become visible through a successful table commit. A failed write may leave unreferenced files for later cleanup, rather than exposing a half-written table. Safe retries still require suitable checkpointing or idempotency.
- **Data skipping.** Partition information and statistics, such as minimum and maximum values, help skip files. These are not automatically general-purpose secondary indexes.
- **Maintenance through supported engines.** Compaction, updates, deletes, and merges avoid much custom file-management code. Delta's `OPTIMIZE`, for example, compacts small files.

Folder layouts can already partition by multiple columns; adding another dimension is not impossible. Changing an existing layout, however, usually requires rewriting data and updating table definitions. Iceberg supports **partition evolution**: new writes use the new layout while old files remain readable under their original layout. That does not reorganize old data automatically, and Delta's layout-changing capabilities are not identical.

### Deleting sensitive data needs a second step

A table-level `DELETE` is not necessarily physical erasure. Old versions may retain the data, and delete markers can hide rows without rewriting their files.

For actual removal, use the format's supported rewrite/purge and cleanup operations: for example, Delta purge where needed followed by `VACUUM`, or Iceberg rewrites and snapshot expiration with file cleanup. Respect retention and active readers, and account for backups and object-store versions. If a leaked value is a credential, rotate it too.

## The takeaway

Choose partitions around **query patterns, write volume, and retention**. Then separately tune file sizes for analytics or key distribution for databases. A convenient date boundary does not guarantee an efficient design.

## References

- [SQL Server partitioned tables and indexes](https://learn.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes)
- [Spark performance tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [DynamoDB partition-key design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html) and [write sharding](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-sharding.html)
- [Iceberg partitioning](https://iceberg.apache.org/docs/latest/partitioning/), [partition evolution](https://iceberg.apache.org/docs/latest/evolution/), and [reliability](https://iceberg.apache.org/docs/latest/reliability/)
- [Delta compaction and data skipping](https://docs.delta.io/optimizations-oss/), [deletion vectors and purge](https://docs.delta.io/delta-deletion-vectors/), and [file cleanup](https://docs.delta.io/delta-utility/)
