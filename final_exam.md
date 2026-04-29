# DS256: SSDS Final Exam (2025)

1. Both HDFS and Kafka use the notion of partitioning and replication to improve scaling and reliability. In what ways are they similar/different?
   - In HDFS record append, a primary replica for a block receives the append request, computes the offset, and forwards this to the other replicas. Similarly, in Kafka, the leader for a partition receives the published message, adds it to the end of its partition, and sends this message to the followers. So both achieve consistent writes in the absence of failures.
   - Both in HDFS and Kafka, we can read from any data node/broker holding a block/partition replica.
   - HDFS uses leasing with the Name Node to decide the current primary replica while in Kafka, the leader for a partition is elected using Zookeeper.
   - HDFS's has better load balancing since it takes into account the capacity of data nodes when placing blocks, while Kafka uses a hash of the message key to decide the partition it goes to, and can hence suffer from imbalance.

2. Say we have an event stream with the following pairs of (time, value) events in the given order: (1,5), (2,2), (4,6), (5,2), (7,8), (8,3), (9,4), (10,7). What is the output from performing a sum over the values for a sliding time window operation with a slide length of 2 time units and a window/batch length of 4 time units. Ignore windows with incomplete lengths. One of the choices is correct.
   - (5+2+6), (2+8+3)
   - (5+2+6), (6+2), (2+8+3), (8+3+4+7)
   - (5+2+6+2), (8+3+4+7)
   - (5+2+6+2), (6+2+8+3), (8+3+4+7)

3. Say you are asked to choose between HDFS and HBase for storing a large corpus of training data samples. Which of these are correct reasons to choose one over the other?
   - HDFS is good if the samples are binary records while HBase is better if they are semi-structured text.
   - HDFS is good if we need to partition the samples based on their labels, while HBase is good is we wish to query the samples by a unique ID.
   - HDFS is better if we are performing a linear scan through the samples to extract a subset of samples for training, while HBase is better if we are extracting a subset of fields/columns from each sample for training.
   - HDFS is better if we keep adding more samples over time, while HBase is better if we are editing the contents of existing samples.

4. What are the differences between a Complex Event Processing (CEP) system and a Publish Subscribe (PubSub) system?
   - CEP systems are easier to weakly scale while PubSub systems are harder to scale across distributed machines.
   - CEP can be used for query processing over typed events while a PubSub system is used for sending events between different entities.
   - CEP can perform window operations and aggregations, while PubSub systems use topics for routing.
   - Both CEP and PubSub systems attempt to achieve low latency execution.

5. What are benefits of using Pregel over Spark RDD/Dataframes for large scale graph processing?
   - For iterative graph algorithms designed using Spark RDDs, the entire graph structure can undergo a "shuffle" over the network/disk for each iteration.
   - Pregel maintains the graph structure in-memory across all iterations of the execution, thus avoiding disk/network costs for "shuffling" the graph topology.
   - For any given graph, Pregel can perform a traversal over the entire graph in a single iteration (superstep).
   - Pregel uses message passing to send information from one vertex to another across different iterations (supersteps) while Spark uses joins/grouping transformations.

6. How do fast/streaming data applications composed as a dataflow of tasks (user logic blocks) achieve scalability using distributed stream processing systems?
   - Applications can use stateful tasks to improve the performance and scalability of stream processing.
   - Different tasks in the dataflow can be executed in parallel across different machines using task parallelism.
   - Upstream tasks in the dataflow can be processing newer events at the same time as downstream tasks that are processing earlier events for pipeline parallelism.
   - Different events in an input stream to a logical task can be executed in parallel by multiple instances of the task using data parallelism.

7. If you had to design HDFS today to support AI/ML workloads, what are the design choices you will make? Describe the workload assumption and justify your design choices/changes for HDFS in detail.
   *(Note: Essay type - Maximum 3500 characters)*

8. Spark Structured Streaming and Apache Kafka are platforms that are used to manage fast data. Discuss and contrast their design approaches they take to achieving scalability, reliability and performance.

9. Say you are the chief architect for a new GenAI company, Existo. You are responsible for assembling the data engineering and training pipeline. Discuss the various factors you will consider in selecting the data platforms in your company and the rationale for the choices of these specific systems.

10. What are some of the scalability limitations of Dynamo?
    - If the hash function of the key does not distribute its output uniformly among the virtual nodes, we may have some virtual nodes be overloaded and that limits scalability.
    - If most of the read and write requests are localized to just a few keys, then the same set of virtual/physical nodes will get overloaded, and this limits scalability.
    - If the physical nodes have varying capacities, then lower capacity physical nodes will get overloaded, and this limits scalability.
    - If the number of write operations to the same key from different clients is high, the number of versions can grow large and cause poor scalability.

11. Pregel/Giraph uses hash partitioning to assign vertices to workers/machines. What are the pros and cons of this approach?
    - Hashing on vertex ID uniformly spreads the vertex load across different partitions
    - Hashing on vertex ID can cause edges to be imbalanced for powerlaw graphs
    - Hashing maintains state of the vertex to partition mapping that can cause memory pressure
    - Partitioners like METIS cannot be used with Giraph

12. How is distributed training of DNN models using Parameter Server (PS) different from Federated Machine Learning (Fed ML)?
    - In Fed ML, we train local DNN models and aggregate these local models into a global model at a central server.
    - In PS, we calculate local gradients and send them to the parameter servers that aggregate the gradients and update the weights.
    - PS offers better privacy of the source data than Fed ML since only the gradients are shared and not the entire local model.
    - PS is better suited for execution on wide-area networks while Fed ML is better suited for execution within data centers.

13. Which of these statements about Pregel are TRUE?
    - The Bulk Synchronous Parallel (BSP) model of Pregel iteratively performs computation and communication phases.
    - A Pregel application stops when all vertices have voted to halt and no messages are awaiting processing at a superstep.
    - The Pregel model allows all vertices to execute their compute() logic in parallel within a superstep.
    - When implementing PageRank using Pregel, all vertices will send a message to all their out-going neighbors in every superstep.