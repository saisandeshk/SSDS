# DA256 Midterm Solutions — Module 1 & 2

## Comprehensive Solutions with Deep Explanations & Key Concepts

---

## Question 1
**Which of these computing devices typically have multi-core processing units and require parallel programming for effective use?**

1. Smart phones
2. GPU workstations
3. CPU-based servers
4. Cloud Virtual Machines

### ✅ Answer: 1, 2, 3, 4 — **ALL of them**

### Deep Explanation:

In the modern era, **all** these computing devices have multi-core processors and benefit from parallel programming:

1. **Smartphones** — Modern smartphones use ARM big.LITTLE or similar heterogeneous multi-core architectures (e.g., Apple M3 chip has performance and efficiency cores). Apps like video rendering, ML inference, and gaming leverage multiple cores. The lecture showed Apple M3 and ARM big.LITTLE as examples of heterogeneous SoC (System on Chip) designs.

2. **GPU workstations** — GPUs are the epitome of parallel processing with thousands of small cores (e.g., NVIDIA Blackwell). They require explicit parallel programming (CUDA, OpenCL) to harness their power. The lecture mentioned GPU workstations used for AI training.

3. **CPU-based servers** — Modern servers use multi-core CPUs like AMD EPYC Zen with 64+ cores. As Moore's Law shifted from clock speed scaling (Dennard scaling ended ~2005) to core count scaling, servers now routinely have 16-128 cores requiring parallel programming. The lecture showed "16 cores × 2GHz CPU = 32 × 10^9 instructions per second."

4. **Cloud Virtual Machines** — Cloud VMs are provisioned from these same multi-core servers. AWS, Azure, GCP offer VMs with many vCPUs that map to physical cores. The lecture discussed cloud computing as "1000s of racks, 100k servers" with virtualized infrastructure.

### 🔑 Key Concepts to Remember:
- **Moore's Law** continues for transistor count (~2x in 3 years) but **Dennard Scaling** (performance/watt doubling) ended, so clock speeds plateaued
- The shift is from **single-core speedup → multi-core parallelism**
- Even smartphones now have heterogeneous multi-core processors
- **Scale-up** (more cores per machine) vs **Scale-out** (more machines) — both need parallel programming

---

## Question 2
**What Spark operations do Google MapReduce's map() and reduce() functions resemble?**

1. Google MR's map() is like Spark's map(). Google MR's reduce() is like Spark's reduce().
2. Google MR's map() is like Spark's mapPartitions(). Google MR's reduce() is like Spark's reduce().
3. Google MR's map() is like Spark's flatMap(). Google MR's reduce() is like Spark's cogroup().
4. Google MR's map() is like Spark's flatMap(). Google MR's reduce() is like Spark's groupByKey() for the shuffle followed by flatMap().

### ✅ Answer: **4**

### Deep Explanation:

This requires understanding the exact signatures of Google MapReduce vs Spark operations:

**Google MapReduce:**
```
map:    (K1, V1)       → List<K2, V2>       # Emits ZERO or MORE key-value pairs
reduce: (K2, List<V2>) → List<K3, V3>       # Gets ALL values grouped by key
```

