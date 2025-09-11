
Cassandra’s storage engine is **partition-based, log-structured, and immutable**. 

---

# Example table

```sql
CREATE KEYSPACE empdb 
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

CREATE TABLE empdb.emp (
    emp_id text, 
    emp_name text, 
    emp_dept text, 
    emp_salary int, 
    PRIMARY KEY (emp_id)
);
```

This means:

* **Partition key = emp\_id**
* Each `emp_id` → **one partition**.
* Inside each partition, the rows are ordered by clustering columns (if any; here none, so one row per partition).

---

# Internal storage structures in Cassandra

### 1. **Partition**

* A partition is the fundamental unit of storage.
* Each partition is identified by its **partition key** (here `emp_id`).
* A partition can contain **one or many rows** depending on clustering columns.
* All rows for a partition are always stored together on the same replica set.

💡 In our example: `emp_id='E101'` is one partition; inside it you store the employee row (name, dept, salary).

---

### 2. **Commitlog**

* First place a write goes.
* Append-only log file on disk for durability.
* Ensures data is not lost even if the node crashes before flushing to disk.
* Shared across all keyspaces/tables in that node.

---

### 3. **Memtable**

* In-memory (RAM) structure specific to a **table**.
* Each table has its own memtable.
* Stores recently written rows/partitions (indexed by partition key).
* Fast reads for hot data before flush.

---

### 4. **SSTable (Sorted String Table)**

* Immutable on-disk files created when a memtable is flushed.
* Each SSTable contains:

  * **Data file (.db)** → actual rows.
  * **Index file** → maps partition key → position in data file.
  * **Summary file** → sampled index for faster lookups.
  * **Bloom filter** → probabilistic structure to quickly check if partition may exist in this SSTable.
  * **Compression info** → for efficient disk use.

💡 Multiple SSTables may hold different versions of the same row. Reads merge them.

---

### 5. **Compaction**

* Process of merging SSTables (to reduce the number of files and purge tombstones).
* After compaction, older redundant versions of rows are discarded.
* Helps keep reads efficient.

---

### 6. **Tombstones**

* Special markers for deletions.
* Stored like normal writes (in commitlog, memtable, SSTable).
* During compaction, tombstones remove older data versions.

---

# Storage flow for a write (example row insert)

```sql
INSERT INTO empdb.emp (emp_id, emp_name, emp_dept, emp_salary)
VALUES ('E101', 'Alice', 'HR', 50000);
```

Timeline:

1. Client sends write to coordinator.
2. Replica nodes (RF=3) that own `hash(E101)` receive mutation.
3. On each replica:

   * Append mutation → **commitlog** (durable).
   * Update **memtable** for `empdb.emp`.
   * Acknowledge back.
4. Later:

   * Memtable flushes → creates a new **SSTable** for `empdb.emp`.
   * Commitlog segments that are fully flushed can be deleted.
   * Over time, multiple SSTables merge via **compaction**.

---

# Storage layout on disk (per node)

Example data directory path:

```
/var/lib/cassandra/data/empdb/emp-<UUID>/
```

Inside this directory you’ll see files like:

```
mc-1-big-Data.db        # SSTable data file
mc-1-big-Index.db       # partition index
mc-1-big-Summary.db     # sampled index
mc-1-big-Filter.db      # bloom filter
mc-1-big-CompressionInfo.db
...
```

Each flush/compaction produces new files with incrementing IDs.

---

# Node-level storage description

* **Each node only stores partitions for token ranges it owns.**
* With RF=3, each partition is stored on **3 nodes**.
* Data is evenly distributed by Murmur3 hash of partition key.
* So if you insert 100 employees, you will have 100 partitions spread across the 7 nodes, each partition replicated 3×.

---

# Use case explanation

* If you design **partition key as emp\_id** → 1 row per partition → great for point lookups.
* If you design **partition key as dept, clustering key as emp\_id** → partitions per dept, rows clustered by emp\_id.

  * e.g., HR partition stores all HR employees together.
  * Useful for queries like "list all HR employees."
  * Partition count = number of depts.

---

✅ So: Cassandra stores data **partitioned by partition key, durable via commitlog, buffered in memtables, flushed as SSTables, and compacted over time.**
Each table lives in its own directory with SSTable components, but commitlog is global.

---

👉 Do you want me to **draw a flowchart-style text diagram** (like arrows showing `Client → Coordinator → Commitlog → Memtable → SSTable → Compaction`) for this table? That might make the storage structure even easier to visualize.
