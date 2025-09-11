---

# 📌 What is Murmur3?

* **Murmur3** is a **hash function**.
* Hash functions take an input (like `name1`, `name42`, etc.) and produce a big number (called a **hash value**).
* Cassandra uses **Murmur3Partitioner** by default to map your partition keys → token values → specific nodes.

---

# 📌 Why Murmur3 in Cassandra?

Cassandra needs a hash function that is:

1. **Fast** → because every read/write involves hashing the partition key.
2. **Uniform distribution** → so data spreads evenly across all nodes.
3. **Deterministic** → the same key always hashes to the same number.

Murmur3 gives all three ✅

---

# 📌 How Murmur3 works (easy steps)

Imagine you have a key: `"name42"`.

1. **Input**: `"name42"`.
2. **Murmur3 Hashing**: Cassandra runs it through Murmur3 algorithm.
3. **Output**: A **64-bit signed integer** (from `-2^63` to `+2^63-1`).

   * Example: `"name42"` → `-4567891234567890001` (just an example).
4. **Mapping to Ring**:

   * Cassandra nodes each “own” a range of these big numbers (tokens).
   * The hash value decides which node stores the partition.

---

# 📌 Token Ring Example with Murmur3

Let’s say you have 4 nodes:

```
casserv1: owns tokens from -9,223,372,036,854,775,808 → -4,611,686,018,427,387,905
casserv2: owns tokens from -4,611,686,018,427,387,904 → 0
casserv3: owns tokens from 1 → 4,611,686,018,427,387,903
casserv4: owns tokens from 4,611,686,018,427,387,904 → 9,223,372,036,854,775,807
```

* `"name1"` → Murmur3 hash = `-7,000,000,000,000,000,000` → falls into **casserv1’s range**.
* `"name42"` → Murmur3 hash = `2,500,000,000,000,000,000` → falls into **casserv3’s range**.
* `"name100"` → Murmur3 hash = `7,800,000,000,000,000,000` → falls into **casserv4’s range**.

So Murmur3 ensures each key lands **uniformly and predictably** on the ring.

---

# 📌 Murmur3 vs Other Partitioners

Earlier Cassandra versions supported other partitioners:

* **RandomPartitioner** (used MD5) → uniform but slower.
* **ByteOrderedPartitioner** → stored keys in sorted order → but led to **hotspots** (bad for distribution).
* **Murmur3Partitioner (default now)** → best balance: **fast + even distribution**.

---

# 📌 Solid DBA Points (Interview-Ready)

* Murmur3 is a **non-cryptographic hash** (not for security, just for distribution).
* It maps partition keys to a **64-bit token space** (from `-2^63` to `+2^63-1`).
* Tokens are evenly divided among nodes → prevents hotspots.
* With **vnodes** each node owns **hundreds of small token ranges**, further balancing load.
* Result: Cassandra achieves **linear scalability** (data evenly distributed, no single node overloaded).

---

# 📌 Real Example

Let’s take `"name10"`:

```sql
SELECT token('name10');
```

Might return: `-1234567890123456789`.

* Coordinator checks the ring → sees which node owns that token range.
* That node + next `RF-1` nodes get the data.

---

✅ In short:
**Murmur3 = the brain that decides “which node stores which row.”**
It turns your partition key into a number, and that number maps onto the cluster ring.

---
