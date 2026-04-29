# 📝 DS256 SSDS — Essay Questions Bank

> Based on the professor's questioning style: emphasizes **design trade-offs**, **system comparisons**, **scalability analysis**, and **critical thinking** about distributed systems.

---

## Module 1: Big Data & Distributed Storage

### E1. GFS Design Rationale
**"Explain the key design assumptions behind GFS and how each assumption influenced specific architectural decisions. Critically evaluate which of these assumptions may no longer hold in modern cloud environments."**

**Key points to cover:**
- Assumptions: commodity hardware failures, large sequential reads, append-heavy writes, high bandwidth > low latency
- How each drove design: single master (simple but bottleneck), 64MB chunks (reduces metadata), relaxed consistency (enables concurrent appends), replication across racks
- Modern challenges: single-master bottleneck (HopsFS scales to 1M ops/sec), SSD-era random reads, cloud object stores vs file systems

---

### E2. GFS Consistency Model
**"Describe GFS's consistency model in detail. Explain the differences between defined, consistent, and inconsistent regions. How do applications cope with this relaxed consistency?"**

**Key points to cover:**
- Definition of each term with the consistency table (serial write → defined, concurrent → consistent-undefined, failure → inconsistent)
- Write flow: data flow vs control flow decoupling
- Record append: at-least-once semantics, padding on overflow
- Application coping strategies: checksums, self-validating records, checkpointing, UUID-based dedup

---

### E3. Speedup Models Comparison
**"Compare and contrast Amdahl's Law (fixed-size speedup), Gustafson's Law (fixed-time/weak scaling), and memory-bounded speedup from Sun & Ni (1993). When is each model most appropriate?"**

**Key points to cover:**
- Fixed-size (Amdahl): S = 1/(s + (1-s)/p), limited by serial fraction
- Fixed-time (Gustafson): S = s + (1-s)·p, assumes parallelizable work scales with processors
- Memory-bounded: between the two, considers memory capacity
- General formulation with parallelism profiles, communication cost Q_N
- Big Data platforms target weak scaling
- Shape of parallelism profile matters — long sequential valleys vs peak parallelism

---

### E4. GFS Write and Mutation Flow
**"Describe the complete write data flow in GFS, including the lease mechanism, primary selection, and the separation of data flow from control flow. What happens during concurrent writes? What happens on failure?"**

**Key points to cover:**
- 7-step write flow (client → master → primary → secondaries)
- Lease mechanism: master grants, primary assigns serial numbers
- Data flow: pipelined linearly for max bandwidth, separate from control flow
- Concurrent writes: serial order maintained → consistent but may be undefined
- Failure: primary fails → no writes done, retry; secondary fails → inconsistent region

---

## Module 2: Big Data Processing

### E5. MapReduce to Spark Evolution
**"Why was Apache Spark developed as an alternative to MapReduce? Discuss the specific limitations of MapReduce that Spark addresses, and explain the RDD abstraction in detail."**

**Key points to cover:**
- MR limitations: disk I/O between stages, no data reuse, poor iterative performance, complex multi-stage chaining
- RDD properties: immutable, distributed, fault-tolerant via lineage
- Lazy evaluation: transformations deferred until actions
- Lineage DAG: enables partition-level recomputation
- Narrow vs wide dependencies and their impact on stages/fault recovery
- In-memory computing: orders of magnitude faster for iterative/interactive workloads

---

### E6. Spark Execution Model
**"Explain how a Spark application executes, from the driver to the final output. Cover the concepts of jobs, stages, tasks, narrow and wide dependencies, and how fault tolerance is achieved."**

**Key points to cover:**
- Driver creates SparkContext, submits jobs on actions
- DAGScheduler cuts lineage at wide dependencies → stages
- TaskScheduler assigns tasks (one per output partition per stage) to executors
- Narrow deps: pipelined within a stage; wide deps: shuffle boundary
- Fault tolerance: recompute lost partitions from lineage; checkpointing for long chains
- Fine-grained task re-execution (only failed partition)

---

### E7. reduceByKey vs groupByKey
**"Explain why reduceByKey is generally preferred over groupByKey for aggregation operations on Pair RDDs. Illustrate with an example and discuss the implications for shuffle cost."**

**Key points to cover:**
- groupByKey: shuffles ALL values across network before aggregation → massive data transfer
- reduceByKey: applies map-side combiner first, then shuffles reduced values → much less data
- Example: word count — groupByKey sends every (word, 1) pair; reduceByKey sends (word, local_count)
- Requirements: reduce function must be associative and commutative
- combineByKey as the general form with createCombiner, mergeValue, mergeCombiners

---

### E8. DataFrames and Catalyst Optimizer
**"How do DataFrames and the Catalyst optimizer improve upon the raw RDD API? What specific optimizations does Catalyst perform?"**

**Key points to cover:**
- Schema information: enables compile-time type checking, optimized memory layout
- Catalyst: predicate pushdown, projection pruning, operation reordering, join strategy selection
- Code generation: JVM bytecode vs interpreter overhead
- Language independence: same plan for Scala/Python/Java
- Tungsten: off-heap memory, cache-friendly layouts
- PySpark benefit: avoids per-row Python function calls

---

## Module 3: ML at Scale

### E9. Parameter Server Architecture
**"Describe the Parameter Server architecture for distributed machine learning. Discuss the three consistency models (Sequential, Eventual, Bounded Delay) and their trade-offs."**

**Key points to cover:**
- Server machines store model params as distributed shared memory (key-value)
- Workers compute gradients, push to servers, pull updated params
- Sequential (BSP): identical to single-thread, slowest but correct
- Eventual: no waiting, fast but may diverge
- Bounded delay: tunable — next task blocked until tasks from (t-Δ) complete
- PS bandwidth bottleneck: O(N) with workers → MPI All-Reduce alternative O(1) bandwidth, O(log N) steps

---

### E10. Three Forms of DNN Parallelism
**"Compare data parallelism, model parallelism, and pipeline parallelism for distributed DNN training. When is each approach appropriate, and how does 3D parallelism combine them?"**

**Key points to cover:**
- **Data parallelism**: full model per GPU, split data; limited by model size fitting in GPU
- **Model parallelism**: split model across GPUs; needed when model > GPU memory; low utilization (~25%)
- **Pipeline parallelism** (GPipe): micro-batches overlap; reduces bubbles to (p-1)/m; stores all intermediate activations
- 3D parallelism (DeepSpeed): addresses all three bottlenecks simultaneously
- Trade-offs: communication overhead, memory requirements, programming complexity
- Each addresses a different scaling dimension

