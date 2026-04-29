# SSDS Final Open-Ended Question Bank

This file is a large bank of exam-style open-ended questions and model answers built from Modules 1 to 5.

How to use this bank:
- Treat each answer as a point-form model answer, not as a script to memorize word-for-word.
- For most SSDS essays, structure your answer in this order: workload, abstraction, partitioning, replication or consistency, bottleneck, tradeoff, recommendation.
- The tags in each title show the main modules covered.

## Part A: Comparison Questions

### CQ01. [M1] Compare a traditional local file system and a distributed file system.
**Model answer**
- A traditional local file system is designed for one machine, low-latency local I/O, and strong control over one set of disks and metadata.
- A distributed file system spreads data across multiple machines so that many clients can share data, aggregate bandwidth, scale capacity, and survive machine failures.
- The local file system usually gives simpler consistency and lower latency, but it is limited by one machine's disk, RAM, and fault domain.
- The distributed file system gains scale and reliability through partitioning and replication, but it pays for that with network cost, metadata coordination, and more complex failure recovery.
- In this course, GFS and HDFS are cluster-style DFS designs optimized for large shared datasets, sequential access, and routine failures, not for tiny low-latency local updates.

### CQ02. [M1] Compare GFS and HDFS.
**Model answer**
- GFS and HDFS are very similar in spirit: both use a master-worker architecture, large chunks or blocks, replication, and throughput-first design for large read-heavy and append-heavy workloads.
- GFS uses the terms master and chunkserver, while HDFS uses NameNode and DataNode, but the control path and data path are conceptually the same.
- GFS emphasizes record append and snapshot as explicit design features, while HDFS lecture material highlights block reports, heartbeats, checkpoint nodes, and backup nodes around NameNode management.
- Both systems assume failures are normal, small files are a problem, and applications care more about high sustained bandwidth than about very low latency.
- The best high-level exam summary is that HDFS is the open Hadoop implementation inspired by the same core design assumptions that originally made GFS successful.

### CQ03. [M1] Compare the large-block design of GFS and HDFS with the small-block design of conventional file systems.
**Model answer**
- Conventional file systems use small blocks because they target mixed workloads with many small files and random updates, so they optimize space efficiency and fine-grained access.
- GFS and HDFS use very large chunks or blocks because their dominant workloads are large files, sequential scans, and large streaming reads and writes.
- Large blocks reduce metadata pressure, reduce master lookups, allow long sequential transfers, and help amortize network and disk overhead.
- The tradeoff is that small files become inefficient, hotspots are more likely when many clients hit one block, and fine-grained updates are not a strength.
- So large blocks are not a universal improvement; they are correct only because the systems are tuned to big-data batch and AI-style workloads rather than desktop-style storage.

### CQ04. [M1] Compare write, append, and record append in GFS or HDFS-style systems.
**Model answer**
- A write updates data at a client-specified offset, so the client decides where the bytes should go.
- A normal append extends the file at the end, but conceptually the client still wants end-of-file semantics rather than a random overwrite.
- Record append is more specialized: the system chooses the final offset and guarantees that a record is appended atomically at least once, even with concurrent writers.
- Record append is useful for merged logs and producer-heavy pipelines because it avoids forcing clients to coordinate exact offsets themselves.
- The main tradeoff is semantic complexity: record append can create padding or duplicate records after retries, so applications must tolerate at-least-once behavior.

### CQ05. [M1] Compare a checkpoint node and a backup node in HDFS.
**Model answer**
- A checkpoint node periodically merges the namespace image and edit log so that NameNode restart does not require replaying an excessively long journal.
- A backup node keeps an up-to-date in-memory copy of the namespace while also being able to create checkpoints.
- Both exist to reduce control-plane fragility around the NameNode, but the backup node is closer to a continuously synchronized helper, while the checkpoint node is more periodic maintenance support.
- Neither should be described as the main data plane for file reads or writes; they are metadata-management roles.
- The right exam angle is that these nodes reduce metadata recovery cost, not that they eliminate the deeper centralized-namespace issue entirely.

### CQ06. [M1, M5] Compare HDFS and Kafka as partitioned, replicated systems.
**Model answer**
- Both HDFS and Kafka partition data across machines and replicate partitions or blocks so the system can scale and survive failures.
- Both also use a single ordering point for writes to preserve a coherent write order: a primary replica for HDFS block mutation and a leader broker for Kafka partition append.
- HDFS is a distributed file system for storing large files and serving large scans, while Kafka is a distributed event log and pub-sub backbone for ordered event streams.
- HDFS is optimized for durable bulk storage and file-style access, whereas Kafka is optimized for continuous ingestion, replay, consumer groups, and offset-based consumption.
- A strong exam conclusion is that HDFS stores the dataset itself, while Kafka stores and transports the event history that feeds downstream systems.

### CQ07. [M1, M4] Compare HDFS and HBase for storing large ML training data.
**Model answer**
- HDFS is better when the dominant pattern is full or near-full scans over a large immutable dataset, which matches many distributed training workloads.
- HBase is better when the workload needs keyed lookup, range scans by row key, partial-column access, or updates to individual records.
- HDFS is simpler and cheaper for bulk training shards, checkpoints, and corpus storage, but it is poor for low-latency lookup of individual samples.
- HBase adds structure and faster row-level access on top of HDFS, but row-key design becomes a central performance decision.
- A good exam answer says the two often coexist: HDFS for the raw corpus and HBase for metadata, sample indexes, or online feature access.

### CQ08. [M2] Compare MapReduce and Spark.
**Model answer**
- MapReduce gives a simple map-shuffle-reduce pattern with strong fault tolerance, but it materializes intermediate state heavily and is awkward for multi-stage and iterative workloads.
- Spark generalizes the programming model, keeps more intermediate data in memory, and uses lineage to recompute lost partitions instead of writing every stage back to disk.
- MapReduce is a better mental model for simple large batch jobs; Spark is better for iterative analytics, interactive exploration, SQL, ML pipelines, and mixed workloads.
- Both still suffer when a workload requires heavy shuffle, because network and disk movement remain expensive.
- The cleanest exam summary is that Spark reduces repeated disk I/O and broadens the compute model, while MapReduce is more rigid and stage-heavy.

### CQ09. [M2] Compare RDDs and DataFrames or Spark SQL.
**Model answer**
- RDDs give the programmer low-level control and can express arbitrary logic, but Spark sees user functions as opaque and therefore cannot optimize much.
- DataFrames and Spark SQL expose schema and operator meaning, so Catalyst can reason about filters, projections, joins, and aggregations.
- Because the engine can see the structure of a DataFrame query, it can push predicates, prune columns, choose better join strategies, and generate more efficient physical plans.
- RDDs are still useful when the logic is highly custom or not naturally relational, but for standard analytics they are usually slower and more verbose.
- So DataFrames are preferred for relational or tabular workloads, while RDDs are the escape hatch for custom distributed logic.

### CQ10. [M2] Compare a logical plan and a physical plan in Spark.
**Model answer**
- The logical plan captures what the computation means: which RDDs or operators depend on which others, and what transformations are requested.
- The physical plan captures how Spark will execute the work: stages, shuffle boundaries, tasks, and concrete operator implementations.
- In RDD terms, the logical plan is the dependency DAG; in DataFrame terms it is the parsed and optimized query tree.
- The physical plan is where Spark cuts the DAG at wide dependencies and turns each stage into tasks running on executors.
- A useful exam line is that the logical plan explains semantics and optimization opportunities, while the physical plan explains scheduling cost and runtime behavior.

### CQ11. [M2] Compare narrow and wide dependencies in Spark.
**Model answer**
- A narrow dependency means each child partition depends on a small fixed number of full parent partitions, so execution can pipeline efficiently without a shuffle.
- A wide dependency means child partitions need data from many parent partitions, so Spark must reshuffle data across the network.
- Narrow dependencies are cheaper, easier to recover from, and often stay within one stage.
- Wide dependencies create stage boundaries, network traffic, disk spill, and barrier-like waiting because reducers depend on many mappers.
- This is one of the most important exam concepts because performance tuning in Spark often reduces to minimizing unnecessary wide dependencies.

### CQ12. [M2] Compare `reduceByKey`, `groupByKey`, and `combineByKey`.
**Model answer**
- `groupByKey` gathers all values for a key and is therefore expensive in both memory and network traffic; it should be used only when the full value list is actually needed.
- `reduceByKey` performs incremental aggregation and supports map-side combining, so it is the best default for associative and commutative reductions like sum, count, min, or max.
- `combineByKey` is the most general of the three because it allows a different accumulator type from the input value type, which is why it is used for averages, variances, and other custom aggregations.
- The common exam mistake is to use `groupByKey` for sums or counts; that is slower and less scalable than `reduceByKey`.
- The clean summary is: `groupByKey` for all-values semantics, `reduceByKey` for simple reductions, and `combineByKey` for flexible custom per-key aggregation.

