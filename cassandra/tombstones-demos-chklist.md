**Test tombstones step by step** 

---

# 🔬 Tombstone Deep Dive Test

We’ll do this test on **any one node in the cluster** (for convenience you can pick a coordinator like `10.166.0.7`).
👉 Because Cassandra is distributed, the tombstones will **replicate automatically** to the other nodes based on your replication factor.

---

## 1. Create a Keyspace with Replication

Run in `cqlsh` (from any node):

```sql
CREATE KEYSPACE tombstonetest
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 2,
  'dc2': 2
} AND durable_writes = true;
```

👉 This ensures each row will be replicated to 2 nodes in **dc1** and 2 nodes in **dc2**.

---

## 2. Create a Test Table

```sql
USE tombstonetest;

CREATE TABLE users (
   id UUID PRIMARY KEY,
   name text,
   city text
) WITH gc_grace_seconds = 600; -- keep tombstones only for 10 minutes
```

---

## 3. Insert Sample Data

```sql
INSERT INTO users (id, name, city) VALUES (uuid(), 'vijaya', 'Hyderabad');
INSERT INTO users (id, name, city) VALUES (uuid(), 'manju', 'Chennai');
INSERT INTO users (id, name, city) VALUES (uuid(), 'bob', 'Delhi');
```

---

## 4. Delete a Row → Create a Tombstone

```sql
SELECT * FROM users; -- copy one UUID

DELETE FROM users WHERE id=<uuid-of-Bob>;
```

Now Cassandra writes a **tombstone marker** instead of immediately deleting data.

---

## 5. Verify Tombstone with `nodetool tablestats`

Run from **any node**:

```bash
nodetool tablestats tombstonetest.users
```

Look for:

* **Tombstone count** (increasing after delete).
* **Droppable tombstone ratio** (when >0, compaction may purge them).

---

## 6. Inspect SSTables Directly

On the node where data is stored, go to:

```
/data/cassandra/data/tombstonetest/users-d04753e09b8011f0a46fd3b7941e88b0/
```
To flust from commit:
```
nodetool flush tombstonetest users
```

👉 Each SSTable = data file.
To inspect tombstones:

```bash
sstabledump /data/cassandra/data/tombstonetest/users-*/nb-*-Data.db | grep -i tombstone
```

You will see **deletion markers**.

---

## 7. Wait for GC Grace Expiry + Compaction

* Tombstone lives for `gc_grace_seconds` (here = 600 sec).
* After a **major compaction**:

```bash
nodetool compact tombstonetest users
```

The deleted row will finally disappear from disk.

---

## 8. Test Zombie Safety

1. Shut down one node in dc2 (`10.138.0.4`).
2. Perform a delete in dc1.
3. Bring the dc2 node back **within gc\_grace\_seconds**.

   * It will receive the tombstone via repair.
   * If you bring it back **after** gc\_grace\_seconds and compaction, the zombie risk exists.

---

# 📂 `/data/cassandra/data/` Summary

* Location where **all keyspace/table SSTables** are stored.
* Structure:

  ```
  /data/cassandra/data/<keyspace>/<table>-<UUID>/
  ```
* Inside you’ll see:

  * `*-Data.db` → actual data (including tombstones until purged).
  * `*-Index.db`, `*-Summary.db`, `*-TOC.txt` → metadata/index.
* Each node keeps its **own copy** of data it is responsible for (based on tokens & replication factor).

---

✅ So you should:

* Run `cqlsh` commands from **any node**.
* Run `nodetool` + `sstabledump` from the node that physically stores the SSTables (you can check which nodes with:

```bash
nodetool getendpoints tombstonetest users <uuid>
```

This shows exactly **which nodes have that row**.

---

Perfect 👍 — let’s build a **real Cassandra lab test plan** for your 2-DC, 5-node cluster. We’ll create a keyspace (`toms`), a test table, then insert, delete, flush, and inspect SSTables to actually see **tombstones in action**.

---

# 🔹 Step 1. Create keyspace and table

Connect with `cqlsh` on any node (example: `vm06`).

```sql
CREATE KEYSPACE toms 
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 2,
  'dc2': 2
};

USE toms;

CREATE TABLE users (
    id UUID PRIMARY KEY,
    name text,
    city text
);
```

👉 Keyspace `toms` is replicated across both DCs.

---

# 🔹 Step 2. Insert rows

```sql
INSERT INTO users (id, name, city) VALUES (uuid(), 'Ravi', 'Delhi');
INSERT INTO users (id, name, city) VALUES (uuid(), 'Yasoda', 'Hyderabad');
INSERT INTO users (id, name, city) VALUES (uuid(), 'Anjali', 'Chennai');
```

Check:

```sql
SELECT * FROM users;
```

---

# 🔹 Step 3. Create tombstones

### A. Hard delete

Delete Yasoda:

```sql
DELETE FROM users WHERE id = <uuid_of_yasoda>;
```

### B. TTL expiry

```sql
INSERT INTO users (id, name, city) VALUES (uuid(), 'TTLUser', 'Pune') USING TTL 30;
```

👉 Wait at least 40 seconds so it expires.

---

# 🔹 Step 4. Flush to SSTables

On each replica node (e.g., `vm06` in dc2):

```bash
nodetool flush toms users
```

---

# 🔹 Step 5. Locate SSTables

On `vm06`, go into the data directory:

```bash
cd /data/cassandra/data/toms/users-*/
ls -ltr | grep Data.db
```

You’ll see files like:

```
nb-1-big-Data.db
nb-2-big-Data.db
nb-3-big-Data.db
```

Pick the **newest** one (last modified).

---

# 🔹 Step 6. Dump and search for tombstones

```bash
cd /usr/share/cassandra/tools/bin   # or wherever sstabledump is
./sstabledump /data/cassandra/data/toms/users-<UUID>/nb-3-big-Data.db | grep -i tombstone
```

You should now see something like:

```json
"tombstone": { "markedForDeleteAt": 1696512345678, "localDeletionTime": 1696512345 }
```

---

# 🔹 Step 7. Observe compaction effect

1. Run a manual compaction:

   ```bash
   nodetool compact toms users
   ```
2. After compaction, check again with `sstabledump`. The deleted row + tombstone should both be gone.

---

# 🔹 Step 8. Multi-node consistency

Since you have **dc1 + dc2**, repeat **Step 5–7** on at least one node in each DC (for example `vm02` in dc1 and `vm06` in dc2).
This shows that tombstones replicate across DCs, and compaction on each node eventually clears them.

---

# ✅ Expected flow in your test

1. Insert Ravi, Yasoda, Anjali → SSTable with live rows.
2. Delete Yasoda → new SSTable with a tombstone.
3. Insert TTLUser (expires) → tombstone created after TTL expiry.
4. Flush → tombstones written to SSTable files.
5. `sstabledump` → shows tombstones.
6. Compaction → merges old data + tombstones → deleted rows gone.

---

**step-by-step test matrix**
Great 👍 — here’s a **step-by-step Cassandra tombstone lab test matrix** you can run gradually across your **5-node, 2-DC cluster (dc1 + dc2)**. This will help you **see inserts, deletes, TTL expiries, tombstones in SSTables, and compaction cleanup** clearly.

---

# 🔹 Test Matrix Plan (Day by Day)

## **Day 1 – Setup & Inserts**

1. Create keyspace & table:

   ```sql
   CREATE KEYSPACE toms 
   WITH replication = {
     'class': 'NetworkTopologyStrategy',
     'dc1': 2,
     'dc2': 2
   };

   USE toms;

   CREATE TABLE users (
       id UUID PRIMARY KEY,
       name text,
       city text
   );
   ```

2. Insert rows:

   ```sql
   INSERT INTO users (id, name, city) VALUES (uuid(), 'Ravi', 'Delhi');
   INSERT INTO users (id, name, city) VALUES (uuid(), 'Yasoda', 'Hyderabad');
   INSERT INTO users (id, name, city) VALUES (uuid(), 'Anjali', 'Chennai');
   ```

3. Validate:

   ```sql
   SELECT * FROM users;
   ```

👉 Expect: 3 rows live, no tombstones yet.

---

## **Day 2 – Hard Delete (Creates Tombstone)**

1. Pick Yasoda’s `id`:

   ```sql
   SELECT * FROM users;
   ```

2. Delete her row:

   ```sql
   DELETE FROM users WHERE id = <uuid_of_yasoda>;
   ```

3. Flush:

   ```bash
   nodetool flush toms users
   ```

4. On `vm06` (dc2), check SSTables:

   ```bash
   ls -ltr /data/cassandra/data/toms/users-*/ | grep Data.db
   ./sstabledump /data/cassandra/data/toms/users-<UUID>/nb-*-Data.db | grep -i tombstone
   ```

👉 Expect: one tombstone marker for Yasoda.

---

## **Day 3 – TTL Expiry (Creates Tombstone Automatically)**

1. Insert row with TTL:

   ```sql
   INSERT INTO users (id, name, city) VALUES (uuid(), 'TTLUser', 'Pune') USING TTL 30;
   ```

2. Wait 40 seconds → row disappears:

   ```sql
   SELECT * FROM users;
   ```

3. Flush:

   ```bash
   nodetool flush toms users
   ```

4. Check newest SSTable on `vm02` (dc1):

   ```bash
   ./sstabledump /data/cassandra/data/toms/users-<UUID>/nb-*-Data.db | grep -i tombstone
   ```

👉 Expect: TTL row converted into a tombstone.

---

## **Day 4 – Verify Multi-DC Replication**

1. On `vm05` (dc2), run:

   ```bash
   SELECT * FROM toms.users;
   ```

   → Deleted rows should also be gone here (tombstones replicated).

2. Flush & dump:

   ```bash
   nodetool flush toms users
   ./sstabledump /data/cassandra/data/toms/users-<UUID>/nb-*-Data.db | grep -i tombstone
   ```

👉 Expect: tombstones present in dc2 node as well.

---

## **Day 5 – Compaction Cleanup**

1. Run manual compaction:

   ```bash
   nodetool compact toms users
   ```

2. After compaction, check SSTables again:

   ```bash
   ls -ltr /data/cassandra/data/toms/users-*/ | grep Data.db
   ./sstabledump /data/cassandra/data/toms/users-<UUID>/nb-*-Data.db | grep -i tombstone
   ```

👉 Expect: Yasoda + TTL tombstones and their old data removed. Final SSTables only have Ravi + Anjali.

---

# 🔹 What You’ll Learn

* **Day 1:** Inserts → live SSTables only.
* **Day 2:** Delete → tombstone written.
* **Day 3:** TTL expiry → tombstone created automatically.
* **Day 4:** Tombstones replicate across DCs.
* **Day 5:** Compaction removes tombstones and cleans up old data.

---

# ✅ Best Practice Notes

* Always check newest `nb-*-Data.db` after flush.
* Don’t forget to `flush` after delete/TTL, otherwise tombstones are still only in memory.
* Compaction is what really removes deleted data from disk — tombstones are just markers until then.

---

👉 Do you also want me to extend this lab to show **“tombstone problem in queries”** (like how a query with filtering hits many tombstones and slows down)? That would complete the hands-on picture.
Perfect 👍 — let’s extend the **tombstone lab** using the names you gave: **Vijaya, Shoba, Mary, Anusha**. We’ll not only create/deletes/TTL rows, but also run queries that show how tombstones can **slow down reads**.

---

# 🔹 Extended Tombstone Lab (with your names)

## **Day 1 – Setup**

```sql
CREATE KEYSPACE toms 
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 2,
  'dc2': 2
};

USE toms;

CREATE TABLE users (
    id UUID PRIMARY KEY,
    name text,
    city text
);
```

---

## **Day 2 – Insert rows**

```sql
INSERT INTO users (id, name, city) VALUES (uuid(), 'Vijaya', 'Delhi');
INSERT INTO users (id, name, city) VALUES (uuid(), 'Shoba', 'Hyderabad');
INSERT INTO users (id, name, city) VALUES (uuid(), 'Mary', 'Chennai');
INSERT INTO users (id, name, city) VALUES (uuid(), 'Anusha', 'Bangalore');
```

Check:

```sql
SELECT * FROM users;
```

👉 Expect 4 rows live.

---

## **Day 3 – Create tombstones**

### A. Hard delete

Delete Shoba:

```sql
DELETE FROM users WHERE id = <uuid_of_shoba>;
```

### B. TTL expiry

Insert Anusha with TTL:

```sql
INSERT INTO users (id, name, city) VALUES (uuid(), 'Anusha', 'Mumbai') USING TTL 30;
```

(Wait 40 seconds → this row will expire → tombstone created.)

### C. Column delete

```sql
UPDATE users SET city = null WHERE id = <uuid_of_mary>;
```

👉 This creates a **cell tombstone** for Mary’s `city`.

---

## **Day 4 – Flush & Check SSTables**

```bash
nodetool flush toms users
ls -ltr /data/cassandra/data/toms/users-*/ | grep Data.db
./sstabledump /data/cassandra/data/toms/users-<UUID>/nb-*-Data.db | grep -i tombstone
```

👉 Expect:

* Tombstone for Shoba (row delete).
* Tombstone for Anusha (TTL expired).
* Tombstone for Mary’s city column.

---

## **Day 5 – Query with tombstone impact**

Turn on tracing:

```sql
TRACING ON;

SELECT * FROM users WHERE name = 'Mary' ALLOW FILTERING;

TRACING OFF;
```

👉 In the trace output, look for lines like:

```
Read 1 live row and 50 tombstone cells
```

This shows Cassandra scanned many tombstones even though it only returned 1 row.

---

## **Day 6 – Compaction**

Run manual compaction:

```bash
nodetool compact toms users
```

Check again:

```bash
./sstabledump /data/cassandra/data/toms/users-<UUID>/nb-*-Data.db | grep -i tombstone
```

👉 After compaction, old data + tombstones are gone; only Vijaya and Mary remain (with Mary’s city=null).

---

# 🔹 What You Learn from This Lab

* **Vijaya** → stays live (normal row).
* **Shoba** → deleted with tombstone.
* **Mary** → column-level tombstone (city=null).
* **Anusha** → TTL row expired → tombstone created.
* Queries (`ALLOW FILTERING`) show tombstone scans = performance cost.
* After **compaction**, tombstones + old data are cleaned permanently.

---

✅ With this, you see **all three types of tombstones**:

1. Row delete (Shoba)
2. TTL expiry (Anusha)
3. Column delete (Mary)

---

👉 Do you want me to also add an example of a **“tombstone-heavy query failure”** (like `TombstoneOverwhelmingException`) and how to fix it? That would complete the real-world troubleshooting angle.

Perfect 👍 — let’s add the **real-world troubleshooting angle**:
how tombstones can cause a **TombstoneOverwhelmingException** and what you can do about it.

---

# 🔹 Tombstone-heavy query failure lab

## Step 1. Insert test data with TTL

Let’s create many rows for **Anusha** that expire quickly (so they turn into tombstones).

```sql
BEGIN BATCH
INSERT INTO users (id, name, city) VALUES (uuid(), 'Anusha', 'Delhi') USING TTL 30;
INSERT INTO users (id, name, city) VALUES (uuid(), 'Anusha', 'Hyderabad') USING TTL 30;
INSERT INTO users (id, name, city) VALUES (uuid(), 'Anusha', 'Chennai') USING TTL 30;
APPLY BATCH;
```

Repeat a few times (so you have 100+ rows for Anusha, all with TTL=30).

👉 After 30–40 seconds, these rows expire → leaving **100+ tombstones** in SSTables.

---

## Step 2. Flush data

```bash
nodetool flush toms users
```

---

## Step 3. Run a query that scans tombstones

Turn tracing ON:

```sql
TRACING ON;
SELECT * FROM users WHERE name = 'Anusha' ALLOW FILTERING;
TRACING OFF;
```

👉 Cassandra will scan through expired rows → tombstones.

In tracing output, you’ll see:

```
Read 0 live rows and 120 tombstone cells
```

---

## Step 4. Simulate failure (optional)

If you insert thousands of TTL rows and then run a big filtering query, you may hit:

```
com.datastax.driver.core.exceptions.InvalidQueryException: 
TombstoneOverwhelmingException: Scanned over X tombstones during query execution (see tombstone_failure_threshold)
```

* **Default threshold:** 100,000 tombstones.
* If Cassandra scans more than this, it throws an exception to protect cluster stability.

---

# 🔹 How to Fix / Mitigate Tombstone Issues

### 1. **Schema design**

* Avoid `ALLOW FILTERING` queries that scan large partitions.
* Use proper partition keys to fetch rows directly.
* For TTL/time-series → use **TimeWindowCompactionStrategy (TWCS)** so tombstones don’t pile up.

### 2. **Compaction**

* Run **UDC or table-level compaction** regularly to purge tombstones.

  ```bash
  nodetool compact toms users
  ```

### 3. **Query tuning**

* Restrict queries with `LIMIT` or tighter `WHERE` conditions.
* Example:

  ```sql
  SELECT * FROM users WHERE id = <uuid>;
  ```

  → fetches direct partition, no tombstone scan.

### 4. **Tuning thresholds** (last resort, not recommended in prod unless you know the risk)

* You can raise limits in `cassandra.yaml`:

  ```yaml
  tombstone_warn_threshold: 1000
  tombstone_failure_threshold: 100000
  ```

  But this doesn’t solve the root cause — only hides the symptom.

### 5. **Monitoring**

* Use `nodetool tablestats` to check:

  ```
  Tombstone Scanned: <number>
  ```
* If numbers are high → schedule compaction.

---

# ✅ What You Learn from This Extended Lab

* **Vijaya** → normal row (control).
* **Shoba** → row delete tombstone.
* **Mary** → column tombstone (`city = null`).
* **Anusha** → TTL expiry tombstones.
* Queries show **how tombstones slow reads** (`Read 0 live rows and X tombstone cells`).
* If you push too far → you’ll see **TombstoneOverwhelmingException**.
* Fix = **better schema design + regular compaction + TWCS for TTL data**.

---

👉 Do you want me to prepare a **production-ready checklist** for your cluster (like daily/weekly tasks to avoid tombstone build-up and avoid these exceptions)?

Got it 👍 — here’s a **production-ready checklist** for your Cassandra cluster to keep **tombstones under control** and avoid `TombstoneOverwhelmingException`.

---

# ✅ Cassandra Tombstone Management Checklist (Daily / Weekly)

---

## 🔹 **Daily Tasks**

### 1. Cluster health check

Run on any node:

```bash
nodetool status
```

* All nodes should be `UN` (Up/Normal).
* If any `DN` or `UJ` → investigate before compaction.

---

### 2. Monitor tombstones

Check per-table stats:

```bash
nodetool tablestats toms.users | grep -i "Tombstone"
```

Look for:

* `Tombstone Scanned` (per read)
* `Tombstone Dropped` (compaction cleanup)

👉 If `Tombstone Scanned` is high (e.g., >1000 per read), schedule compaction for that table.

---

### 3. Run targeted compaction (UDC / table-level)

For 1–3 hot tables per node:

```bash
nodetool flush toms users
nodetool compact toms users
```

👉 Do this in rolling windows (not all nodes at once).
👉 If very large table → use **UDC** (user-defined compaction) on selected SSTables instead of full table compaction.

---

### 4. Query trace spot check

Run one or two queries with tracing:

```sql
TRACING ON;
SELECT * FROM toms.users WHERE name = 'Anusha' ALLOW FILTERING;
TRACING OFF;
```

Check for:

```
Read X live rows and Y tombstone cells
```

👉 If `Y` is much larger than `X`, queries are tombstone-heavy → optimize.

---

## 🔹 **Weekly Tasks**

### 1. Repair system tables

Especially `system_auth`, so role deletes/changes replicate properly:

```bash
nodetool repair system_auth
```

### 2. Full keyspace repair (for data consistency)

```bash
nodetool repair toms
```

👉 Ensures tombstones replicate across DCs.
👉 Prevents **zombie rows** (deleted data coming back).

---

### 3. Review compaction strategy

* Tables with **TTL / time-series** → switch to **TWCS**:

  ```sql
  ALTER TABLE toms.users WITH compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1
  };
  ```
* Tables with **high writes** → stick with **STCS** (default).
* Tables with **read-heavy small partitions** → consider **LCS**.

---

### 4. Disk usage review

Check:

```bash
df -h /var/lib/cassandra
```

👉 If disk free <20% → compaction pressure high.
👉 Plan to scale storage or add nodes.

---

### 5. Audit tombstone thresholds

Check `cassandra.yaml`:

```yaml
tombstone_warn_threshold: 1000
tombstone_failure_threshold: 100000
```

* Don’t raise these in production unless you fully understand the risk.
* If queries regularly hit these → fix schema/query design instead.

---

## 🔹 **Monthly / As Needed**

* Run **major compaction** (only in test/dev or special maintenance):

  ```bash
  nodetool compact toms
  ```
* Review **application TTL usage**: are developers setting unnecessary TTLs that cause tombstones?
* Train dev teams to avoid **“delete everything with ALLOW FILTERING”** patterns.

---

# 🚨 Quick Troubleshooting Guide

* **Slow queries + tombstone warning in logs**
  → Run targeted compaction (`nodetool compact <ks> <table>`).
* **TombstoneOverwhelmingException**
  → Reduce query scope (`WHERE` clause with partition key, add `LIMIT`).
  → Check schema: maybe wrong partition key design.
* **Zombie rows (deleted data comes back)**
  → Run repairs more often. Check `gc_grace_seconds` (default 10 days).

---

# ✅ Summary

* **Daily** → monitor `nodetool tablestats`, compact hot tables, spot-check queries.
* **Weekly** → repair keyspaces, review compaction strategies, check disk usage.
* **Monthly** → schema/TTL review, controlled major compaction in non-prod.
* Always prefer **schema design + TWCS** for TTL workloads over fighting tombstones manually.

---