---

### E11. Federated Learning System Design
**"Describe the system design for federated learning at scale as proposed by Google (Bonawitz et al., 2019). How does the system handle heterogeneous, unreliable mobile devices?"**

**Key points to cover:**
- Protocol: Selection → Configuration → Reporting
- Pace steering: server suggests reconnect times to handle diurnal patterns
- Client selection: subset from thousands that announce availability
- Local training on device with TF, FL plan pushed from server
- Secure aggregation for privacy
- Stragglers ignored; round discarded if too few report
- In-memory aggregation, decentralized selector actors
- Cross-silo vs cross-device differences (data size, availability, privacy concerns)

---

### E12. Synchronous vs Asynchronous Aggregation in FL
**"Compare synchronous, asynchronous, and tiered aggregation strategies in federated learning. What are the trade-offs of each?"**

**Key points to cover:**
- **Synchronous (FedAvg)**: waits for all clients; round-time = slowest client; simple convergence
- **Asynchronous (FedAsync)**: immediate updates; faster but stale updates → divergence; fast clients over-influence
- **Tiered (TiFL, HACCS)**: group clients by latency or data similarity; synchronous within tier
- **Hierarchical (HierFAVG)**: three levels (server, edge nodes, clients); geographic grouping
- Client selection: Oort (utility), REFL (availability), HACCS (data similarity)
- FedAvg is often "good enough" empirically

---

### E13. Orca: LLM Serving
**"Explain the key challenges in serving transformer-based generative models and how Orca addresses them with iteration-level scheduling and selective batching."**

**Key points to cover:**
- Problem with request-level scheduling: early finish waste, late admission delay, incompatible tensor shapes
- Iteration-level scheduling: schedule per iteration, immediate return, fast admission
- Selective batching: flatten non-attention layers (token-wise), separate attention per request
- Prefill vs decode phases: compute-bound vs memory-bound
- KV cache: avoids recomputation but grows with sequence length; memory pressure
- Performance metrics: TTFT (prefill latency) and TPOT (decode speed)
- Pipeline and intra-layer parallelism across GPUs

---

### E14. GNN Training at Scale
**"Describe the challenges in distributed GNN training and how DistDGL addresses them. Compare GNN data parallelism with DNN data parallelism."**

**Key points to cover:**
- Challenges: neighborhood explosion, graph irregularity, cross-partition dependencies
- GNN vs DNN data parallelism: GNN workers' data is NOT independent (graph structure creates dependencies)
- DistDGL: METIS partition (min edge cuts, balance training nodes) + mini-batch parallelism
- Halo vertices for boundary nodes; remote sampling and feature fetching
- Neighborhood sampling to control explosion
- Ring All-Reduce for gradient aggregation
- GNN models typically small → model parallelism less critical

---

## Module 4: NoSQL

### E15. CAP Theorem Analysis
**"Explain the CAP theorem and Brewer's later clarifications. How do real-world distributed systems navigate the CAP trade-offs?"**

**Key points to cover:**
- Original theorem: pick 2 of 3 (Consistency, Availability, Partition Tolerance)
- Brewer's 2012 clarifications: NOT a permanent binary choice; per-operation, per-data decisions
- Consistency and availability are continuous, not binary
- Timeouts force C vs A decisions; latency is the practical trigger
- Retrying indefinitely ≡ choosing C over A
- Real systems: Dynamo (AP), HBase (CP), Spanner (CP with TrueTime)
- ACID vs CAP: ACID C ≠ CAP C; ACID I is core of CAP
- BASE as practical CAP application

---

### E16. Gray's Replication Analysis
**"Explain Jim Gray's analytical model of replication dangers. What are the problems with eager and lazy replication, and what solution does Gray propose?"**

**Key points to cover:**
- Core issue: lock contention + abort/retry grows non-linearly with replicas
- Eager replication: guaranteed consistency but O(N²) or worse scaling
- Lazy replication: converts locking conflicts into reconciliation work; can cause "system delusion"
- Solution: sharding — transactions touch only one shard, no cross-shard 2PC needed
- Cross-shard transactions reintroduce the problem
- Two-tier replication: master replicas + mobile tentative updates
- Commutative/rejectable update constraints

---

### E17. Dynamo System Design
**"Describe Amazon Dynamo's design principles, key mechanisms, and how they achieve 'always writable' semantics. Include consistent hashing, vector clocks, sloppy quorums, and anti-entropy."**

**Key points to cover:**
- Design: AP in CAP, eventual consistency, decentralized (P2P), always writable
- Consistent hashing with virtual nodes: Q partitions, S physical nodes
- Preference list: N replicas, skip same physical node
- Sloppy quorum: W writes + R reads, R+W > N
- Vector clocks: (node, counter) pairs; detect causality, application resolves conflicts
- Hinted handoff: temporary storage during transient failures
- Merkle trees: anti-entropy for permanent failures
- Gossip protocol: membership propagation

---

### E18. NoSQL Database Comparison
**"Compare key-value stores (Dynamo), columnar stores (BigTable/HBase), and graph databases (Neo4j). When would you choose each?"**

**Key points to cover:**
- **Dynamo/Cassandra**: Simple get/put, high availability, eventual consistency, shopping cart-type workloads
- **BigTable/HBase**: Columnar with column families, strong row-level consistency, ordered scans, suitable for analytical workloads on structured data
- **Neo4j**: Property graph model, Cypher queries, optimized for relationship traversal, social networks, knowledge graphs
- Trade-offs: consistency vs availability vs query complexity
- Data model influences query capabilities and performance patterns

---

### E19. Data Lakes vs Data Warehouses
**"Compare data lakes and data warehouses. What are the advantages and limitations of each, and when would you use one over the other?"**

**Key points to cover:**
- Schema: warehouse = schema-on-write, lake = schema-on-read (ELT vs ETL)
- Data types: warehouse = structured only, lake = all types
- Purpose: warehouse = operational/predefined queries, lake = exploratory/ML
- Transactional guarantees: warehouse = ACID, lake = BASE
- Scalability: warehouse limited, lake scales well
- Cost: warehouse expensive ETL, lake cheaper storage but needs skilled users
- Data lake pitfall: can become "data swamp" without metadata management

---

## Module 5: Fast Data & Linked Data

