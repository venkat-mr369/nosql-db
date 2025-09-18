 **detailed troubleshooting steps for Tombstones in Cassandra**.

---

# 🛠 Tombstone Troubleshooting Guide (Production)

## 1. 🔍 Detecting Tombstones

### (a) Using `nodetool cfstats`

Run:

```bash
nodetool cfstats keyspace_name.table_name
```

Look for:

* **Tombstone Scanned Per Read** → If high (e.g., 1000s), queries are scanning too many tombstones.
* **Average/Maximum Tombstone Count**.

👉 If values are large, queries are slow due to tombstones.

---

### (b) Enable Tracing in CQL

```sql
CONSISTENCY QUORUM;
TRACING ON;

SELECT * FROM user_activity WHERE user_id=uuid();
```

In trace output, check:

* `Read n live and m tombstoned cells`
* If `m >> n`, the query is tombstone-heavy.

---

### (c) Monitor Logs

Look at `system.log`:

```bash
grep -i tombstone /var/log/cassandra/system.log
```

Common warnings:

```
Read 100000 live rows and 500000 tombstone cells for query ...
```

⚠️ Indicates serious performance risk.

---

## 2. ⚠️ Common Causes of Excess Tombstones

1. **TTL on Large Partitions**

   * Millions of TTL-expired rows pile up tombstones.
2. **Deletes on Wide Partitions**

   * Example: deleting entire user history in one query.
3. **Frequent Updates**

   * Every update is stored as a new cell + old cell becomes a tombstone.
4. **Misuse of Collections (`set`, `list`, `map`)**

   * Removing items from collections generates tombstones.
5. **`ALLOW FILTERING` Queries**

   * Can scan entire partition including tombstones.

---

## 3. ✅ Fixing Tombstone Issues

### (a) Schema Design Fixes

* Avoid **wide partitions** → use compound keys (time buckets).
* Example (bad vs good):

```sql
-- BAD: all logs for a user in one partition
PRIMARY KEY (user_id, activity_time);

-- GOOD: bucket by day
PRIMARY KEY ((user_id, activity_date), activity_time);
```

### (b) Use TTL Carefully

* Set TTL only on data that really needs expiry.
* Avoid huge TTL-based deletes on large datasets.

### (c) Tune `gc_grace_seconds`

* Default = **10 days**.
* Lower in **dev/test** where quick cleanup is fine.
* Keep high in **prod** to prevent zombie data after repairs.

Example:

```sql
ALTER TABLE user_activity WITH gc_grace_seconds = 86400; -- 1 day
```

---

### (d) Run Repairs Regularly

* Use `nodetool repair` to propagate tombstones across replicas before GC deletes them.
* Automate with cron:

```bash
0 2 * * 0 nodetool repair --full keyspace_name
```

---

### (e) Trigger Manual Compaction (if disk pressure)

If too many tombstones remain:

```bash
nodetool compact keyspace_name table_name
```

⚠️ Use carefully — compaction is CPU and I/O heavy.

---

### (f) Monitor & Alert

* Use Prometheus/Grafana to watch:

  * **Tombstone Scans per Read**
  * **Read Latency**
  * **Compaction Pending Tasks**

---

## 4. 🎯 Example Production Case

### Scenario:

* `user_activity` stores 1 year of logs with `TTL=30 days`.
* After 6 months, queries slow → log shows:

  ```
  Read 500 live cells and 25000 tombstone cells for query ...
  ```

### Steps Taken:

1. Verified with `nodetool cfstats` → High tombstone scans.
2. Changed schema: partition logs by **user + day**.
3. Reduced `gc_grace_seconds` from 10 days → 2 days.
4. Scheduled weekly `nodetool repair`.
5. Ran one-time `nodetool compact` to purge old data.

👉 Result: Latency dropped from **3 sec → 100 ms**.

---

# ✅ Key Takeaways

* **Tombstones are necessary** for deletes & TTL, but too many kill performance.
* Use **right schema**, **right TTL**, and **repairs + compaction** to manage them.
* Always monitor **tombstone scans per read** and **system logs**.

---
