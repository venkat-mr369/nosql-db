 **keyspace + table + data** and walk through exactly **how Cassandra stores it across nodes** with replication. 
 
**Step-by-Step example** using your `prod_data` keyspace and an `emp` table

---

### 🏗 Step 1: Keyspace Definition

You created:

```sql
CREATE KEYSPACE prod_data
WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'DC1': 3,
  'DC2': 2
};
```

👉 This means:

* **DC1 (East-US, 4 nodes):** 3 replicas per partition.
* **DC2 (West-US, 3 nodes):** 2 replicas per partition.
* Total **5 copies** of each row in cluster.

---

### 🏗 Step 2: Table Definition

Let’s create a simple `emp` table:

```sql
CREATE TABLE prod_data.emp (
   emp_id int,
   emp_name text,
   salary int,
   PRIMARY KEY (emp_id)
);
```

* **Partition Key = emp\_id**
  → Controls *which node(s)* the record is stored on.
* Each `emp_id` is hashed by **partitioner** (Murmur3Partitioner) → gives a token.
* Token decides which node is the **primary replica**, then snitch + replication strategy place other replicas.

---

### 🏗 Step 3: Insert Example Data

```sql
INSERT INTO prod_data.emp (emp_id, emp_name, salary) VALUES (1, 'Sriram', 10000);
INSERT INTO prod_data.emp (emp_id, emp_name, salary) VALUES (2, 'Devi', 12000);
INSERT INTO prod_data.emp (emp_id, emp_name, salary) VALUES (3, 'Jaanu', 18000);
-- ... up to 50 Indian employees
```

---

### 🏗 Step 4: Cluster Setup (7 Nodes)

Imagine we have 7 nodes:

* **DC1 (East-US, 4 nodes)**

  * Node1 (RAC1)
  * Node2 (RAC2)
  * Node3 (RAC3)
  * Node4 (RAC1)

* **DC2 (West-US, 3 nodes)**

  * Node5 (RAC1)
  * Node6 (RAC2)
  * Node7 (RAC3)

---

### 🏗 Step 5: How Data is Stored (Replica Placement)

### Example 1: Employee ID = 1 (Sriram)

1. `Murmur3Partitioner` hashes `emp_id=1` → token = say `12345`.
2. Token range determines **primary replica** → suppose Node2 in DC1.
3. Replicas placed:

   * **DC1 (RF=3):** Node2 (primary), Node3, Node4.
   * **DC2 (RF=2):** Node5, Node6.
4. Result: Row `{1, 'Sriram', 10000}` exists on **5 nodes**.

---

### Example 2: Employee ID = 2 (Devi)

1. `emp_id=2` → token = `98765`.
2. Falls into Node6 (DC2) range.
3. Replicas placed:

   * **DC2 (RF=2):** Node6 (primary), Node7.
   * **DC1 (RF=3):** Node1, Node2, Node3.
4. Row `{2, 'Devi', 12000}` exists on **5 nodes**.

---

### Example 3: Employee ID = 3 (Jaanu)

1. `emp_id=3` → token = `54321`.
2. Suppose primary = Node4 in DC1.
3. Replicas placed:

   * **DC1 (RF=3):** Node4, Node1, Node2.
   * **DC2 (RF=2):** Node5, Node7.
4. Row `{3, 'Jaanu', 18000}` exists on **5 nodes**.

---

📊 **General Rule:**

* Every employee row is stored **3 times in DC1** and **2 times in DC2**.
* Placement ensures **no 2 replicas sit on the same rack** (rack-aware).
* Over 50 employees, tokens distribute them evenly → load balanced.

---

### 🏗 Step 6: Real Production Use Case

### Scenario: Payroll System (India HQ)

* **emp table** stores employee data for salary processing.
* Business runs in **two regions (DC1 = Mumbai, DC2 = Bangalore)**.
* Requirement: Survive datacenter outage.

👉 With your setup:

* **Mumbai (DC1):** Keeps 3 replicas (local high availability).
* **Bangalore (DC2):** Keeps 2 replicas (DR + read scaling).

---

### Example Queries

**Read Employee Salary (Consistency LOCAL\_QUORUM):**

```sql
CONSISTENCY LOCAL_QUORUM;
SELECT * FROM prod_data.emp WHERE emp_id = 1;
```

* Query from Mumbai app servers → only 2 nodes in DC1 must agree.
* Query fast, even if Bangalore is offline.

**Update Employee Salary:**

```sql
UPDATE prod_data.emp SET salary = 20000 WHERE emp_id = 1;
```

* Write goes to 3 replicas in DC1 + 2 replicas in DC2.
* Ensures both DCs always have latest salary.

---

### ⚠️ What Happens During Failures?

1. **Rack failure in DC1:**

   * If RAC1 dies (Node1 + Node4), data still available from RAC2 & RAC3.

2. **DC2 outage:**

   * Still safe → all data exists 3 times in DC1.

3. **Single node crash:**

   * Cassandra automatically reroutes queries to replicas.
   * Later `nodetool repair` syncs data back.

---

<img width="1900" height="1164" alt="image" src="https://github.com/user-attachments/assets/98559d0f-d9f0-4fc4-8511-3cbe8bbbecf5" />

Here’s a Visio-style architecture diagram for your 7-node Cassandra cluster showing how replicas are placed for emp_id=1,2,3:

Blue nodes = DC1 (Mumbai, 4 nodes).

Green nodes = DC2 (Bangalore, 3 nodes).

Dashed colored lines = replica groups for each employee row:

🔴 Red = emp_id=1 (Sriram)

🔵 Blue = emp_id=2 (Devi)

🟣 Purple = emp_id=3 (Jaanu)

Each employee row is stored on 5 nodes total (3 in DC1, 2 in DC2), spread across racks for HA/DR.


* With **RF=3 in DC1** and **RF=2 in DC2**, every record (like Sriram, Devi, Jaanu) exists on **5 nodes total**.
* **Snitch + Replication** guarantee that replicas are spread across **different racks & DCs**.
* Queries use **tunable consistency (LOCAL\_QUORUM, QUORUM, ALL)** for balancing speed vs correctness.
* Production use case (Payroll/HR in India) → ensures **HA + DR** across 7-node cluster.

---
