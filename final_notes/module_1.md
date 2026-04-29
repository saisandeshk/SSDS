# Lecture 1.3: Introduction to Distributed File Systems

## DS256 - Scalable Systems for Data Science
### Scale-out Data Storage using GFS/HDFS

---

## 1. Why Do We Need a File System?

A file system serves four fundamental purposes:

### 1.1 Persistence of Data (Reliability)
- Data survives beyond the lifetime of the process that created it
- Data remains intact even after system restarts or power failures
- Ensures long-term storage of information

### 1.2 Logical Organization and Access
- Provides a hierarchical structure (directories and files)
- Human-readable naming conventions
- Easy navigation and management of data

### 1.3 Process Access (Performance)
- Allows programs to read from and write to storage
- Provides efficient I/O operations
- Manages buffering and caching for better performance

### 1.4 Sharing Across Users (Access Control)
- Multiple users can access the same data
- Permission systems (read, write, execute)
- Concurrent access management

---

## 2. What is a File System?

A file system is an **abstraction layer** that hides the complexity of underlying storage media from users and applications.

### Key Abstractions:
- **File**: A logical unit of information with a name
- **Directory**: A container that organizes files hierarchically

### File Attributes:
- **Name**: Human-readable identifier
- **Type**: Extension or format (e.g., .txt, .pdf)
- **Size**: Amount of data stored
- **Access Control**: Permissions (rwx for owner/group/others)
- **Time**: Creation, modification, access timestamps

### Logical Operations:
- **Create**: Make a new file
- **Write**: Add or modify data
- **Read**: Retrieve data
- **Delete**: Remove the file

The beauty of a file system is that it presents the **same interface** regardless of whether the underlying storage is an HDD, SSD, tape drive, or network storage.

---

## 3. File System Modules

A file system is composed of several layered modules, each handling a specific responsibility:

```
┌─────────────────────────────┐
│     Directory Module        │  ← Maps file names to file IDs
├─────────────────────────────┤
│    Access Control Module    │  ← Checks permissions (read/write/execute)
├─────────────────────────────┤
│       Files Module          │  ← Manages file metadata (size, timestamps)
├─────────────────────────────┤
│   Block Allocation Module   │  ← Decides which disk blocks to use
├─────────────────────────────┤
│     Disk Access Module      │  ← Performs actual I/O with hardware
├─────────────────────────────┤
│       Disk Blocks           │  ← Physical storage on the disk
└─────────────────────────────┘
```

### 3.1 Directory Module
- Translates human-readable file names to internal file identifiers
- Maintains the hierarchical structure (parent-child relationships)
- Example: `/home/user/document.txt` → File ID 12345

### 3.2 Access Control Module
- Verifies if the requesting user/process has permission
- Enforces read, write, and execute permissions
- Prevents unauthorized access

### 3.3 Files Module
- Manages file metadata (not the actual data)
- Tracks file size, creation time, modification time
- Maintains pointers to where file data is stored

### 3.4 Block Allocation Module
- Decides which physical blocks on disk will store file data
- Handles allocation when files grow, deallocation when deleted
- Manages free space tracking

### 3.5 Disk Access Module
- Interfaces directly with the storage hardware
- Issues read/write commands to the disk controller
- Handles buffering and scheduling of I/O operations

---

## 4. Unix iNodes

An **iNode** (Index Node) is a data structure in Unix-based file systems that stores metadata about a file and maps the logical file to physical disk blocks.

### What an iNode Contains:
- File type (regular file, directory, symbolic link)
- Permissions (rwx for owner, group, others)
- Owner UID and Group GID
- File size in bytes
- Timestamps (access, modification, change)
- Link count (number of hard links)
- **Pointers to data blocks**

### How iNodes Map to Disk Blocks:

```
┌──────────────────────────────────────────┐
│              iNode Structure             │
├──────────────────────────────────────────┤
│  Metadata (permissions, size, times)     │
├──────────────────────────────────────────┤
│  Direct Block Pointers (12 pointers)     │──→ [Block 1][Block 2]...[Block 12]
├──────────────────────────────────────────┤
│  Single Indirect Pointer                 │──→ [Pointer Block] → [Data Blocks]
├──────────────────────────────────────────┤
│  Double Indirect Pointer                 │──→ [Ptr Block] → [Ptr Block] → [Data]
├──────────────────────────────────────────┤
│  Triple Indirect Pointer                 │──→ [Ptr] → [Ptr] → [Ptr] → [Data]
└──────────────────────────────────────────┘
```

### Why This Multi-Level Structure?
- **Small files** (most common): Use only direct pointers → Fast access
- **Medium files**: Use single indirect → Still reasonably fast
- **Large files**: Use double/triple indirect → Can store huge files

### Key Insight:
The iNode is the bridge between the **logical file system** (what users see) and the **physical storage** (actual disk blocks). When you access `/home/user/file.txt`:
1. Directory lookup gives you the iNode number
2. iNode gives you the block addresses
3. Disk access module reads those blocks

---

## 5. What is a Distributed File System (DFS)?

A **Distributed File System** is a file system where the storage location and the clients accessing the data can be on **different machines** connected over a network.

### Types of Distributed File Systems:

#### 5.1 Client-Server File System
- **Architecture**: Many clients, one server holding all data
- **Example**: NFS (Network File System)
- **Use case**: Small organizations, shared network drives

```
[Client 1] ──┐
[Client 2] ──┼──→ [Single Server with Data]
[Client 3] ──┘
```

#### 5.2 Cluster File System
- **Architecture**: Many clients, many servers in a single cluster
- **Example**: GFS (Google File System), HDFS (Hadoop Distributed File System)
- **Use case**: Big data processing, large-scale storage

```
[Client 1] ──┐      ┌─→ [Server 1]
[Client 2] ──┼──────┼─→ [Server 2]
[Client 3] ──┘      └─→ [Server N]
```

#### 5.3 Peer-to-Peer File System
- **Architecture**: Every node is both client and server
- **Example**: Chord, BitTorrent
- **Use case**: Decentralized storage, file sharing

```
[Node 1] ←──→ [Node 2]
   ↑↓           ↑↓
[Node 3] ←──→ [Node 4]
```

---

## 6. Why Do We Need a Distributed File System?

### 6.1 Remote Access
- Access data from anywhere on the network
- No need to physically be at the storage location

### 6.2 Performance
- **Bandwidth**: Aggregate bandwidth from multiple servers
- **Wide-area locality**: Place data closer to where it's needed
- **Parallel I/O**: Read from multiple servers simultaneously

### 6.3 Reliability
- Replication across multiple machines
- No single point of failure (for data)
- Automatic recovery from failures

### 6.4 Capacity
- Scale storage by adding more machines
- Petabytes of storage possible
- Beyond what any single machine can hold

### 6.5 Access Restrictions
- Physical restrictions (data center access)
- Network-level access control
- Geographical distribution

---

## 7. Amdahl's Laws (Rules of Thumb for System Design)

Gene Amdahl formulated several rules of thumb for designing balanced computer systems. These help us understand the relationships between different system components.

### 7.1 Amdahl's Law of Parallelism
(You already know this from L1.2)

If a computation has a serial part `s` and a parallel part `(1-s)`:
- **Maximum Speedup** = 1/s
- The serial portion limits how much parallelism helps

### 7.2 Amdahl's Balanced System Law

> **"A system needs 1 bit of disk I/O per second for every instruction per second"**

This means:
- If your CPU can execute 1 billion instructions/second (1 GIPS)
- You need 1 Gbit/s = 125 MB/s of disk I/O bandwidth
- Otherwise, the CPU will be waiting for data

### 7.3 Amdahl's Memory Law

> **"The RAM capacity (in MB) should equal the CPU speed (in MIPS)"**

The ratio α (alpha) = RAM MB / CPU MIPS should be approximately 1.

- If CPU = 1000 MIPS, you need ~1000 MB = 1 GB RAM
- Too little RAM → Constant disk swapping
- This ratio has been increasing to 4+ as RAM gets cheaper

### 7.4 Amdahl's I/O Law

> **"Programs do 1 I/O operation per 50,000 instructions"**

This helps estimate I/O requirements:
- 1 billion instructions → 20,000 I/O operations
- Helps size I/O subsystems appropriately

---

## 8. System Component Relationships: A Complete Example

Let's understand how different system metrics relate to each other with a concrete example.

### The Question: 
If you have certain performance requirements, how do you size your system?

### Key Metrics and Their Relationships:

| Component | Metric | Typical Modern Value |
|-----------|--------|---------------------|
| CPU | Instructions/second | 32 × 10⁹ (16 cores × 2 GHz) |
| Disk Controller | I/O bandwidth | 22.5 Gbit/s (SAS-4) |
| RAM | Capacity | 32 GB |
| Single HDD | Transfer rate | ~100-150 MB/s |
| Single SSD | Transfer rate | ~500 MB/s |

### Example: Designing a Balanced System

**Given Requirements:**
- Need to process 1 TB of data
- Must complete in 1 hour
- Want a balanced system

**Step 1: Calculate Required I/O Bandwidth**
```
Data size = 1 TB = 1,000,000 MB
Time = 1 hour = 3,600 seconds
Required bandwidth = 1,000,000 / 3,600 ≈ 278 MB/s
```

