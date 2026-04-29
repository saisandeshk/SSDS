# Lecture 2.1: Big Data Processing with Apache Spark (Part 1)

## DS256 - Scalable Systems for Data Science
### Module 2: Processing Large Volumes of Big Data

---

## 1. Role of Big Data Processing System for Large Data Volumes

When dealing with Big Data (especially Volume), we need a layered architecture where each layer builds upon the previous one, providing increasingly sophisticated capabilities.

### 1.1 The Big Data Processing Stack

```
┌─────────────────────────────────────────────────────┐
│         Specialized Processing System               │  E.g., Spark ML, GraphX
│   (Domain-specific algorithms: ML, Graph, etc.)     │
├─────────────────────────────────────────────────────┤
│          Generic Processing System                  │  E.g., Spark RDD, DataFrames
│   (General-purpose distributed computation)         │
├─────────────────────────────────────────────────────┤
│            Management System                        │  E.g., NoSQL (HBase, Cassandra)
│   (Structured access, queries, indexing)            │
├─────────────────────────────────────────────────────┤
│             Storage System                          │  E.g., GFS/HDFS, Ceph
│   (Reliable, distributed, fault-tolerant storage)   │
└─────────────────────────────────────────────────────┘
                        ▲
                        │
                  Data Arrives
```

### 1.2 Overview of Each Layer

#### Storage System (GFS/HDFS, Ceph)
- **Purpose**: Store massive amounts of data reliably across commodity hardware
- **Key Features**:
  - Distributed across hundreds/thousands of machines
  - Fault-tolerant through replication (typically 3 copies)
  - Optimized for large sequential reads/writes
  - Handles machine failures transparently
- **Examples**: Google File System (GFS), Hadoop Distributed File System (HDFS), Ceph
- **Trade-off**: High throughput for large files, but poor latency for small random accesses

#### Management System (NoSQL)
- **Purpose**: Provide structured access to data with indexing and query capabilities
- **Key Features**:
  - Schema-flexible or schema-less data models
  - Horizontal scalability (add more machines)
  - Support for CRUD operations (Create, Read, Update, Delete)
  - Various consistency models (eventual to strong)
- **Examples**: HBase (column-family), Cassandra (wide-column), MongoDB (document)
- **Trade-off**: Flexibility and scalability over ACID transactions

#### Generic Processing System (Spark RDD, DataFrames)
- **Purpose**: Execute arbitrary distributed computations on large datasets
- **Key Features**:
  - Data-parallel programming model
  - Fault-tolerant execution
  - In-memory computation for iterative algorithms
  - Rich set of transformations and actions
- **Examples**: Apache Spark (RDDs, DataFrames), Apache Flink
- **Trade-off**: General-purpose means not optimized for specific workloads

#### Specialized Processing System (Spark ML, GraphX)
- **Purpose**: Optimized libraries for specific domains
- **Key Features**:
  - Machine Learning: MLlib (classification, regression, clustering)
  - Graph Processing: GraphX (PageRank, connected components)
  - Stream Processing: Spark Streaming, Structured Streaming
- **Examples**: Spark MLlib, GraphX, Spark Streaming
- **Trade-off**: Highly optimized but limited to specific use cases

---

## 2. Storage System Fundamentals: Memory Hierarchy

Understanding the memory hierarchy is **critical** for designing efficient Big Data systems. The key insight is that different storage components have vastly different latencies and bandwidths.

### 2.1 The Memory Hierarchy

```
                    ┌───────────────┐
                    │  CPU Registers │  ← Fastest (< 1 ns)
                    │    (bytes)     │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   L1 Cache    │  ← 0.5 ns
                    │   (32-64 KB)  │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   L2 Cache    │  ← 7 ns
                    │  (256 KB-1MB) │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   L3 Cache    │  ← 20-40 ns
                    │   (8-64 MB)   │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │  Main Memory  │  ← 100 ns
                    │   (4-256 GB)  │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   SSD/NVMe    │  ← 10-150 μs
                    │  (256GB-8TB)  │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │    HDD        │  ← 10-20 ms
                    │  (1TB-20TB)   │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   Network     │  ← 0.5-150 ms
                    │  (Unlimited)  │
                    └───────────────┘

        Speed ▲                          ▼ Capacity
              │                          │
        Cost  │                          ▼ Persistence
```

### 2.2 Key Differences Between Components

| Component | Capacity | Latency | Bandwidth | Volatile? | Cost/GB |
|-----------|----------|---------|-----------|-----------|---------|
| **L1 Cache** | 32-64 KB | 0.5 ns | ~1 TB/s | Yes | Highest |
| **L2 Cache** | 256 KB-1 MB | 7 ns | ~500 GB/s | Yes | Very High |
| **Main Memory (RAM)** | 4-256 GB | 100 ns | 25-100 GB/s | Yes | ~$3-5/GB |
| **SSD** | 256 GB-8 TB | 10-150 μs | 500 MB-7 GB/s | No | ~$0.10/GB |
| **HDD** | 1-20 TB | 10-20 ms | 100-250 MB/s | No | ~$0.02/GB |
| **Network (Local)** | Unlimited | 0.5 ms | 1-100 Gbps | N/A | - |
| **Network (WAN)** | Unlimited | 50-150 ms | Variable | N/A | - |

### 2.3 Why This Matters for Big Data

1. **Memory is fast but limited and volatile** → Can't store everything in RAM
2. **Disk is large but slow** → Need parallelism to achieve throughput
3. **Network is the biggest bottleneck** → Move computation to data, not data to computation

---

## 3. MapReduce

MapReduce is a programming model and runtime system introduced by Google in 2004. It provides a simple abstraction for processing large datasets in parallel across a cluster of commodity machines.

### 3.1 The MapReduce Vision

> *"A simple and powerful interface that enables automatic parallelization and distribution of large-scale computations, combined with an implementation of this interface that achieves high performance on large clusters of commodity PCs."*
> — Dean and Ghemawat, OSDI 2004

### 3.2 Google's Cluster Model (2004)

The MapReduce paper describes the computing environment at Google in 2004:

| Component | Specification (2004) | Modern Equivalent |
|-----------|---------------------|-------------------|
| **CPU** | 2 x86 processors | 16-64 cores |
| **Memory** | 2-4 GB RAM | 64-256 GB RAM |
| **Network** | 100 Mbps - 1 Gbps per machine | 10-100 Gbps |
| **Cluster Size** | 100s - 1000s of machines | 1000s - 10000s |
| **Storage** | IDE disks (local) | SSDs + HDDs |
| **File System** | GFS | HDFS, Cloud Storage |

**Key Characteristics:**
- **Commodity hardware**: Not specialized supercomputers, but regular PCs
- **Failures are common**: With 1000s of machines, something is always failing
- **Low bisection bandwidth**: Network is the bottleneck (not enough bandwidth for all-to-all communication)
- **Storage is local**: Each machine has its own disks (managed by GFS)
- **Jobs submitted to scheduler**: Users don't directly control which machines run their code

### 3.3 MapReduce Design Pattern

MapReduce provides:

1. **Clean Abstraction for Programmers**
   - Users write just two functions: `map` and `reduce`
   - No need to worry about parallelization, distribution, or fault tolerance

2. **Automatic Parallelization & Distribution**
   - Framework handles splitting data and distributing tasks
   - Runs on hundreds/thousands of machines automatically

3. **Fault Tolerance**
   - Automatically re-executes failed tasks
   - No data loss even when machines fail

4. **Batch Data Processing**
   - Designed for large input sizes (TB/PB scale)
   - Not for real-time or interactive queries

### 3.4 Example Applications

| Application | Map Function | Reduce Function |
|-------------|--------------|-----------------|
| **Distributed Grep** | Emit line if matches pattern | Identity (copy to output) |
| **URL Access Frequency** | `<URL, 1>` for each access | Sum counts for each URL |
| **Reverse Web-Link Graph** | `<target, source>` for each link | Concatenate all sources per target |
| **Term-Vector per Host** | `<hostname, term_vector>` | Add term vectors for same host |
| **Inverted Index** | `<word, docID>` for each word | Collect all docIDs per word |
| **Distributed Sort** | `<key, record>` | Emit unchanged (partitioning does the work) |

---

## 4. MapReduce: Data-Parallel Programming Model

### 4.1 The Core Model

MapReduce processes data using two user-defined functions:

```
Input Data (K1, V1 pairs)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  MAP: (K1, V1) → List<K2, V2>                       │
│  - Called once on every input item                  │
│  - Emits zero or more intermediate key/value pairs  │
└─────────────────────────────────────────────────────┘
         │
         ▼
    Intermediate Data (K2, V2 pairs)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  SHUFFLE & SORT (Internal to Framework)             │
│  - Groups all values with the same key together     │
│  - Sorts keys within each group                     │
└─────────────────────────────────────────────────────┘
         │
         ▼
    Grouped Data (K2, List<V2>)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  REDUCE: (K2, List<V2>) → List<K3, V3>              │
│  - Called once on every unique key                  │
│  - Combines all values for a key into output        │
└─────────────────────────────────────────────────────┘
         │
         ▼
Output Data (K3, V3 pairs)
```

### 4.2 Type Signature

```
map:    (K1, V1)        → List<K2, V2>
reduce: (K2, List<V2>)  → List<K3, V3>
```

**Important**: Input key/value types (K1, V1) can be different from intermediate types (K2, V2), which can be different from output types (K3, V3).

### 4.3 Word Count Example

The classic MapReduce example - counting word frequencies in documents:

```python
# Pseudo-code

def map(document_name, document_contents):
    # document_name: e.g., "file1.txt"
    # document_contents: the text content of the file
    for word in document_contents.split():
        emit(word, 1)  # Emit <word, 1> for each occurrence

def reduce(word, counts):
    # word: a unique word
    # counts: list of all 1s emitted for this word [1, 1, 1, ...]
    total = sum(counts)
    emit(word, total)  # Emit <word, total_count>
```

**Execution Flow:**
```
Input: "hello world hello"

MAP:
  emit("hello", 1)
  emit("world", 1)
  emit("hello", 1)

SHUFFLE & SORT:
  "hello" → [1, 1]
  "world" → [1]

REDUCE:
  reduce("hello", [1, 1]) → emit("hello", 2)
  reduce("world", [1])    → emit("world", 1)

Output:
  hello 2
  world 1
```

---

## 5. Inverted Index using MapReduce

An **Inverted Index** is a fundamental data structure for search engines. It maps each word to the list of documents containing that word.

### 5.1 The Task

**Goal**: Given a collection of web pages/documents, build an index that answers:
> "Which documents contain the word X?"

**Example Input:**
```
doc1: "the quick brown fox"
doc2: "the lazy dog"
doc3: "the quick dog jumps"
```

**Desired Output (Inverted Index):**
```
the    → [doc1, doc2, doc3]
quick  → [doc1, doc3]
brown  → [doc1]
fox    → [doc1]
lazy   → [doc2]
dog    → [doc2, doc3]
jumps  → [doc3]
```

### 5.2 MapReduce Solution

```
                    MAP                     SHUFFLE                 REDUCE
                     
doc1: "the quick"   ──→  (the, doc1)    ┐
                         (quick, doc1)  ─┼─→ (the, [doc1,doc2,doc3]) ──→ (the, [doc1,doc2,doc3])
                                        │
doc2: "the lazy"    ──→  (the, doc2)    ┘    (quick, [doc1,doc3])   ──→ (quick, [doc1,doc3])
                         (lazy, doc2)   ───→ (lazy, [doc2])         ──→ (lazy, [doc2])
                                        
doc3: "the quick"   ──→  (the, doc3)         (dog, [doc2,doc3])     ──→ (dog, [doc2,doc3])
                         (quick, doc3)
```

### 5.3 Implementation

```python
def map(url, page_content):
    """
    Input:  url = document identifier
            page_content = text content of the page
    Output: Emits (word, url) for each word in the page
    """
    for word in page_content.split():
        emit(word, url)

def reduce(word, url_list):
    """
    Input:  word = a unique word
            url_list = list of all URLs where this word appears
    Output: Emits (word, distinct_urls) - the inverted index entry
    """
    distinct_urls = remove_duplicates(url_list)
    emit(word, distinct_urls)
```

### 5.4 Why MapReduce is Perfect for This

1. **Embarrassingly Parallel in Map Phase**: Each document can be processed independently
2. **Natural Grouping**: Shuffle automatically groups all occurrences of the same word
3. **Scalable**: Can process billions of web pages
4. **Fault-Tolerant**: If a machine fails, just re-process those documents

---

## 6. Map-Shuffle-Sort-Reduce: The Complete Pipeline

### 6.1 Detailed Execution Flow

```
        ┌───────────────────────────────────────────────────────────────────────────┐
        │                              MAPREDUCE JOB                                │
        └───────────────────────────────────────────────────────────────────────────┘

                    MAP PHASE                                    REDUCE PHASE
        ┌─────────────────────────────┐               ┌─────────────────────────────┐
        │                             │               │                             │
        │   Worker A      Worker B    │               │   Worker X      Worker Y    │
        │  ┌───────┐     ┌───────┐    │               │  ┌───────┐     ┌───────┐    │
        │  │ Map 1 │     │ Map 3 │    │               │  │Reduce1│     │Reduce2│    │
        │  └───┬───┘     └───┬───┘    │               │  └───┬───┘     └───┬───┘    │
        │      │             │        │               │      │             │        │
        │  ┌───┴───┐     ┌───┴───┐    │               │      │             │        │
        │  │ Map 2 │     │ Map 4 │    │               │      │             │        │
        │  └───┬───┘     └───┬───┘    │               │      │             │        │
        └──────┼─────────────┼────────┘               └──────┼─────────────┼────────┘
               │             │                               ▲             ▲
               ▼             ▼                               │             │
        ┌──────────────────────────────────────────────────────────────────────────┐
        │                                                                          │
        │                    SHUFFLE & SORT (Network Transfer)                     │
        │                                                                          │
        │   Each Map task writes intermediate data partitioned by reduce task      │
        │   Each Reduce task fetches its partition from ALL map tasks              │
        │                                                                          │
        └──────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Phase-by-Phase Breakdown

#### Phase 1: MAP
- **What happens**: 
  - Input data is split into M chunks (typically 16-64 MB each)
  - Each map task processes one chunk
  - User's map function is called on each key-value pair
  - Intermediate key-value pairs are buffered in memory

- **Where it runs**: On machines that have the input data (locality optimization)

- **Output**: Intermediate key-value pairs stored locally on the mapper's disk

#### Phase 2: SHUFFLE (The Expensive Part!)
- **What happens**:
  - Intermediate data is partitioned by key (using `hash(key) mod R`)
  - Each reducer fetches its partition from ALL mappers
  - This requires network transfer across the cluster

- **Why it's expensive**:
  - Every mapper must send data to every reducer
  - Network bandwidth becomes the bottleneck
  - If there are M mappers and R reducers, there are M × R data transfers

```
Mappers                          Reducers
┌─────┐                         ┌─────────┐
│ M1  │────────┬───────────────▶│   R1    │
└─────┘        │       ┌───────▶│         │
               │       │        └─────────┘
┌─────┐        │       │        
│ M2  │────────┼───────┤        ┌─────────┐
└─────┘        │       └───────▶│   R2    │
               │       ┌───────▶│         │
┌─────┐        │       │        └─────────┘
│ M3  │────────┴───────┘        
└─────┘                         
                   ▲
                   │
         All-to-All Communication!
         (This is the bottleneck)
```

#### Phase 3: SORT
- **What happens**:
  - Each reducer sorts its received data by key
  - Groups all values with the same key together
  - Prepares data for the reduce function

- **Why it's needed**:
  - Reduce function expects all values for a key together
  - Sorting enables efficient grouping

- **Cost**: External sort may be needed if data doesn't fit in memory

#### Phase 4: REDUCE
- **What happens**:
  - For each unique key, reduce function is called with all its values
  - Output is written to the distributed file system (GFS/HDFS)
  
- **Output**: R output files (one per reducer)

### 6.3 Why Shuffle and Sort Should Be Avoided/Minimized

The shuffle and sort phases are the **most expensive** parts of MapReduce:

| Reason | Impact |
|--------|--------|
| **Network Transfer** | All intermediate data crosses the network |
| **Disk I/O** | Data is written to disk by mappers, read by reducers |
| **Sorting Cost** | O(n log n) for sorting, plus external sort overhead |
| **Synchronization** | Reducers must wait for ALL mappers to complete |
| **Data Amplification** | Same data may be shuffled multiple times in multi-stage jobs |

**Strategies to Minimize Shuffle:**

1. **Combiners**: Pre-aggregate data at the mapper before shuffle
   ```
   Without combiner: Map emits [<a,1>, <a,1>, <a,1>] → 3 items shuffled
   With combiner:    Map emits [<a,3>]              → 1 item shuffled
   ```

2. **Smart Partitioning**: Ensure related data goes to the same reducer

3. **Filter Early**: Reduce data volume before shuffle using map-side filtering

4. **Avoid Unnecessary Shuffle**: Some operations don't need reduce at all

---

## 7. Histogram using MapReduce

### 7.1 The Task

Build a histogram of values, where each bucket contains the count of values falling within a range.

**Example**: Given values [0-11], create buckets of width 4:
- Bucket 0: values 0-3
- Bucket 1: values 4-7
- Bucket 2: values 8-11

### 7.2 Input Data

```
7    2    11   2
2    1    11   4
9    10   6    6
6    3    2    8
0    5    1    10
2    4    8    11
5    0    1    0
```

### 7.3 MapReduce Solution

```python
bucketWidth = 4  # Configurable parameter

def map(key, value):
    """
    Input:  value = a number
    Output: Emit (bucketID, 1)
    """
    bucketID = floor(value / bucketWidth)
    emit(bucketID, 1)

def reduce(bucketID, counts):
    """
    Input:  bucketID = bucket identifier
            counts = list of 1s for each value in this bucket
    Output: Emit (bucketID, frequency)
    """
    frequency = sum(counts)
    emit(bucketID, frequency)
```

### 7.4 Execution Trace

```
Input Values (4 partitions):
    [7,2,9,6,0,2,5]  [2,1,10,3,5,4,0]  [11,11,6,2,1,8,1]  [2,4,6,8,10,11,0]

MAP Phase (floor(value/4)):
    Partition 1:     Partition 2:      Partition 3:       Partition 4:
    (1,1) ← 7        (0,1) ← 2         (2,1) ← 11         (0,1) ← 2
    (0,1) ← 2        (0,1) ← 1         (2,1) ← 11         (1,1) ← 4
    (2,1) ← 9        (2,1) ← 10        (1,1) ← 6          (1,1) ← 6
    (1,1) ← 6        (0,1) ← 3         (0,1) ← 2          (2,1) ← 8
    (0,1) ← 0        (1,1) ← 5         (0,1) ← 1          (2,1) ← 10
    (0,1) ← 2        (1,1) ← 4         (2,1) ← 8          (2,1) ← 11
    (1,1) ← 5        (0,1) ← 0         (0,1) ← 1          (0,1) ← 0

SHUFFLE Phase (28 items transferred across network):
    Key 0 → [(0,1), (0,1), (0,1), (0,1), (0,1), (0,1), (0,1), (0,1), (0,1), (0,1), (0,1), (0,1)]
    Key 1 → [(1,1), (1,1), (1,1), (1,1), (1,1), (1,1), (1,1), (1,1)]
    Key 2 → [(2,1), (2,1), (2,1), (2,1), (2,1), (2,1), (2,1), (2,1)]

REDUCE Phase:
    Bucket 0: 12 values (0,1,2,3)
    Bucket 1: 8 values  (4,5,6,7)
    Bucket 2: 8 values  (8,9,10,11)

Output:
    (0, 12)
    (1, 8)
    (2, 8)
```

### 7.5 Using a Combiner for Optimization

Without combiner: 28 items shuffled
With combiner: Pre-aggregate at each mapper

```python
def combiner(bucketID, counts):
    """Same as reduce - can be used because sum is associative"""
    emit(bucketID, sum(counts))
```

```
After Combiner (before shuffle):
    Partition 1: (0,2), (1,3), (2,1)   → 3 items
    Partition 2: (0,4), (1,2), (2,1)   → 3 items
    Partition 3: (0,3), (1,1), (2,3)   → 3 items
    Partition 4: (0,2), (1,2), (2,3)   → 3 items
    
Total items shuffled: 12 (vs 28 without combiner) → 57% reduction!
```

---

## 8. Limitations of MapReduce

### 8.1 Expressivity Limitations

#### Problem 1: Multi-Stage Computing is Complex

Real analytics pipelines often require multiple stages, but MapReduce only provides a single Map→Reduce step.

```
                                      ┌────────────┐
                                      │Analytics 1 │
┌───────────┐    ┌───────────┐    ┌───┴────────────┴───┐    ┌───────────┐
│PreProcess │───▶│ Transform │───▶│       Join         │───▶│ Visualize │
└───────────┘    └───────────┘    └───┬────────────┬───┘    └───────────┘
                                      │Analytics 2 │
                                      └────────────┘
```

**In MapReduce**:
- Each box = separate MapReduce job
- Each job reads from / writes to HDFS
- Must manage job dependencies manually
- Intermediate data written to disk between stages
- **Result**: Complex code, poor performance for iterative algorithms

#### Problem 2: Complex Code for Simple Transformations

Even simple operations require writing full map/reduce functions:

```python
# In a modern system (SQL or DataFrame):
result = data.filter(x > 5).groupBy(key).sum()

# In MapReduce: Need to write multiple classes, configure jobs, etc.
# Repetitive boilerplate code obscures the actual logic
```

#### Problem 3: Limited Support for Non-Text, Non-Static Data

- Originally designed for web crawl data (text files)
- Poor support for:
  - Streaming data
  - Graph data
  - Complex nested structures
  - Real-time updates

### 8.2 Performance Limitations

#### Problem 1: Iterative Algorithms

Many ML algorithms are iterative (repeat until convergence):

```
┌─────────────────────────────────────────────────────────────────┐
│                      Iterative Algorithm                        │
│                                                                 │
│  ┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐              │
│  │ MR 1 │────▶│ MR 2 │────▶│ MR 3 │────▶│ MR n │────▶ Done    │
│  └──────┘     └──────┘     └──────┘     └──────┘              │
│      │            │            │            │                   │
│      ▼            ▼            ▼            ▼                   │
│    HDFS         HDFS         HDFS         HDFS                  │
│  (write)      (read/write) (read/write) (read/write)            │
│                                                                 │
│  Each iteration: Read from disk → Process → Write to disk      │
│  No data reuse between iterations!                              │
└─────────────────────────────────────────────────────────────────┘
```

**Example: PageRank**
- Requires 10-50 iterations
- Each iteration reads/writes the entire graph
- Same data re-read from disk every iteration
- **Result**: Hours instead of minutes

#### Problem 2: Interactive Queries

- MapReduce has high job startup overhead
- Each query = new MapReduce job
- Not suitable for exploratory data analysis
- **Result**: Minutes per query instead of seconds

---

## 9. Latency and Bandwidth: Understanding the Numbers

This is **critical** for understanding why certain design decisions are made in distributed systems.

### 9.1 The Latency Numbers Every Programmer Should Know

| Operation | Time | Scaled Comparison |
|-----------|------|-------------------|
| **L1 cache reference** | 0.5 ns | 1 second |
| **L2 cache reference** | 7 ns | 14 seconds |
| **Main memory reference** | 100 ns | 3.3 minutes |
| **Send 1KB over 1Gbps network** | 10,000 ns (10 μs) | 5.5 hours |
| **Read 4KB randomly from SSD** | 150,000 ns (150 μs) | 3.5 days |
| **Read 1MB sequentially from memory** | 250,000 ns (250 μs) | 5.8 days |
| **Round trip within datacenter** | 500,000 ns (500 μs) | 11.6 days |
| **Read 1MB sequentially from SSD** | 1,000,000 ns (1 ms) | 23 days |
| **Send 1MB over 1Gbps network** | 8,250,000 ns (8.25 ms) | 190 days |
| **HDD disk seek** | 10,000,000 ns (10 ms) | 231 days |
| **Read 1MB sequentially from HDD** | 20,000,000 ns (20 ms) | 1.3 years |
| **Send packet CA → Netherlands → CA** | 150,000,000 ns (150 ms) | 9.5 years |

### 9.2 Visualizing the Scale

```
Latency Scale (Log Scale)

    0.5ns │█ L1 cache
          │
      7ns │██ L2 cache
          │
    100ns │███ Main memory
          │
          │                         14x slower
          │
    10μs  │████████████ Network (1KB)
          │
   150μs  │██████████████████████ SSD random read
          │
   250μs  │████████████████████████ Memory sequential (1MB)
          │
   500μs  │██████████████████████████████ Datacenter RTT
          │
     1ms  │████████████████████████████████ SSD sequential (1MB)
          │
   8.3ms  │████████████████████████████████████████ Network (1MB)
          │
    10ms  │██████████████████████████████████████████ HDD seek
          │
    20ms  │████████████████████████████████████████████ HDD seq (1MB)
          │
   150ms  │████████████████████████████████████████████████████ WAN RTT
          │
          └─────────────────────────────────────────────────────────────▶
               Nanoseconds        Microseconds        Milliseconds
```

### 9.3 Key Insights

#### Insight 1: Memory is MUCH faster than everything else

```
Reading 1MB:
  From Memory:  250 μs
  From SSD:     1 ms      (4x slower than memory)
  From HDD:     20 ms     (80x slower than memory)
  From Network: 8.25 ms   (33x slower than memory)
```

**Implication**: Keep frequently accessed data in memory!

#### Insight 2: Sequential access >> Random access

```
SSD:
  Random 4KB read:  150 μs
  Sequential 1MB:   1 ms (1,000 μs)
  
  Sequential reads 256 x 4KB = 1MB
  But random 256 reads × 150 μs = 38.4 ms
  
  Sequential is 38x faster!

HDD:
  Seek time:        10 ms
  Sequential 1MB:   20 ms
  
  Random read (seek + read 4KB): ~10 ms
  Sequential 1MB: 20 ms = 256 × 4KB
  
  Sequential is 128x faster!
```

**Implication**: Design for sequential access patterns!

#### Insight 3: Network is expensive

```
Sending data:
  1KB over network:  10 μs
  1MB over network:  8.25 ms
  
Compare to:
  1MB from memory:   250 μs
  
Network is 33x slower than memory for 1MB!
```

**Implication**: Minimize data movement across the network. This is why:
- MapReduce moves computation to data (not data to computation)
- Shuffle phase is the bottleneck
- Data locality is critical

#### Insight 4: Bandwidth of Memory >> Network >> Disk

```
Bandwidth comparison (approximate):

Memory:    ~25-100 GB/s
SSD:       ~0.5-7 GB/s
Network:   ~1-12.5 GB/s (10-100 Gbps)
HDD:       ~0.1-0.25 GB/s

Memory : Network : HDD = 100 : 10 : 1 (order of magnitude)
```

### 9.4 Why This Matters for MapReduce

The shuffle phase in MapReduce involves:
1. **Disk write** (mapper writes intermediate data): ~20 ms/MB
2. **Network transfer** (mapper → reducer): ~8.25 ms/MB
3. **Disk read** (reducer reads intermediate data): ~20 ms/MB
4. **Sort** (in memory or external): additional cost

**Total for 1MB of shuffle**: ~50+ ms

Compare to just processing 1MB in memory: ~250 μs

**Shuffle is 200x slower than in-memory processing!**

This is the fundamental reason why:
- Iterative algorithms are slow in MapReduce
- Spark's in-memory processing is revolutionary
- Combiners are essential for reducing shuffle

---

## 10. Why MapReduce Fails for Modern Workloads

### 10.1 The Core Problem: Disk-Based Processing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     MapReduce Execution Model                           │
│                                                                         │
│   HDFS ──Read──▶ Map ──Write──▶ Local Disk ──Read──▶ Reduce ──Write──▶ HDFS  │
│    │                                │                               │   │
│    ▼                                ▼                               ▼   ▼
│  Disk I/O                       Disk I/O                         Disk I/O
│                                                                         │
│   Every stage involves disk!                                            │
│   No data reuse between stages!                                         │
│   High latency for all operations!                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Summary of MapReduce Limitations

| Category | Limitation | Impact |
|----------|------------|--------|
| **Expressivity** | Only Map→Reduce pattern | Complex pipelines need multiple jobs |
| **Expressivity** | Verbose code | Simple operations require boilerplate |
| **Expressivity** | Text-centric | Poor support for graphs, streams, complex types |
| **Performance** | Disk-based | Every stage reads/writes disk |
| **Performance** | No data reuse | Iterative algorithms re-read data |
| **Performance** | High startup overhead | Not suitable for interactive queries |
| **Performance** | Shuffle bottleneck | Network I/O dominates for many workloads |

### 10.3 The Need for Something Better

This sets the stage for Apache Spark:
- In-memory processing (avoid disk between stages)
- Rich API (beyond Map→Reduce)
- Data reuse across operations (RDD caching)
- Low latency (interactive queries)
- Unified engine (batch, streaming, ML, graphs)

---

## Summary

### Key Takeaways from Lecture 2.1 Part 1:

1. **Big Data Processing Stack**: Storage → Management → Generic Processing → Specialized Processing

2. **Memory Hierarchy**: Understand the massive latency differences between cache, memory, SSD, HDD, and network

3. **MapReduce Model**: map(k,v) → list(k',v') followed by reduce(k', list(v')) → list(k'',v'')

4. **Shuffle is Expensive**: Network + disk I/O makes shuffle the bottleneck

5. **Latency Numbers Matter**: 
   - Memory: 100 ns
   - Network (1MB): 8 ms
   - Disk (1MB): 20 ms
   - Memory is 100-200x faster!

6. **MapReduce Limitations**: 
   - Disk-based (no in-memory reuse)
   - Single stage (complex for pipelines)
   - High latency (bad for iterative/interactive)

7. **This Sets the Stage for Spark**: In-memory, lazy evaluation, rich API

---

## References

1. Dean, J. and Ghemawat, S., "MapReduce: Simplified Data Processing on Large Clusters", USENIX OSDI, 2004
2. Zaharia, M., et al., "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing", USENIX NSDI, 2012
3. Latency Numbers Every Programmer Should Know: https://gist.github.com/jboner/2841832
4. Lin, J. and Dyer, C., "Data-Intensive Text Processing with MapReduce", Morgan & Claypool, 2010

# Lecture 2.1: Big Data Processing with Apache Spark (Part 2)

## DS256 - Scalable Systems for Data Science
### Module 2: Processing Large Volumes of Big Data

---

## 1. Understanding Latency and Bandwidth

Before diving into Spark, let's clarify two fundamental concepts that determine system performance: **Latency** and **Bandwidth**.

### 1.1 What is Latency?

**Latency** is the **time delay** between initiating a request and receiving the first response. Think of it as "how long do I have to wait before something starts happening?"

```
┌──────────────┐                              ┌──────────────┐
│   Request    │ ─────── Latency ──────────▶  │   Response   │
│   Initiated  │        (waiting time)        │   Starts     │
└──────────────┘                              └──────────────┘
                    
     t = 0                                    t = latency
```

**Analogy**: When you order food at a restaurant, latency is how long you wait before the first dish arrives at your table.

**Examples**:
- L1 cache access latency: 0.5 ns
- Main memory access latency: 100 ns
- SSD random read latency: 150 μs
- HDD seek latency: 10 ms
- Network round-trip (same datacenter): 0.5 ms
- Network round-trip (cross-continent): 150 ms

### 1.2 What is Bandwidth?

**Bandwidth** is the **rate** at which data can be transferred once the transfer has started. Think of it as "how much data can flow per second?"

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ═══════════════════════════════════════════════════▶   │
│                                                          │
│  Bandwidth = Amount of Data / Time = MB/s or GB/s        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Analogy**: If latency is how long you wait for the food to arrive, bandwidth is how many dishes per minute the kitchen can produce once they start cooking.

**Examples**:
- Main memory bandwidth: ~25-100 GB/s
- SSD sequential bandwidth: ~0.5-7 GB/s
- Network bandwidth (1 Gbps): ~125 MB/s
- Network bandwidth (10 Gbps): ~1.25 GB/s
- HDD sequential bandwidth: ~100-250 MB/s

### 1.3 Latency vs Bandwidth: The Key Difference

| Aspect | Latency | Bandwidth |
|--------|---------|-----------|
| **Measures** | Time to start | Rate of transfer |
| **Unit** | Time (ns, μs, ms) | Data/Time (MB/s, GB/s) |
| **Affected by** | Distance, processing overhead | Physical capacity of channel |
| **Analogy** | Time to first byte | Bytes per second |

### 1.4 Why Both Matter

For a complete data transfer:
```
Total Time = Latency + (Data Size / Bandwidth)
```

**Small transfers** are dominated by **latency**:
```
Transfer 1 KB over network:
  Latency: 500 μs (datacenter round-trip)
  Transfer: 1KB / 125 MB/s = 8 μs
  Total: ~508 μs  ← Latency dominates!
```

**Large transfers** are dominated by **bandwidth**:
```
Transfer 1 GB over network:
  Latency: 500 μs
  Transfer: 1GB / 125 MB/s = 8 seconds
  Total: ~8 seconds ← Bandwidth dominates!
```

### 1.5 The Critical Numbers (Revisited)

| Operation | Time | Category |
|-----------|------|----------|
| L1 cache reference | 0.5 ns | Memory |
| L2 cache reference | 7 ns | Memory |
| Main memory reference | 100 ns | Memory |
| Read 1MB from memory | 250 μs | Memory |
| Send 1KB over 1Gbps network | 10 μs | Network |
| SSD random read (4KB) | 150 μs | Storage |
| Datacenter round-trip | 500 μs | Network |
| Read 1MB from SSD | 1 ms | Storage |
| Send 1MB over 1Gbps network | 8.25 ms | Network |
| HDD seek | 10 ms | Storage |
| Read 1MB from HDD | 20 ms | Storage |
| Cross-continent round-trip | 150 ms | Network |

---

## 2. Bandwidth Comparison: Memory >> Network >> Disk

### 2.1 The Critical Insight

```
Bandwidth Comparison (Reading 1 MB):