### E20. Kafka Architecture and Design
**"Describe Apache Kafka's architecture, including topics, partitions, consumer groups, and replication. How does Kafka achieve high throughput and fault tolerance?"**

**Key points to cover:**
- Topics → Partitions (ordered, immutable logs)
- Producers: serialize, partition (key-based), batch for throughput
- Consumers: track offset per partition; consumer groups with partition assignment
- Brokers: leader-follower per partition; ZooKeeper for coordination
- High throughput: batching, zero-copy, sequential disk I/O, pub-sub model
- Fault tolerance: replication, in-sync replicas, automatic leader election
- Retention policies: time-based or size-based

---

### E21. Spark Structured Streaming
**"Explain Spark Structured Streaming's execution model. How does it differ from DStream, and what are the output modes and triggering options?"**

**Key points to cover:**
- Stream = unbounded DataFrame; unified batch + stream API
- DStream: RDD-based, no event-time windows, limited API
- Output modes: Append (new rows only), Update (modified rows), Complete (all rows)
- Triggering: Default micro-batch, Interval, Once, Continuous
- Stateful operations: groupBy with maintained state across micro-batches
- Event-time windows vs processing-time windows
- Checkpointing for exactly-once guarantees

---

### E22. Pregel Programming Model
**"Explain the Pregel (BSP-based) vertex-centric programming model. Describe how PageRank and Shortest Path algorithms are implemented using this model."**

**Key points to cover:**
- BSP: Compute → Communicate → Barrier → repeat
- Vertex-centric: logic from perspective of one vertex, executed on all
- Vertex state machine: Active → voteToHalt → Halted → reactivated by message
- API: getSuperstep(), getVertexValue(), sendMsg(), voteToHalt()
- Combiners: merge messages, commutative + associative
- **PageRank**: init 1/|V|, send PR/out_degree, sum incoming, iterate 30 times
- **SSSP**: init source=0/others=INF, send dist+weight, update if smaller, halt when stable
- **WCC/Max Vertex**: propagate max ID, update if received > current
- Giraph: built on Hadoop, ZooKeeper coordination, hash partitioner

---

### E23. Complex Event Processing
**"Describe Complex Event Processing (CEP) concepts including filters, windows, joins, and temporal patterns. How does CEP relate to stream processing systems?"**

**Key points to cover:**
- Events as tuples, event streams as unbounded tables
- Operations: filter/transform, window aggregation, join, temporal sequence patterns
- Window types: sliding length, batch length, sliding time, batch time
- Join execution: retain events in window, match against incoming events
- CEP as a query language or embedded operator in stream processing
- Examples: Siddhi, Esper
- Comparison with Spark Structured Streaming

---

## Cross-Module Questions

### E24. End-to-End Big Data Pipeline
**"Design a complete big data pipeline for processing and analyzing a large-scale social media dataset. Cover storage, processing, NoSQL storage, streaming, and analytics components."**

**Key points to cover:**
- **Storage**: HDFS/GFS for raw data (large files, sequential access, replication)
- **Batch processing**: Spark RDD/DataFrames for ETL, feature engineering
- **NoSQL**: Dynamo for user profiles (high availability), HBase for analytics data (ordered scans)
- **Streaming**: Kafka for real-time feeds → Spark Structured Streaming for online analytics
- **Graph analytics**: Pregel/Giraph for social graph analysis (PageRank, community detection)
- **ML**: Spark ML for distributed training; FL for privacy-preserving personalization

---

### E25. Consistency Models Comparison
**"Compare the consistency models used across the systems studied in this course: GFS, Spark, Dynamo, HBase, and Kafka."**

**Key points to cover:**
- **GFS**: Defined/consistent/inconsistent regions; relaxed for performance
- **Spark**: Immutable RDDs → no consistency issues; lineage for fault tolerance
- **Dynamo**: Eventual consistency; vector clocks for versioning; application-level reconciliation
- **HBase**: Strong per-row consistency; single logical copy
- **Kafka**: Within-partition ordering only; at-least-once/exactly-once delivery
- Trade-off spectrum: strong consistency (HBase) → eventual (Dynamo)

---

### E26. Fault Tolerance Mechanisms
**"Compare fault tolerance mechanisms across GFS, Spark, Dynamo, Kafka, and Pregel. What design philosophy drives each system's approach?"**

**Key points to cover:**
- **GFS**: Replication (3 copies), checksums, fast restart, ops log + checkpoints
- **Spark**: Lineage-based recomputation; checkpointing for long chains
- **Dynamo**: Sloppy quorums, hinted handoff, Merkle trees, gossip
- **Kafka**: Leader-follower replication, in-sync replicas, offset tracking
- **Pregel**: BSP barriers + superstep checkpointing
- Philosophy: eager replication (GFS, Kafka) vs lazy recomputation (Spark) vs eventual sync (Dynamo)

---

### E27. Scalability Bottlenecks
**"For each major system studied (GFS, Spark, Dynamo, Parameter Server), identify the primary scalability bottleneck and how it is addressed or mitigated."**

**Key points to cover:**
- **GFS**: Single master → bottleneck for metadata ops; mitigated by client caching, large chunks, shadow masters; HopsFS addresses this
- **Spark**: Shuffle during wide dependencies → network bottleneck; mitigated by map-side combiners, partitioning, persist
- **Dynamo**: Write conflicts under partitions → mitigated by sloppy quorums, vector clocks
- **Parameter Server**: Server bandwidth O(N) with workers → mitigated by All-Reduce O(1) bandwidth
- **Pregel**: Message volume explosion → mitigated by combiners

---

### E28. ACID vs BASE in Practice
**"Compare ACID and BASE approaches to data management. Using specific examples from the course (HBase, Dynamo, Data Lakes), explain when each is appropriate."**

**Key points to cover:**
- ACID: Atomicity, Consistency (DB rules), Isolation, Durability — simplifies development
- BASE: Basically Available, Soft State, Eventual Consistency — enables scale
- HBase: Row-level ACID, not cross-row → compromise
- Dynamo: Full BASE; application handles conflicts
- Data Lakes: BASE; schema-on-read, no transactional guarantees
- Decision factors: latency requirements, data criticality, operational vs analytical workloads
- Brewer's key insight: these are continuums, not binary choices

---

### E29. Replication Strategies Across Systems
**"Compare replication strategies in GFS, Dynamo, and Kafka. How does each system balance consistency, availability, and performance through its replication design?"**

