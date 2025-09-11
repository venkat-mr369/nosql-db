In Cassandra, when you create a keyspace, you must tell it **how to replicate data across datacenters (DCs)**.
👉 what is **NetworkTopologyStrategy (NTS)**.

---

# 1. What is NetworkTopologyStrategy?

* It’s a **replication strategy** for Cassandra keyspaces.
* It lets you control **how many copies (replicas)** of each partition go into **each datacenter (DC)**.
* Works well for **multi-DC deployments** (common in production).
* Replaces **SimpleStrategy** (which only works in one DC and isn’t recommended for real clusters).

---

# 2. Types of replication counts with NTS

When you create a keyspace, you define it like this:

```sql
CREATE KEYSPACE empdb
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3,
  'DC2': 2,
  'DC3': 0
};
```

This means:

* In `DC1`: keep **3 replicas** of each partition
* In `DC2`: keep **2 replicas**
* In `DC3`: keep **0 replicas** (ignore this DC)

👉 You can choose **different replication factors (RF)** per DC.

---

# 3. Internal behavior (how replicas are placed)

* For each partition:

  * Murmur3 hash → token → assign to primary replica in ring of that DC.
  * Cassandra places the next N–1 replicas **on distinct racks/nodes** within the same DC.
  * Tries to avoid putting multiple replicas of the same partition on the same rack (fault tolerance).

---

# 4. Easy-to-grasp **use cases**

### (a) **Single DC cluster (but using NTS)**

```sql
CREATE KEYSPACE empdb
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3
};
```

* All data lives in DC1 with 3 copies.
* Why NTS here? → future proof. If later you add another DC, you can just alter the keyspace to add replication there.
* **Use case:** small production cluster (say 7 nodes, 1 DC).

---

### (b) **Multi-DC cluster (active-active applications)**

```sql
CREATE KEYSPACE empdb
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'US_EAST': 3,
  'EU_WEST': 3
};
```

* Each partition has 3 copies in **US\_EAST** and 3 copies in **EU\_WEST**.
* Applications in US read/write from local DC, European apps use EU DC.
* Cassandra ensures **eventual consistency** between DCs.
* **Use case:** global app (e.g., banking system, e-commerce).

---

### (c) **Analytics-only datacenter**

```sql
CREATE KEYSPACE logs
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3,
  'AnalyticsDC': 1
};
```

* `DC1` (production) keeps 3 copies (for fault tolerance).
* `AnalyticsDC` (dedicated analytics cluster) keeps **just 1 copy**.
* This avoids wasting storage in the analytics DC, but queries still work locally there.
* **Use case:** reporting, Spark jobs, ETL without overloading production nodes.

---

### (d) **Disaster recovery (DR) DC**

```sql
CREATE KEYSPACE empdb
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'PrimaryDC': 3,
  'DRDC': 2
};
```

* Main writes happen in `PrimaryDC` with 3 copies.
* A backup `DRDC` keeps 2 copies for **failover**.
* Apps don’t query DRDC normally, but if Primary goes down, you can failover apps to DRDC.
* **Use case:** insurance against regional outage.

---

# 5. Why not SimpleStrategy?

* SimpleStrategy just places replicas **around the ring** without caring about DCs or racks.
* If you use it in multi-DC, all replicas may land in the **wrong DC**.
* That’s why in production **always use NTS**.

---

# 6. Validation commands

To check what replication your keyspace is using:

```bash
cqlsh> DESCRIBE KEYSPACE empdb;
```

To see actual token ownership and nodes:

```bash
nodetool status
nodetool getendpoints empdb emp 'E101'
```

---

# 7. In short (easy words)

* **NetworkTopologyStrategy = "Tell Cassandra how many copies you want in each datacenter."**
* Lets you design for:

  * **Performance:** keep local copies near your users.
  * **Fault tolerance:** multiple racks / DCs.
  * **Analytics separation:** lightweight replication in non-prod DCs.
  * **Disaster recovery:** standby DC with fewer replicas.

---


---

# 1. **Single-DC (future-proof setup)**

```sql
CREATE KEYSPACE empdb
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3
};
```

```
DC1 (7 nodes)
 ┌───────────────┐
 │ Partition: E101│
 └───────────────┘
       │
       ├──► Node1 (Replica 1 - Primary)
       ├──► Node2 (Replica 2)
       └──► Node3 (Replica 3)
```

✅ All replicas stay in **DC1**. Easy to extend later by adding another DC.

---

# 2. **Multi-DC (Active-Active app)**

```sql
CREATE KEYSPACE empdb
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'US_EAST': 3,
  'EU_WEST': 3
};
```

```
US_EAST (6 nodes)               EU_WEST (6 nodes)
 ┌───────────────┐              ┌───────────────┐
 │ Partition: E101│              │ Partition: E101│
 └───────────────┘              └───────────────┘
       │                               │
       ├──► NodeA (Replica 1 - Primary)├──► NodeX (Replica 1 - Primary)
       ├──► NodeB (Replica 2)          ├──► NodeY (Replica 2)
       └──► NodeC (Replica 3)          └──► NodeZ (Replica 3)
```

✅ Both DCs have **full 3 replicas each**.
Apps in US read/write from `US_EAST`, apps in EU use `EU_WEST`.

---

# 3. **Analytics / DR DC (light replication)**

```sql
CREATE KEYSPACE empdb
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'PrimaryDC': 3,
  'AnalyticsDC': 1
};
```

```
PrimaryDC (7 nodes)               AnalyticsDC (3 nodes)
 ┌───────────────┐                ┌───────────────┐
 │ Partition: E101│                │ Partition: E101│
 └───────────────┘                └───────────────┘
       │                               │
       ├──► Node1 (Replica 1 - Primary)├──► NodeX (Replica 1 only)
       ├──► Node2 (Replica 2)          │
       └──► Node3 (Replica 3)          │
```

✅ Production gets **3 replicas** for safety.
✅ Analytics DC gets just **1 replica** (enough for queries, saves storage).
✅ Or for **DR**, set RF=2 in DRDC.

---

# Quick analogy (easy English)

* Think of each **datacenter** as a **city library**.
* The **book** (your partition) must be stored in multiple **branches** (nodes) in each city.
* NTS lets you decide:

  * “In City1, keep 3 copies”
  * “In City2, keep 2 copies”
  * “In City3, keep none”
* This way readers in each city can quickly get their book without traveling far.

---

