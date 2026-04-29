# SSDS Final Exam Cross-System and Cross-Application Notes

These notes are organized to help answer mixed-topic final exam questions. They are built primarily from the course notes and lecture slides for Modules 1 to 5, with paper ideas used only when they clearly support the lecture material.

## Primary Course Source Map

- Module 1: Distributed file systems, GFS, HDFS consistency and mutations.
- Module 2: Spark processing stack, RDDs, internals, DataFrames, SQL, Catalyst.
- Module 3: Spark ML, distributed DNN training, federated learning, distributed GNNs, transformers and ORCA serving.
- Module 4: ACID, CAP, BASE, Dynamo, HBase, data lakes.
- Module 5: Kafka, CEP, Structured Streaming, Pregel.

Exam framing note: for the final, prefer the course's terminology and system model even if the real systems have evolved since the original papers. For example, the course material discusses Kafka with ZooKeeper-era terminology, GFS/HDFS with lease-based primaries, and Spark Streaming through the course's Structured Streaming framing.

## How to Answer Cross-Topic Questions

For almost every open-ended answer, use this order:

1. State the workload.
2. State the primary abstraction used by the system.
3. Explain partitioning and where parallelism comes from.
4. Explain replication, consistency, and failure handling.
5. Explain the main performance bottleneck the system is designed to reduce.
6. End with tradeoffs and why this system is better than nearby alternatives.

If the question mixes multiple systems, compare them on the same axes instead of describing them independently.

## The Big Idea of the Course

The course is really about one repeated design question:

How do we scale storage, processing, learning, querying, and serving by partitioning work across machines without losing too much correctness, performance, or programmability?

Every module answers that question differently:

- GFS/HDFS scale storage using block partitioning and replication.
- Spark scales computation using partitioned datasets and staged execution.
- Distributed ML scales training by partitioning data, model state, or both.
- NoSQL systems scale data services by relaxing assumptions from classical RDBMS designs.
- Kafka, CEP, Structured Streaming, and Pregel scale fast or linked data using partitioned streams or graphs.

## 1. One-Line Mental Model for Each System

| System | Mental model | Best at | Main tradeoff |
|---|---|---|---|
| GFS/HDFS | Distributed file system for large immutable or append-heavy files | High-throughput scans over huge files | Poor for fine-grained random updates |
| HBase | Sparse, distributed, wide-column table on top of HDFS | Row-key lookup, range scan, subset-column access | Row-key design is critical; not ideal for full linear scans at lake scale |
| Dynamo | Highly available distributed key-value store | Always-writeable key-based access under failures | Eventual consistency and application-level conflict resolution |
| Spark RDD | General distributed functional compute abstraction | Arbitrary data-parallel batch processing | Limited optimization because lambdas/types are opaque |
| Spark DataFrame/SQL | Declarative distributed table abstraction | Optimized analytics and ETL | Less control than hand-written RDD pipelines |
| Spark Structured Streaming | Incremental execution of DataFrame queries over streams | Stateful stream processing with unified batch/stream model | Micro-batch overhead and state-management complexity |
| Spark ML | Distributed classical ML pipelines | Feature engineering plus scalable non-deep ML | Less expressive than deep learning frameworks |
| Parameter Server | Distributed shared model state for data-parallel training | Flexible distributed parameter synchronization | Central bandwidth bottleneck |
| Federated Learning | Train locally, aggregate globally without moving raw data | Privacy-sensitive edge learning | Statistical heterogeneity and wide-area coordination |
| Distributed GNN training | Distributed message-passing-based neural learning over graphs | Learned graph prediction tasks | Neighborhood explosion and communication overhead |
| Kafka | Partitioned replicated event log and pub-sub broker | High-throughput event ingest, replay, decoupling producers/consumers | By itself it does not do rich analytics |
| CEP | Continuous query and event-pattern engine | Event-time/window/pattern logic over streams | Usually narrower than full distributed analytics platforms |
| Pregel/Giraph | Vertex-centric BSP graph processing model | Iterative graph algorithms over massive graphs | Specialized model, less general than Spark |
| ORCA-style LLM serving | Iteration-level scheduling for generative inference | Better latency/throughput for transformer serving | Serving becomes scheduler and memory-management heavy |

