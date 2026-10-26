# Open table formats: from files to reliable tables

These notes combine the original discussion with two supplied video transcripts: Jordan Tigani's DuckLake seminar and the Iceberg co-creator's talk about Iceberg's origins and evolution. The focus is the big picture and the common optimization techniques, rather than format internals.

Historical observations and proposed features in the talks are not statements about what every current implementation supports.

## Start with the problem: a folder of files is not quite a table

Imagine storing events in HDFS, the Hadoop Distributed File System. HDFS understands files, directories, and blocks. It does not inherently understand SQL tables, column identities, or transactions.

Hive provides a SQL and table-management layer over that data. The Hive Metastore records schemas, partition definitions, and storage locations. Together, they let an engine treat a collection of files as a table.

A common layout looks like this:

```text
events
  year=2026
    month=09
      file-a.parquet
      file-b.parquet
    month=10
      file-c.parquet
```

This is **Hive-style partitioning**. Rows with the same partition values are grouped together. For an October query, the engine can skip September. That is partition pruning.

In a traditional directory-based table, the engine identifies relevant partitions and asks the filesystem which files are there. HDFS's NameNode manages the namespace and file-block locations; DataNodes serve the actual file bytes. Computation can often run near those blocks.

This was useful, but the table definition was incomplete: "Read whatever files are in these locations."

The Iceberg talk illustrates the problem with a partition replacement. If a writer adds new files before removing old files, readers may see duplicate data. If it removes the old files first, readers may see missing data. A crash can leave either intermediate state behind.

The missing feature is not merely faster storage. It is the ability to publish a whole table change at once.

This contrast concerns traditional external, directory-based Hive tables. Hive also has transactional capabilities; not every Hive table has these limitations.

## Object storage changed the architecture and the costs

Cloud systems moved persistent data away from disks attached to compute workers and into shared object storage. Compute could grow, shrink, or shut down independently of storage. Several engines could access the same dataset.

That flexibility came with different access costs. Listing a large number of objects may require many remote, paginated requests. Reading thousands of Parquet footers also requires many requests, even if each footer is small.

Object storage can provide excellent aggregate throughput. The issue is often **request latency and request count**, not simply slow transfer of bytes.

Both transcripts discuss historical object-store listing consistency problems. Those problems helped motivate table formats, but they should not be generalized to all object stores today. Even strongly consistent listing does not make a sequence of file additions and removals into an atomic table transaction.

Another important property is the update model:

> Published data files are generally treated as immutable. Table changes create new files or delete information instead of modifying arbitrary bytes inside existing files.

Many object stores allow replacing an object at the same key, and some preserve versions. "Immutable" here means that the lakehouse treats a published file as an unchanged unit rather than a collection of independently mutable database pages.

Object versioning alone does not solve table versioning. It versions one object; a table needs a consistent state spanning many objects.

## Parquet describes one file; a table format describes the collection

Parquet is a columnar file format. It divides rows into row groups, with column chunks inside each group. Its footer describes the schema and the locations of those chunks, along with statistics when available.

```text
One Parquet file
+--------------------------------------+
| Row group 1: column chunks            |
+--------------------------------------+
| Row group 2: column chunks            |
+--------------------------------------+
| Footer: schema, locations, statistics|
+--------------------------------------+
```

A reader can fetch the footer, locate the columns it needs, and skip row groups that cannot match a predicate.

Parquet is therefore self-describing at the file level. But it does not know whether another file replaced it, which snapshot it belongs to, or whether some rows have been logically deleted.

An open table format adds that missing table-level meaning. It records the schema, committed versions, active files, partition information, statistics, and relevant delete information.

**The decisive change is that committed metadata becomes authoritative.** A file can exist in storage without belonging to the table. An obsolete file can remain in storage without belonging to the current version.

## Metadata management and file management

The DuckLake seminar separates storage management into two responsibilities.

**Metadata management** defines the logical table state and coordinates changes. It answers: Which files are visible in this snapshot? What schema applies? What statistics can help the query? Has this write committed?

**File management** maintains the physical layout. It combines small files, reorganizes poorly clustered data, handles accumulated deletes, and removes obsolete files when safe.

These responsibilities depend on each other. Compaction may replace five files with one larger file, but the replacement still needs an atomic metadata commit. An optimization must not make readers see the old files and the replacement together.

The table format defines the representation and rules. Engines and services execute the maintenance. Choosing a format does not automatically mean that compaction or cleanup will run.

## Snapshots make immutable files behave like a changing table

Suppose the table currently uses files A and B:

```text
Snapshot 10 -> [A, B]
```

A writer creates C to replace B, then commits a new version:

