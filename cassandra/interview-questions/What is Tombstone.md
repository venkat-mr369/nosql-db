**Tombstone** (in Cassandra)

---

## 🔎 What is a Tombstone in Cassandra?

A **tombstone** in Cassandra is a **marker** that indicates a piece of data (row, column, or partition) has been **deleted or expired (via TTL)**.

⚡ **Key Point:** Data is *not immediately deleted* from disk when you issue a `DELETE` or when a TTL expires. Instead, Cassandra writes a **tombstone marker**. The real deletion happens later during **compaction** and **garbage collection**.

---

### 🛠 How Tombstones Work

1. **Delete/TTL applied:**

   * You delete a column/row:

     ```sql
     DELETE FROM users WHERE user_id = 1;
     ```
   * Or insert data with TTL (time-to-live):

     ```sql
     INSERT INTO sessions (id, user) VALUES (123, 'John') USING TTL 60;
     ```

     → After 60 seconds, a tombstone replaces the row.

2. **Tombstone Created:**

   * Cassandra doesn’t erase data right away.
   * It adds a “tombstone marker” to indicate deletion.

3. **Read Behavior:**

   * During reads, Cassandra sees the tombstone and knows the value is deleted, so it won’t return it.
   * But it still has to scan tombstones, which can **slow down reads** if there are too many.

4. **Compaction & Garbage Collection:**

   * Periodically, Cassandra merges SSTables.
   * If the tombstone is **older than `gc_grace_seconds`** (default 10 days), Cassandra safely removes the deleted data and the tombstone itself.

---

## ⚡ Why Cassandra Uses Tombstones (Use Cases)

### 1. **Ensuring Consistency in Distributed Systems**

* Suppose data is deleted in one node but that node goes down before informing others.
* The tombstone marker ensures that when the node comes back or during repair, other replicas also know **“this data is deleted”**.
* Without tombstones, deleted data might reappear (zombie data).

---

### 2. **Expiring Data Automatically (TTL Use Case)**

* Example: **Session management**

  ```sql
  CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id TEXT,
    created_at TIMESTAMP
  );
  INSERT INTO sessions (session_id, user_id) 
  VALUES (uuid(), 'alice') USING TTL 3600;
  ```
* Session data is automatically deleted after 1 hour.
* A tombstone is created at expiry → later removed during compaction.

---

### 3. **Soft Deletes in Large Datasets**

* Instead of physically deleting millions of rows, Cassandra just writes tombstones.
* This is fast and avoids immediate heavy I/O.
* Actual cleanup happens later in background compactions.

---

## ⚠️ Problems with Tombstones

* **High Read Latency:**
  If a query touches many tombstones (like in wide partitions), Cassandra must check each one → slow reads.
* **Tombstone Overload (Tombstone Death):**
  If too many tombstones exist (millions in one partition), queries can time out.
* **Misconfigured `gc_grace_seconds`:**
  If set too low, replicas may not have had time to exchange tombstone info → deleted data might reappear.

---

## ✅ Best Practices for Tombstones

1. **Design schema to avoid wide partitions** (where tombstones accumulate).
2. **Avoid frequent updates/deletes** on the same rows. Use inserts with new partition keys when possible.
3. **Set appropriate `gc_grace_seconds`:**

   * Long enough to allow repair.
   * Shorter in dev/test environments.
4. **Use `nodetool cfstats`** to monitor tombstone counts.
5. **Avoid queries with `ALLOW FILTERING`** that can hit lots of tombstones.

---

## 🎯 Example Scenario

* You have a `user_activity` table storing logs:

```sql
CREATE TABLE user_activity (
  user_id UUID,
  activity_time TIMESTAMP,
  action TEXT,
  PRIMARY KEY (user_id, activity_time)
);
```

* To keep only last 30 days of logs:

```sql
INSERT INTO user_activity (user_id, activity_time, action)
VALUES (uuid(), toTimestamp(now()), 'Login') USING TTL 2592000; -- 30 days
```

👉 After 30 days, data expires → tombstone created → during compaction after `gc_grace_seconds`, Cassandra permanently removes it.

This helps automatically **purge old logs** without manual deletes.

---

✅ **In short:**
Tombstones are Cassandra’s way of **tracking deletions in a distributed system**. They ensure **consistency and automatic cleanup** but can hurt performance if not managed well.

---
