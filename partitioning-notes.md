# Partitioning: the Same Word, Three Different Jobs

Partitioning sounds simple: split data into smaller groups. The confusing part is that those groups do different jobs in different systems. A date partition in SQL Server, a folder in a data lake, and a partition key in DynamoDB might look similar, but we shouldn't design them all the same way.

Let's look at three common uses. They overlap, but each starts with a different problem.

## 1. In a relational table, make old data easy to manage

Say we're keeping six years of billing data, but most everyday queries only care about recent bills. We could divide the table into six-month partitions, keeping the older data available without treating it all as one block for maintenance.

The benefit becomes clear when old data expires. Rather than running a huge `DELETE`, we can retire a whole partition. Oracle supports dropping partitions; SQL Server can truncate a partition or switch it into another table for archiving. The boundaries need to match our retention policy: we can't remove a six-month block until all its data is eligible.

To the application, this is still one table. Queries can also benefit when their filters let the database skip older partitions, which is called **partition elimination**. So the working set can be smaller, but partitioning isn't a shortcut around good indexes.

## 2. In a data lake, avoid reading what we don't need

Now imagine the same billing data lives in Parquet files and we're using Spark to analyze it. If a query asks for last Friday's data, we'd rather not read six years of files first.

Grouping files by date lets Spark skip unrelated partitions. That's **partition pruning**. We might use either of these object-storage layouts:

```text
year=2026/month=10/day=07/
date=2026-10-07/
```

Both are daily partitions. The first looks more hierarchical, but it isn't more granular. Adding an hour level would make it hourly.

There's a small catch with ordinary Hive-style folders: the query generally needs to filter the partition columns. Filtering `event_time` alone may not tell Spark to prune a separate `date` column.

### How small should a partition be?

If most queries ask for the last two hours, monthly partitions are probably coarser than we'd like. They aren't useless, because file and Parquet row-group statistics may still help skip data. But daily or hourly partitions could narrow the search earlier.

Going finer isn't always better, though. If we barely generate any data per hour, minute-level partitions may just leave us with lots of tiny files.

It helps to separate two decisions: **how we group data and how large we make each file**. A daily partition can contain many files, read by many workers. It isn't the same thing as a Spark execution or shuffle partition.

### Why do small files cause overhead?

Spark does more than read the bytes. It has to discover files, track metadata, open files, make storage requests, and read headers or footers. It also spends time planning and scheduling the work. With thousands of tiny files, that overhead can outweigh the useful processing.

Spark can group small files into one task, so it isn't necessarily one task per file. But each file still has a cost.

### Streaming makes this trade-off more obvious

A daily ETL job has a whole day's data to work with, so it can write several reasonably sized files into that day's partition. We don't need exactly one file; a few hundred MB per file is a common starting point, not a rule.

With a Kafka stream, we may want fresh data every ten or twenty seconds. Frequent micro-batches and parallel writers can then produce small files. This isn't because a writer must hold an entire target-sized file in memory; buffering and flushing depend on the implementation. We're balancing freshness against file size.

One practical answer is to write promptly, then periodically combine small files. This is **compaction**: rewriting them into larger files, usually within each partition, without changing the logical rows. Some engines can automate it.

## 3. In a distributed database, don't send all the traffic to one place

Now let's try that date-based design in DynamoDB:

```text
Partition key: date
Sort key:      timestamp + event ID
```

It looks convenient. To read today's events, we know which key to query. But today's incoming events all target that same key, and recent-data queries concentrate there too. We can end up with a **hot partition** while the rest of the database has spare capacity.

Here, the partition key helps route requests to the responsible storage shards or replicas; it isn't a physical server ID. A DynamoDB item lookup needs the full primary key, including the sort key when defined. Cassandra similarly routes by partition-key columns and organizes rows within a partition using clustering columns.

A date-only key isn't always wrong. At modest volumes, it may be enough. If we usually query one customer's bills, customer ID plus month may fit better, although one very busy customer can still become hot. Another option is date plus a stable hash-derived shard number, spreading a day's traffic across several keys.

That extra distribution has a price: reading a time range now means querying the relevant buckets and shards and combining the results. We want enough shards to handle the traffic, not so many that every read becomes expensive.

This is why the lake design doesn't transfer directly. Spark can read one date's files using many workers, while a database shard has serving limits. Some databases also separate compute and storage, so the key distinction isn't physical co-location. It's **where the requests go and how much traffic that destination can handle**.

## Where Delta Lake and Iceberg fit

Back in the lake, compaction introduces another problem: we need to replace files without readers seeing a half-finished change. A writer might also crash after writing some files but before finishing the batch.

**Delta Lake and Iceberg are table formats**, commonly used over Parquet files, that manage this through metadata and commits. Readers see a committed version of the table, not simply whatever files happen to be in a folder. Failed writes can leave unreferenced files for cleanup; safe retries still need appropriate checkpointing or idempotency.

Their metadata also helps skip files using partition information and statistics such as minimum and maximum values. Think of this as avoiding unnecessary reads, not automatically getting database-style secondary indexes. Through supported engines, we also get compaction, updates, deletes, and merges without writing all the file-management code ourselves. Delta's `OPTIMIZE`, for example, compacts small files.

Iceberg also helps when our partitioning needs change. Folders can already partition by multiple columns, but changing an existing layout usually means rewriting data and updating definitions. With **partition evolution**, new writes can use a new layout while old files remain readable under their original one. Changing the partition spec doesn't reorganize old files, and Delta has different layout-changing capabilities. Iceberg's timestamp transforms also let it derive partition filters from time predicates.

### Deleting a row isn't always erasing it

Suppose a secret accidentally lands in the lake. A table-level `DELETE` can hide it from current queries, but old versions or files may still contain it.

Actual removal needs the format's supported purge and cleanup process: Delta purge where needed followed by `VACUUM`, or Iceberg rewrites and snapshot expiration with file cleanup. We still have to account for retention, active readers, backups, and object-store versions. If the secret is a credential, rotate it too.

## The takeaway

Start with what we want partitioning to do: retire old data, skip unnecessary reads, or spread traffic. Then choose the layout around the queries, write volume, and retention policy. A date boundary is convenient, but convenience alone doesn't make it a good partition key.

## References

- [SQL Server partitioned tables and indexes](https://learn.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes)
- [Spark performance tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [DynamoDB partition-key design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html) and [write sharding](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-sharding.html)
- [Iceberg partitioning](https://iceberg.apache.org/docs/latest/partitioning/), [partition evolution](https://iceberg.apache.org/docs/latest/evolution/), and [reliability](https://iceberg.apache.org/docs/latest/reliability/)
- [Delta compaction and data skipping](https://docs.delta.io/optimizations-oss/), [deletion vectors and purge](https://docs.delta.io/delta-deletion-vectors/), and [file cleanup](https://docs.delta.io/delta-utility/)