Memory:     ████████████████████████████████████████ 4,000 MB/s (250 μs)
SSD:        ██████████                               1,000 MB/s (1 ms)
Network:    █████                                      121 MB/s (8.25 ms)
HDD:        ██                                          50 MB/s (20 ms)

Memory is 4x faster than SSD
Memory is 33x faster than Network
Memory is 80x faster than HDD
```

### 2.2 What This Means for System Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                         THE GOLDEN RULE                              │
│                                                                      │
│   Bandwidth of Memory  ≫  Network  ≫  Disk                          │
│                                                                      │
│   Therefore:                                                         │
│   1. Keep data in memory as much as possible                        │
│   2. Minimize network transfers                                      │
│   3. When disk is needed, use sequential access                     │
│   4. Move computation to data, not data to computation              │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 Why MapReduce is Slow

MapReduce violates this principle at every step:

```
MapReduce Flow:
                                                   
HDFS (Disk) ──Read──▶ Map ──Write──▶ Local Disk ──Read──▶ Shuffle ──Network──▶ 
                                                                              │
                      Write ◀──Disk── Reduce ◀──Read──Disk──────────────────┘

Every arrow = expensive I/O operation!
```

### 2.4 Why Spark is Faster

Spark keeps data in memory:

```
Spark Flow (with caching):

HDFS ──Read (once)──▶ Memory ──Transform──▶ Memory ──Transform──▶ Memory ──Action──▶ Result
                        ▲                                              │
                        └──────────────── Reuse! ──────────────────────┘

Data stays in memory between operations!
```

---

## 3. Failures and Performance Trade-offs

### 3.1 Mean Time Between Failures (MTBF)

Research on datacenter failures reveals:

> *"The MTBF across all data centers we investigate (with hundreds of thousands of servers) is only 6.8 minutes, while the MTBF in different data centers varies between 32 minutes and 390 minutes."*

| Cluster Size | MTBF |
|--------------|------|
| Full datacenter (100K+ servers) | 6.8 minutes |
| 1,000 servers | ~680 minutes (~11 hours) |
| 100 servers | ~6,800 minutes (~4.7 days) |

### 3.2 The Key Insight

For a typical Spark job running on 100 machines:
- Job duration: Usually minutes to hours
- MTBF: ~4.7 days

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   "Failures may be infrequent during the lifetime of an            │
│    application execution"                                            │
│                                                                      │
│   Therefore:                                                         │
│   → Optimize for PERFORMANCE first                                  │
│   → Handle failures through RECOVERY mechanisms (lineage)           │
│   → Don't pay for fault tolerance on every operation                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Spark's Approach

Instead of writing intermediate data to disk (like MapReduce does for fault tolerance), Spark:

1. **Keeps data in memory** → Maximum performance
2. **Tracks lineage** → Can recompute lost partitions if failure occurs
3. **Only checkpoints** when explicitly asked → User controls the trade-off

This is a fundamental design philosophy: **optimize for the common case (no failures), handle the rare case (failures) efficiently**.

---

## 4. From MapReduce to Spark

### 4.1 The Evolution

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Google's MapReduce                          │
│                                                                      │
│   • 2004: Original paper                                            │
│   • Programming Model: Map → Shuffle → Reduce                        │
│   • Disk-based: Write to disk after every stage                     │
│   • Limited: Only map and reduce operations                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Apache Hadoop                               │
│                                                                      │
│   • Open-source implementation of MapReduce                         │
│   • HDFS for distributed storage                                    │
│   • Widely adopted in industry                                      │
│   • Same limitations as Google's MapReduce                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Apache Spark                                │
│                                                                      │
│   • 2012: RDD paper (Zaharia et al., NSDI)                          │
│   • In-memory processing                                            │
│   • Rich API: Many transformations and actions                      │
│   • Lazy evaluation for optimization                                │
│   • Unified engine for batch, streaming, ML, graphs                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Key Differences

| Aspect | MapReduce | Spark |
|--------|-----------|-------|
| **Processing Model** | Disk-based | Memory-based |
| **Data Abstraction** | Files | RDDs (Resilient Distributed Datasets) |
| **Intermediate Data** | Written to HDFS | Kept in memory |
| **Operations** | Map, Reduce only | 80+ transformations/actions |
| **Iteration Support** | Poor (disk I/O each iteration) | Excellent (in-memory) |
| **Interactive Queries** | Poor (job startup overhead) | Excellent (keep data cached) |
| **Fault Tolerance** | Replication | Lineage-based recomputation |
| **Speed** | Baseline | 10-100x faster for iterative jobs |

---

## 5. Apache Spark

### 5.1 The Spark Ecosystem

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Higher-Level Abstractions                        │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐│
│  │  Spark SQL   │  │   Spark      │  │    MLlib     │  │  GraphX  ││
│  │              │  │  Streaming   │  │              │  │          ││
│  │  DataFrames  │  │  DStreams    │  │  ML Library  │  │  Graph   ││
│  │  SQL Queries │  │  Real-time   │  │  Algorithms  │  │  Analytics│
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Core Spark Engine                             │
│                                                                      │
│   • RDDs (Resilient Distributed Datasets)                           │
│   • Transformations & Actions                                        │
│   • Batch Processing                                                 │
│   • Task Scheduling & Memory Management                              │
│   • Fault Recovery (Lineage)                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Cluster Managers                              │
│                                                                      │
│          Standalone    │    YARN    │    Mesos    │    K8s          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Core Spark Engine

The foundation of Spark, providing:
- **RDDs**: The fundamental data abstraction
- **Transformations**: Operations that create new RDDs (lazy)
- **Actions**: Operations that trigger computation and return results
- **Batch Processing**: Processing large datasets efficiently

### 5.3 Higher-Level Abstractions

#### Spark SQL & DataFrames
- SQL-like queries on structured data
- DataFrames: RDDs with schema information
- Optimized execution via Catalyst optimizer

#### Spark Streaming
- Process live data streams
- Discretized Streams (DStreams): RDDs over time
- Near real-time processing

#### MLlib
- Machine learning library
- Classification, regression, clustering, collaborative filtering
- Scalable implementations of standard algorithms

#### GraphX
- Graph-parallel computation
- PageRank, connected components, triangle counting
- Built on RDDs for graphs

---

## 6. Spark: A Distributed Execution Engine

### 6.1 Key Components

Let's understand each component clearly:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLUSTER                                         │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         DRIVER NODE                                   │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                    DRIVER PROGRAM                            │   │   │
│   │   │                                                              │   │   │
│   │   │   ┌─────────────────────────┐    ┌───────────────────────┐   │   │   │
│   │   │   │     SparkContext        │    │   Local Variables     │   │   │   │
│   │   │   │   (Connection to        │    │   (Driver's Memory)   │   │   │   │
│   │   │   │    Cluster)             │    │                       │   │   │   │
│   │   │   └─────────────────────────┘    └───────────────────────┘   │   │   │
│   │   │                                                              │   │   │
│   │   │   Your main() function runs here                             │   │   │
│   │   └──────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                                    │ Distributes tasks                       │
│                                    ▼                                         │
│   ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐  │
│   │     WORKER NODE 1    │  │     WORKER NODE 2    │  │   WORKER NODE N  │  │
│   │  ┌────────────────┐  │  │  ┌────────────────┐  │  │ ┌──────────────┐ │  │
│   │  │   EXECUTOR     │  │  │  │   EXECUTOR     │  │  │ │   EXECUTOR   │ │  │
│   │  │                │  │  │  │                │  │  │ │              │ │  │
│   │  │ ┌────┐ ┌────┐  │  │  │  │ ┌────┐ ┌────┐  │  │  │ │ ┌────┐┌────┐│ │  │
│   │  │ │Task│ │Task│  │  │  │  │ │Task│ │Task│  │  │  │ │ │Task││Task││ │  │
│   │  │ └────┘ └────┘  │  │  │  │ └────┘ └────┘  │  │  │ │ └────┘└────┘│ │  │
│   │  │ ┌────┐ ┌────┐  │  │  │  │ ┌────┐ ┌────┐  │  │  │ │ ┌────┐┌────┐│ │  │
│   │  │ │Task│ │Task│  │  │  │  │ │Task│ │Task│  │  │  │ │ │Task││Task││ │  │
│   │  │ └────┘ └────┘  │  │  │  │ └────┘ └────┘  │  │  │ │ └────┘└────┘│ │  │
│   │  │                │  │  │  │                │  │  │ │              │ │  │
│   │  │  [Cache/Data]  │  │  │  │  [Cache/Data]  │  │  │ │ [Cache/Data] │ │  │
│   │  └────────────────┘  │  │  └────────────────┘  │  │ └──────────────┘ │  │
│   └──────────────────────┘  └──────────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Component Definitions

#### Driver Program
- **What it is**: Your main application that runs the Spark logic
- **Where it runs**: On a single node (driver node)
- **Contains**:
  - The `main()` function
  - SparkContext creation
  - RDD definitions and transformations
  - Actions that trigger computation
- **Memory**: Has its own local memory (cannot be too large)

#### SparkContext
- **What it is**: The entry point to Spark functionality
- **What it does**:
  - Represents connection to the Spark cluster
  - Coordinates the execution of tasks
  - Provides methods to create RDDs
- **Creation**: One per Spark application

#### Worker Node
- **What it is**: A physical machine in the cluster
- **What it does**:
  - Hosts one or more executors
  - Provides CPU, memory, and storage resources
- **Analogy**: A server in your datacenter

#### Executor
- **What it is**: A JVM process running on a worker node
- **What it does**:
  - Runs tasks assigned by the driver
  - Stores RDD partitions in memory or disk
  - Returns results to the driver
- **Properties**:
  - **Exclusive to one application**: Each app gets its own executors
  - **Long-running**: Lives for the entire duration of the application
  - **Has its own memory**: Configurable memory allocation

#### Task
- **What it is**: The smallest unit of work
- **What it does**: Executes a single operation on a single partition
- **Properties**:
  - Runs as a **thread** within an executor
  - One task per partition per stage
  - Can run in parallel across multiple executors

### 6.3 Concrete Example: 4-Node Cluster

Let's say we have a cluster with the following specification:

```
Cluster Configuration:
├── Driver Node (1 machine)
│   └── 8 cores, 32 GB RAM
│
└── Worker Nodes (3 machines, each)
    └── 16 cores, 64 GB RAM each
```

**Spark Configuration:**
```python
# Typical configuration for this cluster
spark = SparkSession.builder \
    .appName("MyApp") \
    .config("spark.executor.instances", 6) \      # 6 executors total
    .config("spark.executor.cores", 8) \          # 8 cores per executor
    .config("spark.executor.memory", "28g") \     # 28 GB per executor
    .config("spark.driver.memory", "8g") \        # 8 GB for driver
    .getOrCreate()
```

**What this gives us:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              ACTUAL DEPLOYMENT                           │
│                                                                          │
│  Driver Node (8 cores, 32 GB RAM)                                       │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Driver Program (8 GB)                                               │ │
│  │ - SparkContext                                                      │ │
│  │ - Your code runs here                                               │ │
│  │ - Collect() results come here                                       │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  Worker 1 (16 cores, 64 GB)    Worker 2 (16 cores, 64 GB)               │
│  ┌─────────────────────────┐   ┌─────────────────────────┐              │
│  │ Executor 1 (8c, 28GB)   │   │ Executor 3 (8c, 28GB)   │              │
│  │ ┌─────┐┌─────┐┌─────┐   │   │ ┌─────┐┌─────┐┌─────┐   │              │
│  │ │Task1││Task2││Task3│...│   │ │Task ││Task ││Task │...│              │
│  │ └─────┘└─────┘└─────┘   │   │ └─────┘└─────┘└─────┘   │              │
│  ├─────────────────────────┤   ├─────────────────────────┤              │
│  │ Executor 2 (8c, 28GB)   │   │ Executor 4 (8c, 28GB)   │              │
│  │ ┌─────┐┌─────┐┌─────┐   │   │ ┌─────┐┌─────┐┌─────┐   │              │
│  │ │Task ││Task ││Task │...│   │ │Task ││Task ││Task │...│              │
│  │ └─────┘└─────┘└─────┘   │   │ └─────┘└─────┘└─────┘   │              │
│  └─────────────────────────┘   └─────────────────────────┘              │
│                                                                          │
│  Worker 3 (16 cores, 64 GB)                                             │
│  ┌─────────────────────────┐                                            │
│  │ Executor 5 (8c, 28GB)   │                                            │
│  ├─────────────────────────┤   Maximum parallel tasks:                  │
│  │ Executor 6 (8c, 28GB)   │   6 executors × 8 cores = 48 tasks        │
│  └─────────────────────────┘                                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Summary:**
- **48 tasks** can run in parallel (6 executors × 8 cores)
- **168 GB** total executor memory (6 × 28 GB)
- **8 GB** driver memory for collecting results

---

## 7. Spark RDD (Resilient Distributed Dataset)

### 7.1 What is an RDD?

An **RDD** is Spark's fundamental data abstraction. It represents an **immutable, distributed collection of objects** that can be processed in parallel.

```
                                    RDD<Integer>
                                    ┌──────────┐
                                    │ 8975698  │
    Logical View                    │ 754843   │
    (As seen by Driver)             │ 866347   │
                                    │ 873876   │
                                    │ 45641    │
                                    │ 32764    │
                                    │ 23768423 │
                                    │ 364732   │
                                    │ 7586     │
                                    └──────────┘
                                          │
                                          │
          ┌───────────────────────────────┼───────────────────────────────┐
          │                               │                               │
          ▼                               ▼                               ▼
    ┌───────────┐                   ┌───────────┐                   ┌───────────┐
    │ Partition │                   │ Partition │                   │ Partition │
    │    P1     │                   │    P2     │                   │    P3     │
    │ ───────── │                   │ ───────── │                   │ ───────── │
    │  8975698  │                   │  873876   │                   │  23768423 │
    │  754843   │                   │  45641    │                   │  364732   │
    │  866347   │                   │  32764    │                   │  7586     │
    └───────────┘                   └───────────┘                   └───────────┘
          │                               │                               │
          ▼                               ▼                               ▼
      Worker A                        Worker B                        Worker C
    
    Physical Layout (Distributed across cluster)
```

### 7.2 Key Properties of RDDs

| Property | Meaning | Why It Matters |
|----------|---------|----------------|
| **Resilient** | Can be rebuilt if a partition is lost | Fault tolerance without replication |
| **Distributed** | Partitioned across multiple nodes | Parallel processing |
| **Dataset** | Collection of data elements | Work with data naturally |
| **Immutable** | Cannot be modified after creation | Enables lineage tracking |
| **Lazy** | Computed only when needed | Optimization opportunities |

### 7.3 RDD Characteristics

1. **Collection of homogeneous objects**
   - All elements have the same type
   - Can be any Python/Java/Scala object

2. **Distributed on workers**
   - Split into 1 or more partitions
   - Partitions stored on different machines

3. **Read-only & Immutable**
   - Cannot modify an existing RDD
   - Transformations create new RDDs

4. **Can be rebuilt**
   - Spark tracks how RDD was created (lineage)
   - Can recompute lost partitions

5. **Can be cached**
   - Keep in memory for reuse
   - Avoid recomputation

6. **MapReduce-like operations**
   - Parallel operations execute on workers
   - Driver coordinates execution

---

## 8. Creating and Operating on RDDs (PySpark)

### 8.1 Creating a SparkContext

In PySpark, you first need to create a SparkContext (or use SparkSession which includes it):

```python
# Method 1: Using SparkContext directly
from pyspark import SparkConf, SparkContext

conf = SparkConf().setMaster("local[*]").setAppName("My App")
sc = SparkContext(conf=conf)

# Method 2: Using SparkSession (Modern approach, Spark 2.0+)
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("local[*]") \
    .appName("My App") \
    .getOrCreate()

sc = spark.sparkContext  # Get SparkContext from SparkSession
```

**Master URL Options:**
| Master URL | Meaning |
|------------|---------|
| `local` | Run locally with 1 thread |
| `local[4]` | Run locally with 4 threads |
| `local[*]` | Run locally with all available cores |
| `spark://host:7077` | Connect to Spark cluster |
| `yarn` | Run on YARN cluster |

### 8.2 Creating RDDs

#### Method 1: Parallelize a Collection
```python
# Create RDD from a Python list
data = [1, 2, 3, 4, 5]
rdd = sc.parallelize(data)

# With explicit partitioning
rdd = sc.parallelize(data, numSlices=4)  # 4 partitions
```

**When to use**: Testing, prototyping, small datasets

#### Method 2: Load from External Storage
```python
# From a text file (local or HDFS)
lines = sc.textFile("README.md")

# From HDFS
lines = sc.textFile("hdfs://namenode:9000/user/data/file.txt")

# From multiple files
lines = sc.textFile("logs/*.txt")

# Whole text files (useful for small files)
files = sc.wholeTextFiles("data/")  # Returns (filename, content) pairs
```

**When to use**: Real-world data processing

### 8.3 Basic RDD Information

```python
# Number of partitions
rdd.getNumPartitions()

# First element
rdd.first()

# First n elements
rdd.take(5)

# Collect all elements to driver (be careful with large RDDs!)
rdd.collect()

# Count elements
rdd.count()
```

---

## 9. Passing Functions to Spark (PySpark)

Spark operations take functions as parameters. In Python, there are several ways to pass functions:

### 9.1 Lambda Functions (Inline)

Best for short, simple operations:

```python
# Filter lines containing "error"
errors = lines.filter(lambda line: "error" in line)

# Square each number
squared = nums.map(lambda x: x * x)

# Sum two numbers (for reduce)
total = nums.reduce(lambda x, y: x + y)
```

### 9.2 Named Functions

Better for complex logic or reuse:

```python
def contains_error(line):
    """Check if line contains error."""
    return "error" in line.lower()

def parse_log(line):
    """Parse a log line into components."""
    parts = line.split(",")
    return {
        "timestamp": parts[0],
        "level": parts[1],
        "message": parts[2]
    }

# Use named functions
errors = lines.filter(contains_error)
parsed = lines.map(parse_log)
```

### 9.3 Important Caution: Serialization

When passing functions, Spark serializes (pickles) them to send to executors.

**Problem: Referencing class members**

```python
# DON'T DO THIS - serializes entire object!
class LogProcessor:
    def __init__(self, keyword):
        self.keyword = keyword
    
    def process(self, rdd):
        # This references self.keyword, so entire object is serialized!
        return rdd.filter(lambda line: self.keyword in line)
```

**Solution: Extract to local variable**

```python
# DO THIS - only serializes the string
class LogProcessor:
    def __init__(self, keyword):
        self.keyword = keyword
    
    def process(self, rdd):
        # Extract to local variable first
        keyword = self.keyword
        return rdd.filter(lambda line: keyword in line)
```

---

## 10. Programming with RDDs: Transformations and Actions

### 10.1 Two Types of Operations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  TRANSFORMATIONS                    ACTIONS                              │
│  ─────────────────                  ───────                              │
│  • Create new RDD                   • Return result to driver            │
│  • Lazy (not executed immediately)  • Trigger computation                │
│  • Return: RDD                      • Return: Value or write to storage │
│                                                                          │
│  Examples:                          Examples:                            │
│  - map()                            - count()                            │
│  - filter()                         - collect()                          │
│  - flatMap()                        - first()                            │
│  - union()                          - take(n)                            │
│  - distinct()                       - reduce()                           │
│  - groupByKey()                     - saveAsTextFile()                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Common Transformations

#### map(func)
Apply function to each element, return new RDD:

```python
nums = sc.parallelize([1, 2, 3, 4])
squared = nums.map(lambda x: x * x)
# Result: [1, 4, 9, 16]
```

#### filter(func)
Keep elements that satisfy predicate:

```python
nums = sc.parallelize([1, 2, 3, 4, 5, 6])
evens = nums.filter(lambda x: x % 2 == 0)
# Result: [2, 4, 6]
```

#### flatMap(func)
Map then flatten (one-to-many):

```python
lines = sc.parallelize(["hello world", "hi there"])
words = lines.flatMap(lambda line: line.split(" "))
# Result: ["hello", "world", "hi", "there"]
```

```
                map() vs flatMap()
                
Input RDD:  ["hello world", "hi"]

map(split):     [["hello", "world"], ["hi"]]  ← Nested lists
flatMap(split): ["hello", "world", "hi"]      ← Flattened
```

#### distinct()
Remove duplicates (expensive - requires shuffle):

```python
nums = sc.parallelize([1, 2, 2, 3, 3, 3])
unique = nums.distinct()
# Result: [1, 2, 3]
```

#### union(otherRDD)
Combine two RDDs:

```python
rdd1 = sc.parallelize([1, 2, 3])
rdd2 = sc.parallelize([3, 4, 5])
combined = rdd1.union(rdd2)
# Result: [1, 2, 3, 3, 4, 5]  (duplicates preserved!)
```

### 10.3 Common Actions

#### collect()
Return all elements as a list (use carefully!):

```python
result = rdd.collect()  # Returns Python list
```

⚠️ **Warning**: Only use when result fits in driver memory!

#### count()
Count number of elements:

```python
num_lines = lines.count()
```

#### first()
Return first element:

```python
first_line = lines.first()
```

#### take(n)
Return first n elements:

```python
top_5 = lines.take(5)
```

#### reduce(func)
Aggregate elements using associative function:

```python
nums = sc.parallelize([1, 2, 3, 4, 5])
total = nums.reduce(lambda x, y: x + y)
# Result: 15
```

#### saveAsTextFile(path)
Write to text file:

```python
rdd.saveAsTextFile("hdfs://path/to/output")
```

### 10.4 Transformation Summary Table

| Function | Purpose | Example | Result |
|----------|---------|---------|--------|
| `map(f)` | Apply f to each element | `rdd.map(x => x+1)` | {2,3,4,4} |
| `filter(f)` | Keep elements where f is true | `rdd.filter(x => x!=1)` | {2,3,3} |
| `flatMap(f)` | Map then flatten | `rdd.flatMap(x => x.to(3))` | {1,2,3,2,3,3,3} |
| `distinct()` | Remove duplicates | `rdd.distinct()` | {1,2,3} |
| `union(other)` | Combine RDDs | `rdd.union(other)` | {1,2,3,3,4,5} |
| `intersection(other)` | Common elements | `rdd.intersection(other)` | {3} |
| `subtract(other)` | Remove elements in other | `rdd.subtract(other)` | {1,2} |

---

## 11. Lazy Evaluation

### 11.1 What is Lazy Evaluation?

**Lazy evaluation** means that Spark does not execute transformations immediately. Instead, it records them as a **computation graph** and only executes when an **action** is called.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          LAZY EVALUATION                                 │
│                                                                          │
│   When you write:                                                        │
│   ─────────────────                                                      │
│   lines = sc.textFile("data.txt")   # Nothing happens yet!              │
│   errors = lines.filter(...)         # Still nothing!                   │
│   count = errors.count()             # NOW everything executes          │
│                     ▲                                                    │
│                     │                                                    │
│              Action triggers execution                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Why Lazy Evaluation?

1. **Optimization Opportunity**: Spark can analyze entire computation graph before executing
2. **Reduce Passes**: Combine operations to minimize data reads
3. **Avoid Unnecessary Work**: Skip computation if result not needed

### 11.3 Detailed Example

Let's trace through what happens step by step:

```python
# Step 1: Load data
lines = sc.textFile("README.md")
# Spark does NOT read the file yet!
# It just records: "When needed, read from README.md"

# Step 2: Filter
pythonLines = lines.filter(lambda line: "Python" in line)
# Spark does NOT filter yet!
# It just records: "After reading, apply this filter"

# Step 3: Count (ACTION!)
count = pythonLines.count()
# NOW Spark:
#   1. Reads the file
#   2. Applies the filter
#   3. Counts the results
#   4. Returns the count to driver
```

**Visualization:**

```
          Transformations (Lazy)                    Action (Triggers)
          ─────────────────────                    ─────────────────
          
sc.textFile("README.md") ──▶ filter(contains Python) ──▶ count()
         │                          │                        │
         │                          │                        │
         ▼                          ▼                        ▼
    "Plan to read"          "Plan to filter"        "Execute everything!"
                                                            │
                                                            ▼
                                                    Return: 2
```

### 11.4 Benefits Demonstrated

**Without Lazy Evaluation (Hypothetical):**
```python
lines = sc.textFile("README.md")     # Read entire file into memory
filtered = lines.filter(...)          # Create new collection
first = filtered.first()              # Only need 1 element, but processed all!
```

**With Lazy Evaluation (Actual Spark):**
```python
lines = sc.textFile("README.md")     # Plan to read
filtered = lines.filter(...)          # Plan to filter
first = filtered.first()              # Execute - stop after finding first match!
```

Spark reads the file **only until it finds the first matching line**!

---

## 12. Lineage Graph

### 12.1 What is a Lineage Graph?

A **Lineage Graph** (also called **RDD Dependency Graph**) is a record of all the transformations used to build an RDD. Spark maintains this graph internally.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          LINEAGE GRAPH                                   │
│                                                                          │
│   Purpose:                                                               │
│   1. Enable lazy evaluation - know what to compute when action called   │
│   2. Enable fault tolerance - recompute lost partitions                  │
│   3. Enable optimization - combine operations, prune unnecessary work   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 12.2 Example Lineage Graph

```python
# Code
inputRDD = sc.textFile("log.txt")
errorsRDD = inputRDD.filter(lambda x: "error" in x)
warningsRDD = inputRDD.filter(lambda x: "warning" in x)
badLinesRDD = errorsRDD.union(warningsRDD)
count = badLinesRDD.count()
```

**Lineage Graph:**

```
                        sc.textFile("log.txt")
                               inputRDD
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
           filter("error")             filter("warning")
              errorsRDD                  warningsRDD
                    │                           │
                    └─────────────┬─────────────┘
                                  │
                                  ▼
                              union()
                            badLinesRDD
                                  │
                                  ▼
                              count()
                               Result
```

### 12.3 Lineage for Fault Tolerance

If a partition is lost (machine failure), Spark uses the lineage to recompute just that partition:

```
         Partition P1                    P1 on Machine A fails!
         ────────────                    ─────────────────────────
         
         textFile ──▶ filter ──▶ P1     Machine A crashes, P1 lost
                                        
                                        Spark checks lineage:
                                        "P1 came from filtering a portion
                                         of the input file"
                                        
                                        Recompute on Machine B:
         textFile ──▶ filter ──▶ P1'    Read that portion, apply filter
```

**Key Insight**: Only the lost partition is recomputed, not the entire RDD!

### 12.4 Viewing Lineage

In PySpark, you can view the lineage using `toDebugString()`:

```python
>>> lines = sc.textFile("README.md")
>>> pythonLines = lines.filter(lambda x: "Python" in x)
>>> print(pythonLines.toDebugString())

(2) PythonRDD[2] at RDD at PythonRDD.scala:53 []
 |  README.md MapPartitionsRDD[1] at textFile at NativeMethodAccessorImpl.java:0 []
 |  README.md HadoopRDD[0] at textFile at NativeMethodAccessorImpl.java:0 []
```

This shows:
- `HadoopRDD[0]`: Reading from file system
- `MapPartitionsRDD[1]`: Converting to strings
- `PythonRDD[2]`: Applying the Python filter function

---

## Summary

### Key Takeaways from Lecture 2.1 Part 2:

1. **Latency vs Bandwidth**:
   - Latency = time to start (waiting time)
   - Bandwidth = rate of transfer (throughput)
   - Memory >> Network >> Disk in both metrics

2. **Design for Performance**:
   - Failures are rare during job execution
   - Optimize for common case (no failures)
   - Use lineage for fault recovery when needed

3. **Spark Architecture**:
   - Driver: Runs your main program
   - SparkContext: Connection to cluster
   - Workers: Machines that do the work
   - Executors: JVM processes that run tasks
   - Tasks: Smallest unit of work (threads)

4. **RDD Fundamentals**:
   - Immutable, distributed collections
   - Partitioned across cluster
   - Two operations: Transformations (lazy) and Actions (eager)

5. **Lazy Evaluation**:
   - Transformations are recorded, not executed
   - Actions trigger execution
   - Enables optimization and fault tolerance

6. **Lineage Graph**:
   - Records how RDDs were derived
   - Enables recomputation of lost partitions
   - Foundation of Spark's fault tolerance

---

## References

1. Zaharia, M., et al., "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing", USENIX NSDI, 2012
2. Karau, H., Konwinski, A., Wendell, P., Zaharia, M., "Learning Spark", O'Reilly, 2015 (Chapters 2 & 3)
3. Latency Numbers Every Programmer Should Know: https://gist.github.com/jboner/2841832
4. Spark Programming Guide: https://spark.apache.org/docs/latest/rdd-programming-guide.html

# Lecture 2.2: Spark Transformations and Actions

## DS256 - Scalable Systems for Data Science
### Module 2: Processing Large Volumes of Big Data

---

## 1. Common Transformations: Element-wise

In Spark, **transformations** are operations that create a new RDD from an existing one. Element-wise transformations apply a function to each element of the RDD independently. Let's explore the three fundamental element-wise transformations.

### 1.1 Filter

**filter(func)** applies conditional logic to each element and returns a new RDD containing only elements where the function returns `true`.

```
┌─────────────────────────────────────────────────────────────────────┐
│                          FILTER TRANSFORMATION                       │
│                                                                      │
│   Input RDD: [1, 2, 3, 4, 5, 6]                                     │
│                                                                      │
│   Transformation: filter(lambda x: x > 3)                           │
│                                                                      │
│   Process:                                                           │
│   1 → false (discarded)                                             │
│   2 → false (discarded)                                             │
│   3 → false (discarded)                                             │
│   4 → true  (kept)                                                  │
│   5 → true  (kept)                                                  │
│   6 → true  (kept)                                                  │
│                                                                      │
│   Output RDD: [4, 5, 6]                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**
- User logic (lambda function) returns **true** or **false**
- If true, input element **copies to output** RDD
- If false, input element is **omitted** from output
- **Output RDD type is the same as input RDD type**

**Example:**

```python
# Create RDD
nums = sc.parallelize([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])

# Filter even numbers
evens = nums.filter(lambda x: x % 2 == 0)
# Result: [2, 4, 6, 8, 10]

# Filter numbers greater than 5
large_nums = nums.filter(lambda x: x > 5)
# Result: [6, 7, 8, 9, 10]

# Filter with string RDD
lines = sc.textFile("log.txt")
errors = lines.filter(lambda line: "ERROR" in line)
# Output: Only lines containing "ERROR"
```

### 1.2 Map

**map(func)** applies user logic to each element and returns exactly **one output for each input** item. The output type can be different from the input type.

```
┌─────────────────────────────────────────────────────────────────────┐
│                            MAP TRANSFORMATION                        │
│                                                                      │
│   Input RDD: [1, 2, 3, 4]                                           │
│                                                                      │
│   Transformation: map(lambda x: x * x)                              │
│                                                                      │
│   Process:                                                           │
│   1 → 1                                                             │
│   2 → 4                                                             │
│   3 → 9                                                             │
│   4 → 16                                                            │
│                                                                      │
│   Output RDD: [1, 4, 9, 16]                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**
- Applies user logic to **each element**
- Returns **exactly one output per input**
- **Output type can differ** from input type
- Can perform any user operation (parsing, computation, web fetching, etc.)

**Examples:**

```python
# Example 1: Simple arithmetic transformation
nums = sc.parallelize([1, 2, 3, 4])
squared = nums.map(lambda x: x * x)
# Result: [1, 4, 9, 16]

# Example 2: Type transformation (int → string)
nums = sc.parallelize([1, 2, 3, 4])
strings = nums.map(lambda x: "Number: " + str(x))
# Result: ["Number: 1", "Number: 2", "Number: 3", "Number: 4"]

# Example 3: Parsing strings
logs = sc.parallelize([
    "2024-01-01,ERROR,Connection failed",
    "2024-01-01,INFO,Server started",
    "2024-01-02,WARN,High memory usage"
])

parsed = logs.map(lambda line: {
    'date': line.split(',')[0],
    'level': line.split(',')[1],
    'message': line.split(',')[2]
})
# Result: List of dictionaries with structured data
```

### 1.3 FlatMap

**flatMap(func)** applies user logic to each element and returns **zero or more output items** for each input. The results are then flattened into a single RDD.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FLATMAP TRANSFORMATION                       │
│                                                                      │
│   Input RDD: ["hello world", "hi there"]                            │
│                                                                      │
│   Transformation: flatMap(lambda line: line.split())                │
│                                                                      │
│   Process:                                                           │
│   "hello world" → ["hello", "world"]                                │
│   "hi there"    → ["hi", "there"]                                   │
│                                                                      │
│   Flatten: [["hello", "world"], ["hi", "there"]]                    │
│            ↓                                                         │
│            ["hello", "world", "hi", "there"]                        │
│                                                                      │
│   Output RDD: ["hello", "world", "hi", "there"]                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**
- Applies user logic to **each element**
- Returns **zero or more outputs** per input
- Results are **automatically flattened**
- **Output type can differ** from input type

**Examples:**

```python
# Example 1: Split sentences into words
lines = sc.parallelize(["hello world", "hi there", "goodbye"])
words = lines.flatMap(lambda line: line.split())
# Result: ["hello", "world", "hi", "there", "goodbye"]

# Example 2: Generate range of numbers
nums = sc.parallelize([1, 2, 3])
expanded = nums.flatMap(lambda x: range(1, x + 1))
# Input:  [1, 2, 3]
# 1 → [1]
# 2 → [1, 2]
# 3 → [1, 2, 3]
# Result: [1, 1, 2, 1, 2, 3]

# Example 3: Zero outputs (filtering effect)
nums = sc.parallelize([1, 2, 3, 4, 5])
result = nums.flatMap(lambda x: [x] if x > 2 else [])
# 1 → []
# 2 → []
# 3 → [3]
# 4 → [4]
# 5 → [5]
# Result: [3, 4, 5]
```

### 1.4 Map vs FlatMap: The Key Difference

```
                   map() vs flatMap()

Input RDD:  ["hello world", "hi"]

Using map(split):
  "hello world" → ["hello", "world"]
  "hi"          → ["hi"]

  Result: [["hello", "world"], ["hi"]]  ← Nested lists (RDD of lists)

Using flatMap(split):
  "hello world" → ["hello", "world"]
  "hi"          → ["hi"]

  Flatten: [["hello", "world"], ["hi"]]
           ↓
           ["hello", "world", "hi"]  ← Flattened (RDD of strings)
```

