i have **100 big tables, each \~100 GB**, and each table contains three logical parts you called `tbl1`, `tbl2`, `tbl3`. I have to finesh, a safe schedule that finishes all 100 tables in about a week, and the exact commands to run.

---

# Quick summary

You have many large tables and too many *tombstones* (deleted markers). Tombstones slow reads and use CPU/IO.
Do many **small, targeted** compactions (UDC style) on a few tables at a time, on a schedule that spreads load across the 15 nodes. Don’t do big “major” compactions on many nodes at once — that will cause CPU and IO spikes.

---

# Estimation

* 15-node Cassandra cluster.
* 100 tables total. Each table is \~100 GB and contains three sub-areas (you called them `tbl1`, `tbl2`, `tbl3`).
  (I treat each top-level table as one compaction unit — if `tbl1/tbl2/tbl3` are separate Cassandra tables, treat them as extra items.)
* We will run **5 UDC windows per day** (like Bell), with **3 nodes in each window** (so 3 nodes compact at a time).
* We will compact **one table per node per window** to be safe. That gives **15 table compactions per day** → \~**7 days** to cover 100 tables.

---

# Real-world use case (why we do this)

* Problem: deletes and TTLs create tombstones. When queries scan data, Cassandra may read through many tombstones and slow down.
* Goal: remove old tombstones quickly and steadily so reads stay fast and CPU/IO stay stable.
* Why UDC: It’s surgical — you compact only the SSTables that have lots of tombstones (low blast), instead of merging everything at once.

---

# Simple schedule (how to spread work across nodes)

* 5 windows per day (pick low-traffic times): e.g. 00:30, 06:30, 12:30, 18:30, 22:30
* 15 nodes → group nodes into 5 groups of 3 nodes each (round-robin).
* Each window runs on one node group (3 nodes) and compacts 1 table per node.
* Result: 3 compactions per window × 5 windows = 15 compactions/day → \~7 days to hit 100 tables.

Example daily plan:

* Day 1 compacts tables 1–15
* Day 2 compacts tables 16–30
* ... Day 7 completes tables 91–100

(If you need to finish faster, increase the number of tables per node per window — but only after you verify CPU/IO is safe.)

---

### Step-by-step runbook

#### 1) Preflight checks (before touching a node)

SSH into the node:

```
ssh cassandra@vm01
```

Then run these checks and *only continue if all look OK*:

* Check node state:

```
nodetool status
```

Target node must be `UN` (Up / Normal).

* Check node info (heap, uptime):

```
nodetool info
```

* Check compaction and thread pools:

```
nodetool compactionstats
nodetool tpstats
```

* Check table tombstone stats for a candidate table:

```
nodetool tablestats <KEYSPACE> <TABLE>
```

Look for lines mentioning **tombstones**, **sstables**, **mean partition size**.

* Check CPU, memory, disk:

```
top -b -n1 | head -n 20
df -h /var/lib/cassandra
free -m
```

**Skip the node now (do not run compaction) if:**

* CPU is above **85%** sustained
* Disk free is under **15–20%**
* Active compactions > **3**
* Node not `UN` or gossip unstable

---

#### 2) Pick tables to compact (per node)

* Choose **1 table per node** for each window. Start with the tables that:

  * Show high tombstone counts in `nodetool tablestats`
  * Have lots of deletes or TTLs (time-series tables)
* Don’t choose very large tables with hundreds of SSTables in one go. If a table is huge, schedule it for a surgical SSTable-level compaction (see below).

---

#### 3) Prepare the table (flush memtables)

Before compacting, flush the table so SSTable files are stable:

```
nodetool flush <KEYSPACE> <TABLE>
```

Why? Flushing writes in-memory data to disk so the list of SSTables won’t change while you compact.

---

#### 4) Run compaction (choose one method)

#### A — Table-level (simple, safe)

Run one table at a time:

```
nodetool compact <KEYSPACE> <TABLE>
```

This compacts SSTables for that table on that single node. Monitor while it runs.

#### B — User-Defined SSTable compaction (surgical — preferred for heavy tables)

If a table is very big, prefer to compact only certain SSTable files (old ones with tombstones). This uses JMX `CompactionManager.forceUserDefinedCompaction(...)`. Steps (manual):

1. Find candidate SSTables on the filesystem:

```
ls -ltr /var/lib/cassandra/data/<KEYSPACE>/<TABLE>-*/  # look for old ma-*.db files
```

2. Use a JMX client (like `jmxterm`) and call `forceUserDefinedCompaction()` with those SSTable file paths. **Test this in dev first.**

> NOTE: JMX method is safest and lowest-blast, but needs testing and JMX access.

#### Avoid: Major compaction (merging everything). It is heavy and should be used only in special maintenance windows.

---

#### 5) Monitor while compaction runs

Keep checking:

```
nodetool compactionstats
nodetool tpstats
tail -n 200 /var/log/cassandra/system.log
top -b -n1 | head -n 20
df -h /var/lib/cassandra
```

If CPU or IO goes too high or pending compactions grow, stop scheduling more windows and investigate.

---

#### 6) After compaction — quick checks

* Check table stats:

```
nodetool tablestats <KEYSPACE> <TABLE>
```

You should see fewer SSTables and fewer tombstones read in histograms.

* Check application read latency (APM or DB metrics) — it should improve if tombstones were a problem.

* Record results: node name, table name, start/end time, CPU/IO, whether improvement found.

---

## 7) Repair note (important for deletes)

After tombstones get purged, make sure deletes don’t come back (zombie rows). Keep repairs in your schedule. Example:

```
nodetool repair <KEYSPACE> <TABLE>
```

And check `system_auth` replication if role creation failed earlier:

```
DESCRIBE KEYSPACE system_auth;
nodetool repair system_auth
```

---

# Compaction types — plain language

* **Automatic minor compactions**: Cassandra does these by itself in the background.
* **Table-level (`nodetool compact`)**: compacts all SSTables for that table on one node. Easy to run, but can be heavy if table has many SSTables.
* **User-Defined Compaction (UDC)** via JMX: you pick specific SSTable files to compact — very targeted and low-load.
* **Major compaction**: compacts everything into one file — very heavy, avoid in production.
* **Compaction strategies (policy):**

  * `STCS` — good for heavy writes
  * `LCS` — good for read-latency sensitive small partitions
  * `TWCS` — good for TTL/time-series data (recommended for time-series tables)

If your tables have TTLs/time-series data, consider `TWCS` so old data is removed more naturally.

---

# Safety & throttle controls (if things look bad)

If compaction causes overload, do this:

* Stop further runs for the day.
* Lower compaction throughput:

```
nodetool setcompactionthroughput 20000   # KB/s, lower than default to slow compaction
```

* Consider lowering concurrent compactions (requires config change and a safe restart):

  * `concurrent_compactors` in `cassandra.yaml` — change in a maintenance window.
* Let running compactions finish; don’t start new ones until safe.

---

# How long to finish 100 tables? (concrete plan)

* With 15 compactions per day (3 nodes per window × 5 windows), you compact **15 tables/day**.
* 100 tables ÷ 15/day = **\~7 days** to cover everything once.
* Example daily table ranges:

  * Day 1: tables 1–15
  * Day 2: tables 16–30
  * ...
  * Day 7: tables 91–100

If you must finish faster, increase compactions per node per window — but first check CPU/IO headroom.

---

### Production operator checklist 

1. Identify node group and tables for this window.
2. SSH to node: `ssh cassandra@<node>`
3. Preflight: `nodetool status`, `nodetool info`, `nodetool compactionstats`, `nodetool tpstats`, `nodetool tablestats <KS> <TABLE>`, `top`, `df -h`
4. If safe, flush table: `nodetool flush <KS> <TABLE>`
5. Compact:

   * Simple: `nodetool compact <KS> <TABLE>`
   * Surgical (advanced): pick SSTables and use JMX `forceUserDefinedCompaction()`
6. Monitor: `nodetool compactionstats`, `nodetool tpstats`, `tail -f /var/log/cassandra/system.log`
7. When done: `nodetool tablestats <KS> <TABLE>` + record results
8. Move to next node (stagger by some minutes). If thresholds hit, stop and investigate.

---

### Final tips (short)

* Start conservative: 1 table per node per window.
* Prioritize TTL/time-series and high-delete tables first.
* Use UDC (SSTable-level) for very large tables — it’s safer.
* Run repairs regularly so deletes are not resurrected.
* Test everything first in a staging environment.

---

