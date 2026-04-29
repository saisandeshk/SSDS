# Lecture 4.1: NoSQL Databases — Relational DBs, Replication, CAP & BASE

## DS256 - Scalable Systems for Data Science
### Module 4: NoSQL Databases

> **References**:
> - Lecture slides: M4.NoSQL.pdf (slides 1–52)
> - Gray et al., "The Dangers of Replication and a Solution", SIGMOD 1996
> - Brewer, "CAP Twelve Years Later: How the Rules Have Changed", IEEE Computer 2012

---

## 1. Relational Databases: A Quick Review

### 1.1 Core Data Model

A **relational database** organizes data into **tables** (also called *relations*):

| Term | Meaning |
|------|---------|
| **Table / Relation** | A named, structured collection of data |
| **Column / Field / Attribute** | A named property with a declared data type |
| **Row / Tuple / Record** | One instance of data in the table |
| **Schema** | The definition of table structure (columns, types, constraints) |

**Data types**: integers, floats, strings, timestamps, booleans, etc. Each column has a fixed declared type.

**Operations on data (CRUD)**:
- **Create** → `INSERT`
- **Read** → `SELECT`
- **Update** → `UPDATE`
- **Delete** → `DELETE`

**Schema operations (DDL — Data Definition Language)**:
- `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`

### 1.2 Integrity Constraints

The schema can enforce rules about valid data:

- **Uniqueness constraint**: A column (or combination of columns) must have unique values across all rows — this column becomes a **primary key**.
- **Foreign key constraint**: A column in one table must reference an existing primary key in another table (referential integrity).
  - Example: An `Orders` table with `customer_id` foreign key referencing `Customers.id` — prevents orphan records.

### 1.3 Logical vs. Physical View

A key principle of relational databases is **data independence**:

- **Logical view** (what the user sees):
  - Tables with rows and columns
  - Declarative SQL queries ("what" you want, not "how" to get it)
  - No concern about physical storage

- **Physical layout** (what the DBMS manages internally):
  - Data stored in **blocks** (pages) on disk
  - **B-tree indexes** for fast lookups
  - **Query planner / optimizer** decides execution strategy
  - The optimizer picks the best among many equivalent plans

This separation allows the database to change its physical storage strategy without breaking application code.

---

## 2. DBMS Architecture

A Database Management System (DBMS) is a complex piece of software with several interacting subsystems:

```
┌─────────────────────────────────────────────────────────────┐
│                  Query Evaluation Engine                     │
│     [Parser] → [Optimizer] → [Plan Executor]                │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
  [Transaction   [Lock Manager] [Recovery Manager]
   Manager]
        │             │             │
        └─────────────┼─────────────┘
                      ▼
              [Buffer Manager]    ← manages RAM cache of disk blocks
                      │
              [Disk Space Manager] ← manages files/blocks on disk
```

| Subsystem | Responsibility |
|-----------|---------------|
| **Parser** | Parse and validate SQL syntax |
| **Optimizer** | Generate and choose the best execution plan |
| **Plan Executor** | Execute the chosen plan |
| **Transaction Manager** | Manages begin/commit/abort; enforces ACID |
| **Lock Manager** | Manages locks for concurrency control (isolation) |
| **Recovery Manager** | Write-ahead logging (WAL), crash recovery |
| **Buffer Manager** | Caches disk blocks in RAM; manages eviction |
| **Disk Space Manager** | Lowest layer; manages physical files and blocks |

---

## 3. ACID Properties

ACID is the foundation of relational database guarantees. It defines what a **transaction** promises:

### 3.1 The Four Properties

**A — Atomicity**
- A transaction either **commits completely** or **aborts entirely** — no partial state.
- If a crash occurs mid-transaction, the recovery manager rolls back incomplete changes.
- *Analogy*: A bank transfer (debit + credit) is atomic — you never have money deducted without being credited.

**C — Consistency**
- A transaction takes the database from one **valid state** to another **valid state**.
- "Valid" means all integrity constraints (primary keys, foreign keys, domain rules) are satisfied.
- Note: The *application* defines what "consistent" means; the DB enforces the declared constraints.

**I — Isolation**
- Transactions execute as if they are **alone** — no interference from concurrent transactions.
- Result should be **serializable**: equivalent to executing transactions one at a time in some serial order.
- Implemented via locking (or MVCC in modern systems).

**D — Durability**
- Once a transaction **commits**, its effects are **permanent** — they survive crashes, power failures, etc.
- Implemented via write-ahead logging (WAL): changes written to log on disk before acknowledged.

### 3.2 Why ACID Eases Development

Without ACID, programmers must handle partial failures manually. With ACID:

- **No partial state**: You never see half-applied updates (Atomicity).
- **No need to reason about concurrency**: Serial/serializable execution model simplifies application logic (Isolation).
- **Trust that committed data persists**: No need to verify data survived a crash (Durability).
- **Constraints enforced automatically**: Less validation code in the application (Consistency).

In short: ACID lets developers think of databases as *simple, reliable* storage — a major productivity win.

---

## 4. Limitations of Relational Databases

Despite their power, RDBMSs have fundamental limitations:

### 4.1 Schema Rigidity
- Tables require a **fixed schema** — awkward for unstructured or semi-structured data (JSON documents, social media posts, sensor streams).
- Schema evolution (adding/removing columns) is painful and may require downtime.

### 4.2 Scalability Limits (The Core Problem)

ACID properties **conflict with horizontal scaling**:

- **Locking for isolation**: Locks prevent parallel access → throughput bottleneck.
- **2-phase commit for distributed transactions**: Requires coordination across nodes → high latency, availability risk.
- **Scale-up, not scale-out**: Traditional RDBMSs are designed to run on a single powerful machine (scale-up). Distributing across many machines while preserving ACID is extremely hard.

The fundamental tension: **ACID guarantees require coordination; coordination limits scalability.**

---

## 5. The Cloud Computing Context

### 5.1 Historical Evolution

- **~2005**: Amazon builds massive data centers to handle peak holiday traffic.
- **Yahoo 100ms experiment**: Yahoo measured that a **100ms increase** in page load time caused a **1% drop in user traffic** — milliseconds matter for revenue.
- **~2006+**: Companies like Amazon and Google shift to **microservices architecture**:
  - Break monolithic applications into many small, independent services
  - Each service runs on many machines, scales independently
  - Services communicate via APIs (REST, RPC)

### 5.2 Multi-Tier Architecture

Modern cloud applications have a layered structure:

```
Internet
    │
[Tier 1: Lightweight Web Servers]
    │  (handle HTTP, authentication, routing)
    │
[Tier 2: Microservices (μ-services)]
    │  (business logic, each with own DB)
    │
[Tier 3: Storage (Databases, Object Stores)]
```

- **Tier 1**: Stateless web servers — easy to scale horizontally (just add more).
- **Tier 2**: Stateful microservices — each owns its data, communicates asynchronously.

### 5.3 The Fundamental Question

With data replicated across many machines in a distributed system: **can we guarantee consistency?**

---

## 6. The Dangers of Replication (Gray et al., SIGMOD 1996)

Jim Gray's landmark paper identified why replication is hard and categorized the design space.

### 6.1 Why Replicate?

- **Performance**: Serve reads from nearby replicas (low latency).
- **Availability**: System stays up even if some nodes fail.
- **Scalability**: Distribute read load across replicas.

### 6.2 The Scaleup Pitfall

Gray's key warning: **Replication does not solve scalability if you need synchronous updates.**

- Eager (synchronous) replication requires all replicas to agree before a write commits.
- Adding nodes increases coordination overhead.
- **Deadlocks increase as N³** (N = number of nodes):
  - 10× more nodes → **1000× more deadlocks**
- The system can become *slower* as you add nodes — the opposite of scaling!

### 6.3 Replication Taxonomy

Gray classifies replication along two axes:

**Axis 1: Propagation — When are updates sent to replicas?**
- **Eager (Synchronous)**: Update all replicas before committing. Guarantees consistency; high latency.
- **Lazy (Asynchronous)**: Commit locally, propagate to replicas later. Low latency; risk of stale reads.

**Axis 2: Ownership — Who can accept writes?**
- **Group (Peer-to-Peer)**: Any replica can accept writes.
- **Master**: One designated master handles all writes; replicas are read-only.
- **Two-Tier**: Master + mobile/disconnected clients (special case).

This gives a 2×3 matrix of strategies (some combinations are theoretical):

| | Group | Master | Two-Tier |
|---|---|---|---|
| **Eager** | Serializable; deadlock ∝ N³ | — | — |
| **Lazy** | Reconciliation needed; "system delusion" | RPC to master; deadlock ∝ N² | Tentative + base transactions |

### 6.4 Eager Group Replication (Distributed Transactions)

- Every write acquires **locks on all replicas** simultaneously.
- Commit requires **2-phase commit (2PC)** across all replicas.
- **Serializable** — readers always see consistent state.
- **Problem**: Deadlock probability grows as N³.
  - Two transactions on different nodes can each hold a lock and wait for the other → deadlock.
  - With N nodes, deadlock probability ~ N³.
  - 10× nodes = 1000× more deadlocks → system effectively unusable at large scale.

### 6.5 Lazy Group Replication

- Writes committed locally, propagated asynchronously.
- No global locking → low latency, no deadlocks from 2PC.
- **Problems**:
  - **Stale reads**: A reader may see old data (not yet propagated).
  - **"System delusion"**: Different nodes believe different things are "current" state.
  - **Reconciliation**: When conflicting updates reach each other, must resolve conflicts.
  - Conflict resolution is application-specific and complex.

**Example**: Two users simultaneously update their profile on different nodes. Both commits locally. When propagated, which update wins?

### 6.6 Lazy Master Replication

- One **master node** accepts all writes; replicas are read-only.
- Reads can go to any replica (fast, local).
- Writes: client sends RPC to master → master commits → propagates lazily to replicas.
- **Deadlock rate ~ N²** (better than N³ for eager group, worse than no-replication).
  - Concurrent writes to master still cause lock contention on the master.
- **Read staleness**: Reads from replica may be stale (propagation delay).

### 6.7 Non-Transactional / Convergence