**Key points to cover:**
- **GFS**: Master-directed, 3 replicas across racks, primary-secondary with leases, consistent via serial order
- **Dynamo**: Peer-to-peer, sloppy quorum (R+W>N), preference lists, eventual consistency
- **Kafka**: Leader-follower per partition, in-sync replicas, consumers/producers contact only leader
- Performance trade-offs: GFS optimizes for throughput, Dynamo for availability, Kafka for ordered throughput
- Failure handling: GFS re-replicates, Dynamo hinted handoff + Merkle trees, Kafka promotes in-sync follower

---

### E30. Evolution of Distributed Data Processing
**"Trace the evolution from MapReduce → Spark RDD → Spark DataFrames → Spark Structured Streaming. What problem does each advancement solve?"**

**Key points to cover:**
- **MapReduce → RDD**: Disk I/O between stages → in-memory with lineage; iterative workloads
- **RDD → DataFrame**: Opaque functions → schema-aware optimization (Catalyst); code generation
- **DataFrame → Structured Streaming**: Batch-only → unified batch+stream API; event-time semantics
- Common thread: higher-level abstractions enable better optimization
- Trade-off: expressiveness vs optimization opportunity

---

> 💡 **Essay Writing Strategy**: For 10-15 mark essays, spend ~2 minutes planning your structure. Use the template in the Essay Template document. Always include a specific system example and discuss trade-offs.


# 📝 DS256 SSDS — Essay Questions (Final Exam Style)

> Modeled after the **exact 3 question types** from the 2025 final paper:
> - **Type A**: "If you had to redesign X for Y workloads..."
> - **Type B**: "Compare X and Y on scalability, reliability, and performance"
> - **Type C**: "You are the chief architect. Design a pipeline for..."

---

## TYPE A: Redesign for New Workloads

---

### EA1. Redesigning Dynamo for Social Media Workloads

**"If you had to redesign Amazon Dynamo today to support a social media platform (e.g., Twitter/X) instead of a shopping cart, what design choices would you change? Describe the new workload assumptions and justify your design changes."**

#### Model Answer

**Workload Comparison**

| Aspect | Shopping Cart (Original Dynamo) | Social Media Feed |
|--------|-------------------------------|-------------------|
| Object Size | Small blobs (<1MB) | Variable — tweets (small), media attachments (large), follower lists (very large) |
| Access Pattern | Per-user key access | Fan-out reads (my feed = posts from all followed users), celebrity hotspots |
| Consistency Need | Eventual OK (merge carts) | Eventual OK for feeds, but strong for likes/follower counts (avoid double-count) |
| Write Pattern | Low write-rate per key | Massive write fan-out (1 tweet → millions of feed updates) |
| Read:Write Ratio | Balanced | Extremely read-heavy (100:1 or more) |
| Hotspots | Rare (per-user keys) | Extreme — celebrity accounts, trending topics |

**Design Changes**

**1. Hybrid Consistency Model (per-operation, not per-system)**
- Original: single eventual consistency for everything
- Change: **Tunable per-data-type** — eventual for feed rendering (stale OK), strong for counters (likes, retweets, follower counts)
- **Rationale**: Showing a tweet 2 seconds late is acceptable; showing wrong like counts erodes trust. Use CRDTs (Conflict-free Replicated Data Types) for counters instead of vector clocks — they merge automatically without application-level reconciliation.

**2. Read Path Caching Layer**
- Original: All reads go to preference list replicas
- Change: Add a **distributed in-memory cache** (like Memcached/Redis) in front of Dynamo
- **Rationale**: Social feeds are read millions of times but change infrequently. Without caching, celebrity profile reads would overwhelm the physical nodes in the preference list. Cache absorbs 99%+ of reads, Dynamo handles writes and cache misses.

**3. Fan-out-on-Write vs Fan-out-on-Read**
- Original: Simple get/put per key
- Change: **Hybrid fan-out** — pre-compute feeds for regular users (fan-out-on-write), compute on-the-fly for celebrity followers (fan-out-on-read)
- **Rationale**: When a user with 10 followers posts, write the tweet to each follower's feed (10 writes). When a celebrity with 50M followers posts, DON'T write to 50M feeds — instead, merge celebrity tweets at read time. This avoids write amplification for hot accounts.

**4. Multi-tier Storage**
- Original: All data on same storage tier
- Change: **Hot/warm/cold tiering** — recent tweets in memory, 24-hour tweets on SSD, older tweets on HDD/object storage
- **Rationale**: Social media has extreme temporal locality — 90% of reads are for content from the last few hours. Keeping hot data in memory/SSD dramatically reduces latency at the 99.9th percentile.

**5. Consistent Hashing with Hotspot Mitigation**
- Original: Static virtual node assignment per physical node
- Change: **Dynamic virtual node migration** — detect hot keys at runtime, split overloaded virtual nodes and redistribute to less-loaded physical nodes
- **Rationale**: Trending topics create extreme load on specific hash ranges. Static assignment cannot adapt. Dynamic migration (with gossip protocol updates) balances load within minutes.

**6. Richer Data Model**
- Original: Opaque blob values with get/put only
- Change: Support **secondary indices** on timestamp, user-ID, and hashtags for feed queries
- **Rationale**: "Show me my feed from the last hour" requires range queries by time, not just key lookup. Without secondary indices, the application must scan and filter client-side, wasting bandwidth.

---

### EA2. Redesigning Spark for Real-Time ML Inference

**"If you had to redesign Apache Spark today to support real-time ML inference workloads (not just batch training), what design choices would you make? Describe the workload assumptions and justify your changes."**

#### Model Answer

**Workload Comparison**

| Aspect | Batch Analytics (Original Spark) | Real-Time ML Inference |
|--------|--------------------------------|----------------------|
| Latency | Seconds to minutes acceptable | Single-digit milliseconds required |
| Data Volume | TB-scale datasets | Single request (KB) at a time |
| State | Stateless transformations or periodic checkpoint | Model weights in GPU memory (stateful, long-lived) |
| Compute | CPU-based, data-parallel | GPU-based, model-parallel |
| Scheduling | Job → Stage → Task granularity | Per-request or per-iteration scheduling |
| Fault Tolerance | Lineage recomputation (seconds OK) | Must not drop requests; failover in ms |

**Design Changes**

**1. Persistent GPU Executors**
- Original: Executors are CPU processes that start/stop per application
- Change: Long-lived GPU executors with **pre-loaded model weights** in GPU memory
- **Rationale**: Loading a 70B parameter model takes minutes. Cannot reload per request. Executors must persist across requests, keeping model weights resident on GPUs.