## 2. Cross-Cutting Themes That Connect All Modules

### 2.1 Partitioning Is the Main Source of Scale

Partitioning appears in every module, but the object being partitioned changes:

| Domain | Unit of partitioning | Why it matters |
|---|---|---|
| HDFS/GFS | File blocks/chunks | Enables storage growth, data locality, parallel reads |
| Kafka | Topic partitions | Enables ordered logs, parallel consumers, leader/follower replication |
| HBase | HRegions by row-key range | Enables range scans and load balancing |
| Dynamo | Virtual nodes on a hash ring | Enables incremental scaling and availability |
| Spark | RDD/DataFrame partitions | Enables task-level parallelism |
| Structured Streaming | Input partitions plus state partitions | Ensures scalable stateful processing |
| Pregel | Vertex partitions | Determines communication cost and load balance |
| Parameter Server | Parameter shards | Spreads model state across servers |
| Federated Learning | Clients/devices | Keeps data local and parallelizes training |
| Model parallelism | Layers/tensors | Lets large models fit in memory |

Core exam insight: if you can explain what is partitioned, how it is assigned, and what communication crosses partitions, you can explain most performance tradeoffs.

### 2.2 Replication Improves Reliability but Changes Consistency and Cost

| System | What is replicated | Typical write path | Consistency style |
|---|---|---|---|
| GFS/HDFS | File blocks | Primary orders mutation, replicas apply in order | Relaxed but ordered mutation semantics |
| Kafka | Partition replicas | Leader appends, followers replicate | Leader-based ordered log semantics |
| HBase | Data effectively durable through WAL + HDFS replication | MemStore + WAL, then HFiles on HDFS | Strong per-row semantics, not full RDBMS-style global ACID |
| Dynamo | Key replicas across preference list | Write to W replicas, often sloppy quorum | Eventual consistency with vector clocks |
| Structured Streaming state | Checkpointed operator state | Micro-batch updates + durable checkpoint | Exactly-once style recovery depends on source/sink semantics |
| Federated Learning | Global model copied to clients | Local training then aggregation | Not database consistency; statistical convergence problem |

Core exam insight: replication is never free. It increases storage/network overhead and adds coordination or reconciliation cost.

### 2.3 Locality Is a First-Class Performance Goal

- HDFS/GFS place compute near blocks.
- Spark tries to schedule tasks where partitions are located.
- HBase row-key design tries to keep related rows in nearby ranges.
- Kafka preserves locality and ordering inside a partition.
- Pregel tries to keep communicating vertices on the same worker.
- Model parallel and pipeline parallel training try to reduce inter-device transfers.
- ORCA and modern LLM systems care about where KV cache lives, because memory locality dominates decode performance.

### 2.4 Most Systems Are Optimizing One Bottleneck More Than Another

| System | Bottleneck it mainly tries to reduce | Bottleneck it accepts |
|---|---|---|
| HDFS/GFS | Disk and network throughput for giant files | Small-file latency and random updates |
| Spark | Repeated disk I/O for iterative workloads | Shuffles remain expensive |
| DataFrames/Catalyst | Poor manual query plans | Reduced low-level control |
| Kafka | Producer-consumer decoupling and broker throughput | Rich query capability |
| CEP | Expressing time/pattern logic | Large-scale storage by itself |
| Pregel | Re-shuffling graph topology every iteration | General-purpose flexibility |
| Parameter Server | Flexible distributed gradient synchronization | Central communication hotspot |
| Federated Learning | Raw-data movement and privacy risk | Client heterogeneity and slower convergence |
| ORCA | Idle time and head-of-line blocking in serving batches | More complex scheduling logic |

## 3. Storage and Data Management Systems

## 3.1 GFS/HDFS: What They Assume About Data

GFS/HDFS are built around these workload assumptions:

- Failure is normal on commodity clusters.
- Files are large.
- Reads are mostly large and sequential.
- Writes are mostly append-like.
- High bandwidth matters more than low latency.
- Applications can tolerate relaxed semantics better than they can tolerate poor throughput.

