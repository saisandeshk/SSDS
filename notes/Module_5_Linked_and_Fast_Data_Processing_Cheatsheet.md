# Module 5 Cheatsheet: Linked and Fast Data Processing

## DS256 - Scalable Systems for Data Science

This cheatsheet compresses the three parts of Module 5:

1. Kafka
2. CEP / Fast Data Querying / Spark Streaming
3. Pregel / Large-Scale Graph Processing

---

## 1. Module Map

```text
Data in motion --> Kafka --> CEP / Streaming Analytics --> Graph Analytics

Transport         Query + stream compute              Linked data compute
```

### One-line summary

- **Kafka** moves events reliably at scale.
- **CEP / Spark Streaming** continuously analyzes incoming events.
- **Pregel** runs iterative algorithms over graph-structured linked data.

---

## 2. Kafka Cheatsheet

### Core abstractions

- **Producer**: writes messages
- **Broker**: stores and serves messages
- **Topic**: named logical stream
- **Partition**: ordered append-only shard of a topic
- **Consumer**: reads messages
- **Consumer group**: set of consumers sharing partitions
- **Offset**: position of a message in a partition

### Key ideas

- topic is split into partitions for scale
- ordering is guaranteed **within** a partition, not across partitions
- producers choose partition by key or round-robin
- Kafka uses a **pull model** for consumers
- retention allows replay of old data

### High-yield distinctions

- **Push vs pull**: Kafka uses pull so consumers control rate
- **Keyed partitioning**: same key goes to same partition
- **Consumer group**: each partition is read by only one consumer in that group
- **Leader/follower**: leader serves reads/writes; followers replicate

### Why Kafka is fast

- append-only logs
- batching
- OS page cache
- zero-copy via `sendfile`

### Typical exam points

- ordering only per partition
- offsets tracked by consumers
- rebalancing occurs when consumers join/leave or partitions change
- retention and replay are central Kafka features

---

## 3. CEP and Fast Data Querying Cheatsheet

### Core idea

**CEP = continuous SQL-like querying over event streams**.

### Event model

- event = tuple
- event stream = unbounded table
- continuous query = query that keeps running as new events arrive

### Main operators

- filter
- transform
- aggregate
- window
- join
- pattern / sequence detection

### Window types

| Window | Basis | Emits |
|---|---|---|
| Sliding length | last `N` events | every new event |
| Batch length | last `N` events | every `N` events |
| Sliding time | last `T` time units | every new event |
| Batch time | last `T` time units | once per period |

### Join reminder

- stream joins need windows to keep state finite
- one-window join: only one side retained
- two-window join: both sides retained

### Pattern reminder

CEP can detect ordered subsequences of events, not just individual records.

### CEP engine examples

- Siddhi
- Esper

---

## 4. Spark Streaming / Structured Streaming Cheatsheet

### DStreams (Spark 1.x)

- DStream = sequence of RDDs over time
- micro-batch model
- transform DStreams and write outputs

### Why D-Streams were important

- deterministic micro-batches
- lineage-based recovery
- parallel recomputation on failure
- batch and streaming use same underlying RDD machinery

### D-Streams paper keywords

- **supervised by micro-batches**
- **fault tolerance via lineage + checkpoints**
- **parallel recovery**
- **straggler mitigation via speculative execution**

### Structured Streaming (Spark 2.x+)

- stream modeled as **unbounded table / DataFrame**
- user writes batch-like query
- Spark executes it incrementally

### Query lifecycle

1. define input source
2. transform data
3. define sink and output mode
4. specify trigger / checkpointing
5. start query

### Output modes

- **Append**: only new rows
- **Update**: only changed rows
- **Complete**: full result table each trigger

### Trigger modes

- default micro-batch
- fixed trigger interval
- once
- continuous

### Stateful vs stateless

- **Stateless**: `select`, `filter`, `map`
- **Stateful**: aggregates, grouped state, windows

### Event time vs processing time

- **Event time**: when event happened
- **Processing time**: when Spark processed it

### Event-time windows

- tumbling: one non-overlapping window per event
- sliding / overlapping: event may belong to multiple windows

---

## 5. Pregel Cheatsheet

### Core idea

Pregel is a **vertex-centric BSP model** for large-scale graph processing.

### Computation model

```text
superstep:
read previous messages -> update local state -> send new messages -> vote to halt
```

### Vertex can do

- read messages from previous superstep
- send messages for next superstep
- update its own value
- update outgoing-edge values
- vote to halt

### Termination

Program stops when:

- all vertices are inactive
- no messages are in transit

### Message semantics

- exactly once delivery
- no guaranteed order
- can send to any known vertex ID

### Why vertex-centric is good

- natural for iterative graph algorithms
- easier than repeated joins
- avoids many shared-memory race conditions

### Key optimizations

- **Combiner**: reduce multiple messages for same destination
- **Aggregator**: compute a global reduced value for next superstep

### Graph algorithm mapping

- **WCC**: propagate max or canonical label
- **BFS**: propagate hop count
- **Shortest path**: propagate tentative weighted distance
- **PageRank**: propagate rank mass

### Giraph reminder

- open-source Pregel-style system on Hadoop
- uses master, workers, ZooKeeper, checkpoints, partitioner

---

## 6. Side-by-Side Comparison

| Part | Main abstraction | Computation style | Typical output |
|---|---|---|---|
| Kafka | topic / partition / offset | transport + storage of events | ordered event log |
| CEP | event stream as unbounded table | continuous query | alerts, filtered streams, aggregates |
| Spark Streaming | micro-batch / DStream / DataFrame | repeated incremental execution | sink outputs and result tables |
| Pregel | vertex + edges + messages | iterative supersteps on graphs | updated graph / graph metrics |

---

## 7. Fast Revision Table

| Concept | Remember |
|---|---|
| Kafka partition | unit of parallelism and ordering |
| Kafka offset | consumer position in partition |
| CEP window | finite subset of an unbounded stream |
| Sliding window | overlaps; updates more frequently |
| DStream | sequence of RDDs over time |
| Structured Streaming | stream as unbounded table |
| Append / Update / Complete | how much of the result table is emitted |
| Event time | time inside the record |
| Processing time | time at engine |
| Pregel superstep | synchronous compute + message round |
| Vote to halt | vertex deactivates until messaged |
| Combiner | local message reduction |
| Aggregator | global reduction across vertices |

---

## 8. Most Likely Exam Confusions

- Kafka is not the analytics engine; it is the broker / log system.
- CEP is about continuous query semantics, not just message transport.
- Structured Streaming is higher-level than raw DStreams.
- Event time and processing time are not the same.
- Pregel is not a graph database; it is a graph analytics model.
- Combiner and aggregator solve different problems.
- Vote-to-halt does not mean permanent death; a message can reactivate the vertex.

---

## 9. Final 10-Line Recall

```text
Kafka: brokered publish-subscribe log with topics, partitions, offsets.
Ordering only within a partition.
CEP: continuous SQL over event streams.
Windows make unbounded streams aggregatable.
Spark Streaming: micro-batches; DStreams are sequences of RDDs.
Structured Streaming: stream as unbounded DataFrame/table.
Output modes: append, update, complete.
Event time != processing time.
Pregel: think like a vertex, compute in supersteps.
Messages + voteToHalt + combiners/aggregators = core model.
```