### CQ13. [M2, M1] Compare lineage-based recovery in Spark with replication-based recovery in HDFS.
**Model answer**
- Spark fault tolerance for RDDs is based on lineage: it remembers how a partition was derived and recomputes that partition if it is lost.
- HDFS fault tolerance for base data is based on replication: multiple copies of each block exist ahead of time so the system can keep serving data even after failures.
- Lineage saves storage and is elegant for deterministic intermediate results, but recomputation can be expensive if the dependency chain is long.
- Replication gives faster recovery and better read availability for durable storage, but it consumes extra storage space and network bandwidth continuously.
- That is why Spark uses lineage for intermediate compute state while HDFS uses replication for persistent data blocks.

### CQ14. [M2] Compare hash shuffle and sort shuffle in Spark.
**Model answer**
- Hash shuffle partitions map output by reducer using hashing, but older hash-shuffle implementations could create many small files and high file-management overhead.
- Sort shuffle sorts or groups map output before writing it, which reduces file explosion and generally improves disk efficiency.
- Both still pay the core shuffle cost of network movement, spill, and synchronization across stages.
- The important exam point is not memorizing implementation history but understanding that shuffle is expensive because data must cross partition boundaries.
- Sort shuffle became the preferred design because it manages intermediate data more efficiently than naive hash-based output layouts.

### CQ15. [M3] Compare Spark ML, Scikit-Learn, and PyTorch.
**Model answer**
- Scikit-Learn is strongest for classical ML experimentation on a single machine and provides a wide algorithm toolbox for small to medium datasets.
- Spark ML is strongest when the data already lives in a distributed Spark pipeline and the workload is classical ML plus large-scale feature engineering.
- PyTorch is the right tool for deep learning, especially DNNs, transformers, GNNs, and distributed GPU training.
- Spark ML is not the right abstraction for large transformer training, and PyTorch is not the best abstraction for large-scale ETL.
- A strong exam answer says Scikit-Learn is for exploration, Spark ML is for distributed classical ML pipelines, and PyTorch is for modern deep learning.

### CQ16. [M3] Compare parameter server and ring all-reduce for distributed training.
**Model answer**
- In a parameter-server design, workers push gradients or parameter updates to centralized or sharded servers that aggregate updates and send back model state.
- In ring all-reduce, workers exchange updates directly with one another in a collective pattern, avoiding a central aggregation hotspot.
- Parameter servers are flexible and can work well with asynchronous updates or sparse models, but the servers can become bandwidth bottlenecks as worker count grows.
- Ring all-reduce is excellent for dense synchronous GPU training because no single node becomes the central bottleneck, but it is less flexible for heterogeneity and asynchronous behavior.
- So the usual tradeoff is flexibility and central control versus better communication scaling for dense synchronous workloads.

### CQ17. [M3] Compare data parallelism, model parallelism, and pipeline parallelism.
**Model answer**
- Data parallelism replicates the full model on each worker and splits the training data across workers, which is simple and scales throughput if the model fits in device memory.
- Model parallelism splits the model itself across devices, which is necessary when the model is too large to fit on one device.
- Pipeline parallelism improves utilization in model-parallel settings by overlapping different micro-batches across stages instead of leaving most devices idle.
- Data parallelism is the easiest to use, model parallelism solves memory capacity limits, and pipeline parallelism solves utilization problems created by model parallelism.
- Modern large-model training often uses all three together because each solves a different bottleneck.

### CQ18. [M3] Compare parameter-server training and federated learning.
**Model answer**
- Parameter-server training is meant for centrally managed clusters where data shards are already inside the data center and the goal is high-throughput distributed optimization.
- Federated learning is meant for settings where raw data must remain on client devices or across organizations, so only model updates are sent to a central aggregator.
- Parameter-server training assumes relatively stable high-bandwidth networks and centrally coordinated infrastructure.
- Federated learning must handle slow devices, dropouts, non-IID data, privacy constraints, and wide-area communication.
- The best summary is that parameter servers solve distributed training throughput inside the cluster, while federated learning solves training with decentralized data and privacy constraints.

### CQ19. [M3] Compare cross-device and cross-silo federated learning.
**Model answer**
- Cross-device federated learning involves a huge number of edge devices such as phones, where clients are unreliable, availability is intermittent, and each client may have only a small amount of data.
- Cross-silo federated learning involves a smaller number of reliable organizations or institutions, each with a larger local dataset and more stable compute resources.
- Cross-device FL puts more emphasis on client selection, secure aggregation at massive scale, and handling dropouts.
- Cross-silo FL puts more emphasis on organizational trust boundaries, legal constraints, and possibly richer collaborative modeling across institutions.
- So the difference is not just scale; it is also the operational environment and failure model of the participating clients.

### CQ20. [M3] Compare horizontal and vertical federated learning.
**Model answer**
- Horizontal federated learning means different participants have similar feature spaces but different sets of records, so partitioning is mostly by rows.
- Vertical federated learning means participants hold different feature sets about overlapping entities, so partitioning is mostly by columns.
- Horizontal FL is easier to picture as distributed data-parallel training across clients with different user subsets.
- Vertical FL is harder operationally because parties must align shared entities and combine feature spaces without exposing raw data.
- A strong exam answer says HFL splits examples across participants, while VFL splits attributes across participants.

### CQ21. [M3] Compare distributed DNN training and distributed GNN training.
**Model answer**
- Distributed DNN training mostly worries about partitioning data and model parameters, then synchronizing gradients efficiently.
- Distributed GNN training has those problems too, but it also has graph-specific dependence across partitions because a node's computation depends on neighbors that may live elsewhere.
- The main GNN-specific bottleneck is neighborhood explosion and the need to sample neighbors and fetch remote features during training.
- DNN model size often forces model and pipeline parallelism, while many GNN models are smaller and fit on one machine, so graph and data partitioning dominate instead.
- The clean exam summary is that GNN training is harder because graph structure creates communication during sampling and feature access, not just during gradient synchronization.

### CQ22. [M3, M5] Compare GNN message passing and Pregel message passing.
**Model answer**
- Both use local neighborhood communication on a graph, which is why the comparison is so useful.
- In a GNN, message passing is differentiable and is used to learn node or graph embeddings for predictive tasks.
- In Pregel, message passing is algorithmic and is used to compute graph properties such as PageRank, shortest path, or connected components.
- GNN training requires feature sampling, forward and backward passes, and gradient synchronization, while Pregel focuses on repeated supersteps of compute, send-message, and synchronize.
- So the structural similarity is neighborhood communication, but the purpose differs: learning representations versus computing explicit graph algorithms.

### CQ23. [M3] Compare prefill and decode in transformer inference.
**Model answer**
- Prefill processes the input prompt and builds the initial KV cache, so it handles many prompt tokens at once and is closely tied to time-to-first-token.
- Decode generates output one token at a time, repeatedly consulting the KV cache, so it is usually more sensitive to memory bandwidth and cache management.
- Prefill is easier to batch because many prompt tokens can be processed in parallel.
- Decode is harder to batch efficiently because requests have different lengths and can finish at different times.
- A strong exam answer says prefill is prompt-parallel and compute-heavy, while decode is iterative and often memory-bound.

### CQ24. [M3] Compare conventional request-level batching with ORCA's iteration-level scheduling.
**Model answer**
- Conventional request-level batching groups full requests together and keeps them in the same batch until the batch boundary changes, which creates head-of-line blocking when requests have different lengths.
- ORCA schedules at the granularity of one iteration or one forward step, so finished requests can leave immediately and new requests can join quickly.
- ORCA also uses selective batching: non-attention operations are batched aggressively, while attention is handled in a way that respects token-position differences.
- This eliminates the early-finished and late-joining inefficiencies that waste GPU time in request-level scheduling.
- The best exam conclusion is that ORCA improves both latency and throughput by matching the scheduler to the iterative nature of generative decoding.

### CQ25. [M4] Compare ACID and BASE.
**Model answer**
- ACID emphasizes atomicity, consistency, isolation, and durability, which simplifies application reasoning but usually requires coordination and therefore limits scale.
- BASE emphasizes basically available, soft state, and eventual consistency, which improves availability and scale but pushes more reconciliation burden onto the application or background processes.
- ACID is the right fit when correctness and transactional invariants dominate, such as banking or inventory updates with strict guarantees.
- BASE is attractive when responsiveness and uptime matter more than immediate global agreement, such as caches, feeds, or highly available key-value services.
- The exam-quality conclusion is that BASE is not "better" than ACID; it is a deliberate exchange of coordination for scale and availability.