This is why HDFS is excellent for data lakes, raw corpora, checkpoints, and large batch pipelines, but poor for workloads that need many tiny point lookups or in-place record updates.

### Why these assumptions matter later in the course

- Spark benefits because HDFS exposes large partitions that are easy to process in parallel.
- HBase exists because HDFS alone is bad at low-latency keyed access.
- Kafka resembles HDFS in being append-centric and partitioned, but differs because it is an event log with consumer offsets.
- AI/ML training corpora often fit HDFS well because training usually prefers large sequential shards over random single-record updates.

## 3.2 HBase: Structured Access on Top of HDFS

HBase adds table semantics over HDFS without becoming a full RDBMS.

Main ideas:

- Row key is the main access path.
- Column families are fixed, columns within a family are dynamic.
- Rows are grouped into HRegions by row-key range.
- Writes go to MemStore and WAL, then flush to immutable HFiles on HDFS.
- Reads can fetch only some columns rather than scanning entire training files.

Why it matters:

- It fixes HDFS's poor random row access.
- It supports sparse semi-structured data.
- It supports range scans if row keys are designed well.
- It is better than HDFS when you need online lookups or partial-field access.

## 3.3 Dynamo: Availability-First Key-Value Storage

Dynamo makes a different tradeoff from HBase.

Main ideas:

- Simple get/put interface over keys and blob values.
- Consistent hashing with virtual nodes for scale.
- Replication through preference lists.
- Sloppy quorum and hinted handoff for high write availability.
- Vector clocks plus application-level reconciliation for conflicts.

Why it matters:

- It is ideal when rejecting writes is unacceptable.
- It is poor if you need range scans, relational joins, or rich schemas.
- It shifts complexity from writes to reads and to the application.

## 3.4 HDFS vs Kafka

This is a very likely comparison because the two systems share some design vocabulary but solve different problems.

### Similarities

- Both partition data across machines.
- Both replicate partitions or blocks for fault tolerance.
- Both favor append-oriented write behavior.
- Both use a single ordering point during writes for a partition or block mutation.
- Both are optimized more for throughput than for small random write latency.

### Key differences

| Dimension | HDFS/GFS | Kafka |
|---|---|---|
| Primary abstraction | File split into blocks | Topic split into partitions |
| Workload | Large file storage and scans | Event ingest, replay, pub-sub |
| Read model | Read file bytes, often from a nearby replica | Read ordered records using offsets |
| Write model | File write/append or record append | Partition leader append |
| Replica read path | DFS can read from any suitable replica | In the course model, consumers read from the leader |
| Retention | Until deleted | Time/size-based retention |
| Multi-consumer semantics | Not built around independent replaying consumers | Native consumer groups and replay |
| Ordering guarantee | Within a file block mutation protocol | Within a partition |
| Best use | Corpus storage, checkpoints, large scans | Logs, clickstreams, telemetry, event pipelines |

### Performance interpretation

- HDFS is for storing the dataset itself.
- Kafka is for transporting and replaying the event history of the dataset.
- HDFS optimizes bulk I/O and long-lived persistence.
- Kafka optimizes continuous ingestion, decoupled consumers, and ordered stream replay.

### Important exam nuance

Do not casually say that Kafka consumers can read from any replica in the same way HDFS clients can read from any data node replica. In the course framing, Kafka consumers read from the partition leader to preserve a single committed order.

## 3.5 HDFS vs HBase

This is one of the cleanest storage comparisons in the course.

| Dimension | HDFS | HBase |
|---|---|---|
| Abstraction | Distributed file system | Wide-column distributed table |
| Access pattern | Large scans, whole-file reads | Row-key lookup, range scan, subset-column access |
| Update pattern | Append-heavy, poor for in-place edits | Fine-grained row updates and versioned cells |
| Schema | File-level, application-defined | Column families fixed, qualifiers dynamic |
| Latency target | Throughput-first | Lower-latency online access |
| Best for training data | Yes, if training reads large sequential shards | Yes, if training repeatedly fetches by row key or subset columns |
| Best for metadata/features | Weak | Strong |

