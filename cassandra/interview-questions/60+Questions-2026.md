comprehensive Apache Cassandra DBA Interview Guide. The document covers **8 sections** with 60+ questions, a 24-node capacity planning reference table, and practical solutions:

[60+ Questions-2026-Download](https://github.com/venkat-mr369/nosql-db/blob/main/cassandra/interview-questions/cassandra_dba_interview_guide_2026.docx)

1. **Architecture & Fundamentals** – Ring architecture, write/read paths, gossip protocol, partitioners, vnodes for 24-node clusters
2. **Replication & Consistency** – RF strategies, all consistency levels, tunable consistency (R+W>N formula), hinted handoff, read repair, anti-entropy repair, LOCAL_QUORUM walkthrough for 24-node/2-DC setup
3. **Cluster Topology & Multi-DC (24-Node Focus)** – DC/rack design, snitch selection, rack awareness, capacity planning table, adding/decommissioning nodes, seed node best practices
4. **Data Modeling & CQL** – Partition/clustering keys, golden rules, tombstones, materialized views vs secondary indexes, LWTs, TTL strategies
5. **Compaction Strategies** – STCS vs LCS vs TWCS with use cases, monitoring compaction, disk space management
6. **Operations & Administration** – Essential nodetool commands, rolling restarts, backups/snapshots, JVM tuning, major version upgrades
7. **10 Scenario-Based Questions** – Hot partitions, node failure recovery, zombie data resurrection, bulk load latency, cross-DC stale reads, OOM during repair, adding a third DC, SSTable corruption, timeout storms, on-prem to AWS migration
8. **Troubleshooting & Issues** – Unavailable exceptions, schema agreement failures, commit log exhaustion, GC flapping, streaming failures, inconsistent reads, monitoring stack, production best practices checklist