```text
Snapshot 11 -> [A, C]
```

A reader using Snapshot 10 continues to read A and B. A reader using Snapshot 11 reads A and C. Neither should see a partially completed replacement.

The general write sequence is:

```text
Create new data files
         |
Prepare metadata describing the change
         |
Validate against concurrent changes
         |
Atomically publish the new table state
```

The atomic step is publishing the new state, not uploading all the files simultaneously.

If a writer crashes before publication, its files may become orphans. They exist physically, but no committed snapshot references them. Cleanup can remove them later.

Concurrent writers also need coordination. A writer may validate its assumptions, retry against a newer state, or fail if a conflicting change makes its operation unsafe.

This is how the system can provide ACID behavior around immutable files. Atomicity prevents partial publication; consistency preserves defined invariants; isolation controls concurrent visibility; durability preserves committed state.

A transactional metadata database helps, but its transaction does not automatically cover external object storage. The system still needs durable file preparation, safe publication, conflict handling, and cleanup. Exact isolation guarantees and transaction scope depend on the implementation.

## Time travel is table history, not automatically record history

A snapshot describes the table at a committed version. Time travel means querying an earlier version:

```text
Snapshot 10: customer 42 has status "trial"
Snapshot 11: customer 42 has status "paid"
Snapshot 12: customer 42 is absent
```

This supports reproducible reports, investigation, and rollback.

It does not automatically provide every change to customer 42. That requires comparing snapshots using a stable identity, consuming change data, or maintaining a history table.

The Iceberg talk's row-lineage discussion addresses this distinction. When a file is rewritten, many unchanged rows may move along with changed rows. Knowing which files were replaced does not, by itself, identify exactly which records changed.

The Git analogy is helpful: new snapshots reuse unchanged data files and metadata. They do not normally copy the entire dataset.

Snapshots describe commit-time state, not necessarily event time. An event from last month can arrive in today's commit.

History also depends on retention. Expiring old snapshots and deleting their unneeded files eventually removes the ability to query those versions.

## Schema evolution depends on identity, not just names

The Iceberg transcript gives a useful example:

```text
Rename id to profile_id
```

is not equivalent to:

```text
Drop id
Add a new column named profile_id
```

A rename preserves existing values. Dropping and adding creates a different column.

Likewise, dropping a column and later creating another with the same name should not resurrect the old values. The speaker calls this accidental reappearance "zombie data."

Stable column identifiers let the table distinguish a column's identity from its display name. Readers can interpret older files correctly even after a rename.

Adding columns and supported widening conversions, such as integer to long, may not require rewriting every old file. The allowed operations still depend on the format and engine.

Partition evolution follows a similar principle. A table might move from monthly to daily partitioning without immediately rewriting all historical data. Metadata describes the layouts used by different files.

Iceberg's hidden partitioning also records transformations such as taking the day of a timestamp. An engine can then derive partition pruning from an event-time filter, rather than requiring a separately maintained partition-column predicate.

## Read less data, but never skip a possible match

Both talks return to the same performance principle:

> Avoid unnecessary reads before trying to make those reads faster.

There are several layers:

```text
All table data
    |
Partition pruning
    |
Candidate files
    |
File-level statistics
    |
Selected Parquet files
    |
Row-group or page skipping
    |
Read only required columns
```

For example, a file whose customer IDs range from 100 to 200 cannot contain customer 900. The engine can skip it.

For customer 150, that same file remains a candidate. Min/max statistics do not prove that every value between the bounds exists.

The DuckLake seminar states the correctness rule clearly: keeping too many candidate files costs extra work; keeping too few can produce a wrong result. Pruning must be conservative. Missing or unusable statistics generally mean less skipping, not permission to omit data.

Table metadata can carry file-level statistics, allowing the engine to eliminate files before fetching their Parquet footers. Footers remain useful for navigating and pruning within selected files.

Column projection is a separate optimization: it avoids columns that the query does not need. File skipping and row-group skipping avoid irrelevant rows or storage units.

## Where the metadata lives

The formats pursue similar goals with different representations.

| Format | High-level metadata model |
|---|---|
| Iceberg | Table metadata, snapshots, manifest lists, and manifests describing data and delete files |
| Delta Lake | A transaction log describing changes, with checkpoints for efficient state reconstruction |
| DuckLake | SQL tables in a transactional database describing snapshots, files, statistics, and other table state |

"Manifest" is an Iceberg term here, not a generic name for Delta's transaction log.

An Iceberg read can follow this structure:

```text
Catalog reference
       |
Table metadata
       |
Snapshot's manifest list
       |
Relevant manifests
       |
Selected data and delete files
```