### 1.5 Filter Using Different Transformations

Let's see how to achieve filtering using different transformations:

**Goal:** Filter items greater than 10 from `[5, 15, 8, 12, 3, 20]`

```python
# Original RDD
rdd = sc.parallelize([5, 15, 8, 12, 3, 20])

# Method 1: Using filter() - THE CORRECT WAY
filtered = rdd.filter(lambda item: item > 10)
# Result: [15, 12, 20] ✓ Clean and correct

# Method 2: Using flatMap() - WORKS BUT AWKWARD
filtered = rdd.flatMap(lambda item: [item] if item > 10 else [])
# Process:
# 5  → [] (empty list flattened away)
# 15 → [15]
# 8  → []
# 12 → [12]
# 3  → []
# 20 → [20]
# Result: [15, 12, 20] ✓ Works, but unnecessary complexity

# Method 3: Using map() - WRONG! DON'T DO THIS
filtered = rdd.map(lambda item: item if item > 10 else None)
# Process:
# 5  → None
# 15 → 15
# 8  → None
# 12 → 12
# 3  → None
# 20 → 20
# Result: [None, 15, None, 12, None, 20] ✗ CONTAINS None VALUES!
```

**Why map() fails for filtering:**
- map() **must return exactly one item** for each input
- You can't "skip" an item in map()
- Returning `None` still creates an output element
- Result contains `None` values mixed with actual data

**Comparison Table:**

| Method | Result | Pros | Cons |
|--------|--------|------|------|
| **filter()** | `[15, 12, 20]` | Clean, clear intent, efficient | None |
| **flatMap()** | `[15, 12, 20]` | Works correctly | Unnecessarily complex for filtering |
| **map()** | `[None, 15, None, 12, None, 20]` | N/A | WRONG - includes None values! |

**Best Practice:** Always use `filter()` for filtering. Use `flatMap()` when you genuinely need zero-to-many transformations.

---

## 2. Common Transformations: Pseudo Set Operations

Spark provides set-like operations on RDDs. These are called "pseudo" set operations because RDDs can contain duplicates (unlike mathematical sets).

### 2.1 Distinct

**distinct()** removes duplicate elements from an RDD.

```python
nums = sc.parallelize([1, 2, 2, 3, 3, 3, 4, 5, 5])
unique = nums.distinct()
# Result: [1, 2, 3, 4, 5]
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DISTINCT OPERATION                           │
│                                                                      │
│   Input RDD:  [1, 2, 2, 3, 3, 3, 4, 5, 5]                           │
│                                                                      │
│   Process:                                                           │
│   1. Hash each element by value                                     │
│   2. Shuffle to group identical values                              │
│   3. Keep one copy of each unique value                             │
│                                                                      │
│   Output RDD: [1, 2, 3, 4, 5]                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**⚠️ Important:**
- `distinct()` is **expensive** - it requires a **shuffle operation**
- All data must be sent across the network to check for duplicates
- Use sparingly on large datasets

### 2.2 Union

**union(otherRDD)** concatenates two RDDs into one. **Duplicates are NOT removed**.

```python
rdd1 = sc.parallelize([1, 2, 3])
rdd2 = sc.parallelize([3, 4, 5])
combined = rdd1.union(rdd2)
# Result: [1, 2, 3, 3, 4, 5]  ← Note: 3 appears twice!
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                          UNION OPERATION                             │
│                                                                      │
│   RDD1: [1, 2, 3]                                                   │
│   RDD2: [3, 4, 5]                                                   │
│                                                                      │
│   union(RDD1, RDD2):                                                │
│   Simply concatenates: [1, 2, 3] + [3, 4, 5]                        │
│                                                                      │
│   Result: [1, 2, 3, 3, 4, 5]  ← Duplicates preserved!               │
│                                                                      │
│   To get unique values:                                              │
│   rdd1.union(rdd2).distinct() → [1, 2, 3, 4, 5]                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Points:**
- Very **cheap operation** (no shuffle needed)
- Just concatenates partitions from both RDDs
- Duplicates are **preserved**
- To remove duplicates, chain with `.distinct()`

### 2.3 Intersection

**intersection(otherRDD)** returns only elements that appear in **both** RDDs. Duplicates are removed.

```python
rdd1 = sc.parallelize([1, 2, 3, 3, 4])
rdd2 = sc.parallelize([3, 4, 5, 6])
common = rdd1.intersection(rdd2)
# Result: [3, 4]  ← Common elements, duplicates removed
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                      INTERSECTION OPERATION                          │
│                                                                      │
│   RDD1: [1, 2, 3, 3, 4]                                             │
│   RDD2: [3, 4, 5, 6]                                                │
│                                                                      │
│   Process:                                                           │
│   1. Find elements in both RDDs                                     │
│   2. Remove duplicates                                               │
│                                                                      │
│   Common elements: 3 (appears in both), 4 (appears in both)         │
│                                                                      │
│   Result: [3, 4]                                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**⚠️ Important:**
- **Expensive operation** - requires shuffle
- Automatically removes duplicates
- Use only when necessary

### 2.4 Subtract

**subtract(otherRDD)** returns elements from the first RDD that are **not** in the second RDD.

```python
rdd1 = sc.parallelize([1, 2, 3, 3, 4])
rdd2 = sc.parallelize([3, 4, 5])
difference = rdd1.subtract(rdd2)
# Result: [1, 2]  ← Elements in rdd1 but not in rdd2
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SUBTRACT OPERATION                            │
│                                                                      │
│   RDD1: [1, 2, 3, 3, 4]                                             │
│   RDD2: [3, 4, 5]                                                   │
│                                                                      │
│   Process: Keep elements from RDD1 that don't appear in RDD2        │
│                                                                      │
│   Check each element in RDD1:                                       │
│   1 → Not in RDD2 → Keep                                            │
│   2 → Not in RDD2 → Keep                                            │
│   3 → In RDD2 → Remove (both copies)                                │
│   4 → In RDD2 → Remove                                              │
│                                                                      │
│   Result: [1, 2]                                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.5 Cartesian Product

**cartesian(otherRDD)** returns **all possible pairs** combining each element from the first RDD with each element from the second.

```python
rdd1 = sc.parallelize([1, 2])
rdd2 = sc.parallelize(['a', 'b', 'c'])
pairs = rdd1.cartesian(rdd2)
# Result: [(1,'a'), (1,'b'), (1,'c'), (2,'a'), (2,'b'), (2,'c')]
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CARTESIAN PRODUCT OPERATION                       │
│                                                                      │
│   RDD1: [1, 2]                                                      │
│   RDD2: ['a', 'b', 'c']                                             │
│                                                                      │
│   All combinations:                                                  │
│   1 × 'a' → (1, 'a')                                                │
│   1 × 'b' → (1, 'b')                                                │
│   1 × 'c' → (1, 'c')                                                │
│   2 × 'a' → (2, 'a')                                                │
│   2 × 'b' → (2, 'b')                                                │
│   2 × 'c' → (2, 'c')                                                │
│                                                                      │
│   Result: [(1,'a'), (1,'b'), (1,'c'), (2,'a'), (2,'b'), (2,'c')]    │
│                                                                      │
│   Size: |RDD1| × |RDD2| = 2 × 3 = 6 elements                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**⚠️ Warning:**
- **Extremely expensive** for large RDDs!
- Output size = size(RDD1) × size(RDD2)
- Example: 1000 × 1000 = 1,000,000 elements
- Avoid unless absolutely necessary

### 2.6 Sample

**sample(withReplacement, fraction, seed)** returns a random sample of the RDD.

```python
nums = sc.parallelize(range(100))

# Sample approximately 10% without replacement
sample1 = nums.sample(False, 0.1, seed=42)
# Result: ~10 elements, no duplicates

# Sample approximately 50% with replacement
sample2 = nums.sample(True, 0.5, seed=42)
# Result: ~50 elements, may contain duplicates
```

**Parameters:**
- **withReplacement**:
  - `False`: Each element can appear at most once
  - `True`: Elements can appear multiple times
- **fraction**: Approximate fraction to sample (0.0 to 1.0)
- **seed**: Random seed for reproducibility

**Important Notes:**
- Sample size is **approximate**, not exact
  - `fraction * count` gives expected size, not guaranteed size
- Same `seed` produces same sample **only if RDD hasn't changed**
- Sampling done **per partition** with same fraction
- Useful for:
  - Testing on subset of data
  - Stratified sampling
  - Approximate data exploration

**Examples:**

```python
# Example: Varying sample sizes
data = sc.parallelize(range(1000))

# Small sample
small = data.sample(False, 0.01, 123)  # ~10 items

# Medium sample
medium = data.sample(False, 0.1, 123)  # ~100 items

# Large sample
large = data.sample(False, 0.5, 123)   # ~500 items

# With replacement (for bootstrap sampling)
bootstrap = data.sample(True, 1.0, 123)  # ~1000 items, some duplicated
```

### 2.7 Summary Table: Set Operations

| Operation | Duplicates? | Shuffle? | Output Size | Use Case |
|-----------|-------------|----------|-------------|----------|
| **distinct()** | Removed | Yes | ≤ input size | Remove duplicates |
| **union(rdd2)** | Preserved | No | size1 + size2 | Combine datasets |
| **intersection(rdd2)** | Removed | Yes | ≤ min(size1, size2) | Find common elements |
| **subtract(rdd2)** | Removed | Yes | ≤ size1 | Remove elements |
| **cartesian(rdd2)** | N/A | Yes | size1 × size2 | All combinations |
| **sample(...)** | Depends | No | ≈ fraction × size | Random sampling |

---

## 3. Common Actions: reduce()

Actions trigger computation and return results to the driver or save to storage. Let's start with one of the most important actions: **reduce()**.

### 3.1 What is reduce()?

**reduce(mergeFunc)** combines elements of an RDD using an **aggregation function** to produce a single result.

```python
nums = sc.parallelize([1, 2, 3, 4, 5])
product = nums.reduce(lambda x, y: x * y)
# Result: 120  (1 * 2 * 3 * 4 * 5)
```

### 3.2 Requirements for mergeFunc

The merge function **must be**:

1. **Commutative**: `f(a, b) = f(b, a)`
   - Order of operands doesn't matter

2. **Associative**: `f(f(a, b), c) = f(a, f(b, c))`
   - Order of operations doesn't matter

**Why these requirements?**
- Spark processes partitions **in parallel**
- Combines results in **arbitrary order**
- Function must produce same result regardless of order

**Examples:**

```python
# ✓ Valid functions (commutative and associative):
sum:     lambda x, y: x + y
product: lambda x, y: x * y
max:     lambda x, y: max(x, y)
min:     lambda x, y: min(x, y)

# ✗ Invalid functions:
subtract: lambda x, y: x - y  # NOT commutative: 5-3 ≠ 3-5
divide:   lambda x, y: x / y  # NOT commutative or associative
concat:   lambda x, y: x + y  # For strings, NOT commutative
```

### 3.3 How reduce() Works: Two-Level Reduction

reduce() performs aggregation in **two phases**:

1. **Within each partition** (parallel) - combine elements in each partition
2. **Across partitions** (parallel or sequential) - combine partition results

```
┌─────────────────────────────────────────────────────────────────────┐
│                    REDUCE EXECUTION FLOW                             │
│                                                                      │
│  Input: [2, 5, 3, 1, 6, 4, 7] distributed across 2 partitions      │
│  Function: lambda x, y: x * y  (multiplication)                     │
│                                                                      │
│  ┌──────────────────────┐              ┌──────────────────────┐    │
│  │    Partition 1       │              │    Partition 2       │    │
│  │   [2, 5, 3, 1]       │              │     [6, 4, 7]        │    │
│  └──────────────────────┘              └──────────────────────┘    │
│           │                                      │                  │
│           │ PHASE 1: Reduce within partition    │                  │
│           │         (runs in parallel)           │                  │
│           ▼                                      ▼                  │
│  ┌──────────────────────┐              ┌──────────────────────┐    │
│  │  2 * 5 = 10          │              │   6 * 4 = 24         │    │
│  │ 10 * 3 = 30          │              │  24 * 7 = 168        │    │
│  │ 30 * 1 = 30          │              │                      │    │
│  │                      │              │                      │    │
│  │ Partition result: 30 │              │ Partition result:168 │    │
│  └──────────────────────┘              └──────────────────────┘    │
│           │                                      │                  │
│           └──────────────┬───────────────────────┘                  │
│                          │                                          │
│           PHASE 2: Reduce across partitions                         │
│                          │                                          │
│                          ▼                                          │
│                  ┌──────────────┐                                   │
│                  │  30 * 168    │                                   │
│                  │              │                                   │
│                  │ Final: 5040  │                                   │
│                  └──────────────┘                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.4 Detailed Example: Product Calculation

Let's trace through a product calculation step-by-step:

```python
nums = sc.parallelize([2, 5, 3, 1, 6, 4, 7], 2)  # 2 partitions
product = nums.reduce(lambda x, y: x * y)
```

**Distribution:**
- Partition 1: `[2, 5, 3, 1]`
- Partition 2: `[6, 4, 7]`

**Phase 1: Within Partitions (Parallel)**

**Partition 1:**
```
Step 1: 2 * 5 = 10
Step 2: 10 * 3 = 30
Step 3: 30 * 1 = 30
Partition 1 result: 30
```

**Partition 2:**
```
Step 1: 6 * 4 = 24
Step 2: 24 * 7 = 168
Partition 2 result: 168
```

**Phase 2: Across Partitions**
```
30 * 168 = 5040
Final result: 5040
```

### 3.5 More reduce() Examples

```python
# Example 1: Sum
nums = sc.parallelize([1, 2, 3, 4, 5])
total = nums.reduce(lambda x, y: x + y)
# Result: 15

# Example 2: Maximum
nums = sc.parallelize([23, 45, 12, 67, 34])
maximum = nums.reduce(lambda x, y: max(x, y))
# Result: 67

# Example 3: Minimum
nums = sc.parallelize([23, 45, 12, 67, 34])
minimum = nums.reduce(lambda x, y: min(x, y))
# Result: 12

# Example 4: String concatenation (BE CAREFUL!)
words = sc.parallelize(['hello', 'world', 'spark'])
# This might produce different results depending on partition order!
result = words.reduce(lambda x, y: x + ' ' + y)
# Possible results: "hello world spark" or "world hello spark" etc.
# String concatenation is NOT commutative for order-dependent results!
```

### 3.6 Common Pitfalls

**Pitfall 1: Non-commutative functions**

```python
# WRONG: Subtraction is not commutative
nums = sc.parallelize([10, 5, 3], 2)
# Partition 1: [10, 5] → 10 - 5 = 5
# Partition 2: [3] → 3
# Across: 5 - 3 = 2  OR  3 - 5 = -2  ← Unpredictable!
result = nums.reduce(lambda x, y: x - y)  # Don't do this!
```

**Pitfall 2: Functions with side effects**

```python
# WRONG: Reduce function should be pure
count = 0
def bad_reduce(x, y):
    global count
    count += 1  # Side effect!
    return x + y

result = nums.reduce(bad_reduce)  # Count will be unpredictable
```

**Best Practices:**
- ✓ Use functions that are commutative and associative
- ✓ Keep functions pure (no side effects)
- ✓ Test your reduce function with different orderings
- ✗ Don't use reduce for operations that depend on order
- ✗ Don't modify external state in reduce functions

---

## 4. Common Actions: aggregate()

While `reduce()` is powerful, it has a limitation: the output type must be the same as the input type. What if you want to compute multiple statistics at once, like both sum and count? That's where **aggregate()** comes in.

### 4.1 What is aggregate()?

**aggregate(zeroValue, mergeValue, mergeCombiners)** is a more general version of reduce() that allows the accumulator type to differ from the element type.

**Signature:**
```python
rdd.aggregate(
    zeroValue,      # Initial accumulator value
    mergeValue,     # Combine accumulator with each element within partition
    mergeCombiners  # Combine accumulators across partitions
)
```

### 4.2 Three Parameters Explained

1. **zeroValue**: The initial value for the accumulator
   - Used as starting point for each partition
   - Type can be different from RDD element type

2. **mergeValue(accumulator, element)**: Combines an element into the accumulator
   - Called for each element in a partition
   - Returns updated accumulator

3. **mergeCombiners(acc1, acc2)**: Combines two accumulators
   - Called to combine results from different partitions
   - Returns merged accumulator

### 4.3 How aggregate() Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AGGREGATE EXECUTION FLOW                          │
│                                                                      │
│  Phase 1: Within Each Partition                                     │
│  ─────────────────────────────────                                  │
│                                                                      │
│  1. Start with zeroValue                                            │
│  2. For each element: acc = mergeValue(acc, element)                │
│  3. Result: One accumulator per partition                           │
│                                                                      │
│  Phase 2: Across Partitions                                         │
│  ─────────────────────────────                                      │
│                                                                      │
│  1. Start with zeroValue                                            │
│  2. For each partition result: acc = mergeCombiners(acc, part_acc)  │
│  3. Result: Single final accumulator                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.4 Example 1: Simple Sum (Same as reduce)

Let's start with a simple example where aggregate() does the same thing as reduce():

```python
nums = sc.parallelize([2, 5, 3, 1, 6, 4, 7], 2)  # 2 partitions

# Using aggregate for sum
total = nums.aggregate(
    0,                          # zeroValue: Start with 0
    lambda acc, val: acc + val, # mergeValue: Add value to accumulator
    lambda acc1, acc2: acc1 + acc2  # mergeCombiners: Add accumulators
)
# Result: 28
```

**Execution Trace:**

**Partition 1: [2, 5, 3, 1]**
```
Start:     acc = 0 (zeroValue)
Element 2: acc = 0 + 2 = 2
Element 5: acc = 2 + 5 = 7
Element 3: acc = 7 + 3 = 10
Element 1: acc = 10 + 1 = 11
Partition 1 result: 11
```

**Partition 2: [6, 4, 7]**
```
Start:     acc = 0 (zeroValue)
Element 6: acc = 0 + 6 = 6
Element 4: acc = 6 + 4 = 10
Element 7: acc = 10 + 7 = 17
Partition 2 result: 17
```

**Combine Partitions:**
```
Start:             acc = 0 (zeroValue)
Partition 1 (11):  acc = 0 + 11 = 11
Partition 2 (17):  acc = 11 + 17 = 28
Final result: 28
```

### 4.5 Example 2: Computing Average (Different Accumulator Type)

Here's where aggregate() shines - computing the average requires tracking both sum and count:

```python
nums = sc.parallelize([2, 5, 3, 1, 6, 4, 7], 2)  # 2 partitions

# Accumulator type: (sum, count) tuple
sumCount = nums.aggregate(
    (0, 0),  # zeroValue: (sum=0, count=0)

    # mergeValue: Add value to sum, increment count
    lambda acc, val: (acc[0] + val, acc[1] + 1),

    # mergeCombiners: Add sums and counts
    lambda acc1, acc2: (acc1[0] + acc2[0], acc1[1] + acc2[1])
)

# Calculate average
average = sumCount[0] / float(sumCount[1])
# Result: 28 / 7 = 4.0
```

**Detailed Execution Trace:**

```
┌─────────────────────────────────────────────────────────────────────┐
│              AGGREGATE WITH (SUM, COUNT) ACCUMULATOR                │
│                                                                      │
│  Partition 1: [2, 5, 3, 1]                                          │
│  ──────────────────────────                                         │
│                                                                      │
│  Initial:   (sum=0, count=0)                                        │
│  + value 2: (0+2, 0+1)   = (2, 1)                                   │
│  + value 5: (2+5, 1+1)   = (7, 2)                                   │
│  + value 3: (7+3, 2+1)   = (10, 3)                                  │
│  + value 1: (10+1, 3+1)  = (11, 4)                                  │
│                                                                      │
│  Partition 1 Result: (11, 4)  ← sum=11, count=4                     │
│                                                                      │
│  ─────────────────────────────────────────────────────────────      │
│                                                                      │
│  Partition 2: [6, 4, 7]                                             │
│  ──────────────────────                                             │
│                                                                      │
│  Initial:   (sum=0, count=0)                                        │
│  + value 6: (0+6, 0+1)   = (6, 1)                                   │
│  + value 4: (6+4, 1+1)   = (10, 2)                                  │
│  + value 7: (10+7, 2+1)  = (17, 3)                                  │
│                                                                      │
│  Partition 2 Result: (17, 3)  ← sum=17, count=3                     │
│                                                                      │
│  ─────────────────────────────────────────────────────────────      │
│                                                                      │
│  Combine Partitions:                                                │
│  ──────────────────                                                 │
│                                                                      │
│  Initial:        (sum=0, count=0)                                   │
│  + Part 1 (11,4): (0+11, 0+4)   = (11, 4)                           │
│  + Part 2 (17,3): (11+17, 4+3)  = (28, 7)                           │
│                                                                      │
│  Final Result: (28, 7)                                              │
│                                                                      │
│  Average: 28 / 7 = 4.0                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.6 Visual: Incremental Evaluation

```
                    WITHIN PARTITION 1              WITHIN PARTITION 2

zeroValue:             (0, 0)                          (0, 0)
                         ↓                               ↓
mergeValue:           (0, 0)                          (0, 0)
+ element 2            + 2                            + 6
                     ────────                        ────────
                     (2, 1)                          (6, 1)
                         ↓                               ↓
                     (2, 1)                          (6, 1)
+ element 5/4          + 5                            + 4
                     ────────                        ────────
                     (7, 2)                          (10, 2)
                         ↓                               ↓
                     (7, 2)                          (10, 2)
+ element 3/7          + 3                            + 7
                     ────────                        ────────
                     (10, 3)                         (17, 3)
                         ↓                               ↓
                     (10, 3)                         (17, 3)
+ element 1            + 1                             [done]
                     ────────                        ────────
                     (11, 4)                         (17, 3)
                         │                               │
                         └───────────┬───────────────────┘
                                     │
                        ACROSS PARTITIONS (mergeCombiners)
                                     │
                                     ▼
                    zeroValue:    (0, 0)
                                     ↓
                    + Part 1:   (0+11, 0+4) = (11, 4)
                                     ↓
                    + Part 2:   (11+17, 4+3) = (28, 7)
                                     ↓
                             Final: (28, 7)
                                     ↓
                           Average: 28/7 = 4.0
```

### 4.7 Example 3: Complex Example from Slides

From the lecture slides, here's an example that counts string lengths:

```python
strs = sc.parallelize(['ababab', 'ab', 'abcd'])

# Count total length of all strings
totalLength = strs.aggregate(
    0,                          # zeroValue: start with 0
    lambda acc, val: acc + len(val),  # mergeValue: add string length
    lambda acc1, acc2: acc1 + acc2    # mergeCombiners: add lengths
)
# Result: 6 + 2 + 4 = 12
```

### 4.8 aggregate() vs reduce()

| Feature | reduce() | aggregate() |
|---------|----------|-------------|
| **Accumulator Type** | Same as element type | Can be different |
| **Use Case** | Simple aggregations | Complex aggregations |
| **Example** | Sum, max, min | Average, statistics, custom objects |
| **Parameters** | 1 function | 3 parameters (zeroValue + 2 functions) |
| **Flexibility** | Limited | High |

### 4.9 When to Use aggregate()

Use **aggregate()** when:
- ✓ You need to compute multiple values at once (e.g., sum AND count)
- ✓ Your accumulator type differs from element type
- ✓ You need more control over the aggregation process

Use **reduce()** when:
- ✓ Simple aggregation with same input/output type
- ✓ Function is commutative and associative
- ✓ You don't need intermediate accumulator state

### 4.10 Important Notes

**Note 1: zeroValue is used twice!**
```
Within partitions: Start with zeroValue
Across partitions: Start with zeroValue again!
```

This means zeroValue should be the identity element for your operation:
- For addition: 0
- For multiplication: 1
- For tuple (sum, count): (0, 0)

**Note 2: Functions must be associative**

Both `mergeValue` and `mergeCombiners` should be associative to ensure consistent results regardless of processing order.

---

## 5. Common Actions: Other Actions

Beyond reduce() and aggregate(), Spark provides many other useful actions. Let's explore them with a concrete example.

### 5.1 Example Dataset

We'll use this sample RDD distributed across 3 partitions:

```python
# Create RDD with 3 partitions
rdd = sc.parallelize([
    3, 2, 1,      # Partition 1
    4, 1, 6,      # Partition 2
    2, 5, 6, 7, 5 # Partition 3
], 3)

# Visual representation:
# P1: [3, 2, 1]
# P2: [4, 1, 6]
# P3: [2, 5, 6, 7, 5]
```

### 5.2 count()

Returns the total number of elements in the RDD.

```python
count = rdd.count()
# Result: 12
# Explanation: 3 + 3 + 5 = 12 elements total
```

### 5.3 collect()

Returns **all elements** of the RDD to the driver as a Python list.

```python
all_elements = rdd.collect()
# Result: [3, 2, 1, 4, 1, 6, 2, 5, 6, 7, 5]
```

**⚠️ WARNING:**
```
┌─────────────────────────────────────────────────────────────────────┐
│                       collect() WARNING!                             │
│                                                                      │
│  • collect() brings ALL data to the driver                          │
│  • If RDD is large, driver will run out of memory and crash!        │
│  • Only use when you KNOW the result is small                       │
│                                                                      │
│  Safe:    rdd.filter(...).take(10)        # Get small sample        │
│  UNSAFE:  huge_rdd.collect()              # May crash driver!       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.4 take(n)

Returns the **first n elements** from the RDD, reading from the **fewest partitions possible**.

```python
first_8 = rdd.take(8)
# Result: [3, 2, 1, 4, 1, 6, 2, 5]
# Explanation: Takes from P1 (3 items), P2 (3 items), P3 (2 items)
```

**Characteristics:**
- **Not evenly sampled**: Takes from first partitions until n elements collected
- **Not ordered**: Order depends on partition order
- **Efficient**: Doesn't read entire RDD

```
┌──────────────────────────────────────────────────────────────┐
│                    take(8) Execution                          │
│                                                               │
│  P1: [3, 2, 1]       → Take all 3 (total: 3)                │
│  P2: [4, 1, 6]       → Take all 3 (total: 6)                │
│  P3: [2, 5, 6, 7, 5] → Take first 2 (total: 8) STOP!        │
│                                                               │
│  Result: [3, 2, 1, 4, 1, 6, 2, 5]                            │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 5.5 takeOrdered(n, key=None)

Returns the **first n elements in sorted order** (ascending by default).

```python
# Default: ascending order
first_4_sorted = rdd.takeOrdered(4)
# Result: [1, 1, 2, 2]
# Explanation: Sorts entire RDD, returns first 4

# With custom key (descending)
top_4 = rdd.takeOrdered(4, key=lambda x: -x)
# Result: [7, 6, 6, 5]
# Explanation: Negative key reverses order
```

**Execution:**
1. Spark collects data from all partitions
2. Sorts the data
3. Returns first n elements

### 5.6 top(n)

Returns the **largest n elements** in descending order.

```python
top_4 = rdd.top(4)
# Result: [7, 6, 6, 5]
# Explanation: Returns 4 largest values in descending order
```

**Relationship:**
```python
rdd.top(n) == rdd.takeOrdered(n, key=lambda x: -x)
# Both return largest n elements in descending order
```

### 5.7 takeSample(withReplacement, num, seed=None)

Returns a **random sample** of exactly `num` elements, **evenly sampled** from all partitions.

```python
# Without replacement (no duplicates)
sample_no_replace = rdd.takeSample(False, 6, seed=42)
# Result: [1, 5, 3, 1, 6, 5]
# Explanation: 6 items sampled uniformly from all partitions

# With replacement (duplicates allowed)
sample_with_replace = rdd.takeSample(True, 6, seed=42)
# Result: [1, 5, 2, 2, 6, 5]
# Explanation: Same item can be picked multiple times
```

**Key Differences from take():**

| Feature | take(n) | takeSample(n) |
|---------|---------|---------------|
| **Sampling** | From first partitions | From all partitions uniformly |
| **Deterministic** | Yes (always same) | No (random, unless seed provided) |
| **Count** | Approximately n | Exactly n |
| **Replacement** | N/A | Can choose with/without |

### 5.8 forEach(func)

Applies a function to each element **for side effects only**. Results are NOT returned to driver.

```python
# Example: Save to database (side effect)
def save_to_db(value):
    # Imagine this writes to a database
    print(f"Saving {value} to database")

rdd.forEach(save_to_db)
# No return value
# Each element processed on executors, not returned to driver
```

**Use Cases:**
- Writing to external database
- Updating external counters
- Logging (on executors)

**⚠️ Important:**
- `forEach` runs on **executors**, not driver
- Cannot return values to driver
- Good for distributed side effects

### 5.9 countByValue()

Returns a dictionary mapping each **unique value** to its **count**.

```python
counts = rdd.countByValue()
# Result: {1: 2, 2: 2, 3: 1, 4: 1, 5: 2, 6: 2, 7: 1}
# Explanation:
#   1 appears 2 times
#   2 appears 2 times
#   3 appears 1 time
#   ... and so on
```

**Execution:**
1. Groups elements by value
2. Counts occurrences
3. Returns dictionary to driver

**⚠️ Warning:** Result is brought to driver. If many unique values, may cause memory issues.

### 5.10 Summary Table: All Actions

Using our example RDD: `[3, 2, 1, 4, 1, 6, 2, 5, 6, 7, 5]`

| Action | Result | Notes |
|--------|--------|-------|
| `count()` | `12` | Total number of elements |
| `collect()` | `[3,2,1,4,1,6,2,5,6,7,5]` | ⚠️ All data to driver |
| `take(8)` | `[3,2,1,4,1,6,2,5]` | First partitions, not sorted |
| `takeOrdered(4)` | `[1,1,2,2]` | Smallest 4, ascending |
| `top(4)` | `[7,6,6,5]` | Largest 4, descending |
| `takeSample(False,6)` | `[1,5,3,1,6,5]` | Random uniform sample |
| `takeSample(True,6)` | `[1,5,2,2,6,5]` | With possible duplicates |
| `forEach(print)` | None | Side effects only |
| `countByValue()` | `{1:2, 2:2, 3:1, 4:1, 5:2, 6:2, 7:1}` | Frequency map |

---

## 6. RDD Persistence

One of Spark's key advantages over MapReduce is the ability to **cache RDDs in memory** for reuse across multiple operations. This is crucial for iterative algorithms and interactive queries.

### 6.1 Why Persistence is Needed

**Problem: Recomputation on Every Action**

```python
# Load data
logs = sc.textFile("hdfs://server/logs/*.txt")

# Transform
errors = logs.filter(lambda line: "ERROR" in line)
warnings = logs.filter(lambda line: "WARN" in line)

# Actions
error_count = errors.count()      # Reads and filters logs
warning_count = warnings.count()  # Reads and filters logs AGAIN!
```

**Without persistence:**
```
Action 1 (errors.count()):
  Read logs from HDFS → Filter for ERROR → Count

Action 2 (warnings.count()):
  Read logs from HDFS AGAIN → Filter for WARN → Count
```

**With persistence:**
```python
logs = sc.textFile("hdfs://server/logs/*.txt")
logs.persist()  # Mark for caching

errors = logs.filter(lambda line: "ERROR" in line)
warnings = logs.filter(lambda line: "WARN" in line)

error_count = errors.count()      # Reads logs, CACHES in memory
warning_count = warnings.count()  # REUSES cached logs!
```

```
Action 1 (errors.count()):
  Read logs from HDFS → Cache in memory → Filter for ERROR → Count

Action 2 (warnings.count()):
  Read logs from MEMORY (fast!) → Filter for WARN → Count
```

### 6.2 How Persistence Works

**Step 1: Mark RDD for persistence**
```python
rdd.persist()  # or rdd.cache()
```
- Does **nothing immediately** (lazy!)
- Just marks RDD for caching

**Step 2: First action triggers caching**
```python
count = rdd.count()
```
- RDD is computed
- Results stored in memory (or disk, depending on level)

**Step 3: Subsequent actions reuse cached data**
```python
sample = rdd.take(10)
```
- Reads from cache instead of recomputing
- Much faster!

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PERSISTENCE LIFECYCLE                           │
│                                                                      │
│  rdd.persist()  →  NO IMMEDIATE EFFECT (lazy)                       │
│       │                                                              │
│       ▼                                                              │
│  rdd.count()    →  COMPUTE + CACHE                                  │
│       │             ├─ Read from source                              │
│       │             ├─ Apply transformations                         │
│       │             ├─ Store in memory/disk                          │
│       │             └─ Return count                                  │
│       ▼                                                              │
│  rdd.take(10)   →  READ FROM CACHE (fast!)                          │
│       │             └─ No recomputation needed                       │
│       ▼                                                              │
│  rdd.collect()  →  READ FROM CACHE (fast!)                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 Levels of Persistence

Spark offers multiple storage levels with different trade-offs:

| Storage Level | Memory | Disk | Serialized | Recompute | Use Case |
|---------------|--------|------|------------|-----------|----------|
| **MEMORY_ONLY** | ✓ | ✗ | ✗ | If evicted | Default, fastest |
| **MEMORY_ONLY_SER** | ✓ | ✗ | ✓ | If evicted | Save memory |
| **MEMORY_AND_DISK** | ✓ | ✓ | ✗ | Never | Spill to disk |
| **MEMORY_AND_DISK_SER** | ✓ | ✓ | ✓ | Never | Save memory + reliable |
| **DISK_ONLY** | ✗ | ✓ | ✓ | Never | When memory scarce |

**Detailed Explanations:**

**1. MEMORY_ONLY** (Default)
```python
rdd.persist()  # Same as cache()
# or
rdd.persist(StorageLevel.MEMORY_ONLY)
```
- Stores RDD as **deserialized Java objects** in memory
- **Fastest** access (no deserialization overhead)
- If not enough memory, partitions **not cached** (recomputed on demand)
- Best for: Hot data that fits in memory

**2. MEMORY_ONLY_SER**
```python
from pyspark import StorageLevel
rdd.persist(StorageLevel.MEMORY_ONLY_SER)
```
- Stores RDD as **serialized objects** (more compact)
- Saves memory at cost of CPU (deserialization overhead)
- Good for: Large RDDs that barely fit in memory

**3. MEMORY_AND_DISK**
```python
rdd.persist(StorageLevel.MEMORY_AND_DISK)
```
- Stores in memory if possible
- **Spills to disk** if memory insufficient
- Never recomputes (always cached somewhere)
- Best for: Important RDDs that may not fit in memory

**4. MEMORY_AND_DISK_SER**
```python
rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)
```
- Serialized in memory, spills to disk if needed
- Most memory-efficient reliable option
- Best for: Large important RDDs with limited memory

**5. DISK_ONLY**
```python
rdd.persist(StorageLevel.DISK_ONLY)
```
- Stores only on disk
- Slower than memory but faster than recomputation
- Best for: Very large RDDs with expensive computations

### 6.4 LRU Eviction Policy

When memory is full, Spark uses **Least Recently Used (LRU)** eviction:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         LRU EVICTION                                 │
│                                                                      │
│  Memory Full?                                                        │
│      │                                                               │
│      ├─ NO  → Cache new partition                                   │
│      │                                                               │
│      └─ YES → Evict least recently used partition                   │
│              → Cache new partition                                   │
│                                                                      │
│  Evicted partition:                                                  │
│  ├─ MEMORY_ONLY → Recompute if needed later                         │
│  └─ MEMORY_AND_DISK → Already on disk, read from there              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.5 Fault Tolerance and Persistence

**What happens if a node fails?**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FAULT TOLERANCE WITH CACHING                      │
│                                                                      │
│  Scenario: Node with cached partition fails                         │
│                                                                      │
│  Storage Level           Action Taken                                │
│  ──────────────          ────────────                                │
│                                                                      │
│  MEMORY_ONLY         →  Recompute lost partition using lineage      │
│  MEMORY_ONLY_SER     →  Recompute lost partition                    │
│  MEMORY_AND_DISK     →  Read from disk on another node              │
│                         (if partition was replicated)                │
│                         Otherwise recompute                          │
│  DISK_ONLY           →  Read from disk replica                      │
│                                                                      │
│  Key Insight: Lineage provides fault tolerance                      │
│               No need for expensive replication                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.6 Manual Unpersist

You can manually remove an RDD from cache:

```python
# Persist RDD
rdd.persist()

