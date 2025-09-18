Great — I’ve gone through all the Cassandra documents, scripts, configs, and CQL references you uploaded.
Based on them, I’ve prepared **deep-dive interview questions with solutions**. These cover **Cassandra concepts, installation, configuration, CQL, and troubleshooting**.

---

# 📌 Apache Cassandra Interview Questions and Answers (Deep Dive)

## 1. Cassandra Basics & Architecture

**Q1: What are the key differences between SQL (RDBMS) and NoSQL Cassandra?**
**Answer:**

* **SQL (RDBMS):** Structured schema, supports transactions (ACID), strong consistency. Examples: Oracle, MySQL.
* **Cassandra (NoSQL):** Schema-less (flexible schema), high write throughput, eventual consistency (tunable), decentralized. Stores structured, semi-structured, unstructured data (videos, logs, etc.).

---

**Q2: Explain Cassandra’s architecture in detail.**
**Answer:**

* **Node**: Basic unit where data is stored.
* **Cluster**: Collection of data centers.
* **Data Center (DC):** Logical grouping of nodes.
* **Partitioner**: Determines how data is distributed across nodes.
* **Snitches**: Tell Cassandra about topology (datacenter, rack). Types: `SimpleSnitch`, `PropertyFileSnitch`, `GossipingPropertyFileSnitch`.
* **Replication Strategies**:

  * `SimpleStrategy`: single DC.
  * `NetworkTopologyStrategy`: multiple DCs.
* **Commit Log → Memtable → SSTables** (write path).
* **Read Path** uses Bloom filters, SSTables, caches, read repair.

---

**Q3: How does Cassandra ensure high availability?**
**Answer:**

* **No single point of failure** (every node is equal).
* **Replication across nodes & data centers**.
* **Gossip Protocol** for node communication.
* **Tunable Consistency** for balancing latency vs accuracy.
* **Repair mechanisms** (Full, Incremental, Subrange, Sequential/Parallel).

---

## 2. Cassandra Installation & Configuration

**Q4: Walk through Cassandra installation on Linux using a shell script.**
**Answer:**

* Install Java and dependencies.
* Create cassandra user/group.
* Create directories (`/data/cassandra/data`, `commitlog`, `saved_caches`, `hints`).
* Extract Cassandra binaries into `/opt/apache-cassandra-x.y.z`.
* Edit `cassandra.yaml` for:

  * `listen_address`, `rpc_address`
  * `cluster_name`
  * `seeds`
  * `data_file_directories`, `commitlog_directory`.
* Update environment in `.bash_profile`.
* Start Cassandra with `cassandra -f`.

**Key point:** Automating setup with **bash scripts** improves consistency across nodes.

---

**Q5: What common issues occur at the server-level setup?**
**Answer:**

* **Snitch mismatch error**: “Cannot start node if snitch’s data center differs” → Fix by aligning DC names or set `-Dcassandra.ignore_dc=true` in `cassandra-env.sh`.
* **Firewall blocking**: Must disable or open ports (7000, 7001, 9042, 9160).
* **Directory permissions**: Cassandra user must own `/data` and `/var/log/cassandra`.

---

**Q6: How do you prepare nodes for passwordless SSH in Cassandra cluster automation?**
**Answer:**

* On `server1` → `ssh-keygen`
* Copy key → `ssh-copy-id cassandra@server2` and `server3`.
* This enables automation of `nodetool` and cron repair jobs.

---

## 3. Cassandra Query Language (CQL)

**Q7: Demonstrate table creation with primary and compound keys.**
**Answer:**

* **Simple PK:**

```sql
CREATE TABLE users (
   user_name varchar PRIMARY KEY,
   password varchar
);
```

* **Compound PK (Partition + Clustering):**

```sql
CREATE TABLE emp (
   empID int,
   deptID int,
   first_name varchar,
   last_name varchar,
   PRIMARY KEY (empID, deptID)
);
```

Here, `empID` = Partition Key, `deptID` = Clustering Column.

---

**Q8: Explain TTL and WRITETIME with example.**
**Answer:**

* **TTL (Time to Live):** Data auto-expires.

```sql
INSERT INTO clicks (userid, url, date, name)
VALUES (uuid(), 'http://apache.org', '2023-10-09', 'Mary')
USING TTL 86400;
SELECT TTL(name) FROM clicks WHERE url='http://apache.org' ALLOW FILTERING;
```



* **WRITETIME:** Retrieves timestamp of last write.

```sql
SELECT WRITETIME(name) FROM clicks WHERE url='http://apache.org';
```

---

**Q9: What are Cassandra collections and their limitations?**
**Answer:**

* **Types:** `set`, `list`, `map`.
* Example:

```sql
ALTER TABLE users ADD emails set<text>;
UPDATE users SET emails = emails + {'test@example.com'} WHERE user_id='123';
```



* **Limitations:**

  * Max 64K elements.
  * Stored as one cell per element → performance overhead.
  * Use only for small datasets, else model as separate tables.

---

**Q10: What is the role of secondary indexes in Cassandra? When to use/not use?**
**Answer:**

* Index allows querying by non-primary key column.
* Good for **low-to-medium cardinality** columns (like `gender`).
* Avoid on **high-cardinality or frequently updated columns** (like `email`), as performance degrades.

---

## 4. Repairs, Compaction & Tombstones

**Q11: Explain repair strategies in Cassandra.**
**Answer:**

* **Why Repairs?** To fix inconsistencies due to hinted handoffs, failed nodes, or missed writes.
* **Types:**

  * Full Repair
  * Incremental Repair
  * Sequential vs Parallel
  * Subrange Repair (specific token ranges).
* **Tools:** `nodetool repair`, cron jobs for automated repair.

---

**Q12: What are compaction strategies?**
**Answer:**

* **STCS (SizeTieredCompactionStrategy):** Default, merges similar sized SSTables.
* **LCS (LeveledCompactionStrategy):** Good for read-heavy workloads.
* **DTCS (DateTieredCompactionStrategy):** Good for time-series.
* **UDC (User Defined Compaction):** Manual trigger of compaction.

---

**Q13: What are tombstones in Cassandra?**
**Answer:**

* **Marker for deleted data** (not immediately removed).
* Removed during compaction & garbage collection.
* Too many tombstones → read latency issues.
* Troubleshooting: Use `nodetool cfstats`, tune `gc_grace_seconds`.

---

## 5. Troubleshooting & Monitoring

**Q14: How do you monitor a Cassandra cluster?**
**Answer:**

* **Key KPIs:**

  * Node availability (`nodetool status`).
  * Latency (read/write).
  * Throughput.
  * Disk usage.
  * GC pauses.
  * Exceptions like `UnavailableException`.
* **Tools:** `nodetool`, Prometheus, Grafana, OpsCenter.

---

**Q15: Common performance issues and fixes?**
**Answer:**

* **High Latency:** Too many tombstones, large partitions, wrong consistency level.
* **Disk Full:** Increase compaction, cleanup hints/logs.
* **Down nodes:** Use `nodetool decommission` / `rebootstrap`.
* **Snitch config mismatch:** Align DC/Rack in `cassandra.yaml`.

---