### CQ26. [M4] Compare CA, CP, and AP in the CAP framework.
**Model answer**
- CA means consistent and available only if you effectively ignore partitions, which is realistic mainly in single-site or tightly controlled systems.
- CP systems preserve strong consistency during a partition by making some operations unavailable if they cannot reach the needed quorum.
- AP systems continue responding during a partition, but they may return stale data or create conflicting versions that must be repaired later.
- In real distributed systems, partition tolerance is not optional, so the practical tradeoff is usually consistency versus availability during failures or high latency.
- A mature answer also notes Brewer's later point that consistency and availability are spectra and that network latency can force the same tradeoff even before a clean partition occurs.

### CQ27. [M4] Compare eager replication and lazy replication.
**Model answer**
- Eager replication updates replicas before a write is considered committed, so it gives stronger consistency but requires more coordination and higher latency.
- Lazy replication commits first and propagates changes later, so it improves availability and write latency but allows stale reads and conflict scenarios.
- Eager group replication can become disastrously hard to scale because coordination and deadlock costs grow with system size.
- Lazy replication fits systems where temporary inconsistency is acceptable and background reconciliation is cheaper than blocking user requests.
- The exam answer should always connect the replication style to the workload's tolerance for stale data and failed writes.

### CQ28. [M4] Compare HBase and Dynamo.
**Model answer**
- HBase is a wide-column store built for row-key lookup, range scans, sparse columns, and structured access on top of HDFS.
- Dynamo is a highly available key-value store built for simple `get` and `put` semantics, always-writeable behavior, and eventual consistency.
- HBase uses range-style partitioning through regions and offers stronger row-level structure, while Dynamo uses consistent hashing with virtual nodes and is optimized for elastic key-based access.
- HBase is better for sparse structured records and partial-column access; Dynamo is better for session-like or cart-like objects where write availability matters most.
- A strong answer says HBase optimizes structure and scans, while Dynamo optimizes availability and partition-tolerant key-value service.

### CQ29. [M4] Compare a data warehouse and a data lake.
**Model answer**
- A traditional data warehouse usually imposes stronger schema decisions and ETL before data lands in the warehouse, so the storage is curated and more tightly modeled for analytics.
- A data lake stores raw and semi-processed data more flexibly and lets many downstream consumers interpret or transform it later.
- Warehouses are better when the organization already knows the reporting schema and wants curated structured analytics.
- Lakes are better when data sources are diverse, producers and consumers should be decoupled, and multiple downstream pipelines need access to the same raw assets.
- In this course, HDFS-like storage plus Spark processing is the natural substrate of a data-lake architecture.

### CQ30. [M4, M5] Compare graph databases and Pregel-style graph processing systems.
**Model answer**
- A graph database such as Neo4j is optimized for online graph queries, transactional traversals, and interactive subgraph exploration.
- Pregel is optimized for large-scale iterative graph analytics where the whole graph is processed in bulk supersteps.
- Graph databases are query-centric and low-latency, while Pregel is computation-centric and throughput-oriented.
- If the workload is "find a small path or neighborhood right now," a graph database is usually better; if the workload is "run PageRank or connected components over billions of edges," Pregel is usually better.
- The best exam ending is that the two systems solve different graph problems and often complement one another.

### CQ31. [M5] Compare publish-subscribe systems and CEP systems.
**Model answer**
- A publish-subscribe system focuses on moving events from producers to consumers through topics or channels while keeping the two sides decoupled.
- A CEP system focuses on interpreting event content over time using filters, windows, joins, sequences, and patterns.
- PubSub answers the question "how do events move reliably and scalably?" while CEP answers the question "what derived events or alerts should we compute from them?"
- PubSub often stores or routes the raw event history, while CEP continuously evaluates logic over that event history.
- So PubSub is transport and decoupling, whereas CEP is continuous event reasoning.

### CQ32. [M5] Compare Kafka and Spark Structured Streaming.
**Model answer**
- Kafka is a partitioned, replicated event log and broker, so it excels at ingestion, buffering, replay, and decoupling producers from consumers.
- Structured Streaming is a distributed stream-processing engine that executes incremental DataFrame-style queries over unbounded data.
- Kafka stores and transports the stream, while Structured Streaming computes windows, joins, aggregations, and stateful analytics over that stream.
- Kafka scales through topic partitions and broker replication; Structured Streaming scales through Spark partitions, state partitioning, and distributed task execution.
- The best exam summary is that Kafka is the durable fast-data backbone and Structured Streaming is the scalable compute layer on top of that backbone.

### CQ33. [M5] Compare DStreams and Structured Streaming.
**Model answer**
- DStreams model a stream as a sequence of micro-batched RDDs over time.
- Structured Streaming models a stream as an unbounded table and reuses the DataFrame and SQL abstraction for both batch and streaming.
- DStreams were powerful because they unified stream processing with the Spark RDD engine, but they exposed a lower-level streaming abstraction.
- Structured Streaming is cleaner because it reuses DataFrame semantics, the optimizer, and output modes in a more declarative way.
- The concise exam answer is that Structured Streaming is the higher-level, more unified evolution of Spark's earlier DStream model.

### CQ34. [M5] Compare sliding windows and batch windows in stream processing.
**Model answer**
- Sliding windows overlap, so they produce fresher and more frequent results over recent data.
- Batch windows do not overlap, so they produce simpler periodic summaries with less repeated computation.
- Sliding windows are better for rolling anomaly detection, live dashboards, and near-real-time monitoring.
- Batch windows are better for periodic reports or when lower computation cost matters more than immediate freshness.
- The correct exam point is that the choice changes both the semantics of the answer and the runtime cost of the system.

### CQ35. [M5] Compare one-window joins and two-window joins on streams.
**Model answer**
- A one-window join uses the current event from one stream and matches it against a window of recent events from the other stream.
- A two-window join keeps windows on both streams and joins events that coexist in the active windows of both sides.
- Two-window joins are more general but require more state and can produce many more output pairs.
- One-window joins are cheaper and often appropriate when one side acts as the trigger stream.
- The key exam idea is that windows are required because streams are unbounded, and the chosen join semantics determine both correctness and cost.

### CQ36. [M5] Compare Pregel and Spark or MapReduce for graph processing.
**Model answer**
- Spark and MapReduce are general-purpose dataflow systems, so graph algorithms often become repeated joins, groupings, or shuffles over edge and vertex state.
- Pregel is graph-specific and keeps the graph structure resident across supersteps while vertices exchange messages directly along edges.
- This avoids repeatedly reconstructing or reshuffling the graph topology every iteration, which is why Pregel fits iterative algorithms like PageRank, BFS, or shortest path so naturally.
- Spark is still better when graph processing is only one stage inside a broader ETL or ML pipeline.
- The correct exam conclusion is that Pregel wins on graph-native iterative workloads, while Spark wins on generality and ecosystem integration.

### CQ37. [M5] Compare synchronous BSP graph processing and asynchronous graph processing.
**Model answer**
- Synchronous BSP groups work into supersteps with a barrier between rounds, which makes reasoning, fault tolerance, and deterministic behavior much simpler.
- Asynchronous graph processing allows updates to propagate immediately without waiting for a global barrier, which can reduce idle time and sometimes converge faster.
- The downside of BSP is barrier waiting and sensitivity to stragglers; the downside of asynchronous execution is more complex coordination and harder reasoning about consistency.
- Pregel deliberately chooses synchronous BSP because the model is easier to program and scale robustly.
- So the comparison is clean: BSP favors simplicity and discipline, while asynchronous systems favor flexibility and sometimes speed.

## Part B: Architect Questions

### AQ01. [M1, M3] If you had to redesign HDFS today for AI and ML workloads, what would you change?
**Model answer**
- Workload: assume huge immutable corpora, repeated multi-epoch scans, many concurrent readers, large checkpoint writes, and strong need for reproducibility.
- Storage design: keep large sequential shards as the core abstraction, but strongly discourage tiny files by adding better sharding, sample indexing, and small-file consolidation.
- Placement: use hot and cold tiers so active training shards and current checkpoints live on SSD or NVMe while colder data uses cheaper capacity tiers.
- Metadata: reduce central namespace bottlenecks with better metadata scaling and snapshot or version support for dataset reproducibility.
- Scheduling: integrate storage placement with GPU-aware schedulers and local caches so training jobs read from nearby data and do not starve accelerators.
- Reliability: retain replication for hot data, consider erasure coding for colder data, and optimize checkpoint throughput because long training jobs cannot afford slow recovery.
- Final justification: the redesigned system should remain throughput-first like HDFS, but it must add better metadata scaling, versioning, caching, and tiering for modern AI pipelines.