# Use RDD multiple times
count1 = rdd.count()
count2 = rdd.filter(...).count()

# Done with RDD, free memory
rdd.unpersist()
```

**When to unpersist:**
- RDD no longer needed
- Need memory for other RDDs
- Cache is cluttered with old data

### 6.7 cache() vs persist()

```python
# These are equivalent:
rdd.cache()
rdd.persist()
rdd.persist(StorageLevel.MEMORY_ONLY)

# cache() is just shorthand for persist with default level
```

### 6.8 Example: Iterative Algorithm

**Without caching (slow):**
```python
data = sc.textFile("large_file.txt")

for i in range(10):
    # Each iteration reads file from disk!
    count = data.filter(lambda x: some_condition(x, i)).count()
    print(f"Iteration {i}: {count}")
# Time: Very slow (10 disk reads)
```

**With caching (fast):**
```python
data = sc.textFile("large_file.txt")
data.cache()  # Mark for caching

for i in range(10):
    # First iteration: read from disk and cache
    # Iterations 2-10: read from memory!
    count = data.filter(lambda x: some_condition(x, i)).count()
    print(f"Iteration {i}: {count}")
# Time: Much faster (1 disk read, 9 memory reads)
```

### 6.9 Best Practices

**When to cache:**
- ✓ RDD used multiple times
- ✓ Expensive transformations before the RDD
- ✓ Iterative algorithms (ML, graph processing)
- ✓ Interactive queries on same dataset

**When NOT to cache:**
- ✗ RDD used only once
- ✗ Very large RDDs that won't fit in memory (unless MEMORY_AND_DISK)
- ✗ Simple transformations (faster to recompute)

**Choosing storage level:**
```
┌─────────────────────────────────────────────────────────────────────┐
│               CHOOSING THE RIGHT STORAGE LEVEL                       │
│                                                                      │
│  Question 1: Does RDD fit in memory?                                │
│  ├─ YES → Use MEMORY_ONLY (default)                                 │
│  └─ NO  → Question 2: Is recomputation expensive?                   │
│           ├─ YES → Use MEMORY_AND_DISK                              │
│           └─ NO  → Don't cache or use MEMORY_ONLY                   │
│                                                                      │
│  Question 3: Need to save memory?                                   │
│  └─ YES → Use serialized versions (*_SER)                           │
│                                                                      │
│  Question 4: Can tolerate recomputation loss?                       │
│  ├─ YES → Use MEMORY_ONLY                                           │
│  └─ NO  → Use MEMORY_AND_DISK or DISK_ONLY                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Working with Key/Value Pairs

So far, we've worked with RDDs containing simple values. But many real-world operations require us to work with **key/value pairs** - for example, grouping by key, joining datasets, or computing per-key statistics. Spark provides a special type of RDD called a **Pair RDD** for this purpose.

### 7.1 What is a Pair RDD?

A **Pair RDD** is an RDD where each element is a tuple of **(key, value)**.

```python
# Example Pair RDD
pairRDD = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("apple", 2),
    ("orange", 7),
    ("banana", 4)
])

# Type: RDD[(String, Int)]
# Each element is a 2-tuple (key, value)
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                          PAIR RDD STRUCTURE                          │
│                                                                      │
│   Regular RDD:     [1, 2, 3, 4, 5]                                  │
│   (single values)                                                    │
│                                                                      │
│   Pair RDD:        [("key1", val1), ("key2", val2), ...]            │
│   (key-value)      ─────┬──────────                                 │
│                         │                                            │
│                   Tuple (K, V)                                       │
│                                                                      │
│   Characteristics:                                                   │
│   • Keys are NOT necessarily unique                                 │
│   • Single value per key-value pair (not a collection)              │
│   • Keys can appear multiple times with different values            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Key Characteristics

**Important:** Unlike dictionaries/maps in programming languages:
- **Keys are NOT distinct** - the same key can appear multiple times
- **Each (key, value) is a single pair** - not key → list of values (that comes from transformations like groupByKey)

```python
# In a Pair RDD, this is valid:
[("apple", 5), ("apple", 2), ("apple", 8)]
# Three separate pairs, same key "apple"

# This is NOT how Pair RDDs are stored:
{"apple": [5, 2, 8]}  # This is what you get from groupByKey()
```

### 7.3 Why Use Pair RDDs?

Pair RDDs unlock powerful operations:

| Operation | Purpose | Example |
|-----------|---------|---------|
| **Aggregation** | Combine values by key | Sum sales per product |
| **Grouping** | Collect all values per key | Group users by country |
| **Joins** | Combine two datasets by key | Join user profiles with purchases |
| **Sorting** | Sort by keys | Alphabetically sort products |
| **Counting** | Count occurrences per key | Word frequency |

### 7.4 Creating Pair RDDs

**Method 1: Using map() with tuples**

```python
# From regular RDD
data = sc.parallelize(["apple 5", "banana 3", "apple 2"])

# Extract key-value pairs
pairs = data.map(lambda line: (line.split()[0], int(line.split()[1])))
# Result: [("apple", 5), ("banana", 3), ("apple", 2)]
```

**Method 2: Using map() with custom logic**

```python
# Create user activity pairs
users = sc.parallelize([
    "user1:login",
    "user2:purchase",
    "user1:logout",
    "user3:login"
])

# Create (user, action) pairs
user_actions = users.map(lambda x: tuple(x.split(":")))
# Result: [("user1", "login"), ("user2", "purchase"),
#          ("user1", "logout"), ("user3", "login")]
```

**Method 3: From structured data**

```python
# From log entries
logs = sc.parallelize([
    "2024-01-01 ERROR Connection failed",
    "2024-01-01 INFO Server started",
    "2024-01-02 ERROR Timeout",
    "2024-01-02 WARN High memory"
])

# Create (severity, message) pairs
log_pairs = logs.map(lambda line:
    (line.split()[1], line)
)
# Result: [("ERROR", "2024-01-01 ERROR Connection failed"),
#          ("INFO", "2024-01-01 INFO Server started"), ...]
```

**Method 4: Using keyBy()**

```python
# Create pairs by extracting key from value
words = sc.parallelize(["hello", "world", "hi", "hadoop"])

# Key by first letter
letter_pairs = words.keyBy(lambda word: word[0])
# Result: [("h", "hello"), ("w", "world"), ("h", "hi"), ("h", "hadoop")]
```

### 7.5 Examples from Lecture Slides

**Python:**
```python
# Create Pair RDD
pairs = sc.parallelize([("a", 1), ("b", 2), ("c", 3)])
```

**Java (from slides):**
```java
// Create Pair RDD in Java
JavaPairRDD<String, Integer> pairs =
    sc.parallelizePairs(Arrays.asList(
        new Tuple2<>("a", 1),
        new Tuple2<>("b", 2),
        new Tuple2<>("c", 3)
    ));
```

### 7.6 Operations Available on Pair RDDs

Pair RDDs support **all regular RDD operations** plus additional transformations and actions:

**Regular RDD operations (still work):**
```python
pairs = sc.parallelize([("a", 1), ("b", 2), ("a", 3)])

# Regular transformations
filtered = pairs.filter(lambda kv: kv[1] > 1)  # [("b", 2), ("a", 3)]
mapped = pairs.map(lambda kv: (kv[0], kv[1] * 2))  # [("a", 2), ("b", 4), ("a", 6)]

# Regular actions
count = pairs.count()  # 3
first = pairs.first()  # ("a", 1)
```

**Special Pair RDD operations (covered in next sections):**
- Transformations: mapValues, reduceByKey, groupByKey, join, etc.
- Actions: countByKey, collectAsMap, lookup, etc.

### 7.7 Accessing Keys and Values

Since each element is a tuple, you can access components:

```python
pairs = sc.parallelize([("apple", 5), ("banana", 3)])

# Method 1: Index access
keys = pairs.map(lambda kv: kv[0])      # ["apple", "banana"]
values = pairs.map(lambda kv: kv[1])    # [5, 3]

# Method 2: Unpacking in lambda
keys = pairs.map(lambda (k, v): k)      # ["apple", "banana"]
values = pairs.map(lambda (k, v): v)    # [5, 3]

# Method 3: Built-in methods
keys = pairs.keys()                      # ["apple", "banana"]
values = pairs.values()                  # [5, 3]
```

---

## 8. Transformations on Pair RDDs

Pair RDDs expose powerful transformations designed specifically for key-value data. Let's explore the fundamental ones.

### 8.1 mapValues()

**mapValues(func)** applies a function to **only the values**, keeping keys unchanged.

```python
pairs = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("apple", 2)
])

# Double all values
doubled = pairs.mapValues(lambda v: v * 2)
# Result: [("apple", 10), ("banana", 6), ("apple", 4)]
```

**Why use mapValues() instead of map()?**

```python
# Using map() - must manually preserve keys
result = pairs.map(lambda (k, v): (k, v * 2))

# Using mapValues() - cleaner and clearer intent
result = pairs.mapValues(lambda v: v * 2)
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                         mapValues() BEHAVIOR                         │
│                                                                      │
│   Input:  [("apple", 5), ("banana", 3), ("apple", 2)]               │
│                                                                      │
│   mapValues(lambda v: v * 2)                                         │
│                                                                      │
│   Process:                                                           │
│   ("apple", 5)   →  ("apple", 5*2)   = ("apple", 10)                │
│   ("banana", 3)  →  ("banana", 3*2)  = ("banana", 6)                │
│   ("apple", 2)   →  ("apple", 2*2)   = ("apple", 4)                 │
│                                                                      │
│   Output: [("apple", 10), ("banana", 6), ("apple", 4)]              │
│                                                                      │
│   Key Insight: Keys remain unchanged!                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**More Examples:**

```python
# Example 1: Type transformation
users = sc.parallelize([("user1", "25"), ("user2", "30")])
users_int = users.mapValues(lambda age: int(age))
# Result: [("user1", 25), ("user2", 30)]

# Example 2: String manipulation
products = sc.parallelize([("P1", "apple"), ("P2", "banana")])
upper_products = products.mapValues(lambda name: name.upper())
# Result: [("P1", "APPLE"), ("P2", "BANANA")]

# Example 3: Complex transformation
scores = sc.parallelize([("Alice", 85), ("Bob", 92), ("Charlie", 78)])
grades = scores.mapValues(lambda score:
    "A" if score >= 90 else "B" if score >= 80 else "C"
)
# Result: [("Alice", "B"), ("Bob", "A"), ("Charlie", "C")]
```

### 8.2 reduceByKey()

**reduceByKey(func)** combines values with the **same key** using an associative reduce function.

```python
pairs = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("apple", 2),
    ("orange", 7),
    ("banana", 4)
])

# Sum values by key
totals = pairs.reduceByKey(lambda v1, v2: v1 + v2)
# Result: [("apple", 7), ("banana", 7), ("orange", 7)]
```

**Execution Flow:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                     reduceByKey() EXECUTION                          │
│                                                                      │
│  Input: [("apple",5), ("banana",3), ("apple",2),                    │
│          ("orange",7), ("banana",4)]                                 │
│                                                                      │
│  Step 1: Group by key (conceptually):                               │
│  ────────────────────────────────────                                │
│  "apple"  → [5, 2]                                                   │
│  "banana" → [3, 4]                                                   │
│  "orange" → [7]                                                      │
│                                                                      │
│  Step 2: Reduce within each group:                                  │
│  ──────────────────────────────────                                  │
│  "apple":  5 + 2 = 7                                                 │
│  "banana": 3 + 4 = 7                                                 │
│  "orange": 7                                                         │
│                                                                      │
│  Output: [("apple", 7), ("banana", 7), ("orange", 7)]               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**

1. **Automatically uses map-side combiner** (optimization!)
   - Combines values on each partition before shuffle
   - Reduces network transfer

2. **Function must be associative and commutative**
   - Same requirements as reduce()

3. **More efficient than groupByKey() + reduce()**

```
┌─────────────────────────────────────────────────────────────────────┐
│            reduceByKey() with MAP-SIDE COMBINER                      │
│                                                                      │
│  Partition 1:                      Partition 2:                     │
│  ────────────                      ────────────                     │
│  ("apple", 5)                      ("orange", 7)                    │
│  ("banana", 3)                     ("banana", 4)                    │
│  ("apple", 2)                                                       │
│                                                                      │
│  MAP-SIDE COMBINE:                 MAP-SIDE COMBINE:                │
│  ─────────────────                 ─────────────────                │
│  "apple":  5 + 2 = 7               "orange": 7                      │
│  "banana": 3                       "banana": 4                      │
│        │                                  │                          │
│        └──────────SHUFFLE──────────────── ┘                         │
│                      │                                               │
│        ┌─────────────┴──────────────┐                               │
│        │   REDUCE ACROSS PARTITIONS │                               │
│        └────────────────────────────┘                               │
│                      │                                               │
│  "apple":  7                                                         │
│  "banana": 3 + 4 = 7                                                 │
│  "orange": 7                                                         │
│                                                                      │
│  Benefits:                                                           │
│  • Reduced shuffle data (7 items → 5 items)                         │
│  • Less network transfer                                             │
│  • Faster execution                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**More Examples:**

```python
# Example 1: Finding maximum per key
scores = sc.parallelize([
    ("Alice", 85), ("Bob", 90), ("Alice", 92), ("Bob", 87)
])
max_scores = scores.reduceByKey(lambda v1, v2: max(v1, v2))
# Result: [("Alice", 92), ("Bob", 90)]

# Example 2: Concatenating strings
words = sc.parallelize([
    ("key1", "hello"), ("key2", "world"), ("key1", "spark")
])
concatenated = words.reduceByKey(lambda v1, v2: v1 + " " + v2)
# Result: [("key1", "hello spark"), ("key2", "world")]
# Note: Order might vary!

# Example 3: Product (multiplication)
values = sc.parallelize([
    ("a", 2), ("b", 3), ("a", 4), ("b", 5)
])
products = values.reduceByKey(lambda v1, v2: v1 * v2)
# Result: [("a", 8), ("b", 15)]
```

**Example from Slides: Finding Average per Key**

The slides show using `reduceByKey` for computing averages. However, `reduceByKey` alone cannot compute averages because:
- Average requires tracking both sum AND count
- reduceByKey's output type must match input type

For per-key averages, we need `combineByKey()` (next section!).

### 8.3 combineByKey() - Overview

**combineByKey()** is the most general aggregation function for Pair RDDs. Unlike reduceByKey(), it allows the accumulator type to differ from the value type.

```python
combineByKey(
    createCombiner,      # Create accumulator for first value of a key
    mergeValue,          # Add a value to accumulator
    mergeCombiners,      # Merge two accumulators
    numPartitions=None   # Optional: number of output partitions
)
```

**When to use:**
- Computing per-key averages, standard deviations, etc.
- Accumulator type differs from value type
- Need fine-grained control over aggregation

We'll cover combineByKey() in detail in Section 9 with the per-key average example.

---

## 9. Example: Per Key Average using combineByKey

Computing the average per key is a classic use case for `combineByKey()`. Let's build this step-by-step.

### 9.1 The Problem

Given pairs of (key, value), compute the average value for each key.

```python
# Input
data = sc.parallelize([
    ("Alice", 85),
    ("Bob", 90),
    ("Alice", 92),
    ("Charlie", 78),
    ("Bob", 87),
    ("Alice", 88)
])

# Desired output: (key, average)
# ("Alice", 88.33)  ← (85 + 92 + 88) / 3
# ("Bob", 88.5)     ← (90 + 87) / 2
# ("Charlie", 78.0) ← 78 / 1
```

### 9.2 Why Not reduceByKey()?

```python
# ATTEMPT 1: Using reduceByKey() - WRONG!
averages = data.reduceByKey(lambda v1, v2: (v1 + v2) / 2)
# Problem: This computes a running average, not the true average!
# Alice: (85 + 92)/2 = 88.5, then (88.5 + 88)/2 = 88.25 ✗ WRONG!
# Correct: (85 + 92 + 88)/3 = 88.33
```

**The issue:** We need to track both sum AND count, but reduceByKey's output type must match input type.

### 9.3 The combineByKey() Solution

**Accumulator type:** (sum, count) tuple

**Strategy:**
1. For the first value: Create (value, 1)
2. For subsequent values: Add to sum, increment count
3. Across partitions: Merge sums and counts
4. Final step: Divide sum by count

```python
# Step 1: combineByKey to get (sum, count) per key
sum_counts = data.combineByKey(
    lambda value: (value, 1),              # createCombiner
    lambda acc, value: (acc[0] + value, acc[1] + 1),  # mergeValue
    lambda acc1, acc2: (acc1[0] + acc2[0], acc1[1] + acc2[1])  # mergeCombiners
)

# Step 2: Compute averages
averages = sum_counts.mapValues(lambda (sum, count): sum / float(count))

# Result:
# [("Alice", 88.33), ("Bob", 88.5), ("Charlie", 78.0)]
```

### 9.4 The Three Functions Explained

**1. createCombiner: `lambda value: (value, 1)`**
- Called when we see a key for the FIRST time on a partition
- Creates initial accumulator
- Input: First value for this key
- Output: (sum=value, count=1)

**2. mergeValue: `lambda acc, value: (acc[0] + value, acc[1] + 1)`**
- Called for SUBSEQUENT values of the same key on a partition
- Updates accumulator with new value
- Input: Current accumulator (sum, count) and new value
- Output: (sum+value, count+1)

**3. mergeCombiners: `lambda acc1, acc2: (acc1[0] + acc2[0], acc1[1] + acc2[1])`**
- Called to merge accumulators from different partitions
- Input: Two accumulators (sum1, count1) and (sum2, count2)
- Output: (sum1+sum2, count1+count2)

### 9.5 Detailed Execution Trace

Let's trace the execution with our data distributed across 2 partitions:

**Data Distribution:**
- Partition 1: `[("Alice", 85), ("Bob", 90), ("Alice", 92)]`
- Partition 2: `[("Charlie", 78), ("Bob", 87), ("Alice", 88)]`

**Phase 1: Within Partition 1**

```
Process: [("Alice", 85), ("Bob", 90), ("Alice", 92)]

Alice (first time):
  createCombiner(85) → (85, 1)
  Current state: {"Alice": (85, 1)}

Bob (first time):
  createCombiner(90) → (90, 1)
  Current state: {"Alice": (85, 1), "Bob": (90, 1)}

Alice (second time):
  mergeValue((85, 1), 92) → (85+92, 1+1) = (177, 2)
  Current state: {"Alice": (177, 2), "Bob": (90, 1)}

Partition 1 Result: [("Alice", (177, 2)), ("Bob", (90, 1))]
```

**Phase 2: Within Partition 2**

```
Process: [("Charlie", 78), ("Bob", 87), ("Alice", 88)]

Charlie (first time):
  createCombiner(78) → (78, 1)
  Current state: {"Charlie": (78, 1)}

Bob (first time):
  createCombiner(87) → (87, 1)
  Current state: {"Charlie": (78, 1), "Bob": (87, 1)}

Alice (first time):
  createCombiner(88) → (88, 1)
  Current state: {"Charlie": (78, 1), "Bob": (87, 1), "Alice": (88, 1)}

Partition 2 Result: [("Charlie", (78, 1)), ("Bob", (87, 1)), ("Alice", (88, 1))]
```

**Phase 3: Merge Across Partitions**

```
Partition 1:          Partition 2:
────────────          ────────────
Alice: (177, 2)       Alice: (88, 1)
Bob:   (90, 1)        Bob:   (87, 1)
                      Charlie: (78, 1)

Merge Alice:
  mergeCombiners((177, 2), (88, 1)) → (177+88, 2+1) = (265, 3)

Merge Bob:
  mergeCombiners((90, 1), (87, 1)) → (90+87, 1+1) = (177, 2)

Charlie (only in partition 2):
  (78, 1)  ← No merging needed

Final Result: [("Alice", (265, 3)), ("Bob", (177, 2)), ("Charlie", (78, 1))]
```

**Phase 4: Compute Averages**

```python
averages = sum_counts.mapValues(lambda (sum, count): sum / float(count))

Alice:   265 / 3 = 88.33
Bob:     177 / 2 = 88.5
Charlie: 78 / 1  = 78.0

Result: [("Alice", 88.33), ("Bob", 88.5), ("Charlie", 78.0)]
```

### 9.6 Visual Diagram (from slides)

```
┌─────────────────────────────────────────────────────────────────────┐
│              combineByKey EXECUTION FOR AVERAGE                      │
│                                                                      │
│  PARTITION 1: [("Alice",85), ("Bob",90), ("Alice",92)]              │
│  ─────────────────────────────────────────────────────────           │
│                                                                      │
│  Alice, 85:  createCombiner(85)                → (85, 1)             │
│  Bob, 90:    createCombiner(90)                → (90, 1)             │
│  Alice, 92:  mergeValue((85,1), 92)            → (177, 2)            │
│                                                                      │
│  Partition 1 Accumulators:                                          │
│  {"Alice": (177, 2), "Bob": (90, 1)}                                │
│                                                                      │
│  ═════════════════════════════════════════════════════════          │
│                                                                      │
│  PARTITION 2: [("Charlie",78), ("Bob",87), ("Alice",88)]            │
│  ──────────────────────────────────────────────────────              │
│                                                                      │
│  Charlie, 78: createCombiner(78)               → (78, 1)             │
│  Bob, 87:     createCombiner(87)               → (87, 1)             │
│  Alice, 88:   createCombiner(88)               → (88, 1)             │
│                                                                      │
│  Partition 2 Accumulators:                                          │
│  {"Charlie": (78, 1), "Bob": (87, 1), "Alice": (88, 1)}             │
│                                                                      │
│  ═════════════════════════════════════════════════════════          │
│                                                                      │
│  MERGE ACCUMULATORS ACROSS PARTITIONS:                              │
│  ─────────────────────────────────────                               │
│                                                                      │
│  Alice:  mergeCombiners((177,2), (88,1))  → (265, 3)                │
│  Bob:    mergeCombiners((90,1), (87,1))   → (177, 2)                │
│  Charlie: (78, 1)  [no merge needed]                                │
│                                                                      │
│  Final Accumulators:                                                │
│  [("Alice", (265,3)), ("Bob", (177,2)), ("Charlie", (78,1))]        │
│                                                                      │
│  ═════════════════════════════════════════════════════════          │
│                                                                      │
│  COMPUTE AVERAGES:                                                   │
│  ─────────────────                                                   │
│                                                                      │
│  Alice:   265/3 = 88.33                                              │
│  Bob:     177/2 = 88.5                                               │
│  Charlie: 78/1  = 78.0                                               │
│                                                                      │
│  Result: [("Alice", 88.33), ("Bob", 88.5), ("Charlie", 78.0)]       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.7 Code Example from Slides

```python
# User-provided functions
def createCombiner(value):
    return (value, 1)  # (sum, count)

def mergeValue(acc, value):
    return (acc[0] + value, acc[1] + 1)

def mergeCombiner(acc1, acc2):
    return (acc1[0] + acc2[0], acc1[1] + acc2[1])

# Apply combineByKey
sumCounts = data.combineByKey(
    createCombiner,
    mergeValue,
    mergeCombiner
)

# Compute averages
averages = sumCounts.mapValues(lambda (sum, count): sum / float(count))
```

### 9.8 combineByKey() vs reduceByKey()

| Feature | reduceByKey() | combineByKey() |
|---------|---------------|----------------|
| **Accumulator Type** | Same as value type | Can be different |
| **Simplicity** | Easier to use | More complex |
| **Flexibility** | Limited | Very flexible |
| **Use Case** | Simple reductions (sum, max) | Complex aggregations (average, stats) |
| **Performance** | Faster (simpler) | Slightly slower (more general) |

### 9.9 Common Use Cases for combineByKey()

```python
# Use Case 1: Variance per key
def create(v):
    return (v, v*v, 1)  # (sum, sum_of_squares, count)

def merge_val(acc, v):
    return (acc[0]+v, acc[1]+v*v, acc[2]+1)

def merge_comb(a1, a2):
    return (a1[0]+a2[0], a1[1]+a2[1], a1[2]+a2[2])

stats = data.combineByKey(create, merge_val, merge_comb)
variance = stats.mapValues(lambda (s, ss, c):
    (ss/c) - (s/c)**2  # Variance formula
)

# Use Case 2: Set of unique values per key
def create(v):
    return {v}  # Set with single value

def merge_val(acc, v):
    acc.add(v)
    return acc

def merge_comb(a1, a2):
    return a1.union(a2)

unique_sets = data.combineByKey(create, merge_val, merge_comb)
```

---

## 10. Grouping Transforms on Pair RDDs

Sometimes we want to collect all values for each key. Spark provides several grouping transformations with different use cases.

### 10.1 groupByKey()

**groupByKey()** groups all values for each key into an **iterable** collection.

```python
pairs = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("apple", 2),
    ("orange", 7),
    ("banana", 4),
    ("apple", 1)
])

# Group all values by key
grouped = pairs.groupByKey()

# Convert iterables to lists for viewing
result = grouped.mapValues(list).collect()
# Result: [("apple", [5, 2, 1]), ("banana", [3, 4]), ("orange", [7])]
```

**Output Type:**
```python
# groupByKey() returns RDD[(K, Iterable[V])]
# Each key maps to an iterable of all its values
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                      groupByKey() EXECUTION                          │
│                                                                      │
│  Input:  [("apple",5), ("banana",3), ("apple",2),                   │
│           ("orange",7), ("banana",4), ("apple",1)]                   │
│                                                                      │
│  Step 1: Shuffle data by keys                                       │
│  Step 2: Collect all values per key                                 │
│                                                                      │
│  Output:                                                             │
│  ("apple",  Iterator[5, 2, 1])                                       │
│  ("banana", Iterator[3, 4])                                          │
│  ("orange", Iterator[7])                                             │
│                                                                      │
│  Note: Values are in an Iterator, not a materialized list           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**⚠️ WARNING: groupByKey() can cause Out Of Memory (OOM) errors**

```
┌─────────────────────────────────────────────────────────────────────┐
│                       groupByKey() PITFALL!                          │
│                                                                      │
│  Problem: ALL values for a key must fit in memory on ONE machine    │
│                                                                      │
│  Bad scenario:                                                       │
│  ─────────────                                                       │
│  Key "popular_product" has 1 million values                         │
│  → All 1 million values sent to one machine                         │
│  → Machine runs out of memory!                                      │
│  → Job crashes                                                       │
│                                                                      │
│  Alternative: Use reduceByKey() or aggregateByKey()                 │
│  ────────────                                                        │
│  • Process values incrementally                                      │
│  • Keep only aggregated result (much smaller)                       │
│  • More efficient and safer                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**groupByKey() vs reduceByKey():**

```python
# BAD: groupByKey() + sum
pairs = sc.parallelize([("a", 1), ("a", 2), ("a", 3)])
sums = pairs.groupByKey().mapValues(lambda vals: sum(vals))
# Process: Group all values → [1,2,3] → sum = 6
# Problem: All values must fit in memory

# GOOD: reduceByKey()
sums = pairs.reduceByKey(lambda v1, v2: v1 + v2)
# Process: Incremental reduction → 1+2=3, 3+3=6
# Benefit: Uses map-side combiner, less memory, faster
```

**When to use groupByKey():**
- ✓ When you actually need all values grouped together
- ✓ Number of values per key is small
- ✓ No simple reduction function available
- ✗ For aggregation (use reduceByKey or aggregateByKey instead)

### 10.2 cogroup()

**cogroup(otherRDD)** groups values from **two RDDs** with the same keys.

```python
rdd1 = sc.parallelize([("a", 1), ("b", 2), ("a", 3)])
rdd2 = sc.parallelize([("a", "x"), ("c", "y"), ("a", "z")])

# Cogroup by key
cogrouped = rdd1.cogroup(rdd2)

# Convert to lists for viewing
result = cogrouped.mapValues(lambda (iter1, iter2): (list(iter1), list(iter2))).collect()
# Result: [("a", ([1, 3], ["x", "z"])),
#          ("b", ([2], [])),
#          ("c", ([], ["y"]))]
```

**Key Points:**
- Returns `(key, (Iterable[V1], Iterable[V2]))`
- If key missing in one RDD, that iterator is empty
- Useful for joining related datasets

```
┌─────────────────────────────────────────────────────────────────────┐
│                        cogroup() EXECUTION                           │
│                                                                      │
│  RDD1: [("a",1), ("b",2), ("a",3)]                                  │
│  RDD2: [("a","x"), ("c","y"), ("a","z")]                            │
│                                                                      │
│  cogroup():                                                          │
│                                                                      │
│  Key "a": (values from RDD1, values from RDD2)                      │
│           ([1, 3], ["x", "z"])                                       │
│                                                                      │
│  Key "b": (values from RDD1, values from RDD2)                      │
│           ([2], [])  ← empty because "b" not in RDD2                │
│                                                                      │
│  Key "c": (values from RDD1, values from RDD2)                      │
│           ([], ["y"])  ← empty because "c" not in RDD1              │
│                                                                      │
│  Output: [("a", ([1,3], ["x","z"])),                                │
│           ("b", ([2], [])),                                          │
│           ("c", ([], ["y"]))]                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example from Slides:**

```python
rdd1 = sc.parallelize([(1, 2), (3, 4), (3, 6)])
rdd2 = sc.parallelize([(3, 9), (4, 7)])

result = rdd1.cogroup(rdd2)
# Result: [(1, ([2], [])),
#          (3, ([4, 6], [9])),
#          (4, ([], [7]))]
```

**Can work with more than 2 RDDs:**

```python
rdd1 = sc.parallelize([("a", 1)])
rdd2 = sc.parallelize([("a", 2)])
rdd3 = sc.parallelize([("a", 3)])

result = rdd1.cogroup(rdd2, rdd3)
# Result: [("a", ([1], [2], [3]))]
```

### 10.3 subtractByKey()

**subtractByKey(otherRDD)** removes entries from the first RDD where the key exists in the second RDD.

```python
rdd1 = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("orange", 7)
])

rdd2 = sc.parallelize([
    ("banana", 100),  # Value doesn't matter, only key
    ("grape", 50)
])

# Remove keys that exist in rdd2
result = rdd1.subtractByKey(rdd2)
# Result: [("apple", 5), ("orange", 7)]
# "banana" removed because it exists in rdd2
```

**Key Insight:** Only the **keys** matter in rdd2, values are ignored.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    subtractByKey() EXECUTION                         │
│                                                                      │
│  RDD1: [("apple",5), ("banana",3), ("orange",7)]                    │
│  RDD2: [("banana",100), ("grape",50)]                               │
│                                                                      │
│  Process:                                                            │
│  ────────                                                            │
│  1. Extract keys from RDD2: {"banana", "grape"}                     │
│  2. Keep pairs from RDD1 where key NOT in RDD2 keys                 │
│                                                                      │
│  Check each pair in RDD1:                                           │
│  • ("apple", 5)  → "apple" not in RDD2 → KEEP                       │
│  • ("banana", 3) → "banana" in RDD2    → REMOVE                     │
│  • ("orange", 7) → "orange" not in RDD2 → KEEP                      │
│                                                                      │
│  Output: [("apple", 5), ("orange", 7)]                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example from Slides:**

```python
rdd1 = sc.parallelize([(1, 2), (3, 4), (3, 6)])
rdd2 = sc.parallelize([(3, 9)])

result = rdd1.subtractByKey(rdd2)
# Result: [(1, 2)]
# All pairs with key=3 removed
```

**Use Cases:**
- Filtering out processed records
- Removing blacklisted items
- Set difference based on keys

---

## 11. Join Transforms on Pair RDDs

Joins combine two Pair RDDs based on matching keys. Spark supports several types of joins similar to SQL.

### 11.1 Inner Join (join)

**join(otherRDD)** returns pairs where the key exists in **both** RDDs. Performs a **cross product** of values for the same key.

```python
rdd1 = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("apple", 2)
])

rdd2 = sc.parallelize([
    ("apple", "red"),
    ("banana", "yellow"),
    ("grape", "purple")
])

