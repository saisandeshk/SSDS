# DA256 Midterm (Module 1 and 2)

Multiple Choice Questions. Multiple correct and wrong answers. Negative marking for wrong choice(s). Lowest mark for each question is 0.

Points: -/60

---

## Question 1
Which of these computing devices typically have multi-core processing units and require parallel programming for effective use?

1. Smart phones
2. GPU workstations
3. CPU-based servers
4. Cloud Virtual Machines

---

## Question 2
What Spark operations do Google MapReduce's map() and reduce() functions resemble?

1. Google MR's map() is like Spark's map(). Google MR's reduce() is like Spark's reduce().
2. Google MR's map() is like Spark's mapPartitions(). Google MR's reduce() is like Spark's reduce().
3. Google MR's map() is like Spark's flatMap(). Google MR's reduce() is like Spark's cogroup().
4. Google MR's map() is like Spark's flatMap(). Google MR's reduce() is like Spark's groupByKey() for the shuffle followed by flatMap().

---

## Question 3
Which of these assumptions did Google make in their SOSP paper on the design of GFS?

1. The system stores a small number (1000s) of large files (&gt;1GB)
2. The system runs on commodity machines with lower reliability of components
3. The read workload is limited to large streaming reads.
4. The system should efficiently support multiple concurrent client appends to same file

---

## Question 4
Which of these are valid reasons to use a distributed file system as opposed to a single-machine file system?

1. Higher cumulative access bandwidth
2. Fault tolerance
3. Better data consistency
4. Weak scaling

---

## Question 5
Which of these statements about Spark DataFrames and Spark SQL is/are TRUE?

1. Spark DataFrames follow an imperative programming model while Spark SQL is a declarative programming model.
2. Unlike Spark RDDs, all transformation and action operations in Spark DataFrames are lazily evaluated.
3. Spark DataFrames are flexible since they allow us to update the contents of the table in-place.
4. The Catalyst optimizer uses rule-based transformations of the Abstract Syntax Tree combined with a cost based optimizer to select the nominally best physical plan.

---

## Question 6
What are the benefits of large block sizes in HDFS/GFS?

1. Reduces the amount of metadata maintained in the Name Node for each file.
2. Improves the cumulative read performance through parallel block reads.
3. Improves the read performance by amortizing the overheads for establishing the TCP connection from the client to data node.
4. Reduces the probability of record append conflicts from concurrent clients.

---

## Question 7
Which of these statements on strong and weak scaling are FALSE?

1. Amdahl's law says that perfect strong scaling is theoretically impossible.
2. Gustafson's law says that perfect strong scaling is theoretically impossible.
3. If an application has 10% of sequential code and 90% of parallel code, it is not meaningful to allocate 100 processors to it to achieve strong scaling.
4. Big data platforms are usually designed to achieve strong scaling.

---

## Question 8
What are the downsides of replication of blocks in HDFS/GFS?

1. More numbers of block replicas need to accessed when reading a file.
2. More numbers of block replicas need to be written to when writing to a file.
3. More space is disk occupied for a file due to block replication.
4. There is a higher possibility of permanently losing the contents of a file since there are more replicas on more machines that can fail.

---

## Question 9
Which is the correct order of steps in which the write data flow happens in GFS: (A) Primary asks secondaries to write the received mutation to disk, (B) Primary has a lease on the block, (C) Client sends the write commit request to the Primary, (D) Client sends mutation contents to secondary and primary in a pipelined manner?

1. A, B, C, D
2. B, A, D, C
3. B, D, C, A
4. C, D, B, A

---

## Question 10
Which of these statements about the consistency model of GFS are true?

1. If a region in the block is defined, it is also consistent.
2. The use of record append has severe performance overheads and should be used rarely.
3. A block will always remain in a consistent state as long as only a single client mutates it at a time.
4. If any mutation operation fails, the region it was modifying can be left inconsistent.

---

## Question 11
Which of these formulations from the JPDC article indicate the total execution time for a workload with a finite number of processors and no communication overheads?

1. $T_N(W) = \sum \frac{W_i}{i \cdot \Delta} \cdot \lceil \frac{i}{N} \rceil + Q_N W$
2. $T_N(W) = \sum \frac{W_i}{i \cdot \Delta} \cdot \lceil \frac{i}{N} \rceil$
3. $T_N(W) = \sum \frac{W_i}{i \cdot \Delta}$
4. $T_N(W) = \sum \frac{W_i}{\Delta}$

---

## Question 12
Which of these statements about "lazy evaluation" of RDDs is/are TRUE?

1. Only wide transformation are lazily evaluated. Narrow tranformations are triggered immediately.
2. Lazy evaluation due to an action will can cause multiple RDDs to be created/materialized.
3. Once an RDD has been materialized and persisted, lazy evalaution guarantees that it will never be recreated.
4. Lazy evaluation helps Spark achieve resiliency since RDDs are created on-demand.

---

## Question 13
In the RDD paper (NSDI 2012), what are the pros and cons of RDDs over Distributed Shared Memory (DSM)?

1. RDDs offer coarse-grained write operations while DSM offer fine-grained writes.
2. RDDs use lineage graphs for reliability while DSMs use checkpoint/recovery.
3. RDDs are immutable and offer implicit consistency. Consistency of DSM depends on the application design.
4. RDDs use data locality for task placement. DSMs leave it to application runtimes.

---

## Question 14
Which of these data structures are maintained in the Name Node of GFS/HDFS in a reliable/durable manner?

1. The file system namespace (directories, files, permissions, etc.)
2. The list of block IDs for a given file.
3. The list of Data Nodes that contain a replica of a Block ID.
4. The list of clients that have acquired a lock on a file.

---

## Question 15
What is/are the characteristic(s) of the logical plan and physical plan during Spark RDD execution?

1. The physical plan translates the logical plan into a series of Tasks for execution on specific Executors.
2. The physical plan helps identify the all dependent transformations that need to execute for a given action.
3. The physical plan creates one Job for every action.
4. The logical plan will partition the dataflow into stages separated by shuffle boundaries.