### AQ02. [M1, M2, M3, M4, M5] You are the chief architect of a new GenAI company. Design the end-to-end data and model platform.
**Model answer**
- Ingestion: use Kafka for prompts, user events, tool traces, telemetry, and document-ingest pipelines because it decouples producers from all downstream consumers.
- Durable storage: use HDFS or a data-lake substrate for raw corpora, cleaned corpora, tokenized shards, embeddings, and checkpoints because the dominant workload is bulk scans and durable versioned storage.
- Processing: use Spark batch for ETL, deduplication, filtering, and feature generation, and use Structured Streaming for near-real-time monitoring, quality alerts, and drift metrics.
- Online data: use HBase for keyed metadata or partial-column retrieval, and use Dynamo-style storage only for always-writeable simple key-value state such as sessions or transient personalization state.
- Training: use Spark for data engineering but use PyTorch-style distributed training for deep models, choosing data, model, and pipeline parallelism as model size requires.
- Serving: use an ORCA-like serving layer for large generative models because prefill and decode need scheduler-aware batching and KV-cache management.
- Why this stack: each system is chosen for its own bottleneck instead of forcing one platform to do ingestion, storage, training, and online serving badly.

### AQ03. [M1, M2, M5] Design a web-search or document-search pipeline using the systems from the course.
**Model answer**
- Raw crawl data should land in HDFS because web pages and parsed text are large bulk datasets that are naturally processed in parallel.
- Use Spark to parse documents, tokenize text, remove stop words, and build the inverted index in a scalable batch pipeline.
- Build the web graph from hyperlinks and compute graph scores such as PageRank; if the graph dominates the workload, a Pregel-style engine is better than repeated Spark joins.
- Kafka can ingest fresh crawl updates or click logs so ranking and analytics pipelines can replay the same stream independently.
- Use HBase or another keyed store for serving metadata such as document titles, snippets, and document IDs if low-latency lookup is needed.
- Keep the search stack layered: lake for corpus, Spark or Pregel for offline computation, and a serving layer for low-latency query-time retrieval.
- The design rationale is that indexing and graph computation are throughput-heavy batch workloads, while query serving is a separate low-latency problem.

### AQ04. [M5, M4, M2] Design a fast-data architecture for a smart-campus or IoT monitoring application.
**Model answer**
- Put a Kafka-like broker at the center so sensors and devices can publish events without knowing which analytics components consume them.
- Use CEP for threshold, sequence, and pattern rules such as overflow alerts, power spikes, or device-failure sequences.
- Use Structured Streaming for rolling aggregates, joins across streams, dashboards, and online feature computation over windows.
- Store latest operational state and device metadata in HBase because dashboards and control systems often need keyed access to recent values.
- Archive raw streams to HDFS for offline analytics, historical audits, and training future models.
- Scale comes from data parallelism at the broker, task and data parallelism in the stream processor, and separate storage for hot online state versus cold history.
- This layered design works because no single system is forced to do transport, event logic, analytics, and history at the same time.

### AQ05. [M2, M3, M4, M5] Design a recommendation or personalization platform using course systems.
**Model answer**
- Use Kafka to ingest clicks, views, searches, purchases, and impression events because multiple downstream teams will want to consume the same behavioral stream.
- Persist the full event history in HDFS so offline training and backfills can scan the corpus repeatedly.
- Use Spark to build sessions, features, labels, and training tables from that history.
- Use HBase when online serving needs structured profiles or sparse feature vectors, and use Dynamo-style storage when ultra-high availability for simple key-value user state matters more than stronger structure.
- If graph structure matters, compute offline graph features with Pregel or train a GNN pipeline using distributed GNN tooling.
- Use Structured Streaming for near-real-time counters, freshness features, and live monitoring of drift and business KPIs.
- The right architecture separates stream ingestion, batch training, and low-latency serving because each has a different access pattern and bottleneck.

### AQ06. [M3, M5, M1] Design a privacy-sensitive mobile AI application that learns from user behavior.
**Model answer**
- Keep raw personal data on devices and use federated learning so only model updates, not raw logs, are sent to the cloud.
- Use secure aggregation so the server learns only aggregated updates rather than one device's update directly.
- Prefer cross-device FL techniques such as client selection, round management, and pace steering because device availability and bandwidth are highly variable.
- Use Kafka and Structured Streaming only for non-sensitive operational telemetry, rollout metrics, and aggregate system health.
- Store public or sanitized training corpora, global model versions, and checkpoints in HDFS.
- Use a small keyed metadata store for model versions, configuration, and device enrollment, but do not centralize user raw data unless the privacy model explicitly permits it.
- The main architectural choice is trading simpler centralized training for privacy, legal compliance, and on-device personalization.

### AQ07. [M3, M1] Design a large-scale distributed GNN training stack.
**Model answer**
- Store graph snapshots, node features, and training labels in HDFS because the graph assets are large and must be versioned for reproducible training.
- Partition the graph offline so each worker owns a subgraph, but expect cross-partition edges and plan for remote feature fetches.
- Use a DistDGL-style pipeline with separate phases for sampling, feature fetching, forward pass, and backward pass.
- Use data-parallel training across graph partitions because many GNN models fit on one machine; the harder problem is graph communication, not oversized model parameters.
- Expect neighborhood explosion to dominate, so neighborhood sampling is essential to keep training cost bounded.
- Use ring all-reduce or a similar collective for gradient aggregation, but optimize remote feature fetch first because that is often the real bottleneck.
- The strongest rationale is that distributed GNN training must manage both ML-style gradient flow and graph-style cross-partition data dependence.

### AQ08. [M3] Design an LLM serving platform for long and variable-length requests.
**Model answer**
- Separate the performance model into prefill and decode because they stress hardware differently and should not be scheduled as if they were the same phase.
- Use an ORCA-like scheduler with iteration-level scheduling so early-finished requests can leave immediately and late arrivals can join quickly.
- Use selective batching so non-attention operations benefit from batching while attention remains compatible with per-request token positions.
- Combine model parallelism and pipeline parallelism across GPUs because large generative models usually do not fit cleanly on a single device.
- Manage KV cache carefully because decode is often limited by memory bandwidth and cache growth rather than raw arithmetic throughput.
- Keep model weights and checkpoints in durable bulk storage, but keep the online serving path isolated from slow bulk-storage reads through warm model placement and caching.
- The design goal is not just high throughput; it is high throughput at acceptable time-to-first-token and per-token latency.

### AQ09. [M5, M2, M4] Design a streaming anomaly-detection platform for logs, metrics, or sensors.
**Model answer**
- Use Kafka as the replayable ingestion layer so multiple anomaly detectors, dashboards, and archival jobs can consume the same event stream.
- Use CEP for simple sequence and threshold patterns such as repeated failures, spikes followed by drops, or safety-rule violations.
- Use Structured Streaming for rolling aggregates, joins, model scoring, and event-time windows over the incoming data.
- Keep recent keyed entity state in HBase so detectors can quickly read and update the latest state for users, devices, or services.
- Archive the raw stream in HDFS so you can retrain models, audit incidents, and replay historical scenarios.
- Use checkpoints and offset management so failed streaming jobs can recover without losing their place in the stream.
- The overall architecture separates durable transport from streaming compute and from low-latency keyed state.

### AQ10. [M4, M2, M1] Design an enterprise data-lake platform for many analytics teams.
**Model answer**
- Use HDFS-like storage as the raw landing zone because it scales well for diverse formats and large volumes.
- Separate the lake into raw, cleaned, curated, and feature-ready zones so different consumers can work at different trust levels.
- Use Kafka for continuously arriving source systems and Spark for both batch ETL and streaming transformations.
- Add strong metadata management, schema tracking, and dataset versioning so the flexibility of a lake does not turn into chaos.
- Use HBase only for the slices that truly need low-latency keyed access; do not try to turn the entire lake into an online store.
- Design the platform around producer-consumer decoupling so one producer schema change does not break every downstream team immediately.
- The core reason to choose a lake is flexibility and scale, but governance and metadata are what keep it usable.

### AQ11. [M4, M5] Design an always-writeable shopping-cart and user-preferences platform.
**Model answer**
- Use Dynamo-style key-value storage for shopping carts or transient user preference objects because rejecting writes is unacceptable in these paths.
- Configure replication and quorum settings to balance latency with durability, and use hinted handoff or anti-entropy to tolerate temporary failures.
- Use vector clocks or application-level merge logic so concurrent updates can be reconciled without silently losing user intent.
- Emit every cart or preference mutation to Kafka so downstream analytics, fraud detection, and recommendation systems can replay the event history.
- Store richer analytical history in HDFS and use Spark for offline analysis rather than overloading the online cart store with analytics queries.
- If the product later needs structured profile slices or scan-oriented feature access, add HBase for that separate purpose.
- The decisive architectural principle is that high-availability key-value state and offline analytics should be separate systems.