This is a reusable metadata tree, not necessarily a full traversal on every query. Caching, pruning, and parallel planning affect the actual cost.

DuckLake instead queries database records to identify snapshot-visible candidate files. The database can provide transactions, indexes, joins, and buffer management rather than requiring each engine to recreate those capabilities over metadata files.

The BigQuery architecture described in the seminar uses related ideas: transactional metadata identifies storage objects, and statistics help determine which objects a query needs.

Database-backed metadata can simplify coordination and selective access, but it introduces a database that must be operated and kept available. File-based metadata has different portability and service-dependency tradeoffs. Neither design is universally faster.

The catalog also deserves a distinction: a database-backed catalog may only hold references to file-based metadata. That is not the same as storing the complete file inventory in database tables.

## Small writes and the tiny-file problem

Streaming produces small amounts of data frequently. Columnar storage benefits from larger batches.

Writing every small batch immediately can create:

```text
[tiny] [tiny] [tiny] [tiny] [tiny] ...
```

Each file adds discovery, metadata, request, and scheduling overhead.

One strategy is to publish small files quickly and compact them later. Freshness is good, but maintenance and planning costs rise.

Another is to buffer data before writing larger files. File counts improve, but ordinary file-backed queries may not see the buffered rows yet.

The DuckLake seminar describes inline small writes stored in the catalog database. These rows are a committed, queryable table representation, not merely an uncommitted ingestion buffer. They can later be flushed into Parquet.

Readers must understand both representations and their snapshot visibility. Moving rows from inline storage into files must neither duplicate nor lose them.

A connector that ignores optional pruning can be slower but correct. A connector that ignores inline rows or delete information can be incorrect.

The seminar contrasts this with Iceberg's file-oriented small-write overhead. The other transcript notes that Delta can represent a commit with one metadata file. That reduces metadata overhead; it does not, by itself, eliminate tiny data files.

## Coalescing, repartitioning, and compaction are different operations

In Spark, an execution partition is a unit of distributed work. A table partition, such as month=10, is a storage grouping. These are not interchangeable.

| Operation | Purpose |
|---|---|
| Spark coalesce | Usually reduce execution partitions without a full shuffle |
| Spark repartition | Redistribute rows through a shuffle |
| Table compaction | Rewrite existing small files into fewer, larger files |

Reducing Spark partitions can reduce output-file counts. It does not guarantee one file for each table partition. Repartitioning by table partition columns can group rows by key, but writer limits and other settings can still produce multiple files.

One giant file is not always desirable. Large partitions need enough files for parallel reads. The target is a healthy balance between overhead and parallelism.

The DuckLake speaker also uses "coalescing" for BigQuery maintenance, distinguishing metadata reduction from small-file reduction. That terminology is not a reference to Spark's DataFrame coalesce operation.

## Deletes can trade write work for read work

With copy-on-write, a deletion rewrites affected files without the deleted rows. The new snapshot replaces the old files with those rewritten files.

With merge-on-read, the base files remain unchanged and separate information records the deletes or updates. Readers apply that information.

```text
Copy-on-write:
  Original file -> rewritten file without deleted rows

Merge-on-read:
  Original file + delete information -> visible rows
```

Deletion vectors are one representation; position-based and key-based delete files are others.

Copy-on-write pays more during writes. Deferred changes can reduce write work but add read and maintenance costs. If delete information accumulates indefinitely, reads can degrade. The Iceberg talk discusses improving this situation as part of its format evolution.

## Z-ordering: make file statistics describe smaller regions

Statistics are most useful when related values are grouped together. If every file contains both very small and very large values, each file's range is broad and few files can be skipped.

Z-ordering clusters data across multiple columns. It maps a multidimensional space into a one-dimensional order that tends to preserve locality.

Consider two columns, x and y, each with values from 0 to 3. Each coordinate is a row. A simple Z-order traversal gives these ranks:

```text
                       x
                0     1     2     3

       y = 3   10    11    14    15
       y = 2   08    09    12    13
       y = 1   02    03    06    07
       y = 0   00    01    04    05
```

Read the ranks from 0 through 15. The traversal visits a small 2-by-2 region, then another region, repeating the pattern at larger scales.

For this toy example, the rank comes from interleaving coordinate bits:

```text
x bits:       x1 x0
y bits:       y1 y0
Z-order bits: y1 x1 y0 x0

x = 2 -> 10
y = 1 -> 01

Interleaved bits: 0110 -> rank 6
```

Production implementations need additional handling for real data types and distributions. The example illustrates the geometry.

If four consecutive rows go into each file:

```text
                 x = 0-1       x = 2-3
              +-------------+-------------+
    y = 2-3   |   File C    |   File D    |
              | ranks 8-11  | ranks 12-15 |
              +-------------+-------------+
    y = 0-1   |   File A    |   File B    |
              | ranks 0-3   | ranks 4-7   |
              +-------------+-------------+
```

the statistics become:

| File | x range | y range |
|---|---|---|
| A | 0-1 | 0-1 |
| B | 2-3 | 0-1 |
| C | 0-1 | 2-3 |
| D | 2-3 | 2-3 |

For this query, only C remains a candidate:

```sql
SELECT *
FROM points
WHERE x BETWEEN 0 AND 1
  AND y BETWEEN 2 AND 3;
```

Compare that with files containing scattered points:

```text
File A: (0,0), (1,1), (2,2), (3,3)
File B: (0,1), (1,0), (2,3), (3,2)
File C: (0,2), (1,3), (2,0), (3,1)
File D: (0,3), (1,2), (2,1), (3,0)
```

Every file now has x range 0-3 and y range 0-3. The same predicate cannot exclude any file.

**Z-ordering changes the physical layout, not the answer. It makes statistics more selective.**

It is not an exact lookup index or a perfect nearest-neighbor ordering. Reorganizing files costs work, locality weakens as more dimensions are added, and simple sorting can be sufficient for predominantly one-column predicates.

Partitioning and clustering can also work together: partition broadly by time, then cluster within that layout by commonly filtered columns.

## Shared data, specialized compute

Iceberg's original motivation was reliable SQL-table behavior across several engines. Its larger architectural consequence was a common dataset that different systems could share without maintaining separate synchronized copies.

```text
Spark --------+
Flink --------+
SQL engine ---+--> Shared catalog and committed table state
Python -------+                 |
                         Shared data files
```

This enables centralized data with specialized compute. An engine can be chosen for a workload without first moving the authoritative dataset into that engine's private storage.

This is compute/storage disaggregation, often described as shared-storage architecture. Workers can still have private memory and local caches. "Shared-nothing" does not describe the shared storage layer itself.

Interoperability requires more than reading Parquet. Engines must understand column identities, snapshots, deletes, and other table features. Read support and write support may differ.

Governance also needs separate enforcement. Raw file access can bypass a query engine's row-level policies. Catalog permissions, scoped storage credentials, and trusted execution paths must cooperate. A table format alone does not enforce arbitrary row-level access.

## How the two talks fit together

The Iceberg talk begins with correctness: atomic replacement, safe schema evolution, and coordination across engines. Its conclusion is that reliable shared storage reduces copying, synchronization, and engineering effort.

The DuckLake talk begins with similar goals but focuses on metadata representation. Its argument is that a transactional database can simplify planning, caching, concurrent commits, and small-write handling.

The Iceberg speaker also acknowledges metadata overhead and proposes adaptive metadata structures, cheaper small commits and reads, and improved metadata filtering. These are proposals from the recording, not verified current support. Performance comparisons and managed-service plans in the DuckLake talk likewise remain speaker claims rather than universal guarantees.

Both perspectives converge on the same idea: data and metadata need active management, and a table should behave predictably regardless of which engine accesses it.

## The big picture

A data lake provides scalable file storage. An open table format defines how those files form a consistent, evolving table. A lakehouse is the broader architecture combining that storage with engines, transactions, optimization, maintenance, and governance.

The core mental model is:

> Immutable files hold the data. Committed metadata defines its meaning and visibility. Query planning uses that metadata to read only what might be needed.

Correctness determines which data may be read. Physical organization and statistics determine how much of that data actually needs to be touched.

## Sources

Primary source material supplied in this conversation:

- Jordan Tigani's DuckLake seminar in Carnegie Mellon University's Future Data Systems series, supplied as `pasted-text-28a2a2a4-123d-4158-b4c8-cb1aa6bcce7c.txt`.
- The Iceberg co-creator's talk on Iceberg's origins, reliable shared storage, and proposed evolution, supplied as `pasted-text-9b64ee52-344d-4ced-8fc9-9a8222bd0297.txt`.

Official documentation consulted while preparing the explanations:

- [HDFS architecture](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)
- [Parquet file format](https://parquet.apache.org/docs/file-format/)
- [Iceberg partitioning](https://iceberg.apache.org/docs/latest/partitioning/)
- [Iceberg reliability and concurrent writes](https://iceberg.apache.org/docs/latest/reliability/)
- [Delta transaction protocol](https://github.com/delta-io/delta/blob/master/PROTOCOL.md)
- [Delta optimization and data skipping](https://docs.delta.io/optimizations-oss/)
- [Spark DataFrame coalesce](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.coalesce.html)