# Inner join
result = rdd1.join(rdd2)
# Result: [("apple", (5, "red")),
#          ("apple", (2, "red")),
#          ("banana", (3, "yellow"))]
# "grape" excluded (only in rdd2)
```

**Join Behavior with Multiple Values:**

```python
rdd1 = sc.parallelize([(1, 2), (3, 4), (3, 6)])
rdd2 = sc.parallelize([(3, 9)])

result = rdd1.join(rdd2)
# Key 3 in rdd1: [4, 6]
# Key 3 in rdd2: [9]
# Cross product: (3, (4, 9)), (3, (6, 9))
# Result: [(3, (4, 9)), (3, (6, 9))]
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                       INNER JOIN EXECUTION                           │
│                                                                      │
│  RDD1: [("apple",5), ("banana",3), ("apple",2)]                     │
│  RDD2: [("apple","red"), ("banana","yellow"), ("grape","purple")]   │
│                                                                      │
│  Step 1: Find common keys                                           │
│  ────────────────────────────                                        │
│  Keys in RDD1: {apple, banana}                                      │
│  Keys in RDD2: {apple, banana, grape}                               │
│  Common keys:  {apple, banana}  ← Only these will be in result      │
│                                                                      │
│  Step 2: Cross product of values for each common key                │
│  ──────────────────────────────────────────────────────              │
│                                                                      │
│  "apple":                                                            │
│    RDD1 values: [5, 2]                                               │
│    RDD2 values: ["red"]                                              │
│    Cross product: (5, "red"), (2, "red")                            │
│                                                                      │
│  "banana":                                                           │
│    RDD1 values: [3]                                                  │
│    RDD2 values: ["yellow"]                                           │
│    Cross product: (3, "yellow")                                     │
│                                                                      │
│  Output: [("apple", (5, "red")),                                    │
│           ("apple", (2, "red")),                                    │
│           ("banana", (3, "yellow"))]                                │
│                                                                      │
│  Note: "grape" excluded (only in RDD2, not in both)                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 Left Outer Join

**leftOuterJoin(otherRDD)** returns **all** keys from the **left (first) RDD**. For keys not in the right RDD, value is `None`.

```python
rdd1 = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("orange", 7)
])

rdd2 = sc.parallelize([
    ("apple", "red"),
    ("grape", "purple")
])

# Left outer join
result = rdd1.leftOuterJoin(rdd2)
# Result: [("apple", (5, Some("red"))),
#          ("banana", (3, None)),
#          ("orange", (7, None))]
# All keys from rdd1 present, None for missing rdd2 values
```

**Example from Slides:**

```python
rdd1 = sc.parallelize([(1, 2), (3, 4), (3, 6)])
rdd2 = sc.parallelize([(3, 9)])

result = rdd1.leftOuterJoin(rdd2)
# Result: [(1, (2, None)),
#          (3, (4, Some(9))),
#          (3, (6, Some(9)))]
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                     LEFT OUTER JOIN DIAGRAM                          │
│                                                                      │
│  RDD1: [("a",1), ("b",2), ("c",3)]  ← LEFT side (all kept)          │
│  RDD2: [("a","x"), ("d","y")]       ← RIGHT side (may have nulls)   │
│                                                                      │
│  leftOuterJoin():                                                    │
│                                                                      │
│  Key "a": In both → ("a", (1, Some("x")))                           │
│  Key "b": Only in left → ("b", (2, None))                           │
│  Key "c": Only in left → ("c", (3, None))                           │
│  Key "d": Only in right → NOT INCLUDED                              │
│                                                                      │
│  Output: [("a", (1, Some("x"))),                                    │
│           ("b", (2, None)),                                          │
│           ("c", (3, None))]                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.3 Right Outer Join

**rightOuterJoin(otherRDD)** returns **all** keys from the **right (second) RDD**. For keys not in the left RDD, value is `None`.

```python
rdd1 = sc.parallelize([
    ("apple", 5),
    ("banana", 3)
])

rdd2 = sc.parallelize([
    ("apple", "red"),
    ("banana", "yellow"),
    ("grape", "purple")
])

# Right outer join
result = rdd1.rightOuterJoin(rdd2)
# Result: [("apple", (Some(5), "red")),
#          ("banana", (Some(3), "yellow")),
#          ("grape", (None, "purple"))]
# All keys from rdd2 present, None for missing rdd1 values
```

**Example from Slides:**

```python
rdd1 = sc.parallelize([(1, 2), (3, 4), (3, 6)])
rdd2 = sc.parallelize([(3, 9), (4, 2)])

result = rdd1.rightOuterJoin(rdd2)
# Result: [(3, (Some(4), 9)),
#          (3, (Some(6), 9)),
#          (4, (None, 2))]
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RIGHT OUTER JOIN DIAGRAM                          │
│                                                                      │
│  RDD1: [("a",1), ("b",2)]           ← LEFT side (may have nulls)    │
│  RDD2: [("a","x"), ("c","y")]       ← RIGHT side (all kept)         │
│                                                                      │
│  rightOuterJoin():                                                   │
│                                                                      │
│  Key "a": In both → ("a", (Some(1), "x"))                           │
│  Key "b": Only in left → NOT INCLUDED                               │
│  Key "c": Only in right → ("c", (None, "y"))                        │
│                                                                      │
│  Output: [("a", (Some(1), "x")),                                    │
│           ("c", (None, "y"))]                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.4 Join Summary Table

| Join Type | Keys Included | Missing Value | Example |
|-----------|---------------|---------------|---------|
| **Inner Join** | Only in both RDDs | N/A (excluded) | `join()` |
| **Left Outer** | All from left RDD | None for missing right | `leftOuterJoin()` |
| **Right Outer** | All from right RDD | None for missing left | `rightOuterJoin()` |
| **Full Outer** | All from both RDDs | None for either side | `fullOuterJoin()` |

**Choosing the right join:**

```python
# Use inner join when: You only want matched records
users.join(purchases)  # Users who made purchases

# Use left outer join when: Keep all from left, even if no match
users.leftOuterJoin(purchases)  # All users, with/without purchases

# Use right outer join when: Keep all from right, even if no match
users.rightOuterJoin(purchases)  # All purchases, even deleted users

# Use full outer join when: Keep everything from both sides
users.fullOuterJoin(purchases)  # All users and all purchases
```

---

## 12. Sorting Transforms on Pair RDDs

### 12.1 sortByKey()

**sortByKey(ascending=True)** sorts the RDD by **keys only**. Values are NOT considered in sorting.

```python
pairs = sc.parallelize([
    (3, "three"),
    (1, "one"),
    (4, "four"),
    (2, "two")
])

# Sort by key (ascending)
sorted_asc = pairs.sortByKey(ascending=True)
# Result: [(1, "one"), (2, "two"), (3, "three"), (4, "four")]

# Sort by key (descending)
sorted_desc = pairs.sortByKey(ascending=False)
# Result: [(4, "four"), (3, "three"), (2, "two"), (1, "one")]
```

**Key Characteristics:**
- Only keys are used for sorting
- Values do NOT affect sort order
- Default is ascending order

**Example with Duplicate Keys:**

```python
pairs = sc.parallelize([
    (1, 2),
    (3, 6),
    (3, 5),  # Same key, different value
    (2, 4)
])

sorted_pairs = pairs.sortByKey()
# Result: [(1, 2), (2, 4), (3, 6), (3, 5)]
# OR:     [(1, 2), (2, 4), (3, 5), (3, 6)]
# Order of values with same key is not guaranteed!
```

### 12.2 Custom Key Functions

You can transform keys before sorting using a custom key function:

```python
# Sort by length of string keys
words = sc.parallelize([
    ("hi", 1),
    ("hello", 2),
    ("hey", 3)
])

sorted_by_length = words.sortByKey(keyfunc=lambda k: len(k))
# Result: [("hi", 1), ("hey", 3), ("hello", 2)]

# Treat numeric keys as strings
nums = sc.parallelize([(1, "one"), (10, "ten"), (2, "two")])
sorted_as_str = nums.sortByKey(keyfunc=lambda k: str(k))
# Numeric sort: [(1, "one"), (2, "two"), (10, "ten")]
# String sort:  [(1, "one"), (10, "ten"), (2, "two")]
```

### 12.3 Secondary Sort

For sorting by value when keys are equal, you need to use a custom key:

```python
# Want to sort by key, then by value
pairs = sc.parallelize([
    (3, 6),
    (1, 2),
    (3, 5),
    (2, 4)
])

# Create composite key  (original_key, value)
sorted_pairs = pairs.map(lambda (k, v): ((k, v), (k, v))) \
                    .sortByKey() \
                    .map(lambda (_, (k, v)): (k, v))
# Result: [(1, 2), (2, 4), (3, 5), (3, 6)]
```

**Reference:** For more advanced secondary sorting, see: http://codingjunkie.net/spark-secondary-sort/

---

## 13. Stratified Sampling

### 13.1 sampleByKey()

**sampleByKey(withReplacement, fractions, seed)** allows you to specify **different sampling rates for different keys**.

```python
from pyspark import SparkContext
sc = SparkContext()

# Data with different keys
data = sc.parallelize([
    ("male", 1), ("female", 2), ("male", 3),
    ("female", 4), ("male", 5), ("female", 6),
    ("male", 7), ("female", 8)
])

# Sample different fractions per key
fractions = {
    "male": 0.5,    # Sample 50% of males
    "female": 1.0   # Sample 100% of females
}

sampled = data.sampleByKey(False, fractions, seed=42)
# Approximately:
# - 50% of male records: ~2 records
# - 100% of female records: ~4 records
```

**Signature:**
```python
sampleByKey(
    withReplacement,  # True: can pick same item multiple times
    fractions,        # Dict: {key: fraction}
    seed=None         # Random seed for reproducibility
)
```

**Key Point:** The result is **approximate**, not exact.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    sampleByKey() EXPLANATION                         │
│                                                                      │
│  Input: RDD with keys and values                                    │
│  fractions = {k: fₖ} where fₖ is sampling fraction for key k        │
│                                                                      │
│  For each key k:                                                     │
│  • Count of values: nₖ                                               │
│  • Sampling fraction: fₖ                                             │
│  • Expected sample size: fₖ × nₖ                                     │
│  • Actual sample size: ≈ fₖ × nₖ (approximate!)                      │
│                                                                      │
│  Total expected output: Σₖ (fₖ × nₖ)                                 │
│                                                                      │
│  Use Case: Balanced sampling across categories                      │
│  Example: Equal representation of male/female in dataset            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Example from Slides:**

```python
# Suppose we have user activity data
activities = sc.parallelize([
    ("login", 1), ("purchase", 2), ("login", 3),
    ("logout", 4), ("purchase", 5), ("login", 6)
])

# Sample different rates for different activity types
fractions = {
    "login": 0.3,     # 30% of logins
    "purchase": 1.0,  # 100% of purchases (keep all)
    "logout": 0.5     # 50% of logouts
}

sample = activities.sampleByKey(False, fractions, 12345)
```

**Use Cases:**
- Balancing skewed datasets
- Proportional sampling from different categories
- Testing with representative samples

---

## 14. Actions on Pair RDDs

Pair RDDs support all normal RDD actions, plus some specialized ones.

### 14.1 Normal Actions (Still Work)

All actions from Section 5 work on Pair RDDs:

```python
pairs = sc.parallelize([("a", 1), ("b", 2), ("a", 3)])

# Regular actions
count = pairs.count()           # 3
all_pairs = pairs.collect()     # [("a", 1), ("b", 2), ("a", 3)]
first = pairs.first()           # ("a", 1)
sample = pairs.take(2)          # [("a", 1), ("b", 2)]
```

### 14.2 countByKey()

Returns a **dictionary** mapping each key to its **count** (number of occurrences).

```python
pairs = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("apple", 2),
    ("orange", 7),
    ("apple", 1)
])

counts = pairs.countByKey()
# Result: {"apple": 3, "banana": 1, "orange": 1}
# Dictionary: {key: count}
```

**Use Case:** Frequency analysis by key.

### 14.3 collectAsMap()

Returns the RDD as a **Python dictionary**. If a key appears multiple times, only one value is kept (non-deterministic which one).

```python
pairs = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("orange", 7)
])

dict_result = pairs.collectAsMap()
# Result: {"apple": 5, "banana": 3, "orange": 7}
# Type: Python dict
```

**⚠️ Warning with Duplicate Keys:**

```python
pairs = sc.parallelize([
    ("apple", 5),
    ("apple", 2),  # Duplicate key!
    ("banana", 3)
])

dict_result = pairs.collectAsMap()
# Result: {"apple": 5, "banana": 3}  OR  {"apple": 2, "banana": 3}
# Which value for "apple" is kept is non-deterministic!
```

**Best Practice:** Only use `collectAsMap()` when keys are unique, or you've already reduced by key.

### 14.4 lookup(key)

Returns **list of all values** for a specific key.

```python
pairs = sc.parallelize([
    ("apple", 5),
    ("banana", 3),
    ("apple", 2),
    ("orange", 7),
    ("apple", 1)
])

apple_values = pairs.lookup("apple")
# Result: [5, 2, 1]
# Returns list of all values for key "apple"

banana_values = pairs.lookup("banana")
# Result: [3]

missing_values = pairs.lookup("grape")
# Result: []  (empty list for missing key)
```

**Performance Note:**
- Efficient: Scans only partitions that might contain the key
- Returns all values, not just one

### 14.5 Summary: Special Pair RDD Actions

| Action | Returns | Example |
|--------|---------|---------|
| **countByKey()** | Dict {key: count} | `{"a": 3, "b": 1}` |
| **collectAsMap()** | Dict {key: value} | `{"a": 5, "b": 3}` |
| **lookup(key)** | List of values for key | `[5, 2, 1]` |

---

## 15. Case Study 1: Web Crawl - Extract Title & Build Inverted Index

Now let's apply what we've learned to a real-world scenario: processing web crawl data.

### 15.1 Context: CommonCrawl Dataset