### AQ12. [M1, M4] A company has raw training corpora, keyed metadata, and highly available session state. How would you choose between HDFS, HBase, and Dynamo?
**Model answer**
- Put raw corpora, checkpoints, and large immutable training shards in HDFS because those workloads are scan-heavy and throughput-oriented.
- Put keyed sample metadata, sparse features, and partial-column access patterns in HBase because row-key lookup and subset-column retrieval are its strength.
- Put simple session or always-writeable personalization state in Dynamo-like storage because availability matters more than stronger structured semantics there.
- Do not ask one of these systems to replace the others; they solve different access patterns.
- Use Kafka in front of all three when the system benefits from event sourcing, replay, or decoupled downstream consumers.
- Use Spark as the integration layer for bulk ETL across all of them.
- The best exam answer is workload-first: scan-heavy goes to HDFS, structured keyed access goes to HBase, and availability-first simple key-value state goes to Dynamo.

### AQ13. [M5, M2, M4] Design a graph-analytics platform for fraud, social-network analysis, or network intelligence.
**Model answer**
- Ingest raw edge and event streams through Kafka so the graph can be updated or replayed as new relationships arrive.
- Store historical edge lists and feature snapshots in HDFS because large offline graph construction and reprocessing are bulk workloads.
- Use Spark for graph ETL, feature joins, filtering, and transformation into the graph format expected by the analytics engine.
- Use Pregel or Giraph-style execution for iterative graph algorithms such as PageRank, connected components, weakly connected components, or shortest path.
- If analysts also need online traversal and ad hoc graph exploration, add a graph database for that separate serving need rather than forcing Pregel to be interactive.
- Materialize graph-derived scores into HBase or another serving store so downstream applications can read them with low latency.
- The architecture works because offline graph analytics and online graph querying are different problems with different systems.

### AQ14. [M2, M3, M5] Design a hybrid batch-and-stream MLOps pipeline.
**Model answer**
- Use Kafka as the real-time event backbone and HDFS as the durable historical lake.
- Use Spark batch jobs to build training datasets, backfill features, and retrain models over long horizons.
- Use Structured Streaming to compute fresh rolling features, online aggregates, and drift-monitoring signals from the latest events.
- Store online feature slices or prediction-serving metadata in HBase when low-latency keyed reads are required.
- Train classical models in Spark ML when the problem is tabular and distributed, but hand off deep learning to PyTorch-style training stacks when the model requires GPUs and advanced parallelism.
- Version datasets, feature definitions, and checkpoints together so offline training and online serving stay consistent.
- The central design principle is to avoid separate, inconsistent batch and streaming worlds; the two must share storage, schema, and monitoring conventions.

### AQ15. [M1, M2, M3, M4, M5] Design a course-consistent platform for a startup that needs fast ingestion, durable storage, online metadata, large-scale model training, and graph analytics.
**Model answer**
- Use Kafka for all event ingress because it provides buffering, replay, and clean decoupling between producers and many downstream consumers.
- Use HDFS or a lake substrate for raw data, cleaned data, model artifacts, checkpoints, and graph snapshots because those are large durable assets.
- Use Spark for ETL, feature engineering, SQL analytics, and streaming metrics because it gives one scalable processing substrate across batch and fast data.
- Use HBase for online metadata or partial-column lookup, and reserve Dynamo-like storage for the narrow cases where always-writeable key-value behavior is more important than stronger structure.
- Use distributed DL frameworks for deep learning training and add federated learning only if privacy or data-locality constraints make centralized training unacceptable.
- Use Pregel-style graph processing for iterative whole-graph analytics and add a graph database only if the product also needs online graph traversal.
- The exam-quality rationale is that the platform is layered by workload: broker for streams, lake for bulk assets, Spark for computation, specialized stores for online serving, and specialized engines for graph and large-model workloads.

## Final Reminder

For this course, the strongest open-ended answers do not just define systems. They explain why a system's partitioning model, replication model, consistency choice, and bottleneck-avoidance strategy match a particular workload better than nearby alternatives.

# SSDS Final Exam Open-Ended Question Bank and Model Answers - 2

This file has two goals:

- provide expanded answers for the current final's open-ended questions,
- generate additional likely open-ended questions across all five modules, especially mixed-topic questions.

These are not meant to be memorized word-for-word. They are answer banks. In the exam, pick the points that best match the exact wording of the prompt.

## Primary Course Source Map

- Module 1: HDFS/GFS storage assumptions and mutation semantics.
- Module 2: MapReduce, Spark, RDDs, DataFrames, SQL, Catalyst.
- Module 3: Spark ML, distributed training, parameter server, federated learning, GNNs, LLM serving.
- Module 4: CAP, ACID, BASE, Dynamo, HBase.
- Module 5: Kafka, CEP, Structured Streaming, Pregel.

## How to Structure a Strong Open-Ended Answer

Use this answer shape whenever possible:

1. Start with the workload and constraints.
2. State the candidate systems and their design goals.
3. Compare them on partitioning, replication, consistency, locality, and bottlenecks.
4. Explain the performance implication of each design choice.
5. End with a justified recommendation and tradeoffs.

For architecture questions, use this answer shape:

1. Ingestion
2. Durable storage
3. Batch/stream processing
4. Online serving
5. Monitoring and recovery

## Part A: Expanded Answers for the Current Final's Open-Ended Questions

## Q1. If you had to design HDFS today to support AI/ML workloads, what design choices would you make? Describe the workload assumption and justify your design choices.

### Strong core answer

If I were redesigning HDFS for AI/ML today, I would still keep its original strengths, namely large sequential throughput, fault tolerance on commodity clusters, and locality-aware placement, because modern AI training workloads still read enormous corpora and checkpoints at scale. However, I would revise HDFS around the specific characteristics of AI/ML pipelines: huge immutable datasets, repeated multi-epoch scans, many concurrent trainers, large checkpoint writes, metadata pressure from many small files, and a mix of hot and cold data.

The first design choice is to optimize for large immutable training shards rather than many tiny files. Classic HDFS already assumes large files, and that remains correct because GPU training likes large sequential reads. So I would enforce or strongly encourage sharded formats, for example tar or parquet-like bundles, and I would add sample indexes so that applications can still find records without exploding NameNode metadata. This keeps the storage model aligned with HDFS's throughput-oriented design while fixing one of the biggest AI pain points, which is small-file metadata overhead.

The second design choice is a tiered storage layout. Hot datasets and active checkpoints should sit on SSD or NVMe-backed nodes, while colder corpora can use cheaper disks or erasure-coded archival storage. AI/ML workloads are not uniform: some data is touched every epoch, some only occasionally, and checkpoints need fast durable writes. Tiering improves cost efficiency without giving up the large-scale data-lake model.

The third design choice is stronger support for concurrent readers and large checkpoint writers. Modern training jobs often involve many workers reading the same shards while one or more trainers periodically emit large checkpoint files. I would preserve append-friendly semantics for checkpoint logs and add cheap snapshotting or copy-on-write mechanisms for dataset versioning, because reproducibility is central in ML. Training runs need to know exactly which corpus version and tokenizer version were used.

The fourth design choice is to reduce control-plane bottlenecks. A single metadata bottleneck becomes painful at modern AI scale, especially with trillions of samples or many experiment artifacts. So I would use federated metadata, multi-NameNode high availability, or a partitioned namespace to reduce central pressure. The reason is the same as in the rest of the course: centralized coordination eventually limits scale.

The fifth design choice is explicit support for data locality and cluster-aware scheduling with accelerators. The original HDFS logic already tries to place data near compute, and this is still important because network movement can starve GPUs. In an AI cluster, I would integrate storage placement with training schedulers so that GPU workers read from nearby storage or prefetch data to local caches. For high-value training clusters, feeding the GPU fast enough is often more important than raw storage capacity.

The sixth design choice is to separate bulk training storage from online feature serving. HDFS should remain the lake for corpora, checkpoints, embeddings, and batch outputs, but not be forced into low-latency sample lookup. For keyed sample access or metadata retrieval, HBase is a better companion. This keeps HDFS focused on what it is best at instead of turning it into a general database.

Overall, my HDFS-for-AI design would remain append-oriented and throughput-first, but would add metadata scaling, hot/cold tiering, dataset versioning, cache-aware placement, and better integration with GPU training pipelines. The core justification is that AI training still behaves much more like a large-scale sequential scan workload than a transactional OLTP workload.

### Other valid points to add if space allows

