
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
here you can find the **write path**, **read path**, **flush → SSTable**, **compaction / tombstones**, **hinted handoff & repair**, and **startup / commitlog replay**. 

Assume with your environment:

* keyspace `empdb`, table `emp` with `PRIMARY KEY (emp_id)` (one partition per `emp_id`).
* single-node steps apply per replica (each replica executes the same local steps).
* typical data paths: `/var/lib/cassandra/commitlog`, `/var/lib/cassandra/data/<keyspace>/<table-UUID>/`, `/var/lib/cassandra/hints` (paths may vary by distro).

# Flow — high-level ASCII diagram

Client → Coordinator → \[Replica1, Replica2, Replica3]
Each Replica: Commitlog append → Memtable update (in-memory) → (later) Memtable flush → SSTable file(s) on disk
Reads: Coordinator → Replicas → Merge(memtable + SSTables + tombstones) → (optional read-repair) → client

---

# 1) WRITE PATH — step-by-step (with validation commands)

1. Client computes partition token

   * Driver hashes `emp_id` (Murmur3) → token `T`.
   * Client issues:

     ```sql
     INSERT INTO empdb.emp (emp_id, emp_name, emp_dept, emp_salary) VALUES ('E101','Alice','HR',50000);
     ```
   * Validate token and endpoints:

     ```bash
     # Which nodes are replicas for this partition?
     nodetool getendpoints empdb emp "'E101'"
     # Or in cqlsh see the token:
     SELECT token(emp_id) FROM empdb.emp WHERE emp_id='E101';
     ```

2. Coordinator determines replica nodes and forwards mutation

   * Coordinator sends mutation RPCs to all replicas (RF copies).
   * Coordinator waits for required acknowledgements depending on CL (e.g., QUORUM needs majority).

3. On each replica: append to commitlog (durable) **then** update memtable (in-memory)

   * Order: **commitlog append → memtable update** (ensures replay on crash).
   * Commitlog is append-only; segmented files live in commitlog dir.
   * Validate:

     ```bash
     # Check commitlog directory
     ls -lh /var/lib/cassandra/commitlog
     # Look for commit activity in system logs
     grep -i "CommitLog" /var/log/cassandra/system.log
     ```
   * Memtable updated (table-specific in-memory structure). Validate memtable stats:

     ```bash
     nodetool tablestats empdb emp
     # look for MemtableColumnsCount, MemtableDataSize, MemtableSwitchCount
     # Older versions: nodetool cfstats empdb.emp
     ```

4. Replica ACKs the coordinator

   * Once enough replicas ack (per CL), client gets success.
   * At this moment the data is durable (in commitlog) and readable (from memtable).

---

# 2) MEMTABLE → FLUSH → SSTABLE (on-disk lifecycle)

1. What triggers a memtable flush?

   * Memtable reaches configured size (heap/off-heap limits).
   * memtable\_flush\_period\_in\_ms (time-based) triggers, node shutdown, or explicit `nodetool flush`.
   * When flush happens: memtable contents are written to an **immutable SSTable**.

2. SSTable components (per flush)

   * Data file (`-Data.db`) — actual rows
   * Partition index (`-Index.db`)
   * Summary (`-Summary.db`)
   * Bloom filter (`-Filter.db`)
   * Statistics, TOC, CompressionInfo (if compression enabled)
   * Files exist under:

     ```
     /var/lib/cassandra/data/empdb/emp-<UUID>/
     ```
   * Validate SSTable files:

     ```bash
     ls -lh /var/lib/cassandra/data/empdb/emp-*/  # shows SSTable component files
     nodetool tablestats empdb emp                 # shows "Number of SSTables" and space used
     ```

3. Inspect SSTable contents for a partition

   * Find which SSTables contain your partition:

     ```bash
     nodetool getsstables empdb emp 'E101'
     # returns list of SSTable files that contain the partition
     ```
   * Dump SSTable JSON (slow but exact):

     ```bash
     sstabledump /var/lib/cassandra/data/empdb/emp-<UUID>/ma-*-Data.db | jq .    # or grep for "E101"
     ```
   * Or use `sstablemetadata` to inspect SSTable summaries:

     ```bash
     sstablemetadata /var/lib/cassandra/data/empdb/emp-<UUID>/ma-*-Data.db
     ```

---

# 3) READ PATH — exact inner steps (how Cassandra finds the row)

1. Client issues `SELECT * FROM empdb.emp WHERE emp_id='E101';` (with a consistency level)
2. Coordinator maps partition token → replica list → contacts replicas
3. On each contacted replica:

   * Check **memtable** first (fast in-RAM lookup).
   * For each SSTable on disk:

     * Check **Bloom filter** to skip SSTables that definitely don’t contain partition (fast)
     * If bloom filter says “maybe”, consult the **partition index** (Summary + Index) to seek into Data file and load the partition (seek I/O)
   * Merge results from memtable + 0…N SSTables and apply timestamps to pick latest values (LWT/Lightweight transactions aside).
4. If coordinator detects differences across replicas it can issue **read repair** (in-band) or schedule a background repair (out-of-band).
5. Coordinator returns the resolved row to client.

Validate read internals:

```bash
# Run a read at a chosen consistency:
cqlsh> CONSISTENCY QUORUM;
cqlsh> SELECT * FROM empdb.emp WHERE emp_id='E101';

# See per-table bloom stats and read metrics:
nodetool tablestats empdb emp    # look for BloomFilterFalsePositives, BloomFilterFalseRatio
nodetool compactionstats        # may show read repair activity
nodetool netstats              # shows streaming/read-repair info during repair/stream
```

---

# 4) DELETE / TOMBSTONE lifecycle