**2. Request-Level (not Job-Level) Scheduling**
- Original: Each action creates a job → DAG → stages → tasks
- Change: **Iteration-level scheduling** (Orca-style) — scheduler assigns individual requests to GPU executors per iteration
- **Rationale**: Creating a job/DAG per inference request adds 10-100ms overhead — unacceptable when the SLA is 5ms. Need lightweight per-request routing to pre-loaded model executors.

**3. Streaming Input via Kafka Integration**
- Original: Read from HDFS/files in batch
- Change: Native **Kafka consumer** as the primary input source, with request queuing and priority scheduling
- **Rationale**: Inference requests arrive as a continuous stream, not batch files. Kafka provides durable queuing, backpressure, and replay for failed requests.

**4. Model-Parallel Executor Groups**
- Original: Executors are independent; partitions are independent
- Change: **Executor groups** that collectively hold one model across multiple GPUs with NVLink/InfiniBand interconnection
- **Rationale**: Large models are split across GPUs (model parallelism). An inference request must pass through ALL GPUs holding model layers sequentially. This requires coordinated executor groups, not independent tasks.

**5. KV Cache Management**
- Original: RDD caching in memory/disk with LRU eviction
- Change: **Per-request KV cache** in GPU memory with intelligent eviction based on sequence length and request priority
- **Rationale**: Transformer inference generates KV caches that grow with sequence length. Managing this memory efficiently (like vLLM's PagedAttention) is critical for throughput.

**6. Tail-Latency Optimization**
- Original: Recompute lost partitions from lineage (acceptable for batch)
- Change: **Request hedging** — send duplicate requests to multiple executor groups, return first response
- **Rationale**: p99.9 latency matters for user-facing inference. Hedging absorbs slow-node variance. Lineage recomputation is too slow for real-time SLAs.

---

### EA3. Redesigning Pregel/Giraph for Dynamic Graphs

**"If you had to redesign Pregel today to support dynamic graphs (where vertices and edges are continuously added/removed), what changes would you make? Consider workloads like real-time social network analysis or fraud detection."**

#### Model Answer

**Workload Comparison**

| Aspect | Static Graph (Original Pregel) | Dynamic Graph (Social/Fraud) |
|--------|-------------------------------|------------------------------|
| Graph Mutation | Fixed — loaded once, never changes | Edges/vertices added/removed continuously |
| Computation | Full graph recomputation per algorithm run | Incremental — only affected subgraph needs update |
| Latency | Batch — minutes to hours acceptable | Near-real-time — seconds for fraud alerts |
| Scale | Billions of edges, processed at once | Streaming edges at 10K+ per second |

**Design Changes**

**1. Incremental Supersteps**
- Original: All vertices active in superstep 0; compute on entire graph
- Change: **Event-triggered activation** — only vertices affected by a graph mutation become active
- **Rationale**: Adding one edge shouldn't require recomputing PageRank for the entire graph. Only the affected vertex and its neighborhood should re-execute. Reduces per-update cost from O(V) to O(affected neighborhood).

**2. Streaming Graph Ingestion**
- Original: Load entire graph from HDFS before computation starts
- Change: **Kafka-based edge stream** — new edges/vertices arrive as events, applied incrementally to the in-memory graph
- **Rationale**: Dynamic graphs change continuously. Batch loading + full recomputation is too slow for fraud detection (need alerts within seconds of suspicious transaction).

**3. Hybrid BSP + Asynchronous Execution**
- Original: Strict BSP — global barrier after every superstep
- Change: **Asynchronous message processing within a bounded window** — vertices can process messages from recent supersteps without waiting for global barrier
- **Rationale**: Global barriers become expensive when only a small fraction of vertices are active. Async processing allows fast local convergence while maintaining global consistency bounds.

**4. Temporal Awareness**
- Original: No notion of time in vertex/edge attributes
- Change: **Timestamped edges** with window-based retention and temporal graph algorithms
- **Rationale**: Fraud detection needs "transactions in the last 5 minutes forming a cycle." Temporal edges enable time-windowed subgraph queries without materializing the full history.

**5. Dynamic Partitioning**
- Original: Hash partitioning once at load time
- Change: **Streaming repartitioning** — monitor edge distribution, migrate vertices between workers when load imbalance exceeds threshold
- **Rationale**: Dynamic graphs develop hotspots (viral content, new celebrities). Static partitioning creates permanent imbalance. Periodic rebalancing keeps workers evenly loaded.

---

## TYPE B: Compare Platforms on Specific Axes

---

### EB1. GFS/HDFS vs Dynamo

**"HDFS and Dynamo are both distributed storage systems but serve very different workloads. Compare and contrast their design approaches for scalability, reliability, and performance."**

#### Model Answer

| Dimension | HDFS | Dynamo |
|-----------|------|--------|
| **Design Goal** | Store and serve large files for batch analytics | Store and serve small key-value objects for interactive services |
| **Data Model** | Hierarchical file system (directories, files, blocks) | Flat key-value store (key → blob) |
| **Consistency** | Strong for metadata (single master); relaxed for data (defined/consistent/inconsistent) | Eventual (sloppy quorums, vector clocks) |
| **Coordination** | Centralized master (NameNode) | Fully decentralized (P2P, gossip) |

**Scalability**
- HDFS: Scales data by adding DataNodes; metadata bottlenecked at single NameNode (mitigated by HopsFS). Designed for weak scaling of data volume.
- Dynamo: Scales by adding physical nodes → redistributing virtual nodes via consistent hashing. No centralized bottleneck. Scales both data and request throughput linearly.
- **Key difference**: HDFS has a centralized metadata bottleneck; Dynamo's fully decentralized design scales more uniformly but sacrifices query richness.

**Reliability**
- HDFS: 3-way replication across racks; NameNode has shadow replicas and operation logs; checksums per 64KB block for corruption detection; fast restart (seconds).
- Dynamo: N-way replication via preference lists; hinted handoff for transient failures; Merkle trees for permanent divergence detection; gossip for membership propagation.
- **Key difference**: HDFS detects and repairs corruption proactively (checksums + re-replication). Dynamo tolerates divergence and repairs lazily (anti-entropy). HDFS prioritizes data integrity; Dynamo prioritizes availability.

**Performance**
- HDFS: Optimized for high-throughput sequential reads (streaming large files). 64MB blocks minimize metadata lookups. Poor for random reads of small objects.
- Dynamo: Optimized for low-latency point reads/writes (99.9th percentile SLA). Consistent hashing enables O(1) key lookup. Poor for range scans or batch processing.
- **Key difference**: HDFS is throughput-oriented (MB/s); Dynamo is latency-oriented (ms).

**When to choose which**: HDFS for data lakes, training data, batch ETL. Dynamo for user profiles, session state, shopping carts — any workload needing always-on availability with simple access patterns.

---

### EB2. Parameter Server vs Federated Learning

**"Parameter Server and Federated Learning both enable distributed model training. Compare their approaches to scalability, reliability, privacy, and performance."**

#### Model Answer

| Dimension | Parameter Server | Federated Learning |
|-----------|-----------------|-------------------|
| **Setting** | Data center (high bandwidth, reliable) | Wide-area / edge (low bandwidth, unreliable) |
| **Data Location** | Centralized — partitioned across workers | Decentralized — stays on edge devices |
| **What's Shared** | Gradients (push to server, pull updated params) | Local models or model updates (sent to aggregation server) |
| **Privacy** | Low — gradients can leak training data | High — raw data never leaves device; secure aggregation possible |

**Scalability**
- PS: Server bandwidth bottleneck O(N) with workers. Mitigated by sharding parameters across multiple servers. Works well up to ~1000 workers in a data center.
- FL: Server handles model aggregation from 100s of selected clients per round (from 1000s-millions available). Communication bottleneck at server managed by client selection and pace steering. Scales to millions of devices.
- **Key difference**: PS scales compute (more GPUs → faster training). FL scales participation (more devices → more diverse data). PS bottleneck is bandwidth; FL bottleneck is stragglers and heterogeneity.

**Reliability**
- PS: Server failure = training stops (unless replicated). Worker failure = retry the batch. Bounded-delay model tolerates slow workers.
- FL: Round discarded if too few clients report. Individual device failures are expected and handled gracefully (stragglers ignored). Devices are unreliable by design.
- **Key difference**: PS assumes mostly-reliable infrastructure with occasional failure. FL assumes failure is the norm and designs around it.

**Privacy**
- PS: Gradients sent to centralized servers — vulnerable to gradient inversion attacks. All training data centralized.
- FL: Data never leaves device. Secure aggregation masks individual updates. Differential privacy can be applied locally.
- **Key difference**: FL was explicitly designed for privacy; PS was not. This is the primary motivation for FL's existence.

**Performance**
- PS: Fast convergence — high bandwidth within data center, synchronous or bounded-delay updates, full data visible to all workers.
- FL: Slower convergence — limited bandwidth to edge, straggler effects, non-IID data distributions across clients cause model drift.
- **Key difference**: PS converges faster; FL pays a convergence penalty for privacy and decentralization. FedAvg often needs 10-100x more rounds than centralized SGD.

---

### EB3. HBase vs Dynamo/Cassandra

**"HBase and Cassandra are both NoSQL databases used at scale. Compare and contrast their approaches to data modeling, consistency, and performance."**

#### Model Answer

| Dimension | HBase (BigTable-style) | Cassandra (Dynamo-style) |
|-----------|----------------------|--------------------------|
| **Data Model** | Columnar: row-key → column family → column qualifier → version | Wide-column similar, but designed as key-value with CQL |
| **Consistency** | Strong per-row (single logical copy via master) | Tunable quorum (R, W, N) — eventual to strong |
| **Architecture** | Master-based (HMaster + RegionServers), HDFS for persistence | Masterless P2P (consistent hashing ring), local storage |
| **Write Path** | WAL → MemStore → HFile (immutable, on HDFS) | Commit log → MemTable → SSTable (local disk) |
| **Read Path** | MemStore → BlockCache → HFile | MemTable → row cache → bloom filter → SSTable |
| **Partitioning** | Row-key range (ordered) | Consistent hashing (hash-based, unordered by default) |

**Scalability**: HBase scales by splitting regions (auto-sharding on row-key ranges). Single HMaster is a coordination bottleneck (mitigated by ZooKeeper leader election). Cassandra's masterless ring scales more uniformly — add nodes, rebalance virtual tokens automatically.

**Consistency**: HBase provides strong per-row consistency — simpler for applications but limits availability during master failure. Cassandra's tunable consistency (ONE, QUORUM, ALL) lets applications choose per-query, but developers must reason about staleness.

**Performance**: HBase excels at **ordered range scans** (row-key ordered). Cassandra excels at **write-heavy workloads** (always writable, no master coordination). For read-heavy analytics with scans → HBase. For write-heavy operational workloads → Cassandra.

---

### EB4. Pregel/Giraph vs Spark GraphX

**"Pregel/Giraph and Spark GraphX both support large-scale graph processing. Compare and contrast their execution models, performance characteristics, and usability."**

#### Model Answer

| Dimension | Pregel/Giraph | Spark GraphX |
|-----------|--------------|-------------|
| **Model** | Vertex-centric BSP with message passing | RDD-based with graph-specific operators (Pregel-like on top) |
| **State** | Graph in-memory across all supersteps; only messages traverse network | Graph stored as RDDs; persisted/cached as needed; rebuilt on failure |
| **Iteration** | Native — supersteps are first-class concept | Through iterative Spark jobs; requires explicit loop in driver |
| **Shuffling** | Only messages are shuffled; graph topology is static, in-memory | Graph structure may be shuffled during joins every iteration |

**Performance**: Pregel/Giraph avoids reshuffling graph topology — only messages cross the network (proportional to number of active edges, not total graph size). GraphX must join the edge/vertex RDDs each iteration, causing a shuffle proportional to graph size, even when only a few vertices are active. For iterative algorithms (PageRank, SSSP), Giraph is typically **2-10x faster**.

**Usability**: GraphX benefits from Spark's unified ecosystem — same DataFrame/SQL/ML APIs, same cluster, same monitoring. Giraph requires a separate Hadoop cluster setup. For teams already using Spark, GraphX avoids operational complexity. For graph-heavy workloads at scale, Giraph's performance advantage justifies the dedicated infrastructure.

**Fault Tolerance**: GraphX uses Spark's lineage — lost partitions recomputed from RDD lineage. Giraph uses periodic superstep checkpointing — on failure, rolls back to last checkpoint and replays. Both work but with different granularity and cost.

---

## TYPE C: "You Are the Architect"

---

### EC1. Real-Time Fraud Detection Pipeline

**"You are the chief architect of a digital payments company processing 1 million transactions per second. Design a data pipeline for real-time fraud detection. Discuss the choice of platforms at each stage and justify your decisions."**

#### Model Answer

```
Transaction Stream → Ingestion → Feature Enrichment → ML Scoring → Alert/Action → Data Lake
```

**1. Event Ingestion: Apache Kafka**
- Transactions arrive as events with (timestamp, sender, receiver, amount, location, device_id)
- Kafka topics partitioned by sender_id for ordering guarantees per user
- **Justification**: Kafka handles 1M+ msgs/sec with durable storage, exactly-once semantics with idempotent producers. Partition-level ordering ensures a user's transactions are processed in sequence (critical for detecting velocity patterns).

**2. Real-Time Feature Enrichment: Spark Structured Streaming + HBase**
- Streaming join: enrich each transaction with user profile, historical spending patterns
- HBase stores user profiles and recent transaction history (row-key = user_id)
- **Justification**: HBase provides low-latency point lookups by user_id. Spark Structured Streaming handles stateful windowed aggregations (e.g., "total spent in last 5 minutes"). Event-time windows with watermarks handle out-of-order transactions.

**3. ML Fraud Scoring: Online Inference**
- Pre-trained fraud model deployed on GPU-equipped servers
- Each enriched transaction scored in real-time (<10ms SLA)
- Use Orca-style iteration-level scheduling for batching inference requests
- **Justification**: Low latency is critical — block fraudulent transactions before settlement. Selective batching maximizes GPU utilization while meeting latency SLAs.

**4. Graph Analysis: Pregel/Giraph (Async)**
- Model transaction network as a graph (users = vertices, transactions = edges)
- Detect suspicious patterns: money laundering cycles, rapid fan-out/fan-in
- Community detection and anomaly scoring on the transaction graph
- **Justification**: Graph algorithms (connected components, cycle detection) express fraud patterns naturally. Pregel's vertex-centric model parallelizes well across the massive transaction graph.

**5. Alert & Action: CEP Engine (Siddhi)**
- Define fraud rules as temporal event patterns: "3+ failed transactions in 2 minutes from different locations"
- Trigger alerts, block cards, notify users
- **Justification**: CEP's pattern-matching over event streams handles rule-based detection complementing ML scores. Low-latency execution (<ms per rule evaluation).

**6. Historical Storage: Data Lake (HDFS/S3 + Spark)**
- All transactions archived to data lake in Parquet format
- Batch Spark jobs retrain fraud models daily on historical data
- **Justification**: Parquet's columnar format enables efficient feature extraction for retraining. HDFS provides scalable, cost-effective storage for petabytes of transaction history.

**Cross-Cutting Decisions**:
- **Consistency**: Eventual for enrichment (slight staleness OK), strong for fraud decisions (block/allow is irrevocable)
- **Fault Tolerance**: Kafka provides replay on failure; Spark checkpoints stateful aggregations; Pregel checkpoints supersteps

---

### EC2. IoT Smart City Platform

**"You are the chief architect for a smart city IoT platform that manages 100,000 sensors (water, power, traffic). Design the data pipeline from sensor to dashboard and predictive analytics."**

#### Model Answer

**1. Sensor Data Ingestion: MQTT → Kafka**
- Sensors publish via MQTT (lightweight IoT protocol) to edge gateways
- Edge gateways forward to Kafka topics (one per sensor type: water_level, power_consumption, traffic_flow)
- **Justification**: MQTT handles resource-constrained sensors. Kafka decouples sensors from consumers, provides durable buffering for bursty traffic, handles 100K+ sensors easily.

**2. Real-Time Alerting: CEP (Siddhi)**
- Continuous queries: "water level > 90% capacity," "power consumption spike > 3σ from mean"
- Window-based anomaly detection (sliding time windows of 5 minutes)
- **Justification**: CEP provides sub-second alerting for safety-critical events. SQL-like pattern syntax is maintainable by operations teams.

**3. Stream Processing: Spark Structured Streaming**
- Running aggregations: per-building power consumption, per-zone traffic flow
- Streaming joins: correlate water usage with occupancy data
- Event-time windows with watermarks for late-arriving sensor data
- **Justification**: Unified batch+stream API allows same code for real-time dashboards and historical reports. Handles late/out-of-order events gracefully.

**4. Operational Storage: HBase**
- Time-series data: row-key = sensor_id + reverse_timestamp
- Column families: raw_value, computed_metrics, alerts
- **Justification**: HBase's ordered row-key scans enable efficient "show me last 24 hours for this sensor" queries. Column families separate hot (recent) from cold (historical) data.

**5. Predictive Analytics: Spark ML on Data Lake**
- Daily batch training on HDFS data lake (Parquet format)
- Models: water demand forecasting, traffic flow prediction, anomaly detection
- Feature engineering using Spark DataFrames
- **Justification**: Spark ML scales to millions of sensor readings. Parquet's columnar format enables efficient feature extraction (read only needed columns).

**6. Dashboard: Redis Cache + Web API**
- Pre-computed dashboard metrics cached in Redis for sub-ms response
- Invalidated on each Spark Structured Streaming micro-batch
- **Justification**: Dashboards need instant response for 100s of concurrent users. Cache absorbs read load; Spark handles compute.

---

### EC3. Distributed Training Platform for a Research Lab

**"You are designing the infrastructure for a university research lab that needs to train models ranging from small CNNs to large language models. Design a flexible training platform. Consider both single-GPU and multi-node training needs."**

#### Model Answer

**1. Data Management: HDFS + Spark DataFrames**
- Research datasets stored on HDFS (ImageNet, Common Crawl, custom datasets)
- Spark DataFrames for data cleaning, augmentation, format conversion
- Store processed data in TFRecord/WebDataset format for efficient streaming
- **Justification**: Researchers bring diverse datasets. HDFS handles storage scalably. Spark enables reproducible ETL pipelines shared across the lab.

**2. Small Model Training (fits in 1 GPU): Single-Node PyTorch**
- Standard PyTorch training loop with local data loading
- Data stored on NVMe SSD attached to GPU node for fast random access
- **Justification**: Overhead of distributed frameworks not justified for small models. Direct SSD access provides maximum data loading throughput.

**3. Medium Model Training (fits across 2-8 GPUs): Data Parallelism + All-Reduce**
- PyTorch DistributedDataParallel (DDP) with NCCL backend
- Each GPU holds full model replica; mini-batch split across GPUs
- Ring All-Reduce for gradient synchronization
- **Justification**: All-Reduce achieves O(1) peak bandwidth vs O(N) for PS. DDP is simpler to implement than PS for data center settings. NCCL optimizes for GPU-to-GPU communication.

**4. Large Model Training (>single node): 3D Parallelism (DeepSpeed)**
- Data parallelism across nodes
- Model parallelism within a node (tensor splitting across GPUs)
- Pipeline parallelism across GPU groups (micro-batch pipelining)
- **Justification**: LLMs require all three dimensions. DeepSpeed/Megatron provide off-the-shelf implementations. ZeRO optimizer reduces memory footprint by partitioning optimizer states.

**5. Experiment Tracking: MLflow + HBase**
- MLflow for experiment metadata (hyperparameters, metrics, model versions)
- HBase for storing per-step training metrics at scale (row-key = experiment_id + step)
- **Justification**: Researchers need to compare 100s of runs. MLflow provides UI; HBase handles the volume of per-step metrics efficiently.

**6. Privacy-Sensitive Collaboration: Federated Learning (Flotilla)**
- Cross-silo FL for multi-institution collaborations (medical imaging, etc.)
- Synchronous FedAvg with secure aggregation
- **Justification**: Research collaborations across hospitals/institutions cannot share raw data (HIPAA/GDPR). FL enables joint model training; secure aggregation prevents model inversion attacks.

---

### EC4. E-Commerce Recommendation System

**"You are the chief architect of a large e-commerce platform. Design the data infrastructure for a real-time recommendation system that serves personalized product suggestions. Cover data storage, processing, model training, and serving."**

#### Model Answer

**1. User Interaction Streaming: Kafka**
- Topics: click_stream, purchase_events, search_queries, cart_events
- Partitioned by user_id for per-user ordering
- 7-day retention for replay/reprocessing
- **Justification**: Captures the full user behavior funnel in real-time. Partition-by-user ensures chronological ordering of a user's actions.

**2. User/Product Profiles: Dynamo (or Cassandra)**
- User profiles: browsing history, purchase history, preferences
- Product catalog: features, category, embeddings
- Key = user_id or product_id; blob = JSON profile
- **Justification**: Always-available reads for real-time recommendation serving. Eventual consistency acceptable — showing a slightly stale recommendation is fine. Dynamo's tunable quorum balances latency vs freshness.

**3. Graph Relationships: Neo4j or Pregel**
- User-product bipartite interaction graph
- Product-product co-purchase graph
- Collaborative filtering: "users who bought X also bought Y"
- **Justification**: Graph structure captures relationships naturally. Neo4j for real-time traversal queries. Pregel/Giraph for batch PageRank-style scoring on the full graph.

**4. Feature Engineering: Spark DataFrames**
- Batch processing: aggregate click rates, purchase frequencies, user segments
- Join user profiles with product catalog for feature vectors
- Output: feature store (HBase with row-key = user_id or product_id)
- **Justification**: Spark scales ETL to billions of interactions. DataFrame optimizations (Catalyst) enable efficient joins. HBase feature store provides low-latency serving.

**5. Model Training: Distributed PyTorch + All-Reduce**
- Deep learning recommendation model (two-tower, wide-and-deep)
- Data parallelism with mini-batch training
- Daily retraining on full interaction history
- **Justification**: Recommendation models with user/product embedding tables can be large (100s of GB). Data parallelism distributes the training data; embedding tables may require model parallelism.

**6. Real-Time Serving: Orca-style Batched Inference**
- Pre-loaded model on GPU servers
- User request → lookup features from HBase → score candidates → rank → return top-K
- Batch inference requests from multiple users for GPU efficiency
- **Justification**: p99 latency < 50ms is critical for user experience. Batching amortizes GPU kernel launch costs. KV cache not needed (not autoregressive), but feature cache is important.

**7. A/B Testing: Kafka + Spark Streaming**
- Route user traffic to model variants via Kafka topic partitioning
- Spark Streaming computes real-time metrics (CTR, conversion) per variant
- **Justification**: Cannot wait for batch results — need to detect bad models within hours, not days. Streaming metrics enable rapid experiment decisions.

---

### EC5. Multi-Modal AI Training Pipeline

**"You are designing a training pipeline for a multi-modal AI system that processes text, images, and video together (like GPT-4V or Gemini). Discuss the unique challenges compared to text-only LLM training and your platform choices."**

#### Model Answer

**Unique Challenges vs Text-Only**

| Challenge | Text-Only LLM | Multi-Modal AI |
|-----------|--------------|----------------|
| Data Size | ~1TB tokenized text | 10-100TB+ (video frames, high-res images) |
| Data Format | Uniform token sequences | Heterogeneous: text tokens, image patches, video frames |
| Model Architecture | Decoder-only transformer | Multiple encoders (vision, text) + cross-attention fusion |
| Memory | Large but predictable (# params × bytes per param) | Variable — video inputs have much larger activation maps |
| Data Loading | Sequential text streaming | Multi-resolution images + temporal video sampling |

**Platform Choices**

**1. Multi-Tier Storage**: Object storage (S3/HDFS) for raw media + SSD cache for training-active data + pre-processed feature cache in Redis/HBase. **Rationale**: Video data is too large for SSD-only; tiered caching keeps active training data fast while archiving the full corpus cheaply.

**2. Data Processing**: Spark DataFrames for metadata ETL + custom GPU-based preprocessing (DALI/ffmpeg) for image/video decoding. **Rationale**: Image/video decoding is compute-intensive — must be done on GPU to avoid CPU bottleneck in data loading pipeline.

**3. Training**: 3D parallelism with separate parallelism strategies per encoder. Vision encoder → tensor parallelism (large activation maps). Text decoder → pipeline parallelism (many layers). Cross-attention → data parallelism. **Rationale**: Different modality encoders have different bottleneck profiles. One-size-fits-all parallelism is suboptimal.

**4. Checkpointing**: Incremental, asynchronous checkpointing (DeepSpeed) to object storage. Checkpoint only changed parameters between modalities. **Rationale**: Training runs take weeks; a single failure without checkpointing wastes millions of GPU-hours. Incremental saves reduce I/O overhead vs full checkpoint.

---

> 💡 **How to approach these in the exam**: Spend 2-3 minutes drawing the pipeline diagram first, then fill in each stage with platform + justification. Always end with "Cross-Cutting Decisions" covering consistency, fault tolerance, and scaling trade-offs.