- Use replication for hot data and erasure coding for colder data to balance reliability and storage cost.
- Add transparent dataset caching near GPU clusters to reduce repeated multi-epoch reads.
- Support immutable dataset snapshots for reproducibility and rollback.
- Build in telemetry for skew, hotspot blocks, and slow readers because GPU idle time is expensive.
- Optimize for large checkpoint write bursts and fast recovery after trainer failure.

### Alternative valid answer direction

A different but still strong answer is to argue that a modern AI-oriented HDFS should become more like a lakehouse substrate: immutable bulk data in HDFS, structured metadata outside HDFS, and a processing layer like Spark responsible for most indexing, filtering, and transformation. This is still defensible as long as the answer keeps HDFS in a throughput-first storage role.

## Q2. Spark Structured Streaming and Apache Kafka are platforms used to manage fast data. Discuss and contrast their approaches to scalability, reliability, and performance.

### Strong core answer

Spark Structured Streaming and Apache Kafka are both used in fast-data applications, but they solve different layers of the problem. Kafka is primarily a distributed event broker and replayable log, while Structured Streaming is a distributed continuous processing engine over streams modeled as unbounded tables. So the most accurate answer is that they are complementary rather than direct substitutes.

Kafka achieves scalability mainly through topic partitioning. Each partition is an ordered log owned by a leader and replicated to followers. Producers append to partitions, consumers read using offsets, and consumer groups allow different partitions to be processed in parallel by different consumers. This makes Kafka extremely good at decoupling producers and consumers, absorbing bursty writes, and replaying historical streams. Its performance design is throughput-oriented: batching, sequential disk writes, zero-copy fetch paths, and partition-based horizontal scaling.

Structured Streaming achieves scalability through distributed execution over Spark partitions and micro-batches or continuous triggers. It treats stream records as an incrementally growing DataFrame and executes operators such as filters, joins, aggregations, and event-time windows in a fault-tolerant distributed manner. Its scalability comes from partitioned state, Spark task parallelism, and the optimizer's ability to plan execution over structured operators. Unlike Kafka, it is not just transporting data; it is computing over it.

Their reliability models also differ. Kafka provides durability by replicating partitions and retaining records for time or size windows. Because consumers track offsets, they can recover after failure and re-read data. Structured Streaming provides fault tolerance through lineage, checkpointed progress, and durable state. If a job fails, it can restart from checkpointed offsets and state and continue processing, often with exactly-once style behavior when the source and sink semantics support it. In short, Kafka gives durable event history, while Structured Streaming gives durable stream computation.

Performance-wise, Kafka is optimized for very high ingest throughput and low-overhead transport. Structured Streaming is optimized for scalable analytics, not for being the cheapest possible log transport. Kafka's core bottleneck is partition count and broker capacity. Structured Streaming's core bottleneck is often state management, shuffles, and micro-batch scheduling overhead. Kafka is better when the problem is moving streams reliably between systems. Structured Streaming is better when the problem is joining, aggregating, windowing, or enriching those streams.

The best production architecture often uses both together: Kafka as the ingestion and replay backbone, Structured Streaming as the stateful compute layer, and a sink such as HBase, files, or another Kafka topic for outputs. This combined answer is stronger than forcing a false either-or choice.

### Other valid points to add if space allows

- Kafka preserves ordering within a partition; Structured Streaming preserves query semantics over distributed operators.
- Kafka stores stream history; Structured Streaming stores operator state and progress metadata.
- Kafka on its own does not express rich event-pattern logic; Structured Streaming does not replace the need for a durable broker.
- Structured Streaming integrates more naturally with Spark SQL, DataFrames, and batch ETL.

### High-scoring conclusion line

Kafka solves fast-data transport and replay, whereas Structured Streaming solves fast-data computation. Kafka is the log; Structured Streaming is the distributed query engine over that log.

## Q3. You are the chief architect for a new GenAI company, Existo. Discuss the factors you would consider in selecting data platforms and justify your choices.

### Strong core answer

For a GenAI company, the correct data-platform choices depend on the full lifecycle: data ingestion, corpus storage, preprocessing, model training, online serving, and monitoring. No single platform is optimal for all of these, so I would choose a layered architecture where each system matches the specific workload it is best at.

For ingestion, I would use Kafka. A GenAI company continuously receives documents, user interactions, feedback events, prompts, tool traces, and operational telemetry. Kafka is a strong fit because it decouples producers from downstream systems, scales through partitions, and allows multiple consumers such as training pipelines, analytics jobs, and monitoring dashboards to replay the same streams independently.

For durable corpus storage, I would use HDFS or an HDFS-like data-lake substrate. The raw corpus, cleaned corpus, tokenized shards, embeddings, and model checkpoints are large, append-heavy, and mostly read through large scans. That matches HDFS well. It is cheaper and more natural for large immutable datasets than forcing this workload into an online database.

For preprocessing and analytics, I would use Spark DataFrames and Structured Streaming. Spark is strong for deduplication, filtering, quality scoring, token statistics, joins with metadata, and large-scale ETL. Structured Streaming can consume Kafka events for online monitoring, data quality alerts, and model-drift metrics.

For low-latency metadata lookup, I would choose HBase when access is keyed and partially column-oriented, for example looking up document metadata, feature subsets, user profile features, or experiment-tracking rows. If the most critical production service instead demands extremely high write availability under failures for simple key-value objects such as session state or online personalization state, then a Dynamo-style key-value store becomes more attractive. So my choice between HBase and Dynamo would depend on whether I need stronger structure and scans or maximum always-writeable availability.

For deep model training, I would not use Spark ML for the core LLM training loop. I would use a deep learning stack such as PyTorch distributed with data parallelism, model parallelism, and pipeline parallelism as needed. Spark's role is data engineering; the training framework's role is GPU-efficient optimization.

For privacy-sensitive edge personalization, I would consider federated learning, but only if the product genuinely needs on-device training or regulatory data locality. Otherwise, centralized training is simpler and converges faster. The key is to adopt FL only when privacy and edge data justify the additional systems complexity.

For LLM serving, I would use an ORCA-like serving layer or a modern equivalent that recognizes the difference between prefill and decode. Standard request-level batching performs poorly for generative serving because requests vary greatly in length. Iteration-level scheduling, KV-cache management, and careful GPU utilization are essential.

For monitoring and online evaluation, I would combine Kafka and Structured Streaming. Kafka captures the event stream; Structured Streaming computes aggregates, error rates, latency distributions, drift signals, and business KPIs in near real time.

So the overall architecture is: Kafka for ingestion, HDFS for the corpus lake, Spark for ETL and analytics, HBase or Dynamo depending the serving pattern, distributed DL frameworks for model training, and ORCA-like serving for inference. The rationale is simple: each layer is chosen to match its abstraction and bottleneck.

### Alternative valid answer profiles

#### Cost-sensitive startup version

- HDFS-like lake plus Spark plus Kafka as the core.
- Delay adding HBase or Dynamo until a real low-latency serving bottleneck appears.
- Centralized training only, no federated learning initially.

#### Privacy-sensitive consumer-AI version

- Add federated learning for on-device personalization.
- Keep raw user data local where possible.
- Use centralized aggregation only for models and telemetry.

#### Retrieval-heavy enterprise-AI version

- Keep HDFS lake for raw documents and offline pipelines.
- Add HBase for document metadata and fast row-key retrieval.
- Optionally add graph or vector-oriented services if the product is retrieval or knowledge heavy.

### Good closing line

The right architecture is not the most feature-rich stack, but the one where each platform's abstraction matches one stage of the GenAI lifecycle without forcing one system to do every job badly.

## Part B: Important Current-Final Topics Rewritten as Open-Ended Answers

## Q4. Compare HDFS and Kafka as partitioned, replicated systems. Explain both the similarity and the crucial differences.

HDFS and Kafka are similar in that both partition data across machines and replicate those partitions for reliability. Both also use a single logical ordering point for writes to maintain consistency: in HDFS, the primary replica orders block mutations under a lease, while in Kafka the partition leader orders appends. This means both systems can provide a well-defined per-partition or per-block write order under normal operation.

However, they are optimized for very different abstractions. HDFS is a distributed file system designed for storing large files and serving high-throughput scans. Kafka is a distributed event log designed to decouple producers and consumers, preserve ordered streams within a partition, and support replay using offsets. HDFS is where you would store a training corpus or a large checkpoint. Kafka is where you would store the stream of new events, clicks, prompts, or telemetry that continuously feed downstream systems.

Their read semantics also differ. In HDFS, clients can often read from a convenient replica of a block. In the course's Kafka model, consumers read from the leader replica of a partition. Kafka also has explicit retention and consumer-group semantics that HDFS does not: the same partition can be replayed by multiple independent consumers, which is central to event-driven architecture.