**Why NOT Option 1 (Spark's map)?**
- Spark's `map()` produces **exactly one output per input** item: `T → U`
- Google MR's `map()` can emit **zero or more** outputs per input (it uses `emit()` calls in a loop)
- Therefore Google MR's map ≠ Spark's map

**Why NOT Option 2 (Spark's mapPartitions)?**
- `mapPartitions()` processes an **entire partition** at once: `Iterator[T] → Iterator[U]`
- Google MR's map processes **one key-value pair at a time** (called once per record), not a whole partition

**Why NOT Option 3 (Spark's cogroup)?**
- `cogroup()` groups values from **multiple RDDs** by key: `RDD[(K,V1)].cogroup(RDD[(K,V2)])` → `RDD[(K, (Iterable[V1], Iterable[V2]))]`
- Google MR's reduce works on values from a **single** grouped stream, not multiple RDDs

**Why Option 4 is CORRECT:**
- **Google MR's map() ≈ Spark's flatMap()**: Both take one input and produce **zero or more** outputs. flatMap: `T → List<U>`, which is exactly `(K1,V1) → List<K2,V2>` in MR.
- **Google MR's reduce() ≈ groupByKey() + flatMap()**: 
  - Google MR's reduce receives `(K2, List<V2>)` — this is exactly what `groupByKey()` produces: `RDD[(K, Iterable[V])]`
  - The shuffle/sort in MapReduce corresponds to the **shuffle** in `groupByKey()`
  - Then the reduce function itself applies logic to each group, producing zero or more outputs — this is the `flatMap()` step

### 🔑 Key Concepts to Remember:
- **map()**: 1 input → exactly 1 output
- **flatMap()**: 1 input → 0 or more outputs (flattened)
- **mapPartitions()**: operates on entire partition iterator
- Google MR's map uses `emit()` to produce multiple outputs → like flatMap
- Google MR's reduce gets `(key, [values])` → like groupByKey's output

---

## Question 3
**Which of these assumptions did Google make in their SOSP paper on the design of GFS?**

1. The system stores a small number (1000s) of large files (>1GB)
2. The system runs on commodity machines with lower reliability of components
3. The read workload is limited to large streaming reads.
4. The system should efficiently support multiple concurrent client appends to same file

### ✅ Answer: **1, 2, 4**

### Deep Explanation:

From the GFS paper (SOSP 2003), the key design assumptions were:

1. ✅ **Small number of large files** — The paper states: *"few million files, each typically 100 MB or larger in size"* and *"Multi-GB files common case. Small files need not be optimized."* Google's workload was web crawls, inverted indices, and page rank data — all large files.

2. ✅ **Commodity machines with lower reliability** — The paper explicitly states: *"Inexpensive commodity components that often fail"* and *"Detect, tolerate, and recover promptly from failures on a routine basis."* This is a fundamental GFS design principle — failures are the NORM, not the exception.

3. ❌ **Read workload is NOT limited to large streaming reads** — The paper says: *"Large streaming reads: Each op reads >1 MB; Same client will read contiguous region of a file"* AND *"Small random reads: Reads a few KBs at some arbitrary offset. Apps often batch and sort their small reads to do sequential access."* So GFS supports BOTH large streaming reads AND small random reads (though optimized for large streaming reads). The question says "limited to" large streaming reads, which is FALSE.

4. ✅ **Multiple concurrent client appends** — The paper explicitly supports this: *"Append to the same file by hundreds of (producer) clients"* and *"Atomicity with minimal synchronization overhead."* GFS introduced the **Record Append** operation specifically for this use case (producer-consumer queues). The design ensures *"well-defined semantics for multiple concurrent clients."*

### 🔑 Key Concepts to Remember:
- GFS optimizes for **large files** (not small files), **sequential access** (not random), and **appends** (not overwrites)
- GFS supports BOTH large streaming reads AND small random reads
- **Record Append** is a key GFS innovation for concurrent writers
- **Commodity hardware** with frequent failures is a fundamental assumption
- *"High sustained bandwidth is more important than low latency"* — batch over interactive

---

## Question 4
**Which of these are valid reasons to use a distributed file system as opposed to a single-machine file system?**

1. Higher cumulative access bandwidth
2. Fault tolerance
3. Better data consistency
4. Weak scaling

### ✅ Answer: **1, 2, 4**

### Deep Explanation:

1. ✅ **Higher cumulative access bandwidth** — With multiple Data Nodes, the aggregate disk I/O bandwidth increases. If one HDD provides 100-150 MB/s, 100 machines provide 10-15 GB/s cumulative bandwidth. The lecture noted: *"We use many disks in parallel (HDFS/GFS approach)"* and this is a key reason for DFS under "Performance: Bandwidth, Wide-area locality."

2. ✅ **Fault tolerance** — DFS replicates data across multiple machines (default 3 replicas in GFS/HDFS). If one machine fails, data is still available from other replicas. GFS is designed to *"detect, tolerate, and recover promptly from failures."* Block replicas are spread across machines AND racks for maximum reliability.

3. ❌ **Better data consistency** — DFS actually has **WORSE** consistency compared to single-machine file systems. Single-machine FS has strong consistency (one copy of data, one writer at a time). DFS must manage consistency across replicas, leading to relaxed consistency models. GFS has a complex consistency model with regions being "consistent," "defined," or "inconsistent." The storage systems comparison table shows: single-machine FS has "Strong (single machine)" consistency while DFS has "Varies (often relaxed)."

4. ✅ **Weak scaling** — Weak scaling (Gustafson's Law) means increasing problem size proportionally with the number of processors. DFS enables this: as data grows, you add more machines, each storing and processing its share. Big data platforms are designed for weak scaling — *"Problem size increases with number of processors."* You can add more Data Nodes to store more data and read it faster.

### 🔑 Key Concepts to Remember:
- DFS sacrifices consistency for scalability, fault tolerance, and performance
- **Strong scaling** = fixed problem size, add more processors → limited by Amdahl's Law
- **Weak scaling** = problem size grows with processors → Gustafson's Law
- Big data platforms are designed for **weak scaling**, not strong scaling
- DFS advantages: bandwidth, fault tolerance, capacity, remote access, weak scaling

---

## Question 5
**Which of these statements about Spark DataFrames and Spark SQL is/are TRUE?**

1. Spark DataFrames follow an imperative programming model while Spark SQL is a declarative programming model.
2. Unlike Spark RDDs, all transformation and action operations in Spark DataFrames are lazily evaluated.
3. Spark DataFrames are flexible since they allow us to update the contents of the table in-place.
4. The Catalyst optimizer uses rule-based transformations of the Abstract Syntax Tree combined with a cost based optimizer to select the nominally best physical plan.

### ✅ Answer: **4**

### Deep Explanation:

1. ❌ **DataFrames are NOT imperative** — Both DataFrames AND Spark SQL follow a **declarative** programming model. The notes explicitly state: *"DataFrames embody the declarative programming paradigm"* and contrast this with RDDs which are imperative ("Tell HOW to do it"). DataFrames say "WHAT to do" while Spark's Catalyst optimizer decides HOW. Both DataFrame DSL and SQL are declarative — users specify intent, not execution steps.

2. ❌ **RDDs are ALSO lazily evaluated** — This statement implies RDDs are not lazily evaluated, which is wrong. Both RDDs AND DataFrames use lazy evaluation. In RDDs, transformations (map, filter, etc.) are lazy — only actions trigger execution. DataFrames also use lazy evaluation: *"Transformations (select, filter, join, groupBy) build up a logical plan but do not execute anything."* The "Unlike Spark RDDs" part makes this statement false.

3. ❌ **DataFrames do NOT allow in-place updates** — DataFrames are **immutable** (backed by immutable RDDs). The notes state: *"DataFrames themselves are immutable (backed by immutable RDDs)"* and *"ACID properties do not apply since DataFrames are immutable — there are no in-place updates."* Operations like `withColumn()` create NEW DataFrames; they don't modify existing ones.

4. ✅ **Catalyst uses rule-based + cost-based optimization** — The Catalyst optimizer pipeline has four phases:
   - **Analysis** (rule-based): Resolve references using catalog
   - **Logical Optimization** (rule-based): Constant folding, predicate pushdown, projection pruning
   - **Physical Planning** (rule-based + **cost-based**): Generate multiple physical plans, use cost model to select the cheapest
   - **Code Generation**: Compile to Java bytecode
   
   The notes confirm: Physical Planning uses cost model to *"Select CHEAPEST plan"* by estimating table sizes, selectivity, etc. For join selection, it chooses broadcast join for small tables vs sort-merge join for large ones.

### 🔑 Key Concepts to Remember:
- Both **DataFrames** and **SQL** are **declarative** (not imperative)
- Both **RDDs** and **DataFrames** use **lazy evaluation**
- DataFrames are **immutable** — no in-place updates
- Catalyst has **4 phases**: Analysis → Logical Optimization → Physical Planning → Code Generation
- Physical planning uses **both** rule-based and cost-based optimization
- The AST (Abstract Syntax Tree) is the core data structure — everything is represented as **trees of typed nodes**

---

## Question 6
**What are the benefits of large block sizes in HDFS/GFS?**

1. Reduces the amount of metadata maintained in the Name Node for each file.
2. Improves the cumulative read performance through parallel block reads.
3. Improves the read performance by amortizing the overheads for establishing the TCP connection from the client to data node.
4. Reduces the probability of record append conflicts from concurrent clients.

### ✅ Answer: **1, 3**

### Deep Explanation:

1. ✅ **Reduces metadata size** — The lecture explicitly states: *"Reduce metadata size on Name Node, O(# blocks)."* With a 64MB block size instead of 4KB, there are 16,000x fewer blocks for the same file. Since the Name Node maintains in-memory metadata for every block (~64 bytes per block entry), larger blocks dramatically reduce memory consumption. A 1TB file with 64MB blocks = ~16,000 blocks; with 4KB blocks = ~268 million blocks.

2. ❌ **Parallel block reads is NOT specifically a benefit of LARGE block sizes** — Parallel block reads depend on having multiple blocks distributed across multiple Data Nodes, which happens regardless of block size. In fact, with larger blocks, a small file might fit in a single block on one Data Node, meaning LESS parallelism. Large block sizes don't inherently improve cumulative read performance through parallel reads — that's a property of distributing blocks across nodes.

3. ✅ **Amortizes TCP connection overhead** — The lecture states large blocks enable *"Single persistent TCP connection to Data Node"* with *"Smaller overheads."* It also notes large blocks *"Avoid frequent client communications with Name Node"* and *"Allows large metadata caches at client."* When reading a 64MB block, the TCP setup overhead (latency of connection establishment) is amortized over a large data transfer. With 4KB blocks, you'd need to establish connections for 16,000x more blocks to read the same data.

4. ❌ **Does NOT reduce record append conflicts** — Record append conflicts depend on the number of concurrent writers to the SAME block, not the block size. In fact, larger blocks might mean more concurrent appends to the same block (since the block has more space), potentially increasing conflicts. Record append conflict handling is managed by the primary replica's serialization, not block size.

### 🔑 Key Concepts to Remember:
- GFS uses **64MB** blocks (HDFS v3 uses **128MB**)
- Large blocks reduce **metadata** overhead (fewer blocks to track)
- Large blocks amortize **TCP connection setup** and **Name Node communication**
- Name Node keeps all metadata **in memory** — ~64 bytes per block
- Large blocks work well for **large sequential reads** (GFS's target workload)
- Only bytes actually used are stored on disk (no over-allocation for partially-filled blocks)

---

## Question 7
**Which of these statements on strong and weak scaling are FALSE?**

1. Amdahl's law says that perfect strong scaling is theoretically impossible.
2. Gustafson's law says that perfect strong scaling is theoretically impossible.
3. If an application has 10% of sequential code and 90% of parallel code, it is not meaningful to allocate 100 processors to it to achieve strong scaling.
4. Big data platforms are usually designed to achieve strong scaling.

### ✅ Answer (FALSE statements): **2, 4**

### Deep Explanation:

1. **TRUE (not the answer)** — Amdahl's Law governs strong scaling. For strong scaling, problem size is FIXED and you add more processors. Speedup = 1 / (s + (1-s)/p), where s is the serial fraction. As p→∞, Speedup → 1/s. If there is ANY serial fraction (s > 0), perfect speedup (Speedup = p) is impossible. Even 1% serial code limits max speedup to 100x. So Amdahl's Law does say perfect strong scaling is theoretically impossible. This statement is TRUE.

2. ❌ **FALSE** — Gustafson's Law governs **WEAK scaling**, NOT strong scaling. Gustafson's Law says: Scaled Speedup = s + p·(1-s), where the problem size grows with the number of processors. Gustafson's Law says nothing about strong scaling being impossible — it provides an ALTERNATIVE view where weak scaling CAN achieve near-linear speedup. The statement incorrectly attributes a strong scaling claim to Gustafson's Law, which is about weak scaling.

3. **TRUE (not the answer)** — With s=0.1 (10% sequential) and p=100, Amdahl's Law gives: Speedup = 1/(0.1 + 0.9/100) = 1/(0.1 + 0.009) = 1/0.109 ≈ 9.17x. So 100 processors give only ~9.2x speedup. The maximum possible speedup is 1/0.1 = 10x. Using 100 processors for a 9.2x speedup (vs. max 10x) is wasteful. The lecture noted: *"Meaningful only for small values of p, small fractions of s."* This statement is TRUE.

4. ❌ **FALSE** — Big data platforms are designed for **WEAK scaling**, not strong scaling. Gustafson's Law guides their design: as you add more machines, you process MORE data (larger problem). The lecture states this clearly. In HDFS/GFS, adding more Data Nodes means storing and processing more data. In Spark, adding more executors means processing larger datasets. You don't add 100 machines to process the same 1GB file faster — you add them to process 100GB.

### 🔑 Key Concepts to Remember:
- **Amdahl's Law (Strong Scaling)**: Speedup = 1/(s + (1-s)/p), fixed problem size, limited by serial fraction
- **Gustafson's Law (Weak Scaling)**: Scaled Speedup = s + p·(1-s), problem grows with processors
- Max speedup in strong scaling = **1/s** (e.g., 10% sequential → max 10x speedup)
- Big data platforms use **weak scaling** — more data, more machines
- Strong scaling hits diminishing returns quickly; weak scaling can scale near-linearly

---

## Question 8
**What are the downsides of replication of blocks in HDFS/GFS?**

1. More numbers of block replicas need to be accessed when reading a file.
2. More numbers of block replicas need to be written to when writing to a file.
3. More space is disk occupied for a file due to block replication.
4. There is a higher possibility of permanently losing the contents of a file since there are more replicas on more machines that can fail.

### ✅ Answer: **2, 3**

### Deep Explanation:

1. ❌ **NOT a downside** — When READING, you only need to read from ONE replica (the closest/fastest one). The Name Node returns replica locations *"in the order of its closeness to the reader."* Replication actually HELPS reads by providing more choices and reducing read latency. You don't need to read all replicas — any one copy is sufficient.

2. ✅ **TRUE downside** — When WRITING, data must be written to ALL replicas. In GFS, the write data flow involves: client sends data to all replicas (pipelined), primary writes and forwards to secondaries, and the write is only acknowledged after ALL replicas confirm. With 3x replication, every write is 3x the work. The lecture describes the pipeline: "Client sends block data to all replicas" → "Secondary writes mutation in serial # order" → "Primary acks to client after acks from all secondaries."

3. ✅ **TRUE downside** — With default 3x replication, every file consumes 3x disk space. A 1TB file uses 3TB of disk. This is a significant cost. The lecture notes replication as a key design choice: *"Each block has n=3 replicas by default."*

4. ❌ **NOT true** — More replicas mean LESS chance of permanent data loss, not more. If you have 3 replicas across different machines and racks, all 3 must fail simultaneously to lose data. The probability of losing data decreases exponentially with the number of replicas. The lecture states: *"Spread replicas across machines and across racks — Survives even if a rack goes offline, e.g. switch fails."* Replication IMPROVES reliability.

### 🔑 Key Concepts to Remember:
- Replication downsides: **more write cost** (all replicas written) and **more disk space** (n copies)
- Replication upsides: **fault tolerance** (data survives failures), **read performance** (choose closest replica)
- Default replication factor: **3** in both GFS and HDFS
- Block placement strategy: replicas on different machines AND different racks
- Write pipeline in GFS is pipelined to maximize bandwidth utilization

---

## Question 9
**Which is the correct order of steps in which the write data flow happens in GFS?**

(A) Primary asks secondaries to write the received mutation to disk
(B) Primary has a lease on the block
(C) Client sends the write commit request to the Primary
(D) Client sends mutation contents to secondary and primary in a pipelined manner

### ✅ Answer: **3 (B, D, C, A)**

### Deep Explanation:

The GFS write data flow, step by step:

**Step 1 — B: Primary has a lease on the block**
The Master grants a chunk lease to one of the replicas, making it the "primary." This establishes who controls the mutation order. The lease must exist BEFORE any write can happen. *"Master grants chunk lease to one of the replicas: primary. Primary picks a serial order for all mutations to a chunk."*

**Step 2 — D: Client sends mutation contents to secondary and primary in a pipelined manner**
The client sends the actual data to ALL replicas (both primary and secondaries). This is the DATA flow — decoupled from the control flow. *"Client sends block data to all replicas, in any order. Worker stores in LRU buffer."* Data flows in a pipelined fashion using maximum bandwidth paths.

**Step 3 — C: Client sends the write commit request to the Primary**
Once ALL replicas have received the data (acknowledged receipt), the client sends a WRITE REQUEST (commit) to the primary. *"When all workers ack, client sends write request to primary."* This is the CONTROL flow.

**Step 4 — A: Primary asks secondaries to write the received mutation to disk**
The primary assigns a serial number to the mutation, applies it locally, then forwards the write request to all secondaries. *"Primary assigns serial # to mutation. Applies mutation locally. Forwards write request to all secondaries."* Secondaries write in serial number order and ack back to primary.

### Why this order is critical:
- **B before D**: Must have a designated primary before sending data
- **D before C**: Data must be at all replicas before commit request (separation of data flow from control flow)
- **C before A**: Primary must receive commit request before instructing secondaries
- **Key design insight**: GFS **decouples data flow from control flow** — data flows using maximum bandwidth paths, while control flow goes through the primary for serialization

### 🔑 Key Concepts to Remember:
- GFS write flow: **Lease → Data Pipeline → Commit → Apply**
- **Data flow** (client → all replicas) is **decoupled** from **control flow** (client → primary → secondaries)
- Data is pipelined to maximize bandwidth; control flow ensures serial order
- Primary assigns **serial numbers** to mutations for consistency
- If primary fails before writing: no writes done, client retries
- If secondary fails after primary writes: region left **inconsistent**

---

## Question 10
**Which of these statements about the consistency model of GFS are true?**

1. If a region in the block is defined, it is also consistent.
2. The use of record append has severe performance overheads and should be used rarely.
3. A block will always remain in a consistent state as long as only a single client mutates it at a time.
4. If any mutation operation fails, the region it was modifying can be left inconsistent.

### ✅ Answer: **1, 4**

### Deep Explanation:

1. ✅ **TRUE** — By definition in the GFS consistency model:
   - **Consistent**: All replicas have the same byte contents for that region
   - **Defined**: It is consistent AND reflects the complete update from a single write client
   - Therefore, **defined ⊂ consistent** (defined implies consistent). If a region is defined, it must also be consistent. The lecture states: *"Defined: It is consistent, and the region reflects the complete update performed by a single write client."*

2. ❌ **FALSE** — Record append is a key feature of GFS designed to be efficient for the common case of producer-consumer workloads. It was specifically designed for *"hundreds of (producer) clients"* appending concurrently. It provides *"Atomicity with minimal synchronization overhead."* GFS applications are encouraged to *"rely on appends rather than overwrites."* Record append doesn't have "severe performance overheads" — it's a core, optimized operation.

3. ❌ **FALSE** — Even with a single client, a mutation can FAIL. If the mutation fails (e.g., primary writes but a secondary fails), the region can become **inconsistent**. The lecture states: *"If only one client is mutating a block, and it succeeds, the affected region is defined (and consistent)."* The key word is **"and it succeeds"** — if it fails, even single-client mutations can leave the region inconsistent. The statement says "always remain consistent" which ignores the failure case.

4. ✅ **TRUE** — The lecture explicitly states: *"Failed mutations → Inconsistent region"* and *"Different clients may see different data at different times."* When a mutation fails (e.g., primary wrote but a secondary didn't, or the write was partial), different replicas may have different data for that region, making it inconsistent. There is **no rollback** in GFS: *"No Rollback."*

### 🔑 Key Concepts to Remember:
- **Defined ⊂ Consistent**: Defined implies consistent, but not vice versa
- Single client success → **Defined** (and consistent)
- Multiple concurrent clients, all succeed → **Consistent but undefined** ("mingled fragments")
- Any failure → **Inconsistent** (no rollback!)
- GFS has **NO rollback** mechanism
- Record append guarantees **at-least-once** semantics at a GFS-chosen offset
- Applications handle duplicates using checksums and UUIDs

---

## Question 11
**Which of these formulations from the JPDC article indicate the total execution time for a workload with a finite number of processors and no communication overheads?**

1. $T_N(W) = \sum \frac{W_i}{i \cdot \Delta} \cdot \lceil \frac{i}{N} \rceil + Q_N W$
2. $T_N(W) = \sum \frac{W_i}{i \cdot \Delta} \cdot \lceil \frac{i}{N} \rceil$
3. $T_N(W) = \sum \frac{W_i}{i \cdot \Delta}$
4. $T_N(W) = \sum \frac{W_i}{\Delta}$

### ✅ Answer: **2**

### Deep Explanation:

This question is about the JPDC 1993 paper (Jun and Ni) on scalability and parallel speedup. Let's build up the understanding:

**Key variables:**
- W = total work for application
- W_i = amount of work with degree of parallelism i
- m = maximum degree of parallelism
- Δ = computing capacity of one processor
- N = number of available processors
- T_N(W) = time to execute W on N processors

**Option 4: T₁(W) = Σ W_i/Δ** — This is the time on a **SINGLE processor** (all work done sequentially). Each unit of work W_i takes W_i/Δ time on one processor regardless of its parallelism degree. This is the baseline.

**Option 3: T_∞(W) = Σ W_i/(i·Δ)** — This is the time with **INFINITE processors** and no communication overhead. Work W_i with parallelism degree i can use i processors, giving speedup of i. But you can't use more than i processors for work with degree-of-parallelism i.

**Option 2: T_N(W) = Σ W_i/(i·Δ) · ⌈i/N⌉** — This is the time with **FINITE N processors** and **no communication overhead**. The ceiling factor ⌈i/N⌉ accounts for the fact that when the degree of parallelism i exceeds N (available processors), some work must be serialized. For example, if i=7 tasks need to run but only N=4 processors are available, ⌈7/4⌉ = 2 rounds are needed. This is the correct answer.

**Option 1: T_N(W) = Σ W_i/(i·Δ) · ⌈i/N⌉ + Q_N·W** — This includes **communication overhead** Q_N·W. The question asks for NO communication overhead, so this is wrong.

### 🔑 Key Concepts to Remember:
- **Single processor**: T₁(W) = Σ W_i/Δ (all work sequential)
- **Infinite processors, no comm**: T_∞(W) = Σ W_i/(i·Δ) (max parallelism exploited)
- **Finite N processors, no comm**: T_N(W) = Σ W_i/(i·Δ) · ⌈i/N⌉ (limited by N)
- **Finite N processors, with comm**: Add Q_N·W communication overhead
- ⌈i/N⌉ = ceiling function representing how many "rounds" are needed when i > N
- Speedup S_N(W) = T₁(W) / T_N(W)

---

## Question 12
**Which of these statements about "lazy evaluation" of RDDs is/are TRUE?**

1. Only wide transformations are lazily evaluated. Narrow transformations are triggered immediately.
2. Lazy evaluation due to an action will can cause multiple RDDs to be created/materialized.
3. Once an RDD has been materialized and persisted, lazy evaluation guarantees that it will never be recreated.
4. Lazy evaluation helps Spark achieve resiliency since RDDs are created on-demand.

### ✅ Answer: **2, 4**

### Deep Explanation:

1. ❌ **FALSE** — ALL transformations (both narrow AND wide) are lazily evaluated. There is no distinction between narrow and wide transformations in terms of lazy evaluation. `map()` (narrow) is just as lazy as `groupByKey()` (wide). Both only execute when an action is called. The notes confirm: lazy evaluation applies to ALL transformations; only **actions** trigger computation.

2. ✅ **TRUE** — When an action is called, Spark must execute the entire chain of transformations that lead to the final RDD. For example:
   ```python
   rdd1 = sc.textFile("data.txt")      # Not materialized yet
   rdd2 = rdd1.map(parse)               # Not materialized yet
   rdd3 = rdd2.filter(is_valid)          # Not materialized yet
   rdd4 = rdd3.reduceByKey(add)          # Not materialized yet
   result = rdd4.collect()               # ACTION! All RDDs materialize
   ```
   When `collect()` is called, rdd1, rdd2, rdd3, and rdd4 all get created/materialized (or at least the pipeline executes). Multiple RDDs are materialized as part of executing the DAG.

3. ❌ **FALSE** — Even if an RDD is persisted (cached), it can still be **evicted from memory** due to memory pressure (LRU eviction). If evicted, it WILL be recomputed using the lineage graph. Spark does NOT guarantee that a cached RDD will never be recreated. The lineage graph is preserved precisely for this reason — to allow recomputation if needed.

4. ✅ **TRUE** — Lazy evaluation means RDDs are only created when needed (on-demand). This connects directly to resiliency: since Spark tracks the lineage (the sequence of transformations), it can recreate any lost partition by replaying transformations from the parent RDDs. The notes state: *"Lazy evaluation helps Spark achieve resiliency since RDDs are created on-demand"* and *"Spark tracks how RDD was created (lineage) — Can recompute lost partitions."* The RDD paper (NSDI 2012) highlights this as a key advantage over DSM.

### 🔑 Key Concepts to Remember:
- **ALL transformations** are lazy (both narrow and wide)
- Only **actions** trigger computation (collect, count, save, etc.)
- Lazy evaluation enables **optimization** (Spark can see the full plan) and **resiliency** (lineage-based recomputation)
- `persist()`/`cache()` SUGGESTS keeping an RDD in memory, but does NOT guarantee it
- Evicted cached RDDs are recomputed from lineage
- Each action creates one **Job**; jobs are divided into **Stages** at shuffle boundaries

---

## Question 13
**In the RDD paper (NSDI 2012), what are the pros and cons of RDDs over Distributed Shared Memory (DSM)?**

1. RDDs offer coarse-grained write operations while DSM offer fine-grained writes.
2. RDDs use lineage graphs for reliability while DSMs use checkpoint/recovery.
3. RDDs are immutable and offer implicit consistency. Consistency of DSM depends on the application design.
4. RDDs use data locality for task placement. DSMs leave it to application runtimes.

### ✅ Answer: **1, 2, 3, 4 — ALL of them**

### Deep Explanation:

All four statements correctly describe differences between RDDs and DSM as discussed in the NSDI 2012 paper:

1. ✅ **Coarse-grained vs fine-grained writes** — RDDs are transformed through **coarse-grained** operations (map, filter, join applied to the entire dataset). You can't update a single element in an RDD — you apply transformations to all elements. DSM supports **fine-grained** writes (update individual memory locations). This coarse-grained nature is what enables efficient fault recovery via lineage.

2. ✅ **Lineage vs checkpoint/recovery** — RDDs achieve fault tolerance through **lineage graphs**: tracking the sequence of transformations that created each RDD. If a partition is lost, it can be recomputed from its lineage. DSM uses **checkpointing** (periodically saving state to stable storage) and **rollback recovery**. Lineage is more efficient because: (a) it doesn't require the overhead of checkpointing during normal execution, (b) only the lost partition needs recomputation (not the entire state).

3. ✅ **Immutability and implicit consistency** — RDDs are **immutable** (read-only after creation), so there are no concurrent write conflicts, no race conditions, and no need for locks. Consistency is **implicit**. DSM allows concurrent reads AND writes to shared memory, meaning the application must carefully manage consistency using locks, barriers, or consistency protocols. This is left to the application designer.

4. ✅ **Data locality for task placement** — Spark schedules tasks based on **data locality** — running computation on the node that has the data. The driver *"schedules tasks based on data locality"* and *"tracks cached data locations for optimal scheduling."* DSM systems generally leave task placement to the application runtime or OS, without built-in data locality optimization.

### 🔑 Key Concepts to Remember:

| Aspect | RDD | DSM |
|--------|-----|-----|
| **Read** | Coarse or fine-grained | Fine-grained |
| **Write** | Coarse-grained (transformations) | Fine-grained (any memory cell) |
| **Fault Recovery** | Lineage (recomputation) | Checkpoint/rollback |
| **Consistency** | Implicit (immutable) | Application-dependent |
| **Task Placement** | Data locality aware | Application runtime decides |
| **Straggler Mitigation** | Backup (speculative) tasks | Difficult |
| **Memory Use** | In-memory, spill to disk | All in memory |

---

## Question 14
**Which of these data structures are maintained in the Name Node of GFS/HDFS in a reliable/durable manner?**

1. The file system namespace (directories, files, permissions, etc.)
2. The list of block IDs for a given file.
3. The list of Data Nodes that contain a replica of a Block ID.
4. The list of clients that have acquired a lock on a file.

### ✅ Answer: **1, 2**

### Deep Explanation:

The Name Node maintains three main data structures **in memory**, but only some are **durably persisted**:

1. ✅ **File system namespace** — This is durably maintained using the **transaction log (operations log)** and **checkpoints**. The lecture shows the Name Node architecture with "Directory Namespace, File Attributes" and "Transaction Log." The namespace includes directory structure, file names, permissions, etc. This is persisted reliably because losing the namespace means losing all knowledge of what files exist.

2. ✅ **File to Block ID mapping** — This is also durably maintained (persisted in the transaction log/checkpoints). The architecture diagram shows "File → BlockID[]" as a key data structure in the Name Node. You need to know which blocks compose which files, and this mapping is critical for data retrieval.

3. ❌ **Block to Data Node mapping — NOT durably maintained** — This is the key insight! The lecture explicitly states: *"Block to Data Node mapping built on-demand at startup."* And: *"Can be reconstructed at any time. Fault tolerance, less consistency issues with Data Node flux. Data Node is the final arbiter of blocks it has."* When the Name Node starts, Data Nodes send **block reports** (block id, generation stamp, length) and the mapping is rebuilt. This is NOT written to the transaction log because:
   - It can be reconstructed from Data Node block reports
   - Data Nodes may fail, be added, or removed — the mapping changes frequently
   - Data Node is the "final arbiter" of which blocks it has (data may "spontaneously vanish" due to disk failure)
   - Storing this mapping durably would create consistency issues

4. ❌ **Client locks** — GFS/HDFS uses **leases** (not locks in the traditional sense), and these are **transient** (not durably stored). Leases have expiration times and are managed in memory. If the Name Node restarts, leases expire and can be re-granted. The namespace locking for concurrent operations is also transient.

### 🔑 Key Concepts to Remember:
- **Durably stored**: File namespace + File→Block mapping (via transaction log + checkpoints)
- **NOT durably stored**: Block→DataNode mapping (reconstructed from block reports at startup)
- Name Node keeps ALL metadata **in memory** for fast operations (~64 bytes per block)
- **Block reports**: DataNodes send block reports immediately after registration, then every hour
- **Heartbeats**: DataNodes send heartbeats every 3 seconds; 10 minutes without → considered dead
- **Checkpoint Node/Backup Node**: Periodically combines checkpoint + journal for faster recovery
- The transaction log is replicated on other machines for Master reliability

---

## Question 15
**What is/are the characteristic(s) of the logical plan and physical plan during Spark RDD execution?**

1. The physical plan translates the logical plan into a series of Tasks for execution on specific Executors.
2. The physical plan helps identify all dependent transformations that need to execute for a given action.
3. The physical plan creates one Job for every action.
4. The logical plan will partition the dataflow into stages separated by shuffle boundaries.

### ✅ Answer: **1, 3**

### Deep Explanation:

1. ✅ **TRUE** — The physical plan converts the logical DAG of RDDs into actual executable **Tasks** that run on specific **Executors**. The notes state: *"Physical Plan: Converts the dataflow into specific tasks for execution on Workers/Executors (how to compute)."* The physical plan creates stages, and each stage consists of tasks — one task per partition in the last RDD of the stage. Tasks are then scheduled on executors by the TaskScheduler.

2. ❌ **FALSE** — Identifying dependent transformations is the job of the **LOGICAL plan**, not the physical plan. The logical plan captures the DAG of RDD dependencies (what RDDs to compute and their relationships). The notes state: *"Logical Plan: Converts the application to a dataflow of dependencies (what RDDs to compute and their relationships)."* The physical plan concerns itself with HOW to execute, not WHAT dependencies exist.

3. ✅ **TRUE** — Each action triggers the creation of exactly one Job. The notes confirm: *"Job: Created for each action() in the driver program."* The physical plan translates each action into a job, which is then divided into stages and tasks. The DAG Scheduler's `runJob()` is called for each action.

4. ❌ **FALSE** — Partitioning the dataflow into stages separated by shuffle boundaries is done during the creation of the **PHYSICAL plan**, not the logical plan. The logical plan just captures the RDD dependency graph (narrow vs wide dependencies). The **DAG Scheduler** (part of physical planning) walks the logical plan and creates stages by splitting at shuffle (wide dependency) boundaries. The notes describe the Stage Creation Algorithm: *"Walk backwards through the dependency chain. Add RDDs with NarrowDependency to current stage. When hitting ShuffleDependency, start a new stage."*

### 🔑 Key Concepts to Remember:

| Concept | Logical Plan | Physical Plan |
|---------|-------------|---------------|
| **Focus** | WHAT to compute | HOW to compute |
| **Contains** | DAG of RDD dependencies | Stages, Tasks, scheduling |
| **Dependencies** | Narrow vs Wide identified | Stages split at shuffle boundaries |
| **Execution** | Not executable directly | Executable Tasks on Executors |

- **Logical Plan** → DAG of RDDs with dependencies (narrow/wide)
- **Physical Plan** → Jobs (one per action) → Stages (split at shuffles) → Tasks (one per partition)
- **Narrow dependencies** are pipelined within a single task (no intermediate storage)
- **Wide dependencies** create stage boundaries (require shuffle)
- Stage 0 = final stage (ResultStage with ResultTasks)
- Parent stages = ShuffleMapStages with ShuffleMapTasks

---

## 📝 Master Summary: Critical Concepts to Remember

### Module 1: Storage Systems
1. **GFS/HDFS Architecture**: Master/NameNode (metadata) + DataNodes (data), client talks to both
2. **Block Size**: 64MB GFS / 128MB HDFS — reduces metadata, amortizes TCP overhead
3. **Replication**: 3 copies default, across machines and racks
4. **Write Flow**: Lease → Pipeline Data → Commit to Primary → Apply to Secondaries
5. **Consistency**: Defined ⊂ Consistent; failures → inconsistent; NO rollback
6. **NameNode Durability**: Namespace + File→Block mapping persisted; Block→DataNode mapping is NOT (rebuilt from block reports)
7. **Amdahl's Law**: Strong scaling, Speedup = 1/(s + (1-s)/p), max = 1/s
8. **Gustafson's Law**: Weak scaling, Scaled Speedup = s + p(1-s)

### Module 2: Apache Spark
1. **RDD**: Immutable, Distributed, Lazy, Fault-tolerant via lineage
2. **Transformations**: Lazy (map, filter, flatMap, groupByKey, reduceByKey) — create new RDDs
3. **Actions**: Trigger execution (collect, count, reduce, save)
4. **Narrow vs Wide**: Narrow = no shuffle, pipeline; Wide = shuffle, stage boundary
5. **Logical Plan**: DAG of RDD dependencies
6. **Physical Plan**: Jobs → Stages → Tasks on Executors
7. **DataFrames**: Declarative, schema-aware, Catalyst-optimized
8. **Catalyst**: Analysis → Logical Opt → Physical Planning (rule + cost-based) → Code Gen
9. **Google MR map ≈ flatMap, MR reduce ≈ groupByKey + flatMap**
10. **RDD vs DSM**: Coarse-grained writes, lineage recovery, implicit consistency, data locality
