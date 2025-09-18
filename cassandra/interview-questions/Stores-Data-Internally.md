---

# 🔎 Deep Dive: How Cassandra Stores Data Internally

## 1. **Keyspace Creation**

Example:

```sql
CREATE KEYSPACE prod_data
WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3,
  'DC2': 2
};
```

* **Keyspace = database container.**
* Defines **replication factor (RF)** per datacenter.
* Cassandra does not store data in the keyspace directly → keyspace contains **tables**.

---

## 2. **Table Creation**

```sql
CREATE TABLE prod_data.emp (
   emp_id int,
   emp_name text,
   salary int,
   PRIMARY KEY (emp_id)
);
```

* Table = **Column Family** in Cassandra.
* **Partition Key = emp\_id** decides data distribution.
* Each row identified by a **Partition Key → Token** (using partitioner).

---

## 3. **Write Path (Arrow Flow)**

When you insert a row:

```sql
INSERT INTO prod_data.emp (emp_id, emp_name, salary)
VALUES (1, 'Sriram', 10000);
```

### Flow (internally):

```
Client Query
   ↓
Coordinator Node (receives request)
   ↓
Partitioner (Murmur3) → Hash(emp_id=1) → Token
   ↓
Replica Selection (based on RF=3 in DC1, RF=2 in DC2)
   ↓
Chosen Nodes (5 replicas total)
   ↓
On Each Replica:
   CommitLog (append-only, durable)
       ↓
   MemTable (in-memory structure per table)
       ↓ (flush when full or on schedule)
   SSTables (immutable disk files, stored on disk)
       ↓
   Compaction (merges multiple SSTables, clears tombstones)
```

---

## 4. **Read Path (Arrow Flow)**

When you run:

```sql
SELECT * FROM prod_data.emp WHERE emp_id=1;
```

### Flow:

```
Client Query
   ↓
Coordinator Node (based on token map)
   ↓
Consistency Check (LOCAL_QUORUM, QUORUM, etc.)
   ↓
Replicas Respond
   ↓
On Each Replica:
   Row Cache / Key Cache? (if hit → fast return)
   ↓
   Bloom Filter (check if SSTable has key)
   ↓
   Partition Summary → Partition Index → Row Data
   ↓
   MemTable checked first (newest data)
   ↓
Coordinator merges results from replicas
   ↓
Client gets final response
```

---

## 5. **Compaction & Tombstone Cleanup**

* Multiple SSTables merged → fewer files.
* Deleted rows (tombstones) permanently removed after `gc_grace_seconds`.

---

# 🎯 Arrow-Based Diagram (Storage Flow)

Here’s a **visual text-based arrow diagram** of Cassandra write path:

```
[Client Write Request]
         ↓
 [Coordinator Node]
         ↓
 [Partitioner → Token]
         ↓
 [Replica Selection (Snitch + RF)]
         ↓
 ┌─────────────────────────────────┐
 │   Replica Node (example DC1N2) │
 └─────────────────────────────────┘
         ↓
     [Commit Log]  ← durable log
         ↓
     [MemTable]    ← in-memory buffer
         ↓ (flush)
     [SSTable]     ← immutable disk file
         ↓
   [Compaction] → merges SSTables, clears tombstones
```

And for the **read path**:

```
[Client Read Request]
         ↓
 [Coordinator Node]
         ↓
 [Consistency Level Check]
         ↓
 [Replicas Queried]
         ↓
  ┌──────────────────────────────┐
  │ Replica Node (search order): │
  │  1. Row/Key Cache            │
  │  2. MemTable                 │
  │  3. SSTable (via BloomFilter)│
  └──────────────────────────────┘
         ↓
 [Coordinator Merges Results]
         ↓
 [Return to Client]
```

---

# ✅ Key Takeaways

* **Keyspace** defines replication policy, not actual storage.
* **Tables (Column Families)** store rows partitioned by key → distributed across nodes.
* **Writes → CommitLog + MemTable → SSTables**.
* **Reads → Cache → MemTable → SSTables (with Bloom filters & indexes)**.
* **Compaction** + **Tombstones** maintain efficiency over time.

---
