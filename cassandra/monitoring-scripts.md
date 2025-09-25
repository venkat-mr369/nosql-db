### 🛠️ Cassandra Daily Monitoring (Basic → Advanced)

---



#### **1. Basic Health Checks**

```bash
# Cluster status (nodes up/down, DC, rack, load, tokens)
nodetool status

# Check if gossip and native transport are active
nodetool info | egrep "Gossip|Native Transport|Load"

# Verify JVM heap usage
nodetool gcstats

# Check for dropped mutations (warnings if high)
nodetool tpstats | grep Dropped
```

---

#### **2. Storage & Disk**

```bash
# Per-node load (data size handled)
nodetool info | grep "Load"

# Per-table space usage
nodetool tablestats | egrep "Table:|Space used"

# Check disk free space on data & commitlog dirs
df -h /var/lib/cassandra/data
df -h /var/lib/cassandra/commitlog
```

---

#### **3. Compaction & Repairs**

```bash
# Active compactions
nodetool compactionstats

# Pending compactions (should not grow unbounded)
nodetool compactionstats | grep "pending tasks"

# Repair history (look for failed repairs)
nodetool repair -pr
```

---

#### **4. Performance Metrics**

```bash
# Thread pool activity (blocked, pending tasks)
nodetool tpstats

# Latency stats (read/write/cas/other ops)
nodetool tablestats keyspace_name.table_name

# Hinted handoff counts
nodetool netstats | grep "Hints"
```

---

#### **5. System Table Queries (via cqlsh)**

```sql
-- Find slow queries
SELECT * FROM system_traces.sessions LIMIT 5;

-- View schema versions (ensure all nodes in sync)
SELECT peer, schema_version FROM system.peers;

-- Check partition size warnings
SELECT * FROM system.size_estimates LIMIT 5;
```

---

#### **6. Logs & Alerts**

```bash
# Check Cassandra logs for WARN/ERROR
grep -iE "WARN|ERROR" /var/log/cassandra/system.log | tail -20

# GC issues
grep -i "GCInspector" /var/log/cassandra/system.log | tail -20
```

---

#### **7. Advanced / Production Level**

* **Metrics collection**: Export JMX metrics → Prometheus + Grafana dashboards.
* **Alerting thresholds**:

  * Node down (`nodetool status` UN)
  * Heap > 75%
  * Disk > 80%
  * Dropped mutations increasing
  * Compaction pending tasks too high
  * Repair failures
* **Scheduled jobs**:

  * Daily `nodetool repair` (incremental, per keyspace/table).
  * Weekly full repair in multi-DC setups.

---