So the best summary is that HDFS and Kafka both use partitioning and replication to scale, but HDFS is storage for large files whereas Kafka is storage-plus-transport for ordered event streams.

### Add if needed

- HDFS optimizes throughput for large file scans.
- Kafka optimizes append throughput and multi-consumer replay.
- Kafka is not a replacement for a data lake.
- HDFS is not a replacement for a message broker.

## Q5. Suppose you must choose between HDFS and HBase for a large corpus of training samples. How would you decide?

The choice depends on the access pattern. If training is done by repeatedly scanning a large dataset sequentially, HDFS is usually better because it is optimized for large immutable files, block partitioning, and high aggregate throughput. This matches how many distributed training jobs consume data: as sharded files read in parallel by workers.

If the workload instead requires fetching training samples by key, updating records, retrieving only a subset of fields, or serving metadata online to other systems, HBase is better. HBase provides row-key lookup, range scans, sparse columns, and versioned cells on top of HDFS. This makes it far more suitable than HDFS for semi-structured training metadata or feature-store-like access patterns.

So the real rule is: HDFS for full scans of bulk data, HBase for keyed or partial structured access. A strong answer can also mention that the two often coexist, with HDFS holding the full corpus and HBase holding metadata, labels, sample indexes, or feature subsets.

## Q6. What is the difference between a CEP system and a PubSub system?

A PubSub system such as Kafka is primarily about communication. It delivers events from producers to consumers through topics, decouples the two sides, and supports scaling through partitions and consumer groups. Its central question is: how do events move reliably and scalably through the system?

A CEP system is about reasoning over event content, time, and patterns. It treats streams as unbounded relations and supports operations such as filters, transformations, windows, joins, and pattern detection. Its central question is: what higher-level meaning or derived event should be produced from the raw event stream?

So PubSub is transport and decoupling; CEP is continuous query processing over event streams. They are often used together, with a PubSub broker feeding a CEP engine or distributed stream processor.

## Q7. Why is Pregel often a better fit than Spark RDD/DataFrames for large-scale graph algorithms?

Pregel is designed specifically for iterative graph algorithms. It uses a vertex-centric BSP model where each vertex runs compute(), sends messages to neighbors, and synchronizes at superstep barriers. The graph structure remains resident across iterations, so repeated computation over the same topology does not require reconstructing or repeatedly shuffling the full graph.

In Spark, graph algorithms often require joins, groupBy-like operations, or other wide dependencies in every iteration. These repeated shuffles are expensive because they involve network transfer, disk I/O, and stage barriers. That makes Spark less efficient for graph-native iterative workloads such as PageRank or connected components.

Pregel therefore wins when the dominant workload is iterative message passing over a stable graph. Spark still wins when graph analytics is just one stage in a larger ETL or ML pipeline, because Spark is more general.

## Q8. What are the scalability limitations of Dynamo?

Dynamo scales well for key-value access, but not perfectly. First, skew can still occur. If the hash function or token assignment does not distribute load well, some virtual nodes or physical nodes become hotspots. Second, workload skew can dominate even if hashing is uniform: if most requests target a few keys, the same replica set becomes overloaded. Third, heterogeneity complicates balancing. Dynamo uses virtual nodes to help, but different machine capacities still make placement and recovery more complex. Fourth, concurrent updates to the same key create many versions, which increases read-time reconciliation cost and can hurt scalability.

The high-level exam point is that partitioning solves average-case scale, not hotspot keys, skewed traffic, or heavy conflict on the same object.

## Q9. How is distributed DNN training using Parameter Server different from Federated Learning?

Parameter Server is designed for distributed training inside a cluster or data center. Workers compute gradients on local data shards and exchange parameter updates with parameter servers. The goal is to scale training throughput when data is already cluster-local or centrally managed.

Federated Learning is designed for edge or wide-area environments where raw data cannot or should not be centralized. Devices download a global model, train locally on private data, and send model updates or local models back for aggregation. The goal is not just scale, but also privacy and data locality.

So the core distinction is that PS is a cluster coordination architecture for centralized or cluster-controlled training, while FL is a privacy-aware distributed learning paradigm for decentralized data.

## Part C: New Likely Open-Ended Questions Across All Modules

## Q10. Explain how the design assumptions of GFS/HDFS influence the performance of Spark.

Spark performs well on HDFS-backed workloads because the design assumptions align. HDFS stores large files in replicated blocks across many machines and is optimized for high-throughput sequential reads. Spark, in turn, builds partitions and tasks over those blocks and tries to schedule computation near the data. This minimizes network movement and makes it natural to process huge datasets in parallel.

The alignment is especially strong for batch analytics and iterative processing over large files. Spark avoids repeated re-reading when it can cache data, but HDFS still provides the scalable persistence layer. The main mismatch appears when datasets are composed of many tiny files or when the workload needs low-latency random record access. Those cases create metadata pressure and poor locality, and they are exactly why systems like HBase exist.

So the right answer is that HDFS gives Spark a partitioned, locality-aware storage substrate, but only for the class of workloads HDFS was designed for: large, throughput-oriented data access.

## Q11. Why are DataFrames and SQL usually preferred over RDDs for analytics workloads?

DataFrames and SQL expose structure to Spark. The engine can see schemas, filters, projections, joins, and aggregations, so Catalyst can optimize the plan. It can push down filters, prune unused columns, reorder operations, and choose better physical strategies. RDDs do not offer this because the core logic is hidden inside user functions that Spark treats as opaque.

This does not mean RDDs are obsolete. They are still useful when the logic is highly custom or not naturally relational. But for classic analytics workloads, DataFrames are preferred because they give both a simpler programming model and a faster execution model.

## Q12. Discuss ACID, CAP, and BASE in the context of cloud microservices.

ACID gives strong correctness guarantees through atomicity, consistency, isolation, and durability. These guarantees simplify application logic, but they often rely on coordination and locking, which become bottlenecks in large distributed settings.

CAP explains that during partitions, a system must trade between stronger consistency and higher availability. In large cloud environments, partition tolerance is not optional, so the practical tradeoff is between consistency and availability under failure.

BASE is the engineering response used by many cloud systems. It accepts that some services can be basically available, use soft state, and converge eventually. This is attractive in microservice architectures where latency and uptime matter greatly. Dynamo is the clearest course example of this design point.

A strong answer should conclude that cloud systems do not reject correctness, but selectively weaken global coordination when the workload can tolerate it.

## Q13. Compare HBase and Dynamo for an e-commerce or personalization platform.

If the workload is primarily key-based access with a strong requirement that writes should almost never be rejected, Dynamo is the better fit. Shopping carts, user preferences, and session objects are classic examples because availability directly affects revenue and customer experience. Dynamo's key-value model, consistent hashing, sloppy quorum, and hinted handoff were designed for this exact situation.

If the workload instead needs structured rows, sparse attributes, subset-column reads, or range-based access, HBase is better. It can support richer profile records and scan-oriented access patterns that Dynamo's hash-based design handles poorly.

So the answer depends on whether the dominant pressure is always-writeable key access or structured sparse data access. Mentioning that both may coexist in a larger system is a high-quality conclusion.

## Q14. Design a fast-data architecture for an IoT or smart-city application using the systems in the course.

I would start with sensors and edge devices producing events into a PubSub broker such as Kafka. Kafka is a good fit because it decouples data producers from downstream consumers and absorbs bursts through durable partitioned logs. On top of Kafka, I would use CEP for threshold rules, sequence patterns, and event-time detections such as overflow alerts or anomaly sequences. For broader analytics such as rolling aggregates, joins, dashboards, and drift tracking, I would use Spark Structured Streaming.

For durable archival and keyed operational retrieval, I would store results in HBase. That allows dashboards or control systems to fetch recent values, metadata, or device histories efficiently. Historical raw streams can also be persisted to HDFS for later offline analytics and model training.

This design works because each layer solves a different piece of the problem: Kafka for transport, CEP for event logic, Structured Streaming for scalable continuous computation, and HBase or HDFS for storage and serving.

## Q15. Explain data parallelism, model parallelism, and pipeline parallelism for training GPT-like models. When is each necessary?

Data parallelism splits training data across workers while keeping a full copy of the model on each worker. It is the simplest way to increase throughput, but it only works if the full model fits in each worker's memory.

Model parallelism splits the model itself across workers, either across layers or within layers. This becomes necessary when the model is too large to fit on a single GPU. The tradeoff is more communication between devices and potentially low utilization if only one stage is active at a time.

Pipeline parallelism improves utilization in model-parallel settings by splitting mini-batches into micro-batches and overlapping execution across model stages. This reduces device idle time, though it introduces pipeline bubbles and activation-storage overhead.