**Step 2: How Many Disks for Bandwidth?**
```
If using HDDs (100 MB/s each):
    Disks needed = 278 / 100 ≈ 3 HDDs

If using SSDs (500 MB/s each):
    Disks needed = 278 / 500 ≈ 1 SSD
```

**Step 3: How Many Disks for Capacity?**
```
If each disk is 1 TB:
    Disks needed = 1 TB / 1 TB = 1 disk

If each disk is 500 GB:
    Disks needed = 1 TB / 500 GB = 2 disks
```

**Step 4: Choose the Larger Number**
```
For bandwidth: 3 HDDs
For capacity: 1 HDD
→ Need at least 3 HDDs (bandwidth is the bottleneck)
```

**Step 5: Match CPU and RAM (Amdahl's Laws)**
```
If disk I/O = 278 MB/s ≈ 2.2 Gbit/s
CPU should handle ≈ 2.2 billion instructions/sec
RAM should be ≈ 2-8 GB (with α = 1 to 4)
```

### The Key Insight:

**You're only as fast as your slowest component!**

```
         ┌─────────────┐
         │    CPU      │ ← Can process 32 billion ops/sec
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │    RAM      │ ← Can supply 25 GB/s
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │ Disk I/O    │ ← Can only do 278 MB/s  ← BOTTLENECK!
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │   Network   │ ← Depends on configuration
         └─────────────┘
```

In Big Data systems, **disk I/O is almost always the bottleneck**, which is why:
- We use many disks in parallel (HDFS/GFS approach)
- We try to move computation to data (not data to computation)
- We use replication which also helps read parallelism

---

## 9. DFS Characteristics

When designing or evaluating a Distributed File System, consider these key characteristics:

### 9.1 Transparency
- **Access Transparency**: Same API regardless of where data is stored
- **Location Transparency**: Don't need to know physical location
- **Performance Transparency**: Consistent performance expectations
- **Scaling Transparency**: System can grow without API changes

### 9.2 Concurrency & Consistency
- Multiple clients can access same file simultaneously
- Need to define what happens with concurrent writes
- Trade-off between strong consistency and performance

### 9.3 Replication
- Multiple copies of data on different machines
- Improves fault tolerance
- Can improve read performance (read from any replica)
- Challenge: Keeping replicas consistent

### 9.4 Fault Tolerance
- System continues operating despite failures
- Automatic detection of failed components
- Automatic recovery and data repair
- No single point of failure (ideally)

### 9.5 Efficiency
- High throughput for large transfers
- Low latency for small operations
- Efficient use of network bandwidth
- Minimal overhead from distribution

### 9.6 Heterogeneity
- Work with different hardware (CPUs, disks, networks)
- Support different operating systems
- Handle varying network speeds

### 9.7 Security
- Authentication (who is accessing?)
- Authorization (what are they allowed to do?)
- Encryption (protect data in transit and at rest)
- Audit logging

---

## 10. Storage Systems Comparison

Different storage systems make different trade-offs. Here's a comprehensive comparison:

| Storage System | Sharing | Persistence | Distributed Cache/Replicas | Consistency | Example |
|----------------|---------|-------------|---------------------------|-------------|---------|
| **Main Memory** | Within process | No (volatile) | No | N/A (single location) | RAM, CPU cache |
| **File System** | Within machine | Yes | No | Strong (single machine) | ext4, NTFS |
| **Distributed File System** | Across network | Yes | Yes (replicas) | Varies (often relaxed) | GFS, HDFS, NFS |
| **Web** | Global (read-mostly) | Yes | Yes (CDN caches) | Weak (eventual) | HTTP/HTTPS |
| **Distributed Shared Memory** | Across cluster | No | Yes (cached copies) | Varies | Treadmarks |
| **Remote Objects** | Across network | Varies | Optional | Strong or eventual | Java RMI, CORBA |
| **Persistent Object Store** | Across network | Yes | Optional | Strong | Object databases |
| **Peer-to-Peer** | Across internet | Yes | Yes (distributed) | Eventual | BitTorrent, IPFS |

### Detailed Breakdown:

#### Main Memory
- **Sharing**: Only within a single process
- **Persistence**: Lost when power off
- **Use case**: Running programs, temporary data

#### Traditional File System
- **Sharing**: Multiple processes on same machine
- **Persistence**: Survives restarts
- **Use case**: Local storage on laptops, servers

#### Distributed File System
- **Sharing**: Any machine on the network
- **Persistence**: Yes, with replication for reliability
- **Replicas**: Multiple copies for fault tolerance
- **Use case**: Big data storage, shared enterprise storage

#### Web
- **Sharing**: Global, primarily read access
- **Persistence**: Yes (at origin servers)
- **Caching**: Extensive (browsers, CDNs, proxies)
- **Consistency**: Weak - you might see stale data
- **Use case**: Websites, APIs

#### Distributed Shared Memory (DSM)
- **Sharing**: Makes distributed memory look like shared memory
- **Persistence**: No - it's still RAM
- **Consistency**: Must be carefully managed
- **Use case**: Parallel computing applications

#### Peer-to-Peer
- **Sharing**: No central authority
- **Persistence**: As long as some peer has the data
- **Replicas**: Data spread across many peers
- **Consistency**: Eventual (no central coordinator)
- **Use case**: File sharing, blockchain

---

## 11. Big Data Sizes in the Era of Deep Learning

Modern machine learning success is largely due to **massive datasets**. Neural networks aren't new (50+ years old), but we only recently have enough data and compute to train them effectively.

### Historical Milestones:

#### Netflix Prize (2006)
- **Task**: Movie recommendation system
- **Data**: 100 million ratings
- **Scale**: 17,000 movies, 500,000 users
- **Impact**: Sparked interest in recommendation systems

#### ImageNet (2009)
- **Task**: Image classification
- **Data**: 14 million images
- **Scale**: 20,000+ categories
- **Impact**: Enabled deep learning breakthrough (AlexNet, 2012)

#### IBM Watson Beats Jeopardy (2011)
- **Task**: Question answering
- **Compute**: 5,500 independent experiments
- **Scale**: 2,000 CPU hours each, generating 10 GB error-analysis data
- **Architecture**: Massively parallel processing

#### Google Brain Recognizes Cats (2012)
- **Task**: Unsupervised feature learning
- **Data**: 10 million 200×200 pixel images
- **Scale**: 20,000 classes
- **Compute**: 1,000 machines with 16,000 cores
- **Impact**: Showed deep learning can discover concepts without labels

#### Facebook DeepFace (2014)
- **Task**: Face recognition
- **Data**: 4.4 million labeled faces
- **Scale**: 4,000 people, 1,000 samples each
- **Result**: Near human-level face verification

#### GPT-3 (2020)
- **Task**: Text generation
- **Data**: 
  - 400 billion tokens from Common Crawl
  - 19 billion tokens from WebText2
  - 67 billion tokens from books
  - 3 billion tokens from Wikipedia
- **Model**: 175 billion parameters
- **Impact**: Showed emergent abilities in large language models

#### ChatGPT (2022)
- **Base**: GPT-3 with 175 billion parameters
- **Fine-tuning**: Reinforcement Learning from Human Feedback (RLHF)
- **Human Data**: 33,000 human prompts for instruction tuning
- **Impact**: Made AI accessible to general public

### The Trend:
```
Dataset Size Over Time (Log Scale)

10 PB   │                                    ●  LLMs (2023)
        │                               ●  GPT-3
1 PB    │
        │
100 TB  │                          ●  Google Brain
        │
10 TB   │                     ●  ImageNet
        │
1 TB    │                ●  Netflix
        │           
100 GB  │      ●  Earlier ML
        │
        └────────────────────────────────────────────→
              2005    2010    2015    2020    2025
```

---

## 12. Training the ChatGPT Model

### The Infrastructure Challenge

Training models like ChatGPT requires massive distributed systems. Here's how it works:

### Microsoft Azure "Singularity" - AI Cloud Infrastructure

ChatGPT was trained on Microsoft Azure's specialized AI infrastructure called **Singularity**.

#### Key Features:

##### 1. Incremental Memory Checkpointing
```
┌─────────────┐     ┌─────────────┐
│  Worker 1   │────▶│   Storage   │
│  (GPU/CPU)  │     │  Checkpoint │
└─────────────┘     └─────────────┘
       │
       ▼
   Training continues while checkpoint saves
```

- Periodically saves the state of all workers (CPU and GPU memory)
- Used for **recovery** if a machine fails
- Used for **elastic resizing** (adding/removing machines)

##### 2. De-duplication for Efficiency
- Uses checksums to identify duplicate memory content
- Train and loader processes across workers often have similar data
- A worker's memory over time has similar content
- De-duplication reduces checkpoint size significantly

##### 3. Transparent Scaling
> "To scale up or scale down a job, we simply change the number of devices the workers are mapped to. This is completely transparent to the user..."

This means:
- Can add more GPUs when available
- Can reduce GPUs if higher-priority job needs them
- Training continues without restarting

##### 4. Storage Bottleneck
Even with all optimizations:
- **50% of checkpoint latency** goes to Azure Blob Storage
- Storage I/O is still the bottleneck, even for AI workloads

### The Investment Scale

> "Microsoft invested $1 billion in OpenAI in 2019, and in return OpenAI has built its AI models on Microsoft's Azure AI supercomputing technologies..."

This investment paid for:
- Thousands of high-end GPUs
- Specialized networking (high bandwidth, low latency)
- Massive storage infrastructure
- Custom software stack for distributed training

### Why This Matters for This Course

The ChatGPT training example shows why we need:
1. **Distributed File Systems** - Store petabytes of training data
2. **Fault Tolerance** - Training runs for weeks; machines will fail
3. **High Bandwidth** - Move data fast enough to keep GPUs busy
4. **Scalability** - Use 1000s of machines efficiently

---

## Summary

### Key Takeaways from Lecture 1.3:

1. **File systems abstract storage** - Same interface regardless of underlying media

2. **DFS extends this across networks** - Enabling scale, reliability, and remote access

3. **Amdahl's Laws guide system design** - Balance CPU, memory, and I/O

4. **The bottleneck is usually I/O** - This is why we parallelize storage

5. **Different storage systems for different needs** - Trade-offs between consistency, performance, and availability

6. **Big Data drives AI progress** - Modern ML needs massive datasets that only DFS can handle

7. **Training large models requires distributed systems** - Everything we learn in this course applies to real AI infrastructure

---

## References

1. Coulouris, Dollimore, Kindberg and Blair, "Distributed Systems: Concepts and Design", 5th Edition, Chapter 12
2. Silberschatz, Galvin, Gagne, "Operating System Concepts", 9th Edition, Chapter 12
3. Jim Gray, Prashant Shenoy, "Rules of Thumb in Data Engineering", ICDE 2000
4. Bell, Gray and Szalay, "Petascale Computational Systems", IEEE Computer, 2006
5. OpenAI, "Language Models are Few-Shot Learners" (GPT-3 paper)
6. Microsoft Research, "Singularity: Planet-Scale, Preemptive and Elastic Scheduling of AI Workloads"


# Lecture 1.4: The Google File System (GFS)

## DS256 — Scalable Systems for Data Science
### Module 1: Introduction to Big Data & Distributed Storage

---

## 1. Motivation: Why Was GFS Built?

### 1.1 The Google Search Problem (Circa 2000)

Google's core mission was to **crawl, index, and search the entire World Wide Web**. This required a pipeline of massive data operations:

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Web Crawl  │────▶│  Pre-processing  │────▶│  Search & Rank   │
│  (millions   │     │  (Index, Graph,  │     │  (Query-time     │
│   of pages)  │     │   PageRank)      │     │   lookups)       │
└──────────────┘     └──────────────────┘     └──────────────────┘
```

#### Pre-processing Steps:
1. **Web Crawl**: Crawlers fetch millions of web pages (HTML text) from the internet
2. **Build Inverted Index**: Map each keyword to the set of URLs (web pages) containing it
3. **Build Web Graph**: Extract hyperlinks between pages to form a directed graph
4. **Compute PageRank**: Run the PageRank algorithm over the web graph to score and rank pages by importance

#### At Search Time:
- Look up the query keywords in the inverted index to find matching URLs
- Use PageRank scores to rank the results
- Return the top results to the user

### 1.2 Inverted Index — How Search Works

An inverted index reverses the mapping from "document → words" to "word → documents":

**Step 1: URL to HTML Text Mapping (Forward Index)**
```
URL     Content
u1      "We the People of India, having solemnly…"
u2      "It was the best of times, it was the…"
u3      "Call me Ishmael. Some years ago…"
u4      "Here's my number, call me maybe…"
u5      "People call me the best…"
u6      "Number of people in India is…"
u7      "Best years of my life…"
```

**Step 2: Parse, Tokenize, Remove Stop Words**

Each document is broken into tokens (words). Common stop words (the, of, it, was, is, in, etc.) and contractions are removed. The remaining keywords form the document's keyword set.

**Step 3: Invert the Index**
```
Keyword     URLs containing it
──────────  ──────────────────
People      u1, u5, u6
India       u1, u6
Best        u2, u5, u7
Call        u3, u4, u5
Ishmael     u3
Some        u3
Years       u3, u7
Here        u4
Number      u4, u6
Life        u7
```

Now a search query like `"People India"` can quickly find `u1` and `u6` by intersecting the posting lists.

### 1.3 Web Graph and PageRank

**Web Graph Construction:**
- Extract all hyperlinks from each crawled page
- Build a directed graph: each page is a node, each hyperlink is a directed edge

**PageRank Algorithm:**
- Models a "random surfer" who randomly clicks links
- Pages that are linked to by many important pages get a higher rank
- Iteratively computed over the entire web graph until convergence

```
URL     PageRank
u1      0.02
u2      0.30
u3      0.08
u4      0.10
u5      0.20
u6      0.25
u7      0.05
```

### 1.4 Modern Relevance: LLM Training Data

The same web crawl infrastructure is now used for training Large Language Models:
- **Common Crawl** provides the raw text (WET files) from billions of web pages
- **LLaMA** (Meta) and **Falcon** (TII) both use Common Crawl as a primary training data source
- The pipeline involves extensive filtering, deduplication, and quality scoring before the text is used for training

---

## 2. Design Goals of GFS

GFS was designed with very specific workload assumptions that differ from traditional file systems:

### 2.1 Failure as the Norm
- The system is built from **hundreds or thousands of inexpensive commodity machines**
- Component failures (disks, memory, network, power) are **routine, not exceptional**
- The system must **constantly monitor itself**, detect failures, tolerate them, and **recover automatically**

### 2.2 Large Files, Not Small Files
- A **modest number of large files**: a few million files, each typically **100 MB or larger**
- Multi-GB files are the **common case**
- Small files are supported but **not optimized for**

### 2.3 Read-Heavy, Append-Mostly Workload
- **Large streaming reads**: each operation reads ≥1 MB; clients read contiguous regions of a file sequentially
- **Small random reads**: a few KB at arbitrary offsets; performance-conscious applications batch and sort these to advance sequentially
- **Writes are mostly large, sequential appends**: files are written once (append-only) and then read many times
- **Random writes** are supported but need not be efficient
- Files are **seldom modified after creation**

### 2.4 Concurrent Appends by Many Producers
- Hundreds of producer clients may **concurrently append** to the same file (e.g., a log file, a merged-results file)
- **Atomicity with minimal synchronization** is essential
- A consumer may be **reading the file while writers are still appending**

### 2.5 Bandwidth over Latency
- **High sustained throughput** (bulk data processing) is more important than **low latency** (interactive response)
- Applications are batch-oriented, not interactive

---

## 3. GFS Interface (API)

GFS provides a **familiar file system interface** but does **not implement the full POSIX standard**.

### 3.1 Standard Operations
Files are organized **hierarchically in directories** and identified by **pathnames**.

| Operation | Description |
|-----------|-------------|
| `create`  | Create a new file |
| `delete`  | Remove a file |
| `open`    | Open a file for reading/writing |
| `close`   | Close an open file |
| `read`    | Read data from a file |
| `write`   | Write data to a file at a specified offset |

### 3.2 New Operations Unique to GFS

| Operation | Description |
|-----------|-------------|
| **Snapshot** | Creates a copy of a file or directory tree almost instantaneously (copy-on-write) |
| **Record Append** | Appends a record atomically at least once, even with concurrent writers. GFS chooses the offset. |

**Why not full POSIX?**
- GFS is accessed by Google's own applications, not arbitrary Unix programs
- Relaxing the interface allows simpler design and better performance
- No need to hook into the Linux vnode layer

---

## 4. GFS Architecture

### 4.1 Components Overview

A GFS cluster consists of three types of entities:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          GFS Cluster                                │
│                                                                     │
│  ┌────────────────┐                                                 │
│  │   Master       │  (Single)  — Stores all metadata                │
│  │  (NameNode)    │  — Namespace, file→chunk mapping                │
│  │                │  — Chunk→chunkserver mapping                    │
│  │                │  — Lease management, GC, rebalancing            │
│  └───────┬────────┘                                                 │
│          │ Heartbeats, Instructions                                 │
│          │                                                          │
│  ┌───────▼────────┐  ┌────────────────┐       ┌────────────────┐   │
│  │ ChunkServer 1  │  │ ChunkServer 2  │  ...  │ ChunkServer N  │   │
│  │  (DataNode)    │  │  (DataNode)    │       │  (DataNode)    │   │
│  │ ┌────┐ ┌────┐  │  │ ┌────┐ ┌────┐ │       │ ┌────┐ ┌────┐  │   │
│  │ │Blk1│ │Blk3│  │  │ │Blk1│ │Blk2│ │       │ │Blk2│ │Blk3│  │   │
│  │ └────┘ └────┘  │  │ └────┘ └────┘ │       │ └────┘ └────┘  │   │
│  │  Linux FS      │  │  Linux FS     │       │  Linux FS      │   │
│  └────────────────┘  └───────────────┘       └────────────────┘   │
│                                                                     │
│  ┌────────────────┐                                                 │
│  │  GFS Clients   │  — Application-linked library                   │
│  │                │  — Metadata ops → Master                        │
│  │                │  — Data ops → ChunkServers directly              │
│  └────────────────┘                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 The Single Master

**Design choice:** A single master **vastly simplifies the design** and enables globally optimal decisions for chunk placement, replication, and load balancing.

**Critical design constraint:** The master must **never become a bottleneck**.

How this is achieved:
- Clients **never read or write file data through the master**
- Clients only contact the master for **metadata operations** (which chunkservers to talk to)
- Clients **cache** the metadata (chunk locations) and interact **directly with chunkservers** for data

#### Read Flow (Step by Step):

```
Client                          Master                    ChunkServer
  │                               │                            │
  │ 1. (filename, chunk_index)    │                            │
  │──────────────────────────────▶│                            │
  │                               │                            │
  │ 2. (chunk_handle, locations)  │                            │
  │◀──────────────────────────────│                            │
  │                               │                            │
  │        [Client caches this info]                           │
  │                               │                            │
  │ 3. (chunk_handle, byte_range) │                            │
  │────────────────────────────────────────────────────────────▶│
  │                               │                            │
  │ 4. chunk_data                 │                            │
  │◀───────────────────────────────────────────────────────────│
```

1. Client translates `(filename, byte_offset)` into `(filename, chunk_index)` using the fixed chunk size (64 MB). Sends request to master.
2. Master replies with the **chunk handle** (unique 64-bit ID) and the **locations** of all replicas. Client caches this using `(filename, chunk_index)` as the key.
3. Client sends a read request to the **closest** chunkserver replica, specifying `(chunk_handle, byte_range)`.
4. ChunkServer returns the requested data.

**Optimization:** Client typically requests **multiple chunks** in one master request; master may proactively return info for subsequent chunks.

### 4.3 ChunkServers (Data Nodes / Workers)

- Each chunkserver stores chunks as **ordinary Linux files** on its local disk
- Data is read/written by specifying `(chunk_handle, byte_offset)`
- ChunkServers are **unaware of GFS file semantics** — they just store opaque chunks
- Linux's buffer cache handles caching automatically — no additional caching layer needed

### 4.4 Clients

- GFS client code is a **library linked into applications**
- It implements the GFS API and handles communication with master and chunkservers
- **Clients do NOT cache file data** — why?
  - Most applications **stream through huge files** sequentially
  - Working sets are **too large to be cached** effectively
  - Eliminating client caches removes **cache coherence complexity**
- Clients **do cache metadata** (chunk locations) for a limited time

---

## 5. Chunk Size: 64 MB

### 5.1 Why So Large?

The chunk size of **64 MB** (128 MB in HDFS v3) is **vastly larger** than typical filesystem block sizes (~4 KB). This is a deliberate design choice.

Each chunk replica is stored as a **plain Linux file** on the chunkserver and is extended only as needed (**lazy space allocation** — no wasted space from internal fragmentation).

### 5.2 Advantages of Large Chunk Size

| Advantage | Explanation |
|-----------|-------------|
| **Fewer client-master interactions** | Reads/writes on the same chunk only need one initial metadata request. Critical for large sequential reads. |
| **Reduced network overhead** | Client can maintain a single **persistent TCP connection** to a chunkserver for many operations on the same chunk. |
| **Smaller metadata on Master** | Fewer chunks means fewer entries in the master's in-memory metadata tables. Allows all metadata to fit in RAM. |

### 5.3 Disadvantage: Hot Spots

- A small file consists of perhaps **only one chunk**
- If many clients access the same small file, the chunkserver(s) storing that chunk become **hot spots**
- **In practice**, this was rare because applications mostly read large multi-chunk files sequentially

**Real-world hot spot incident:** When GFS was first used by a batch-queue system, an executable (single chunk) was started on hundreds of machines simultaneously, overloading the chunkservers. **Fix:** Store such files with a **higher replication factor** and stagger start times.

---

## 6. Master Metadata

The master stores three types of metadata, **all kept in memory** for fast access:

### 6.1 Three Types of Metadata

| Metadata | Persistent? | How Maintained |
|----------|-------------|----------------|
| **File & chunk namespaces** | Yes (operation log) | Mutations logged to disk |
| **File → chunk mapping** | Yes (operation log) | Mutations logged to disk |
| **Chunk → chunkserver mapping** | **No** | Built on-demand at startup from chunkserver reports |

### 6.2 Why In-Memory?

- Master operations become **extremely fast**
- Enables efficient **background scanning** for:
  - **Garbage collection** of orphaned chunks
  - **Re-replication** when replicas are lost
  - **Chunk migration** for load balancing
- **Memory usage is compact**: ~64 bytes per chunk, ~64 bytes per file (filenames stored with prefix compression)
- For a cluster with millions of files, this is only tens of MB — easily fits in RAM

### 6.3 Why Is Chunk Location NOT Persisted?

The master does **not** store chunk-to-chunkserver mapping persistently. Instead, it asks each chunkserver at startup what chunks it has (via **Block Reports**).

**Rationale:**
- In a cluster of hundreds of servers, **machines constantly join, leave, fail, restart, change names**
- Keeping persistent mappings in sync would be complex and error-prone
- The **chunkserver is the final authority** on what chunks it actually has on disk — data may "spontaneously vanish" (e.g., disk failure)
- Simpler to just **poll at startup** and maintain via heartbeats thereafter

### 6.4 The Operation Log

The operation log is the **most critical piece of GFS**:

- It is the **only persistent record** of metadata
- It serves as a **logical timeline** defining the order of all operations
- Files, chunks, and their versions are identified by the logical times at which they were created

**Persistence and Replication:**
- The operation log is **replicated on multiple remote master machines**
- A metadata change is **not visible to clients** until the log record has been **flushed to disk both locally and on all replicas**
- Log records are **batched** before flushing to reduce I/O overhead

**Recovery via Checkpoints:**
- The master periodically **checkpoints** its entire in-memory state to disk
- Checkpoint format: a **compact B-tree** that can be directly **memory-mapped** without parsing
- Recovery process: load the latest checkpoint + replay only the log records after it
- Checkpointing is done in a **separate thread** without blocking incoming mutations
- Takes about **1 minute** for a cluster with a few million files

```
┌──────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Ops Log  │────▶│   Checkpoint     │────▶│   Recovery       │
│ (append) │     │ (periodic B-tree │     │ (load checkpoint │
│          │     │  snapshot)       │     │  + replay log)   │
└──────────┘     └──────────────────┘     └──────────────────┘
```

---

## 7. Chunk Replication

### 7.1 Default Replication Factor: 3

Every chunk is stored on **3 different chunkservers** by default.

**Why 3 replicas?**
- **Fault tolerance**: Survive the simultaneous failure of 2 chunkservers (or 2 disks)
- **Read performance**: Clients can read from the **closest** replica, distributing read load
- **Availability**: Even during replication/recovery, data remains accessible

### 7.2 Replica Placement Strategy

Replicas are spread across **both machines and racks**:

```
        Rack 1                 Rack 2                 Rack 3
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  ChunkServer A  │   │  ChunkServer C  │   │  ChunkServer E  │
│  [Chunk X - R1] │   │  [Chunk X - R2] │   │                 │
│                 │   │                 │   │                 │
│  ChunkServer B  │   │  ChunkServer D  │   │  ChunkServer F  │
│                 │   │                 │   │  [Chunk X - R3] │
└─────────────────┘   └─────────────────┘   └─────────────────┘
```

**Benefits of cross-rack placement:**
- Survives even if an **entire rack goes offline** (switch failure, power failure)
- **Read traffic** can exploit the aggregate bandwidth of multiple racks

**Trade-off:**
- **Write traffic** must flow across racks (higher latency), but this is acceptable because GFS prioritizes write throughput over write latency

---

## 8. Master's Housekeeping Responsibilities

### 8.1 Garbage Collection

GFS uses **lazy deletion** — a simpler and more reliable approach than immediate deletion:

1. When a file is deleted, the master **logs the deletion** but does not immediately reclaim storage
2. The file is **renamed to a hidden name** (with a deletion timestamp)
3. During the master's regular namespace scan, hidden files **older than 3 days** (configurable) are removed
4. Orphaned chunks (not referenced by any file) are identified and their metadata erased
5. ChunkServers learn about orphaned chunks via heartbeats and **delete their local replicas**

**Advantages of lazy garbage collection:**
- **Simple and reliable** in a large distributed system where failures are common
- **Merges with existing background scans** — amortized cost
- Provides a **safety net** against accidental deletion (files can be undeleted within 3 days)

### 8.2 Stale Replica Detection

- Each chunk has a **version number** maintained by the master
- When a lease is granted, the master **increments the version number** and informs all up-to-date replicas
- If a chunkserver was down during a mutation, its chunk version **will not advance** → it becomes **stale**
- On restart, the chunkserver reports its chunks with version numbers; master detects staleness
- Stale replicas are **never served to clients** and are garbage collected

### 8.3 Re-Replication

When the number of replicas falls below the target (e.g., a chunkserver dies):
- The master **re-replicates** the chunk by instructing a chunkserver to copy from an existing valid replica
- **Priority**: chunks with fewer replicas get higher priority (1 replica > 2 replicas)
- Chunks blocking client progress are boosted in priority
- **Throttled**: limits on concurrent cloning per chunkserver and per cluster to avoid overwhelming network

### 8.4 Rebalancing

- Master periodically examines replica distribution
- Moves replicas for better **disk space** and **load balancing**
- Gradually fills new chunkservers rather than swamping them instantly

---

## 9. Heartbeat Mechanism

The master communicates with chunkservers via **periodic HeartBeat messages**:

```
                    Master
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
     ChunkServer  ChunkServer  ChunkServer
          1           2           N
```

### 9.1 What the Master Sends (Piggybacked Instructions):
- Replicate specific chunks to other chunkservers
- Remove local replicas that are orphaned or stale
- Shut down the node
- Send an immediate block report

### 9.2 What the ChunkServer Reports Back:
- Confirmation that it is alive
- Storage capacity and fraction in use
- Number of data transfers in progress
- List of chunks it holds (Block Report — full report sent hourly, first report sent immediately at startup)

### 9.3 Failure Detection:
- Default heartbeat interval: **3 seconds**
- If no heartbeat received for **10 minutes**, the master considers the chunkserver **dead** and schedules re-replication of its chunks

---

## 10. Fault Tolerance

### 10.1 High Availability

#### Fast Recovery
- Both master and chunkservers can **restore state and restart in seconds**
- No distinction between normal and abnormal shutdown — servers are routinely killed and restarted
- Clients experience only a brief hiccup while reconnecting

#### Chunk Replication
- Default **3 replicas** on different racks
- Master continuously monitors and clones to maintain the target replication level
- Checksums protect against silent data corruption

#### Master Replication
- Operation log and checkpoints are **replicated on multiple machines**
- If the master machine fails, monitoring infrastructure starts a **new master process** on another machine using the replicated state
- Clients access the master via a **DNS alias** (e.g., `gfs-test`) that can be redirected

#### Shadow Masters
- **Read-only** shadow masters provide file system access even when the primary is down
- They are **shadows, not mirrors** — may lag the primary by fractions of a second
- They replay the operation log and poll chunkservers independently
- Useful for applications that can tolerate slightly stale metadata

### 10.2 Data Integrity: Checksums

Each chunkserver independently verifies data integrity:

```
┌──────────────────────────────────────────────────────┐
│                    One Chunk (64 MB)                  │
├──────┬──────┬──────┬──────┬──────┬──────┬────────────┤
│ 64KB │ 64KB │ 64KB │ 64KB │ 64KB │ 64KB │    ...     │
│ blk  │ blk  │ blk  │ blk  │ blk  │ blk  │            │
├──────┼──────┼──────┼──────┼──────┼──────┼────────────┤
│ CRC  │ CRC  │ CRC  │ CRC  │ CRC  │ CRC  │    ...     │
│  32b │  32b │  32b │  32b │  32b │  32b │            │
└──────┴──────┴──────┴──────┴──────┴──────┴────────────┘
```

- Every **64 KB block** within a chunk has a **32-bit checksum**
- Checksums are stored **in memory** and **persisted with logging**, separate from user data
- **On read**: chunkserver verifies checksums of overlapping blocks **before** returning data. If mismatch → return error, report to master, master clones from another replica.
- **On append**: incrementally update checksum for the last partial block; compute new checksums for new full blocks
- **On overwrite**: must read and verify the first and last blocks being partially overwritten, then write, then recompute checksums
- **During idle periods**: chunkservers scan inactive chunks to detect latent corruption

---

## 11. Performance Results (From the GFS Paper)

### 11.1 Micro-Benchmarks

Test setup: 1 master, 2 master replicas, 16 chunkservers, 16 clients, 100 Mbps Ethernet, 1 Gbps inter-switch link.

| Metric | 1 Client | 16 Clients | Theoretical Limit |
|--------|----------|------------|-------------------|
| **Read** | 10 MB/s (80% of limit) | 94 MB/s (75% of 125 MB/s link limit) | 125 MB/s |
| **Write** | 6.3 MB/s (50% of limit) | 35 MB/s (52% of 67 MB/s limit) | 67 MB/s |
| **Record Append** | 6.0 MB/s | 4.8 MB/s | Limited by last-chunk chunkserver |

**Write is slower than read** because each write must go to 3 replicas (3x network cost).

### 11.2 Real-World Cluster Characteristics

| Metric | Cluster A (R&D) | Cluster B (Production) |
|--------|-----------------|----------------------|
| Chunkservers | 342 | 227 |
| Available disk | 72 TB | 180 TB |
| Used disk | 55 TB | 155 TB |
| Number of files | 735K | 737K |
| Number of chunks | 992K | 1,550K |
| Metadata at chunkservers | 13 GB | 21 GB |
| Metadata at master | 48 MB | 60 MB |
| Read rate (sustained) | 580 MB/s | 380 MB/s |
| Write rate (sustained) | 1-25 MB/s | 13-117 MB/s |

**Key observation:** Master metadata is only **48–60 MB** — confirming that in-memory metadata is practical. Chunkserver metadata (mostly checksums) is much larger at 13–21 GB.

### 11.3 Deployment Scale

By the time of the paper:
- **50+ GFS clusters** deployed at Google
- Each with **thousands of storage nodes**
- Managing **petabytes of data**
- GFS underpins higher-level systems like **BigTable**

---

## 12. Summary of Key Design Decisions

| Design Decision | Rationale |
|----------------|-----------|
| **Single master** | Simplifies design; global knowledge for placement. Bottleneck avoided by separating metadata from data path. |
| **Large chunks (64 MB)** | Reduces metadata, reduces master interactions, enables persistent TCP connections. |
| **No client data caching** | Files too large for caching; eliminates cache coherence complexity. |
| **In-memory metadata** | Fast operations; compact (64 bytes/chunk); enables efficient background scanning. |
| **Chunk locations not persisted** | Chunkservers are the source of truth; eliminates sync issues in a dynamic cluster. |
| **Operation log + checkpoints** | Reliable persistence with fast recovery; logical timeline for ordering. |
| **Lazy garbage collection** | Simpler, more reliable, provides undelete safety net. |
| **Relaxed consistency model** | Simpler design; applications handle relaxed semantics with checksums and dedup. |
| **Append-optimized** | Matches the write-once-read-many workload; atomic record append for concurrent producers. |
| **Cross-rack replication** | Survives rack failures; increases aggregate read bandwidth. |

---

## References

1. Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung, "The Google File System", SOSP 2003
2. DS256 Lecture 1.4 Slides — Yogesh Simmhan, IISc Bangalore
3. LLaMA: Open and Efficient Foundation Language Models, Meta AI
4. The RefinedWeb Dataset for Falcon LLM

# Lecture 1.5: GFS Consistency, Mutations & HDFS

## DS256 — Scalable Systems for Data Science
### Module 1: Introduction to Big Data & Distributed Storage

---

## 1. Data Mutations in GFS

GFS supports three types of data mutations (operations that modify chunk contents):

### 1.1 Write

A **write** places data at a **client-specified file offset**.

```
write(fileId, offset, bytes[])
```

- The client knows exactly where in the file the data should go
- Overwrites any existing data at that offset
- Used when the client controls the file layout

### 1.2 Append

An **append** writes data at the client's perception of the end-of-file:

```
s = getFileSize(fileId)
write(fileId, s, bytes[])
```

- The client first queries the file size, then writes at that offset
- This is a **"regular" append** — essentially a write at the current EOF
- **Problem**: In a concurrent setting, multiple clients may compute the same EOF and overwrite each other's data — this is **not safe for concurrent use**

### 1.3 Record Append (GFS-Specific)

A **record append** causes a record to be **appended atomically at least once**, at an **offset chosen by GFS** (not the client):

```
offset = recordAppend(fileId, bytes[])
```

- GFS guarantees the record is written **as a contiguous, atomic unit**
- The system chooses the offset — the client gets back the offset where the record was placed
- **Critical for concurrent producers**: hundreds of clients can safely append to the same file without external synchronization
- Used for **producer-consumer queues** and **merged result files**
- Record append is restricted to be at most **1/4 of the chunk size** (16 MB) to limit worst-case fragmentation

---

## 2. GFS Consistency Model

GFS uses a **relaxed consistency model** — simpler to implement but requires applications to handle some complexity.

### 2.1 Region States

Consistency is defined for **regions** within a chunk — specific byte ranges that are written to by clients.

A region can be in one of three states:

| State | Definition |
|-------|-----------|
| **Defined** | All replicas have the **same content** for that region, AND the region reflects the **complete, unmingled update** from a single write operation. (Implies consistent.) |
| **Consistent** | All replicas have the **same byte contents** for that region, but the content may contain **mingled fragments** from multiple concurrent writers. |
| **Inconsistent** | Different replicas have **different byte contents** for that region. Different clients may see different data at different times. |

### 2.2 When Does Each State Occur?

| Scenario | Write Result | Record Append Result |
|----------|-------------|---------------------|
| **Serial success** (one writer, no concurrency) | **Defined** | **Defined** (interspersed with inconsistent padding/duplicates) |
| **Concurrent successes** (multiple writers, all succeed) | **Consistent but undefined** | **Defined** (interspersed with inconsistent) |
| **Any failure** | **Inconsistent** | **Inconsistent** |

### 2.3 Visual Example of Consistency States

```
                 Chunk Replica 1         Chunk Replica 2         Chunk Replica 3
                ┌────────────────┐      ┌────────────────┐      ┌────────────────┐
                │                │      │                │      │                │
  Single        │   AAAAAAAAAA   │      │   AAAAAAAAAA   │      │   AAAAAAAAAA   │
  Writer (C1)   │   (defined)    │      │   (defined)    │      │   (defined)    │
  succeeds      │                │      │                │      │                │
                ├────────────────┤      ├────────────────┤      ├────────────────┤
                │                │      │                │      │                │
  Concurrent    │   AABBCCAABB   │      │   AABBCCAABB   │      │   AABBCCAABB   │
  Writers       │   (consistent  │      │   (consistent  │      │   (consistent  │
  C1,C2,C3      │    but undef.) │      │    but undef.) │      │    but undef.) │
  all succeed   │                │      │                │      │                │
                ├────────────────┤      ├────────────────┤      ├────────────────┤
                │                │      │                │      │                │
  Failed        │   AABB????     │      │   AABBCCDD     │      │   AABB??CC     │
  mutation      │   (inconsist.) │      │   (inconsist.) │      │   (inconsist.) │
                │                │      │                │      │                │
                └────────────────┘      └────────────────┘      └────────────────┘
```

### 2.4 Key Insight: Consistent but Undefined

When multiple clients write concurrently and all succeed:
- All replicas end up with **identical content** (consistent) because all mutations are applied in the same serial order
- But the content is a **mixture of fragments** from different clients (undefined) — no single client's complete write is guaranteed to be intact
- **Data flow** from different clients may arrive in different orders and be buffered in memory
- **Control flow** (commit to disk) is serialized — FCFS (First Come, First Served)
- **No rollback** — once a partial write succeeds, it stays

---

## 3. Leases and Mutation Order

The core mechanism for maintaining **consistent mutation order across replicas** is the **lease system**.

### 3.1 How Leases Work

1. The master grants a **chunk lease** to one of the chunk's replicas → this replica becomes the **primary**
2. The primary **assigns serial numbers** to all mutations it receives
3. All replicas (primary + secondaries) apply mutations in **serial number order**
4. The **global mutation order** is defined by:
   - The **lease grant order** (which primary is chosen, in what order)
   - Within a lease, the **serial numbers** assigned by the primary

**Lease properties:**
- Initial timeout: **60 seconds**
- Primary can request **extensions** (piggybacked on heartbeats) indefinitely while the chunk is being mutated
- Master can **revoke** a lease early (e.g., before a rename/snapshot)
- If master loses contact with primary, it **waits for the lease to expire** before granting a new one (safety)

---

## 4. Write Data Flow — The Full Protocol

This is the most important protocol in GFS. It shows how data and control flow are **decoupled** for maximum efficiency.

### 4.1 The Seven Steps

```
                    Master
                      │
            ┌─────────┤
            │ Step 1   │ Step 2
            │ Request  │ Reply (primary ID,
            │ lease    │ secondary locations)
            ▼ info     │
          Client ◄─────┘
            │
            │ Step 3: Push data to ALL replicas (any order)
            │         Each chunkserver stores in LRU buffer
            │
            ├──────────────▶ Primary         (data)
            ├──────────────▶ Secondary A     (data)
            └──────────────▶ Secondary B     (data)
            
            │ Step 4: All replicas ACK data received
            │         Client sends WRITE REQUEST to Primary
            │
            ▼
          Primary
            │ Step 5: Assigns serial # to mutation
            │         Applies mutation locally
            │         Forwards write request to all secondaries
            │
            ├──────────────▶ Secondary A  (apply in serial # order)
            └──────────────▶ Secondary B  (apply in serial # order)
            
            │ Step 6: Secondaries ACK to Primary
            │
            ▼
          Primary
            │ Step 7: Primary ACKs to Client
            ▼
          Client (write complete)
```

### 4.2 Detailed Step-by-Step

| Step | Action | Details |
|------|--------|---------|
| **1** | Client → Master | Client asks which chunkserver holds the current lease (primary) and the locations of all replicas |
| **2** | Master → Client | Master replies with primary identity + secondary locations. Client **caches** this. Only re-contacts master if primary becomes unreachable. |
| **3** | Client → All Replicas | Client pushes data to **all replicas** (primary + secondaries), in **any order**. Each chunkserver stores the data in an internal **LRU buffer cache** (not yet written to disk). |
| **4** | Client → Primary | Once **all** replicas ACK receiving the data, client sends a **write request** to the primary (identifying the data pushed in step 3). |
| **5** | Primary → Secondaries | Primary **assigns a serial number** to the mutation. Applies it **locally** first. Then forwards the write request (with serial #) to all secondaries. |
| **6** | Secondaries → Primary | Each secondary applies the mutation in **serial number order** and ACKs to primary. |
| **7** | Primary → Client | Primary ACKs to client after receiving ACKs from **all secondaries**. Errors at any replica are reported. |

### 4.3 Failure Handling

| Failure Scenario | What Happens |
|-----------------|-------------|
| **Primary fails before writing** | No writes were performed. Client retries from the beginning. |
| **Primary succeeds, some secondaries fail** | Write is reported as **failed** to the client. The affected region is left **inconsistent**. Client retries (steps 3–7), potentially multiple times before falling back to a full retry. |
| **Large write straddles chunk boundary** | GFS client code breaks it into **multiple write operations**, each following the above flow. May be interleaved with concurrent operations from other clients. |

### 4.4 Why Decouple Data Flow from Control Flow?

This is a key architectural insight:

```
Control Flow:  Client ──▶ Primary ──▶ Secondaries
               (Ensures serial order, consistency)

Data Flow:     Client ──▶ Nearest chunkserver ──▶ Next nearest ──▶ ...
               (Pipelined, topology-aware, maximizes bandwidth)
```

**Control flow** (which writes happen in which order) flows from client → primary → secondaries. This ensures a single serial order.

**Data flow** (the actual bytes) is pushed through a **pipeline** along a chain of chunkservers, optimized for network topology:

- Each machine sends data to the **nearest machine** that hasn't received it yet
- "Distance" is estimated from **IP addresses** (same rack < different rack < different datacenter)
- **Pipelined**: once a chunkserver receives some data, it **starts forwarding immediately** (doesn't wait for the full transfer)
- Fully utilizes each machine's **outbound bandwidth** without splitting it among multiple recipients

**Performance:** Transferring B bytes to R replicas takes approximately:
```
Time ≈ B/T + R × L
```
Where `T` = network throughput, `L` = latency between two machines. With 100 Mbps links and <1 ms latency, **1 MB can be distributed in ~80 ms**.

---

## 5. Atomic Record Append — Detailed Protocol

### 5.1 How It Works

1. Client pushes data to all replicas of the **last chunk** of the file
2. Client sends append request to the **primary**
3. Primary checks: **will this record fit in the current chunk?**
   - **If NO** (would exceed 64 MB): Primary **pads** the chunk to the end of its capacity, tells secondaries to do the same, tells client to **retry on the next chunk**
   - **If YES**: Primary appends the record to its replica at a chosen offset, tells secondaries to write at the **exact same offset**, ACKs to client with the offset

### 5.2 Failure and Retry Behavior

- If the append **fails at any replica**, the client **retries**
- This means replicas may contain **duplicate records** (same record written multiple times)
- GFS guarantees: the record is written **at least once** as an atomic unit at the **same offset on all replicas of some chunk**, if the operation reports success
- Regions with successful appends are **defined** (consistent)
- Intervening regions (padding, failed partial appends) are **inconsistent**

### 5.3 The "At Least Once" Semantics

```
Chunk on Replica 1          Chunk on Replica 2          Chunk on Replica 3
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│ Record A  ✓      │        │ Record A  ✓      │        │ Record A  ✓      │
│ Record B  ✓      │        │ Record B  ✓      │        │ Record B  ✓      │
│ Record C (failed)│        │ Record C  ✓      │        │ Record C (failed)│
│ Padding          │        │ Padding          │        │ Padding          │
│ Record C  ✓      │ ◄──    │ Record C  ✓      │ ◄──    │ Record C  ✓      │ ◄── Retry
│ Record D  ✓      │        │ Record D  ✓      │        │ Record D  ✓      │
└──────────────────┘        └──────────────────┘        └──────────────────┘
                                   ▲
                                   │
                              All replicas have Record C at the SAME offset (the retry)
                              But some have a partial/failed copy earlier too
```

---

## 6. Snapshot

Snapshot creates a **copy of a file or directory tree** almost **instantaneously** using **copy-on-write (CoW)**.

### 6.1 Snapshot Process

1. Master **revokes** all outstanding leases on chunks of the file (or waits for them to expire)
   - This ensures any subsequent writes must contact the master first
2. Master **logs** the snapshot operation to the operation log
3. Master **duplicates the metadata** of the source file → creates a new snapshot file pointing to the **same chunks**
4. Master **increments the reference counter** on each shared chunk

### 6.2 Copy-on-Write Behavior

When a client later tries to **write to a shared chunk** (reference count > 1):

```
Before Write:
  Source File ────┐
                  ├──▶ Chunk C (refcount = 2)
  Snapshot File ──┘

After Write:
  Source File ─────────▶ Chunk C' (new copy, refcount = 1) ← writes go here
  Snapshot File ───────▶ Chunk C  (original, refcount = 1) ← unchanged
```

1. Client sends write request to master
2. Master notices `refcount > 1` for chunk C
3. Master picks a **new chunk handle C'**
4. Master asks chunkservers holding C to **locally copy** C to C' (local copy = fast, no network transfer)
5. Master grants a lease on C' and replies to client
6. Client writes to C' normally

**Key benefit:** Snapshots are **O(metadata)** — only metadata is duplicated, not data. Data is only copied when actually modified.

---

## 7. Implications for Applications

The relaxed consistency model places some burden on applications. GFS applications use these techniques:

### 7.1 Append Rather Than Overwrite

- Virtually all GFS applications **append** to files rather than overwriting
- Appending is more efficient and more resilient to application failures

### 7.2 Checkpointing

**Pattern 1: Single writer**
- A writer generates a file from **beginning to end**
- After writing all data, it **atomically renames** the file to a permanent name, OR
- It **periodically checkpoints** how much has been successfully written (with application-level checksums)
- Readers **only process data up to the last checkpoint** (which is in the defined state)
- This lets writers **restart incrementally** on failure and keeps readers from processing incomplete data

**Pattern 2: Multiple writers (producer-consumer)**
- Multiple producers concurrently **record-append** to the same file
- Record append preserves each writer's output
- Readers handle padding and duplicates:
  - Each record contains a **checksum** so readers can verify validity and detect fragments
  - Each record contains a **UUID** so readers can identify and filter duplicates
  - These checks are implemented in a **shared library** used by all GFS applications

---

## 8. Master: Namespace Management and Locking

### 8.1 Namespace Representation

- GFS does **not** have a per-directory data structure listing all files
- Instead, the namespace is a **lookup table** mapping **full pathnames to metadata**
- Stored compactly using **prefix compression**
- No hard links or symbolic links

### 8.2 Locking Mechanism

Each node in the namespace tree has an associated **read-write lock**.

For an operation on `/d1/d2/.../dn/leaf`:
- Acquire **read locks** on: `/d1`, `/d1/d2`, ..., `/d1/d2/.../dn`
- Acquire a **read lock** or **write lock** on: `/d1/d2/.../dn/leaf`

**Example: Preventing conflicts between snapshot and file creation**

| Operation | Locks Acquired |
|-----------|---------------|
| Snapshot `/home/user` → `/save/user` | Read: `/home`, `/save` — Write: `/home/user`, `/save/user` |
| Create `/home/user/foo` | Read: `/home`, `/home/user` — Write: `/home/user/foo` |

These serialize correctly because they **conflict on `/home/user`** (write lock vs read lock).

**Key property:** Multiple file creations in the **same directory** can proceed **concurrently**:
- Each acquires a **read lock** on the directory (prevents deletion/rename)
- Each acquires a **write lock** on its own filename (serializes attempts to create the same filename)

**Deadlock prevention:** Locks are acquired in a **consistent total order** — by level in the namespace tree, then lexicographically within the same level.

---

## 9. Master: Replica Placement Strategy

### 9.1 Goals
- **Maximize data reliability and availability**
- **Maximize network bandwidth utilization**

### 9.2 Placement Rules

| Rule | Details |
|------|---------|
| Spread across **machines** | Guards against individual disk/machine failure |
| Spread across **racks** | Guards against rack-level failures (switch, power) |
| No DataNode has more than **one replica** of a block | Ensures machine failure loses at most one replica |
| No rack has more than **two replicas** of a block | Ensures rack failure loses at most two replicas |

### 9.3 Placement for New Chunks
The master considers:
1. Place on chunkservers with **below-average disk utilization** (equalize over time)
2. **Limit recent creations** per chunkserver (avoids imminent write traffic overload)
3. **Spread across racks**

### 9.4 Read Optimization
When returning chunk locations to a reader, the master returns replicas **sorted by closeness** to the reader (same rack > same datacenter > remote).

### 9.5 Distance Metric

```
Distance = sum of distances to common ancestor

Same machine:       distance = 0
Same rack:          distance = 2  (node→rack→node)
Different rack:     distance = 4  (node→rack→switch→rack→node)
Different datacenter: distance = 6
```

---

## 10. Hadoop Distributed File System (HDFS)

HDFS is the **open-source implementation** inspired by GFS, forming the storage layer of the Apache Hadoop ecosystem.

### 10.1 GFS vs HDFS Terminology

| GFS Term | HDFS Term |
|----------|-----------|
| Master | **NameNode** |
| ChunkServer | **DataNode** |
| Chunk | **Block** |
| Chunk Size: 64 MB | Block Size: **128 MB** (HDFS v3) |
| Operation Log | **Journal / EditLog** |
| Checkpoint | **FSImage** |
| Shadow Master | **Checkpoint Node / Backup Node** |

### 10.2 Architecture (Same as GFS)

```
┌─────────────────────────────────────────────────────────┐
│                     HDFS Cluster                        │
│                                                         │
│  ┌───────────────┐     ┌───────────────────────────┐    │
│  │   NameNode    │     │  Namespace (FSImage)       │    │
│  │               │────▶│  File → Block mapping      │    │
│  │               │     │  Block → DataNode mapping   │    │
│  │               │     │  EditLog (Journal)          │    │
│  └───────┬───────┘     └───────────────────────────┘    │
│          │                                               │
│          │ Heartbeats (every 3 sec)                      │
│          │ Block Reports (every 1 hour)                  │
│          │                                               │
│  ┌───────▼───────┐  ┌───────────────┐  ┌─────────────┐  │
│  │  DataNode 1   │  │  DataNode 2   │  │  DataNode N  │  │
│  │  ┌────┐┌────┐ │  │  ┌────┐┌────┐ │  │  ┌────┐     │  │
│  │  │Blk1││Blk3│ │  │  │Blk1││Blk2│ │  │  │Blk2│     │  │
│  │  └────┘└────┘ │  │  └────┘└────┘ │  │  └────┘     │  │
│  │  Linux FS     │  │  Linux FS     │  │  Linux FS   │  │
│  └───────────────┘  └───────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 11. HDFS Block Reports

### 11.1 Purpose
A DataNode identifies all block replicas in its possession by sending a **block report** to the NameNode.

### 11.2 Contents
Each block report entry contains:
- **Block ID**: unique identifier
- **Generation stamp**: version number (analogous to GFS chunk version number)
- **Block length**: how many bytes are stored

### 11.3 Timing
- **First block report**: sent **immediately** after DataNode registration (at startup)
- **Subsequent reports**: sent **every hour**
- NameNode can request an **immediate block report** via heartbeat response

---

## 12. HDFS Heartbeat Mechanism

### 12.1 DataNode → NameNode Heartbeat

| Property | Value |
|----------|-------|
| Default interval | **3 seconds** |
| Dead threshold | **No heartbeat for 10 minutes** → considered out of service |
| Action on death | NameNode schedules **re-replication** of all blocks on dead DataNode |

### 12.2 Heartbeat Payload (DataNode → NameNode)
- Storage capacity (total disk)
- Fraction of storage in use
- Number of data transfers currently in progress

### 12.3 Heartbeat Response (NameNode → DataNode)
- **Replicate** specific blocks to other DataNodes
- **Remove** local replicas (orphaned/stale)
- **Shut down** the node
- **Send immediate block report**

---

## 13. HDFS Checkpoint and Backup Nodes

### 13.1 The Problem with Journal Replay

- Recreating NameNode state from a week's worth of journal (edit log) entries can take **hours**
- The journal grows continuously as operations are performed
- Need a mechanism to periodically compact the state

### 13.2 Checkpoint Node

```
                    NameNode
                      │
          ┌───────────┤
          │ Download   │ Upload
          │ FSImage +  │ new FSImage
          │ Journal    │
          ▼            │
     Checkpoint Node ──┘
     (merges locally)
```

1. Downloads the current **FSImage (checkpoint)** and **Journal (edit log)** from the NameNode
2. **Merges** them locally: applies all journal entries to the FSImage
3. Uploads the **new FSImage** back to the NameNode
4. NameNode can now **truncate the journal** (discard entries already merged into the checkpoint)

### 13.3 Backup Node

The Backup Node is a **more capable** version of the Checkpoint Node:

| Feature | Checkpoint Node | Backup Node |
|---------|----------------|-------------|
| Creates periodic checkpoints | ✓ | ✓ |
| Maintains in-memory namespace | ✗ | ✓ (synced with NameNode) |
| Receives live journal stream | ✗ | ✓ (applies in real-time) |
| Can serve read-only queries | ✗ | ✓ (**Read-only NameNode**) |
| Checkpoint creation | Requires download | Done **locally** (faster) |

The Backup Node accepts the journal stream of namespace transactions from the active NameNode, saves them to its local store, and applies them to its own namespace image in memory. This is analogous to GFS's shadow master.

---

## 14. HDFS Block Creation Pipeline

### 14.1 How Data is Written

When a client writes a block, DataNodes form a **pipeline**:

```
Client ──▶ DataNode 1 ──▶ DataNode 2 ──▶ DataNode 3
           (closest)                      (furthest)
```

The pipeline order **minimizes total network distance** from the client to the last DataNode.

### 14.2 Write Protocol Details

| Property | Details |
|----------|---------|
| Packet size | Data pushed as **64 KB packet buffers** |
| ACK handling | **Asynchronous** — client doesn't wait for each packet to be ACKed before sending the next |
| Outstanding ACKs | Maximum number of unacknowledged packets in flight |
| Checksums | Client **generates checksums** for each block; DataNode **stores checksums** alongside the block |
| Read verification | Client verifies checksums during reads to **detect corruption** |

---

## 15. HDFS Block Placement Policy

### 15.1 Replica Placement Rules

| Replica | Placement |
|---------|-----------|
| **1st replica** | On the **writer's node** (same machine as the client, if the client is on a DataNode) |
| **2nd replica** | On a **different node** in a **different rack** |
| **3rd replica** | On another **different node** in the **same rack** as the 2nd replica |

**Constraints:**
- No DataNode has more than **one replica** of a block
- No rack has more than **two replicas** of a block

### 15.2 Trade-off
- Writing: 2nd replica requires **cross-rack transfer** (higher latency)
- Reading: NameNode returns replicas **sorted by closeness** to the reader → fast reads from nearby replicas
- Reliability: Survives the failure of any single rack

---

## 16. HDFS Replication Management

### 16.1 Detecting Under/Over-Replication

The NameNode detects replication anomalies from **block reports**:

**Under-replication:**
- Placed in a **priority queue** (block with only 1 replica has **highest priority**)
- A background thread scans the queue and decides where to place new replicas
- Factors: disk space utilization, rack diversity, bandwidth limits

**Over-replication:**
- Remove replicas without reducing the number of **racks** hosting the block
- Prefer removing from the DataNode with the **least available disk space**

### 16.2 Real-World Recovery Example (from GFS paper)

| Scenario | Recovery Time | Details |
|----------|---------------|---------|
| **1 chunkserver killed** (15,000 chunks, 600 GB) | **23.2 minutes** | 91 concurrent clones, 6.25 MB/s per clone, effective rate = 440 MB/s |
| **2 chunkservers killed** (32,000 chunks, 1.32 TB) | **2 minutes** for critical chunks | 266 chunks reduced to 1 replica → cloned at highest priority to 2x replication |

---

## 17. HDFS Balancer

The Balancer is a tool that **equalizes disk space utilization** across DataNodes in the cluster.

### 17.1 Goal
The **utilization** of any node (percentage of disk used) should differ from the **cluster average utilization** by no more than a configurable **threshold**.

### 17.2 How It Works
- **Iteratively moves replicas** from nodes with higher utilization to nodes with lower utilization
- Maintains **data availability** during moves (ensures replication factor is never violated)
- **Minimizes inter-rack copying** (prefers intra-rack moves when possible)
- **Limits bandwidth consumed** to avoid impacting application performance

---

## 18. Disk Failures in the Real World

### 18.1 The Reality of MTTF

Disk manufacturers quote Mean Time To Failure (MTTF) of **1,000,000 hours** (~114 years). But this is misleading:

- With **1000 disks** in a cluster, expected failure rate = 1,000,000 / 1,000 = **1,000 hours** = **~42 days per failure**
- With **10,000 disks**, expect a disk failure roughly **every 4 days**
- Research (Schroeder & Gibson, USENIX FAST 2007) showed **real-world failure rates are significantly higher** than manufacturer specifications

### 18.2 Annual Failure Rates (Observed)

Real-world studies show:
- Annual replacement rates of **2-4%** for enterprise disks
- Up to **8-10%** for consumer-grade disks
- Failure rates **increase with disk age** (not constant as MTTF assumes)

**This is precisely why GFS/HDFS design assumes failures are routine.**

---

## 19. Erasure Coding: An Alternative to Full Replication

### 19.1 The Problem with 3x Replication

3x replication provides excellent fault tolerance but at **200% storage overhead** — for every 1 TB of data, you need 3 TB of disk space.

### 19.2 Erasure Coding Concept

Erasure coding tolerates failures **without full replication** using parity information:

```
  Data Units (k=4)           Parity Units (m=2)
┌──────┬──────┬──────┬──────┬──────┬──────┐
│  D1  │  D2  │  D3  │  D4  │  P1  │  P2  │
└──────┴──────┴──────┴──────┴──────┴──────┘
```

- **k data blocks** + **m parity blocks** = total (k+m) blocks
- Can tolerate the loss of **any m blocks** (data or parity)
- Missing blocks are **reconstructed** from the remaining blocks using algebraic operations (Reed-Solomon encoding)

### 19.3 Storage Efficiency Comparison

| Method | Storage Overhead | Fault Tolerance |
|--------|-----------------|-----------------|
| **3x Replication** | 200% (3x storage) | Survives loss of any 2 copies |
| **RS(6,3)** — 6 data + 3 parity | 50% (1.5x storage) | Survives loss of any 3 blocks |
| **RS(4,2)** — 4 data + 2 parity | 50% (1.5x storage) | Survives loss of any 2 blocks |

### 19.4 Trade-offs

| Aspect | Replication | Erasure Coding |
|--------|-------------|----------------|
| **Storage efficiency** | Poor (3x) | Good (1.5x) |
| **Read performance** | Excellent (direct access) | Good (direct access if block available) |
| **Recovery cost** | Low (just copy) | High (compute from multiple blocks) |
| **Write overhead** | Low (just replicate) | Higher (compute parity) |
| **Best for** | Hot data, frequent access | Cold/warm data, archival |

HDFS supports erasure coding natively (since Hadoop 3.0) as an alternative to replication for infrequently accessed data.

---

## 20. Cloud Storage Categories

Modern cloud platforms offer multiple storage tiers, each serving different use cases:

### 20.1 IaaS Storage Categories

| Category | AWS | Azure | Use Case |
|----------|-----|-------|----------|
| **Object Storage** | S3 | Blob Storage | Unstructured data, media, backups, data lakes |
| **Block Storage** | Elastic Block Storage (EBS) | Azure Disks | VM disks, databases, low-latency I/O |
| **Network File System** | Elastic File System (NFS), Lustre | Azure Files (NFS, SMB), HPC Cache | Shared file access, HPC workloads |
| **Backup** | AWS Backup | Azure Backup | Data protection, compliance |
| **Sync & Transfer** | DataSync, Snow/Import-Export | FileSync, Bulk Transfer Disks | Data migration, hybrid cloud |

### 20.2 Where GFS/HDFS Fits

GFS/HDFS is a **cluster file system** — it maps most closely to the **Network File System** category in cloud storage, but at a much larger scale. Cloud object storage (S3, Azure Blob) has largely replaced HDFS for many use cases, but HDFS remains important for Hadoop-ecosystem workloads.

---

## 21. Other Distributed Storage Systems

| System | Year | Key Innovation |
|--------|------|---------------|
| **Ceph** | 2006 (OSDI) | Scalable, high-performance DFS with CRUSH algorithm for decentralized placement |
| **Kademlia** | 2002 | Peer-to-peer DHT using XOR distance metric for routing |
| **HopsFS** | 2017 | Scaled HDFS to **1 million+ operations per second** by distributing NameNode metadata across multiple nodes |

---

## 22. Summary: Key Takeaways

### From GFS Consistency & Mutations:

1. **Three mutation types** — write (client offset), append (client EOF), record append (GFS-chosen offset, atomic, at-least-once)

2. **Relaxed consistency** — defined > consistent > inconsistent. Applications use checksums, UUIDs, and checkpointing to handle the relaxed model.

3. **Leases ensure mutation order** — one primary per chunk assigns serial numbers; all replicas apply in the same order

4. **Data flow decoupled from control flow** — control ensures ordering (client → primary → secondaries); data is pipelined along the network topology for maximum throughput

5. **Snapshots use copy-on-write** — O(metadata) cost; data only copied on modification

### From HDFS:

6. **HDFS is the open-source GFS** — same architecture with different terminology (NameNode, DataNode, Block)

7. **Checkpoint/Backup Nodes** solve journal replay latency — periodic merging of FSImage + EditLog

8. **Pipeline writes** with 64 KB packets, async ACKs, and client-generated checksums

9. **Erasure coding** offers 50% overhead (vs 200% for 3x replication) for cold data

10. **Disk failures are routine** — real-world MTTF is much worse than manufacturer specs; design must assume failure

---

## References

1. Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung, "The Google File System", SOSP 2003
2. Konstantin Shvachko, Hairong Kuang, Sanjay Radia, Robert Chansler, "The Hadoop Distributed File System", IEEE MSST 2010
3. HDFS Architecture Guide, D. Borthakur, 2008
4. Konstantin V. Shvachko, "HDFS Scalability: The Limits to Growth", ;login: April 2010
5. Bianca Schroeder, Garth A. Gibson, "Disk Failures in the Real World", USENIX FAST 2007
6. Ismail, Mahmoud, et al., "Scaling HDFS to more than 1 million operations per second with HopsFS", IEEE/ACM CCGrid 2017
7. Eltabakh et al., "CoHadoop: Flexible Data Placement and Its Exploitation in Hadoop", VLDB 2011
8. Weil, Sage A., et al., "Ceph: A Scalable, High-Performance Distributed File System", OSDI 2006
9. DS256 Lecture 1.5 Slides — Yogesh Simmhan, IISc Bangalore