**CommonCrawl** (https://commoncrawl.org) is a massive dataset containing petabytes of web crawl data:
- Billions of web pages
- Stored in HDFS
- Each record: (URL, HTML content)

**Our Tasks:**
1. Extract page titles from HTML
2. Build an inverted index (keyword → list of URLs)

### 15.2 Task 1: Extract Title for URL

**Goal:** Parse HTML and extract the `<title>` tag.

**Input:** Pair RDD of (URL, HTML)
**Output:** Pair RDD of (URL, title)

```python
# Input: RDD[(URL, HTML_content)]
HTMLRdd = sc.parallelize([
    ("http://example.com/page1", "<html><title>Example Page</title><body>...</body></html>"),
    ("http://example.com/page2", "<html><title>Another Page</title><body>...</body></html>")
])

# Function to extract title from HTML
def parseOutTitle(html):
    """Extract title from HTML string."""
    import re
    match = re.search(r'<title>(.*?)</title>', html, re.IGNORECASE)
    if match:
        return match.group(1)
    return "No Title"

# Apply transformation
titleRdd = HTMLRdd.mapValues(lambda html: parseOutTitle(html))

# Result:
# [("http://example.com/page1", "Example Page"),
#  ("http://example.com/page2", "Another Page")]
```

**Execution:**
```
┌─────────────────────────────────────────────────────────────────────┐
│                    EXTRACT TITLE TRANSFORMATION                      │
│                                                                      │
│  Input HTMLRdd:                                                      │
│  ("url1", "<html><title>Page 1</title>...</html>")                  │
│  ("url2", "<html><title>Page 2</title>...</html>")                  │
│                                                                      │
│  mapValues(parseOutTitle):                                           │
│  │                                                                   │
│  ├─ url1: parseOutTitle(...) → "Page 1"                             │
│  └─ url2: parseOutTitle(...) → "Page 2"                             │
│                                                                      │
│  Output titleRdd:                                                    │
│  ("url1", "Page 1")                                                  │
│  ("url2", "Page 2")                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 15.3 Task 2: Build Inverted Index

**Goal:** Create a mapping from keywords to list of URLs containing that keyword.

**Conceptual Flow:**
```
URL → HTML → Words → (URL, Word) → (Word, URL) → (Word, [URLs])
```

**Step-by-Step Implementation:**

**Step 1: Extract words from HTML**

```python
# Function to parse HTML and extract words
def parseOutWords(html):
    """Extract all words from HTML."""
    import re
    # Remove HTML tags
    text = re.sub(r'<[^>]+>', ' ', html)
    # Extract words (alphanumeric sequences)
    words = re.findall(r'\b[a-zA-Z]+\b', text.lower())
    return words

# flatMap to get (URL, word) pairs
HTMLWordRdd = HTMLRdd.flatMap(lambda (url, html):
    [(url, word) for word in parseOutWords(html)]
)

# Example output:
# [("url1", "the"), ("url1", "quick"), ("url1", "brown"),
#  ("url2", "the"), ("url2", "lazy"), ...]
```

**Step 2: Filter stopwords and identify keywords**

```python
# Stopwords list (words to ignore)
STOPWORDS = {"the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for"}

# Filter out stopwords
HTMLKeywordRdd = HTMLWordRdd.filter(lambda (url, word):
    word not in STOPWORDS
)

# Example output:
# [("url1", "quick"), ("url1", "brown"), ("url2", "lazy"), ...]
```

**Step 3: Invert to (keyword, URL)**

```python
# Swap key and value
keyUrlRdd = HTMLKeywordRdd.map(lambda (url, keyword):
    (keyword, url)
)

# Example output:
# [("quick", "url1"), ("brown", "url1"), ("lazy", "url2"), ...]
```

**Step 4: Group by keyword**

```python
# Group all URLs for each keyword
keyUrlsRdd = keyUrlRdd.groupByKey()

# Convert iterables to lists
invertedIndex = keyUrlsRdd.mapValues(list)

# Example output:
# [("quick", ["url1", "url3"]),
#  ("brown", ["url1"]),
#  ("lazy", ["url2", "url4"]),
#  ...]
```

### 15.4 Complete Code

```python
# Complete inverted index pipeline
def buildInvertedIndex(HTMLRdd):
    """Build inverted index from (URL, HTML) RDD."""

    # Step 1: Extract words
    HTMLWordRdd = HTMLRdd.flatMap(lambda (url, html):
        [(url, word) for word in parseOutWords(html)]
    )

    # Step 2: Filter stopwords
    STOPWORDS = {"the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for"}
    HTMLKeywordRdd = HTMLWordRdd.filter(lambda (url, word): word not in STOPWORDS)

    # Step 3: Invert to (keyword, URL)
    keyUrlRdd = HTMLKeywordRdd.map(lambda (url, keyword): (keyword, url))

    # Step 4: Group by keyword
    keyUrlsRdd = keyUrlRdd.groupByKey().mapValues(list)

    return keyUrlsRdd

# Usage
invertedIndex = buildInvertedIndex(HTMLRdd)
```

### 15.5 Example Execution Trace

```
┌─────────────────────────────────────────────────────────────────────┐
│                  INVERTED INDEX CONSTRUCTION                         │
│                                                                      │
│  Input (3 documents):                                                │
│  ──────────────────────                                              │
│  url1: "The quick brown fox"                                         │
│  url2: "The lazy dog"                                                │
│  url3: "Quick brown dog jumps"                                       │
│                                                                      │
│  STEP 1: Extract words (flatMap)                                    │
│  ─────────────────────────────────                                   │
│  [("url1", "the"), ("url1", "quick"), ("url1", "brown"),            │
│   ("url1", "fox"), ("url2", "the"), ("url2", "lazy"),               │
│   ("url2", "dog"), ("url3", "quick"), ("url3", "brown"),            │
│   ("url3", "dog"), ("url3", "jumps")]                                │
│                                                                      │
│  STEP 2: Filter stopwords                                           │
│  ──────────────────────────                                          │
│  [("url1", "quick"), ("url1", "brown"), ("url1", "fox"),            │
│   ("url2", "lazy"), ("url2", "dog"), ("url3", "quick"),             │
│   ("url3", "brown"), ("url3", "dog"), ("url3", "jumps")]            │
│                                                                      │
│  STEP 3: Invert (map)                                               │
│  ─────────────────────                                               │
│  [("quick", "url1"), ("brown", "url1"), ("fox", "url1"),            │
│   ("lazy", "url2"), ("dog", "url2"), ("quick", "url3"),             │
│   ("brown", "url3"), ("dog", "url3"), ("jumps", "url3")]            │
│                                                                      │
│  STEP 4: Group by keyword (groupByKey)                              │
│  ───────────────────────────────────────                             │
│  quick  → ["url1", "url3"]                                           │
│  brown  → ["url1", "url3"]                                           │
│  fox    → ["url1"]                                                   │
│  lazy   → ["url2"]                                                   │
│  dog    → ["url2", "url3"]                                           │
│  jumps  → ["url3"]                                                   │
│                                                                      │
│  Final Inverted Index:                                               │
│  ──────────────────────                                              │
│  {                                                                   │
│    "quick":  ["url1", "url3"],                                       │
│    "brown":  ["url1", "url3"],                                       │
│    "fox":    ["url1"],                                               │
│    "lazy":   ["url2"],                                               │
│    "dog":    ["url2", "url3"],                                       │
│    "jumps":  ["url3"]                                                │
│  }                                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Usage:** Search for documents containing "quick":
```python
results = invertedIndex.lookup("quick")
# Returns: ["url1", "url3"]
```

---

## 16. Case Study 2: Web Graph and PageRank

PageRank is Google's original algorithm for ranking web pages based on link structure.

### 16.1 Task 1: Build WWW Link Graph

**Goal:** Extract all links from web pages and build an adjacency list representation of the web graph.

**Input:** (URL, HTML content)
**Output:** (source_URL, [destination_URLs])

**Step 1: Extract links from HTML**

```python
def parseOutLinks(html):
    """Extract all <a href="..."> URLs from HTML."""
    import re
    # Find all href attributes
    links = re.findall(r'href=["\']([^"\']+)["\']', html, re.IGNORECASE)
    return links

# Apply to HTMLRdd
links = HTMLRdd.mapValues(lambda html: parseOutLinks(html))

# Result: (src_url, [dest_url1, dest_url2, ...])
# Example:
# [("http://A.com", ["http://B.com", "http://C.com"]),
#  ("http://B.com", ["http://C.com", "http://D.com"]),
#  ...]
```

**Adjacency List Representation:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                      WEB GRAPH STRUCTURE                             │
│                                                                      │
│  Page A ──┐                                                          │
│           ├──▶ Page B ────▶ Page D                                  │
│           │       │                                                  │
│           └───────┼──▶ Page C ◀────┐                                │
│                   │                 │                                │
│                   └─────────────────┘                                │
│                                                                      │
│  Adjacency List:                                                     │
│  ───────────────                                                     │
│  A: [B, C]                                                           │
│  B: [C, D]                                                           │
│  C: []                                                               │
│  D: []                                                               │
│                                                                      │
│  Interpretation:                                                     │
│  • Page A links to B and C                                           │
│  • Page B links to C and D                                           │
│  • Pages C and D have no outgoing links                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.2 Task 2: Calculate PageRank

**PageRank Algorithm:**

PageRank is an iterative algorithm that computes the "importance" of each page based on the link structure.

**Core Idea:**
- A page is important if it's linked to by important pages
- Importance is distributed among all outgoing links

**Formula:**
```
PR(page) = (1 - d) + d × Σ(PR(linking_page) / num_outlinks(linking_page))

Where:
- PR(page) = PageRank of the page
- d = damping factor (typically 0.85)
- Σ = sum over all pages linking to this page
- num_outlinks = number of outgoing links from linking page
```

### 16.3 PageRank Implementation in Spark

**Algorithm Overview:**

1. Initialize all pages with rank = 1.0
2. For each iteration:
   a. Each page distributes its rank equally to its neighbors
   b. Each page receives contributions from its inbound links
   c. Update ranks with formula: `rank' = 0.15 + 0.85 × contributions`
3. Repeat for 30 iterations (or until convergence)

**Complete Code:**

```python
def computeContribs(urls, rank):
    """
    Compute contributions from a page to its neighbors.

    Args:
        urls: List of destination URLs
        rank: Current PageRank of source page

    Yields:
        (dest_url, contribution) for each destination
    """
    num_urls = len(urls)
    for url in urls:
        yield (url, rank / num_urls)


# Load link graph: (source_url, [dest_urls])
links = sc.parallelize([
    ("A", ["B", "C"]),
    ("B", ["C", "D"]),
    ("C", ["A"]),
    ("D", ["C"])
])

# Initialize all ranks to 1.0
ranks = links.map(lambda (url, neighbors): (url, 1.0))

# Run PageRank for 30 iterations
for iteration in range(30):
    # Step 1: Join links with current ranks
    # Result: (src_url, ([dest_urls], rank))
    links_with_ranks = links.join(ranks)

    # Step 2: Compute contributions
    # flatMap to emit (dest_url, contribution) for each link
    contribs = links_with_ranks.flatMap(
        lambda (src, (urls, rank)): computeContribs(urls, rank)
    )

    # Step 3: Aggregate contributions per page and compute new rank
    ranks = contribs.reduceByKey(lambda v1, v2: v1 + v2) \
                    .mapValues(lambda contrib: 0.15 + 0.85 * contrib)

# Collect final ranks
final_ranks = ranks.collect()
for (url, rank) in final_ranks:
    print(f"{url} has rank: {rank}")
```

### 16.4 Detailed Execution Trace (One Iteration)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  PAGERANK ITERATION EXAMPLE                          │
│                                                                      │
│  Link Graph:                                                         │
│  ───────────                                                         │
│  A → [B, C]                                                          │
│  B → [C]                                                             │
│  C → [A]                                                             │
│                                                                      │
│  Initial Ranks (iteration 0):                                       │
│  ─────────────────────────────                                       │
│  A: 1.0                                                              │
│  B: 1.0                                                              │
│  C: 1.0                                                              │
│                                                                      │
│  ITERATION 1:                                                        │
│  ────────────                                                        │
│                                                                      │
│  Step 1: Join links with ranks                                      │
│  ──────────────────────────────                                      │
│  A: ([B, C], 1.0)                                                    │
│  B: ([C], 1.0)                                                       │
│  C: ([A], 1.0)                                                       │
│                                                                      │
│  Step 2: Compute contributions (flatMap)                            │
│  ────────────────────────────────────────                            │
│  A sends to B: 1.0 / 2 = 0.5                                         │
│  A sends to C: 1.0 / 2 = 0.5                                         │
│  B sends to C: 1.0 / 1 = 1.0                                         │
│  C sends to A: 1.0 / 1 = 1.0                                         │
│                                                                      │
│  Contributions RDD:                                                  │
│  [("B", 0.5), ("C", 0.5), ("C", 1.0), ("A", 1.0)]                   │
│                                                                      │
│  Step 3: Aggregate contributions (reduceByKey)                      │
│  ──────────────────────────────────────────────                      │
│  A: 1.0                                                              │
│  B: 0.5                                                              │
│  C: 0.5 + 1.0 = 1.5                                                  │
│                                                                      │
│  Step 4: Apply PageRank formula (mapValues)                         │
│  ────────────────────────────────────────────                        │
│  A: 0.15 + 0.85 × 1.0 = 1.0                                          │
│  B: 0.15 + 0.85 × 0.5 = 0.575                                        │
│  C: 0.15 + 0.85 × 1.5 = 1.425                                        │
│                                                                      │
│  Updated Ranks (after iteration 1):                                 │
│  ───────────────────────────────────                                 │
│  A: 1.0                                                              │
│  B: 0.575                                                            │
│  C: 1.425                                                            │
│                                                                      │
│  Repeat for 30 iterations until convergence...                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.5 Understanding the Algorithm

**Key Steps:**

1. **join(links, ranks)**: Combine link structure with current page ranks
2. **flatMap(computeContribs)**: Each page distributes its rank to neighbors
3. **reduceByKey(add)**: Sum all contributions received by each page
4. **mapValues(applyFormula)**: Apply damping factor and compute new rank

**Why 30 iterations?**
- PageRank converges (stabilizes) after ~10-50 iterations
- More iterations = more accurate ranks
- Can also check for convergence (rank change < threshold)

**Damping Factor (0.85):**
- Models random web surfing behavior
- 85% chance: Follow a link
- 15% chance: Jump to random page
- Prevents rank from accumulating on dead ends

### 16.6 Real-World Considerations

```python
# In production, add:
# 1. Handle pages with no outlinks (dangling nodes)
# 2. Normalize ranks to sum to 1.0
# 3. Check for convergence
# 4. Cache links RDD (used in every iteration)

links.cache()  # Important! Reused in every iteration

# Convergence check:
for iteration in range(100):
    new_ranks = compute_iteration(links, ranks)

    # Check if ranks changed significantly
    diff = new_ranks.join(ranks) \
                    .mapValues(lambda (new, old): abs(new - old)) \
                    .values() \
                    .max()

    if diff < 0.001:  # Converged
        print(f"Converged after {iteration} iterations")
        break

    ranks = new_ranks
```

---

## 17. Case Study 3: Web Search

Now let's combine the inverted index and PageRank to implement a simple search engine.

### 17.1 Search Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      WEB SEARCH PIPELINE                             │
│                                                                      │
│  1. User Query: "spark apache"                                      │
│          │                                                           │
│          ▼                                                           │
│  2. Lookup Keywords in Inverted Index                               │
│     ────────────────────────────────────                             │
│     "spark"  → [url1, url2, url3, url5]                             │
│     "apache" → [url2, url3, url4]                                   │
│          │                                                           │
│          ▼                                                           │
│  3. Find Intersection (URLs with ALL keywords)                      │
│     ───────────────────────────────────────────                      │
│     [url1, url2, url3, url5] ∩ [url2, url3, url4] = [url2, url3]    │
│          │                                                           │
│          ▼                                                           │
│  4. Lookup PageRank for Matching URLs                               │
│     ──────────────────────────────────────                           │
│     url2 → PR = 0.85                                                │
│     url3 → PR = 1.23                                                │
│          │                                                           │
│          ▼                                                           │
│  5. Sort by PageRank (Descending)                                   │
│     ─────────────────────────────────                                │
│     url3: PR=1.23  ← Best result                                    │
│     url2: PR=0.85                                                   │
│          │                                                           │
│          ▼                                                           │
│  6. Select Top N Results                                            │
│     ───────────────────────                                          │
│     Top 10 results                                                   │
│          │                                                           │
│          ▼                                                           │
│  7. Join with Titles and Return                                     │
│     ──────────────────────────────                                   │
│     [(url3, "Apache Spark Documentation", 1.23),                    │
│      (url2, "Spark Tutorial", 0.85)]                                │
│          │                                                           │
│          ▼                                                           │
│  8. Display to User                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 17.2 Implementation Approach 1: Using Driver Memory

**Simpler but not scalable:**

```python
def search_v1(query, invertedIndex, pageRanks, titles):
    """
    Search using driver memory for intersection.

    Args:
        query: Search phrase (e.g., "spark apache")
        invertedIndex: RDD[(keyword, [urls])]
        pageRanks: RDD[(url, rank)]
        titles: RDD[(url, title)]

    Returns:
        Top 10 results with titles and ranks
    """
    # Step 1: Split query into keywords
    keywords = query.lower().split()

    # Step 2: Lookup each keyword in inverted index (brings to driver!)
    url_lists = []
    for keyword in keywords:
        urls = invertedIndex.lookup(keyword)
        if urls:
            url_lists.append(set(urls[0]))  # lookup returns list of values

    # Step 3: Find intersection on driver
    if not url_lists:
        return []

    matching_urls = set.intersection(*url_lists)

    # Step 4: Lookup PageRank for matching URLs
    # Filter pageRanks to matching URLs
    ranked_matches = pageRanks.filter(lambda (url, rank): url in matching_urls)

    # Step 5: Sort by PageRank (descending) and take top 10
    top_urls = ranked_matches.map(lambda (url, rank): (rank, url)) \
                              .top(10)  # top() gives descending order

    # Step 6: Get top URLs
    top_url_list = [url for (rank, url) in top_urls]

    # Step 7: Join with titles
    results = titles.filter(lambda (url, title): url in top_url_list) \
                    .collectAsMap()

    # Step 8: Combine with ranks
    final_results = []
    for (rank, url) in top_urls:
        title = results.get(url, "No Title")
        final_results.append((url, title, rank))

    return final_results
```

**Limitation:** Uses driver memory for intersection - not scalable for many keywords.

### 17.3 Implementation Approach 2: Fully Distributed

**Better approach using only RDD operations:**

```python
def search_v2(query, invertedIndex, pageRanks, titles):
    """
    Fully distributed search using only RDD operations.

    More scalable than v1.
    """
    keywords = query.lower().split()

    # Step 1: Get RDD of (keyword, url) from inverted index
    # Explode the grouped structure
    keyword_url_pairs = invertedIndex.flatMap(
        lambda (kw, urls): [(kw, url) for url in urls]
    )

    # Step 2: Filter to only our query keywords
    relevant_pairs = keyword_url_pairs.filter(
        lambda (kw, url): kw in keywords
    )

    # Step 3: Count how many keywords each URL matches
    url_keyword_counts = relevant_pairs.map(lambda (kw, url): (url, 1)) \
                                       .reduceByKey(lambda a, b: a + b)

    # Step 4: Keep only URLs that match ALL keywords
    num_keywords = len(keywords)
    matching_urls = url_keyword_counts.filter(
        lambda (url, count): count == num_keywords
    ).keys()  # Get just the URLs

    # Step 5: Join with PageRank
    # Create (url, None) pairs to join with ranks
    url_pairs = matching_urls.map(lambda url: (url, None))
    ranked_results = url_pairs.join(pageRanks)  # (url, (None, rank))

    # Step 6: Sort by rank and take top 10
    top_10 = ranked_results.map(lambda (url, (_, rank)): (rank, url)) \
                           .top(10)

    # Step 7: Get URLs from top 10
    top_urls = sc.parallelize([url for (rank, url) in top_10])

    # Step 8: Join with titles
    url_title_map = top_urls.map(lambda url: (url, None)) \
                            .join(titles) \
                            .collectAsMap()

    # Step 9: Combine results
    results = []
    for (rank, url) in top_10:
        title = url_title_map.get(url, ("", "No Title"))[1]
        results.append({
            "url": url,
            "title": title,
            "score": rank
        })

    return results
```

### 17.4 Word Co-occurrence for Search Suggestions

**Goal:** Suggest related searches based on word co-occurrence.

```python
# Build co-occurrence matrix from web pages
def build_cooccurrence(HTMLRdd):
    """
    Build word co-occurrence matrix.

    Output: (word1, word2) → count
    """
    # For each page, get all word pairs
    word_pairs = HTMLRdd.flatMap(lambda (url, html):
        words = parseOutWords(html)
        # Generate all pairs
        pairs = []
        for i in range(len(words)):
            for j in range(i + 1, len(words)):
                if words[i] < words[j]:  # Normalize order
                    pairs.append(((words[i], words[j]), 1))
                else:
                    pairs.append(((words[j], words[i]), 1))
        return pairs
    )

    # Count co-occurrences
    cooccurrence = word_pairs.reduceByKey(lambda a, b: a + b)
    return cooccurrence


# Use for suggestions
def suggest_related_words(word, cooccurrence, top_n=5):
    """Find words that frequently co-occur with given word."""
    # Filter to pairs containing our word
    related = cooccurrence.filter(lambda ((w1, w2), count):
        w1 == word or w2 == word
    )

    # Extract the other word and count
    other_words = related.map(lambda ((w1, w2), count):
        (w2 if w1 == word else w1, count)
    )

    # Sort by count and take top N
    suggestions = other_words.top(top_n, key=lambda (word, count): count)
    return [word for (word, count) in suggestions]


# Example usage
cooccurrence = build_cooccurrence(HTMLRdd)
suggestions = suggest_related_words("spark", cooccurrence, 5)
# Might return: ["apache", "scala", "hadoop", "cluster", "rdd"]
```

### 17.5 Complete Search System

```python
# Initialize all components
HTMLRdd = sc.textFile("hdfs://web_crawl/*.html")  # Load web pages

# Build inverted index
invertedIndex = buildInvertedIndex(HTMLRdd)
invertedIndex.cache()  # Cache for fast lookups

# Extract titles
titles = HTMLRdd.mapValues(parseOutTitle)
titles.cache()

# Build link graph and compute PageRank
links = HTMLRdd.mapValues(parseOutLinks)
links.cache()
pageRanks = computePageRank(links, iterations=30)
pageRanks.cache()

# Build co-occurrence matrix for suggestions
cooccurrence = build_cooccurrence(HTMLRdd)
cooccurrence.cache()

# Search function
def search(query):
    """Main search function."""
    # Get results
    results = search_v2(query, invertedIndex, pageRanks, titles)

    # Get suggestions
    keywords = query.split()
    suggestions = []
    for kw in keywords:
        suggestions.extend(suggest_related_words(kw, cooccurrence, 3))

    return {
        "results": results,
        "suggestions": list(set(suggestions))
    }

# Example usage
search_results = search("apache spark")
print(f"Found {len(search_results['results'])} results")
print(f"Suggestions: {', '.join(search_results['suggestions'])}")
```

---

## Summary

### Key Takeaways from Lecture 2.2:

**1. Transformations on RDDs:**
- **Element-wise**: filter, map, flatMap
  - filter: Keep elements matching predicate
  - map: One-to-one transformation
  - flatMap: One-to-many transformation with flattening
- **Set operations**: distinct, union, intersection, subtract, cartesian, sample
  - Most require shuffle (expensive!)
  - union is cheap (no shuffle)

**2. Actions on RDDs:**
- **reduce(func)**: Aggregate using commutative & associative function
  - Two-level reduction: within partitions, then across partitions
- **aggregate(zeroVal, mergeValue, mergeComb)**: General aggregation
  - Accumulator type can differ from element type
  - Perfect for computing averages, statistics
- **Others**: collect, take, takeOrdered, top, takeSample, forEach, countByValue

**3. RDD Persistence:**
- **Why**: Avoid recomputation when RDD used multiple times
- **How**: persist() or cache() marks RDD; cached on first action
- **Levels**: MEMORY_ONLY, MEMORY_AND_DISK, DISK_ONLY, etc.
- **LRU eviction**: Automatically manages memory
- **Best for**: Iterative algorithms, interactive queries

**4. Pair RDDs (Key/Value):**
- **Creation**: map to tuples, keyBy()
- **Keys not unique**: Multiple values per key allowed
- **All regular operations** still work

**5. Pair RDD Transformations:**
- **mapValues**: Transform values, keep keys
- **reduceByKey**: Aggregate values by key (uses map-side combiner!)
- **combineByKey**: General aggregation with different accumulator type
  - createCombiner, mergeValue, mergeCombiner
  - Perfect for per-key averages
- **groupByKey**: Collect all values per key (⚠️ can cause OOM!)
- **cogroup**: Group values from two RDDs by key
- **subtractByKey**: Remove keys present in another RDD

**6. Joins:**
- **join (inner)**: Only keys in both RDDs
- **leftOuterJoin**: All keys from left, None for missing right
- **rightOuterJoin**: All keys from right, None for missing left
- Cross product when multiple values per key

**7. Other Operations:**
- **sortByKey**: Sort by keys (values ignored)
- **sampleByKey**: Different sampling rates per key

**8. Pair RDD Actions:**
- **countByKey**: Count per key
- **collectAsMap**: Convert to dictionary
- **lookup(key)**: Get all values for a key

**9. Case Study 1 - Web Crawl:**
- Extract titles: mapValues(parseTitle)
- Build inverted index: flatMap → filter → map → groupByKey
- Keyword → [URLs] mapping

**10. Case Study 2 - PageRank:**
- Build link graph: mapValues(parseLinks)
- Iterative algorithm: join, flatMap, reduceByKey, mapValues
- Damping factor: 0.15 + 0.85 × contributions
- 30 iterations to convergence

**11. Case Study 3 - Web Search:**
- Combine inverted index + PageRank
- Intersection of keyword results
- Sort by PageRank, return top N
- Word co-occurrence for suggestions

### Performance Tips:

1. **Use reduceByKey over groupByKey** whenever possible
2. **Cache/persist RDDs** that are used multiple times
3. **Avoid collect()** on large RDDs (OOM risk)
4. **Minimize shuffles** (distinct, groupByKey, joins are expensive)
5. **Use map-side combiners** (reduceByKey does this automatically)
6. **Choose right persistence level** based on memory/recomputation trade-off

### Common Patterns:

```python
# Word count
words.map(lambda w: (w, 1)).reduceByKey(lambda a, b: a + b)

# Per-key average
data.combineByKey(
    lambda v: (v, 1),
    lambda acc, v: (acc[0] + v, acc[1] + 1),
    lambda a1, a2: (a1[0] + a2[0], a1[1] + a2[1])
).mapValues(lambda (sum, count): sum / count)

# Join and filter
users.join(purchases).filter(lambda (id, (user, purchase)): purchase > 100)
```

---

## References

1. Zaharia, M., et al., "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing", USENIX NSDI, 2012

2. Karau, H., Konwinski, A., Wendell, P., Zaharia, M., "Learning Spark: Lightning-Fast Big Data Analysis", O'Reilly Media, 2015
   - Chapter 3: Programming with RDDs
   - Chapter 4: Working with Key/Value Pairs

3. Apache Spark Documentation:
   - RDD Programming Guide: https://spark.apache.org/docs/latest/rdd-programming-guide.html
   - Spark API Documentation: https://spark.apache.org/docs/latest/api/python/

4. Page, L., Brin, S., Motwani, R., Winograd, T., "The PageRank Citation Ranking: Bringing Order to the Web", Stanford InfoLab, 1999

5. Common Crawl: https://commoncrawl.org/

6. Secondary Sort in Spark: http://codingjunkie.net/spark-secondary-sort/

7. Course Materials:
   - Lecture Slides: DS256 L2.2 - Big Data Processing with Apache Spark
   - Professor: Yogesh Simmhan, IISc Bangalore

# Lecture 2.3: Spark Internals — Logical Plans, Physical Plans & Execution

## DS256 - Scalable Systems for Data Science
### Module 2: Processing Large Volumes of Big Data

---

## 1. Spark Architecture Overview

Before diving into Spark's internal execution model, let's understand the high-level architecture of how a Spark application runs on a cluster.

### 1.1 Components of a Spark Application

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SPARK APPLICATION                             │
│                                                                      │
│   ┌──────────────────────────┐                                       │
│   │      Driver Program      │                                       │
│   │  ┌────────────────────┐  │                                       │
│   │  │   SparkContext      │  │  ← Entry point for all Spark ops     │
│   │  │                    │  │  ← Coordinates with Cluster Manager   │
│   │  │  User Code:        │  │                                       │
│   │  │  - Define RDDs     │  │                                       │
│   │  │  - Transformations │  │                                       │
│   │  │  - Actions         │  │                                       │
│   │  └────────┬───────────┘  │                                       │
│   └───────────┼──────────────┘                                       │
│               │                                                      │
│               │  Logical Plan → Physical Plan → Task Scheduling      │
│               │                                                      │
│   ┌───────────▼──────────────────────────────────────────────────┐   │
│   │                    Cluster of Workers                         │   │
│   │                                                               │   │
│   │  ┌─────────────┐  ┌─────────────┐       ┌─────────────┐     │   │
│   │  │  Worker 1    │  │  Worker 2    │  ...  │  Worker N    │     │   │
│   │  │ ┌─────────┐  │  │ ┌─────────┐  │       │ ┌─────────┐  │     │   │
│   │  │ │Executor │  │  │ │Executor │  │       │ │Executor │  │     │   │
│   │  │ │ ┌─────┐ │  │  │ │ ┌─────┐ │  │       │ │ ┌─────┐ │  │     │   │
│   │  │ │ │Task │ │  │  │ │ │Task │ │  │       │ │ │Task │ │  │     │   │
│   │  │ │ │Task │ │  │  │ │ │Task │ │  │       │ │ │Task │ │  │     │   │
│   │  │ │ │Task │ │  │  │ │ │Task │ │  │       │ │ │Task │ │  │     │   │
│   │  │ │ └─────┘ │  │  │ │ └─────┘ │  │       │ │ └─────┘ │  │     │   │
│   │  │ └─────────┘  │  │ └─────────┘  │       │ └─────────┘  │     │   │
│   │  └─────────────┘  └─────────────┘       └─────────────┘     │   │
│   └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 The Two Plans

When you write a Spark application, Spark converts it into execution through **two key planning phases**:

| Phase | What It Does | Key Abstraction |
|-------|-------------|-----------------|
| **Logical Plan** | Determines the chain of RDDs and their dependencies (the "what") | RDD lineage graph with dependency types |
| **Physical Plan** | Determines how to execute the logical plan on the cluster (the "how") | Stages, Tasks, Shuffle boundaries |

Think of it this way:
- The **Logical Plan** is like an architect's blueprint — it describes the structure and relationships
- The **Physical Plan** is like the construction schedule — it describes the order, the workers, and how materials move

### 1.3 From User Code to Execution

```
User Code              Logical Plan            Physical Plan           Execution
─────────          ──────────────          ──────────────          ─────────────
                   
 RDD               RDD DAG with            Stages split            Tasks scheduled
 transformations → dependency edges    →   at shuffle          →   on Executors
 + action          (Narrow/Wide)           boundaries              (pipelined within
                                           + Tasks per stage       each stage)
```

---

## 2. Logical Plan: RDD Dependencies

The logical plan is the **computing chain** — a directed acyclic graph (DAG) of RDDs connected by dependencies. Understanding dependencies is the single most important concept for understanding Spark internals.

### 2.1 The General Logical Plan

Every Spark job follows this general structure:

```
Step 1:  Create initial RDDs         (from HDFS, memory, etc.)
         │
         ▼
Step 2:  Apply transformations       (map, filter, join, groupByKey, ...)
         │                           Each produces one or more new RDDs
         ▼
Step 3:  Call an action              (count, collect, save, foreach, ...)
         │                           Triggers actual computation
         ▼
Step 4:  Results sent to Driver      (driver applies a final function)
```

**Key insight:** Transformations are **lazy** — they just build up the logical plan (the DAG). No computation happens until an **action** is called. This is what allows Spark to optimize the entire plan before executing anything.

### 2.2 How RDDs Are Produced

Each `transformation()` produces **one or more new RDDs**. The number of RDDs produced is often **more than you'd expect**, because some transformations are internally composed of multiple sub-transformations.

For simple transformations, the mapping is straightforward:

| Transformation | RDD Produced | compute() Logic |
|:--------------|:-------------|:----------------|
| `map(f)` | MappedRDD | `iterator(partition).map(f)` |
| `filter(f)` | FilteredRDD | `iterator(partition).filter(f)` |
| `flatMap(f)` | FlatMappedRDD | `iterator(partition).flatMap(f)` |
| `mapPartitions(f)` | MapPartitionsRDD | `f(iterator(partition))` |
| `mapPartitionsWithIndex(f)` | MapPartitionsRDD | `f(partition.index, iterator(partition))` |
| `sample(...)` | PartitionwiseSampledRDD | `PoissonSampler.sample(iterator(partition))` |

For complex transformations (like `groupByKey`, `reduceByKey`, `join`, `distinct`), **multiple intermediate RDDs** are created internally. We'll examine these in detail below.

### 2.3 Every RDD Has Five Properties

Internally, each RDD is represented through a **common interface** that exposes five pieces of information (from the RDD paper):

| Property | Description | Example |
|----------|-------------|---------|
| `partitions()` | List of Partition objects (the atomic pieces of the dataset) | An HDFS-backed RDD has one partition per HDFS block |
| `dependencies()` | List of dependencies on parent RDDs | `map` → NarrowDependency; `groupByKey` → ShuffleDependency |
| `compute(partition, context)` | Function to compute elements of a partition given parent iterators | `parent.iterator(split).map(f)` for a MappedRDD |
| `preferredLocations(partition)` | Nodes where partition can be accessed faster (data locality) | HDFS block locations |
| `partitioner()` | Metadata about hash/range partitioning (if any) | `HashPartitioner(numPartitions)` |

---

## 3. Narrow vs. Wide Dependencies (The Most Important Concept)

This is the **foundational distinction** that drives everything in Spark's execution model — how stages are formed, how tasks are pipelined, how fault recovery works, and how shuffles happen.

### 3.1 Narrow Dependency (Full Dependency)

**Definition:** Each partition of the **parent RDD** is used by **at most one partition** of the child RDD.

Equivalently: each partition of the child RDD depends on a **small, fixed number** of entire parent partitions.

```
Parent RDD                  Child RDD
┌──────────┐               ┌──────────┐
│ Part 0   │──────────────▶│ Part 0   │    ← 1:1 mapping
├──────────┤               ├──────────┤
│ Part 1   │──────────────▶│ Part 1   │    ← Each parent partition
├──────────┤               ├──────────┤       is fully consumed by
│ Part 2   │──────────────▶│ Part 2   │       exactly one child partition
└──────────┘               └──────────┘
```

**Key properties of Narrow Dependencies:**
- Parent and child partitions **can be on the same worker** — no data needs to move across the network
- Multiple narrow-dependency transformations can be **pipelined** in a single task (no intermediate data stored)
- Fault recovery is **cheap**: only the lost partition (and its specific parent) needs to be recomputed
- **No shuffle required**

**Types of Narrow Dependencies:**

| Type | Pattern | Example Transformations |
|------|---------|------------------------|
| **OneToOneDependency** (1:1) | Each child partition depends on exactly one parent partition | `map()`, `filter()`, `flatMap()`, `mapPartitions()` |
| **RangeDependency** (1:1 with ranges) | Retains partition boundaries from parent | `union()` |
| **N:1 NarrowDependency** | Multiple parent partitions → one child partition | `coalesce(shuffle=false)`, co-partitioned `join()` |

### 3.2 Wide Dependency (Shuffle Dependency / Partial Dependency)

**Definition:** Each partition of the **parent RDD** may be depended upon by **multiple partitions** of the child RDD. Equivalently: each child partition depends on **a part of** (not the entire) each parent partition.

```
Parent RDD                  Child RDD
┌──────────┐          ┌───▶┌──────────┐
│ Part 0   │─────┬────┤    │ Part 0   │    ← Each child partition
├──────────┤     │    └───▶├──────────┤       needs a PIECE of
│ Part 1   │──┬──┼────┬───▶│ Part 1   │       EVERY parent partition
├──────────┤  │  │    │    ├──────────┤
│ Part 2   │──┼──┴────┼───▶│ Part 2   │    ← This is a SHUFFLE!
└──────────┘  │       │    └──────────┘
              └───────┘
```

**Key properties of Wide Dependencies:**
- Data must **move across the network** (shuffle) — records from one parent partition go to different child partitions
- The shuffle is a **barrier**: all tasks on the upstream side must complete before downstream tasks can start
- Intermediate shuffle data is **materialized to disk** (not pipelined)
- Fault recovery is **expensive**: a single failed node might cause the loss of data needed by ALL child partitions, requiring recomputation of the entire parent stage
- **Shuffles are costly!** They involve disk I/O, network communication, serialization/deserialization, and barrier synchronization

**Examples of Wide Dependencies:**

| Transformation | Why It's Wide |
|---------------|---------------|
| `groupByKey()` | Records with the same key from ALL parent partitions must be gathered into ONE child partition |
| `reduceByKey()` | Same as above, but with map-side combine |
| `sortByKey()` | Records must be repartitioned by key ranges |
| `distinct()` | Duplicate records could be in any parent partition |
| `join()` (non-co-partitioned) | Matching keys from two RDDs could be in any partition |
| `repartition()` | Explicitly redistributes data |

### 3.3 Visual Summary: Narrow vs Wide

```
NARROW DEPENDENCIES                          WIDE DEPENDENCIES
(Black arrows — no shuffle)                  (Red arrows — SHUFFLE!)

  ┌───┐    ┌───┐                              ┌───┐    ┌───┐
  │ A │───▶│ B │   1:1 (map, filter)          │ A │╲  ╱│ B │
  └───┘    └───┘                              └───┘ ╲╱ └───┘
                                              ┌───┐ ╱╲ ┌───┐
  ┌───┐    ┌───┐                              │   │╱  ╲│   │
  │ A │───▶│   │                              └───┘    └───┘
  └───┘    │ B │   Range (union)
  ┌───┐    │   │                              groupByKey, reduceByKey,
  │ C │───▶│   │                              sortByKey, join, distinct
  └───┘    └───┘

  ┌───┐    ┌───┐
  │ A │───▶│   │
  └───┘    │ B │   N:1 (co-partitioned join,
  ┌───┐    │   │        coalesce)
  │ C │───▶│   │
  └───┘    └───┘

Key rule: If parent partitions' records go to
EXACTLY ONE child partition → Narrow
If they split across MULTIPLE children → Wide
```

### 3.4 When Does a Wide Dependency Become Narrow?

An important subtlety: some transformations that are **usually wide** can become **narrow** if certain conditions are met:

**`join(otherRDD)` can be narrow if:**
- Both input RDDs have the **same partitioner** (e.g., both HashPartitioner)
- AND the **same number of partitions**
- This is called a **co-partitioned join** (or hash join)
- Since records with the same key are already on the same partition, no shuffle is needed!

**`cogroup(otherRDD)` can be narrow if:**
- Same partitioner type AND same number of partitions for all input RDDs

This is why **Spark tracks the partitioner** used to generate each RDD — it can use this information to avoid unnecessary shuffles. For example:

```python
# RDD A was created with HashPartitioner(3)
# RDD B was created with HashPartitioner(3)
# join(A, B) → NarrowDependency (no shuffle needed!)

# RDD A uses HashPartitioner(3)
# RDD C uses RangePartitioner(3)
# join(A, C) → ShuffleDependency for at least one of them
```

---

## 4. Detailed Logical Plans for Complex Transformations

Many transformations are internally more complex than they appear. Understanding their internal RDD chains is crucial for performance tuning.

### 4.1 groupByKey(numPartitions)

```
                    ShuffleDependency
Input RDD ─────────────────────────────────▶ ShuffledRDD
RDD[(K,V)]          (shuffle all            RDD[(K, Iterable[V])]
                     records by key)              │
                                                  │ OneToOneDependency
                                                  ▼
                                            MapPartitionsRDD
                                            RDD[(K, Iterable[V])]
                                            (cast ArrayBuffer → Iterable)
```

**Important:** `groupByKey()` has **NO map-side combine**. Why?
- Map-side combine for `groupByKey` would insert all records into a hash table on the map side
- This doesn't reduce the amount of data shuffled (all records must still be sent)
- It just creates many objects in the Old Generation of the JVM heap, causing GC pressure
- So it's actually **worse** to do map-side combine for `groupByKey`!

### 4.2 reduceByKey(func, numPartitions)

```
                 OneToOneDependency           ShuffleDependency          OneToOneDependency
Input RDD ─────────────────────────▶ MapPartitionsRDD ──────────────▶ ShuffledRDD ──────────────▶ MapPartitionsRDD
RDD[(K,V)]                          (map-side combine)               (shuffled data)             (reduce-side aggregate)
```

**Key difference from `groupByKey`:** `reduceByKey` **does** perform map-side combine!
- Before shuffle: applies the reduce function locally to combine values with the same key
- After shuffle: applies the reduce function again to combine across partitions
- This is equivalent to having a Combiner in MapReduce
- **Result: significantly less data is shuffled**

```
Example: Word Count — reduceByKey vs groupByKey

Data: [(a,1), (b,1), (a,1), (b,1), (a,1)]  across 2 partitions

With groupByKey:
  Partition 1: (a,1), (b,1), (a,1)  ──shuffle──▶  (a, [1,1,1,1])  then sum
  Partition 2: (b,1), (a,1)         ──shuffle──▶  (b, [1,1])       then sum
  Shuffled records: 5 (ALL records moved)

With reduceByKey:
  Partition 1: (a,1), (b,1), (a,1) → combine → (a,2), (b,1)  ──shuffle──▶  (a, 2+2=4)
  Partition 2: (b,1), (a,1)        → combine → (a,1), (b,1)  ──shuffle──▶  (b, 1+1=2)
  Shuffled records: 4 (FEWER records moved — and even fewer with more data)
```

### 4.3 distinct(numPartitions)

`distinct()` deduplicates records. Since duplicates can exist in different partitions, a shuffle is needed.

```
                  OneToOne          (internally uses reduceByKey)          OneToOne
Input RDD ──────────────────▶ MappedRDD ──────────────────────────────▶ MapPartitionsRDD
RDD[T]      map(x → (x, null))  RDD[(T,null)]   reduceByKey            RDD[(T,null)]
                                                 (shuffle + dedup)           │
                                                                             │ map(x → x._1)
                                                                             ▼
                                                                        MappedRDD
                                                                        RDD[T]
```

**Steps:**
1. Map each record `x` to `(x, null)` — transforms RDD[T] into RDD[(K,V)] format required for shuffle
2. `reduceByKey` with a no-op combiner — this shuffles by key and keeps only one copy per key (deduplication)
3. Map back: extract just the key from `(key, null)` to get the deduplicated RDD[T]

### 4.4 cogroup(otherRDD, numPartitions)

`cogroup()` groups records from **two or more RDDs** by key. It's the building block for `join()`, `intersection()`, and other multi-RDD operations.

```
RDD a ──────────┐
RDD[(K,V)]      │     ShuffleDependency      OneToOneDependency
                ├───▶ CoGroupedRDD ─────────▶ MapPartitionsRDD
RDD b ──────────┘     or                      RDD[(K, (Iterable[V], Iterable[W]))]
RDD[(K,W)]            OneToOneDependency
                      (depends on partitioners)
```

**Critical question: Is the dependency Narrow or Wide?**

The dependency between each parent RDD and CoGroupedRDD depends on TWO factors:

| Factor | Narrow (OneToOne) | Wide (Shuffle) |
|--------|-------------------|----------------|
| **Same # of partitions?** | ✓ Required | ✗ Different counts |
| **Same partitioner type?** | ✓ Required | ✗ Different types |

```
Example scenarios:

Scenario 1: Both RDDs have HashPartitioner(3)
  RDD a (Hash, 3 parts) ──OneToOne──▶ CoGroupedRDD (Hash, 3 parts)
  RDD b (Hash, 3 parts) ──OneToOne──▶

Scenario 2: Different partitioners
  RDD a (Range, 3 parts) ──OneToOne──▶ CoGroupedRDD (Range, 3 parts)
  RDD b (Hash, 3 parts)  ──Shuffle───▶

Scenario 3: Different partition counts
  RDD a (Hash, 3 parts) ──Shuffle──▶ CoGroupedRDD (Hash, 4 parts)
  RDD b (Hash, 3 parts) ──Shuffle──▶
```

### 4.5 join(otherRDD, numPartitions)

`join()` performs an inner join of two `RDD[(K,V)]` and `RDD[(K,W)]`. Internally, it uses `cogroup()` as the building block:

```
                    cogroup                        mapValues              flatMap
RDD a ──┐                                    
        ├──▶ CoGroupedRDD ──▶ MappedValuesRDD ──────────────────────▶ FlatMappedValuesRDD
RDD b ──┘    RDD[(K,(Iter[V],  RDD[(K,(Iter[V],   Cartesian product    RDD[(K, (V, W))]
              Iter[W]))]        Iter[W]))]         of V and W values
```

**Steps:**
1. `cogroup()` groups both RDDs by key → `RDD[(K, (Iterable[V], Iterable[W]))]`
2. For each key, compute the **Cartesian product** of the two Iterables
3. `flatMap()` flattens the results

**Performance:** If both input RDDs are hash-partitioned with the same partitioner, the cogroup step uses OneToOneDependency (no shuffle needed) — this is called a **hash join**.

### 4.6 sortByKey(ascending, numPartitions)

```
                    ShuffleDependency                  OneToOneDependency
Input RDD ──────────────────────────────▶ ShuffledRDD ──────────────────────▶ MapPartitionsRDD
RDD[(K,V)]    uses RangePartitioner       (records partitioned              (records sorted within
              to determine partition       by key ranges)                    each partition)
              boundaries
```

**How it works:**
1. A **RangePartitioner** is used — it samples the RDD to determine partition boundaries (e.g., partition 0 gets keys A-F, partition 1 gets G-M, etc.)
2. Shuffle distributes records to the correct partition based on key range
3. Within each partition, records are sorted using **TimSort** (a hybrid merge-sort/insertion-sort)
4. The final result: records across all partitions are in global sorted order

### 4.7 union(otherRDD)

```
RDD a ──┐     RangeDependency (1:1)
        ├───▶ UnionRDD
RDD b ──┘     (simply concatenates partition lists)
```

- `union()` **never moves data** — it just creates a new RDD whose partitions are the union of the parent partitions
- Uses `RangeDependency` to retain the borders of original RDDs
- **Very cheap operation** — no shuffle, no data copy

### 4.8 cartesian(otherRDD)

```
RDD a (m partitions) ──┐
                        ├──▶ CartesianRDD (m × n partitions)
RDD b (n partitions) ──┘
```

- Output has `m × n` partitions — the i-th partition of CartesianRDD depends on partition `i/n` of RDD a and partition `i%n` of RDD b
- **NarrowDependency** — each parent partition is fully consumed
- But each partition of RDD a or RDD b is used by multiple child partitions (this looks like a wide dependency but it's technically narrow because each parent partition is used **in its entirety** — no partial dependency)

### 4.9 coalesce(numPartitions, shuffle)

Reduces or increases the number of partitions:

```
Case 1: coalesce(3, shuffle=false) — Decrease partitions only
                    NarrowDependency (N:1)
RDD (5 partitions) ────────────────────────▶ CoalescedRDD (3 partitions)
                    (just groups parent       
                     partitions together)     

Case 2: coalesce(10, shuffle=true) — Can increase partitions
                    OneToOne                 ShuffleDependency           
RDD (5 partitions) ──────────▶ MappedRDD ───────────────────▶ ShuffledRDD ──▶ CoalescedRDD
                   (assign     (records      (shuffle by        (10 partitions)
                    increasing  with keys)    hash of key)
                    keys via
                    round-robin)
```

- `shuffle=false`: Can only **decrease** partitions (merging). NarrowDependency. No network I/O.
- `shuffle=true`: Can increase or decrease. Assigns monotonically increasing keys in a round-robin fashion, then shuffles by hash of key for uniform distribution.
- `repartition(n)` is exactly `coalesce(n, shuffle=true)`

### 4.10 The Primitive: combineByKey()

Many shuffle-based transformations (`groupByKey`, `reduceByKey`, `aggregateByKey`) are all internally implemented using **`combineByKey()`**, which is the most general aggregation primitive in Spark:

```scala
def combineByKey[C](
    createCombiner: V => C,          // First record with a key: create initial combiner
    mergeValue: (C, V) => C,         // Subsequent records: merge value into combiner
    mergeCombiners: (C, C) => C,     // After shuffle: merge combiners from diff. partitions
    partitioner: Partitioner,
    mapSideCombine: Boolean = true
): RDD[(K, C)]
```

**How it works step by step:**

```
Partition 1: (a,1), (b,2), (a,3)           Partition 2: (a,4), (b,5)

Step 1: createCombiner — first occurrence of each key
  (a,1) → combiner_a = createCombiner(1) = 1
  (b,2) → combiner_b = createCombiner(2) = 2

Step 2: mergeValue — subsequent occurrences of same key
  (a,3) → combiner_a = mergeValue(1, 3) = 4
  (a,4) → combiner_a' = createCombiner(4) = 4  [on partition 2]
  (b,5) → combiner_b' = createCombiner(5) = 5  [on partition 2]

  After map-side combine:
  Partition 1: (a, 4), (b, 2)
  Partition 2: (a, 4), (b, 5)

Step 3: Shuffle by key

Step 4: mergeCombiners — combine results from different partitions
  key a: mergeCombiners(4, 4) = 8
  key b: mergeCombiners(2, 5) = 7
```

**Why this matters:** Understanding `combineByKey` explains why `reduceByKey` (which uses it with `mapSideCombine=true`) is more efficient than `groupByKey` (which uses it with `mapSideCombine=false`).

---

## 5. Physical Plan: From Logical DAG to Stages and Tasks

The physical plan answers: **"Given this DAG of RDDs, how do we actually execute it on a cluster?"**

### 5.1 The Problem: How to Execute a Complex DAG?

Consider a complex DAG with many RDDs connected by both narrow and wide dependencies:

```
      RDD_A                                    RDD_E
        │ (narrow)                               │ (narrow)
      RDD_B                                    RDD_F
        │ (narrow)                             ╱
      RDD_C ──────────(wide/shuffle)──────▶ RDD_G
                                               │ (narrow)
                                             RDD_H (final)
```

**Naive approach 1: One task per RDD pair**
- Create a task for every arrow in the graph
- Problem: **too many intermediate results stored** — every RDD would be materialized

**Naive approach 2: One giant task for everything**
- Try to compute everything in a single task from the final RDD backwards
- Problem: **shuffle dependency blocks pipelining** — you can't pipeline across a shuffle because you need ALL parent partitions to compute ANY child partition

### 5.2 The Solution: Cut at Shuffle Boundaries → Stages

**Spark's strategy:**

> **Check backwards from the final RDD. Add each NarrowDependency into the current stage. Break out for a new stage when there's a ShuffleDependency.**

```
Stage 2                          Stage 1                          Stage 0 (final)
┌─────────────────────┐         ┌─────────────────────┐         ┌─────────────────────┐
│                     │         │                     │         │                     │
│  RDD_A              │         │  RDD_E              │         │  RDD_G              │
│    │ (narrow)       │         │    │ (narrow)        │         │    │ (narrow)       │
│  RDD_B              │         │  RDD_F              │         │  RDD_H (final)      │
│    │ (narrow)       │         │                     │         │                     │
│  RDD_C              │         │                     │         │                     │
│                     │         │                     │         │                     │
└──────────┬──────────┘         └──────────┬──────────┘         └─────────────────────┘
           │                               │                              ▲
           │        SHUFFLE                │         SHUFFLE              │
           └───────────────────────────────┴──────────────────────────────┘
```

**Rules for stage creation:**
1. Start from the **final RDD** (the one on which the action was called)
2. Walk **backwards** through the lineage graph
3. Keep adding RDDs to the current stage while following **NarrowDependencies**
4. When you hit a **ShuffleDependency**, **cut** — the RDD on the other side starts a new parent stage
5. Recurse on the parent stages

**Stage numbering:** Since stages are determined backwards, the **last stage** (containing the final RDD) gets **id 0**.

### 5.3 Tasks: The Unit of Execution

Within each stage, the number of tasks is determined by the **number of partitions in the last RDD of the stage**.

```
Stage 2 (3 partitions in RDD_C)     →  3 ShuffleMapTasks
Stage 1 (2 partitions in RDD_F)     →  2 ShuffleMapTasks
Stage 0 (4 partitions in RDD_H)     →  4 ResultTasks
```

**Two types of tasks:**

| Task Type | When Used | What It Does |
|-----------|-----------|-------------|
| **ShuffleMapTask** | Stages that produce intermediate shuffle output | Computes partition data, partitions it by key, writes shuffle output to local disk for the next stage to fetch |
| **ResultTask** | The final stage that produces the job's result | Computes partition data and sends the result back to the driver |

**Analogy to MapReduce:**
- `ShuffleMapTask` ≈ Mapper (produces partitioned output for the next stage)
- `ResultTask` ≈ Reducer (produces the final result) when it reads from a shuffle, or Mapper when the stage has no parents

### 5.4 Pipelining Within a Stage

Within a single stage (all NarrowDependencies), Spark **pipelines** the computation. This is the key optimization that makes Spark much more efficient than approaches that materialize every intermediate RDD.

**Pipelining means: no intermediate data is stored.**

```
Without pipelining (bad):                    With pipelining (Spark):
─────────────────────────                    ──────────────────────

for record in input:                         for record in input:
    temp1 = f(record)                            result = g(f(record))
    store temp1                                  emit result
                                             
for record in temp1:                         // record, f(record) are immediately
    result = g(record)                       // garbage collected after g() runs
    emit result                              // No intermediate storage needed!
```

**At the record level, pipelining looks like this:**

```
Record 1 ──▶ f(record1) ──▶ g(f(record1)) ──▶ emit
Record 2 ──▶ f(record2) ──▶ g(f(record2)) ──▶ emit
Record 3 ──▶ f(record3) ──▶ g(f(record3)) ──▶ emit
   ...
```

Each record flows through the **entire chain** of transformations within the stage before the next record starts. This is equivalent to:

```scala
for (record <- records) {
    g(f(record))     // f and g are pipelined — no intermediate storage
}
```

**Why pipelining stops at shuffle boundaries:**
- With a ShuffleDependency, computing ONE partition of the child RDD requires data from ALL partitions of the parent RDD
- You can't stream records one-by-one from parent to child — you need the complete shuffle output first
- So shuffle data must be **materialized** (written to disk), creating a barrier between stages

**Exception — not all narrow dependencies can be fully pipelined:**

Some transformations within a narrow dependency require consuming ALL records before producing output (e.g., `sortByKey` within a partition, or `mapPartitions` where `f` needs all records). In these cases, intermediate results must be stored in memory within the task, but still no data moves across the network.

### 5.5 Complete Example: From Logical to Physical Plan

Let's trace a complete example:

```scala
val data1 = sc.parallelize(Array((1,'a'), (2,'b'), (3,'c'), (4,'d')), 3)
val hashPairs1 = data1.partitionBy(new HashPartitioner(3))

val data2 = sc.parallelize(Array((1,"A"), (2,"B"), (3,"C"), (4,"D")), 2)
val rangePairs2 = data2.map(x => (x._1, x._2.charAt(0)))

val data3 = sc.parallelize(Array((1,'X'), (2,'Y')), 2)

val rangePairs = rangePairs2.union(data3)

val result = hashPairs1.join(rangePairs)

result.foreach(println)    // ACTION triggers execution
```

**Logical Plan (DAG):**

```
ParallelCollectionRDD (data1)
     │ (narrow)
 ShuffledRDD (partitionBy → HashPartitioner(3))     ← shuffle boundary
     │
     │                  ParallelCollectionRDD (data2)
     │                       │ (narrow)
     │                  MappedRDD (map)
     │                       │                  ParallelCollectionRDD (data3)
     │                       │ (narrow)              │ (narrow)
     │                  UnionRDD (union) ◄────────────┘
     │                       │
     └───── join (cogroup) ──┘                    ← shuffle boundary (for UnionRDD side)
                  │ (narrow)
           MappedValuesRDD
                  │ (narrow)
          FlatMappedValuesRDD (final)
```

**Physical Plan (Stages):**

```
Stage 2: ParallelCollectionRDD → ShuffledRDD
         (3 ShuffleMapTasks — writes shuffle output)

Stage 1: ParallelCollectionRDD(data2) → MappedRDD ─┐
         ParallelCollectionRDD(data3) ──────────────┤→ UnionRDD
         (4 ShuffleMapTasks — writes shuffle output)

Stage 0: CoGroupedRDD → MappedValuesRDD → FlatMappedValuesRDD
         (3 ResultTasks — produces final output)
         [This stage starts AFTER Stages 1 and 2 complete]
```

**Execution order:**
1. **Stage 2** and **Stage 1** can execute **in parallel** (no dependency between them)
2. **Stage 0** waits for both Stage 1 and Stage 2 to complete
3. Stage 0 fetches shuffle output from Stages 1 and 2, then pipelines through CoGroupedRDD → MappedValuesRDD → FlatMappedValuesRDD

---

## 6. Job Creation and Scheduling

### 6.1 Jobs, Stages, and Tasks Hierarchy

```
Application
├── Job 1 (triggered by action 1, e.g., count())
│   ├── Stage 0 (ResultStage)
│   │   ├── ResultTask 0
│   │   ├── ResultTask 1
│   │   └── ResultTask 2
│   ├── Stage 1 (ShuffleMapStage)
│   │   ├── ShuffleMapTask 0
│   │   └── ShuffleMapTask 1
│   └── Stage 2 (ShuffleMapStage)
│       ├── ShuffleMapTask 0
│       ├── ShuffleMapTask 1
│       └── ShuffleMapTask 2
├── Job 2 (triggered by action 2, e.g., saveAsTextFile())
│   ├── Stage 3 (ResultStage)
│   └── Stage 4 (ShuffleMapStage)
└── ...
```

**Key relationships:**
- **One application** = one SparkContext = one driver program
- **Each `action()`** in the driver creates **one job**
- Each job is split into **stages** at shuffle boundaries
- Each stage contains **tasks** (one per output partition)
- A **TaskSet** is the collection of all tasks in a stage

### 6.2 How Actions Create Jobs

| Action | processPartition (per partition) | resultHandler (combine results) |
|--------|-------------------------------|-------------------------------|
| `reduce(func)` | Reduce records within partition → partial result | Reduce partial results across partitions |
| `collect()` | Return array of records | Concatenate all arrays |
| `count()` | Count records in partition | Sum all counts |
| `foreach(f)` | Apply f to each record | No combination needed |
| `take(n)` | Take up to n records | Take first n from results |
| `saveAsHadoopFile(path)` | Write records to HDFS | No result returned |
| `countByKey()` | Count per key in partition → Map | Merge maps |

### 6.3 The DAGScheduler: Orchestrating Execution

The DAGScheduler is the brain of Spark's execution engine. Here's the complete flow:

```
User calls action (e.g., rdd.count())
         │
         ▼
   DAGScheduler.runJob(rdd, processPartition, resultHandler)
         │
         ▼
   Create JobId, submit JobSubmitted event
         │
         ▼
   handleJobSubmitted():
     1. newStage() — walk backwards from final RDD,
        cut at ShuffleDependencies → create stage DAG
     2. submitStage(finalStage)
         │
         ▼
   submitStage(stage):
     1. Find missingParentStages (parent stages not yet computed)
     2. If parents missing → recursively submit parents first,
        add current stage to waitingStages
     3. If no parents missing → submitMissingTasks(stage)
         │
         ▼
   submitMissingTasks(stage):
     1. Create ShuffleMapTask or ResultTask for each partition
     2. Package into TaskSet
     3. Submit to TaskScheduler
         │
         ▼
   TaskScheduler.submitTasks(taskSet):
     1. Wrap in TaskSetManager
     2. Add to scheduling queue (FIFO or Fair)
     3. backend.reviveOffers() → DriverActor sends tasks to Executors
         │
         ▼
   Executor receives task, deserializes, runs compute() chain
```

### 6.4 Task Scheduling: FIFO vs Fair Scheduler

| Scheduler | Behavior | Use Case |
|-----------|----------|----------|
| **FIFO** (default) | First job submitted runs first. Later jobs wait. | Simple batch processing, single-user |
| **Fair** | All jobs get a fair share of resources. Jobs run concurrently. | Multi-user, interactive queries |

---

## 7. Shuffle Mechanisms

Shuffle is the most expensive operation in Spark. Let's understand how it works internally.

### 7.1 Hash Shuffle (Spark < 1.2)

In the original hash shuffle, each map task creates **one file per reducer**:

```
Map Task 0:  ┌── file_0_0 (for reducer 0)
             ├── file_0_1 (for reducer 1)
             └── file_0_2 (for reducer 2)

Map Task 1:  ┌── file_1_0 (for reducer 0)
             ├── file_1_1 (for reducer 1)
             └── file_1_2 (for reducer 2)

Map Task 2:  ┌── file_2_0 (for reducer 0)
             ├── file_2_1 (for reducer 1)
             └── file_2_2 (for reducer 2)

Total files = M × R  (M map tasks × R reducers)
```

**Problem:** With 1000 mappers and 1000 reducers → **1,000,000 files!** This creates enormous disk I/O, file system overhead, and memory pressure for file handles.

### 7.2 Sort Shuffle (Spark >= 1.2, Default)

Sort shuffle creates **one file per map task** with an index of offsets for each reducer:

```
Map Task 0:  ┌── shuffle_0.data  (single sorted file)
             └── shuffle_0.index (offsets for each reducer partition)
             
             shuffle_0.data:
             ┌─────────────────┬─────────────────┬─────────────────┐
             │  Reducer 0 data │  Reducer 1 data │  Reducer 2 data │
             └─────────────────┴─────────────────┴─────────────────┘
                    ▲                  ▲                  ▲
             index: 0                1024              2048

Total files = M × 2  (M map tasks × 2 files each: data + index)
```

**Uses TimSort** — a hybrid of merge sort and insertion sort, efficient for real-world data with existing order.

**Much better:** With 1000 mappers → only **2000 files** (vs. 1,000,000).

### 7.3 Why Shuffle Is Costly

```
┌──────────────────────────────────────────────────────────────────┐
│                    SHUFFLE COST BREAKDOWN                         │
│                                                                  │
│  1. DISK I/O                                                     │
│     - Map side: serialize + write shuffle output to local disk   │
│     - Reduce side: read fetched data from local disk             │
│                                                                  │
│  2. NETWORK COMMUNICATION                                        │
│     - Reduce tasks fetch shuffle data from ALL map tasks         │
│     - Data crosses the network (potentially across racks)        │
│                                                                  │
│  3. SERIALIZATION / DESERIALIZATION                              │
│     - Objects must be serialized for network transfer            │
│     - Deserialized on the reduce side                            │
│                                                                  │
│  4. BARRIER SYNCHRONIZATION                                      │
│     - ALL map tasks must complete before ANY reduce task starts   │
│     - One slow map task delays the entire next stage             │
│                                                                  │
│  5. MEMORY PRESSURE                                              │
│     - Shuffle buffers, sort buffers consume memory               │
│     - Can trigger spills to disk if memory is insufficient       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. Worker and Executor Model

### 8.1 Hierarchy

```
Cluster
├── Worker 1 (one physical/virtual machine)
│   └── Executor 1 (one JVM process)
│       ├── Thread Pool
│       │   ├── Thread → Task A (one output partition)
│       │   ├── Thread → Task B (one output partition)
│       │   └── Thread → Task C (one output partition)
│       ├── Block Manager (manages cached RDD partitions)
│       └── Shuffle Manager (coordinates shuffle I/O)
│
├── Worker 2
│   └── Executor 2
│       ├── Thread Pool
│       │   ├── Thread → Task D
│       │   └── Thread → Task E
│       ├── Block Manager
│       └── Shuffle Manager
│
└── Worker N
    └── Executor N
        └── ...
```

### 8.2 Key Design Decisions

| Component | Design | Why |
|-----------|--------|-----|
| **Worker** | One machine | Physical resource boundary |
| **Executor** | One JVM process per Worker | Memory isolation, independent GC |
| **Task** | One thread within Executor | Lightweight — thread creation overhead is much lower than process creation |

**Each task is responsible for computing ONE output partition.** The total number of concurrent tasks an executor can run = number of cores allocated to it.

### 8.3 Block Manager

Each Executor has a **Block Manager** that manages:
- **Cached/persisted RDD partitions** (in memory and/or on disk)
- **Shuffle output blocks** (written by ShuffleMapTasks)
- **Broadcast variables** (shared read-only data sent to all executors)

The Block Manager coordinates with the **DAGScheduler** for:
- Determining data locality (schedule tasks where the data already is)
- Managing persistence levels (MEMORY_ONLY, MEMORY_AND_DISK, DISK_ONLY, etc.)

---

## 9. Spark Tuning

### 9.1 Controlling Parallelism

```
More Partitions → More Tasks → More Parallelism → Faster Execution
(up to the number of available cores)
```

| What to Control | How to Control It | When to Use |
|----------------|-------------------|-------------|
| **Input partitions** | `sc.textFile(path, minPartitions)` | Increase if input is small but cluster is large |
| **Shuffle partitions** | `numPartitions` parameter in wide transforms | Default may be too few for large datasets |
| **Repartition** | `rdd.repartition(n)` or `rdd.coalesce(n)` | After filter reduces data size, or to increase parallelism |
| **Spark SQL shuffle partitions** | `spark.sql.shuffle.partitions` (default: 200) | Adjust based on data size |

### 9.2 Data Partitioning Strategies

**Why partition matters:** Controlling partitioning can **eliminate shuffles entirely** for subsequent operations.

```python
# BAD: Join without pre-partitioning → full shuffle on both sides
result = large_rdd.join(small_rdd)   # 2 shuffles

# GOOD: Pre-partition large RDD, then join → shuffle only small RDD
large_rdd = large_rdd.partitionBy(HashPartitioner(100)).persist()
result = large_rdd.join(small_rdd)   # only 1 shuffle (small_rdd)

# BEST: Pre-partition both → no shuffle at all
large_rdd = large_rdd.partitionBy(HashPartitioner(100)).persist()
small_rdd = small_rdd.partitionBy(HashPartitioner(100)).persist()
result = large_rdd.join(small_rdd)   # 0 shuffles (co-partitioned!)
```

**Spark tracks partitioners:**
- Transformations like `partitionBy`, `groupByKey`, `reduceByKey`, `join`, `sort` **set** a partitioner on the output RDD
- `map()` and `flatMap()` **unset** the partitioner (because the user function might change the key)
- `filter()`, `mapValues()`, `flatMapValues()` **retain** the partitioner (they don't change the key)
- You can check with `rdd.partitioner` — returns `Some(HashPartitioner(n))` or `None`

### 9.3 Built-in Partitioners

| Partitioner | How It Works | When to Use |
|-------------|-------------|-------------|
| **HashPartitioner** | `partition = hash(key) % numPartitions` | Default. Good for uniformly distributed keys |
| **RangePartitioner** | Samples data, creates roughly equal key ranges per partition | `sortByKey()`. Good when you need ordered output |
| **Custom Partitioner** | User defines `numPartitions` and `getPartition(key)` | Domain-specific co-location (e.g., partition URLs by domain) |

### 9.4 Caching and Persistence

```python
# Cache in memory (default: MEMORY_ONLY)
rdd.cache()              # Alias for persist(MEMORY_ONLY)

# Persist with specific storage level
rdd.persist(StorageLevel.MEMORY_AND_DISK)
rdd.persist(StorageLevel.DISK_ONLY)
rdd.persist(StorageLevel.MEMORY_ONLY_SER)     # Serialized, more compact
rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)
```

**When to cache/persist:**
- ✅ RDD is used **across multiple actions** (e.g., iterative algorithms like PageRank)
- ✅ RDD is **expensive to recompute** (long chain of transformations, or involves a shuffle)
- ✅ RDD is used by **multiple downstream transformations** that branch the DAG
- ❌ RDD is too large to fit in memory (will cause eviction thrashing)
- ❌ RDD is cheap to recompute (e.g., simple map on an in-memory dataset)

**Eviction policy:** LRU (Least Recently Used) at the RDD partition level. If a new partition can't fit, the least recently used partition from a **different RDD** is evicted. Partitions from the same RDD are not evicted (to avoid cycling).

### 9.5 Memory Allocation

```
┌─────────────────────────────────────────────────────────┐
│                 Executor JVM Heap                        │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │         Unified Memory (60% of heap)              │   │
│  │                                                    │   │
│  │  ┌──────────────────┐ ┌────────────────────────┐  │   │
│  │  │ Execution Memory │ │    Storage Memory      │  │   │
│  │  │  (shuffle, join,  │ │  (cached RDDs,         │  │   │
│  │  │   sort buffers)   │ │   broadcast vars)      │  │   │
│  │  │                  │ │                        │  │   │
│  │  │  ◄── can borrow ──►                        │  │   │
│  │  │      from each   ──►                        │  │   │
│  │  │      other        │ │                        │  │   │
│  │  └──────────────────┘ └────────────────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌────────────────────────────────┐                     │
│  │   User Memory (40% of heap)    │ ← User data        │
│  │   + Reserved Memory (300 MB)   │ ← Buffer for OOM   │
│  └────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

- **Execution memory** and **Storage memory** can **borrow from each other** dynamically
- If execution needs more memory and storage has unused space, execution can borrow it (and vice versa)
- Execution memory can evict storage (cached data can be recomputed), but storage cannot evict execution (shuffle data cannot be recomputed without re-running the task)

### 9.6 Static vs. Dynamic Resource Allocation

| Mode | Behavior | Trade-off |
|------|----------|-----------|
| **Static** | Fixed number of executors allocated at submission time | Guaranteed resources, but wastes resources during idle phases |
| **Dynamic** | Executors added/removed based on task queue demand and idle time | Better utilization, but startup latency when scaling up |

---

## 10. RDD Fault Tolerance via Lineage

One of the most elegant aspects of Spark's design is how it handles fault tolerance through **lineage** rather than data replication.

### 10.1 How Lineage-Based Recovery Works

```
Original execution:
  HDFS Block → RDD_A → RDD_B → RDD_C → RDD_D (result)
                (map)   (filter)  (map)

If a partition of RDD_C is lost (executor crash):
  1. Spark checks RDD_C's lineage: "I was created by filter() on RDD_B"
  2. Checks RDD_B's lineage: "I was created by map() on RDD_A"
  3. Checks RDD_A's lineage: "I was created from HDFS block X"
  4. Recomputes: read HDFS block X → map → filter → RDD_C partition recovered!

Only the LOST PARTITION is recomputed, not the entire RDD.
Recomputation can happen on a DIFFERENT node.
```

### 10.2 Narrow vs. Wide Dependencies and Fault Recovery

```
NARROW DEPENDENCY RECOVERY (Cheap):
  Lost: Partition 2 of RDD_C
  Need to recompute: Only Partition 2 of RDD_B → Partition 2 of RDD_C
  Other partitions: UNAFFECTED

WIDE DEPENDENCY RECOVERY (Expensive):
  Lost: Partition 2 of ShuffledRDD
  Need to recompute: ALL partitions of the parent RDD that contributed
                     to Partition 2 of ShuffledRDD
  This means: Re-run ALL map tasks of the parent stage!
  That's why: Shuffle output is WRITTEN TO DISK (not just kept in memory)
              → so it can survive executor failures without full re-execution
```

### 10.3 When to Checkpoint

For very long lineage chains (e.g., iterative algorithms with hundreds of iterations), recomputation from the beginning would be too expensive. In such cases, use **checkpointing**:

```python
# Save RDD to reliable storage (HDFS), truncating its lineage
sc.setCheckpointDir("hdfs://...")
rdd.checkpoint()    # Must be called before any action on this RDD
```

After checkpointing, the RDD's lineage is replaced with "I was loaded from HDFS checkpoint", so recovery is fast regardless of the original lineage length.

**Difference from persist/cache:**
- `cache()/persist()` stores data in executor memory/disk — **lost if executor dies**
- `checkpoint()` stores data in HDFS — **survives executor failures** — but is slower to write

---

## 11. Putting It All Together: Complete Execution Walkthrough

Let's trace the complete execution of a word count job:

```python
lines = sc.textFile("hdfs://input.txt")           # HadoopRDD (4 partitions)
words = lines.flatMap(lambda l: l.split())         # FlatMappedRDD (narrow)
pairs = words.map(lambda w: (w, 1))                # MappedRDD (narrow)
counts = pairs.reduceByKey(lambda a, b: a + b)     # ShuffledRDD + MapPartitionsRDD (wide)
counts.saveAsTextFile("hdfs://output")             # ACTION → triggers execution
```

### Step 1: Build Logical Plan

```
HadoopRDD ──(narrow)──▶ FlatMappedRDD ──(narrow)──▶ MappedRDD ──(SHUFFLE)──▶ ShuffledRDD ──(narrow)──▶ MapPartitionsRDD
                                                                                                              │
                                                                                                         saveAsTextFile
```

### Step 2: Create Physical Plan (Stages)

```
Stage 1 (ShuffleMapStage):                     Stage 0 (ResultStage):
┌────────────────────────────────────┐        ┌────────────────────────────┐
│ HadoopRDD                         │        │ ShuffledRDD                │
│   │ flatMap (pipelined)            │        │   │ mapPartitions          │
│ FlatMappedRDD                     │        │ MapPartitionsRDD           │
│   │ map (pipelined)               │        │   │ saveAsTextFile         │
│ MappedRDD                         │──────▶ │ (write to HDFS)           │
│   │ map-side combine (pipelined)  │shuffle │                            │
│ MapPartitionsRDD                  │        │ 4 ResultTasks              │
│                                    │        └────────────────────────────┘
│ 4 ShuffleMapTasks                 │
└────────────────────────────────────┘
```

### Step 3: Execute Stage 1 (4 ShuffleMapTasks in parallel)

```
Task 0 (on Worker A):
  Read HDFS block 0
  → flatMap(split) → pipelined, no intermediate storage
  → map(w → (w,1)) → pipelined
  → map-side combine: {(hello,3), (world,2), ...}
  → Write shuffle output to local disk (sorted by partition key)

Task 1 (on Worker B): same for HDFS block 1
Task 2 (on Worker C): same for HDFS block 2
Task 3 (on Worker A): same for HDFS block 3

All 4 tasks run in parallel. Records are PIPELINED (streamed one-by-one
through flatMap → map → combine). No intermediate RDD is materialized.
```

### Step 4: Execute Stage 0 (4 ResultTasks, after Stage 1 completes)

```
Task 0 (on Worker B):
  Fetch shuffle partition 0 from all map tasks (across network)
  → Merge-combine: reduce values for each key
  → Write result partition to HDFS

Task 1 (on Worker C): same for shuffle partition 1
Task 2 (on Worker A): same for shuffle partition 2
Task 3 (on Worker B): same for shuffle partition 3
```

### Step 5: Job Complete

All 4 result partitions written to HDFS. `saveAsTextFile` returns.

---

## 12. Summary: Key Concepts Cheat Sheet

### Dependencies:

```
┌─────────────────────────────────────────────────────────────────────┐
│  NARROW (no shuffle)              │  WIDE (shuffle required)        │
│  ─────────────────               │  ────────────────────           │
│  map, filter, flatMap            │  groupByKey, reduceByKey        │
│  mapPartitions, mapValues        │  sortByKey, distinct            │
│  union, coalesce(shuffle=false)  │  join (non-co-partitioned)      │
│  co-partitioned join/cogroup     │  repartition, coalesce(true)    │
│  cartesian                       │  intersection                   │
│                                  │                                  │
│  ✓ Pipelined within stage        │  ✗ Barrier between stages       │
│  ✓ No network I/O               │  ✗ Disk + network I/O           │
│  ✓ Cheap fault recovery          │  ✗ Expensive fault recovery     │
└─────────────────────────────────────────────────────────────────────┘
```

### Execution Hierarchy:

```
Application ──has──▶ Jobs ──split into──▶ Stages ──contain──▶ Tasks
   │                  │                    │                    │
   │                  │                    │                    │
 1 per              1 per               Cut at               1 per
 SparkContext       action()            shuffle              output
                                        boundaries           partition
```

### Performance Rules:

1. **Minimize shuffles** — they are by far the most expensive operation
2. **Use `reduceByKey` over `groupByKey`** — map-side combine reduces shuffle data
3. **Pre-partition and persist** — co-partitioned RDDs avoid shuffles on joins
4. **Cache wisely** — cache RDDs reused across actions, not one-time RDDs
5. **Control partition count** — too few = underutilized cluster; too many = scheduling overhead
6. **Use `mapValues`/`flatMapValues`** instead of `map` when possible — preserves partitioner
7. **Checkpoint long lineage chains** — prevents expensive recomputation in iterative algorithms
8. **Pipeline-friendly transformations** (map, filter) are essentially free within a stage

---

## References

1. Zaharia, M., Chowdhury, M., Das, T., Dave, A., Ma, J., McCauley, M., Franklin, M.J., Shenker, S., Stoica, I., "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing", USENIX NSDI, 2012

2. Armbrust, M., Xin, R.S., Lian, C., Huai, Y., Liu, D., Bradley, J.K., Meng, X., Kaftan, T., Franklin, M.J., Ghodsi, A., Zaharia, M., "Spark SQL: Relational Data Processing in Spark", ACM SIGMOD, 2015

3. Lijie Xu (Jerry Lead), "Spark Internals", https://github.com/JerryLead/SparkInternals
   - Chapter 2: Job Logical Plan
   - Chapter 3: Job Physical Plan

4. Jacek Laskowski, "The Internals of Apache Spark", https://books.japila.pl/apache-spark-internals/overview/

5. Karau, H., Konwinski, A., Wendell, P., Zaharia, M., "Learning Spark: Lightning-Fast Big Data Analysis", O'Reilly Media, 2nd Edition, Chapter 7

6. Apache Spark Documentation:
   - RDD Programming Guide: https://spark.apache.org/docs/latest/rdd-programming-guide.html
   - Spark Configuration: https://spark.apache.org/docs/latest/configuration.html

7. Course Materials:
   - Lecture Slides: DS256 L2.3 — Big Data Processing with Apache Spark
   - Professor: Yogesh Simmhan, IISc Bangalore

# Lecture 2.4: Spark DataFrames, SQL & Catalyst Optimizer

## DS256 - Scalable Systems for Data Science
### Module 2: Processing Large Volumes of Big Data

---

## 1. Limitations of Spark RDD

Before understanding DataFrames, it is critical to understand **why** they were needed — i.e., the fundamental limitations of the RDD abstraction that motivated a higher-level API.

### 1.1 Why RDDs Fall Short

Spark RDDs only offer high-level constructs on **iterations over RDD items** and **invocation patterns**. The core problems are:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    LIMITATIONS OF SPARK RDDs                         │
│                                                                      │
│  1. OPAQUE LAMBDA EXPRESSIONS                                       │
│     ─────────────────────────                                       │
│     • User functions (lambdas) are "black boxes" to Spark           │
│     • Spark cannot inspect, analyze, or optimize them               │
│     • No automatic query optimization is possible                   │
│                                                                      │
│  2. OPAQUE TYPES                                                    │
│     ──────────────                                                  │
│     • RDD elements are arbitrary Java/Python objects                │
│     • Spark only knows they are homogeneous (same type)             │
│     • No type-specific behavior or storage optimization             │
│                                                                      │
│  3. IMPERATIVE PROGRAMMING MODEL                                    │
│     ────────────────────────────                                    │
│     • Users tell Spark HOW to execute (step by step)                │
│     • Relies entirely on users to optimize code                     │
│     • Spark engine has no freedom to rearrange operations           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Example of the Problem:**

```python
# RDD approach — Spark sees opaque functions, cannot optimize
rdd.filter(lambda x: x.age > 21) \
   .map(lambda x: (x.name, x.age)) \
   .reduceByKey(lambda a, b: a + b)

# Spark has NO IDEA:
#   - What "x.age > 21" means (it's just arbitrary Python code)
#   - That only "name" and "age" columns are needed (not all fields)
#   - How to reorder or combine these operations for efficiency
```

Because Spark cannot look inside user functions, it cannot apply classic database optimizations like **predicate pushdown**, **projection pruning**, or **constant folding** — optimizations that relational databases have used for decades.

---

## 2. DataFrames

### 2.1 What Are DataFrames?

**DataFrames** are a higher-level abstraction built on top of RDDs, inspired by the `pandas` DataFrame in Python and data frames in R. While RDDs were inspired by Python native operators like `map`, DataFrames bring a **declarative, SQL-like** programming model to Spark.

```
┌──────────────────────────────────────────────────────────────────────┐
│                      RDD vs. DataFrame                               │
│                                                                      │
│   RDD:                           DataFrame:                         │
│   ────                           ──────────                         │
│   • "How to do it"               • "What to do"                    │
│   • Opaque user functions        • SQL-like DSL operators           │
│   • Row-based                    • Row AND column based             │
│   • User must optimize           • Spark optimizes for you          │
│   • Different APIs per language  • Uniform across languages         │
│                                                                      │
│   Key Insight: DataFrames let Spark PARSE your query,               │
│   UNDERSTAND your intention, and OPTIMIZE execution.                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Core Design Principles:**
- **More expressive and simpler**: Compose SQL-like queries using a high-level Domain Specific Language (DSL) and APIs
- **Tell Spark what to do**: Spark parses the query, understands the intention, and optimizes/arranges operations for efficient execution
- **Uniform across language bindings**: Avoids user-defined opaque code as much as possible, so Python, Scala, Java, and R all get the same performance
- **Interoperable**: You can drop down to the RDD level whenever you need fine-grained control

### 2.2 RDD vs. Datasets vs. DataFrames

```
┌──────────────────────────────────────────────────────────────────────┐
│                 RDD vs. Dataset vs. DataFrame                        │
│                                                                      │
│  ┌─────────┐    ┌───────────┐    ┌─────────────┐                    │
│  │   RDD   │    │  Dataset   │    │  DataFrame  │                    │
│  └─────────┘    └───────────┘    └─────────────┘                    │
│                                                                      │
│  Immutable       Immutable         Immutable                        │
│  Distributed     Distributed       Distributed                      │
│  Collection      Collection        Collection                       │
│                                                                      │
│  • Lower-level   • Strongly-typed  • Data organized into            │
│  • More control     (Java/Scala      named columns                  │
│  • More coding      only)          • Imposes structure              │
│    required      • Easier debug    • Domain specific                │
│  • Row-based     • Compact           language API                   │
│                    bytecode        • Untyped at compile             │
│                    (Tungsten)        time (types checked             │
│                                      at runtime)                    │
│                                                                      │
│  PERFORMANCE: DataFrame ≥ Dataset >> RDD                            │
│  Both DataFrame and Dataset use Catalyst Optimizer + Tungsten       │
│                                                                      │
│  INTEROPERABILITY: dataset.rdd.take(10)  ← Move between them       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Static Typing and Runtime Type Safety:**
- **Dataset**: Easier to debug due to strong typing (compile-time type checks in Java/Scala)
- **RDD** on Java/Scala: Also strongly typed, but no optimizer benefits
- **DataFrame**: Untyped at compile time — types are checked at runtime, but faster for interactive/ad-hoc queries

**Performance Hierarchy:**
- **Dataset and DataFrame** use SparkSQL's **Catalyst optimizer** for speed and space efficiency
- **Dataset** is efficient for batch execution with specialized **Tungsten serialization/deserialization** and compact bytecode
- **DataFrame** is untyped and **faster for interactive** workloads
- **RDD** gives the most **fine-grained control** but leaves optimization entirely to the user

---

## 3. Declarative Programming

DataFrames embody the **declarative programming** paradigm — the same philosophy used by SQL and relational database management systems (RDBMS).

```
┌──────────────────────────────────────────────────────────────────────┐
│          IMPERATIVE vs. DECLARATIVE PROGRAMMING                      │
│                                                                      │
│  Imperative (RDD):                Declarative (DataFrame/SQL):      │
│  ─────────────────                ────────────────────────────       │
│  "Tell HOW to do it"              "Tell WHAT to do"                 │
│                                                                      │
│  rdd.filter(lambda x: x > 3)     SELECT Part, Items                │
│     .map(lambda x: (x, 1))       FROM Widget                       │
│     .reduceByKey(add)             WHERE Part = 'Bolts'              │
│                                                                      │
│  User specifies each step.        System determines the best        │
│  User must optimize.              execution plan using the          │
│                                   schema of the data/table.         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Why Declarative Matters:**
- The **system** (Spark's Catalyst optimizer) determines the best execution plan
- Uses the **schema** (column names, types, sizes) for the data to plan execution
- Enables automatic optimizations like predicate pushdown, column pruning, join reordering, etc.

---

## 4. Working with DataFrames

### 4.1 DataFrames as Distributed In-Memory Tables

DataFrames are **distributed in-memory tables** with named columns and schemas.

**Key Properties:**
- Schema can be **defined explicitly** by the user
- **Immutable** — Spark keeps a lineage of all transformations (just like RDDs)
- Adding or changing column names/types **creates new DataFrames** while previous versions are preserved
- Backed by RDDs under the hood

### 4.2 Defining Schema Using DDL

You can define a schema explicitly using DDL (Data Definition Language) syntax:

```python
# Define schema using DDL string
schema = "name STRING, age INT, city STRING"

# Or using StructType
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("name", StringType(), True),
    StructField("age", IntegerType(), True),
    StructField("city", StringType(), True)
])