### Rule of thumb

- Use HDFS when the job reads most of the dataset anyway.
- Use HBase when the application needs keyed retrieval, partial columns, or updates to existing rows.

### Training-data interpretation

If you are training by linearly scanning many samples, HDFS is usually better.

If you are storing rich per-sample metadata and repeatedly querying by ID or fetching only some fields, HBase is better.

## 3.6 HBase vs Dynamo

These systems are both NoSQL, but they target different ends of the design space.

| Dimension | HBase | Dynamo |
|---|---|---|
| Data model | Wide-column, row-key plus column family | Key-value blobs |
| Partitioning | Range partitioning by row key | Hash partitioning by consistent hashing |
| Read patterns | Exact row key and range scans | Exact key lookup |
| Consistency | Stronger per-row semantics | Eventual consistency |
| Conflict handling | Simpler row semantics | Vector clocks plus reconciliation |
| Availability stance | Good, but not "always writeable" in Dynamo's sense | Explicitly availability-first |
| Best use | Sparse tables, partial-field reads, scans | Shopping cart/session/preference style services |

### Exam summary line

HBase gives you structure and locality for scans; Dynamo gives you availability and elastic hash-based key access.

## 3.7 CAP, ACID, and BASE in the Course

These ideas explain why NoSQL systems look different from relational databases.

### ACID

- Atomicity, consistency, isolation, durability simplify application logic.
- They usually require coordination.
- Coordination limits horizontal scalability and partition-time availability.

### CAP

- During partitions, the hard choice is really between stronger consistency and higher availability.
- The course stresses that partition tolerance is not optional in large distributed settings.

### BASE

- Basically Available
- Soft State
- Eventual Consistency

BASE fits microservice-style cloud systems where fast response and uptime matter more than immediately perfect agreement.

### How this ties to specific systems

- Dynamo is the clearest availability-first example.
- HBase keeps more structure and stronger row semantics.
- Kafka uses leader-based ordering to keep log semantics manageable.
- HDFS uses synchronous ordered replica mutation for a block, but not a full database transaction model.

## 4. Big Data Processing Systems

## 4.1 MapReduce to Spark: Why the Shift Happened

MapReduce made large-scale batch processing simple and fault tolerant, but it materializes stages heavily and is inefficient for iterative workloads and interactive analytics.

Spark improves this by:

- keeping intermediate data in memory when useful,
- supporting richer operators than just map and reduce,
- building lineage-based fault recovery,
- supporting batch, SQL, ML, graph, and streaming styles in one ecosystem.

Connection to Module 1: HDFS gives Spark the scalable storage substrate. Spark then becomes the compute layer over that storage.

## 4.2 Spark RDD vs Spark DataFrame/SQL

This is the main processing-abstraction comparison in Module 2.

| Dimension | RDD | DataFrame/SQL |
|---|---|---|
| Programming style | Imperative | Declarative |
| Engine visibility into logic | Low, lambdas opaque | High, schema and operators visible |
| Optimization | Mostly manual | Catalyst can optimize |
| Data layout | Row/object oriented | Row and column aware |
| Best use | Arbitrary custom logic | Analytics, ETL, relational-style pipelines |

### Why DataFrames are usually faster

- Spark can understand filters, projections, joins, and aggregations.
- Catalyst can push predicates, prune columns, reorder work, and choose better plans.
- RDDs hide too much logic inside user code.

### Why RDDs still matter

- When you need arbitrary control.
- When the abstraction is not naturally relational.
- When you need to reason directly about partitions and dependencies.

## 4.3 Narrow vs Wide Dependencies in Spark

This is central to Spark performance.

- Narrow dependency: child partition depends on a small fixed set of whole parent partitions. These pipeline well and avoid shuffle.
- Wide dependency: child partition needs pieces from many parent partitions. This causes shuffle, barrier synchronization, disk/network overhead, and more expensive recovery.

Cross-topic link:

- Spark shuffles are to Spark what network-heavy cross-partition communication is to Pregel, PS, or distributed GNN training.
- Any question about scalability should ask: where are the wide dependencies or all-to-all exchanges?