1. Delete creates a **tombstone** (special marker) with a timestamp.
2. Tombstones are written to commitlog & memtable → flushed into SSTables like normal writes.
3. Tombstones prevent deleted data from being returned during reads (they mask older cell values).
4. Tombstones are removed permanently only after a compaction that is past `gc_grace_seconds` (default often 864000 sec or 10 days—check your config).

   * This period ensures replicas that were down can be repaired before tombstones are removed.
5. Validation:

```bash
# See tombstone metrics (tablestats includes tombstone-related counts)
nodetool tablestats empdb emp | egrep -i 'Tombstone|tombstone|SSTable count'

# Inspect SSTables for tombstones via sstabledump (search for tombstone markers)
sstabledump /path/to/SSTable-Data.db | jq 'select(.tombstone == true)'
```

---

# 5) COMPACTION — how SSTables are merged & tombstones purged

* Background compaction merges older SSTables into fewer SSTables, resolves cell-level versions (based on timestamps), and may purge tombstones if older than `gc_grace_seconds`.
* Types: SizeTieredCompaction (STCS), LeveledCompaction (LCS), TimeWindowCompaction (TWCS) — chosen by table compaction strategy.
* Validation & control:

```bash
# See compaction activity
nodetool compactionstats

# Trigger a manual compaction (use with caution in production)
nodetool compact empdb emp
```

---

# 6) HINTED HANDOFF & REPAIR (when a replica was down)

* If a replica is down during a write, the coordinator may keep a **hint** for that replica (if hinted handoff enabled). Hints live in hints directory and are sent when replica comes back.
* If hints are insufficient (or disabled), `nodetool repair` is used to synchronize data across replicas; repair performs anti-compaction and streaming.
* Validation:

```bash
nodetool netstats         # shows hint delivery / streaming activity
ls -lh /var/lib/cassandra/hints
nodetool repair empdb    # repairs the keyspace (can be scoped)
```

---

# 7) STARTUP / COMMITLOG REPLAY (node recovery)

* On node restart, Cassandra replays any un-flushed mutations from the commitlog to rebuild memtables prior to accepting requests (ensures no committed writes are lost).
* Validation:

  * Check system logs for "Starting commitlog replay" / "Replayed commitlog segments".
  * After startup, `nodetool status` should show node `UN` (Up/Normal).

```bash
nodetool status
grep -i "commitlog" /var/log/cassandra/system.log
```

---

# 8) ON-DISK FILES & PATHS — quick reference (what you will actually see on disk)

```
/var/lib/cassandra/commitlog/
  CommitLog-<seg>.log            # append-only segments

/var/lib/cassandra/data/empdb/emp-<table-UUID>/
  mc-<N>-Data.db                 # SSTable Data
  mc-<N>-Index.db                # Partition index
  mc-<N>-Summary.db              # Index summary samples
  mc-<N>-Filter.db               # Bloom filter
  mc-<N>-Statistics.db
  mc-<N>-CompressionInfo.db
```

Inspect with:

```bash
ls -lh /var/lib/cassandra/data/empdb/emp-*/
sstabledump /var/lib/cassandra/data/empdb/emp-*/ma-*-Data.db | less
```

---

# 9) Useful configuration knobs (what to tune / watch)

* `commitlog_sync` = `periodic` or `batch` (controls fsync behavior / durability).
* `commitlog_sync_period_in_ms` (if periodic) — how often commitlog fsyncs.
* `memtable_heap_space_in_mb` / `memtable_offheap_space_in_mb` — memtable sizing.
* `memtable_cleanup_threshold` — when memtables get aggressively cleaned up.
* `gc_grace_seconds` — how long tombstones are retained before being purged.
* `compaction` strategy for the table (STCS / LCS / TWCS) — impacts write amplification, read latency, disk space.

(Always check your `cassandra.yaml` and table properties for exact names/values before tuning.)

---

# 10) Quick validation checklist — exact commands mapped to steps

* Which nodes are replicas for a partition:

  ```bash
  nodetool getendpoints empdb emp "'E101'"
  ```

* After write (durable in commitlog + visible in memtable):

  ```bash
  nodetool tablestats empdb emp
  cqlsh> CONSISTENCY QUORUM; SELECT * FROM empdb.emp WHERE emp_id='E101';
  ls -lh /var/lib/cassandra/commitlog
  ```

* Force flush → create SSTable, inspect SSTable:

  ```bash
  nodetool flush empdb emp
  nodetool tablestats empdb emp   # SSTable count increases
  nodetool getsstables empdb emp 'E101'
  sstabledump /var/lib/cassandra/data/empdb/emp-*/ma-*-Data.db | jq '.'
  ```

* See compaction & tombstone activity:

  ```bash
  nodetool compactionstats
  nodetool tablestats empdb emp | egrep -i 'tombstone|Tombstone|SSTable'
  nodetool compact empdb emp     # forcing compaction (careful)
  ```

* Hinted handoff / repair:

  ```bash
  nodetool netstats
  ls -lh /var/lib/cassandra/hints
  nodetool repair empdb
  ```

* Startup / commitlog replay:

  ```bash
  grep -i "commitlog" /var/log/cassandra/system.log
  nodetool status
  ```

---

# Practical tips & gotchas (short)

* **Partition = partition-key**. One partition per distinct PK. Keep partitions bounded in size (MBs, not GBs).
* **Avoid hot partitions** (one PK receiving huge traffic) — use bucketing.
* **Tombstone storms** (mass deletes) can kill read performance; stagger deletes or use TTLs with care.
* **Compaction & repair** are I/O heavy — schedule during low traffic; ensure enough disk headroom (compaction needs space).
* **Check memtable & commitlog** sizes frequently after heavy writes — they’ll tell you when to tune.

---