df = spark.createDataFrame(data, schema)
```

### 4.3 Projections and Filters

**Projections** select specific columns, and **filters** select specific rows:

```python
# Projection: Select specific columns
df.select("name", "age")

# Filter: Select specific rows
df.where(df.age > 21)
# or equivalently
df.filter(df.age > 21)

# Combine projection and filter
df.select("name", "age").where(df.age > 21)
```

### 4.4 Aggregation and Join

```python
# Aggregation with groupBy
# Note: count() here is the aggregator for groupBy(), NOT the action df.count()
df.groupBy("city").count()

# Join two DataFrames
airports = ...  # DataFrame with City, State, Country, IATA columns
flights = ...   # DataFrame with Origin, Destination, Date, Delay, Distance, Airline columns

# Join flights with airport info
flights.join(airports, flights.Origin == airports.IATA)
```

**Example Data Model:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Airports Table:                                                     │
│  ┌──────────┬───────┬─────────┬──────┐                               │
│  │  City    │ State │ Country │ IATA │                               │
│  ├──────────┼───────┼─────────┼──────┤                               │
│  │ Seattle  │  WA   │  USA    │ SEA  │                               │
│  │ New York │  NY   │  USA    │ JFK  │                               │
│  │ Bangalore│  KA   │  India  │ BLR  │                               │
│  └──────────┴───────┴─────────┴──────┘                               │
│                                                                      │
│  Flights Table:                                                      │
│  ┌────────┬──────┬────────────┬───────┬──────────┬─────────┐         │
│  │ Origin │ Dest │    Date    │ Delay │ Distance │ Airline │         │
│  ├────────┼──────┼────────────┼───────┼──────────┼─────────┤         │
│  │  SEA   │ SFO  │ 01010710   │  31   │   590    │   AA    │         │
│  │  SEA   │ SFO  │ 01010955   │  104  │   590    │   UA    │         │
│  │  SEA   │ SFO  │ 01010730   │   5   │   590    │   AA    │         │
│  │  LAX   │ SFO  │ 01010600   │  15   │   400    │   DL    │         │
│  └────────┴──────┴────────────┴───────┴──────────┴─────────┘         │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.5 Lazy Evaluation in DataFrames

Just like RDDs, DataFrames use **lazy evaluation**:
- Transformations (select, filter, join, groupBy) build up a **logical plan** but do **not** execute anything
- Execution only occurs when an **action** (output operation) is called, such as `show()`, `count()`, `save()`, or `collect()`
- This allows the Catalyst optimizer to see the **entire plan** and optimize across all operations

### 4.6 Immutability vs. Modifications

DataFrames themselves are **immutable** (backed by immutable RDDs). However, you can create **new, different DataFrames** from existing ones:

```python
# Add a column — creates a NEW DataFrame
df_with_status = df.withColumn("status", df.delay > 0)