## 4.4 Spark Structured Streaming: What It Really Adds

Structured Streaming models a stream as an unbounded table and incrementally executes DataFrame queries.

What it gives you:

- unified batch and stream API,
- stateless and stateful operations,
- event-time windows,
- checkpoints for recovery,
- append/update/complete output modes,
- integration with sinks like Kafka.

What it does not replace:

- Kafka as an ingestion and replay substrate,
- CEP engines for highly specialized event-pattern syntax,
- HBase or Dynamo as serving stores.

## 4.5 Kafka vs CEP vs Structured Streaming

These three are often confused. They solve different layers of the pipeline.

| System | Main role | Best question it answers |
|---|---|---|
| Kafka | Durable transport and replay | How do producers and consumers communicate at scale? |
| CEP | Event logic and pattern detection | What sequence/window/pattern should trigger an alert? |
| Structured Streaming | Distributed stateful stream analytics | How do we execute a continuous data pipeline at scale with recovery? |

### Kafka vs CEP

- Kafka routes and stores events.
- CEP reasons about event content, time windows, joins, and patterns.
- Kafka topics are a transport abstraction.
- CEP queries are a continuous analytic abstraction.

### CEP vs Structured Streaming

- CEP is often more focused on event patterns, temporal logic, and SQL-like event queries.
- Structured Streaming is a larger distributed processing framework with DataFrame semantics, state stores, and sink integration.
- CEP can be thought of as an operator or query layer inside a larger fast-data architecture.

### Kafka plus Structured Streaming is often the right answer

Kafka absorbs bursty producers and preserves replayable partitions.
Structured Streaming consumes those partitions, performs aggregations or joins, and writes results onward.

## 4.6 Stream Windows: What to Remember

The course distinguishes event-window styles clearly:

- Sliding length window: last N events, trigger for each new event.
- Batch length window: last N events, trigger once every N events.
- Sliding time window: last T time units, trigger repeatedly as time advances.
- Batch time window: collect a time bucket and trigger at bucket boundary.

Cross-topic interpretation:

- Window definitions affect both semantics and runtime cost.
- Sliding windows give fresher outputs but more repeated computation/state maintenance.
- Batch windows reduce trigger frequency but increase result latency.

## 5. Graph and Linked Data Processing

## 5.1 Why Pregel Exists Instead of Just Using Spark

General distributed dataflow systems can process graphs, but they are not ideal for iterative graph algorithms.

The key issue is that many graph algorithms repeatedly propagate information along edges. In Spark this often means repeated joins, groupings, and shuffles of graph structure or messages. Pregel instead keeps graph structure resident and uses iterative supersteps with message passing.

## 5.2 Spark vs Pregel

| Dimension | Spark RDD/DataFrame approach | Pregel/Giraph |
|---|---|---|
| Core abstraction | General distributed collections/tables | Vertex-centric BSP graph model |
| Per-iteration cost | Often joins/groupBy/shuffles | Message passing over fixed graph partitions |
| Graph topology handling | Can be re-materialized repeatedly | Kept in memory across supersteps |
| Generality | High | Specialized |
| Best use | ETL plus mixed analytics | Large iterative graph algorithms |

### Why Pregel can outperform Spark on graph workloads

- Graph structure stays in memory across iterations.
- Message passing follows graph edges directly.
- Synchronization is per superstep, not per relational join plan.
- The model is closer to PageRank, BFS, connected components, and similar algorithms.

### Why Spark is still useful

- Better when graph processing is one stage inside a bigger pipeline.
- Better when you want SQL, ETL, ML, and graph work in one environment.
- Better when the graph is not the only data model in play.

## 5.3 Pregel and GNNs: Similarity and Difference

This is an excellent cross-module connection.

### Similarity

- Both are based on message passing over a graph.
- Both update node-local state using neighbor information.
- Both care deeply about graph partitioning and communication locality.

### Difference

| Aspect | Pregel | GNN |
|---|---|---|
| Goal | Compute graph algorithm result | Learn parameters for predictive tasks |
| Update rule | User-defined algorithmic compute() | Differentiable aggregation/update functions |
| Output | PageRank, components, paths, etc. | Node, edge, or graph predictions |
| Learning | Not central | Central |

