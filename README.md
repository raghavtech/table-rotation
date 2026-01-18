# Table Rotation for Transient Data

## Short Description

High-velocity transient data does not need permanent storage—but it does need predictable performance.

This article introduces **Table Rotation**, a database access pattern where users interact with a single logical table while the application transparently rotates writes across multiple physical partitions. By ensuring that “today’s data” always lives in exactly one active partition, this approach minimizes query fan-out, reduces contention, and enables constant-time cleanup—without exposing complexity to the user.

---
<img width="1312" height="880" alt="table_rotation" src="https://github.com/user-attachments/assets/5dfd66dd-6c9d-4cf5-9105-f75dbf483c56" />


## Main Article

Modern applications generate large volumes of short-lived data—session state, intermediate results, audit breadcrumbs, progress markers, and ephemeral telemetry. While this data is often critical right now, it rapidly loses value after a short retention window.

Traditional approaches—time-based partitioning, TTL deletes, or background cleanup jobs—tend to accumulate hidden costs:

- Increasing query fan-out as partitions grow  
- Expensive delete operations  
- Index churn and fragmentation  
- Unpredictable performance under concurrency  

**Table Rotation** offers a different approach.

In this pattern, the frontend and consumers see a single logical table, but behind the scenes the table is split into **N physical partitions (or tables)**. At any given time:

- Only one partition is “active” for writes  
- The application tier deterministically routes all reads and writes for the current time window (for example, “today”) to that partition  
- At a fixed rotation boundary (daily, hourly, etc.), the active partition advances  
- The oldest partition is reset or truncated and reused, rather than deleted row by row  

Crucially, **all rotation logic lives in the application tier**. The database remains simple, and the behavior is completely transparent to the user and querying clients.

This makes the pattern conceptually similar to **Oracle redo logs**:

- Data is written sequentially  
- Storage is reused in a circular fashion  
- Old data disappears automatically as the system advances  

However, unlike redo logs, this design operates at the **application data layer**, making it portable across databases and adaptable to business semantics.

---

## Why This Works Well for Transient Data

- Queries for “current data” always hit exactly one partition  
- Cleanup is O(1): truncate and reuse  
- Index size remains bounded  
- Locking and contention are dramatically reduced  
- Performance remains stable regardless of system age  

---

## When to Use It

- Data required only for hours or days  
- High write throughput with predictable access patterns  
- Systems where query latency consistency matters more than historical retention  
- Scenarios where deleting data is more expensive than overwriting it  

---

## When Not to Use It

- Long-term analytical data  
- Ad-hoc historical queries  
- Compliance-driven retention requirements  

---

**Summary**

Table Rotation is not a replacement for traditional partitioning—it is a **specialized pattern** for a very common, often overlooked class of data. When applied correctly, it simplifies systems, stabilizes performance, and removes entire categories of operational pain.