For modern GPT-like models, all three are often combined: data parallelism for throughput, model parallelism for capacity, and pipeline parallelism for utilization.

## Q16. Why is LLM serving different from normal batch inference? Explain using prefill, decode, and ORCA.

LLM serving is different because generative inference is iterative. The prompt is processed in a prefill phase, where many tokens can be handled together and the KV cache is built. After that, decoding generates one token at a time, repeatedly consulting the growing KV cache. This means the compute profile changes during the lifetime of a request.

Prefill is comparatively more compute-intensive. Decode is often more memory-bandwidth-bound because each new token must access large cached state while doing relatively little fresh computation. This leads to different latency metrics: time-to-first-token depends heavily on prefill and queueing, while time-per-output-token depends on decode efficiency and cache behavior.

ORCA improves serving because it schedules work at iteration granularity instead of holding batches until every request in the batch completes. This reduces head-of-line blocking and makes better use of GPU time when request lengths vary.

## Q17. Compare GNN message passing with Pregel message passing.

Both GNNs and Pregel rely on neighborhood communication, which is why the comparison is so useful. In Pregel, each vertex runs a user-defined compute() function, consumes incoming messages, updates local state, and sends messages onward in the next superstep. This is algorithmic graph processing.

In GNNs, each node aggregates feature information from its neighbors using differentiable functions and then updates its embedding. This is learned graph processing. The objective is usually node, edge, or graph prediction, not just algorithmic graph properties.

So the similarity is message passing over graph structure, while the difference is the purpose: Pregel computes graph algorithms, GNNs learn trainable graph representations.

## Q18. How would you design an end-to-end recommendation or personalization pipeline using the systems from this course?

I would ingest user interactions through Kafka because it naturally captures clicks, views, searches, and purchases as event streams. Kafka also supports replay, which is useful for both offline retraining and online debugging. These events would be persisted to HDFS as the historical data lake. Spark would then run batch ETL, sessionization, feature engineering, and model-training data preparation over this lake.

For online serving, I would choose between HBase and Dynamo based on workload. If I need structured user profiles, sparse features, and range-like access, HBase is better. If I need highly available key-value serving such as session personalization or cart state, Dynamo-style design is better.

If relationship structure matters, such as user-item or user-user graphs, I would add Pregel for graph algorithms like PageRank or connected components, and possibly distributed GNN training if the goal is learned recommendations rather than only graph scores. Structured Streaming can then monitor online performance, drift, and feedback loops in near real time.

## Q19. Explain why sharding solves some scalability problems but not all.

Sharding helps because it partitions data and allows different shards to scale independently. If a transaction or computation touches only one shard, contention is lower and performance can scale almost linearly. This is why Gray argued that divide-and-conquer through sharding is the only stable path for many distributed services.

However, sharding does not eliminate cross-shard coordination. As soon as operations span multiple shards, the system may need locks, distributed commit, or cross-partition communication. At that point the same scalability issues reappear in a new form. Similarly, hashing or partitioning does not prevent hotspot keys or skewed workloads.

So the correct answer is that sharding solves the average-case scaling of disjoint work, but not the hard cases of cross-shard coordination, skew, or hot objects.

## Q20. Design a course-consistent architecture for a privacy-sensitive mobile AI application.

For a privacy-sensitive mobile AI application, I would avoid collecting all raw data in the cloud. Instead, I would use federated learning so that each device trains locally on personal data and only model updates are aggregated centrally. This directly addresses the privacy and data-locality concerns emphasized in the course slides.

For telemetry, monitoring, and non-sensitive event streams, Kafka remains useful because the platform still needs operational logs, crash traces, and aggregate usage events. Structured Streaming can compute service health, fairness, and rollout metrics over those streams. HDFS remains useful for storing public or sanitized training corpora, checkpoints, and experiment artifacts. An HBase-like service can support low-latency metadata or device-profile lookups where appropriate.

This answer is strong because it does not force a single privacy mechanism onto the entire system. Sensitive user data stays local through FL, while non-sensitive operational infrastructure still uses the batch and streaming systems from the rest of the course.

## Q21. Why did Spark replace MapReduce for many large-scale analytics workloads?

MapReduce was a breakthrough because it gave programmers a simple abstraction and gave systems automatic parallelization and fault tolerance over commodity clusters. However, its model is rigid and stage materialization is expensive. Many workloads, especially iterative analytics and machine learning, repeatedly read and write intermediate state to disk, which creates large overhead.

Spark replaced MapReduce in many settings because it offers a richer compute model with lineage-based fault tolerance and in-memory reuse of intermediate results. This makes it much better for iterative algorithms, interactive analytics, and multi-stage pipelines. Spark also supports multiple abstractions in one platform: RDDs for custom logic, DataFrames and SQL for declarative analytics, Structured Streaming for fast data, and ML libraries for downstream tasks.

The strongest comparison point is that MapReduce is excellent for simple large batch jobs, but Spark generalizes the model and reduces repeated I/O costs, especially when the same working set is reused across iterations.

## Q22. Compare Spark ML, Scikit-Learn, and PyTorch for machine learning workloads.

Scikit-Learn is strongest for exploration, classical ML, and small to medium datasets on a single machine. It has a rich set of algorithms and works well with notebook-style data science. Its main weakness is that it does not scale naturally to distributed data.

Spark ML is strongest when the dataset is already in a distributed Spark pipeline and the workload is classical rather than deep learning. It integrates naturally with DataFrames, feature engineering, and pipeline APIs, so it is a good choice for operational ML over large tabular data. Its weakness is that it is narrower than specialized ML libraries and does not target deep neural training well.

PyTorch is strongest for deep learning. It is the right abstraction for DNNs, transformers, GNNs, and distributed GPU training using data, model, and pipeline parallelism. It is not primarily a large-scale ETL framework, so in many real pipelines Spark prepares the data and PyTorch trains the model.

So the best answer is: Scikit-Learn for single-machine classical ML exploration, Spark ML for distributed classical ML pipelines, and PyTorch for modern deep learning.

## Q23. Explain the difference between sliding and batch windows in stream processing, and why the choice matters.

Sliding windows produce overlapping views of recent data and usually trigger more frequently. A sliding length window keeps the last N events and emits a result whenever a new event arrives. A sliding time window keeps the last T time units and produces updated outputs as time advances. These windows give fresher results but require more repeated computation and more careful state maintenance.

Batch windows produce non-overlapping chunks. A batch length window groups every N events together, while a batch time window groups all events in a time bucket and triggers at the boundary. These windows reduce trigger frequency and may simplify downstream processing, but they increase latency because the system waits for the full bucket to close.

The choice matters because it changes both semantics and cost. If an application needs immediate detection of anomalies or rolling averages, sliding windows are usually better. If it cares more about periodic summaries than instant response, batch windows are often cheaper and simpler.

## Part D: Quick Extra Prompts for Self-Testing

Use these as rapid revision questions. Try answering each in 5 to 8 sentences.

1. Why is Kafka a better ingestion backbone than HDFS for continuous event streams?
2. Why is HDFS a better corpus store than HBase for multi-epoch full-dataset training?
3. Why are shuffles the central cost in Spark and what are the closest analogues in ML and graph systems?
4. Why is HBase row-key design as important as Dynamo token placement?
5. Why is a PubSub broker not enough for stream analytics?
6. Why is a parameter server not the same thing as federated learning?
7. Why can Spark be the wrong abstraction for PageRank even though Spark is general?
8. Why is decode, not prefill, often the serving bottleneck in LLM inference?

## Part E: Final Exam Answer Templates

## Template 1: Compare Two Systems

System A and System B both scale by partitioning and replication, but they optimize different abstractions and workloads. System A is designed for [workload], so its partitioning strategy is [strategy] and its main bottleneck is [bottleneck]. System B is designed for [other workload], so it instead uses [other strategy] and accepts [tradeoff]. Therefore, the right choice depends on whether the dominant requirement is [requirement 1] or [requirement 2].

## Template 2: Architecture Design Question

I would separate the system into ingestion, durable storage, processing, online serving, and monitoring. For ingestion I would use [system] because [reason]. For bulk durable storage I would use [system] because [reason]. For batch or stream analytics I would use [system] because [reason]. For low-latency serving I would use [system] because [reason]. This layered choice is justified because each stage has a different access pattern and bottleneck.

## Template 3: Tradeoff Question

The main tradeoff is between stronger coordination and better scalability or availability. If we choose stronger consistency, we gain [benefit] but pay [cost]. If we relax consistency or centralization, we gain [benefit] but accept [cost]. The best design depends on whether the application values [goal] more than [goal].

## Final Reminder

The strongest answers in this course do not just describe a system. They explain why its partitioning, replication, and execution model match a particular workload better than the alternatives.