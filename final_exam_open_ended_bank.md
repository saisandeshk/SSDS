# SSDS Final Exam Open-Ended Question Bank and Model Answers

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