Exam summary line: Pregel is graph computation; GNNs are graph learning.

## 6. Machine Learning at Scale

## 6.1 Spark ML vs Deep Learning Frameworks

| Dimension | Spark ML | TensorFlow/PyTorch style frameworks |
|---|---|---|
| Main target | Classical ML pipelines | Deep neural networks |
| Strength | Scalable preprocessing plus standard ML | Flexible neural architectures and GPU training |
| Best fit | Tabular or classical distributed ML | CNNs, transformers, GNNs, large DNNs |

Common exam synthesis:

- Use Spark for large-scale data preparation.
- Hand off to PyTorch/TensorFlow for deep learning.
- This directly links Module 2 and Module 3.

## 6.2 Data Parallelism vs Model Parallelism vs Pipeline Parallelism

| Technique | What is split | Why use it | Limitation |
|---|---|---|---|
| Data parallelism | Training data | Scale throughput when model fits on each worker | Entire model must fit on each worker |
| Model parallelism | Model layers/tensors | Train models too large for one device | Cross-device communication and idle time |
| Pipeline parallelism | Micro-batches across model stages | Improve utilization of model-parallel training | Activation memory and pipeline bubbles |

### Best intuition

- Data parallelism uses many copies of the same model.
- Model parallelism uses one distributed model.
- Pipeline parallelism keeps different parts of the model busy at once.

## 6.3 Parameter Server vs All-Reduce

### Parameter Server

- Workers compute gradients.
- Parameter servers hold sharded parameters.
- Workers push/pull updates.
- Flexible and easy to reason about.

Main weakness: bandwidth into the PS grows with number of workers.

### All-Reduce

- No centralized server.
- Workers collectively aggregate gradients.
- Reduces peak bottleneck from central hot spots.

Main tradeoff: communication becomes more structured and collective.

Exam line: PS gives flexible distributed shared state, but all-reduce scales communication better for tightly synchronized training.

## 6.4 Parameter Server vs Federated Learning

This is one of the strongest compare-and-contrast questions in Module 3.

| Dimension | Parameter Server | Federated Learning |
|---|---|---|
| Deployment setting | Usually data center cluster | Edge devices / wide-area clients |
| Data movement | Data typically already centralized or cluster-local | Raw data stays local |
| What is communicated | Gradients/parameters | Model updates/local models |
| Main objective | Scale training throughput | Preserve privacy and use edge data |
| Main challenge | PS bandwidth and synchronization | Client heterogeneity, privacy, unreliable clients |

Important exam correction: PS does not automatically provide better privacy than FL simply because it exchanges gradients. Gradient leakage is a known issue, and the course slides explicitly warn about it.

## 6.5 Federated Learning vs Cloud Training

Cloud training centralizes data.
Federated learning centralizes only model aggregation.

Why FL exists:

- privacy,
- regulation,
- edge-local data generation,
- wide-area deployment.

Why it is harder:

- clients differ in hardware,
- clients may join or drop out,
- local data distributions are non-IID,
- communication is expensive and intermittent.

## 6.6 LLM Inference: Prefill vs Decode

This is a crucial modern Module 3 serving distinction.

| Phase | What happens | Bottleneck |
|---|---|---|
| Prefill | Process prompt tokens together, build KV cache | More compute-bound |
| Decode | Generate one token at a time using existing KV cache | More memory-bandwidth-bound |

Why this matters:

- TTFT is dominated by prompt processing and queueing.
- TPOT is dominated by repeated KV cache access and decode scheduling.
- This is why LLM serving needs different system design than ordinary batch inference.

## 6.7 ORCA's Core Insight

Traditional serving systems batch whole requests. ORCA instead schedules at iteration granularity.

Why that helps:

- short requests do not wait behind long ones as badly,
- finished requests can leave sooner,
- new requests can join between iterations,
- GPU utilization improves without as much head-of-line blocking.

Cross-topic link: ORCA does for generative serving what better scheduling and state management do in streaming systems. It changes the granularity at which the system makes decisions.

