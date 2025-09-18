**glitches (issues/bugs/operational problems)** can appear. 

In production clusters, they often aren’t “bugs” in Cassandra code but **symptoms of misconfiguration, resource pressure, or topology issues**. 


---

# 🔎 Common “Glitches” in Cassandra (and Fixes)

## 1. **Snitch / Topology Mismatch**

**Symptom:**

* Log error:

  ```
  Cannot start node if snitch's data center (dc1) differs from previous data center (datacenter1).
  ```
* Node won’t join cluster.

**Cause:**

* `cassandra.yaml` or `cassandra-rackdc.properties` misaligned with other nodes.

**Fix:**

* Ensure **same DC/Rack naming** across all nodes.
* If redeploying node, run:

  ```bash
  nodetool decommission
  rm -rf /data/cassandra/*
  ```
* Last resort: `-Dcassandra.ignore_dc=true` (not recommended in prod).

---

## 2. **Tombstone Glitch**

**Symptom:**

* Query latency spikes.
* Logs:

  ```
  Read 500 live cells and 500000 tombstone cells for query...
  ```
* Queries timeout.

**Cause:**

* Heavy use of `DELETE`, TTL expiry, or wide partitions.

**Fix:**

* Redesign schema → avoid wide partitions.
* Reduce `gc_grace_seconds` if repairs are frequent.
* Run `nodetool compact` after deletes.
* Monitor `cfstats` for tombstone warnings.

---

## 3. **Commitlog Corruption**

**Symptom:**

* Node crashes after restart.
* Logs:

  ```
  Corrupted commit log detected...
  ```

**Cause:**

* Unexpected shutdown or disk full.

**Fix:**

* Clear commitlogs (safe if data flushed):

  ```bash
  rm -rf /data/cassandra/commitlog/*
  ```
* Restart Cassandra.
* Ensure **commitlog disks** have sufficient space & separate mount.

---

## 4. **Hinted Handoff Glitch**

**Symptom:**

* Node rejoins cluster → huge spike in CPU/disk usage.
* Latency issues.

**Cause:**

* Node was down for too long, many hints stored.

**Fix:**

* Use `nodetool disablehandoff` before taking node down for maintenance.
* Run `nodetool repair` after bringing it back.

---

## 5. **GC (Garbage Collection) Pauses**

**Symptom:**

* Node marked down intermittently.
* `system.log` shows long GC pauses.

**Cause:**

* Heap pressure, large SSTables, too many tombstones.

**Fix:**

* Tune `jvm-options` (`G1GC` preferred in Cassandra 4.x).
* Increase heap cautiously (but never >50% of RAM).
* Compact SSTables to reduce GC work.

---

## 6. **Network Glitches**

**Symptom:**

* Cluster splits into “two halves”.
* Writes fail with `UnavailableException`.

**Cause:**

* Firewall blocking gossip ports (7000/7001).
* Unstable inter-DC link.

**Fix:**

* Validate ports open: `nc -zv host 7000`.
* Ensure NTP clock sync across nodes.
* If multi-DC → prefer `LOCAL_QUORUM` consistency.

---

## 7. **Repair Glitches**

**Symptom:**

* `nodetool repair` never finishes.
* Disk space explodes during repair.

**Cause:**

* Running **parallel full repairs** on all nodes.
* Repair overlaps causing compaction storms.

**Fix:**

* Schedule **subrange repairs** (one node at a time).
* Automate with scripts/cron (your docs show this in BA/CSC clusters).
* Use **incremental repair** in Cassandra 4.x.

---

# ⚡ Real Production Example (7-node cluster)

Suppose in your 7-node cluster:

* **DC1 has 4 nodes**.
* **DC2 has 3 nodes**.
* Issue: After restart, Node2 shows **different datacenter glitch**.

### Root Cause:

`cassandra-rackdc.properties` on Node2 says:

```
dc=datacenter1
rack=RAC1
```

But others say:

```
dc=DC1
rack=RAC1
```

### Fix:

Update Node2’s file → `dc=DC1`.
Clear system tables (`system/local`, `system/peers`) if mismatch persists.
Restart Node2 → joins cluster cleanly.

---

✅ **Key Takeaway:**
Most “glitches” in Cassandra are:

* **Topology misconfigurations (snitch/DC/rack mismatch)**
* **Data model issues (tombstones, wide partitions)**
* **Operational issues (commitlog, hinted handoff, repair overlaps)**

---