- No transactions, no locks.
- **Convergence property**: Even without coordination, replicas will *eventually* converge to the same state if updates are designed carefully.
- Requires updates to be **commutative** (order of application doesn't matter) and **idempotent** (applying same update twice = applying once).
- Example: Lotus Notes used this approach (1990s groupware).
- This is essentially **Eventual Consistency** (precursor to BASE).

### 6.8 Two-Tier Replication

- **Base nodes**: Always connected; store the authoritative data.
- **Mobile nodes**: May be disconnected for long periods (laptops, field workers).
- Two types of transactions:
  - **Base transactions**: Executed on base nodes when connected; immediately durable.
  - **Tentative transactions**: Executed on mobile node while disconnected; marked as tentative.
- When mobile node reconnects: tentative transactions submitted to base nodes.
  - May be accepted (promoted to base transactions) or rejected (rolled back on mobile).
- Models disconnected operation — important for mobile/offline use cases.

### 6.9 Sharding

Sharding (partitioning) is a complementary strategy to replication:

- **Partition** the dataset into **K shards** using `Hash(key) % K`
- Each shard stored on a different machine
- **Single-shard transactions**: Only one machine involved → scale well, no coordination overhead.
- **Multi-shard transactions**: Multiple machines involved → requires 2-phase commit (2PC) → performance collapses.
- **Rule of thumb**: Design data model so most transactions are single-shard.

**Sharding + Replication**:
- For availability, each shard is itself replicated across multiple nodes.
- Now writes must go to multiple replicas of that shard → introduces lock contention again.
- The scaling benefits of sharding are partially offset by replication overhead.

---

## 7. BASE Methodology

### 7.1 Origins

BASE was invented at **eBay** as a pragmatic response to the scaling limits of ACID. The name is a deliberate contrast to ACID:
- **ACID** = strong guarantees, limited scalability
- **BASE** = weaker guarantees, high scalability

BASE ≡ "CAP in practice" — it is the practical application of choosing Availability over Consistency (the AP corner of CAP).

### 7.2 The Three Components

**B — Basically Available**
- The system provides **rapid responses** even during partial failures.
- May return **stale or approximate data** rather than waiting for a consistent view.
- "Available" doesn't mean 100% uptime — it means the system doesn't hang or block indefinitely.
- Trade: some requests may get slightly wrong answers, but they always get *an* answer quickly.

**S — Soft State**
- State in **Tier 1** (web/app servers) is **ephemeral** — not permanently stored.
- Web servers keep no persistent data; they can restart cleanly at any time.
- No state survives a server restart (session state must be external, e.g., in a database or cache).
- This enables **stateless horizontal scaling** of Tier 1 — just add more web servers.
- "Soft" = volatile, temporary, not the ground truth.

**E — Eventual Consistency**
- The system does not guarantee immediate consistency but promises that **all replicas will converge** to the same state *eventually* (given no new updates).
- Techniques used in Eventual Consistency:
  - **Optimistic answers**: Return cached/local data immediately; don't wait for global consensus.
  - **Skip locks**: Don't acquire distributed locks for reads (much faster).
  - **Offline cleanup**: Reconcile inconsistencies in background processes (asynchronously).
  - **Stale data acceptance**: Serve slightly outdated data for reads; apply writes asynchronously.

### 7.3 BASE vs. ACID Trade-Off

| | ACID | BASE |
|---|---|---|
| **Consistency** | Strong (immediate) | Weak (eventual) |
| **Availability** | Lower (blocking) | Higher (non-blocking) |
| **Scalability** | Limited (coordination) | High (minimal coordination) |
| **Complexity** | Low (system handles it) | Higher (app must handle inconsistency) |
| **Use case** | Banking, financial systems | Social media, recommendations, caching |

---

## 8. CAP Theorem

### 8.1 The Theorem (Eric Brewer, 2000 / 2002)

Eric Brewer introduced CAP as a conjecture in 2000; formally proven by Gilbert and Lynch in 2002.

**CAP Theorem**: In a distributed system, you can have at most **2 of these 3 properties**:

```
        Consistency
           /\
          /  \
         /    \
        / CA   \
       /        \
      /___________\
   Availability   Partition
                  Tolerance
```

**Definitions**:

| Property | Definition |
|----------|-----------|
| **C — Consistency** | Every read gets the **most recent write** (or an error). All nodes see the same data at the same time. (Note: this is linearizability, stronger than ACID's C) |
| **A — Availability** | Every request gets a **response** (not an error), though the data may not be the most recent. |
| **P — Partition Tolerance** | The system continues operating despite **network partitions** (some nodes can't communicate). |

**The forced choice**: In a real distributed system, **network partitions can always happen** (hardware failures, network splits). So you always need P. Therefore the real choice is: **C vs. A during a partition**.

### 8.2 The Three Combinations

**CA — Consistent + Available (forfeit Partition Tolerance)**
- Only possible in a **single-site or tightly coupled** system where network partitions cannot occur.
- Examples: Single-site relational DBs, cluster databases, LDAP, xFS file system.
- Traits: 2-phase commit, cache validation protocols.
- *In practice, if a partition occurs, must stop (unavailable) to preserve consistency.*

**CP — Consistent + Partition Tolerant (forfeit Availability)**
- System remains consistent even during partitions, but some nodes become **unavailable**.
- Examples: Distributed databases (HBase), distributed locking systems (ZooKeeper), majority protocols (Paxos, Raft).
- Traits: Pessimistic locking, minority partitions become unavailable (refuse requests).
- *If your partition can't reach a quorum, it refuses to answer → unavailable.*

**AP — Available + Partition Tolerant (forfeit Consistency)**
- System remains available during partitions, but data may be **inconsistent/stale**.
- Examples: Coda (distributed file system), DNS, web caches.
- Traits: Expiration/leases, conflict resolution (optimistic concurrency), optimistic updates.
- *Nodes serve whatever data they have, even if stale → available but possibly inconsistent.*

### 8.3 ACID vs. CAP

| | ACID | CAP |
|---|---|---|
| **C** | Application-defined consistency (constraints satisfied) | Linearizability (most recent value) |
| **I (Isolation)** | Core concern — serializable transactions | Corresponds to CAP's C (isolation = consistency between concurrent ops) |
| **A (Atomicity)** | Atomic transactions | Not directly related |
| **D (Durability)** | Committed data survives crashes | Compatible with both AP and CP |

- **ACID ≈ CP**: Strong consistency + partition tolerance (at cost of availability).
- **ACID's C ≠ CAP's C**: ACID consistency is about application invariants; CAP consistency is about data recency/linearizability.
- **Isolation (ACID's I)** is the property most relevant to CAP's C — serializable isolation = linearizable consistency.

---

## 9. CAP Twelve Years Later: Brewer's Revisit (2012)

In 2012, Brewer himself published a revised perspective on CAP, correcting misconceptions that had accumulated over 12 years.

### 9.1 "2 of 3 is Misleading"

The original framing "pick 2 of 3" is an oversimplification:

- **Partitions are rare**: In a well-maintained LAN, partitions almost never happen. The choice of CA vs. CP vs. AP only matters *when a partition occurs*.
- **Properties are continuous, not binary**: Consistency is a spectrum (from linearizability to eventual consistency). Availability is a spectrum (from 100% to 0%). Treating them as binary "all or nothing" is too simplistic.
- **Better framing**: The real trade-off is between **consistency and latency** (CAP-latency connection).

### 9.2 CAP-Latency Connection

Network partitions and latency are deeply related:

- A **partition** = nodes cannot communicate (network timeout or failure).
- From the perspective of one node: it cannot distinguish "the other node is slow" from "the other node is partitioned."
- A timeout forces a **partition decision**: the node must choose:
  - **Cancel the operation** → preserve consistency, decrease availability (CP choice).
  - **Proceed without confirmation** → stay available, risk inconsistency (AP choice).

**Key insight**: You don't need a full partition to face this trade-off. Any **high-latency network** forces the same decision. So "CAP" is really about **consistency vs. latency** in all distributed systems — not just during failures.

### 9.3 Managing Partitions

Rather than a fixed static choice (always CA, always CP, always AP), modern systems dynamically **manage partitions**:

**Three-phase partition management**:
1. **Detect** the partition: Timeout, heartbeat failure, or explicit partition signal.
2. **Enter partition mode**: Explicitly limit some operations to maintain invariants; log what happened.
   - Decide which invariants to maintain (e.g., "never oversell inventory") vs. which to relax (e.g., "allow stale profile reads").
   - Systems use **compensating transactions** for operations that must eventually be reconciled.
3. **Recover**: When partition heals, merge state, resolve conflicts, restore consistency.

### 9.4 Conflict Resolution Mechanisms

When AP systems merge divergent state after a partition, conflicts must be resolved:

**Version Vectors**
- Each node maintains a **version vector**: a set of `(node_id, logical_timestamp)` pairs.
- Captures the **causal history** of each data item.
- If version vector A > version vector B (element-wise), A causally dominates B → B is outdated.
- If neither dominates, there is a **concurrent conflict** → must resolve.
- Example: `{A:3, B:2}` vs `{A:2, B:3}` → concurrent writes, conflict.

**CRDTs (Conflict-free Replicated Data Types)**
- Data structures designed to **always merge without conflicts**.
- Key insight: Design operations to be **commutative** (a ⊕ b = b ⊕ a) and **associative** — order of application doesn't matter.
- Examples:
  - **G-Counter** (grow-only counter): Merge by taking element-wise max.
  - **LWW-Register** (last-write-wins): Keep value with highest timestamp.
  - **2P-Set** (two-phase set): Add set + remove set; element present if in add-set but not remove-set.
- CRDTs guarantee **state convergence** without coordination — naturally eventual consistent.
- Used in: Amazon DynamoDB (shopping cart), Riak, Redis (some data types).

### 9.5 ACID vs. BASE vs. CAP — The Full Picture

```
                 Consistency Spectrum
Strong ←─────────────────────────────────────→ Weak

Linearizable   Serializable   Read-Committed   Eventual
    │               │               │             │
  ACID CP      Traditional       RDBMS default  BASE/AP
               RDBMS (ideal)     with tuning
```

- **ACID** aims for strong consistency (Serializable Isolation) — trades availability.
- **BASE** embraces eventual consistency — trades strong consistency for availability.
- **CAP** defines the theoretical boundary of what's achievable.
- No approach is universally better — the choice depends on application requirements.

---

## 10. Practical Implications: When to Use What

| Scenario | Approach | Reasoning |
|----------|----------|-----------|
| Banking / financial transactions | ACID / CP | Correctness is critical; availability can be traded |
| Social media feed | BASE / AP | Slight staleness acceptable; high availability needed |
| DNS lookups | AP | Availability and partition tolerance paramount |
| Inventory management | CP with careful partitioning | Cannot oversell; may tolerate brief unavailability |
| User profile reads | AP with eventual consistency | Stale profile for 1s is fine; blocking is not |
| Shopping cart | CRDT-based AP | Merge divergent carts; never block add-to-cart |

---

## 11. Key Concepts Summary

| Concept | Key Point |
|---------|-----------|
| **ACID** | Atomicity, Consistency, Isolation, Durability — strong guarantees for transactions |
| **RDBMS Limitation** | ACID conflicts with horizontal scaling — coordination overhead kills throughput |
| **Gray's Taxonomy** | Eager vs. Lazy × Group vs. Master vs. Two-Tier replication |
| **Deadlock scaling** | Eager group: N³; Lazy master: N²; Non-transactional: no deadlocks |
| **Sharding** | Hash(key)%K partitioning; single-shard transactions scale; multi-shard needs 2PC |
| **BASE** | Basically Available + Soft State + Eventual Consistency — weaker but scalable |
| **CAP Theorem** | Pick 2 of {C, A, P}; since P always needed, real choice is C vs. A during partition |
| **CA** | Single-site DBs; 2PC; no partition tolerance |
| **CP** | ZooKeeper, HBase; consistent but unavailable during partition |
| **AP** | DNS, Coda, web caches; available but stale during partition |
| **Brewer 2012** | "2 of 3" is misleading; real trade-off is consistency vs. latency; manage partitions |
| **Version Vectors** | (node, logical_time) pairs; detect concurrent conflicts via causal ordering |
| **CRDTs** | Commutative/associative data structures; converge without coordination |
| **ACID C ≠ CAP C** | ACID C = application invariants; CAP C = linearizability (data recency) |
| **ACID I ↔ CAP C** | Isolation (ACID) most closely corresponds to Consistency (CAP) |

---

## 12. Key Takeaways

1. **Relational databases give strong ACID guarantees** at the cost of scalability — locking and 2PC make distributed operation expensive.

2. **Replication is a double-edged sword**: Improves availability and read performance, but eager replication causes deadlocks that scale as N³ — 10× more nodes = 1000× more deadlocks.

3. **Gray's insight (1996)**: There is no free lunch in replication. Every strategy involves trade-offs between consistency, availability, and performance.

4. **The cloud forced new thinking**: Milliseconds matter for revenue (Yahoo's 100ms experiment); microservices require each service to scale independently.

5. **BASE was born from pragmatism**: eBay needed to handle huge scale; accepted eventual consistency in exchange for high availability and throughput.

6. **CAP Theorem**: In a distributed system, you cannot simultaneously guarantee Consistency, Availability, and Partition Tolerance. Since partitions are inevitable, the real choice is **C vs. A during a partition**.

7. **Brewer's revision (2012)**: CAP is more nuanced than "pick 2 of 3":
   - Properties are continuous spectra, not binary.
   - Partitions are rare; the same trade-off appears in high-latency communication.
   - Modern systems *manage* partitions dynamically rather than making a static choice.

8. **CRDTs and version vectors** are practical tools for building AP systems that converge correctly without coordination overhead.

---

## References

- Gray, J. et al., "The Dangers of Replication and a Solution", ACM SIGMOD 1996
- Brewer, E., "CAP Twelve Years Later: How the Rules Have Changed", IEEE Computer, Feb 2012
- Slides: M4.NoSQL.pdf, slides 1–52 (DS256, IISc)


# Lecture 4.2: Dynamo — Amazon's Highly Available Key-Value Store

## DS256 - Scalable Systems for Data Science
### Module 4: NoSQL Databases

> **References**:
> - Lecture slides: M4.NoSQL.pdf (slides 60–90)
> - DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store", ACM SOSP 2007

---

## 1. Motivation: Why Dynamo?

### 1.1 The Problem with RDBMS at Amazon Scale

Amazon runs one of the largest e-commerce platforms: tens of millions of customers, peak loads using tens of thousands of servers across many data centers worldwide. **Even the slightest outage has significant financial consequences and impacts customer trust.**

Many Amazon services only need **primary-key access** — no complex joins, no multi-table transactions. Examples:
- Shopping cart (add/remove items)
- Best sellers list
- User preferences
- Session management
- Product catalog

For these use cases, a relational database is **overkill** and actively harmful:
- **Expensive hardware** required for performance
- **Skilled DBAs** needed for operation
- **ACID properties limit availability** — consistency-over-availability trade-off in RDBMS replication
- **Limited load balancing** in traditional RDBMS cluster setups

### 1.2 The Dynamo Solution: Distributed Hashtable

Dynamo is a **highly available, distributed key-value store** (distributed hash table) built for Amazon's internal services. It provides:
- **Primary-key only access** — `get(key)` and `put(key, value)`
- Values are **blobs** (binary objects), up to 1 MB
- **High availability**: writable even during disk failures, network failures, datacenter failures
- **Eventually consistent** — trades strong consistency for availability (AP in CAP)

> **Scale example**: The Shopping Cart Service handled **10 million requests and 3 million checkouts per day** (2007), requiring the system to always accept writes even during failures.

> **Important**: Dynamo (internal Amazon system) is **different from AWS DynamoDB** (the commercial product). DynamoDB is inspired by Dynamo but is a separate managed service.

---

## 2. Amazon's Service Architecture

Amazon's platform is a **service-oriented architecture** with hundreds of services:

```
Client Requests
       │
[Page Rendering Components]   ← stateless workflows, with caching
       │
[Request Routing]
       ├──────────────────────→ [Aggregator Services]
       │
[Request Routing]
       ├──────────────────────→ [Services]
       │
[Dynamo instances]    [Amazon S3]    [Other datastores]
```

- Each **page request** may trigger calls to **over 150 services**.
- Services are often stateful (generate responses from persistent state).
- **SLAs are expressed and measured at the 99.9th percentile** of the distribution — not average. Amazon's philosophy: *"build a system where all customers have a good experience, rather than just the majority"*.
- Why 99.9th percentile? Customers with longer histories need more processing — they appear at the tail of the distribution. SLA targets the tail.

---

## 3. Dynamo Design Goals and Requirements

### 3.1 Goals

| Goal | Details |
|------|---------|
| **Simple Query Model** | `get(key)` and `put(key, value)` only; unique keys; blob values; no transactions; no isolation across keys |
| **High Availability** | Always writeable — even during failures; "always writeable" data store |
| **Guaranteed SLA** | 300ms response for 99.9% of requests at peak 500 requests/second |
| **vs. ACID** | Weak consistency, no Isolation (only single-key ops), no cross-key atomicity |
| **Non-hostile environment** | Internal Amazon system; security not a concern; all nodes trusted |

### 3.2 System Assumptions

- **Query Model**: Simple read/write of a single data item identified by a unique key. State stored as binary blobs. No multi-key operations, no relational schema.
- **ACID**: Dynamo targets the "C" in ACID (consistency) to be weak for high availability. No isolation guarantees. Only single-key operations.
- **Efficiency**: Services must meet SLAs at 99.9th percentile. Performance targets are not based on averages.
- **Scale**: Initially targets hundreds of storage nodes; each service runs its own Dynamo instance.

---

## 4. Design Principles

### 4.1 Optimistic Replication ("Always Writeable")

Traditional replication: synchronous coordination → strong consistency → but unavailable during failures.

Dynamo's approach: **optimistic replication** — changes propagate to replicas in the background; concurrent, disconnected work is tolerated.

- **Changes propagate asynchronously** — no blocking for replica acknowledgment before returning to client.
- **Server and network failures tolerated** — writes proceed even if some replicas are down.

### 4.2 When to Resolve Conflicts

The key design question: **when** to resolve update conflicts?

**Traditional approach**: Resolve during **writes** → reject writes that can't be made consistent → limits availability.

**Dynamo's approach**: Resolve during **reads** → writes are **never rejected**:
- Push complexity of conflict resolution to reads.
- Dynamo targets the design space of an **"always writeable"** data store.
- Rejecting customer updates (e.g., "cannot add to cart") is unacceptable.
- *"Writes are never rejected"* — even during network partitions.

### 4.3 Who Resolves Conflicts

Two options:
- **Data store resolves**: Simple policies only (e.g., "last write wins" by timestamp). Limited.
- **Application resolves**: Application knows the data schema and can choose the best resolution method.
  - Example: Shopping cart application can **merge** conflicting cart versions (union of items), ensuring no item is ever lost.
  - Dynamo exposes conflicting versions to the application.

### 4.4 Other Design Principles

- **Incremental scalability**: Can add/remove one storage host (node) at a time with minimal impact.
- **Symmetry**: Every node has the same set of responsibilities — no distinguished coordinator or master. Simplifies provisioning.
- **Decentralization (P2P)**: Avoid centralized control — centralized control leads to single points of failure and outages.
- **Heterogeneity**: Work distribution proportional to node capability — can add powerful nodes without upgrading all.
- **All nodes trusted** (non-hostile environment).

---

## 5. Techniques Overview

Dynamo combines several well-known distributed systems techniques (Table 1 from paper):

| Problem | Technique | Advantage |
|---------|-----------|-----------|
| Partitioning | Consistent Hashing | Incremental scalability |
| High availability for writes | Vector clocks + reconciliation during reads | Version size decoupled from update rates |
| Handling temporary failures | Sloppy Quorum + Hinted Handoff | High availability; durability even when some replicas unavailable |
| Recovering from permanent failures | Anti-entropy using Merkle trees | Synchronizes divergent replicas in background |
| Membership and failure detection | Gossip-based membership protocol | Preserves symmetry; no centralized registry |

---

## 6. System Interface

Dynamo exposes two operations:

```
get(key)
  → returns: object (or list of objects with conflicting versions) + context

put(key, context, object)
  → determines where replicas should be placed
  → writes all replicas to disk
  → context: metadata about the object (version info, vector clock)
```

- **Key**: MD5 hash applied → 128-bit identifier → used to determine which storage nodes are responsible.
- **Context**: Encodes system metadata (version, vector clock); passed back by caller on put to identify which version is being updated. Stored along with object for validation.
- **Value**: Opaque byte array — Dynamo doesn't interpret it.

---

## 7. Partitioning: Consistent Hashing

### 7.1 Why Not Regular Hashing?

Regular hashing (`Hash(key) % N`): Adding/removing a node changes N → almost all keys must be remapped → massive data movement. Not scalable.

### 7.2 Consistent Hashing Basics

- The **output range of the hash function** is treated as a fixed **circular ring** (the hash space wraps around).
- Each key is hashed to a position on the ring.
- Each node is assigned a position on the ring.
- A key is assigned to the **first node clockwise** from its hash position on the ring.
- **Advantage**: Adding/removing a node only affects its immediate neighbors — other nodes are unaffected.

```
          H(Key K) ──────→ position on ring
                    Walk clockwise → first node = coordinator
```

### 7.3 Challenge with Basic Consistent Hashing

1. **Non-uniform load distribution**: Random position assignment leads to uneven key distribution.
2. **Ignores heterogeneity**: Doesn't account for different node capacities.
3. **Load imbalance when nodes join/leave**: Only immediate neighbors rebalance.

### 7.4 Virtual Nodes (Tokens)

Dynamo uses a variant of consistent hashing with **virtual nodes** (also called *tokens*):

- Instead of mapping a physical node to **one** point on the ring, each physical node is assigned **multiple positions** on the ring.
- Each position = one **virtual node** (token).
- A key maps to a virtual node; the virtual node maps to a physical node.
- Two-level mapping:
  - **H(key) = Coordinator VirtualNode** (static — deterministic hash)
  - **Map(VirtualNode) → PhysicalNode** (dynamic — can change as nodes join/leave)

**Strategy 3 (production)**: Divide hash space into **Q equally-sized partitions** (virtual nodes). Each physical node assigned ≈ Q/S virtual nodes, where S = number of physical nodes.

```
Q partitions (virtual nodes) on ring
Each physical node Pi responsible for ~Q/S virtual nodes
```

### 7.5 Advantages of Virtual Nodes

- **Even load distribution**: Each physical node gets ~equal share of keys.
- **Graceful departure**: When a physical node leaves, its virtual nodes are **randomly and uniformly redistributed** to remaining physical nodes → load balancing.
- **Graceful joining**: New physical node **steals** virtual nodes uniformly from existing nodes → doesn't overload a single neighbor.
- **Heterogeneity**: High-capacity nodes can be assigned more virtual nodes proportionally.
- **Decoupled partitioning and placement**: Can change partition placement at runtime without re-hashing.

**Example** (from slides):
- Ring has virtual nodes A, B, C, D, E, F, G, H, I, J.
- Physical nodes P1, P2, P3, P4 each own several virtual nodes.
- Key k9: Hash(k9) → lands near virtual node C → C is owned by physical node P1.

---

## 8. Replication

### 8.1 N-Way Replication

- Each data item is replicated at **N physical nodes** (N is a configurable parameter).
- **Coordinator virtual node** stores the **first copy**.
- The **next N-1 clockwise virtual nodes** in the ring also store copies.
- **Skip virtual nodes belonging to the same physical node** to ensure replicas are on distinct physical machines.
- **Preference list**: The list of **physical nodes** responsible for storing a particular key.
  - Contains more than N nodes to handle virtual node collisions.
  - Example: `PreferenceList(k7) = {B, D, E}` (3 physical nodes).

```
Example with N=3, Key K, Preference list = {B, C, D}:
  - Hash(K) → between A and B on ring
  - B stores first copy (coordinator)
  - C stores second copy
  - D stores third copy
```

### 8.2 Cross-Datacenter Replication

- Preference list nodes are spread across **multiple data centers** (connected via high-speed links).
- Can handle **entire data center failures** without data loss.

---

## 9. Membership and Failure Detection

### 9.1 Gossip Protocol

Membership information (which nodes are alive, which virtual nodes they own) is propagated via **gossip**:

- Each physical node contacts a **randomly chosen peer every second**.
- The two nodes **reconcile their membership change histories** — bidirectional information exchange.
- Result: **Eventually consistent view of membership** and VN→PN mapping across all nodes.
- Every node knows the token ranges handled by its peers → can route `get`/`put` directly to the right node (**zero-hop DHT**).

### 9.2 Seeds

- To avoid logically partitioned rings (two groups that don't know about each other), Dynamo uses **seeds** — nodes known to all other nodes.
- Seeds are discovered via external configuration or static config.
- All nodes reconcile with seeds → logical partitions become highly unlikely.

### 9.3 Failure Detection

- **Transient failures**: If node A can't reach node B, A considers B failed locally (for the purpose of routing) and uses alternative nodes. A periodically retries B.
- **Permanent changes**: Node addition and removal are done **centrally** (admin command) and propagated via gossip. Only permanent changes trigger re-balancing.
- **Key insight**: Failure detection is **purely local** — A only needs to know if B is unreachable for A's own requests. No global consistent view of failure state needed.

### 9.4 Adding/Removing Storage Nodes

- **Adding node X**: X assigned Q/S tokens scattered across ring. For each key range assigned to X, existing nodes transfer the appropriate keys to X. Confirmation round ensures no duplicate transfers.
- **Removing node X**: Its tokens redistributed to remaining nodes in a way that preserves uniform distribution.
- Strategy 3 makes bootstrapping faster (partition files can be relocated as units).

---

## 10. Key-Value Operations

### 10.1 put() and get() Interface

```
put(key, context, object)  ← write: add or update an item
get(key)                   ← read: returns (possibly multiple) versioned values
```

### 10.2 Request Routing

Any physical node in the system can receive a client request. Two routing strategies:
1. **Load balancer**: Client routes to a random Dynamo node → node forwards to correct coordinator if needed (one extra hop).
2. **Partition-aware client library**: Client downloads Dynamo membership state every 10 seconds, routes directly to correct coordinator node (zero extra hop, lower latency).

**Coordinator**: The node that handles a request. If the receiving node is in the **top-N of the preference list** for the key, it acts as coordinator. Otherwise, it **forwards** the request to the first node in the preference list.

### 10.3 put() Execution

1. Coordinator **generates a new vector clock** for the new version.
2. Coordinator **writes the new version locally**.
3. Coordinator **sends the new version** (with vector clock) to the **N highest-ranked reachable nodes** in the preference list.
4. **Returns success** to client once at least **W-1 other nodes** acknowledge (W total including coordinator).
5. The remaining replicas may be updated asynchronously (eventual consistency).

### 10.4 get() Execution

1. Coordinator **requests all existing versions** of data for the key from N highest-ranked reachable nodes.
2. **Waits for R responses** before returning result.
3. If coordinator receives **multiple versions**, it returns all **causally unrelated** versions (concurrent branches) to the client.
4. **Read repair**: If stale versions were returned during the read, coordinator updates those stale nodes with the latest version (opportunistic repair).
5. Client performs reconciliation and writes back the merged version.

---

## 11. Sloppy Quorum (N, R, W)

### 11.1 Quorum Parameters

Dynamo uses three configurable parameters:
- **N**: Number of replicas for each data item.
- **W**: Minimum number of replicas that must acknowledge a **write** before returning success to client.
- **R**: Minimum number of replicas that must respond to a **read** before returning to client.

Latency of a put/get is dictated by the **slowest of the W (or R) responding replicas** — R and W are usually less than N for better latency.

```
Quorum condition: R + W > N
  → guarantees that read and write sets overlap
  → at least one node in any read set has the latest write
  → allows client to merge inconsistencies
```

**Typical configuration**: **(N, R, W) = (3, 2, 2)** — used by majority of Amazon's Dynamo instances.

### 11.2 Consistency vs. Availability Trade-off

| Config | Effect |
|--------|--------|
| R=1, W=N | Writes must reach all replicas (slow writes) but reads are fast |
| R=N, W=1 | Writes are fast (single node), reads need all replicas |
| R=2, W=2, N=3 | Balanced; tolerates 1 node failure for both reads and writes |
| W=1 | Never reject a write as long as 1 node is alive |

### 11.3 Sloppy Quorum vs. Strict Quorum

Traditional (strict) quorum: Only the designated N nodes in the preference list can participate.
**Sloppy quorum**: Use the **first N healthy nodes** from the preference list, even if those are not the designated N nodes.
- If some of the designated N nodes are down/unreachable, use the next healthy nodes in the ring.
- This maintains W/R guarantees even during failures.
- "Sloppy" = not strict about which N nodes — just pick the first N healthy ones.

---

## 12. Hinted Handoff (Handling Temporary Failures)

### 12.1 Problem

If Dynamo used strict quorum membership, it would be unavailable during node failures (couldn't reach W replicas).

### 12.2 Solution: Hinted Handoff

When a node in the preference list is **temporarily unavailable**, a different node accepts the write:

**Example** (N=3, Preference list = {B, C, D}):
1. Node **D** is temporarily down during a `put()`.
2. Instead of failing, the coordinator sends the replica to node **E** (next node in ring).
3. **E stores the replica in a separate local database** (not its regular data).
4. **E has a "hint"** in metadata: "this replica belongs to D, but D was down."
5. When D **recovers**, E **delivers the replica to D** and deletes its local copy.

This ensures:
- **Always N replicas stored somewhere** (though not always the designated nodes).
- **"Always writeable"**: Writes never fail due to temporary node failures.
- Applications needing highest availability can set W=1 — a write succeeds as long as any single node is alive.

### 12.3 Multi-Datacenter Hinted Handoff

Each replica in the preference list is placed on nodes from **different datacenters** → can handle entire datacenter failures using hinted handoff.

---

## 13. Data Versioning and Vector Clocks

### 13.1 The Versioning Problem

Dynamo uses **eventual consistency** — updates propagate asynchronously. A `put()` may return before all replicas are updated. Subsequent `get()` may see old data.

**Dynamo's model**: `put()` is treated as an **append** (not in-place update):
- Creates a new **immutable version** of the object.
- Multiple versions can coexist across replicas.
- The system **does not internally resolve** concurrent conflicting versions — it exposes them to the application.

### 13.2 Why Multiple Versions?

Consider a shopping cart:
- User adds item X on node A → version v1.
- Network partition — A and B diverge.
- User adds item Y on node B → version v2 (based on different state).
- Both versions must be preserved — taking either alone would lose an item.
- Cart application **merges** both versions (union of items).

### 13.3 Vector Clocks

A **vector clock** is a list of **(node, counter) pairs** associated with every version of every object. It captures **causality** between versions.

**Format**: `[(Sx, 2), (Sy, 1), (Sz, 1)]` — Sx handled 2 updates, Sy handled 1, Sz handled 1.

**How it works**:

- **On put()**: The coordinator **increments its counter** in the vector clock for this key.
  - E.g., if Sx coordinates the put, Sx's entry in the vector clock is incremented.
- **Comparing versions** (determining causal relationship):
  - If all counters of version A ≤ all counters of version B (for matching nodes): **A is an ancestor of B** → A causally happened before B → A can be discarded (subsumed).
  - Otherwise (some counters of A > B, some < B): **concurrent writes** → conflict → both versions must be kept.

**Example** (from slides and paper):

```
D1: [(Sx,1)]                  ← Sx writes first version
D2: [(Sx,2)]                  ← Sx writes again; D1 is ancestor of D2
D3: [(Sx,2),(Sy,1)]           ← Sy writes based on D2
D4: [(Sx,2),(Sz,1)]           ← Sz writes based on D2 (concurrent with Sy!)
  → D3 and D4 are CONCURRENT (conflict) → both returned to client
D5: [(Sx,3),(Sy,1),(Sz,1)]    ← Sx reconciles D3+D4, writes merged result
```

| Pair | Relationship |
|------|-------------|
| D1 vs D2 | D1 ancestor of D2 → discard D1 |
| D2 vs D3 | D2 ancestor of D3 → discard D2 |
| D3 vs D4 | Concurrent (conflict) → keep both, return to client |
| D5 | Reconciles D3+D4 |

### 13.4 get() with Multiple Versions

- If `get()` finds multiple causally unrelated versions (concurrent branches), it returns **all of them** along with the context (which contains the combined vector clocks).
- Client performs **semantic reconciliation** (merges the versions using application logic).
- Client writes the reconciled version back with `put()`, passing the context that subsumes all the conflicting branches.

### 13.5 Vector Clock Truncation

Vector clocks can grow large if many nodes coordinate writes. Dynamo limits clock size:
- Each (node, counter) pair also stores a **timestamp** (last time that node updated the item).
- If clock size exceeds threshold (10 pairs), the **oldest pair** is removed.
- Downside: Can reduce reconciliation accuracy (may not detect all ancestor relationships).
- In practice, this hasn't caused issues in production.

---

## 14. Replica Synchronization: Anti-Entropy with Merkle Trees

### 14.1 Problem: Permanent Failures

Hinted handoff handles **transient** failures. For **permanent** failures (node dies permanently), hinted replicas may never be delivered. Replicas can **permanently diverge**.

### 14.2 Anti-Entropy

Dynamo uses **anti-entropy** to keep replicas synchronized in the background:
- Replicas periodically compare their data sets and synchronize any differences.
- Challenge: How to efficiently find which keys differ without transferring all data?

### 14.3 Merkle Trees

Dynamo uses **Merkle trees** (hash trees) for efficient replica comparison:

**Structure**:
- **Leaves**: Hash of the value of each individual key in a virtual node.
- **Internal nodes**: Hash of their children's hashes.
- **Root**: Single hash summarizing all data in the virtual node.

```
Example Merkle tree for VN-A on PN-x:
                    Root: H(H(H(D1),H(D2),...), H(H(D7),H(D8)))
                   /                                              \
        K1-4: H(H(D1),H(D2),H(D3),H(D4))         K5-8: H(H(D5),H(D6),H(D7),H(D8))
       /                    \                       /                   \
 K1-2: H(H(D1),H(D2))  K3-4: H(D3,D4)    K5-6: H(D5,D6)     K7-8: H(D7,D8)
 /         \             /       \          /         \          /         \
K1:H(D1)  K2:H(D2)  K3:H(D3)  K4:H(D4) K5:H(D5)  K6:H(D6) K7:H(D7)  K8:H(D8)
```

**One Merkle tree per virtual node** (not per physical node).

**Synchronization process**:
1. Two nodes exchange the **root hash** of their Merkle tree for a given virtual node.
2. If roots **match**: Trees are identical → no synchronization needed.
3. If roots **differ**: Recursively check children (left child, right child).
4. Continue down the tree until reaching **leaf nodes** with different hashes → those keys are out of sync.
5. Only transfer the differing key-value pairs.

**Advantages**:
- Each branch checked **independently** — no need to download entire tree.
- **Minimizes data transfer**: Only syncs the specific keys that differ.
- **Reduces disk reads**: Quickly narrows down which keys need synchronization.

**Disadvantage**: When a node joins or leaves, many key ranges change → Merkle trees for those ranges must be recalculated (addressed by Strategy 3 partitioning which stores partitions as separate files).

---

## 15. Implementation

### 15.1 Software Components

Each storage node has **three main software components**:
1. **Request coordination**: Event-driven (SEDA architecture); state machine per client request; handles read/write quorum logic, retries, versioning.
2. **Membership and failure detection**: Gossip protocol, ring membership.
3. **Local persistence engine**: Pluggable storage backend.

All implemented in **Java**.

### 15.2 Pluggable Local Persistence

Dynamo can use different storage engines depending on application access patterns:
- **Berkeley DB (BDB) Transactional Data Store**: Handles objects of tens of KB; most production instances use this.
- **BDB Java Edition**
- **MySQL**: Handles larger objects.
- **In-memory buffer with persistent backing store**.

### 15.3 Request Coordination (SEDA Architecture)

- Built on an **event-driven messaging substrate** (SEDA — Staged Event-Driven Architecture).
- Message processing pipeline split into multiple stages.
- All communications via **Java NIO channels**.
- Coordinator state machine:
  1. Send read requests to N nodes.
  2. Wait for minimum required responses (R or W).
  3. If too few replies in time bound → fail the request.
  4. Gather all data versions, determine which to return.
  5. If versioning enabled → perform syntactic reconciliation, generate opaque write context.

### 15.4 Read Repair

After a successful `get()`, the coordinator **updates stale nodes** with the latest version opportunistically. This reduces the load on the anti-entropy process.

---

## 16. Experiences and Lessons Learned

### 16.1 How Amazon Services Use Dynamo

Dynamo instances differ by their **version reconciliation logic** and **read/write quorum characteristics**. Three main usage patterns:

**Pattern 1: Business-logic-specific reconciliation**
- Most common usage.
- Each data item replicated across multiple nodes.
- On divergence, **client application performs reconciliation** using its domain knowledge.
- Example: Shopping cart service **merges** conflicting cart versions (union of items — "add to cart" is never lost, though deleted items may resurface).

**Pattern 2: Timestamp-based reconciliation**
- Client doesn't implement custom reconciliation.
- On divergence, Dynamo performs simple **"last write wins"** based on physical timestamp.
- Example: Session management service — latest session state is always correct.

**Pattern 3: High performance read engine**
- Set R=1, W=N.
- Data rarely updated, reads are extremely frequent.
- Dynamo functions as a replicated, partitioned **read cache** with incremental scalability.
- Example: Product catalog, promotional items.

### 16.2 Balancing Performance and Durability (Write Buffering)

**Problem**: At 99.9th percentile, Dynamo running on commodity hardware has I/O bottlenecks.

**Solution**: Write buffering optimization:
- Each storage node maintains an **in-memory object buffer**.
- Writes are stored in the buffer first, then **periodically flushed to disk** by a writer thread.
- Reads check the buffer first (high hit rate).
- Result: **5× reduction in 99.9th percentile write latency** during peak traffic.

**Trade-off**: A server crash can lose writes in the buffer. Mitigated by having the coordinator choose one replica to do a **"durable write"** directly to disk — coordinator waits only for 1 durable write, not W.

### 16.3 Ensuring Uniform Load Distribution

**Problem**: During low load, imbalance ratio can be as high as 20%; during high load, ~10%.

**Three Partitioning Strategies** (evolution of Dynamo's approach):

**Strategy 1: T random tokens per node, partition by token value**
- Original strategy.
- Each node assigned T random tokens on the ring.
- Ranges vary in size → uneven load.
- Problems:
  - Key range scanning is expensive (random ranges, must read each node's store).
  - When nodes join/leave, key ranges change → must recalculate many Merkle trees.
  - Bootstrapping very slow (can't take a snapshot of entire key space easily).

**Strategy 2: T random tokens per node, equal-sized partitions**
- Hash space divided into Q equal partitions.
- Tokens only used to **order nodes** → map keys to ordered node list.
- Partition placed on first N nodes encountered clockwise from partition end.
- Decouples partitioning from placement.
- Problem: Still has complex membership information.

**Strategy 3: Q/S tokens per node, equal-sized partitions** ← **(Production choice)**
- Q equal-sized partitions.
- Each node gets Q/S tokens.
- When node leaves: its tokens distributed randomly to remaining nodes.
- When node joins: it "steals" tokens uniformly from existing nodes.
- **Advantages**:
  - Strategy 3 achieves the **best load balancing efficiency** (Figure 8).
  - Reduces metadata per node by **3 orders of magnitude** compared to Strategy 1.
  - **Faster bootstrapping**: Partition files can be archived/relocated as units.
  - **Easier archival**: Each partition stored in a separate file → can archive entire key space cleanly.
  - **Changing placement at runtime**: Partitioning scheme can be updated dynamically.

**Load balancing efficiency** = ratio of average requests per node to maximum requests per node. Strategy 3 approaches 1.0 (perfect) even with small metadata.

### 16.4 Divergent Versions: How Often?

In production (shopping cart service profiled over 24 hours):
- **99.94%** of requests saw exactly **1 version** (no conflict).
- **0.00057%** saw 2 versions.
- **0.00047%** saw 3 versions.
- **0.00009%** saw 4 versions.

**Takeaway**: Divergent versions are rare. Most arise from **concurrent robot-automated updates** (not human users), triggered during busy periods.

### 16.5 Client-driven vs. Server-driven Coordination

**Server-driven**: Load balancer selects a random Dynamo node → node acts as coordinator (may add a network hop).

**Client-driven**: Client periodically downloads Dynamo membership state (polls random node every 10 seconds), routes requests directly to the appropriate coordinator.

**Performance comparison** (Table 2 from paper):

| | 99.9th %ile Read Latency (ms) | 99.9th %ile Write Latency (ms) | Avg Read (ms) | Avg Write (ms) |
|---|---|---|---|---|
| **Server-driven** | 68.9 | 68.5 | 3.9 | 4.02 |
| **Client-driven** | 30.4 | 30.4 | 1.55 | 1.9 |

- Client-driven: **~50% lower latency at 99.9th percentile** (eliminates load balancer hop and its variability).
- Client-driven: **3–4ms lower average**.
- Client-driven: Load distribution is implicitly fair (uniform key assignment to nodes).
- Downside: Client can have stale membership for up to 10 seconds.

### 16.6 Balancing Background vs. Foreground Tasks

Each node runs background tasks: replica synchronization (anti-entropy), hinted handoff.

Early problem: Background tasks triggered resource contention → degraded foreground put/get performance.

Solution: **Admission controller** — background tasks run only when foreground operations are not resource-constrained:
- Monitors 99th percentile DB read latency (last 60 seconds) vs. preset threshold (50ms).
- If below threshold → more time slices for background tasks.
- If above threshold → reduce background task allocation.
- Feedback loop maintains balance between background and foreground.

---

## 17. Evaluation Results

### 17.1 Latency (December 2006 Peak Season)

- Measured on a live system with **(N=3, R=2, W=2)**, ~couple hundred nodes.
- **Average read/write**: ~3-4ms.
- **99.9th percentile read/write**: ~10-100ms (order of magnitude higher than average).
- **Latencies follow a diurnal pattern** (higher during day, lower at night) — matches request rate.
- 99.9th percentile latencies affected by: variability in request load, object sizes, locality patterns.

### 17.2 Key Insight: 99.9th Percentile Target

Traditional systems optimize for averages. Dynamo optimizes for **the tail** (99.9th percentile):
- Average latencies tend to be low (~4ms) because Dynamo's storage engine caches hot items.
- Tail latencies (99.9th %) are an order of magnitude higher.
- Load balancers and network add variability → higher tail latency in server-driven coordination.
- Write latencies always slightly higher than read latencies (writes go to disk; reads may hit cache).

---

## 18. Summary: Dynamo's Key Techniques

### 18.1 The Complete Picture

```
Client
  │
  ├──put(key, context, object)──→ [Coordinator PN]
  │                                     │
  │                             Vector clock update
  │                                     │
  │                             ┌───────┴────────┐
  │                         Write to          Write to
  │                         Replica 1        Replica 2
  │                              ...          (W-1 acks)
  │                                     │
  │◄────────────────── success ──────────┘
  │
  ├──get(key)──────────────────→ [Coordinator PN]
  │                                     │
  │                      Request from N healthy replicas
  │                                     │
  │                         Wait for R responses
  │                                     │
  │                    If multiple versions: return all
  │◄─── versioned values + context ──────┘
  │
  │  [Client reconciles, writes back D5 = merged]
```

### 18.2 Techniques Summary

| Technique | How it works | Why it matters |
|-----------|-------------|----------------|
| **Consistent Hashing** | Hash space as ring; keys assigned to first clockwise node | Incremental add/remove of nodes |
| **Virtual nodes** | Physical nodes own multiple positions; Q equal partitions | Load balance; heterogeneity; graceful failures |
| **N-way replication** | Each key stored at N physical nodes (preference list) | Fault tolerance and durability |
| **Sloppy Quorum (N,R,W)** | Write to W, read from R; R+W>N | Availability + tuneable consistency |
| **Hinted Handoff** | Temporary node down → another node holds replica with hint | "Always writeable" even during failures |
| **Vector Clocks** | (node, counter) pairs track causality between versions | Detect concurrent writes; enable reconciliation |
| **Semantic Reconciliation** | Application merges conflicting versions | Never lose data; business-logic-aware |
| **Merkle Trees** | Hash tree per VN; compare roots to find divergent keys | Efficient anti-entropy; minimal data transfer |
| **Gossip Protocol** | Each node gossips with random peer every second | Decentralized membership; no SPOF |

### 18.3 Key Takeaways

1. **"Always writeable"**: Writes are never rejected. Conflict resolution pushed to reads. Complexity goes to the application.

2. **AP system**: Dynamo deliberately chooses Availability + Partition Tolerance over Consistency. It is **eventually consistent** — all replicas converge eventually if no new writes.

3. **Tuneable consistency**: Via (N, R, W) parameters — same system can be configured for different consistency/availability/durability trade-offs.

4. **Decentralized by design**: No master, no distinguished coordinator, no central registry. Symmetry of responsibility. Avoids single points of failure.

5. **Production success**: Provided 99.9995% successful responses (without timeout) over two years. No data loss events. SLAs maintained even during holiday peak traffic.

6. **Dynamo ≠ DynamoDB**: Dynamo is an internal Amazon system. AWS DynamoDB is a commercial product inspired by but different from Dynamo.

---

## 19. Key Concepts Summary Table

| Concept | Key Point |
|---------|-----------|
| **Consistent hashing** | Ring-based hash space; key → first clockwise node; incremental scalability |
| **Virtual nodes (tokens)** | Each physical node → multiple positions; Q equal partitions; load balance |
| **Preference list** | N physical nodes responsible for a key; coordinator = first in list |
| **N, R, W** | Replication factor; read quorum; write quorum; R+W>N for consistency |
| **Sloppy quorum** | Use first N *healthy* nodes, not strictly designated N nodes |
| **Hinted handoff** | Down node's write goes to a substitute with a "hint"; delivered on recovery |
| **Vector clock** | (node, counter) pairs per version; detects causality vs. concurrency |
| **Syntactic reconciliation** | System detects ancestor → drops older version automatically |
| **Semantic reconciliation** | Client merges concurrent conflicting versions (e.g., cart merge) |
| **Merkle tree** | Hash tree per virtual node; efficient background replica sync |
| **Anti-entropy** | Background process using Merkle trees to sync divergent replicas |
| **Gossip protocol** | Decentralized membership; each node gossips with random peer every second |
| **Seeds** | Known nodes; prevent logical ring partitions during gossip |
| **Read repair** | After get(), coordinator refreshes stale replicas opportunistically |
| **Strategy 3** | Q/S tokens per node, equal partitions; best load balance, production choice |
| **Write buffering** | In-memory buffer + async flush; 5× lower 99.9th percentile write latency |
| **Client-driven coordination** | Client routes directly to coordinator; ~50% lower tail latency vs. server-driven |

---

## References

- DeCandia, G. et al., "Dynamo: Amazon's Highly Available Key-value Store", ACM SOSP 2007
- Slides: M4.NoSQL.pdf, slides 60–90 (DS256, IISc)


# Lecture 4.3: Other NoSQL Databases — HBase, Neo4J, and Data Lakes

## DS256 - Scalable Systems for Data Science
### Module 4: NoSQL Databases

> **References**:
> - Lecture slides: M4.NoSQL.pdf (slides 80–121)
> - Apache HBase Reference Guide: hbase.apache.org/book.html
> - Graph Databases for Dummies: Neo4j Edition, Dr. Jim Webber & Rik Van Bruggen, 2020
> - Data lake management: challenges and opportunities, VLDB Endowment, August 2019

---

## Part 1: HBase — Columnar Wide-Column Store

---

## 1. Background: Google BigTable → HBase

**Google BigTable** is Google's internal distributed storage system for structured data (introduced 2006). It powers products like Google Search (web crawl data), Gmail, Google Maps, and YouTube.

**HBase** is the **open-source variant** of BigTable, built on top of Apache Hadoop. It lives in the Hadoop ecosystem and uses HDFS as its underlying storage layer.

### Why Not a Relational Database?

Consider storing web crawl data for CNN: each URL (row) has content, anchor text, metadata — but the number and names of columns vary wildly between pages. A relational DB would need thousands of nullable columns or complex normalization. HBase handles this naturally via its flexible column model.

---

## 2. HBase Data Model

HBase is a **columnar** or **wide-column store**. It distributes data across machines while still giving you the concept of a table with rows.

### Core Concepts

| Concept | Description | Example |
|---------|-------------|---------|
| **Row Key** | Unique identifier for a row (like primary key in RDBMS) | `"Boboo%a_banana"` |
| **Column Family** | Fixed, predefined group of related columns (schema-fixed) | `SOCIAL_NETWORK`, `ACTIVITY` |
| **Column Key / Qualifier** | Uniquely identifies a column within a family (dynamic, varies per row) | `SN_NAME`, `USER_ID`, `TYPE` |
| **Version** | Timestamp — keeps multiple copies of a cell over time | `t3`, `t5`, `t8` |
| **Cell** | Single value at a (row-key, column-family, column-key, version) coordinate | `"Nice pic, bro!"` |

### Key Insight: Variable Columns

Unlike RDBMS where every row has the same columns, in HBase:
- **Column families are fixed** (defined at table creation time) — think of them as groups
- **Column keys within a family are dynamic** — each row can have different columns
- **Versions allow time-travel** — you can retrieve data "as of" a certain timestamp

### Example from Google BigTable: Web Crawl

```
Row Key: "com.cnn.www"
  Column Family 'contents':
    contents: (t6) <html>...
    contents: (t5) <html>...   ← multiple versions of HTML
    contents: (t3) <html>...
  Column Family 'anchor':
    anchor:cnnsi.com (t9) "CNN"
    anchor:my.look.ca (t8) "CNN.com"
```
- Row key = reversed URL (`com.cnn.www`)
- `contents:` family stores page HTML at different timestamps
- `anchor:` family stores what other sites link to CNN as

---

## 3. HBase Data Model Example: Social Network

The slides show a social network use case very well. The table has **two column families**:
- `SOCIAL_NETWORK` — user identity info
- `ACTIVITY` — what the user did

| ROW KEY | SN NAME | USER ID | PAGE ADDRESS | TYPE | DATE | TEXT |
|---------|---------|---------|--------------|------|------|------|
| `Boboo%a_banana` | Boboo | a_banana | boboo.com/a_peach | Like | 6/3/2018 | |
| `Boboo%a_pineapple` | Boboo | a_pineapple | boboo.com/a_banana | Comment | 6/13/2018 | Nice pic, bro! |
| `Boboo%a_watermelon` | Boboo | a_watermelon | boboo.com/a_pineapple | Comment | 6/10/2018 | Hey, that's my jacket! |
| `Chiching%a_cucumber` | Chiching | a_cucumber | chiching.com/a_kohlrabi | Comment | 5/25/2018 | Wow! What a bike! |
| `Chiching%a_kohlrabi` | Chiching | a_kohlrabi | chiching.com/a_kohlrabi | Comment | 5/25/2018 | Yeah, I know :D |

**Design decisions visible here:**
- **Row key = `username%post_id`** — encoding user + post creates a natural range scan for all of Boboo's activities (all rows starting with `Boboo%` are co-located)
- **Column families split identity vs. activity** — physical separation enables reading only the ACTIVITY family without fetching the SN_NAME/USER_ID columns
- **TEXT column is empty for "Like"** — no wasted space in HBase (sparse columns are free)

---

## 4. HBase Operations

### 4.1 Get

Retrieve all columns of a specific row (by exact row key).

```java
// Fetch a specific row with filters
Get get = new Get(Bytes.toBytes("Boboo%a_banana"));
get.addFamily(Bytes.toBytes("ACTIVITY"));         // only ACTIVITY column family
get.addColumn(Bytes.toBytes("ACTIVITY"),
              Bytes.toBytes("TYPE"));              // only the TYPE column
get.setTimeRange(minTimestamp, maxTimestamp);       // optional: specific version range
Result result = table.get(get);
```

**Options:** `addFamily` (get all columns in a family), `addColumn` (get one specific column), `setTimeRange`/`setTimestamp` (for versioned data)

### 4.2 Put

Add a new row (if key doesn't exist) or update an existing row (if key exists).

```java
Put put = new Put(Bytes.toBytes("Boboo%a_grape"));
put.addColumn(Bytes.toBytes("SOCIAL_NETWORK"),
              Bytes.toBytes("SN_NAME"),
              Bytes.toBytes("Boboo"));
put.addColumn(Bytes.toBytes("ACTIVITY"),
              Bytes.toBytes("TYPE"),
              Bytes.toBytes("Share"));
table.put(put);
```

### 4.3 Delete

Remove a row from a table. (Note: HFiles are immutable — delete uses a **Tombstone marker**, which is resolved during compaction.)

```java
Delete delete = new Delete(Bytes.toBytes("Chiching%a_kohlrabi"));
table.delete(delete);
```

### 4.4 Scan

Iterate over **multiple rows** matching a prefix or range — the most powerful operation for range queries.

```java
// Returns rows beginning with "row" within column families "cf" and "attr"
public static final byte[] CF   = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Table table = ...    // instantiate a Table instance

Scan scan = new Scan();
scan.addColumn(CF, ATTR);
scan.setRowPrefixFilter(Bytes.toBytes("row"));   // prefix filter: all rows starting with "row"
ResultScanner rs = table.getScanner(scan);
try {
    for (Result r = rs.next(); r != null; r = rs.next()) {
        // process result...
    }
} finally {
    rs.close();   // always close the ResultScanner!
}
```

**Social network example:** To get all of Boboo's activities, scan with prefix `"Boboo%"`. All relevant rows are co-located because of the row key design.

---

## 5. HBase Data Layout

Data in an HBase table is **partitioned by row-key ranges**, not hashed (unlike Dynamo).

```
HBase Table
├── HRegion 1 (row keys: "Aardvark%" → "Mango%")
│   ├── HStore (SOCIAL_NETWORK column family)  ← in-memory + spills to HFile
│   └── HStore (ACTIVITY column family)
├── HRegion 2 (row keys: "Mango%" → "Zebra%")
│   ├── HStore (SOCIAL_NETWORK column family)
│   └── HStore (ACTIVITY column family)
└── HLog (write-ahead log, per HRegionServer)
```

### Hierarchy

| Level | Name | Description |
|-------|------|-------------|
| Table | — | Logical table |
| Partition | **HRegion** (tablet) | A contiguous row-key range; unit of distribution and load balancing |
| Per-column-family storage | **HStore** | One HStore per column family per HRegion; smallest unit of storage |
| In-memory buffer | **MemStore** | Fast in-memory writes; flushed to disk when full |
| On-disk file | **HFile** | Immutable sorted file on HDFS |
| Recovery log | **HLog** | Write-ahead log (WAL) for durability |

### HRegion Splitting

When an HRegion grows too large (rows exceed capacity), it is **split into two HRegions**. The Master then assigns the new regions to different HRegionServers for load balancing.

> **Analogy:** Think of HRegions like pages in a phone book. When a page gets too full (too many people with names starting with "Sm"), you split it into two pages.

### Append-Only Model + Compaction

HBase uses an **append-only model** — data is never updated in place:
- Updates create new versions (new timestamps)
- Deletes create Tombstone markers
- **Compaction** (background process) physically merges HFiles, applies deletes/updates, and removes old versions

---

## 6. Cluster Architecture

### 6.1 Three Key Entities

```
                Client
                  |
              Zookeeper   ←——— HMaster
             /         \           |
    HRegionServer     HRegionServer
    ┌─────────────┐   ┌─────────────┐
    │ HLog        │   │ HLog        │
    │ HRegion1    │   │ HRegion2    │
    │  MemStore   │   │  MemStore   │
    │  StoreFile  │   │  StoreFile  │
    │  HFile      │   │  HFile      │
    └──────┬──────┘   └──────┬──────┘
           |                 |
    [DFS Client]       [DFS Client]
    /    |    \         /    |    \
DataNode DataNode DataNode DataNode DataNode
                 (HDFS)

Note: HBase is a CLIENT to HDFS — it uses HDFS for persistent storage
```

### 6.2 HRegionServers

- Store **HRegions** (partitions of the table) on HDFS
- Handle all read/write requests from clients for their assigned regions
- Each server runs an **HLog** (write-ahead log) shared across all its regions

### 6.3 META Table

The **META table** is a special HBase catalog table:
- Maps: `(region name, starting row key)` → `region ID`
- Clients read the META table to find which HRegionServer owns the region for a given row key
- Updated after **each write** — always reflects current region assignment

**Example flow:** Client wants to read row `"Boboo%a_banana"`:
1. Client queries META table: "Which server has the region containing `Boboo%a_banana`?"
2. META says: "HRegionServer-3 has rows from `Boboo%` to `Charles%`"
3. Client contacts HRegionServer-3 directly

### 6.4 Master (HMaster)

- Stores **metadata** that maps data to region servers
- **Monitors health** of all HRegionServers (via heartbeats through Zookeeper)
- Can run **multiple Masters** — one is elected leader via Zookeeper
- **Dynamically migrates regions** to idle region servers for load balancing
- If an HRegionServer fails, Master reassigns its regions to other servers

---

## 7. HBase Write and Read Path

### 7.1 Write Path

```
Client writes "put(Boboo%a_grape, ...)"
         │
         ▼
1. HLog (write-ahead log) — commit log written first for durability
         │
         ▼
2. MemStore (in-memory) — fast random write
         │
         ▼  (when MemStore is full)
3. HFile (on HDFS) — batched, sorted write with index
```

**Performance trick:** Writing randomly to memory (MemStore) is fast. When flushing to disk, the writes are **sorted by row key and batched** — sequential disk writes are far faster than random disk writes.

**HFile structure:** Data blocks + index at end + in-memory index for fast lookups.

### 7.2 Immutability and Tombstones

- **HFiles are immutable once written** — you cannot modify an existing HFile
- **Updates** = new version written to MemStore, later flushed to a new HFile
- **Deletes** = a **Tombstone marker** written to MemStore; the old data still exists in older HFiles until compaction
- **Compaction** physically merges HFiles, dropping old versions and applying Tombstones

### 7.3 Read Path

```
Read request for row "Boboo%a_banana"
         │
         ▼
1. MemStore (most recent writes, in memory) — fastest
         │ (if not found)
         ▼
2. LRU BlockCache (recently read HFile blocks, in memory)
         │ (if not found)
         ▼
3. HFile on disk (HDFS) — slowest but persistent
```

**Why this order?** MemStore has the most recent writes (which override older data). The BlockCache is a read-optimized cache. HFiles are the ground truth on disk.

---

## 8. ACID Semantics in HBase

HBase provides **strong guarantees at the row level**, but weaker across multiple rows.

### 8.1 Atomicity

- **Row-level atomic:** A `Put` either wholly succeeds or wholly fails — even if it writes across multiple column families. If you're writing `SOCIAL_NETWORK` + `ACTIVITY` data for a row in one Put, they both succeed or both fail.
- **NOT atomic across rows:** APIs that mutate multiple rows are NOT atomic. If you update rows for both Boboo and Chiching, one might succeed and the other fail.
- Mutations within a row happen in a **well-defined order**.

> **Example:** `Put("Boboo%a_grape", {SN_NAME: "Boboo", TYPE: "Share"})` — either both columns are written or neither is. You will never see a partial write with SN_NAME present but TYPE missing.

### 8.2 Consistency

- All rows will reflect a **complete row that existed at some point in the table's history** — you will never read an intermediate/partial state.

### 8.3 Consistency of Scans

- Scans do **NOT exhibit snapshot isolation** (unlike PostgreSQL/MySQL SERIALIZABLE).
- Any row returned by a scan will have existed at some point in time.
- Scans must include all data written **prior** to the scan starting.
- **Implication:** Two rows returned by the same scan may reflect different moments in time — no global "snapshot" across the whole scan.

### 8.4 Visibility

- A successful mutation is **immediately visible to that client**.
- Reads on a row return a **subsequence of writes** in the order they happened — monotonically forward movement. You will never read an older value after reading a newer one.

> **Example of monotonic reads:** If Boboo's comment is updated three times (v1→v2→v3), a client that reads v2 will only ever see v2 or v3 on subsequent reads — never v1 again.

### 8.5 Durability

- If a write returns "success", it is **durable** — it has been recorded to the HLog.
- Any data that can be read is also durable.

### Summary Table

| Property | HBase Guarantee |
|----------|----------------|
| Atomicity | Row-level only (NOT multi-row) |
| Consistency | Each row reflects a valid historical state |
| Isolation | No snapshot isolation for scans |
| Visibility | Immediate + monotonically forward |
| Durability | Yes — WAL ensures success = durable |

---

## 9. HBase Feature Summary

| Feature | Details |
|---------|---------|
| **Data Model** | Columnar store: row-keys + column-families + column-keys |
| **Access Pattern** | Data accessed via row-keys |
| **Partitioning** | By row-key ranges (HRegions) + column families (HStores) |
| **Distribution** | Distributed layout on HRegionServers |
| **Routing** | Lookups via META catalog table |
| **Persistence** | HDFS (reliability via HDFS replication) |
| **Consistency** | Strong consistency at row-level (one logical copy) |
| **Performance** | Fast put using in-memory MemStore |

---

## Part 2: Neo4J Graph Database (Overview)

---

## 10. Graph Data Structures

A graph is a collection of **nodes (vertices)** and **edges (relationships)** between them. Different types of graphs:

| Type | Description | Example |
|------|-------------|---------|
| **Simple Graph** | One relationship type between any pair of nodes | Social graph (friends) |
| **Multigraph** | Multiple different relationship types between node pairs | Two people who are both friends AND colleagues |
| **Pseudograph** | Like multigraph, plus **loops** (a node can relate to itself) | A person follows themselves |
| **Connected** | Path exists between every pair of nodes | Social network of one friend group |
| **Disconnected** | Some nodes have no path to others | Network with isolated components |
| **Weighted** | Edges have numeric values (cost, distance, etc.) | Road network with travel times |
| **Unweighted** | All edges are equal | Friendship graph |
| **Directed** | Edges have a direction (A→B ≠ B→A) | Twitter follows (A follows B doesn't mean B follows A) |
| **Undirected** | Edges are bidirectional | Facebook friends (symmetric) |

### Network Topology Types

- **Random:** Average distributions, no hierarchical structure
- **Small-World:** High local clustering + short average path lengths ("six degrees of separation"); hub-and-spoke architecture
- **Scale-Free:** Hub-and-spoke architecture preserved at multiple scales; high power-law distribution (a few very well-connected nodes, many with few connections) — e.g., the internet, citation networks

---

## 11. Graph Algorithms vs. Graph Queries

Both work on graphs but answer different questions:

**Unweighted shortest path** = minimum number of hops (edges traversed)
- Graph: A–B, A–C, C–D, D–E
- Shortest path A→E: A→D→E = **2 hops**

**Weighted shortest path** = minimum total edge cost
- Same graph with weights: A→C = 20, C→D = 20, D→E = 10 (total = 50) vs. A→D = 60, D→E = 10 (total = 70)
- Shortest path A→E: A→C→D→E = **cost 50** (not 2 hops, but cheapest)

> **Key insight:** The "shortest" path depends on what you're optimizing — hops or cost. Algorithms like Dijkstra/Bellman-Ford handle weighted paths; BFS handles unweighted.

---

## 12. Property Graph Data Model

Neo4J uses the **Property Graph** model:

### Vertices (Nodes)
- Have a **unique ID**
- Have a **type/label** (e.g., `:Person`, `:Product`, `:Warehouse`)
- Have **name:value properties** (e.g., `name: 'Rosa'`, `location: 'Melbourne'`)

### Edges (Relationships)
- Have a **direction** (always directed in Neo4J)
- Have a **type** (e.g., `:FRIEND`, `:BUYS`, `:STORED`, `:LOCATED_IN`)
- Can have **properties** (e.g., `date: 22/05/2020`, `cost: 10.00`)
- Supports **multi-graph** (multiple edges between same pair of nodes)

### Logistics Network Example

```
Warehouse(Melbourne) ←—TRAIN(time:10, cost:1)—— Warehouse(Sydney)
                                  |
                               ROAD(time:14, cost:0.5)
                                  |
                              Customer(Rosa)
                                  |
                               SEARCHED
                                  |
                            Classification(headphones)
                                  |
                              TYPE_OF
                                  |
                             Product(type:in-ear, brand:acme)
```

Customers can be friends (`FRIEND`), buy products (`BOUGHT`), products are stored in warehouses (`STORED`), and classified by type (`TYPE_OF`).

### People-Buying-Products Example

Tabular data:
```
Emil Eifrem   → Phone     (22/05/2020)
Emil Eifrem   → Computer  (20/05/2020)
Jim Webber    → Computer  (08/05/2020)
Rik Van Bruggen → Bike    (06/05/2020)
```

Graph model: `(Person {name:'X'}) -[:BUYS {date:Y}]-> (Product {type:'Z'})`

---

## 13. Why Not Model Graphs in RDBMS? The JOIN BOMB

Consider modeling a simple network (Server–App–User–DB–VM) in a relational DB:
- Every many-to-many relationship needs its own **join table**
- Complex relationships lead to an **explosion of join tables** (the "JOIN BOMB")
- Querying "which users are 3 hops from this server" requires deeply nested JOINs
- Performance degrades severely with depth of traversal

**Example:** The slide shows a relational schema for a telecom network with tables like `Person`, `Calls`, `WorkedFor`, `WorkingFor`, `Person_language`, `City_Country`, `Person_operator`, `Operator_rebrand`... all requiring JOINs to query relationships.

**Graph DB advantage:** Relationships are **first-class citizens** (edges), not second-class join tables. Deep traversal is efficient by design.

---

## 14. Cypher Query Language

Cypher is Neo4J's **declarative, pattern-matching** query language. You describe the graph pattern you're looking for, not how to find it.

### Basic Pattern Syntax
- `(n:Label {prop:'val'})` = a node
- `-[:RELATIONSHIP]->` = a directed edge
- `-[:RELATIONSHIP*1..2]->` = variable-length path (1 to 2 hops)
- `(a)<-[:REL]-(b)` = edge going the other way

### Example 1: Direct Friends of Alice (1-hop)

```cypher
MATCH (:Person {name:'Alice'})-[:FRIEND_OF]->(p:Person)
RETURN p
```

Returns all Person nodes that Alice directly points to via `FRIEND_OF` edges.

### Example 2: Friends and Friends-of-Friends (1 or 2 hops), excluding Alice herself

```cypher
MATCH (a:Person {name:'Alice'})-[:FRIEND_OF*1..2]->(p:Person)
WHERE a <> p
RETURN p
```

`*1..2` means "traverse 1 to 2 FRIEND_OF edges". `WHERE a <> p` excludes Alice from results (since she could reach herself via a cycle).

### Example 3: Chaining Paths — What did Alice's friends buy?

```cypher
MATCH (a:Person {name:'Alice'})
      -[:FRIEND_OF*1..2]->(p:Person)
      -[:BOUGHT]->(prod:Product)
WHERE a <> p
RETURN prod
```

This chains two path patterns:
1. Find friends/friends-of-friends of Alice
2. From those people, follow BOUGHT edges to Product nodes

> **Real use case:** Product recommendations — show products bought by people within Alice's social circle.

### Example 4: Aggregation and Sorting

```cypher
-- Average age of Alice's 1-2 hop friends
MATCH (a:Person {name:'Alice'})-[:FRIEND_OF*1..2]->(p:Person)
WHERE a <> p
RETURN avg(p.age)

-- Sum of ages along a path
RETURN reduce(totalAge = 0, n IN nodes(p) | totalAge + n.age) AS totalAges

-- Top 10 oldest friends, descending
RETURN p ORDER BY p.age DESC LIMIT 10
```

### Example 5: Global Query (no starting point)

```cypher
MATCH (app:Application)<-[:USES]-(proc:Process)-[:USED_BY]->
      (bl:BusinessLine)-[:LOCATED_IN]->(b:Building)
RETURN DISTINCT app.name AS Application, b.name AS Building
ORDER BY app.name ASC;
```

**"Which applications are used in specific buildings of the corporation?"**

This is a **global query** — no anchoring node. It must scan the entire graph. **Costly!** Graph DBs prefer queries with a known starting point.

### Example 6: ShortestPath Algorithm

```cypher
MATCH (b:Building {name:"Loc_100"}), (rto:RTO {name:"0-2 hrs"})<-
      [:BUSINESSPROCESS_HAS_RTO]-(bp:BusinessProcess)
WITH b, bp
MATCH p = ShortestPath(b-[*..3]-bp)
RETURN p;
```

**"What business processes with RTO of 0-2 hours would be affected by a fire at Loc_100?"**

Uses two anchor nodes (Building + RTO) and the `ShortestPath` algorithm with depth limit of 3.

### Loading Data from CSV

```cypher
LOAD CSV WITH HEADERS FROM "personbuysproduct.csv" AS csv
MERGE (p:Person {name: csv.Person})        -- create Person if not exists
MERGE (pr:Product {name: csv.Product})     -- create Product if not exists
CREATE (p)-[b:BUYS {date: csv.Date}]->(pr) -- create BUYS relationship
RETURN "Import Successful!"
```

`MERGE` = create-if-not-exists (idempotent); `CREATE` = always create new.

---

## Part 3: Data Lakes

---

## 15. Data Warehousing: Background

Before data lakes, organizations used **data warehouses** — centralized repositories of clean, structured data for analytics. Two approaches:

### 15.1 Bottom-Up Approach (Data Mart)

Build one **data mart** (subject-specific warehouse) first, add more over time.

```
     Sales Mart    → expanded to → Marketing Mart → HR Mart
        ↑                             ↑               ↑
   (from sales           (from marketing          (from HR
    transact. DB)          tools)                  systems)
```

**Pros (green):**
- Relatively inexpensive and easy to implement
- Good **Proof of Concept (POC)** for data warehousing

**Cons (red):**
- Creates **silos of information** — Sales Mart and Marketing Mart may define "customer" differently!
- Postpones difficult decisions (schema design, integration logic)
- Requires an overall integration plan eventually

> **Real example:** Company A builds a sales data mart. Two years later they add a marketing mart, but "customer ID" in sales doesn't match "customer ID" in marketing (different systems). Now integration is a nightmare.

### 15.2 Top-Down Approach (Enterprise-Wide Warehouse)

Build a **comprehensive warehouse first**, then derive data marts from it.

```
 All Source Systems → [Central Enterprise Warehouse] → Sales Mart
                                                     → Marketing Mart
                                                     → HR Mart
```

**Pros (green):**
- **Integrated** — single source of truth, consistent definitions
- **Scalable** — all marts built on same foundation

**Cons (red):**
- **Expensive and time-consuming** upfront
- **Prone to failure** — if the initial warehouse design is wrong, everything downstream breaks

---

## 16. Extract-Transform-Load (ETL)

ETL is the **"plumbing" work** of data warehousing — how data moves from source systems into the warehouse.

```
[Source Systems]                [Staging Area]              [Data Warehouse]

  CRM Database   ──Extract──→   Raw Data   ──Transform──→   Clean Tables
  Sales DB       ──Extract──→   Raw Data   ──Transform──→   Clean Tables
  Log Files      ──Extract──→   Raw Data   ──Transform──→   Clean Tables
                                               │
                                            Load ↓
                                      [Data Warehouse]
```

| Stage | What Happens | Example |
|-------|-------------|---------|
| **Extract** | Get data from various sources | Pull sales records from Oracle DB, read CSV files from marketing tools |
| **Transform** | Convert format, clean data | Convert dates to YYYY-MM-DD, merge customer records from two systems |
| **Load** | Insert into target warehouse | Insert clean records into the reporting database |

ETL is **costly and time-consuming** — often 70–80% of a data warehouse project's effort.

---

## 17. "Dirty" Data

Before data can be warehoused, it must be cleaned. **Dirty data** types:

| Type | Description | Example |
|------|-------------|---------|
| **Dummy Values** | Placeholder data that isn't real | `DOB: 01/01/1900` (used when unknown) |
| **Missing Data** | NULL or absent fields | `phone_number: NULL` for half the records |
| **Multipurpose Fields** | One column used for multiple things | `notes` field contains phone numbers, dates, or addresses |
| **Cryptic Data** | Codes only some people understand | `status: "X"` — what does X mean? |
| **Contradicting Data** | Same fact stored differently in two places | CRM says customer is in Mumbai, billing says Delhi |
| **Inappropriate Use of Address Lines** | Packing extra data into address fields | Address line 2: "Attn: John Smith, dept 42" |
| **Violation of Business Rules** | Data that breaks domain logic | Order date after delivery date |
| **Reused Primary Keys** | A key reassigned after a record is deleted | Customer ID 1234 was deleted and reassigned to new customer |
| **Non-Unique Identifiers** | Multiple records with same supposed-unique key | Two customers with email john@example.com |
| **Data Integration Problems** | Conflicts when merging from multiple systems | Same product called "Widget" in one system, "WDGT-X" in another |

---

## 18. Data Cleaning Steps

Data cleaning transforms dirty raw data into reliable, consistent information:

| Step | Description | Example |
|------|-------------|---------|
| **Parsing** | Break fields into components | Split `"John Smith"` → `first: "John"`, `last: "Smith"`; parse `"123 Main St, Apt 4"` into street/apartment |
| **Correcting** | Fix errors using algorithms | Validate and correct PIN codes using geographic lookup; spell-check product names |
| **Standardizing** | Convert to consistent format | All dates → `YYYY-MM-DD`; all phone numbers → `+91-XXXXX-XXXXX` |
| **Matching** | Detect and correlate duplicates | `"John Smith, 42 Oak Lane"` = `"J. Smith, 42 Oak Ln"` → same person |
| **Consolidating** | Merge all records into standard architecture | Combine customer records from CRM + ERP into single customer master |

---

## 19. Data Warehousing Solutions: Databases

Traditional databases used for warehouses:
- **Relational model** with SQL querying
- **OLTP** (Online Transaction Processing): low latency, thousands of requests/second
- **OLAP** (Online Analytical Processing): batch processing, reporting, and analysis
- **ACID guarantees**
- Storage and compute are **co-located** in same system

### Challenges of Database-Backed Warehouses

| Challenge | Why It Hurts |
|-----------|-------------|
| **Limited scalability** | Single-machine or tightly-coupled cluster limits; hard to scale to petabytes |
| **Rigid schema** | Semi-structured data (JSON logs, XML) doesn't fit cleanly into tables |
| **No non-SQL analytics** | Can't run ML models, graph algorithms, or custom code natively; User-Defined Functions (UDFs) are complex |

---

## 20. What is a Data Lake?

A **Data Lake** is a **massive collection of datasets** that:
- May be **hosted in different storage systems** (HDFS, cloud blob storage, Cassandra, etc.)
- May **vary in their formats** (CSV, JSON, Parquet, images, logs, PDFs)
- May have **limited or missing metadata** (no schema documentation)
- May **change autonomously over time** (datasets updated independently by producers)

> **Analogy:** A data warehouse is like a **bottled water store** — everything is clean, labeled, standardized, ready to use. A data lake is like a **natural lake** — everything flows in from various sources in various forms; you need to filter and process it yourself before drinking.

---

## 21. Data Lake Benefits

### 21.1 Decouples Producers from Consumers

In a traditional warehouse, ETL must run before analytics teams can access data. In a data lake:
- **Operations team** (producers) writes raw data directly
- **Analytics team** (consumers) reads and processes it independently
- They can be in **different enterprises** — producers and consumers don't need to coordinate

### 21.2 Independent Processing

Each consumer can apply **their own transformation logic**. The same raw server logs might be:
- Used by the security team to detect intrusions
- Used by the ops team for capacity planning
- Used by ML engineers to train anomaly detection models

All consuming the same raw data, transforming it differently.

### 21.3 Scalable Platform

Data lakes are built on **distributed big data platforms** (not single-machine RDBMS) — designed to scale to petabytes.

### 21.4 Flexible Data Source

- Data lakes can **feed data warehouses** (data lake as a staging layer)
- Or support **one-off analyses** (ad hoc explorations without needing a warehouse)

### 21.5 Manages Transient Data

Handles data that doesn't need permanent storage — stream it in, process it, discard it (e.g., real-time sensor readings you only need for 24 hours).

---

## 22. Data Lake Architecture

A data lake has multiple stages/services working together:

```
[Diverse Data Sources]
  DB snapshots, API feeds, IoT sensors, log files, images
           │
           ▼
    ┌─────────────────────────────────────────────────────────┐
    │                      DATA LAKE                         │
    │                                                         │
    │  [Ingest & Extract]  →  [Clean]  →  [Versioned Store]  │
    │       │                                  │              │
    │  JSON / CSV / Parquet files          Metadata, Indices  │
    │                                          │              │
    │                              [Discovery & Integration]  │
    │                               Unified Data Model        │
    └─────────────────────────────────────────────────────────┘
           │
           ▼
    [Data Consumers: Analytics, ML, Warehouses]
```

### Component Details

#### 22.1 Data Ingest
- Accept data from **diverse sources and formats**
- **Indexing** for later retrieval
- **Versioning** — track when datasets arrived/changed
- **Basic sanity checks:**
  - Duplicate detection (same dataset ingested twice?)
  - Version evolution (is this a new version of an existing dataset?)

#### 22.2 Data Extraction
- Convert **raw data → data model**
- Both **manual** (human-written extraction scripts) and **automated** (rule-based extraction)
- Example: extract customer IDs from raw JSON API responses into a structured table

#### 22.3 Data Cleaning
- **Quality checks** on raw and extracted data
- Apply the cleaning steps (parsing, correcting, standardizing, matching, consolidating)

#### 22.4 Data Discovery
- **Generate and enrich data catalogs** — "what datasets exist in this lake?"
- **Query-driven discovery** — find datasets relevant to your query
- **Graph-structured linkages** between datasets (dataset A's customer IDs appear in dataset B)
- Example: A data scientist searching for "purchase data from Q1 2024" uses the catalog to find the right dataset

#### 22.5 Metadata Management

This is **critical** — without good metadata, a data lake becomes a **data swamp**:

```
Good Metadata:  "customers_Q1_2024.parquet" → schema, source, owner, creation date
Missing Metadata: "file_2024_01_15_export.parquet" → what is this? who owns it? is it current?
```

- **Extract, manage, and index** metadata from all datasets
- **Link** to existing knowledge bases (e.g., "this column contains ISO country codes")
- When metadata degrades: **Lake → Swamp** (data is there but nobody can find or trust it)

#### 22.6 Data Integration
- **Construct a mediated (unified) schema** from different sources — different datasets may have different schemas for the same concept
- **On-demand integration at query time** — unlike ETL which pre-integrates everything, data lakes integrate lazily
- **Sample-driven schema mapping** — automatically infer how schemas from different datasets map to each other by sampling data

#### 22.7 Data Versioning
- When a new dataset arrives: is it a new dataset or a **new version of an existing one**?
- Track **provenance** — where did this data come from? What transformations were applied?
- Handle **schema evolution** — when a source changes its schema, data lake must adapt

---

## 23. Data Lakes vs. Data Warehouses

| Aspect | Data Warehouses | Data Lakes |
|--------|----------------|-----------|
| **Data form** | Cleaned, transformed via ETL | Raw, retained as-is from sources with limited metadata |
| **Schema** | Pre-defined schema, ready for querying ("schema on write") | On-demand schema for each analysis ("schema on read") — uses ELT (Extract-Load-Transform) |
| **Data types** | Structured only (relational tables) | Structured + semi-structured + unstructured (JSON, images, logs) |
| **Use case** | Operational needs, well-defined and pre-defined questions | Exploratory analysis, predictive modeling, ML. Not suitable for operations |
| **Cost** | Expensive ETL, data/schema maintenance. Faster to define structured queries | Cheaper maintenance, but requires users to understand raw data and define ELT |
| **New data sources** | Harder to integrate (need to fit new ETL pipeline) | Easier to add (just ingest) |
| **Scalability** | Limited (often RDBMS-based) | Scales well to large data (distributed big data platform) |

> **Key difference in order:** Warehouse = ETL (transform before loading). Lake = ELT (load first, transform on query).

---

## 24. Data Lakes and Databases

Data lakes use **big data platforms**, not relational databases:

```
┌──────────────────────────────────────────────────┐
│              Data Lake Architecture               │
│                                                    │
│  Storage Layer (distributed)                       │
│  ┌─────────────────────────────────────────────┐  │
│  │  HDFS  │  Cassandra  │  S3/GCS/Azure Blob  │  │
│  └─────────────────────────────────────────────┘  │
│                   ↕ (decoupled)                    │
│  Data Format Layer                                 │
│  ┌──────────────────────────────────────────────┐ │
│  │  Parquet (structured)  │  JSON (semi-struct.) │ │
│  │  Text/Image (unstructured)                    │ │
│  └──────────────────────────────────────────────┘ │
│                   ↕ (decoupled)                    │
│  Compute Layer                                     │
│  ┌──────────────────────────────────────────────┐ │
│  │  Apache Spark ecosystem                       │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**Key principle: Decoupled storage from compute**
- Storage can be scaled independently from compute
- Can swap storage (HDFS → cloud) without changing compute logic
- Can run multiple compute engines against the same storage

---

## 25. Data Lakes Using Apache Spark

Spark is the de-facto compute engine for data lakes:

| Capability | How Spark Provides It |
|------------|----------------------|
| **ETL/ELT workloads** | RDDs (low-level), DataFrames (SQL-like), SparkSQL (SQL queries on raw files) |
| **Machine learning** | MLlib — train models directly on data lake data |
| **Diverse file formats** | Reads/writes JSON, CSV, Parquet, linked data, Avro, etc. |
| **Diverse file systems** | Works with HDFS, Amazon S3, Azure Blob, Google GCS |

**Example Spark workflow on a data lake:**

```python
# 1. Read raw JSON data from HDFS (ELT — load first)
raw_df = spark.read.json("hdfs://datalake/raw/purchases/2024/")

# 2. Transform (schema on read)
clean_df = raw_df.select("customer_id", "product", "amount") \
                 .filter(raw_df.amount > 0) \
                 .withColumn("date", to_date("purchase_timestamp"))

# 3. Write as Parquet for faster future queries
clean_df.write.parquet("hdfs://datalake/processed/purchases_2024/")

# 4. Run ML (MLlib)
from pyspark.ml.classification import RandomForestClassifier
model = RandomForestClassifier(featuresCol="features", labelCol="churned")
```

---

## 26. Limitations of Data Lakes

### No ACID

Data lakes do **not provide ACID** guarantees:
- **No Atomicity:** A write of multiple files can fail halfway — some files written, others not
- **No Isolation:** Two concurrent jobs reading/writing the same data can interfere
- **No Consistency:** No schema enforcement; any consumer can write anything

### Instead: BASE

Data lakes provide **BASE semantics**:
- **B**asically **A**vailable: Always available (distributed, no single point of failure)
- **S**oft state: Data may be transitionally inconsistent
- **E**ventually consistent: Over time, data converges to a consistent state

> **Trade-off summary:** Data lakes trade ACID for scalability and flexibility. If you need strong transactional guarantees, use a database. If you need petabyte-scale heterogeneous analytics, use a data lake.

---

## 27. Quick Comparison: HBase vs. Dynamo vs. Data Lake

| Feature | Dynamo | HBase | Data Lake |
|---------|--------|-------|-----------|
| **Data model** | Key-value (blob) | Wide-column (row+column families) | Any format (raw) |
| **Access** | Primary key only | Row key + column family | Query-driven (Spark/SQL) |
| **Consistency** | Eventually consistent (AP) | Strong at row-level (CP) | BASE (AP) |
| **ACID** | No | Row-level only | No |
| **Partitioning** | Consistent hashing (circular) | Row-key range partitioning | Storage-system dependent |
| **Storage** | Custom internal | HDFS | HDFS / Cloud / Cassandra |
| **Best for** | Always-writable key-value (shopping cart) | Structured wide-column analytics (user activity logs) | Exploratory analytics, ML on diverse data |

---
