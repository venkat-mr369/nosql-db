In Cassandra, **system keyspaces** are special, 
internally managed keyspaces created automatically by Cassandra to store metadata, cluster state, authentication data, and tracing information. 
They are crucial for Cassandra’s operation and **should never be dropped or altered directly**. 

---

### 1. **system**

* **Purpose:** Core metadata and operational state of the cluster.
* **Contents:**

  * Stores information about cluster topology, tokens, peers, schema definitions, and schema history.
  * Holds gossip and snitch metadata (which node has which token ranges).
  * Maintains hints, repair history, and compaction information.
* **Use Case:**

  * When nodes join or leave, the `system` keyspace ensures cluster coordination.
  * Schema changes (new tables, altered tables) are recorded here for distribution across the cluster.
* **Example Tables:**

  * `peers`: Information about other nodes in the cluster.
  * `local`: Information about the local node.
  * `sstable_activity`: Tracks activity of SSTables.

---

### 2. **system_schema**

* **Purpose:** Stores schema metadata (similar to `INFORMATION_SCHEMA` in SQL databases).
* **Contents:**

  * Definitions of keyspaces, tables, indexes, types, functions, and aggregates.
* **Use Case:**

  * When you run `DESCRIBE TABLE` in CQL, Cassandra queries `system_schema`.
  * Schema versioning and propagation across nodes.
* **Example Tables:**

  * `tables`: Stores metadata of all tables.
  * `columns`: Stores information about table columns.
  * `keyspaces`: Stores keyspace definitions.

---

### 3. **system_auth**

* **Purpose:** Authentication and authorization.
* **Contents:**

  * Stores user credentials and role-based access control (RBAC).
* **Use Case:**

  * When a client connects with a username/password, Cassandra checks credentials in `system_auth`.
  * Role hierarchies and permissions are stored here.
* **Example Tables:**

  * `roles`: Stores user and role definitions.
  * `role_permissions`: Stores permissions granted to roles.

---

### 4. **system_distributed**

* **Purpose:** Stores distributed metadata for cluster-wide operations.
* **Contents:**

  * Tracks schema versions and repairs across the cluster.
* **Use Case:**

  * Ensures schema agreement across nodes.
  * Stores repair history for anti-entropy operations.
* **Example Tables:**

  * `parent_repair_history`: Stores details of completed repairs.
  * `repair_history`: Stores details of repair sessions.

---

### 5. **system_traces**

* **Purpose:** Stores tracing information for query performance debugging.
* **Contents:**

  * Records query execution events, timings, and nodes involved.
* **Use Case:**

  * If you enable tracing (`TRACING ON;`), query execution paths are written here.
  * Useful for finding slow queries or identifying hotspots.
* **Example Tables:**

  * `sessions`: High-level details about tracing sessions.
  * `events`: Step-by-step events in query execution.

---

### 6. **system_views**

* **Purpose:** Provides system-level views (virtual tables) for monitoring and introspection.
* **Contents:**

  * Virtual tables that expose runtime stats without persisting to disk.
* **Use Case:**

  * You can query system status (similar to `SHOW STATUS` in MySQL).
* **Example Tables:**

  * `built_indexes`: Lists indexes built in the cluster.
  * `sstable_tasks`: Shows background tasks on SSTables.

---

### 7. **system_virtual_schema**

* **Purpose:** Defines **virtual keyspaces** and tables.
* **Contents:**

  * Holds definitions of system virtual tables.
* **Use Case:**

  * Allows Cassandra to expose runtime information (like thread pools, client connections) as queryable tables.
* **Example Tables:**

  * Metadata about virtual keyspaces and tables (similar to "views of views").

---

✅ **Key Notes:**

* These system keyspaces are **Essential for Cassandra’s internal functioning**.
* You can query them for insights but **should not alter/drop them**.
* They serve roles similar to `INFORMATION_SCHEMA`, `PERFORMANCE_SCHEMA`, and `mysql` in MySQL, or `pg_catalog` in PostgreSQL.

---