## 7. End-to-End Architecture Patterns Across Modules

## 7.1 Large-Scale Training Data Pipeline

Typical good stack:

- Kafka for live ingestion of logs/events.
- HDFS for raw durable corpus and large training shards.
- Spark DataFrames for cleaning, filtering, deduplication, and feature generation.
- HBase for metadata, sample lookup, or partial-field retrieval if needed.
- PyTorch distributed or similar for deep model training.

Why this is coherent:

- Kafka handles continuous arrival.
- HDFS handles durable high-throughput storage.
- Spark handles scalable transformation.
- Training frameworks handle GPU-scale model optimization.

## 7.2 Smart City / IoT Fast Data Pipeline

Good layered answer from course material:

- Sensors and edge devices produce events.
- Kafka or another broker decouples producers from analytics.
- CEP identifies threshold, pattern, and temporal events.
- Structured Streaming computes aggregates and joins continuously.
- HBase archives operational data and supports lookups/dashboards.

Why this is good:

- fast data path for alerts,
- durable event history,
- scalable processing,
- online serving of current or historical state.

## 7.3 Online Recommender or Personalization Stack

- Kafka for clicks, views, and session events.
- HDFS for offline corpus and feature history.
- Spark for batch feature engineering and retraining.
- Dynamo-style store when write availability and key-based access dominate.
- HBase when profile or feature access needs row/column structure and scans.
- Optional graph layer using Pregel or GNN if relationships matter strongly.

## 7.4 Graph-Aware ML Pipeline

- Kafka ingests interactions.
- HDFS stores historical edges and node features.
- Spark performs ETL and snapshot generation.
- Pregel runs graph algorithms like PageRank or components.
- Distributed GNN training learns predictive embeddings.

Best exam synthesis line: Pregel computes graph structure-derived signals; GNNs learn from graph structure and features.

## 7.5 GenAI Company Stack

A strong modular answer usually looks like this:

- HDFS or a data-lake-style storage layer for raw corpora and checkpoints.
- Kafka for ingestion of user telemetry, prompts, feedback, and pipeline events.
- Spark for data cleaning, deduplication, quality filtering, and analytics.
- HBase for low-latency metadata/feature lookup if row-keyed access matters.
- Dynamo-like system if the most critical workload is always-writeable key-value serving.
- Distributed DNN training with data/model/pipeline parallelism.
- ORCA-like serving for generative inference.
- Structured Streaming for online metrics and drift monitoring.

## 8. Fast Memory Hooks for the Final

Use these short thesis lines in answers when you need to quickly connect modules.

- HDFS is a throughput-first data lake substrate; HBase is its low-latency structured-access companion.
- Kafka transports streams; CEP interprets event patterns; Structured Streaming scales stateful continuous analytics.
- Spark is a general compute fabric; Pregel is a specialized graph-compute fabric.
- DataFrames beat RDDs when the engine can see the schema and optimize the plan.
- Dynamo chooses availability first; HBase chooses structure and stronger row semantics.
- Parameter Server centralizes model state; federated learning avoids centralizing data.
- Data parallelism scales throughput, model parallelism scales capacity, pipeline parallelism scales utilization.
- LLM serving is not just inference at scale; decode is memory-bound and scheduler-dominated.

## 9. Common Exam Pitfalls

- Do not confuse CAP consistency with ACID consistency.
- Do not call Kafka a full analytics engine.
- Do not treat HDFS as a low-latency random-access store.
- Do not treat HBase and Dynamo as interchangeable just because both are NoSQL.
- Do not say Spark and Pregel are equivalent for iterative graph workloads.
- Do not claim parameter server is inherently private.
- Do not describe federated learning as merely parameter server over the internet; the privacy and data-locality goals are different.
- Do not forget that row-key design is central in HBase.
- Do not forget that shuffles dominate Spark cost just as communication dominates many distributed ML and graph workloads.

## 10. Final Summary

If you have to reduce the whole course to one sentence:

Choose the system whose abstraction matches the workload, whose partitioning gives the right parallelism, whose consistency model is strong enough but not stronger than necessary, and whose main bottleneck matches the performance goal you care about.