# Drop columns
df_slim = df.drop("delay")

# Rename columns
df_renamed = df.withColumnRenamed("delay", "flight_delay")
```

**Important:** ACID (Atomicity, Consistency, Isolation, Durability) properties do **not** apply since DataFrames are immutable — there are no in-place updates to worry about.

---

## 5. Spark SQL

### 5.1 Overview

**Spark SQL** provides an **ANSI SQL:2003-compatible** query interface over structured data with a schema. It allows users to write standard SQL queries that Spark then optimizes and executes.

**Key Features:**
- Permits abstraction to DataFrames/Datasets
- Connects to **Apache Hive**, **JSON**, **CSV**, **Parquet**, and other data sources
- Accessible via **JDBC/ODBC** and a **SQL Shell**
- Generates **optimized query plans** via the Catalyst optimizer

```python
# Register a DataFrame as a temporary view
df.createOrReplaceTempView("flights")

# Run SQL queries directly
result = spark.sql("""
    SELECT Origin, avg(Delay) as avg_delay 
    FROM flights 
    WHERE Distance > 500 
    GROUP BY Origin
""")
```

### 5.2 Joins in Spark SQL

Spark SQL supports all standard join types:

```python
# SQL-style join
spark.sql("""
    SELECT f.*, a.City, a.Country
    FROM flights f
    JOIN airports a ON f.Origin = a.IATA
""")

# DataFrame API join
flights.join(airports, flights.Origin == airports.IATA, "inner")
```

---

## 6. Execution Model

### 6.1 From High-Level APIs to Low-Level Execution

The computation expressed in high-level DataFrame or SQL APIs is **decomposed into low-level optimized and generated RDD operations**:

```
┌──────────────────────────────────────────────────────────────────────┐
│                       EXECUTION MODEL                                │
│                                                                      │
│  User Code (DataFrame / SQL)                                        │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────────┐                                                │
│  │  Catalyst        │  Parse → Analyze → Optimize → Plan            │
│  │  Optimizer        │                                               │
│  └────────┬─────────┘                                                │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │  Generated RDD   │  Optimized, compact, single-function          │
│  │  Operations      │  RDD code                                     │
│  └────────┬─────────┘                                                │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │  Scala Bytecode  │  Compiled for executors' JVMs                 │
│  └─────────────────┘                                                │
│                                                                      │
│  IMPORTANT:                                                         │
│  • Generated RDD operation code is NOT accessible to users          │
│  • These RDD ops are NOT the same as the user-facing RDD APIs       │
│  • They are internal, optimized operations generated by Catalyst    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 7. Catalyst Optimizer — Overview

The **Catalyst Optimizer** is the heart of Spark SQL's execution engine. It is what makes DataFrames and Spark SQL fast — it takes user operations and transforms them into an optimized execution plan.

### 7.1 Core Idea

```
┌──────────────────────────────────────────────────────────────────────┐
│                     CATALYST OPTIMIZER                                │
│                                                                      │
│  • DataFrames keep track of their schema                            │
│  • DataFrames support various relational operations                 │
│  • A DataFrame represents a LOGICAL PLAN to compute a dataset      │
│  • NO execution occurs until an "action" output operation is        │
│    called (e.g., save, show, collect)                               │
│  • Enables RICH OPTIMIZATION across ALL operations used to          │
│    build the DataFrame                                              │
│  • User operations are captured using an Abstract Syntax Tree       │
│    (AST) rather than opaque Python/Scala functions                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 7.2 SQL Stack and Query Planning at a Glance

The Catalyst optimizer transforms a SQL or DataFrame expression through multiple phases to arrive at optimized physical execution:

```
┌──────────────────────────────────────────────────────────────────────┐
│                   QUERY PLANNING PIPELINE                            │
│                                                                      │
│   SQL Query ─────┐                                                  │
│                  ▼                                                   │
│   DataFrame ──▶ Unresolved    ──▶  Logical   ──▶  Optimized         │
│   DSL           Logical Plan       Plan           Logical Plan      │
│                                                                      │
│                      ▲                               │               │
│                      │                               ▼               │
│                   Catalog            ┌──────────────────────────┐    │
│                                      │  Physical   Physical     │    │
│                                      │  Plan 1     Plan 2  ... │    │
│                                      └───────────┬──────────────┘   │
│                                                  │                   │
│                                           Cost Model                │
│                                                  │                   │
│                                                  ▼                   │
│                                        Selected Physical Plan       │
│                                                  │                   │
│                                           Code Generation           │
│                                                  │                   │
│                                                  ▼                   │
│                                               RDDs                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

The pipeline consists of **four major phases**: Analysis, Logical Optimization, Physical Planning, and Code Generation. Each phase transforms a tree of nodes — expressions, logical operators, or physical operators.

---

## 8. Catalyst Optimizer — Deep Dive (from SIGMOD 2015 Paper)

*Reference: "Spark SQL: Relational Data Processing in Spark," Michael Armbrust et al., ACM SIGMOD 2015.*

The Catalyst optimizer is an **extensible query optimizer** built using functional programming constructs in Scala. Its design had two key motivations:
1. Make it easy to **add new optimization techniques and features** to Spark SQL
2. Enable **external developers to extend** the optimizer (e.g., data-source-specific rules, new data types)

Unlike previous extensible optimizers that required a complex domain-specific language (DSL) for specifying rules and an "optimizer compiler" to translate them into executable code, Catalyst uses **standard Scala features** — particularly **pattern matching** — allowing developers to write optimization rules in a full programming language while keeping them concise and readable.

### 8.1 Trees — The Core Data Structure

At its core, Catalyst represents everything as **trees**. The main data type is a tree composed of **node objects**, where each node has a **node type** and **zero or more children**. New node types are defined as subclasses of a `TreeNode` class. These tree objects are **immutable** and can be manipulated using **functional transformations**.

**Example — A Simple Expression Language:**

Consider three node classes for a simple expression language:
- `Literal(value: Int)` — a constant value
- `Attribute(name: String)` — an attribute from an input row (e.g., `"x"`)
- `Add(left: TreeNode, right: TreeNode)` — sum of two expressions

The expression `x + (1 + 2)` is represented as a tree:

```
┌──────────────────────────────────────────────────────────────────────┐
│                  TREE FOR: x + (1 + 2)                               │
│                                                                      │
│                       Add                                           │
│                      /   \                                          │
│              Attribute    Add                                       │
│              (name="x")  /   \                                      │
│                     Literal  Literal                                │
│                     (val=1)  (val=2)                                │
│                                                                      │
│  In Scala: Add(Attribute(x), Add(Literal(1), Literal(2)))          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

This tree representation is used for **everything** in Catalyst: expressions, logical plans, physical plans — they are all trees of typed nodes.

### 8.2 Rules — Transforming Trees

**Rules** are functions that transform one tree into another. While a rule can run arbitrary code on the tree (since it is just a Scala object), the most common approach is to use **pattern matching** functions that **find and replace subtrees** with a specific structure.

Trees offer a `transform` method that applies a pattern-matching function **recursively on all nodes** of the tree, transforming the ones that match each pattern to a result.

**Example — Constant Folding Rule:**

```scala
// Rule: If we see Add(Literal(c1), Literal(c2)), replace with Literal(c1+c2)
tree.transform {
  case Add(Literal(c1), Literal(c2)) => Literal(c1 + c2)
}

// Applied to x + (1 + 2):
//
//        Add                        Add
//       /   \          ──▶         /   \
//  Attr(x)  Add               Attr(x)  Literal(3)
//          /   \
//     Lit(1)  Lit(2)
//
// Result: x + 3
```

**Key Properties of Rules and Pattern Matching:**

1. **Partial functions**: A rule only needs to match a **subset** of all possible input trees. Catalyst automatically **skips and descends into** subtrees that do not match. This means rules are **modular** — they only reason about the cases where a given optimization applies.

2. **Multiple patterns in one transform call**: Rules can match multiple patterns simultaneously, making it concise to implement several transformations at once:

```scala
tree.transform {
  case Add(Literal(c1), Literal(c2)) => Literal(c1 + c2)   // Constant folding
  case Add(left, Literal(0))         => left                 // x + 0 = x
  case Add(Literal(0), right)        => right                // 0 + x = x
}
```

3. **Batches and fixed-point execution**: Rules are grouped into **batches**. Each batch is executed repeatedly until the tree reaches a **fixed point** — i.e., the tree stops changing after applying its rules. This means each rule can be **simple and self-contained**, yet still achieve larger **global effects** through repeated application. For example, repeated application would constant-fold `(x+0)+(3+3)` → `(x)+(6)` → `x+6`.

4. **Sanity checks**: After each batch, developers can run sanity checks on the new tree (e.g., verifying that all attributes have been assigned types), often also written via recursive matching.

5. **Arbitrary Scala code**: Rule conditions and bodies can contain **arbitrary Scala code**, giving Catalyst more power than domain-specific optimizer languages, while keeping simple rules concise.

6. **Easy to reason about and debug**: Functional transformations on immutable trees make the entire optimizer easy to understand and debug. They also enable parallelization (though this is not yet exploited).

### 8.3 The Four Phases of Query Planning

Catalyst uses its tree transformation framework in **four phases**:

```
┌──────────────────────────────────────────────────────────────────────┐
│               FOUR PHASES OF CATALYST QUERY PLANNING                 │
│                                                                      │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐    │
│   │ Analysis │──▶│ Logical  │──▶│ Physical │──▶│    Code      │    │
│   │          │   │Optimiz.  │   │ Planning │   │ Generation   │    │
│   └──────────┘   └──────────┘   └──────────┘   └──────────────┘    │
│                                                                      │
│   Resolve       Apply rule-     Generate        Compile parts       │
│   references    based           physical        of the query        │
│   using         optimizations   plans; use      to Java             │
│   catalog                       cost model      bytecode            │
│                                 to select                           │
│                                                                      │
│   Purely        Purely          Rule-based      Code                │
│   Rule-based    Rule-based      + Cost-based    Generation          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

Each phase uses **different types of tree nodes** — Catalyst includes libraries for expressions, data types, and logical and physical operators.

---

#### 8.3.1 Phase 1: Analysis

**Goal:** Resolve all references in the logical plan — figure out what each column name refers to and what type it has.

Spark SQL begins with a relation to be computed, either from:
- An **Abstract Syntax Tree (AST)** returned by a SQL parser, or
- A **DataFrame** object constructed using the API

In both cases, the relation may contain **unresolved attribute references or relations**. For example, in `SELECT col FROM sales`, the type of `col`, or even whether it is a valid column name, is not known until we look up the table `sales`.

An attribute is **unresolved** if we do not know its type or have not matched it to an input table (or an alias).

**What the Analyzer Does:**

The analyzer uses Catalyst rules and a **Catalog** object (which tracks all tables in all data sources) to resolve attributes. It builds an "unresolved logical plan" tree and then applies rules that:

```
┌──────────────────────────────────────────────────────────────────────┐
│                      ANALYSIS RULES                                  │
│                                                                      │
│  1. LOOK UP RELATIONS BY NAME                                       │
│     Find the actual table/data source from the catalog              │
│     e.g., "sales" → the actual sales table metadata                 │
│                                                                      │
│  2. MAP NAMED ATTRIBUTES                                            │
│     Map attributes like "col" to the actual input provided          │
│     by a given operator's children                                  │
│     e.g., "col" → column #3 of the "sales" table                   │
│                                                                      │
│  3. DETERMINE ATTRIBUTE IDENTITY                                    │
│     Determine which attributes refer to the same value and          │
│     give them a unique ID                                           │
│     e.g., "col = col" → both refer to the same column              │
│     (enables optimizations like eliminating redundant checks)       │
│                                                                      │
│  4. PROPAGATE AND COERCE TYPES                                      │
│     Propagate and coerce types through expressions                  │
│     e.g., "1 + col" — we cannot know the return type of this       │
│     until we resolve "col" and possibly cast subexpressions         │
│     to compatible types (e.g., INT + FLOAT → FLOAT)                │
│                                                                      │
│  Total: ~1000 lines of code for all analyzer rules                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Example:**

```sql
SELECT (col + 1) FROM sales
```

The analyzer must:
1. Look up `sales` in the catalog → find the table
2. Find `col` in the `sales` table → resolve it to a specific column
3. Determine the type of `col` (e.g., `INT`)
4. Infer the type of `col + 1` (e.g., `INT + INT → INT`)
5. Can also infer schema for semi-structured data (e.g., JSON)

---

#### 8.3.2 Phase 2: Logical Optimization

**Goal:** Apply standard rule-based optimizations to simplify and improve the logical plan.

This phase applies well-known database optimization techniques. These include:

```
┌──────────────────────────────────────────────────────────────────────┐
│                 LOGICAL OPTIMIZATION RULES                           │
│                                                                      │
│  1. CONSTANT FOLDING                                                │
│     Replace expressions with constants when possible                │
│     e.g., 1 + 2 → 3                                                │
│                                                                      │
│  2. PREDICATE PUSHDOWN                                              │
│     Push filter conditions as close to the data source as possible  │
│     e.g., Filter AFTER Join → Filter BEFORE Join (fewer rows to    │
│     join)                                                           │
│                                                                      │
│  3. PROJECTION PRUNING                                              │
│     Only read the columns that are actually needed                  │
│     e.g., SELECT name FROM users → only read the "name" column,    │
│     skip all other columns                                          │
│                                                                      │
│  4. NULL PROPAGATION                                                │
│     Simplify expressions involving NULL                             │
│     e.g., NULL + x → NULL, NULL AND x → NULL (in some cases)       │
│                                                                      │
│  5. BOOLEAN EXPRESSION SIMPLIFICATION                               │
│     Simplify boolean logic                                          │
│     e.g., x AND true → x, x OR false → x                          │
│                                                                      │
│  Total: ~800 lines of code for all logical optimization rules       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Why Trees + Rules Make This Easy:**

Adding new optimization rules is remarkably simple. For example, when the team added the fixed-precision `DECIMAL` type to Spark SQL, they wanted to optimize aggregations (SUM, AVG) on small-precision DECIMALs. The entire rule was only **12 lines of code**:

```scala
// Simplified version: optimize SUM on small DECIMALs
// Cast to unscaled 64-bit LONG, aggregate, then convert back
object DecimalAggregates extends Rule[LogicalPlan] {
  val MAX_LONG_DIGITS = 18
  def apply(plan: LogicalPlan): LogicalPlan = {
    plan transformAllExpressions {
      case Sum(e @ DecimalType.Expression(prec, scale))
        if prec + 10 <= MAX_LONG_DIGITS =>
        MakeDecimal(Sum(LongValue(e)), prec + 10, scale)
    }
  }
}
```

This converts high-precision decimal math to fast 64-bit long integer math when possible — a significant performance win.

Similarly, a **12-line rule** optimizes `LIKE` expressions with simple regular expressions into `String.startsWith` or `String.contains` calls, which are much faster. The freedom to use arbitrary Scala code in rules makes these optimizations straightforward.

**Predicate Pushdown — Visualized:**

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PREDICATE PUSHDOWN                                 │
│                                                                      │
│  BEFORE Optimization:            AFTER Optimization:                │
│                                                                      │
│     Project(name)                   Project(name)                   │
│         │                               │                           │
│     Filter(age > 21)                Join(id = id)                   │
│         │                           /          \                    │
│     Join(id = id)          Filter(age > 21)   Scan(orders)          │
│     /          \                 │                                   │
│  Scan(users)  Scan(orders)   Scan(users)                            │
│                                                                      │
│  The filter is pushed DOWN past the join to reduce the number       │
│  of rows that enter the join — much less data to process!           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

#### 8.3.3 Phase 3: Physical Planning

**Goal:** Convert the optimized logical plan into one or more physical plans that can be executed on the Spark engine (using RDD operations), and select the best one.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PHYSICAL PLANNING                                 │
│                                                                      │
│  Optimized Logical Plan                                             │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────────────────────────────────────┐                    │
│  │  Physical Planner generates MULTIPLE plans  │                    │
│  │                                              │                    │
│  │  Plan A: SortMergeJoin + HashAggregate      │                    │
│  │  Plan B: BroadcastHashJoin + HashAggregate  │                    │
│  │  Plan C: ShuffledHashJoin + SortAggregate   │                    │
│  └─────────────────────────────────────────────┘                    │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────────────────────────────────────┐                    │
│  │  Cost Model evaluates each plan             │                    │
│  │  (estimates table sizes, selectivity, etc.) │                    │
│  │                                              │                    │
│  │  → Select CHEAPEST plan                     │                    │
│  └─────────────────────────────────────────────┘                    │
│         │                                                            │
│         ▼                                                            │
│  Selected Physical Plan                                             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Physical planning involves two types of optimization:**

1. **Rule-based physical optimizations:**
   - **Pipelining**: Combine projections or filters into a single Spark RDD `map` operation (avoid multiple passes over the data)
   - **Push operations into data sources**: For data sources that support predicate or projection pushdown (e.g., a JDBC source can run a `WHERE` clause directly on the database)

2. **Cost-based optimization (CBO):**
   - Currently used primarily for **selecting join algorithms**
   - For relations known to be **small**, Spark SQL uses a **broadcast join** (sends the small table to all nodes via a peer-to-peer broadcast facility)
   - For large relations, it may choose **sort-merge join** or **shuffled hash join**
   - **How table sizes are estimated:**
     - From the **in-memory cache** (if the table is cached)
     - From **external file sizing** (e.g., Parquet file metadata)
     - From the **result of a subquery** with a `LIMIT`
   - Costs can be estimated **recursively** for a whole tree using a rule

**Total: ~500 lines of code for physical planning rules.**

---

#### 8.3.4 Phase 4: Code Generation

**Goal:** Compile parts of the query into **Java bytecode** for maximum execution speed.

This is where Catalyst really shines for performance. Since Spark SQL often operates on **in-memory datasets**, processing is **CPU-bound** (not I/O bound). Code generation eliminates the overhead of interpretation.

**The Problem with Interpretation:**

```
┌──────────────────────────────────────────────────────────────────────┐
│           INTERPRETED vs. CODE-GENERATED EVALUATION                  │
│                                                                      │
│  INTERPRETED (Slow):                                                │
│  For each row of data:                                              │
│    1. Walk down the AST tree                                        │
│    2. At each node, check type (if/else/switch)                     │
│    3. Dispatch to the correct evaluation function                   │
│    4. Call virtual functions at each node                            │
│    → LOTS of branches and virtual function calls                    │
│    → Costly for billions of rows!                                   │
│                                                                      │
│  CODE-GENERATED (Fast):                                             │
│  The entire expression tree is compiled into a                      │
│  single, tight loop of native bytecode                              │
│    → No tree walking, no virtual dispatch                           │
│    → Runs at near-native speed                                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**How It Works — Quasiquotes:**

Catalyst leverages Scala's **quasiquotes** feature for code generation. Quasiquotes allow the **programmatic construction of Abstract Syntax Trees (ASTs)** in the Scala language, which can then be fed to the Scala compiler at runtime to generate bytecode.

```scala
// Code generation function — converts expression tree to Scala AST
def compile(node: Node): AST = node match {
  case Literal(value)     => q"$value"                           // Constant value
  case Attribute(name)    => q"row.get($name)"                   // Column access
  case Add(left, right)   => q"${compile(left)} + ${compile(right)}"  // Addition
}

// Example:
//   Expression: Add(Literal(1), Attribute("x"))
//   Generated code: 1 + row.get("x")
//
// This is then compiled to JVM bytecode and runs at native speed!
```

**Key Properties of Quasiquotes:**

1. **Type-checked at compile time**: Only appropriate ASTs or literals can be substituted, making them much safer than string concatenation
2. **Highly composable**: The code generation rule for each node does not need to know how the trees returned by its children were built
3. **Further optimized by the Scala compiler**: The generated code gets additional expression-level optimizations that Catalyst might have missed
4. **Practical**: Even new contributors to Spark SQL could quickly add rules for new expression types

**Performance Impact:**

The paper demonstrates that code-generated evaluation achieves performance very close to hand-written code, and is **dramatically faster** than interpreted evaluation:

```
┌──────────────────────────────────────────────────────────────────────┐
│  PERFORMANCE: Evaluating x + x + x (1 billion times)                │
│                                                                      │
│  Interpreted:  ████████████████████████████████████████  ~38 sec    │
│  Hand-written: ████                                      ~4 sec     │
│  Generated:    ████                                      ~4 sec     │
│                                                                      │
│  Code generation is ~10x faster than interpretation!                │
│  And nearly identical to hand-tuned code!                           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Total: ~700 lines of code for the code generator.**

---

### 8.4 DataFrame DSL — How Expressions Are Captured

The DataFrame Domain Specific Language (DSL) is what makes all of this optimization possible. When you write DataFrame operations, you are NOT writing opaque functions — you are building an **Abstract Syntax Tree (AST)** that Catalyst can inspect and optimize.

**Relational Operators:**
- **Projection**: `select()`
- **Filter**: `where()`
- **Join**: `join()`
- **Aggregation**: `groupBy()`

All of these operators take **expression objects**, not arbitrary user functions. This is the key difference from the RDD API.

```python
# DataFrame API — expressions are captured as AST nodes
employees.join(dept, employees("deptId") == dept("id")) \
         .where(employees("gender") == "female") \
         .groupBy(dept("id"), dept("name")) \
         .agg(count("name"))

# employees("deptId") is NOT a Python function call —
# it creates an Expression object that Catalyst can analyze
```

**Schema evaluation is eager** (catches type errors immediately), while **execution is lazy** (waits for an action). This gives users the best of both worlds: immediate feedback on errors, but optimized execution when the computation runs.

---

### 8.5 Extension Points

Catalyst's composable rule-based design makes it inherently extensible. Two key public extension points are:

#### 8.5.1 Data Sources API

Developers can define new data sources by implementing one of several interfaces with varying degrees of optimization:

```
┌──────────────────────────────────────────────────────────────────────┐
│                   DATA SOURCE INTERFACES                             │
│                                                                      │
│  1. TableScan (simplest)                                            │
│     → Return ALL rows as RDD of Row objects                         │
│     → No pushdown, read everything                                  │
│                                                                      │
│  2. PrunedScan                                                      │
│     → Takes array of desired column names                           │
│     → Only returns those columns (column pruning)                   │
│                                                                      │
│  3. PrunedFilteredScan                                              │
│     → Takes desired columns AND filter predicates                   │
│     → Enables both column pruning AND predicate pushdown            │
│     → Filters are "advisory" — source can return false positives    │
│                                                                      │
│  4. CatalystScan (most advanced)                                    │
│     → Given complete Catalyst expression trees                      │
│     → Maximum optimization opportunity                              │
│                                                                      │
│  All data sources also expose NETWORK LOCALITY information          │
│  (which machines each partition is most efficient to read from)     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Built-in data sources implemented using this API:**
- **CSV files**: Scans the whole file, user can specify schema
- **Avro**: Self-describing binary format for nested data
- **Parquet**: Columnar file format, supports column pruning and filter pushdown
- **JDBC**: Scans ranges of a table from an RDBMS in parallel, pushes filters into the RDBMS

**Example — Query Federation with JDBC:**

```sql
-- Register a MySQL table
CREATE TEMPORARY TABLE users
USING jdbc OPTIONS (driver "mysql" url "jdbc:mysql://userDB/users")

-- Register a JSON data source
CREATE TEMPORARY TABLE logs
USING json OPTIONS (path "logs.json")

-- Query across both sources!
SELECT users.id, users.name, logs.message
FROM users JOIN logs WHERE users.id = logs.userId
AND users.registrationDate > "2015-01-01"
```

Under the hood, the JDBC data source uses `PrunedFilteredScan`, which pushes the filter down to MySQL. MySQL only returns matching rows:

```sql
-- What MySQL actually executes (pushed down):
SELECT users.id, users.name FROM users
WHERE users.registrationDate > "2015-01-01"
```

This **dramatically reduces data transfer** from the remote database.

#### 8.5.2 User-Defined Types (UDTs)

Users can register custom types that map to Catalyst's built-in types. For example, to register 2D points:

```scala
class PointUDT extends UserDefinedType[Point] {
  def dataType = StructType(Seq(
    StructField("x", DoubleType),
    StructField("y", DoubleType)
  ))
  def serialize(p: Point) = Row(p.x, p.y)
  def deserialize(r: Row) = Point(r.getDouble(0), r.getDouble(1))
}
```

Once registered, Spark SQL will:
- Recognize `Point` objects in DataFrames
- Store them in **columnar format** when caching (compressing x and y as separate columns)
- Make them **writable to all data sources** (which see them as pairs of `DOUBLE`s)
- Allow **UDFs to operate directly** on the custom type

This capability is used extensively in Spark's **MLlib** machine learning library, where vector types (both sparse and dense) are registered as UDTs.

---

### 8.6 Advanced Features Built on Catalyst

#### 8.6.1 Schema Inference for Semi-Structured Data (JSON)

Spark SQL includes a **schema inference algorithm** for JSON data that works in a **single pass** over the data:

```
┌──────────────────────────────────────────────────────────────────────┐
│                  JSON SCHEMA INFERENCE                                │
│                                                                      │
│  Input JSON records:                                                │
│  {"text": "Tweet about #Spark", "tags": ["#Spark"],                 │
│   "loc": {"lat": 45.1, "long": 90}}                                │
│  {"text": "Another tweet", "tags": [],                              │
│   "loc": {"lat": 39, "long": 88.5}}                                │
│  {"text": "No location", "tags": ["#tweet", "#location"]}          │
│                                                                      │
│  Inferred Schema:                                                   │
│  text   STRING NOT NULL                                             │
│  tags   ARRAY<STRING NOT NULL> NOT NULL                             │
│  loc    STRUCT<lat FLOAT NOT NULL, long FLOAT NOT NULL>             │
│                                                                      │
│  Algorithm:                                                         │
│  • For each field (by path from root), find MOST SPECIFIC type     │
│    that matches all observed instances                              │
│  • INT → LONG → DECIMAL → FLOAT → STRING (generalization chain)   │
│  • Uses a single REDUCE operation over the data                    │
│  • Merges schemas from individual records with an associative      │
│    "most specific supertype" function                               │
│  • Single-pass AND communication-efficient (high local reduction)  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

#### 8.6.2 Integration with MLlib

MLlib uses DataFrames as the standard data representation in its **pipeline API**:
- A **pipeline** is a graph of transformations: feature extraction → normalization → dimensionality reduction → model training
- Each pipeline stage takes and produces DataFrames
- The **vector UDT** stores both sparse and dense vectors using four primitive fields: a boolean (type), a size, an array of indices, and an array of double values
- Using DataFrames made it much easier to **expose all algorithms in all languages** (Scala, Java, Python, R)

#### 8.6.3 Query Federation

Spark SQL's data sources API enables **query federation** — querying across multiple heterogeneous data sources (JDBC databases, JSON files, Parquet, etc.) in a single program, with predicates pushed down to each source to minimize data transfer.

---

### 8.7 User-Defined Functions (UDFs)

Spark SQL supports **inline UDF registration** — you can register Scala, Java, or Python functions as UDFs and use them in SQL queries. Unlike traditional databases that require separate packaging and registration, Spark SQL UDFs can use the **full Spark API** internally.

```python
# Register a UDF in Python
from pyspark.sql.functions import udf
from pyspark.sql.types import FloatType

@udf(returnType=FloatType())
def predict(age, weight):
    return model.predict([age, weight])

# Use in SQL
spark.udf.register("predict", predict)
spark.sql("SELECT predict(age, weight) FROM users")
```

UDFs can also be accessed via **JDBC/ODBC** by business intelligence tools.

---

## 9. Query Planning: Cost-Based Optimizer

The **Cost-Based Optimizer (CBO)** in Catalyst estimates the cost of different physical plans and selects the cheapest one.

```
┌──────────────────────────────────────────────────────────────────────┐
│               COST-BASED OPTIMIZATION                                │
│                                                                      │
│  1. Represent queries as TREES                                      │
│  2. Apply RULES to manipulate them                                  │
│  3. Generate DIFFERENT PLANS based on manipulation                  │
│  4. ESTIMATE the execution cost of each plan                        │
│  5. SELECT the CHEAPEST plan for actual execution                   │
│                                                                      │
│  Cost estimation sources:                                           │
│  • In-memory cache → exact sizes known                              │
│  • External file sizing → file metadata (Parquet, etc.)             │
│  • Result of a subquery with LIMIT → estimated from subquery       │
│  • Table statistics → row count, column cardinality                │
│                                                                      │
│  Primary use: JOIN ALGORITHM SELECTION                              │
│  • Small table detected → Broadcast Hash Join                      │
│  • Large tables → Sort Merge Join or Shuffled Hash Join            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 10. Performance Evaluation (from SIGMOD 2015 Paper)

### 10.1 SQL Performance

Compared against **Shark** and **Impala** on the AMPLab big data benchmark (web analytics workload with scans, aggregation, joins, and UDFs):

- **Spark SQL is substantially faster than Shark** in all queries (due to code generation reducing CPU overhead)
- **Spark SQL is generally competitive with Impala** (which uses C++ and LLVM) — code generation closes the gap
- Setup: 6 EC2 i2.xlarge machines, 110 GB dataset in Parquet format

### 10.2 DataFrame API vs. Native Spark Code

```
┌──────────────────────────────────────────────────────────────────────┐
│        DATAFRAME API vs. NATIVE SPARK CODE                           │
│        (Distributed aggregation: avg of b for each value of a)      │
│        (1 billion integer pairs, 100,000 distinct values of a)      │
│                                                                      │
│  Native Python API:   ████████████████████████████████  ~200 sec    │
│  Native Scala API:    ████████████                      ~100 sec    │
│  DataFrame API:       ██████                            ~50 sec     │
│                                                                      │
│  DataFrame is 12x faster than Python, 2x faster than Scala!        │
│                                                                      │
│  Why?                                                               │
│  • Python: Only logical plan in Python; physical execution          │
│    compiled to native JVM bytecode                                  │
│  • Scala: DataFrame avoids expensive key-value pair allocation      │
│    that occurs in hand-written Scala code (via code generation)     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 10.3 Pipeline Performance

For a two-stage pipeline (relational filter + word count):
- **Integrated DataFrame pipeline** is **2x faster** than separate SQL + Spark jobs
- Avoids materializing intermediate results to HDFS
- SparkSQL pipelines the `map` for word count with the relational operators for filtering

---

## 11. Clash of the Titans: MapReduce vs. Spark (VLDB 2015)

*Reference: Juwei Shi et al., "Clash of the Titans: MapReduce vs. Spark for Large Scale Data Analytics," VLDB 2015.*

### 11.1 Setup

- Hadoop 2.4.0 vs. Spark 1.3.0
- 4 servers @ 32 CPU cores, 2.9 GHz
- 9 disk drives at 7.2k RPM with 1 TB each (aggregate bandwidth: ~125 GB/s reads, ~45 GB/s writes)
- 190 GB RAM per server
- 1 Gbps Ethernet switch
- 32 containers per node for MapReduce on YARN
- 8 Spark workers per node with 4 threads each

### 11.2 Key Findings — Where Spark Wins

**Spark is ~2.5x, 5x, and 5x faster than MapReduce** for Word Count, k-Means, and PageRank respectively:

| Factor | Explanation |
|--------|-------------|
| **Hash-based aggregation** | Spark's hash-based combine (map-side reduction) is ~40% more efficient than MapReduce's sort-based combine |
| **RDD caching** | Reduced CPU and disk overheads for iterative algorithms (PageRank, k-Means) — up to 90% improvement |
| **Data pipelining** | Avoids materialization of intermediate results between stages |
| **Task loading** | Spark's context switch is 10x faster than MapReduce task startup |

### 11.3 Key Findings — Where MapReduce Wins

**MapReduce is 2x faster than Spark for Sort workload:**

| Factor | Explanation |
|--------|-------------|
| **Shuffle efficiency** | MapReduce overlaps Shuffle with Map phase, hiding network overhead |
| **Open files overhead** | Map stage in Spark is slower with more Reducers due to more open files |
| **GC overhead** | Increasing JVM heap size for Spark causes garbage collection overhead |

### 11.4 Common Observations

- For one-pass jobs: **Map is CPU-bound**, **Reduce is network-bound** (disk I/O is NOT the bottleneck — network is), so spills often do not have a significant penalty
- **Input parsing** is often an overhead — RDD caching helps, OS/HDFS caching does not
- **GC overhead** becomes a bottleneck if heap size per task drops to 64 MB with 128 MB split
- **Disk caching** is a bottleneck for RDD if CPU and disk I/O capacities are unbalanced

---

## 12. Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                        KEY TAKEAWAYS                                 │
│                                                                      │
│  1. DataFrames provide a declarative, SQL-like API that lets        │
│     Spark optimize execution automatically                          │
│                                                                      │
│  2. Catalyst Optimizer uses TREES + RULES for 4-phase query         │
│     planning: Analysis → Logical Optimization → Physical            │
│     Planning → Code Generation                                      │
│                                                                      │
│  3. Code generation brings near-native performance by compiling     │
│     expressions to JVM bytecode (10x faster than interpretation)    │
│                                                                      │
│  4. Catalyst is EXTENSIBLE: new data sources, UDTs, UDFs, and      │
│     optimization rules can be added with minimal code               │
│                                                                      │
│  5. DataFrames are 2-12x faster than hand-written Spark code        │
│     thanks to Catalyst optimizations                                │
│                                                                      │
│  6. Spark excels at iterative workloads (caching, pipelining)       │
│     while MapReduce is better at shuffle-heavy workloads (Sort)     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

*References:*
- *Lecture slides: L2.4 Big Data Processing with Apache Spark, DS256, IISc Bangalore*
- *Armbrust, M., et al. "Spark SQL: Relational Data Processing in Spark." SIGMOD 2015.*
- *Shi, J., et al. "Clash of the Titans: MapReduce vs. Spark for Large Scale Data Analytics." VLDB 2015.*
