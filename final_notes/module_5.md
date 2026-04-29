# Lecture 5.1: Apache Kafka — Publish-Subscribe Event Broker

## DS256 - Scalable Systems for Data Science
### Module 5: Linked and Fast Data Processing

> **References**:
> - Lecture slides: M5.Linked and Fast Data Processing.pdf (slides 5–33)
> - Kreps, Narkhede, Rao, "Kafka: A Distributed Messaging System for Log Processing", NetDB 2011
> - Kafka: The Definitive Guide, Neha Narkhede, Gwen Shapira & Todd Palino, O'Reilly, 2017

---

## 1. Motivation: The Need for Fast Data

### 1.1 The Four V's of Big Data

Big Data is characterized by **four V's**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FOUR V's of BIG DATA                        │
├────────────────┬────────────────┬────────────────┬─────────────────┤
│    VOLUME      │   VELOCITY     │    VARIETY     │    VERACITY     │
│                │                │                │                 │
│  Petabytes of  │  Speed of data │  Structured,   │  Quality and    │
│  data stored   │  generation    │  unstructured, │  reliability    │
│  and processed │  and arrival   │  semi-struct.  │  of data        │
└────────────────┴────────────────┴────────────────┴─────────────────┘
```

**Velocity** creates the "Fast Data" challenge — data arrives continuously at high rates and must be processed with low latency.

### 1.2 Real-World Example: IISc Smart Campus IoT Stack

Consider the **IISc Smart Campus IoT System** (a real deployment):

```
┌─────────────────────────────────────────────────────────────────────┐
│                   IISc Smart Campus Scale                           │
├─────────────────────────────────────────────────────────────────────┤
│  • 440 Acres, 8 km perimeter                                        │
│  • 50 buildings (offices, hostels, residences, stores)              │
│  • 10,000 people                                                    │
│  • Water usage: 4 million liters/day                                │
│  • Power consumption: 10 MW                                         │
├─────────────────────────────────────────────────────────────────────┤
│                        Sensor Network                               │
├─────────────────────────────────────────────────────────────────────┤
│  • Hundreds of wireless motes and sensors                           │
│  • Water level sensors in OHTs (Overhead Tanks) and GLRs            │
│  • Water quality sensors (TDS, temperature)                         │
│  • Flow sensors for monitoring inflow/outflow                       │
│  • O(minute) sampling intervals                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Analytics Requirements**:
- **Real-time sensing**: Water level, quality, network health
- **Online operational decisions**: Water quality alerts, overflow/underflow alerts, battery drain alerts
- **Low-latency processing**: Smart power and transportation need sub-second latency

### 1.3 The IoT Analytics Pipeline

```
                                    ┌──────────────────────────────────┐
                                    │         Analytics Pipeline       │
                                    ├──────────────────────────────────┤
Water                               │  • Complex Event Processing      │
Infra. ──► Sensors/Motes ──► Edge  │    (Siddhi CEP)                  │
           │                  │     │  • Interactive Analytics         │
           ▼                  │     │    (Apache Spark)                │
       Raspberry Pi           │     │  • Graph Analytics               │
       Android Phones  ───────┼────►│    (GoFFish)                     │
                              │     │  • Batch Analytics               │
                              │     │    (Hadoop)                      │
                              │     └──────────────────────────────────┘
                              ▼
                    ┌─────────────────┐     ┌───────────────┐
                    │ MESSAGE BROKER  │────►│  Data Archive │
                    │ (Apache Apollo) │     │   (HBase)     │
                    └─────────────────┘     └───────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Streaming Ingest│
                    │ (Apache Storm)  │
                    └─────────────────┘
```

**Key Insight**: A **Message Broker** sits at the center of the pipeline, decoupling data producers (sensors) from consumers (analytics engines).

---

## 2. Publish-Subscribe Systems

### 2.1 What is Publish-Subscribe?

A **Publish-Subscribe (Pub-Sub) System** is an **asynchronous, distributed, event-based communication paradigm**.

**Core Concepts**:
- **Event**: A data item representing an observation or state change, generated periodically or when some condition is met
- **Event Stream**: A series of events from the same logical source
- **Producers (Publishers)**: Generate and publish events
- **Consumers (Subscribers)**: Receive and process events
- **Topics**: Named logical channels that categorize event streams

### 2.2 Why Not Direct Communication?

Without a pub-sub system, producers and consumers must communicate directly:

```
Traditional Approach (Point-to-Point):

   Producer A ────────────────────► Consumer X
       │                                │
       └─────────────────────────► Consumer Y
   
   Producer B ────────────────────► Consumer X
       │                                │
       └─────────────────────────► Consumer Y

Problems:
  ✗ Each producer must know all consumers
  ✗ Adding new consumer requires updating all producers
  ✗ If consumer goes down, producer loses data or blocks
  ✗ Scaling is complex — N producers × M consumers = N×M connections
```

**Challenges** with direct communication:
1. **Many consumers interested in the same stream** — must duplicate data to each
2. **Producers and consumers don't know each other** — tight coupling is problematic
3. **Consumers may temporarily go offline** — no buffering mechanism
4. **Scaling** — adding producers or consumers is difficult

### 2.3 The Pub-Sub Solution: Broker-Mediated Communication

```
Publish-Subscribe Architecture:

   Publisher A ──┐
                 │     ┌──────────────┐
   Publisher B ──┼────►│              │     ┌─────────────┐
                 │     │    BROKER    │────►│ Consumer X  │
   Publisher C ──┼────►│              │     └─────────────┘
                 │     │  ┌────────┐  │
   Publisher D ──┘     │  │Topic 1 │  │     ┌─────────────┐
                       │  ├────────┤  │────►│ Consumer Y  │
        EVENTS ───────►│  │Topic 2 │  │     └─────────────┘
                       │  ├────────┤  │
                       │  │Topic 3 │  │     ┌─────────────┐
                       │  └────────┘  │────►│ Consumer Z  │
                       └──────────────┘     └─────────────┘

Benefits:
  ✓ Decoupled producers and consumers
  ✓ Multiple consumers can subscribe to same topic
  ✓ Persistence buffers data if consumers are slow/offline
  ✓ Horizontal scaling through partitioning
```

**Key Benefits**:
1. **Decoupling**: Producers and consumers are independent; neither needs to know about the other
2. **Scalability**: Broker can handle many producers and consumers
3. **Persistence**: Messages stored durably, allowing asynchronous consumption
4. **Replay**: Consumers can re-process historical data

---

## 3. Apache Kafka: Introduction

### 3.1 Origins at LinkedIn

**Kafka** was developed at **LinkedIn** (now open-source under Apache) to solve specific data infrastructure challenges:

**LinkedIn's Requirements**:
- **Collecting system and application metrics**: CPU usage, application performance
- **Request tracing**: Debugging distributed service calls
- **User activity tracking**: Pageviews, clicks, searches, social actions

**The Core Problem**: Different systems needed the same data:
- Real-time dashboards needed live metrics
- Search relevance needed user activity data
- Recommendation engines needed click streams
- Data warehouses needed everything for offline analysis

**Existing Solutions Were Inadequate**:
1. **Traditional enterprise messaging (JMS, ActiveMQ, RabbitMQ)**:
   - Rich delivery guarantees (often overkill for log data)
   - Low throughput due to per-message acknowledgments
   - Poor distributed support
   - Performance degrades when messages accumulate

2. **Log aggregators (Scribe, Flume)**:
   - Designed for offline consumption only
   - Expose implementation details (e.g., "minute files")
   - Use push model (can overwhelm slow consumers)

### 3.2 Kafka's Design Goals

Kafka was designed with specific goals:

| Goal | Description |
|------|-------------|
| **Decouple producers and consumers** | Push-pull model with broker in between |
| **Persistence** | Store messages durably within the messaging system |
| **High throughput** | Process hundreds of thousands of messages per second |
| **Horizontal scaling** | Scale out by adding more brokers as data grows |
| **Real-time + Batch** | Support both online consumers and offline data warehousing |

### 3.3 Kafka vs. Traditional Systems

```
Traditional Messaging (JMS)              Kafka
─────────────────────────                ──────
Per-message ACKs                         Offset-based tracking
Push model                               Pull model
Rich delivery guarantees                 At-least-once (simpler)
Complex routing                          Topic-partition based
In-memory queues                         Append-only logs on disk
Degrades when backlogged                 Constant performance regardless of data size
```

---

## 4. Kafka Core Concepts

### 4.1 Messages (Events)

A **message** is the fundamental unit of data in Kafka:

```
┌─────────────────────────────────────────────────────────┐
│                      MESSAGE                            │
├─────────────────────────────────────────────────────────┤
│  Key (optional)      │  Array of bytes                  │
├──────────────────────┼──────────────────────────────────┤
│  Value (payload)     │  Array of bytes (opaque to Kafka)│
├──────────────────────┼──────────────────────────────────┤
│  Timestamp           │  When message was produced       │
├──────────────────────┼──────────────────────────────────┤
│  Headers (optional)  │  Key-value metadata              │
└─────────────────────────────────────────────────────────┘
```

**Key Points**:
- Also called: **Event**, **Row**, **Tuple**, **Record**
- Payload is an **opaque array of bytes** — Kafka doesn't interpret it
- **Key** is optional but important for partitioning
- **Serialization** is the producer's responsibility (Avro, JSON, Protobuf, etc.)

### 4.2 Topics and Partitions

A **topic** is a **named logical channel** for related messages (like a database table or a mailbox).

```
                          TOPIC: user-activity
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Partition 0:  [msg0] [msg1] [msg2] [msg3] [msg4] ───► writes      │
│                                                                     │
│   Partition 1:  [msg0] [msg1] [msg2] [msg3] ───────────► writes     │
│                                                                     │
│   Partition 2:  [msg0] [msg1] ─────────────────────────► writes     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

                              ▲
                              │
                    Messages assigned to
                    partitions based on key
```

**Topics**:
- Logical categorization of messages
- Like a **table** in a database or a **PO Box** for mail

**Partitions**:
- Topics are **sharded into partitions** for parallelism
- Each partition is an **ordered, immutable, append-only log**
- **Ordering is guaranteed within a partition**, NOT across partitions
- Each message within a partition has a unique sequential **offset**

**Why Partitions?**
1. **Parallelism**: Multiple producers/consumers can work on different partitions simultaneously
2. **Scalability**: Partitions can be distributed across multiple brokers
3. **Throughput**: More partitions = higher potential throughput

### 4.3 Offsets

An **offset** is a unique sequential ID for each message within a partition:

```
    Partition 0:
    ┌──────┬──────┬──────┬──────┬──────┬──────┐
    │ 0    │ 1    │ 2    │ 3    │ 4    │ 5    │ ← Offsets
    │ msg  │ msg  │ msg  │ msg  │ msg  │ msg  │
    └──────┴──────┴──────┴──────┴──────┴──────┘
      ▲                           ▲        ▲
      │                           │        │
    Oldest                     Consumer   Newest
    (deletable)                position   (producer writes here)
```

**Key Properties**:
- **Not explicit message IDs** — derived from position in log
- **Not consecutive** — to get next offset, add current message length to current offset
- Consumers track their **current offset** to know where to resume
- Offsets are **immutable** — never reused even after message deletion

### 4.4 Batches

A **batch** is a collection of messages destined for the same partition:

```
Producer batching:

    msg1 ─┐
    msg2 ─┼─► [BATCH] ──────────────────────► Partition 0
    msg3 ─┘         (same partition)
    
    msg4 ─┐
    msg5 ─┼─► [BATCH] ──────────────────────► Partition 1
    msg6 ─┘         (same partition)
```

**Trade-off**:
- **Larger batches** = Higher **throughput** (amortized network overhead)
- **Larger batches** = Higher **latency** (wait for batch to fill)

**Why Batching Matters** (from paper):
> "Batching greatly improved the throughput by amortizing the RPC overhead. In Kafka, a batch size of 50 messages improved the throughput by almost an order of magnitude."

---

## 5. Producers

### 5.1 Producer Overview

**Producers** are clients that **publish (write) messages to Kafka topics**.

```
                  ┌───────────────────────────────────────┐
                  │            PRODUCER                    │
                  │                                        │
    Message ─────►│  1. Serialize message/key              │
    + Topic       │                                        │
    + Key*        │  2. Determine partition (via key)      │
                  │                                        │
                  │  3. Batch messages for partition       │
                  │                                        │
                  │  4. Send batch to broker               │
                  │                                        │
                  └───────────────────────────────────────┘
                                    │
                                    ▼
                            ┌───────────────┐
                            │ Kafka Broker  │
                            └───────────────┘
```

**Producer Workflow**:

1. **Create message with topic and optional key**
2. **Serialize** the message and key to byte arrays
3. **Partition selection**:
   - If key is provided: `hash(key) % num_partitions`
   - If no key: Round-robin or random assignment
4. **Batch** messages destined for the same partition
5. **Send** batches to the appropriate Kafka broker

### 5.2 Sample Producer Code (from paper)

```java
// Create a producer
producer = new Producer(…);

// Create a message (payload is bytes)
message = new Message("test message str".getBytes());

// Create a batch (MessageSet)
set = new MessageSet(message);

// Send to topic
producer.send("topic1", set);
```

**Key Points**:
- Messages are **byte arrays** — producer chooses serialization
- Multiple messages can be sent in a **single publish request**
- Topic specified at send time

### 5.3 Partitioning Strategies

```
Partitioning Decision:

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Message with Key?                                                 │
│         │                                                           │
│         ├── YES ──► hash(key) % num_partitions                      │
│         │           All messages with same key go to same partition │
│         │           (preserves ordering for that key)               │
│         │                                                           │
│         └── NO  ──► Round-robin or random partition                 │
│                     (distributes load evenly)                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Example**: 
- User activity events keyed by `user_id`
- All events for user "alice" go to same partition → ordered processing
- Different users spread across partitions → parallel processing

### 5.4 Sync vs. Async Sending

**Synchronous Send**:
```
Producer ──► send() ──► wait for ACK ──► send next message
                            │
                    (blocks until broker confirms)
```
- Waits for acknowledgment from Kafka before sending next message
- Returns: topic, partition, and offset of written record
- **Lower throughput**, but **guaranteed delivery confirmation**

**Asynchronous Send**:
```
Producer ──► send() ──► send() ──► send() ──► ...
                │          │          │
                └──────────┴──────────┴──► ACKs arrive later
```
- Does **not wait** for broker reply
- **Higher throughput** (especially important for log aggregation)
- Optional **callback** to handle errors asynchronously

**From the paper**:
> "The Kafka producer currently doesn't wait for acknowledgements from the broker and sends messages as fast as the broker can handle. This significantly increased the throughput of the publisher. With a batch size of 50, a single Kafka producer almost saturated the 1Gb link between the producer and the broker."

**Trade-off**: For log aggregation, losing occasional messages is acceptable → async preferred. For critical data, sync or async with callbacks is needed.

---

## 6. Consumers

### 6.1 Consumer Overview

**Consumers** are clients that **subscribe to topics and read (consume) messages**.

```
              ┌───────────────────────────────────────────┐
              │            CONSUMER                        │
              │                                            │
              │  1. Subscribe to topic(s)                  │
              │                                            │
              │  2. Poll broker for messages               │
              │     (includes offset in request)           │
              │                                            │
              │  3. Process messages                       │
              │                                            │
              │  4. Commit offset (tell broker "I'm done") │
              │                                            │
              └───────────────────────────────────────────┘
                                    ▲
                                    │ Pull
                                    ▼
                            ┌───────────────┐
                            │ Kafka Broker  │
                            └───────────────┘
```

### 6.2 Pull vs. Push Model

**Kafka uses the Pull Model** — consumers request data from brokers.

```
Push Model (Traditional):                Pull Model (Kafka):
─────────────────────────                ─────────────────────
Broker ──push──► Consumer                Consumer ──pull──► Broker

Problems:                                Benefits:
• Broker controls rate                   • Consumer controls rate
• Can overwhelm slow consumers           • Can process at max sustainable rate
• Hard to rewind                         • Easy to rewind (just change offset)
• Consumer can be flooded                • Natural backpressure
```

**From the paper**:
> "At LinkedIn, we find the pull model more suitable for our applications since each consumer can retrieve the messages at the maximum rate it can sustain and avoid being flooded by messages pushed faster than it can handle. The pull model also makes it easy to rewind a consumer."

### 6.3 Sample Consumer Code (from paper)

```java
// Subscribe to topic with 1 message stream
streams[] = Consumer.createMessageStreams("topic1", 1);

// Iterate over messages (blocks if no messages)
for (message : streams[0]) {
    bytes = message.payload();
    // Process the message bytes
}

// Note: Iterator NEVER terminates — blocks waiting for new messages
```

**Key Points**:
- Consumer creates **message streams** for subscribed topics
- Iterator **blocks** when no messages available (doesn't return empty)
- Consumer processes messages **sequentially within a partition**

### 6.4 Consumer Groups

A **Consumer Group** is a set of consumers that **cooperatively consume messages from a topic**.

```
                        TOPIC: user-clicks
    ┌────────────────────────────────────────────────────────┐
    │                                                        │
    │  Partition 0  ───────────────────────────────┐         │
    │                                              │         │
    │  Partition 1  ───────────────────────────────┤         │
    │                                              │         │
    │  Partition 2  ───────────────────────────────┤         │
    │                                              │         │
    │  Partition 3  ───────────────────────────────┘         │
    │                                                        │
    └────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
    ┌─────────────────────┐         ┌─────────────────────┐
    │  Consumer Group A   │         │  Consumer Group B   │
    │  (Analytics Team)   │         │  (ML Pipeline)      │
    ├─────────────────────┤         ├─────────────────────┤
    │ Consumer 1: P0, P1  │         │ Consumer 1: P0, P1  │
    │ Consumer 2: P2, P3  │         │ Consumer 2: P2, P3  │
    └─────────────────────┘         └─────────────────────┘
    
    • Each message delivered to ONE consumer per group
    • Different groups get INDEPENDENT copies of ALL messages
```

**Key Rules**:
1. **Each partition consumed by exactly ONE consumer** within a group
2. **A consumer can consume multiple partitions**
3. **More consumers than partitions** → some consumers sit **idle**
4. **Different consumer groups** are completely **independent** — no coordination needed between groups

**Why Consumer Groups?**
1. **Horizontal scaling**: Add more consumers to process partitions in parallel
2. **Fault tolerance**: If a consumer dies, its partitions are reassigned
3. **Load balancing**: Work distributed across consumers
4. **Multiple use cases**: Different groups can process same data independently

### 6.5 Partition Assignment Strategies

When a consumer group has multiple consumers, partitions must be assigned. Two strategies:

**Range Assignment**:
```
Topics: T1 (P0, P1, P2), T2 (P0, P1, P2)
Consumers: C1, C2

Assignment:
  C1: T1-P0, T1-P1, T2-P0, T2-P1
  C2: T1-P2, T2-P2

(Range of partitions from each topic assigned to each consumer)
```

**Round-Robin Assignment**:
```
Topics: T1 (P0, P1, P2), T2 (P0, P1, P2)
Consumers: C1, C2

Assignment:
  C1: T1-P0, T1-P2, T2-P1
  C2: T1-P1, T2-P0, T2-P2

(Each consumer gets roughly equal number of partitions overall)
```

### 6.6 Offset Management and Commits

Consumers must **track their progress** — which messages have been processed.

```
Offset Tracking:

Partition:  [0] [1] [2] [3] [4] [5] [6] [7] [8] [9]
                              ▲           ▲
                              │           │
                    Last Committed     Current
                       Offset         Position
                         (4)            (7)
                              
Messages 0-4: Committed (won't be reprocessed on restart)
Messages 5-7: Processed but not committed (will be reprocessed on crash)
Messages 8-9: Not yet fetched
```

**Commit**: Tells the broker "I have successfully processed messages up to this offset"

**Sync Commit**:
```java
consumer.commitSync();  // Blocks until broker confirms
```
- Blocks until broker acknowledges
- Safer but slower

**Async Commit**:
```java
consumer.commitAsync(callback);  // Returns immediately
```
- Non-blocking
- Optional callback for error handling
- Faster but may lose commits on failure

**Commit Frequency Trade-off**:
- **Commit every message**: Safest, but slow (many round-trips)
- **Commit periodically**: Faster, but may reprocess some messages on crash
- **Commit at batch boundaries**: Good balance

### 6.7 Rebalancing

**Rebalancing** occurs when the partition-to-consumer mapping must change:

**Triggers**:
- A consumer **joins** the group
- A consumer **leaves** the group (crashes or shuts down)
- A new **partition is added** to the topic
- **Heartbeat timeout** — consumer hasn't polled or committed

**Rebalance Process**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      REBALANCE PROCESS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Group Coordinator (a broker) detects change                     │
│                                                                     │
│  2. All consumers notified → stop processing                        │
│                                                                     │
│  3. Consumer Group Leader computes new assignment                   │
│     (using Range or Round-Robin strategy)                           │
│                                                                     │
│  4. New assignments distributed to all consumers                    │
│                                                                     │
│  5. Consumers resume processing their assigned partitions           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Important**: During rebalance, **no messages are consumed** — brief pause in processing.

**Heartbeats**:
- Polling and commit operations serve as **heartbeats** to the coordinator
- If a consumer fails to send heartbeats (e.g., stuck in long processing), coordinator assumes it's dead → triggers rebalance
- Use **rebalance listener** to commit pending work before rebalance

### 6.8 Standalone Consumer

A consumer can operate **outside a consumer group** by manually assigning partitions:

```java
// Manually assign specific partitions
consumer.assign(Arrays.asList(
    new TopicPartition("topic1", 0),
    new TopicPartition("topic1", 2)
));

// Must manually check for new partitions
// No automatic rebalancing
```

**Use Cases**:
- Single consumer that needs specific partitions
- When automatic rebalancing is undesirable
- When consumer knows exactly which partitions it needs

---

## 7. Brokers and Clusters

### 7.1 What is a Broker?

A **broker** is a single Kafka server that:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KAFKA BROKER                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  From Producers:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Receives messages from producers                           │   │
│  │ • Assigns offsets to messages within partitions              │   │
│  │ • Commits messages to disk storage                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  To Consumers:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Responds to fetch requests for partitions                  │   │
│  │ • Sends committed messages to consumers                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Storage:                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ • Stores partition data on disk                              │   │
│  │ • Maintains in-memory offset index                           │   │
│  │ • Handles message retention/deletion                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Message Retention

Kafka provides **durable storage** of messages with configurable retention:

```
Retention Policy:

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   [Old messages]  ←── DELETE ──  [Retention Window]  ←── NEW       │
│                                                                     │
│   Configured by:                                                    │
│   • Time: e.g., retain for 7 days (log.retention.hours)            │
│   • Size: e.g., retain up to 1 GB (log.retention.bytes)            │
│   • Compaction: Keep latest value per key                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**From the paper**:
> "Kafka solves this problem by using a simple time-based SLA for the retention policy. A message is automatically deleted if it has been retained in the broker longer than a certain period, typically 7 days. This solution works well in practice."

**Key Insight**: Because Kafka's performance doesn't degrade with data size, long retention (days/weeks) is feasible.

### 7.3 Kafka Cluster

A **Kafka cluster** consists of multiple brokers for scalability and fault tolerance:

```
                        KAFKA CLUSTER
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │    BROKER 1     │  │    BROKER 2     │  │    BROKER 3     │     │
│  │                 │  │                 │  │                 │     │
│  │ topic1/P0 (L)   │  │ topic1/P0 (F)   │  │ topic1/P0 (F)   │     │
│  │ topic1/P1 (F)   │  │ topic1/P1 (L)   │  │ topic1/P1 (F)   │     │
│  │ topic2/P0 (F)   │  │ topic2/P0 (F)   │  │ topic2/P0 (L)   │     │
│  │                 │  │                 │  │                 │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
│  (L) = Leader replica    (F) = Follower replica                     │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      ZooKeeper                               │   │
│  │  (Cluster coordination, leader election, configuration)      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Concepts**:
- Each partition has one **leader** and multiple **followers**
- All produce/consume requests go through the **leader**
- Followers **replicate** data for fault tolerance
- If leader fails, a follower is **promoted** to leader

### 7.4 Leaders and Followers

**Leaders**:
- Each partition has a **single replica designated as leader**
- **All produce and consume requests** go through the leader
- Ensures **consistency** — single point of serialization

**Followers**:
- **Replicate messages from the leader**
- Stay **in-sync** with the most recent messages
- Can be **promoted to leader** if current leader fails

**In-Sync Replicas (ISR)**:
```
Leader tracks sync status of each follower:

  Leader (offset 100)
      │
      ├── Follower 1: offset 98   → In-sync (within threshold)
      │
      ├── Follower 2: offset 100  → In-sync (fully caught up)
      │
      └── Follower 3: offset 85   → Out-of-sync (>10 sec behind)

Only in-sync replicas can be promoted to leader.
```

**From slides**:
> "Leader keeps track of sync status of each follower based on offset requests from follower. In sync are up to date, can be promoted as leader. Out of sync are <10 secs old."

---

## 8. Kafka Internals

### 8.1 ZooKeeper Coordination

Kafka relies on **Apache ZooKeeper** for distributed coordination:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ZooKeeper Functions                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  • Maintaining configuration information                            │
│  • Naming (hierarchical namespace like a file system)               │
│  • Distributed synchronization (barriers, mutexes)                  │
│  • Group services                                                   │
│  • In-memory operations (fast reads)                                │
│  • Consensus-based writes (strong consistency)                      │
│  • Total global ordering of operations                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**ZooKeeper API**:
- **Create path**: `/brokers/ids/1`
- **Set value of path**: Write configuration data
- **Read value of path**: Get configuration
- **Delete path**: Remove node
- **List children**: See all items under a path
- **Watchers**: Get notified when path changes

**Special Features**:
- **Ephemeral nodes**: Automatically deleted when creating client disconnects
- **Persistent nodes**: Remain until explicitly deleted
- **Replicated**: Data replicated across multiple ZooKeeper servers

### 8.2 Cluster Membership

**Broker Registration**:
```
When broker starts:
  1. Creates ephemeral node at /brokers/ids/<broker-id>
  2. Node contains: hostname, port, topics, partitions
  3. If broker crashes → ephemeral node automatically deleted
  4. Other brokers/consumers watching this path get notified

/brokers
  └── ids
       ├── 1  (ephemeral) → {host: broker1.example.com, port: 9092, ...}
       ├── 2  (ephemeral) → {host: broker2.example.com, port: 9092, ...}
       └── 3  (ephemeral) → {host: broker3.example.com, port: 9092, ...}
```

**Controller Election**:
```
First broker to register at /controller becomes the Controller:

  1. Broker 1 starts → creates /controller → Broker 1 is Controller
  2. Broker 2 starts → /controller exists → registers watch, waits
  3. Broker 1 fails → /controller deleted (ephemeral)
  4. Broker 2 notified → creates /controller → Broker 2 is Controller
  
Controller Epoch: Increments with each new controller
  • Prevents stale operations from old controller
  • Messages include epoch; receivers ignore stale epochs
```

**Controller Responsibilities**:
- Monitor broker failures via ZooKeeper watches
- **Elect new partition leaders** when brokers fail
- Inform all brokers of leader changes

### 8.3 Consumer Coordination

**Consumer Group State in ZooKeeper**:

```
/consumers
  └── <group-id>
       ├── ids               (Consumer Registry - ephemeral)
       │    ├── consumer-1   → {subscribed topics}
       │    └── consumer-2   → {subscribed topics}
       │
       ├── owners            (Ownership Registry - ephemeral)
       │    └── <topic>
       │         ├── 0       → consumer-1
       │         ├── 1       → consumer-2
       │         └── 2       → consumer-1
       │
       └── offsets           (Offset Registry - persistent)
            └── <topic>
                 ├── 0       → 12345
                 ├── 1       → 23456
                 └── 2       → 34567
```

**Rebalance Algorithm** (from paper):

```
Algorithm: Rebalance process for consumer Ci in group G

For each topic T that Ci subscribes to:
  1. Remove partitions owned by Ci from ownership registry
  
  2. Read broker and consumer registries from ZooKeeper
  
  3. Compute PT = partitions available in all brokers under topic T
  
  4. Compute CT = all consumers in G that subscribe to topic T
  
  5. Sort PT and CT
  
  6. Let j = index position of Ci in CT
     Let N = |PT| / |CT|
     
  7. Assign partitions from j*N to (j+1)*N - 1 in PT to consumer Ci
  
  8. For each assigned partition p:
     - Set owner of p to Ci in ownership registry
     - Let Op = offset of partition p in offset registry
     - Start thread to pull data from partition p starting at offset Op
```

**Handling Concurrent Rebalances**:
> "When there are multiple consumers within a group, each of them will be notified of a broker or consumer change. However, the notification may come at slightly different times. So, it is possible that one consumer tries to take ownership of a partition still owned by another consumer. When this happens, the first consumer simply releases all partitions, waits a bit, and retries. In practice, the rebalance process often stabilizes after only a few retries."

---

## 9. Efficient Data Transfer

### 9.1 Simple Storage Layout

Kafka uses a **simple, append-only log structure**:

```
                    PARTITION STORAGE
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Logical Log (Partition 0):                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ [msg] [msg] [msg] [msg] ... [msg] [msg] [msg]               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Physical Implementation (Segment Files):                           │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │ Segment 0   │  │ Segment 1   │  │ Segment 2   │   (~1GB each)   │
│  │ 00000000000 │  │ 00014517018 │  │ 00030706778 │                 │
│  │             │  │             │  │             │                 │
│  │ [msg 0]     │  │ [msg N]     │  │ [msg M]     │                 │
│  │ [msg 1]     │  │ [msg N+1]   │  │ [msg M+1]   │                 │
│  │ ...         │  │ ...         │  │ ...         │   ◄── Writes    │
│  └─────────────┘  └─────────────┘  └─────────────┘       (append)  │
│        │                │                │                         │
│        └────────────────┼────────────────┘                         │
│                         ▼                                          │
│            In-Memory Offset Index:                                 │
│            [0, 14517018, 30706778, ...]                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Design Decisions**:

1. **Append-only writes**: Messages always appended to end of log
2. **Segment files**: Log split into ~1GB segment files
3. **No explicit message IDs**: Offset = position in log
4. **In-memory index**: Maps first offset of each segment for fast lookup

**From the paper**:
> "Unlike typical messaging systems, a message stored in Kafka doesn't have an explicit message id. Instead, each message is addressed by its logical offset in the log. This avoids the overhead of maintaining auxiliary, seek-intensive random-access index structures."

### 9.2 Efficient Storage Format

**Kafka's storage overhead is minimal** — only 9 bytes per message:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MESSAGE FORMAT                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────┬────────────┬────────────┬──────────────────────┐   │
│  │  CRC (4B)  │  Magic (1B)│  Attrs (1B)│     Payload (var)    │   │
│  └────────────┴────────────┴────────────┴──────────────────────┘   │
│                                                                     │
│  + Key length, Value length, Timestamp = ~9 bytes overhead          │
│                                                                     │
│  vs. ActiveMQ: ~144 bytes overhead (JMS headers + indexing)         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**From the paper**:
> "On average, each message had an overhead of 9 bytes in Kafka, versus 144 bytes in ActiveMQ. This means that ActiveMQ was using 70% more space than Kafka to store the same set of 10 million messages."

### 9.3 Page Cache Instead of Application Cache

**Conventional approach**: Application maintains in-memory cache of messages

**Kafka's approach**: Rely on **OS page cache** instead

```
Conventional (ActiveMQ):                 Kafka:
────────────────────────                 ──────

┌──────────────────────┐                 ┌──────────────────────┐
│    Application       │                 │    Application       │
│  ┌────────────────┐  │                 │  (no message cache)  │
│  │ Message Cache  │  │                 └──────────┬───────────┘
│  │  (in heap)     │  │                            │
│  └────────────────┘  │                            ▼
└──────────┬───────────┘                 ┌──────────────────────┐
           │                             │   OS Page Cache      │
           ▼                             │  (managed by kernel) │
┌──────────────────────┐                 └──────────┬───────────┘
│   OS Page Cache      │                            │
└──────────┬───────────┘                            ▼
           │                             ┌──────────────────────┐
           ▼                             │       Disk           │
┌──────────────────────┐                 └──────────────────────┘
│       Disk           │
└──────────────────────┘

Problems:                                Benefits:
• Double buffering                       • No double buffering
• GC overhead for large heap             • Minimal GC (no in-heap cache)
• Cache lost on restart                  • Cache survives restart
• Complex cache invalidation             • OS handles caching efficiently
```

**From the paper**:
> "We rely on the underlying file system page cache. This has the main benefit of avoiding double buffering — messages are only cached in the page cache. This has the additional benefit of retaining warm cache even when a broker process is restarted."

### 9.4 Zero-Copy Transfer (sendfile)

**Conventional data transfer** (file to network):

```
Step 1: read() system call
  Disk → Page Cache → Application Buffer

Step 2: write() system call  
  Application Buffer → Socket Buffer → Network

Total: 4 data copies, 2 system calls
```

**Kafka's zero-copy** using `sendfile()`:

```
sendfile() system call:
  Disk → Page Cache → Network (via DMA)

Total: 2 data copies, 1 system call
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                ZERO-COPY DATA TRANSFER                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Traditional:                                                       │
│  ┌──────┐    ┌────────────┐    ┌─────────┐    ┌────────┐           │
│  │ Disk │───►│ Page Cache │───►│ App Buf │───►│ Socket │───► Net   │
│  └──────┘    └────────────┘    └─────────┘    └────────┘           │
│      1             2                3              4                │
│                                                                     │
│  Zero-Copy (sendfile):                                              │
│  ┌──────┐    ┌────────────┐                                        │
│  │ Disk │───►│ Page Cache │─────────────────────────────────► Net  │
│  └──────┘    └────────────┘                                        │
│      1             2         (DMA transfer, bypasses user space)   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**From the paper**:
> "On Linux and other Unix operating systems, there exists a sendfile API that can directly transfer bytes from a file channel to a socket channel. This typically avoids 2 of the copies and 1 system call. Kafka exploits the sendfile API to efficiently deliver bytes in a log segment file from a broker to a consumer."

### 9.5 Stateless Broker Design

**Conventional messaging systems**: Broker tracks what each consumer has consumed

**Kafka's design**: **Consumer tracks its own offset**

```
Traditional (Stateful Broker):           Kafka (Stateless Broker):
───────────────────────────              ─────────────────────────

Broker maintains:                        Consumer maintains:
• Per-consumer ACK state                 • Current offset
• Message delivery status                • Commit progress
• Complex bookkeeping                    

Delete messages when:                    Delete messages when:
• All consumers have ACK'd               • Retention period expires
  (must track all consumers)               (simple time-based rule)

Problems:                                Benefits:
• Complex coordination                   • Simple broker design
• Broker must know consumers             • No consumer tracking overhead
• Hard to delete messages                • Easy message deletion
• Can't rewind                           • Can rewind (just change offset)
```

**From the paper**:
> "In Kafka, the information about how much each consumer has consumed is not maintained by the broker, but by the consumer itself. Such a design reduces a lot of the complexity and the overhead on the broker."

**Rewind Capability**:
> "A consumer can deliberately rewind back to an old offset and re-consume data. This violates the common contract of a queue, but proves to be an essential feature for many consumers. For example, when there is an error in application logic in the consumer, the application can re-play certain messages after the error is fixed."

---

## 10. Delivery Guarantees

### 10.1 At-Least-Once Delivery

**Kafka guarantees at-least-once delivery** (not exactly-once in the original design):

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DELIVERY SEMANTICS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  At-Most-Once:    May lose messages, never duplicates               │
│                   (fire and forget)                                 │
│                                                                     │
│  At-Least-Once:   Never lose messages, may have duplicates     ◄── │
│                   (Kafka's guarantee)                               │
│                                                                     │
│  Exactly-Once:    Never lose, never duplicate                       │
│                   (requires 2PC, expensive)                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**From the paper**:
> "In general, Kafka only guarantees at-least-once delivery. Exactly-once delivery typically requires two-phase commits and is not necessary for our applications."

### 10.2 When Duplicates Occur

```
Scenario: Consumer crash before commit

1. Consumer fetches messages [5, 6, 7, 8, 9]
2. Consumer processes messages 5, 6, 7
3. Consumer CRASHES before committing offset 7
4. New consumer takes over
5. New consumer reads last committed offset = 4
6. New consumer re-fetches messages [5, 6, 7, 8, 9]
7. Messages 5, 6, 7 processed AGAIN (duplicates)
```

**Handling Duplicates**:
> "If an application cares about duplicates, it must add its own de-duplication logic, either using the offsets that we return to the consumer or some unique key within the message. This is usually a more cost-effective approach than using two-phase commits."

### 10.3 Ordering Guarantees

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ORDERING GUARANTEES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Within a partition:   Messages ALWAYS in order                     │
│                        (sequential offsets, single consumer)        │
│                                                                     │
│  Across partitions:    NO ordering guarantee                        │
│                        (different partitions, different consumers)  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Implication**: If you need strict ordering for related messages, **use the same key** so they go to the same partition.

### 10.4 Data Integrity

**CRC Checksums**:
```
Each message includes a CRC (Cyclic Redundancy Check):

┌──────────────┬────────────────────────────────────────┐
│  CRC (4B)    │           Message Content              │
└──────────────┴────────────────────────────────────────┘
       │
       └── Computed at write time, verified at read time
           Detects I/O errors and network corruption
```

**From the paper**:
> "To avoid log corruption, Kafka stores a CRC for each message in the log. If there is any I/O error on the broker, Kafka runs a recovery process to remove those messages with inconsistent CRCs."

### 10.5 Durability (Replication)

**Original Kafka (paper)**: No built-in replication

> "If a broker goes down, any message stored on it not yet consumed becomes unavailable. If the storage system on a broker is permanently damaged, any unconsumed message is lost forever. In the future, we plan to add built-in replication in Kafka to redundantly store each message on multiple brokers."

**Modern Kafka**: Full replication support added

```
Replication Factor = 3:

  Broker 1:  topic1-P0 (Leader)
  Broker 2:  topic1-P0 (Follower)
  Broker 3:  topic1-P0 (Follower)

If Broker 1 fails:
  → Broker 2 or 3 promoted to leader
  → No data loss
```

---

## 11. Performance

### 11.1 Producer Performance

**Experimental Setup** (from paper):
- 2 Linux machines (8 cores, 16GB RAM, 6 disks RAID 10)
- 1 Gb network link
- 10 million messages, 200 bytes each

```
┌─────────────────────────────────────────────────────────────────────┐
│                  PRODUCER THROUGHPUT COMPARISON                     │
├────────────────────────────┬────────────────────────────────────────┤
│ System                     │ Messages/Second                        │
├────────────────────────────┼────────────────────────────────────────┤
│ Kafka (batch=1)            │ ~50,000                                │
│ Kafka (batch=50)           │ ~400,000                               │
│ RabbitMQ                   │ ~25,000                                │
│ ActiveMQ                   │ ~5,000                                 │
└────────────────────────────┴────────────────────────────────────────┘
```

**Why Kafka is Faster**:

1. **No ACK wait**: Producer doesn't wait for broker acknowledgment (fire-and-forget for logs)
2. **Efficient format**: 9 bytes overhead vs. 144 bytes in ActiveMQ (70% less space)
3. **Batching**: Batch of 50 = 8x throughput improvement (amortized RPC overhead)

### 11.2 Consumer Performance

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CONSUMER THROUGHPUT COMPARISON                     │
├────────────────────────────┬────────────────────────────────────────┤
│ System                     │ Messages/Second                        │
├────────────────────────────┼────────────────────────────────────────┤
│ Kafka                      │ ~22,000                                │
│ RabbitMQ                   │ ~5,000                                 │
│ ActiveMQ                   │ ~4,000                                 │
└────────────────────────────┴────────────────────────────────────────┘
```

**Why Kafka Consumer is Faster**:

1. **Efficient storage format**: Fewer bytes transferred
2. **No per-message ACK**: Batch offset commits
3. **sendfile (zero-copy)**: Direct disk-to-network transfer
4. **No complex delivery tracking**: Broker doesn't track consumer state

### 11.3 Performance Characteristics

```
┌─────────────────────────────────────────────────────────────────────┐
│                  KAFKA PERFORMANCE PROPERTIES                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Linear scalability with data size (up to many terabytes)        │
│                                                                     │
│  ✓ Constant performance regardless of unconsumed message backlog    │
│                                                                     │
│  ✓ Sequential I/O (append-only writes, sequential reads)            │
│                                                                     │
│  ✓ OS page cache efficient (producer/consumer access sequential)    │
│                                                                     │
│  ✓ Minimal GC pressure (no in-heap message caching)                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. Kafka Deployment at LinkedIn

### 12.1 Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         LINKEDIN KAFKA DEPLOYMENT                            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MAIN DATACENTER                              ANALYSIS DATACENTER            │
│  ┌────────────────────────────┐               ┌────────────────────────────┐ │
│  │                            │               │                            │ │
│  │  ┌──────────┐ ┌──────────┐ │               │  ┌──────────────────────┐  │ │
│  │  │ Frontend │ │ Frontend │ │               │  │ Embedded Consumers   │  │ │
│  │  │ Service  │ │ Service  │ │               │  │ (pull from live DC)  │  │ │
│  │  └────┬─────┘ └────┬─────┘ │               │  └──────────┬───────────┘  │ │
│  │       │             │      │               │             │              │ │
│  │       ▼             ▼      │               │             ▼              │ │
│  │  ┌────────────────────────┐│               │  ┌──────────────────────┐  │ │
│  │  │    Load Balancer       ││               │  │   Kafka Cluster      │  │ │
│  │  └──────────┬─────────────┘│               │  │   (replica)          │  │ │
│  │             │              │               │  └──────────┬───────────┘  │ │
│  │             ▼              │               │             │              │ │
│  │  ┌────────────────────────┐│               │             ▼              │ │
│  │  │   Kafka Brokers        ││──────────────►│  ┌──────────────────────┐  │ │
│  │  └────────────────────────┘│               │  │  Hadoop / DWH        │  │ │
│  │             │              │               │  │  (offline analysis)  │  │ │
│  │             ▼              │               │  └──────────────────────┘  │ │
│  │  ┌────────────────────────┐│               │                            │ │
│  │  │ Real-time Services     ││               │                            │ │
│  │  │ (same DC consumers)    ││               │                            │ │
│  │  └────────────────────────┘│               │                            │ │
│  │                            │               │                            │ │
│  └────────────────────────────┘               └────────────────────────────┘ │
│                                                                              │
│  End-to-end latency: ~10 seconds average                                     │
│  Volume: Hundreds of GB and ~1 billion messages per day (2011)               │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 12.2 Use Cases

1. **Real-time services**: Consume events with sub-second latency (same datacenter)
2. **Offline analytics**: Load data into Hadoop for batch processing
3. **Cross-datacenter replication**: Pull from live DC to analysis DC
4. **Ad-hoc querying**: Run simple scripts against raw event streams

### 12.3 Data Loading into Hadoop

**Kafka MapReduce InputFormat**:
```
MapReduce Job
    │
    ├── Task 1: Read from Kafka partition 0
    ├── Task 2: Read from Kafka partition 1
    └── Task 3: Read from Kafka partition 2
    
    • Tasks can fail and restart (stateless broker helps)
    • Offsets stored in HDFS on successful completion
    • No message loss or duplication
```

### 12.4 Serialization with Avro

**LinkedIn uses Apache Avro** for message serialization:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AVRO SERIALIZATION                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Message Payload:                                                   │
│  ┌──────────────────┬───────────────────────────────────────────┐  │
│  │  Schema ID (4B)  │         Serialized Avro Data              │  │
│  └──────────────────┴───────────────────────────────────────────┘  │
│                                                                     │
│  Schema Registry Service:                                           │
│  • Maps Schema ID → Actual Avro Schema                              │
│  • Schemas are immutable (same ID always = same schema)             │
│  • Consumer fetches schema once per ID, caches it                   │
│  • Enables schema evolution (add fields, etc.)                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.5 Auditing and Monitoring

**End-to-end validation**:
```
Producer:
  1. Periodically generates monitoring events
  2. Records: topic, count of messages, time window
  3. Publishes monitoring events to special topic

Consumer:
  1. Counts received messages per topic
  2. Compares with monitoring events
  3. Validates no data loss in pipeline
```

---

## 13. Summary: Key Design Decisions

### 13.1 Design Philosophy

| Aspect | Traditional Messaging | Kafka |
|--------|----------------------|-------|
| **Delivery model** | Push | Pull |
| **State location** | Broker tracks consumption | Consumer tracks offset |
| **Message ID** | Explicit unique ID | Implicit offset |
| **Caching** | Application-level cache | OS page cache |
| **Network transfer** | Copy through user space | Zero-copy (sendfile) |
| **Storage** | Complex indexes | Append-only log |
| **Acknowledgment** | Per-message ACK | Batch offset commit |
| **Retention** | Until consumed | Time/size based |
| **Delivery guarantee** | Often exactly-once | At-least-once |

### 13.2 Trade-offs Made

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KAFKA'S DESIGN TRADE-OFFS                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ High throughput      vs.  ✗ Rich delivery guarantees            │
│    (async, batching)           (simplified to at-least-once)        │
│                                                                     │
│  ✓ Scalability          vs.  ✗ Complex routing                     │
│    (partitions, brokers)       (simple topic-partition model)       │
│                                                                     │
│  ✓ Simplicity           vs.  ✗ Features                            │
│    (stateless broker)          (consumers handle offset tracking)   │
│                                                                     │
│  ✓ Performance          vs.  ✗ Ordering                            │
│    (parallel partitions)       (order only within partition)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.3 When to Use Kafka

**Good Use Cases**:
- Log aggregation and processing
- Event streaming pipelines
- Metrics collection
- Activity tracking
- Stream processing input
- Commit log for distributed systems
- Message bus for microservices

**May Not Be Ideal For**:
- Transactional messaging requiring exactly-once
- Very low-latency messaging (sub-millisecond)
- Complex routing/filtering at broker level
- Small deployments (ZooKeeper overhead)

---

## 14. Key Takeaways

1. **Pub-Sub decouples producers and consumers** — enables scalability, persistence, and replay

2. **Topics are partitioned** — enabling parallel production and consumption

3. **Consumers pull, not push** — consumer controls rate, easy rewind

4. **Consumer groups enable scaling** — partitions distributed among group members

5. **Offsets replace message IDs** — simpler storage, sequential access

6. **Stateless broker** — consumer tracks progress, broker just stores logs

7. **OS page cache > application cache** — no double buffering, survives restarts

8. **Zero-copy transfer** — sendfile API eliminates unnecessary data copies

9. **Time-based retention** — simple deletion policy, handles slow consumers

10. **At-least-once delivery** — simple, high-throughput; apps handle deduplication

---

## 15. Glossary

| Term | Definition |
|------|------------|
| **Message/Event** | Unit of data in Kafka; array of bytes with optional key |
| **Topic** | Named category for messages; like a database table |
| **Partition** | Ordered, immutable sequence of messages; unit of parallelism |
| **Offset** | Sequential ID of message within partition |
| **Producer** | Client that publishes messages to topics |
| **Consumer** | Client that reads messages from topics |
| **Consumer Group** | Set of consumers that cooperatively consume a topic |
| **Broker** | Single Kafka server; stores partitions, serves clients |
| **Cluster** | Multiple brokers working together |
| **Leader** | Broker responsible for a partition's reads and writes |
| **Follower** | Replica that copies data from leader |
| **ISR (In-Sync Replica)** | Follower caught up with leader |
| **ZooKeeper** | Coordination service for cluster membership, leader election |
| **Rebalance** | Redistributing partitions among consumers |
| **Commit** | Consumer acknowledging message processing |
| **Retention** | How long messages are kept before deletion |

---

## References

1. Kreps, J., Narkhede, N., & Rao, J. (2011). Kafka: A Distributed Messaging System for Log Processing. *NetDB Workshop*.

2. Narkhede, N., Shapira, G., & Palino, T. (2017). *Kafka: The Definitive Guide*. O'Reilly Media.

3. Apache Kafka Documentation: https://kafka.apache.org/documentation/

4. LinkedIn Engineering Blog: https://engineering.linkedin.com/kafka

---

*Last updated: Module 5 - Linked and Fast Data Processing*

# Lecture 5.2: Complex Event Processing, Fast Data Querying, and Spark Streaming

## DS256 - Scalable Systems for Data Science
### Module 5: Linked and Fast Data Processing

> **References**:
> - Lecture slides: M5.FastLinkedDataProcessing.pdf (slides 31-69)
> - Zaharia, Das, Li, Hunter, Shenker, Stoica, "Discretized Streams: Fault-Tolerant Streaming Computation at Scale", SOSP 2013
> - Learning Spark, 2nd Edition, Chapter 8
> - Slide-cited CEP background examples based on Siddhi / WSO2 stream processing material

---

## 1. Motivation: From Event Arrival to Online Decisions

### 1.1 Why Kafka Alone Is Not Enough

In the previous topic, Kafka solved the **transport** problem for fast data:

- Producers can publish events at high throughput.
- Consumers can independently read, replay, and scale.
- The broker decouples ingestion from downstream processing.

But a message broker by itself does **not** answer the next question:

> How do we continuously analyze those incoming events and turn them into alerts, aggregates, and actions?

That is the role of:

1. **Complex Event Processing (CEP)** and **fast data querying**
2. **Distributed stream processing systems**
3. **Spark Streaming / Structured Streaming**

### 1.2 The Core Need in Fast Data Systems

Fast data systems are useful when they can do more than store streams. They must support:

- **Continuous filtering** of relevant events
- **Windowed aggregation** over recent data
- **Joins** across multiple live streams
- **Pattern detection** across time
- **Low-latency outputs** to dashboards, alerts, databases, or other topics
- **Fault tolerance at scale** when processing across clusters

### 1.3 Three Complementary Views of Stream Analytics

```
Raw events --> Broker --> Query / Processing Layer --> Actions / Results

Sensors, apps         Kafka      CEP / Spark Streaming    Alerts, metrics,
logs, clicks                     / Structured Streaming   models, dashboards
```

This part of the module studies the problem at three levels:

1. **CEP level**: What queries do we want to ask over event streams?
2. **System level**: How do we execute those queries continuously and in parallel?
3. **Spark level**: How does Spark provide scalable stream processing?

---

## 2. Complex Event Processing and Fast Data Querying

### 2.1 What Is Complex Event Processing?

**Complex Event Processing (CEP)** is a form of **continuous SQL-like querying over streams of events**.

The slides emphasize four key ideas:

1. **Events can be viewed as tuples** in a relational table.
2. **An event stream is like an unbounded table**.
3. **CEP detects composite events** from sequences or patterns of simpler events.
4. CEP can be used either as:
   - a **single operator** inside a larger stream-processing pipeline, or
   - a **standalone language/platform** for event analytics.

### 2.2 Events, Streams, and Continuous Queries

The database analogy is important:

| Batch / Relational View | Streaming / CEP View |
|---|---|
| Table | Unbounded event stream |
| Row / tuple | Event |
| One-time SQL query | Continuous query |
| Static result | Continuously updated result stream |

This means CEP takes familiar data-management ideas such as `filter`, `join`, and `aggregate`, but applies them to **data that never stops arriving**.

### 2.3 Why It Is Called "Complex" Event Processing

An individual event is usually simple:

- a power reading
- a temperature reading
- a stock price tick
- a tank-level sensor update

The **complex event** is the higher-level meaning we infer from several simple events together, for example:

- a **power consumption spike**
- an **abnormal sequence** of sensor readings
- a **stream join** revealing correlated behavior across devices
- a **rolling average** crossing a threshold

So CEP is not just about reading one event at a time. It is about recognizing **temporal structure** and **multi-event relationships**.

---

## 3. Sample Event Streams from the Slides

The slides introduce several example streams to show what fast data queries look like:

```text
PowerStream<timestamp, location, consumptionKWH>
ACUnitStream<timestamp, location, unitId, temperature>
TankLevelStream<timestamp, building, tankId, levelCms>
StockTickStream<timestamp, scripCode, price>
```

### 3.1 Why These Schemas Matter

Each stream has:

- a **timestamp** for when the event was observed
- one or more **entity identifiers** such as `location`, `unitId`, or `tankId`
- one or more **measured values** such as `consumptionKWH`, `temperature`, or `price`

These attributes are what enable stream queries such as:

- filter on a threshold
- group by an entity
- join streams on `location`
- detect time-ordered patterns

### 3.2 Example Use Cases

- `PowerStream`: detect high power usage or sudden spikes
- `ACUnitStream`: normalize temperature units and correlate HVAC behavior with power consumption
- `TankLevelStream`: detect overflow / underflow conditions
- `StockTickStream`: track moving averages or short-lived price anomalies

---

## 4. Filters and Transformations

### 4.1 Basic Idea

A **filter** selects only those events satisfying a predicate.

A **transformation** changes the event content before sending it to a result stream.

This is the streaming equivalent of applying `WHERE`, `SELECT`, or `MAP` continuously.

### 4.2 Slide Example: Filter Query

```sql
from PowerStream[consumptionKWH > 10]
insert into HighPowerStream
```

**Meaning**:

- Continuously inspect `PowerStream`
- Retain only events where `consumptionKWH > 10`
- Emit matching events into `HighPowerStream`

This is the simplest kind of real-time alerting pipeline.

### 4.3 Slide Example: Transformation Query

```sql
from ACUnitStream#transform.Op((temperatureF - 32) * 5 / 9)
insert into ACUnitMetricStream
```

**Meaning**:

- Continuously read AC unit temperature events
- Convert Fahrenheit to Celsius
- Write transformed events to another stream

### 4.4 Why This Matters

Filters and transformations are important because they:

- reduce downstream load
- standardize data formats
- compute derived values early
- create cleaner result streams for later joins and aggregations

---

## 5. Windows and Aggregation Operations

### 5.1 Why Windows Are Necessary

Streams are conceptually **unbounded**. That creates a problem:

> How do we compute aggregates such as average, count, or sum if the data never ends?

The solution is to define a **window**, which is a finite subset of the stream.

### 5.2 Windowed Aggregation

The slide presents the following idea:

```sql
from PowerStream#window(60 min)
select avg(consumptionKWH)
```

**Meaning**:

- Keep the events falling inside a 60-minute window
- Compute the average `consumptionKWH` over that window
- Continuously update or emit the result depending on the window semantics

### 5.3 What a Window Does Conceptually

```
Unbounded stream:   e1 e2 e3 e4 e5 e6 e7 e8 e9 ...

Window at time t:         [ e4 e5 e6 e7 ]

Aggregate: avg / sum / count / max / min over that window
```

Without a window, many stream queries would require infinite state.

### 5.4 Why Aggregation Is Central in Fast Data

Windowed aggregates are used for:

- recent average power consumption
- number of events in the last minute
- rolling counts per entity
- moving averages for stock prices
- short-term anomaly detection

---

## 6. Window Query Types

The slides explicitly list four important window types.

### 6.1 Sliding Length Window

**Definition**:

- Keeps the **last N events**
- Triggers on **every new event**

**Use case**:

- last 100 stock ticks
- last 20 sensor readings

### 6.2 Batch Length Window

**Definition**:

- Keeps the **last N events**
- Triggers **once every N events**

This is event-count-based tumbling behavior.

**Use case**:

- compute a summary for every block of 100 readings

### 6.3 Sliding Time Window

**Definition**:

- Keeps events from the **last N time units**
- Triggers on **every new event**

**Use case**:

- average over the last 5 minutes, recomputed whenever a new reading arrives

### 6.4 Batch Time Window

**Definition**:

- Keeps events from the **last N time units**
- Triggers **once at the end of each time period**

This is time-based tumbling behavior.

### 6.5 Summary Table

| Window Type | Basis | What It Retains | When It Emits |
|---|---|---|---|
| Sliding length | Event count | Last `N` events | Every new event |
| Batch length | Event count | Last `N` events | Every `N`th event |
| Sliding time | Time | Events from last `T` units | Every new event |
| Batch time | Time | Events from last `T` units | End of each period |

### 6.6 Important Distinction

The main difference is not just **what data is stored**, but **when the query is triggered**.

- Sliding windows favor low-latency continuous updates.
- Batch windows favor periodic summaries.

---

## 7. Composing Windows Across Streams

One slide illustrates a multi-step window query:

1. Form windows over two streams, say 30 seconds each
2. Compute average over each window
3. Treat those averages as result streams
4. Join the result streams and compute the difference

This is important because it shows that streaming queries are **composable**.

You do not stop at one operator. A realistic streaming workflow often looks like:

```text
raw stream --> window --> aggregate --> join --> compare --> alert
```

This composition is a major reason CEP systems need expressive query languages.

---

## 8. Join Queries on Streams

### 8.1 What a Stream Join Means

A **join query** matches events from:

- two different streams, or
- two logical views of the same stream

using a common attribute such as `location`, `sensorId`, or `timestamp bucket`.

The slides describe it as similar to a database join, but performed over a **window of events**, not a permanently bounded table.

### 8.2 Slide Example: Power and AC Join

```sql
from PowerStream#window(1 min) join ACUnitStream
on PowerStream.location = ACUnitStream.location
```

**Meaning**:

- Retain one minute of power events
- Match incoming AC-unit events on the same `location`
- Emit joined results when matches occur

### 8.3 Why Joins Need Windows

If both streams were left unbounded without retention control, the system would need to store all past events forever.

The window makes the join state **finite and meaningful**, for example:

- "match only if the power reading and AC event are within the last minute"

### 8.4 Real Meaning of a Stream Join

A stream join answers questions like:

- Is high power usage at a building correlated with high AC temperature?
- Is a tank-level anomaly happening at the same location as a pump event?
- Are two event sources referring to the same real-world object or place?

---

## 9. Join Window Execution Semantics

The slides distinguish between two important cases.

### 9.1 One-Window Join

One stream is bounded by a window and retained in memory, while events from the second stream are matched against it.

**Implication**:

- Only events arriving from the second stream trigger output.

This matters because **triggering behavior** affects both semantics and performance.

### 9.2 Two-Window Join

If both streams are windowed:

- the system retains events from both streams
- any arriving event may trigger a match and output

### 9.3 Intuition

```
Case A: one-window join
S2 window retained; each new S1 event probes S2

Case B: two-window join
Both S1 and S2 retain recent events; each new event probes the other side
```

### 9.4 Why This Distinction Matters

- It determines which arrivals can produce output.
- It determines how much state must be stored.
- It determines the computational cost of the join.

---

## 10. Temporal Sequences and Event Patterns

### 10.1 More Than Filtering or Joining

CEP becomes especially powerful when it looks for **ordered subsequences of events over time**.

The slide describes this as detecting a subsequence in which some property holds across the events.

### 10.2 Example: Detecting a Power Spike

The slides provide a pattern of the form:

```sql
from every e = PowerStream ->
PowerStream[e.consumptionKWH - consumptionKWH > 10]
insert into PowerConsumptionSpikeStream
```

The exact surface syntax is platform-specific, but the intended meaning is clear:

- Observe an event `e`
- Look for a later power event where the consumption is more than 10 units higher
- Emit a new event describing a spike

### 10.3 Why Pattern Queries Are Important

Pattern queries capture things that a simple aggregate cannot, such as:

- sudden jumps
- ordered transitions
- repeated abnormal signals
- multi-step failure signatures
- fraud-like sequences in logs or transactions

### 10.4 Regular-Expression-Like Matching

The slides note that CEP can even use **regular-expression-style pattern matching** over events.

That means event streams can be treated not just as tables, but also as **temporal sequences** in which order is part of the query.

---

## 11. Smart Water Management Examples (Siddhi-Style Queries)

One slide provides a compact set of representative Siddhi queries. They are useful because they show how different CEP query classes look in one language.

### 11.1 Filter Query

```sql
define stream inStream (height int);
from inStream[height < 150] select height
insert into outStream;
```

**Meaning**: retain only low-height readings.

### 11.2 Sequence Query

```sql
from every e1 = inStream,
     e2 = inStream[e1.height == e2.height],
     e3 = inStream[e3.height == e2.height]
select e1.height as h1, e2.height as h2, e3.height as h3
insert into outStream;
```

**Meaning**: find three related events satisfying a value relationship.

### 11.3 Pattern Query

```sql
from every e1 = inStream ->
     e2 = inStream[e1.height == e2.height] ->
     e3 = inStream[e2.height == e3.height]
select e1.height as h1, e2.height as h2, e3.height as h3
insert into outStream;
```

**Meaning**: find an explicitly ordered chain of matching events.

### 11.4 Aggregate Over a Batch Window

```sql
from inStream#window.lengthBatch(60)
select avg(height) as AvgHeight
insert into outStream;
```

**Meaning**: compute one average for each non-overlapping block of 60 events.

### 11.5 Aggregate Over a Sliding Window

```sql
from inStream#window.length(60)
select avg(height) as AvgHeight
insert into outStream;
```

**Meaning**: continuously compute the average over the most recent 60 events.

### 11.6 What This Slide Is Teaching

The point is not Siddhi syntax by itself. The point is that a CEP language should support:

- filters
- multi-event sequence logic
- explicit temporal patterns
- tumbling-style aggregation
- sliding aggregation

---

## 12. CEP Engines

The slides summarize CEP engines as systems that:

- take input stream sources
- accept CEP queries
- perform event matching
- run standalone or embedded inside other systems
- often provide SQL-like query models

### 12.1 Example Platforms

- **Siddhi**
- **Esper**
- and related stream / CEP engines

### 12.2 What a CEP Engine Actually Does

A CEP engine is not just a parser for a query language. Internally it must manage:

- event ingestion
- window state
- pattern matching state machines
- join buffers
- trigger logic
- output streams or sinks

So the engine is both:

1. a **language runtime**, and
2. a **stream execution system**

---

## 13. Comparison Slides: What They Are Comparing

Three lecture slides compare technologies. For exams, the key point is understanding the **comparison axes**, not memorizing every historical number.

### 13.1 Comparison of Event Processing Technologies

The event-processing-platform comparison slide evaluates platforms such as:

- TIBCO StreamBase
- IBM InfoSphere Streams
- DataTorrent RTS
- Amazon Kinesis
- WSO2 CEP
- Fujitsu Interstage CEP
- SQLstream

#### Main Feature Axes in the Slide

- SQL-like query support
- query composition
- temporal pattern / sequence queries
- core query types: joins and windows
- window types: sliding, time, tuple, batch
- GIS integration
- result display support
- database integration
- distributed processing
- visual debugging
- messaging integration such as JMS / Kafka
- built-in fault tolerance
- high availability
- scalability
- operator library richness
- language / API extensibility
- open-source licensing

#### Course-Level Takeaways

1. Some systems are strong **query engines** but less focused on full distributed execution.
2. Some systems are strong **stream platforms** but weaker as full CEP languages.
3. WSO2 CEP is highlighted in the comparison as a relatively feature-rich CEP platform with SQL-like queries and messaging integration.
4. Amazon Kinesis appears more as an ingestion / stream platform than a rich CEP language runtime.
5. Historical throughput numbers on these slides are **not directly apples-to-apples**, because workloads and configurations differ.

### 13.2 Comparison of Distributed Stream Processing Technologies

The distributed stream-computing comparison slide includes:

- S4
- Storm
- Spark Streaming
- Samza
- Apex
- Flink

#### Main Feature Axes in the Slide

- SQL-like queries
- database / relational DB connection
- join and window support
- window types
- fault tolerance model
- high availability
- scalability
- reported performance

#### Important Distinctions the Slide Is Conveying

| System | Main Historical Emphasis in the Slide |
|---|---|
| S4 | Early distributed stream system, more limited feature set |
| Storm | Event-at-a-time dataflow, historically popular, strong operational simplicity |
| Spark Streaming | Micro-batch model with recomputation-based fault recovery |
| Samza | Stream processing with state persistence |
| Apex | Stateful stream processing with joins and windows |
| Flink | Rich windowing and strong event-time semantics |

#### Specific Spark Streaming Insight

The slide explicitly labels Spark Streaming's fault tolerance as **recompute**.

That captures the D-Stream design idea:

- store lineage / state
- rebuild lost work deterministically
- avoid full hot replication

### 13.3 Comparison of CEP Engines / Libraries

This slide compares libraries such as:

- Esper
- Siddhi
- RuleCore
- Cayuga

#### Feature Axes in the Slide

- SQL-like queries
- DB connection
- open-source license
- debugging support
- window-type support
- sequence support
- event aggregation
- nested queries
- data extraction
- parameterization
- best reported performance

#### Main Takeaways

1. **Siddhi** is shown as supporting a broad set of window types and high throughput.
2. **Esper** is shown as strong in SQL-like querying, debugging, and several library-style features.
3. Different CEP engines trade off language richness, tooling, and performance.

### 13.4 Why These Comparison Slides Matter

These slides are teaching that fast-data systems differ along three separate dimensions:

1. **Expressiveness**: What queries can be written?
2. **Systems support**: How scalable and fault-tolerant is execution?
3. **Operational integration**: How well does the system connect to brokers, DBs, UIs, and external services?

---

## 14. Distributed Stream Processing Background

Before Spark Streaming, the slides briefly summarize the conventional **distributed stream processing** model.

### 14.1 Continuous Dataflow Model

In this model, the user application is a **dataflow graph** of logic blocks or tasks.

Each task:

- consumes one event at a time
- produces zero or more output events
- may run in parallel with other tasks

### 14.2 Types of Parallelism in the Slide

The slide mentions three types:

1. **Data parallelism** across events or streams
2. **Task parallelism** across parallel branches in the dataflow
3. **Pipelined execution** across downstream operators

### 14.3 Limitations Highlighted by the Slide

- limited support for rich stateful execution semantics
- routing / coordination complexity
- no natural global ordering guarantee
- stream joins are often awkward or weakly supported
- outputs can be interleaved in hard-to-reason-about ways

### 14.4 Example Systems Mentioned

- **Storm**
- **Flink**
- related distributed stream engines

---

## 15. Distributed Micro-Batch Processing

The next slide introduces an alternative design:

> **Distributed micro-batch stream processing**

### 15.1 Core Idea

Instead of processing each record immediately in a long-running operator, the system:

1. collects records into small batches
2. processes each batch using parallel tasks
3. emits results batch by batch

### 15.2 Spark Connection

The slide ties this to **Spark 1.x**.

This is the key conceptual shift behind Spark Streaming:

- treat stream processing as a series of tiny batch jobs
- reuse a batch execution engine
- gain deterministic recovery and scalable scheduling

### 15.3 Visual Intuition

```
Input stream --> micro-batches --> parallel tasks --> output micro-batches
```

This model trades a bit of latency for stronger fault tolerance and simpler large-scale execution.

---

## 16. Spark Streaming 1.x: DStreams

### 16.1 What Is a DStream?

The slide defines a **DStream** as a sequence of data arriving over time.

More precisely:

- it is represented as a **sequence of RDDs**
- one RDD corresponds to one micro-batch / time step
- it can be created from sources such as **Kafka** and **HDFS**

### 16.2 Operations on DStreams

Two categories are emphasized:

1. **Transformation operations**
   - produce a new DStream
2. **Output operations**
   - write data to an external system

### 16.3 Why DStreams Were Powerful

DStreams let Spark reuse its batch abstractions for streaming:

- RDD-based fault tolerance
- familiar transformation style
- cluster-wide parallel execution

### 16.4 Limitation Mentioned in the Slide

The slide explicitly notes:

- DStreams inherit limitations similar to RDDs
- they do **not naturally support event windows** the way later Structured Streaming does
- they are more naturally aligned with **processing windows / micro-batches**

This is an important transition point in the history of Spark Streaming.

---

## 17. D-Streams from the SOSP 2013 Paper

The main paper for Spark Streaming is Zaharia et al., SOSP 2013, which introduced **Discretized Streams (D-Streams)**.

### 17.1 The Problem the Paper Is Solving

The paper starts from a practical observation:

- much big data arrives in real time
- many applications need decisions in seconds, not hours
- cluster-scale streaming must tolerate **faults** and **stragglers**

The paper argues that the conventional continuous-operator model makes this hard.

### 17.2 Why Continuous Operator Systems Were Not Enough

The paper describes two classic recovery strategies:

1. **Replication**
   - two copies of each node / operator
   - expensive because it can roughly double hardware cost
   - also requires synchronization so replicas see inputs in the same order

2. **Upstream backup**
   - parents replay old messages to rebuild failed state
   - recovery is serial and can be slow
   - does not naturally solve straggler problems

The paper's core critique is:

> Long-lived, mutable, continuously updated operators are difficult to recover efficiently at large scale.

### 17.3 Design Goals in the Paper

The paper explicitly wants:

1. scalability to hundreds of nodes
2. minimal cost beyond base processing
3. second-scale latency
4. second-scale recovery from faults and stragglers

---

## 18. The D-Stream Computation Model

### 18.1 Core Idea

D-Streams structure streaming computation as a series of:

- **small time intervals**
- **deterministic batch computations**
- over **immutable distributed datasets**

Instead of one long-running mutable operator, the system repeatedly does:

1. collect records for interval `t`
2. store them reliably as a distributed dataset
3. run deterministic operations on that dataset
4. produce output and updated state datasets

### 18.2 Why This Helps

Because the computations are deterministic and state is stored in immutable datasets:

- recovery can be based on **lineage**
- lost partitions can be recomputed in **parallel**
- speculative execution can mitigate stragglers

### 18.3 Spark Implementation View

In Spark Streaming:

- each interval becomes an RDD
- a DStream is a sequence of those RDDs
- state can be represented by additional RDDs

### 18.4 Visual Intuition

```
time t=1: input batch --> RDD1 --> transformations --> state/output RDDs
time t=2: input batch --> RDD2 --> transformations --> state/output RDDs
time t=3: input batch --> RDD3 --> transformations --> state/output RDDs
```

The paper's key claim is that this makes streaming look like a succession of tiny batch jobs.

---

## 19. Example D-Stream Program from the Paper

The paper gives a running count example:

```scala
pageViews = readStream("http://...", "1s")
ones = pageViews.map(event => (event.url, 1))
counts = ones.runningReduce((a, b) => a + b)
```

### 19.1 What It Means

- `pageViews` groups incoming events into 1-second batches
- `ones` converts each event to `(url, 1)`
- `counts` maintains a running aggregate over time

### 19.2 The Important Insight

This program is not executed as one endless operator.

Instead, each time step creates:

- one input RDD
- derived RDDs from `map`
- state RDDs for the running count

The DStream is therefore a **logical stream abstraction**, while the actual runtime structure is a **lineage graph of RDDs**.

---

## 20. Timing Considerations and Late Records

### 20.1 What Time Defines a Batch?

The paper makes an important semantic point:

- D-Streams place records into batches based on **arrival time to the system**
- not necessarily the original external event time

This choice keeps execution simple and predictable.

### 20.2 Handling Out-of-Order / Late Data

The paper suggests two approaches:

1. **Slack time**
   - wait a bit before processing a batch so slightly late events can still arrive

2. **Application-level correction**
   - compute a result quickly
   - later emit corrected results when more late events arrive

### 20.3 Why This Matters

Late data is not a Spark-only problem. It is a fundamental streaming problem.

Any real stream system must choose between:

- lower latency
- more complete results
- more correction / retraction logic

This idea later becomes much more explicit in Structured Streaming's event-time model.

---

## 21. D-Stream API from the Paper

### 21.1 Two Kinds of Operations

Spark Streaming exposes:

1. **Transformations**
   - create new DStreams from existing ones
2. **Output operations**
   - write data to external systems

### 21.2 Stateless Transformations

These are similar to batch operations and apply per interval, for example:

- `map`
- `reduceByKey`
- `groupBy`
- `join`

### 21.3 Stateful Operations

The paper highlights several stateful operators.

#### Windowing

The `window` operation groups data from multiple recent intervals into one RDD.

Example idea:

- `words.window("5s")` gives an RDD for `[0,5)`, then `[1,6)`, then `[2,7)`, and so on

#### Incremental Aggregation

The `reduceByWindow` family supports repeated aggregations over sliding windows.

There are two important cases:

1. **Associative merge only**
   - can combine values
   - but may repeatedly re-sum older window contents

2. **Associative and invertible**
   - can add new contributions and subtract expired ones
   - much more efficient for sliding windows

#### State Tracking

The paper also presents a `track` operation for per-key state.

This is conceptually:

- initialize state when a key first appears
- update state when new events for that key arrive
- remove stale state after timeout

This pattern is the ancestor of later stateful streaming APIs.

### 21.4 Output Operations

The paper mentions:

- `save`
- `foreachRDD`

These allow each micro-batch result to be sent to a database, file system, dashboard logic, or arbitrary Spark code.

---

## 22. Consistency Semantics of D-Streams

### 22.1 The Paper's Main Claim

One major benefit of D-Streams is **clean consistency semantics**.

Because time is explicitly discretized into intervals:

- each output RDD corresponds to a well-defined prefix of the stream
- the result is deterministic given the inputs
- users do not need to reason about operator replicas observing different interleavings

### 22.2 Why This Is Better Than Ad Hoc Event Interleaving

In continuous systems, if different nodes lag differently, a cluster-wide state snapshot can be inconsistent.

With D-Streams, the logical state is defined at interval boundaries.

That means the system provides a conceptually clean **exactly-once style cluster-level result**, assuming the output side is handled carefully.

### 22.3 Practical Interpretation

The exact result of an interval is as if:

- all prior intervals had completed correctly
- all tasks were deterministic
- no message reordering ambiguity mattered within that interval definition

---

## 23. Unification with Batch and Interactive Processing

One of the paper's most important ideas is that D-Streams and batch jobs use the **same underlying abstraction: RDDs**.

### 23.1 Why This Is Powerful

It means a streaming system can:

1. **join live streams with historical batch data**
2. **run streaming logic on old data in batch mode**
3. **support ad-hoc interactive queries** on live in-memory state

### 23.2 Examples from the Paper

- compare incoming events with historical RDDs
- run streaming reports over historical data
- use an interactive shell to inspect current stream state

### 23.3 Why This Was a Big Deal

Traditionally, organizations maintained separate systems for:

- online streaming
- offline batch analytics
- interactive exploration

The paper argues that this split wastes both engineering effort and data freshness.

---

## 24. D-Streams Versus Continuous Operator Systems

The paper includes a concise comparison. The main ideas are worth remembering.

| Aspect | D-Streams | Continuous Operator Systems |
|---|---|---|
| Latency | Typically 0.5-2 s | Often lower, record-at-a-time |
| Consistency | Interval-based, deterministic | Often harder due to interleaving |
| Late records | Slack or app-level correction | Out-of-order handling / synchronization |
| Fault recovery | Parallel recomputation | Replication or serial replay |
| Straggler handling | Speculative execution possible | Typically difficult |
| Batch integration | Natural via RDDs | Usually separate systems |

### 24.1 Key Trade-Off

D-Streams deliberately give up the absolute lowest latency in order to gain:

- simpler semantics
- efficient parallel recovery
- a common engine for stream and batch processing

---

## 25. Spark Streaming System Architecture

The paper implements D-Streams in **Spark Streaming**.

### 25.1 Main Components

The paper describes three major components:

1. **Master**
   - tracks D-Stream lineage
   - schedules tasks

2. **Worker nodes**
   - receive data
   - store input and state partitions
   - run tasks

3. **Client library**
   - sends data into the system

### 25.2 Reliable Input Handling

When data is pushed directly from clients:

- Spark Streaming replicates the input to workers before acknowledging receipt
- if a worker fails before acknowledgment, the client resends to another worker

This is necessary because recomputation only works if the original input is reliably stored.

### 25.3 Block Store Abstraction

Data is stored in blocks with unique IDs.

Because blocks and RDD partitions are immutable:

- tracking them is simpler
- any node holding the same block can serve it
- lineage can refer to them precisely

### 25.4 Interval Execution

At each interval boundary:

- workers report which blocks they received
- the master launches tasks for that interval
- Spark's scheduler exploits data locality and pipelining

This is what turns the logical D-Stream program into actual work on the cluster.

---

## 26. Optimizations Added for Stream Processing

The paper explains that Spark needed several changes to become a good streaming engine.

### 26.1 Network Communication

Spark's data plane was rewritten to use **asynchronous I/O** so remote inputs could be fetched faster.

### 26.2 Timestep Pipelining

Tasks from a later interval can begin before the previous interval fully finishes, when dependencies allow it.

This improves utilization because cluster resources would otherwise sit idle at the tail of each interval.

### 26.3 Scheduler Optimizations

The scheduler was tuned to launch hundreds of tasks every few hundred milliseconds.

This is essential because streaming uses many more short jobs than batch workloads do.

### 26.4 Storage-Layer Improvements

Spark Streaming added:

- asynchronous checkpointing
- better storage performance
- zero-copy I/O where possible

### 26.5 Lineage Cutoff

Lineage graphs can otherwise grow forever, so once an RDD is checkpointed, older lineage can be discarded.

### 26.6 Master Recovery Support

Because a streaming job is long-lived, the master also needs a recovery path if it fails.

---

## 27. Memory Management in Spark Streaming

The paper notes that the block store uses an **LRU policy**:

- new blocks stay in memory when possible
- old blocks may spill to disk if needed
- very old history can be forgotten after a configurable timeout

### 27.1 Important Insight

In many streaming workloads, the maintained state is much smaller than the raw input stream.

Examples:

- counts per key
- session summaries
- running aggregates

So the memory footprint can be manageable even for high-throughput streams.

---

## 28. Fault Recovery: Parallel Recompution Instead of Full Replication

### 28.1 Main Idea

When a node fails, Spark Streaming can recompute lost RDD partitions **in parallel across the cluster**.

This is the paper's signature contribution.

### 28.2 Why Parallel Recovery Is Possible

Because:

- state is in immutable RDDs
- transformations are deterministic
- lineage captures fine-grained dependencies

multiple machines can help reconstruct the missing state.

### 28.3 Checkpointing and Recovery

The paper periodically checkpoints some state RDDs.

When failure occurs:

1. identify lost partitions
2. start from the latest checkpoint
3. recompute missing partitions in parallel

### 28.4 Analytical Result from the Paper

The paper compares upstream backup against parallel recovery. If the pre-failure load is `lambda` and the cluster has `N` machines, then the paper derives:

- upstream backup catch-up time: `t_up = lambda / (1 - lambda)`
- parallel recovery catch-up time: approximately `t_par ~= lambda / (N(1 - lambda))`

The exact modeling assumptions are simplified, but the intuition is strong:

> More machines can participate in recovery, so recovery time shrinks substantially.

---

## 29. Straggler Mitigation

### 29.1 Why Stragglers Matter

At large scale, not all failures are crashes. Some nodes become **slow**.

These stragglers can dominate end-to-end latency.

### 29.2 D-Streams Solution

Because tasks are deterministic and short-lived, the system can run **speculative backup copies** of slow tasks, similar to batch systems.

### 29.3 Heuristic Used in the Paper

The implementation marks a task as slow when it runs more than about **1.4x the median task duration** in its stage.

### 29.4 Why This Is Hard in Continuous Operators

In a traditional continuous operator system, a slow operator usually carries mutable state and a long history.

Launching a backup is difficult because the backup must first rebuild that state.

In D-Streams, speculation is much cleaner because work is already divided into deterministic tasks over well-defined intervals.

---

## 30. Master Recovery

The paper also addresses failure of the Spark master.

### 30.1 Strategy

Persist enough metadata to durable storage so that a new master can resume execution.

The paper stores items such as:

- D-Stream graph / lineage metadata
- user function metadata
- latest checkpoint time
- recent RDD identifiers

### 30.2 Why This Works

Because recomputation is deterministic, rerunning some work is acceptable.

The paper emphasizes that output operators should be **idempotent**, so re-executing an interval does not corrupt downstream results.

---

## 31. Performance Results from the Paper

### 31.1 Throughput and Scaling

The paper reports that Spark Streaming can scale nearly linearly to 100 nodes.

Representative numbers in the paper include:

- about **6 GB/s** for a Grep workload at sub-second latency on 100 nodes
- about **2.3 GB/s** for more CPU-intensive workloads such as WordCount / TopKCount

### 31.2 Comparison with Commercial Systems

The paper argues Spark Streaming's **per-node throughput** is comparable to commercial streaming engines, while also scaling well in clusters.

### 31.3 Comparison with Storm and S4

Historically, the paper reports Spark Streaming was faster than the tested open-source competitors for the chosen workloads, while also offering stronger recovery semantics.

The paper's high-level conclusion is:

- Spark Streaming is not just expressive
- it is also operationally competitive at scale

---

## 32. Fault-Recovery Evaluation Results

### 32.1 Recovery Latency

The paper shows recovery delays on the order of:

- well under a second in several tested cases
- still only a few seconds even with longer checkpoint intervals

### 32.2 Checkpoint Trade-Off

More frequent checkpoints:

- reduce recovery time
- add more overhead

Less frequent checkpoints:

- reduce normal overhead
- require more recomputation after failure

### 32.3 Node Count Effect

The paper shows that using more nodes reduces recovery time, which directly supports the parallel recovery argument.

---

## 33. Real Applications Ported to Spark Streaming

The paper studies two realistic applications.

### 33.1 Video Distribution Monitoring (Conviva)

This application tracks video delivery behavior across:

- regions
- CDNs
- client devices
- ISPs

The existing implementation had separate systems for:

- live streaming analytics
- historical analytics / ad-hoc queries

Spark Streaming allowed the authors to unify these more naturally.

#### Why This Example Matters

It demonstrates the paper's larger thesis:

> Streaming, batch, and interactive analysis should not be isolated silos.

### 33.2 Mobile Millennium Traffic Estimation

This application estimates traffic conditions using noisy, sparse GPS data.

The paper shows that D-Streams can support even heavy machine-learning-style workloads by:

- incrementally incorporating new data
- combining live data with historical data
- scaling across many nodes

---

## 34. Discussion and Limitations from the Paper

### 34.1 Main Limitation: Batching Adds Latency

D-Streams introduce a fixed minimum latency because data is grouped into batches.

This makes them less suitable for extremely low-latency applications such as high-frequency trading.

### 34.2 Batch Interval Trade-Off

The batch interval directly controls the trade-off between:

- latency
- throughput
- scheduling overhead

### 34.3 Memory Usage

Keeping multiple state RDD versions can use more memory than mutable-state continuous operators.

### 34.4 Approximate Results as a Future Direction

The paper suggests it may be possible to emit approximate partial results during recovery.

### 34.5 Long-Term Importance

Even if later systems evolved beyond the original DStream API, the paper's key contribution remains highly influential:

- discretize stream processing
- gain strong recovery and a unified engine

---

## 35. Structured Streaming: Spark 2.x and Later

The lecture then moves from classic Spark Streaming / DStreams to **Structured Streaming**.

### 35.1 Main Shift in Abstraction

Structured Streaming models a stream as an **unbounded table / DataFrame**.

That means users can write streaming logic using the same high-level DataFrame / SQL style used for batch jobs.

### 35.2 Why This Is Better Than Raw DStreams

Compared with DStreams, Structured Streaming gives:

- a higher-level API
- better integration with the Spark SQL optimizer
- clearer output semantics
- explicit event-time windowing
- better unification of batch and stream programming

### 35.3 Slide Summary

The slides emphasize:

- Spark 2.x
- unified API for batch and stream processing
- a continuum between real-time and batch

This is conceptually a modernized and more declarative successor to the D-Stream model.

---

## 36. Incremental Execution of Batch Queries

One of the most important Structured Streaming ideas is:

> A streaming query is written like a batch query, but executed incrementally as new data arrives.

### 36.1 The Execution Pipeline in the Slide

The slide describes the process as:

1. create a micro-batch
2. optimize the logical plan
3. update the result table
4. emit the new result to the output sink

### 36.2 Visual Intuition

```
User code --> logical plan --> optimized plan --> repeated incremental execution
```

This is the central Structured Streaming idea:

- users describe *what* result they want
- Spark decides *how* to maintain it incrementally over time

---

## 37. Streaming Query Lifecycle

The slides present a five-step lifecycle.

```text
Define input sources
    --> Transform data
    --> Define output sink and output mode
    --> Specify processing details
    --> Start the query
```

### 37.1 Step 1: Define Input Sources

The input source populates and updates the input DataFrame.

The stream can be unbounded and come from sources such as:

- socket
- files
- Kafka
- other supported stream inputs

### 37.2 Step 2: Transform Data

Use standard DataFrame transformations.

The slides distinguish:

- **stateless operations**: `select`, `filter`, `map`
- **stateful operations**: operations such as `count` that need prior rows or maintained state

### 37.3 Step 3: Define Output Sink and Output Mode

The sink decides where results go, such as:

- console
- file
- Kafka

The output mode decides **what portion of the result table** gets emitted.

### 37.4 Step 4: Specify Processing Details

This includes trigger behavior and checkpointing.

### 37.5 Step 5: Start the Query

`start()` launches the streaming query.

The slides note:

- it is non-blocking
- you can use `awaitTermination()`
- you can stop the query with `stop()`

---

## 38. Output Modes in Structured Streaming

The slides list three output modes.

### 38.1 Append Mode

Only rows newly added since the last trigger are emitted.

This is appropriate when earlier rows will not change, which is common for stateless streaming transformations.

### 38.2 Update Mode

Only rows modified since the last trigger are emitted.

This is useful for incremental stateful queries where some groups change over time.

### 38.3 Complete Mode

The entire current result table is emitted on each trigger.

This is conceptually simple but only practical when the result table remains small enough.

### 38.4 Why Output Mode Matters

The output mode controls:

- result semantics
- sink behavior
- network / sink cost
- whether a query is practical at scale

---

## 39. Writing Structured Streaming Output to Kafka

One slide shows writing a result DataFrame to Kafka.

### 39.1 Slide Pattern

The result table is converted so Kafka gets `key` and `value` columns:

```python
counts = ...  # DataFrame[word: string, count: long]

streamingQuery = (counts
    .selectExpr(
        "cast(word as string) as key",
        "cast(count as string) as value")
    .writeStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
    .option("topic", "wordCounts")
    .outputMode("update")
    .option("checkpointLocation", checkpointDir)
    .start())
```

### 39.2 What This Teaches

To write to Kafka from Structured Streaming:

- select or derive the columns to publish
- cast them into Kafka-friendly `key` / `value` fields
- configure bootstrap servers and topic
- choose an output mode
- enable checkpointing

This is the bridge from analytics back into the messaging layer.

---

## 40. Triggers and Processing Modes

The slides describe four trigger styles.

### 40.1 Default Micro-Batch Trigger

Run a micro-batch whenever the previous one finishes.

This is the standard mode.

### 40.2 Trigger Interval

Accumulate rows for a fixed time interval, then run one micro-batch.

Example: every 1 second, every 5 seconds, etc.

### 40.3 Once Trigger

Run one micro-batch with all currently available data.

This is useful for backfills or one-shot incremental processing.

### 40.4 Continuous Trigger

The slide notes a **continuous** mode, where execution can process one row at a time.

The lecture still primarily frames Spark around micro-batch execution, but this slide shows Spark's evolution toward lower-latency execution options.

### 40.5 Checkpointing and Exactly-Once Guarantees

The slide explicitly says that checkpointing to persistent storage supports **exactly-once guarantees** on the input stream.

Conceptually, checkpointing records enough progress and state information so the engine can resume correctly after failure.

---

## 41. Full Structured Streaming Example

The slides include a socket-based word-count example in Python.

### 41.1 Source and Query

```python
from pyspark.sql.functions import *

spark = SparkSession...

lines = (spark
    .readStream.format("socket")
    .option("host", "localhost")
    .option("port", 9999)
    .load())

words = lines.select(split(col("value"), "\\s").alias("word"))
counts = words.groupBy("word").count()
```

### 41.2 Sink and Trigger

```python
streamingQuery = (counts
    .writeStream
    .format("console")
    .outputMode("complete")
    .trigger(processingTime="1 second")
    .option("checkpointLocation", checkpointDir)
    .start())

streamingQuery.awaitTermination()
```

### 41.3 What the Example Shows

This example exercises the full lifecycle:

- define source
- transform input
- define sink
- choose output mode
- choose trigger interval
- configure checkpointing
- start and wait for query termination

---

## 42. Stateless and Stateful Transformations

### 42.1 Stateless Transformations

These only need the current row or current micro-batch row values.

Examples from the slides:

- `select()`
- `filter()`
- `map()`

For a query containing only stateless operations, the slide notes that **append** and **update** modes are supported, but **complete** mode is not needed.

### 42.2 Stateful Transformations

Stateful operations perform **incremental aggregation**.

They need the system to remember prior information across micro-batches.

The slide emphasizes that for such queries:

- state must be maintained across batches
- distributed execution must still produce correct results
- the query must use a `groupBy` clause for distributed keyed aggregation

### 42.3 Why This Distinction Is Important

This is the difference between:

- simply transforming each row, and
- maintaining a continuously updated result table over time

---

## 43. Stateful Distributed Execution

The slides give a concrete systems view of stateful execution.

### 43.1 Key Requirement

Partitioning and shuffles must be done **consistently** so that the same executor keeps handling the same `groupBy` key values.

For example, if the key is `sensorId`, then updates for a given sensor should continue to go to the same logical state location.

### 43.2 Why This Matters

If updates for one key were randomly sent to different executors on each micro-batch:

- the state would fragment
- aggregations would be incorrect
- recovery would become messy

### 43.3 Fault Recovery

The slide also shows that state is backed by **external storage** so it can be recovered after failure.

This is the structured counterpart to the older D-Stream emphasis on checkpointed state and recomputation.

---

## 44. Non-Temporal Aggregations

The slides next discuss aggregations that run **since the beginning of the stream**, rather than within explicit time windows.

### 44.1 Global Aggregation

Example from the slide:

```python
runningCount = sensorReadings.groupBy().count()
```

This computes one continuously updated global count.

### 44.2 Grouped Aggregation

Example from the slide:

```python
baselineValues = sensorReadings.groupBy("sensorId").mean("value")
```

This maintains a running aggregate **per key**.

### 44.3 Built-In Aggregation Functions Mentioned

The slide lists several common aggregators:

- `sum()`
- `mean()`
- `stddev()`
- `countDistinct()`
- `collect_set()`
- `approx_count_distinct()`

### 44.4 Chaining Aggregators

The slide also shows that multiple aggregations can be computed together, for example:

- count
- mean
- set collection

in the same grouped query.

This is useful because many real streaming dashboards need several metrics for the same key at once.

---

## 45. Aggregations with Event-Time Windows

This is one of the most important modern Structured Streaming topics.

### 45.1 Event Time vs Processing Time

The slides distinguish two notions of time:

1. **Event time**
   - the timestamp stored in the record itself
   - when the event actually happened

2. **Processing time**
   - when the engine happens to process the record

For streaming analytics, this difference is crucial because data can arrive late or out of order.

### 45.2 Slide Example

```python
(sensorReadings
    .groupBy("sensorId", window("eventTime", "5 minute"))
    .count())
```

This groups by both:

- entity key: `sensorId`
- event-time window: 5 minutes

### 45.3 Why Event Time Is Better for Semantics

Suppose an event that happened at 12:07 arrives only at 12:15.

If we use processing time, it would be counted with 12:15 data.

If we use event time, it is assigned to the window where it belongs, namely the one covering 12:05-12:10.

That is exactly what the slide diagrams illustrate.

---

## 46. Tumbling and Overlapping Event-Time Windows

### 46.1 Tumbling Window Example

The slide shows mapping event time to **5-minute tumbling windows** such as:

- 12:00-12:05
- 12:05-12:10
- 12:10-12:15

Each event belongs to exactly one such window.

### 46.2 Overlapping / Sliding Window Example

The final event-time slide shows overlapping windows, for example:

```python
(sensorReadings
    .groupBy("sensorId", window("eventTime", "10 minute", "5 minute"))
    .count())
```

This means:

- window length = 10 minutes
- slide interval = 5 minutes

So windows overlap, such as:

- 12:00-12:10
- 12:05-12:15
- 12:10-12:20

An event may belong to more than one window.

### 46.3 What the Slide Diagrams Teach

The diagrams make three points explicit:

1. event arrival order may differ from event occurrence time
2. event-time bucketing uses the event timestamp, not arrival order
3. overlapping windows can map a single event into multiple aggregates

### 46.4 Important Boundary of These Slides

The slides stop at **event-time windowing semantics**. They do not go into watermarking in detail here.

So the main exam takeaway is:

- understand event time
- understand processing time
- understand tumbling vs overlapping windows

---

## 47. Big-Picture Evolution: CEP to D-Streams to Structured Streaming

This lecture segment fits together as follows:

### 47.1 CEP Perspective

CEP asks:

- what patterns, joins, filters, and windows do we want to compute over streams?

### 47.2 D-Stream / Spark Streaming Perspective

D-Streams ask:

- how can we execute streaming computation scalably with deterministic micro-batches, lineage, and parallel recovery?

### 47.3 Structured Streaming Perspective

Structured Streaming asks:

- how can we expose stream processing through a higher-level, table-oriented API with explicit output modes and event-time semantics?

### 47.4 One-Line Summary

```
CEP gives the query semantics;
D-Streams give the fault-tolerant execution model;
Structured Streaming gives the modern Spark API and result semantics.
```

---

## 48. High-Yield Exam Distinctions

### 48.1 CEP vs Stream Processing Engine

- **CEP** focuses on event-query semantics such as windows, joins, and patterns.
- A **stream processing engine** focuses on how those queries are executed at scale.

### 48.2 DStream vs Structured Streaming

- **DStream**: sequence of RDDs, micro-batch oriented, lower-level Spark 1.x API
- **Structured Streaming**: unbounded DataFrame / table, higher-level Spark 2.x+ API

### 48.3 Processing Time vs Event Time

- **Processing time**: when the system sees the record
- **Event time**: when the event actually occurred

### 48.4 Sliding vs Batch / Tumbling Windows

- **Sliding**: update continuously or at shorter steps with overlap
- **Batch / tumbling**: non-overlapping blocks, emit once per block

### 48.5 Replication vs Replay vs Recomputation

- **Replication**: expensive but fast failover
- **Upstream backup / replay**: cheaper but slower recovery
- **Lineage-based recomputation**: Spark Streaming's key idea for scalable recovery

---

## 49. Additional Reading

The slides list the following CEP-related references:

- Srinath Perera, "A Gentle Introduction to Stream Processing"
- Simmhan and Perera, "Big Data Analytics Platforms for Real-time Applications in IoT"
- Dayarathna and Perera, "Recent advancements in event processing"
- Nasiri et al., "Evaluation of distributed stream processing frameworks for IoT applications in Smart Cities"
- Shukla et al., "RiotBench"
- Cugola and Margara, "Processing flows of information: From data stream to complex event processing"

For Spark-specific follow-up, the most important references remain:

- the SOSP 2013 D-Streams paper
- Learning Spark, Chapter 8

---

## 50. Coverage Checklist Against Slides and Paper

### 50.1 CEP and Fast Data Querying Slides (31-47)

- [x] CEP as continuous SQL over event streams
- [x] events as tuples and streams as unbounded tables
- [x] composite events from simple events
- [x] CEP as operator vs standalone platform
- [x] sample event stream schemas
- [x] filters and transformations
- [x] windowed aggregation
- [x] sliding length, batch length, sliding time, batch time windows
- [x] composing windows across streams
- [x] join queries over streams
- [x] one-window vs two-window join execution semantics
- [x] temporal sequence / pattern detection
- [x] smart water management Siddhi examples
- [x] CEP engines and example platforms
- [x] comparison of event processing technologies
- [x] comparison of distributed stream processing technologies
- [x] comparison of CEP libraries / engines
- [x] additional reading slide

### 50.2 Spark Streaming / Structured Streaming Slides (48-69, excluding Pregel transition)

- [x] distributed stream processing dataflow model
- [x] task, data, and pipeline parallelism
- [x] limits of traditional continuous stream processing
- [x] distributed micro-batch stream processing
- [x] DStreams as sequences of RDDs
- [x] DStream transformations and outputs
- [x] DStream limitations relative to later APIs
- [x] Structured Streaming as unbounded table / DataFrame
- [x] incremental execution of batch queries over streams
- [x] streaming query lifecycle
- [x] input sources
- [x] stateless vs stateful transformations
- [x] sinks and output modes: append / update / complete
- [x] writing output to Kafka
- [x] trigger modes: default, interval, once, continuous
- [x] checkpointing for exactly-once input guarantees
- [x] `start()`, `stop()`, and `awaitTermination()` semantics
- [x] full Python word-count example
- [x] execution model: micro-batch, optimize plan, update result, emit sink
- [x] stateful distributed execution and consistent partitioning
- [x] non-temporal aggregations
- [x] global aggregation
- [x] grouped aggregation
- [x] built-in aggregate functions and chaining
- [x] event-time windows
- [x] event time vs processing time
- [x] tumbling windows
- [x] overlapping / sliding event-time windows

### 50.3 Spark Streaming Paper Coverage

- [x] motivation for large-scale low-latency streaming
- [x] problems with continuous operator systems
- [x] replication vs upstream backup limitations
- [x] design goals of D-Streams
- [x] D-Stream computation model
- [x] running-count example and RDD lineage intuition
- [x] timing considerations and late-data handling
- [x] D-Stream API: transformations, outputs, windows, incremental aggregation, tracking state
- [x] consistency semantics
- [x] unification with batch and interactive processing
- [x] D-Streams vs continuous systems comparison
- [x] Spark Streaming architecture
- [x] execution flow and block storage
- [x] streaming-specific optimizations
- [x] memory management
- [x] parallel recovery
- [x] straggler mitigation
- [x] master recovery
- [x] performance evaluation
- [x] recovery evaluation
- [x] real applications: Conviva and Mobile Millennium
- [x] discussion and limitations

### 50.4 Final Check

All lecture topics in the requested CEP / fast data querying / Spark Streaming segment have been covered in these notes. The only deliberate boundary is that the final transition slide to **Pregel / linked data analytics** is not expanded here, because that belongs to the third part of Module 5.

# Lecture 5.3: Pregel and Large-Scale Graph Processing

## DS256 - Scalable Systems for Data Science
### Module 5: Linked and Fast Data Processing

> **References**:
> - Lecture slides: M5.FastLinkedDataProcessing.pdf (slides 69-109)
> - Malewicz et al., "Pregel: A System for Large-Scale Graph Processing", SIGMOD 2010
> - Apache Giraph lecture material cited in the slides

---

## 1. Motivation: Why Linked Data Analysis Matters

### 1.1 From Fast Data to Linked Data

The previous part of Module 5 focused on **fast data**:

- event streams
- low-latency processing
- CEP and Spark Streaming

This part shifts to **linked data**, where the main structure is not a stream or a table, but a **graph**.

In graph problems, the central question is not only:

> What is the value of this record?

but also:

> How is this entity connected to others, and what can we infer from those connections?

### 1.2 Why Graphs Matter in Real Systems

The slides begin by emphasizing that **graphs are commonplace**.

They arise in:

- **Web and social networks**
  - web graph
  - citation networks
  - Twitter, Facebook, Internet topology
- **Knowledge networks and relationships**
  - Google's Knowledge Graph
  - NELL and similar knowledge bases
- **Cybersecurity**
  - telecom call graphs
  - financial transaction graphs
  - malware or attack-propagation graphs
- **Internet of Things / Smart Cities**
  - transport networks
  - power grids
  - water networks
- **Bioinformatics**
  - gene sequencing
  - gene expression networks

### 1.3 What Makes Graph Analytics Different

Unlike ordinary tabular analytics, graph analytics studies:

- connectivity
- reachability
- influence
- clustering
- paths and flows
- propagation through links

So graph analytics is fundamentally about **relationships**, not just isolated records.

---

## 2. Real-World Graph Problems from the Slides

The lecture illustrates the value of graph analytics using three application domains.

### 2.1 Smart Cities and IoT

**Problem**:

- transport and utility-network analysis
- infrastructure optimization

**Challenges**:

- multiple data sources
- heterogeneous systems
- large distributed networks

**Graph problems**:

- Eulerian paths
- MaxCut
- centrality

### 2.2 Cybersecurity

**Problem**:

- detecting anomalies and bad actors in communication and transaction networks

**Challenges**:

- scale
- real-time behavior
- nontrivial patterns of suspicious interaction

**Graph problems**:

- belief propagation
- community analysis

### 2.3 Social Informatics

**Problem**:

- discovering communities
- analyzing spread of information
- understanding collective behavior

**Challenges**:

- uncertainty in data
- constantly evolving graph structure
- need for new analytics routines

**Graph problems**:

- clustering
- shortest paths
- flow computations

### 2.4 Big Picture

These examples show that graph processing is useful whenever the main signal lies in:

- who connects to whom
- how influence propagates
- which nodes are central or clustered
- which paths exist in a network

---

## 3. Common Families of Graph Algorithms

The lecture groups graph algorithms into three broad families.

### 3.1 Traversals

These study paths and flows through a graph.

Examples in the slide:

- Breadth-First Search (BFS)
- shortest path
- minimum spanning tree
- Eulerian paths
- MaxCut

### 3.2 Clustering

These study closeness or grouping among vertices.

Examples in the slide:

- community detection and evolution
- connected components
- k-means-style grouping
- maximal independent set

### 3.3 Centrality

These measure the importance of vertices in a network.

Examples in the slide:

- PageRank
- betweenness centrality

### 3.4 Why This Taxonomy Matters

This classification is useful for exams because it tells you what kind of problem a graph algorithm is solving:

- **traversal** asks where you can go
- **clustering** asks who belongs together
- **centrality** asks who matters most

---

## 4. Why Large-Scale Graphs Are Challenging

The slides strongly emphasize that graph processing is hard at scale.

### 4.1 Graphs Are Huge

The lecture gives representative graph sizes such as:

- Amazon graph: tens of millions of vertices and tens of millions of edges
- Wikipedia: tens of millions of vertices and hundreds of millions of edges
- Twitter-like graphs: tens of millions of vertices and billions of edges

The precise numbers matter less than the main point:

> Real graphs are large enough to require distributed processing.

### 4.2 Graphs Are Irregular

Real-world graphs are rarely uniform.

The slides note two key structural problems:

1. **Adjacency matrices are wasteful for sparse graphs**
2. **Power-law degree distributions create skew**

That means:

- some vertices have very few neighbors
- some vertices have extremely many neighbors
- load per worker can become highly uneven after partitioning

### 4.3 Graph Algorithms Have Uneven Execution Cost

The slides give BFS as the classic example:

- the frontier is small initially
- then it grows rapidly
- then it shrinks again

So the degree of parallelism varies over time.

### 4.4 Consequence for Distributed Systems

Because graph structure and algorithm cost are both irregular:

- balancing work is hard
- communication can be heavy
- naive partitioning can create hotspots

---

## 5. Why Traditional Models Are Not a Great Fit

### 5.1 Shared-Memory Algorithms Do Not Scale

The slides explicitly note that shared-memory graph algorithms do not scale to very large graphs.

Once the graph exceeds the memory or compute limits of one machine, the system needs distributed execution.

### 5.2 Why MapReduce / Spark Feel Awkward for Iterative Graph Algorithms

The lecture says graphs do not fit naturally into **MapReduce / Spark-style tuple-centric abstractions**.

Reasons:

- graph algorithms are inherently **iterative**
- this causes **multiple jobs** or repeated transformations
- the abstraction is tuple-based rather than graph-centric
- graph structure may need to be re-shuffled repeatedly

### 5.3 Why Graph Databases Are Different

The slides also distinguish **graph querying** from **graph analytics**.

Graph databases such as:

- Neo4j
- FlockDB
- 4Store
- Titan

are often designed for graph storage and query workloads, but not necessarily for large-scale iterative analytics on huge graphs.

### 5.4 Key Distinction

Graph databases are often optimized for:

- interactive traversal queries
- entity/relationship storage

Whereas large-scale graph analytics systems are optimized for:

- bulk iterative computation over very large graphs
- repeated propagation of state across edges

---

## 6. PageRank as the Motivating Example

### 6.1 What PageRank Measures

The slides introduce **PageRank** as a **centrality** metric.

It measures the importance of a vertex based on the importance of vertices linking to it.

### 6.2 Recursive Definition

The lecture uses the standard ideas:

- $P(n)$ = PageRank of vertex $n$
- $|G|$ = number of vertices in the graph
- $\alpha$ = random jump probability
- $L(n)$ = set of vertices linking to $n$
- $C(m)$ = out-degree of vertex $m$

The recursive intuition is:

$$
P(n) = \frac{\alpha}{|G|} + (1-\alpha) \sum_{m \in L(n)} \frac{P(m)}{C(m)}
$$

Each node's rank depends on incoming contributions from its neighbors.

### 6.3 Why PageRank Is Iterative

You usually do not compute PageRank in one shot.

Instead:

1. initialize ranks
2. send contributions along outgoing edges
3. aggregate incoming contributions
4. update ranks
5. repeat until convergence or fixed iteration count

This makes it an excellent example of an **iterative graph algorithm**.

---

## 7. Why PageRank in Spark Is Awkward

The slides show a Spark implementation and then explain its pain points.

### 7.1 Spark Implementation Pattern

The program maintains:

- adjacency information (`links`)
- current ranks (`ranks`)
- per-iteration contributions (`contribs`)

Each iteration repeatedly performs:

- `join`
- `flatMap`
- `reduceByKey`

### 7.2 Problems Highlighted in the Slide

#### Not Very Intuitive

Graph algorithms are iterative and relationship-based, but the Spark formulation looks like repeated manipulation of tuples and joins.

#### Poor Performance

The static graph structure is shuffled again and again through repeated joins.

In particular:

- one shuffle can happen to join graph structure with current values
- another shuffle can happen to aggregate incoming messages / rank values

### 7.3 Main Lesson

Spark can implement graph algorithms, but it is not the most natural abstraction for them.

This is the opening for a graph-specific model like **Pregel**.

---

## 8. Pregel: Google's Graph Processing Model

### 8.1 What Pregel Was Designed For

The slides summarize Pregel as:

- designed for **iterative graph algorithms**
- scalable and fault-tolerant
- flexible enough to express arbitrary graph algorithms

### 8.2 The Underlying Idea

Pregel is a **vertex-centric** system.

That means the programmer does not write the algorithm as:

- global joins over edges, or
- centralized graph traversals

Instead, the programmer answers:

> What should a single vertex do when it receives messages?

### 8.3 Inspiration from BSP

The paper and the slide both state that Pregel is inspired by **Valiant's Bulk Synchronous Parallel (BSP)** model.

That is the core execution model underlying Pregel.

---

## 9. Bulk Synchronous Parallel (BSP)

### 9.1 Core Pattern

The slides summarize BSP as:

```text
Compute -> Communicate -> Compute -> Communicate -> ...
```

### 9.2 Why Bulk Messaging Helps

Instead of many tiny asynchronous communications, BSP organizes communication into phases.

That helps amortize communication overhead and gives the system a clean global rhythm.

### 9.3 Supersteps

Pregel packages BSP into **supersteps**.

Each superstep has a logical structure:

1. active vertices compute
2. messages are sent
3. synchronization barrier
4. next superstep begins, using messages just delivered

### 9.4 Why This Matters

This model makes distributed graph computation easier to reason about because communication happens between synchronized rounds.

---

## 10. Vertex-Centric BSP in Pregel

### 10.1 Computation from the View of One Vertex

Pregel's central idea is **vertex-centric programming**.

In each superstep, every active vertex conceptually runs the same user-defined function in parallel.

### 10.2 What a Vertex Can Do

According to the slides, in superstep $S$, a vertex can:

- read messages sent in superstep $S-1$
- send messages to be read in superstep $S+1$
- modify its own state
- modify state of outgoing edges

### 10.3 Input and Output View

The slides depict the model as:

```text
Input graph --> supersteps over vertices --> vertices vote to halt --> Output graph
```

### 10.4 Important Consequence

This is still distributed computation, but the user writes only **local logic**.

The system handles:

- distribution
- synchronization
- message transport
- failure handling

---

## 11. Why the Vertex-Centric Model Is Attractive

The slides ask: what is the advantage?

### 11.1 Local Reasoning

Users focus on the behavior of a single vertex.

This is analogous to how `map` lets users think about one tuple at a time.

### 11.2 Independence and Parallelism

Each vertex processes its own local state and messages.

That makes large-scale parallelization natural.

### 11.3 Simpler Concurrency Semantics

The slides and paper emphasize that the synchronous model avoids many classic asynchronous-programming problems:

- deadlocks
- races due to unsynchronized shared state
- hard-to-reason-about interleavings

This is one of Pregel's major usability wins.

---

## 12. The Pregel Model of Computation from the Paper

The paper gives the formal model.

### 12.1 Input Graph

The input is a **directed graph** where:

- each vertex has a unique identifier
- each vertex has a mutable value
- each outgoing edge has a target vertex ID and mutable edge value

### 12.2 Graph State

Each vertex owns:

- its own value
- its outgoing edges
- incoming messages for the current round
- active/inactive status

### 12.3 High-Level Execution Sequence

Pregel computation consists of:

1. input loading
2. repeated supersteps
3. output generation

### 12.4 Important Modeling Choice

Edges are **not first-class computation entities**.

Computation is done at vertices, and edges mainly act as:

- relationship structure
- message routes
- holders of edge values

---

## 13. Supersteps and Message Passing Semantics

### 13.1 Round-Based Messaging

Messages sent during superstep $S$ are available to the destination vertex during superstep $S+1$.

This gives a clean temporal model:

- read old messages now
- send new messages for next round

### 13.2 Important Message Properties

The lecture and paper emphasize:

- no guaranteed message delivery order
- messages are delivered **exactly once**
- messages are not duplicated
- a vertex can send messages to any vertex whose ID is known
- typically messages are sent to neighbors, but that is not strictly required

### 13.3 Why Message Passing Was Chosen

The paper explicitly argues against remote reads / shared-memory emulation for two reasons:

1. message passing is expressive enough for graph algorithms
2. remote reads would incur high latency and be harder to optimize in a distributed cluster

### 13.4 Batching Benefit

Pregel can batch messages, which helps amortize network overhead.

---

## 14. Termination and the Vertex State Machine

### 14.1 Active and Inactive States

A vertex can be:

- **active**
- **inactive**

### 14.2 Superstep 0 Behavior

In superstep 0, every vertex starts active.

### 14.3 Vote to Halt

A vertex can deactivate itself by calling **vote to halt**.

This means:

- "I currently have no more work to do."

### 14.4 Reactivation by Message

If an inactive vertex later receives a message, it becomes active again.

### 14.5 Global Termination Condition

The algorithm stops when:

- all vertices are inactive, and
- no messages are in transit

This is the system-level stopping condition.

### 14.6 Why This Is Elegant

Termination is decentralized:

- each vertex makes local progress decisions
- the framework detects global quiescence

---

## 15. Vertex-Centric Programming Interface

### 15.1 Programming Idea

The programmer writes logic from the perspective of one vertex.

Vertices conceptually know about:

- their own value(s)
- their outgoing edges

### 15.2 Core User Function

In the paper's API, the user subclasses `Vertex` and overrides:

```cpp
virtual void Compute(MessageIterator* msgs) = 0;
```

### 15.3 What the Vertex API Exposes

The paper includes methods such as:

- `vertex_id()`
- `superstep()`
- `GetValue()` / mutable access to current value
- `GetOutEdgeIterator()`
- `SendMessageTo(...)`
- `VoteToHalt()`

The Giraph lecture API slide gives the same core flavor:

- `compute(msgs)`
- `getSuperstep()`
- `getVertexValue()`
- edge iterator access
- `sendMsg(...)`
- `sendMsgToAllEdges(...)`
- `voteToHalt()`

### 15.4 Why This API Is Powerful

The API is small, but sufficient for many algorithms because most graph routines can be expressed as repeated local updates plus message exchange.

---

## 16. The Maximum-Value Example

The slides illustrate Pregel using a simple "max vertex" algorithm.

### 16.1 Problem

Each vertex has a value. The goal is for every vertex in a connected region to learn the maximum value.

### 16.2 Logic

Each vertex:

1. checks incoming messages
2. updates its stored value if it sees a larger one
3. if changed, sends the larger value to all neighbors
4. votes to halt

### 16.3 Why This Example Is Good

It illustrates the full Pregel pattern:

- local state update
- message propagation
- repeated activation until convergence
- natural termination when no value changes anymore

---

## 17. Weakly Connected Components (WCC) via Label Propagation

The lecture presents the maximum-value code as a way to compute **weakly connected components**.

### 17.1 Idea

Initialize each vertex with some label, commonly its own ID.

Then repeatedly propagate the largest label seen.

Eventually, all vertices in the same weakly connected component converge to the same maximum label.

### 17.2 Slide Code Logic

The code in the slide does:

```cpp
hasChanged = (getSuperstep() == 0)
for each message m:
    if m > currentValue:
        currentValue = m
        hasChanged = true
if hasChanged:
    send currentValue to all neighbors
voteToHalt()
```

### 17.3 Why It Works

Within each component, the maximum label keeps propagating outward until every reachable vertex receives it.

### 17.4 WCC Intuition

This algorithm is effective because component membership can be reduced to a repeated local exchange of labels.

---

## 18. Shortest Path in Pregel

### 18.1 Problem

Given a source vertex, compute shortest path distance to all vertices.

### 18.2 Slide Logic

The slide algorithm initializes:

- source vertex distance = 0
- all others = `INF`

Then each vertex:

1. computes the minimum between its current value and incoming candidate distances
2. if a smaller value is found, updates itself
3. sends `(new distance + edge weight)` to each outgoing neighbor
4. votes to halt

### 18.3 Wavefront Interpretation

This creates a relaxation wave that propagates from the source through the graph.

Only vertices whose estimate improved continue sending updates.

### 18.4 What the Paper Adds

The paper explains that:

- the algorithm terminates when no vertex improves anymore
- if all edge weights are non-negative, the process converges correctly
- values remaining at `INF` indicate unreachable vertices

### 18.5 Why This Example Is Important

Shortest path is one of the clearest demonstrations that Pregel can express classical graph algorithms with very little code.

---

## 19. Breadth-First Search (BFS)

The slides also show BFS as a special case of shortest path.

### 19.1 Difference from Weighted Shortest Path

In BFS, the graph is effectively unweighted, so distance advances by exactly 1 level per edge.

### 19.2 Slide Logic

The slide code says:

- initialize source with value 0 and others with `INF`
- if a vertex is at `INF` and receives its first message, that message defines its level
- then it sends `level + 1` to its outgoing edges
- vote to halt

### 19.3 Key Optimization Insight

In BFS, only the **first visit** matters, because the first discovered level is already the shortest.

This makes BFS a clean frontier-expansion algorithm in Pregel.

---

## 20. PageRank in Pregel

### 20.1 Pregel PageRank Logic

The slide code and the paper describe this pattern:

1. initialize vertex rank to $1 / |V|$
2. in each superstep, sum incoming rank contributions
3. update vertex rank
4. divide rank by number of outgoing edges and send to all neighbors
5. stop after fixed iterations or convergence

### 20.2 Typical Update Rule

The paper version uses:

$$
\text{newRank} = \frac{0.15}{|V|} + 0.85 \cdot \text{sumIncomingContribs}
$$

### 20.3 Why Pregel Fits Better Than Repeated Spark Joins

Pregel naturally treats each vertex as the stateful entity and each edge as a message route.

So PageRank becomes:

- receive contributions
- update local rank
- send contributions

instead of repeated external joins between adjacency and rank datasets.

---

## 21. Combiners

### 21.1 Motivation

Sending many messages, especially across machines, is expensive.

The paper and the lecture both discuss **combiners** as an optimization.

### 21.2 Core Idea

If several messages headed to the same vertex can be merged safely, replace them with one combined message.

For example:

- in shortest path, if only the minimum tentative distance matters, keep the minimum
- in an additive algorithm, if only the sum matters, combine partial sums

### 21.3 Correctness Requirement

The combining operation must be:

- commutative
- associative

because Pregel does not guarantee which messages are combined or in what order.

### 21.4 Why It Matters

The paper reports that combiners can significantly reduce message traffic, and the lecture explicitly notes speedups such as around **4x** for shortest-path-style workloads.

### 21.5 Exam Interpretation

Combiners are an optimization for **communication cost**, not a change to the core model.

---

## 22. Aggregators

### 22.1 What Aggregators Are

Pregel aggregators provide a way to do **global communication / reduction** across vertices.

Each vertex contributes a value in superstep $S$, the system reduces them, and the result becomes available in superstep $S+1$.

### 22.2 Typical Uses

The paper explains that aggregators can be used for:

- global statistics
- convergence checks
- global coordination decisions
- selecting a special vertex
- maintaining counters or histograms

### 22.3 Examples

- sum of out-degrees gives total number of edges
- min or max over IDs can select a distinguished vertex
- a convergence metric can tell all vertices when to switch logic or terminate

### 22.4 Sticky Aggregators

The paper also mentions **sticky aggregators**, which can retain information across supersteps.

This is useful for global state that should persist unless explicitly changed.

### 22.5 Why Aggregators Matter

Pregel is mainly vertex-local, but some algorithms still need a little global coordination.

Aggregators provide exactly that without breaking the overall model.

---

## 23. Topology Mutations

### 23.1 Why They Are Needed

Some graph algorithms must modify graph structure itself.

Examples from the paper:

- clustering algorithms that merge vertices
- minimum spanning tree logic that removes edges

### 23.2 Supported Mutation Types

Pregel allows requests to:

- add vertices
- remove vertices
- add edges
- remove edges

### 23.3 Determinism Concern

Multiple vertices may issue conflicting mutation requests in the same superstep.

The paper handles this using:

- a partial ordering of mutation types
- user-defined conflict handlers where needed

### 23.4 Important Concept

These changes take effect in the next superstep, which preserves deterministic round semantics.

---

## 24. Input and Output Abstraction

The paper deliberately decouples graph computation from any one storage format.

### 24.1 Input

Graphs may come from:

- text files
- relational data
- Bigtable or other storage systems

### 24.2 Output

Outputs can be written in any format needed by the application.

### 24.3 Why This Design Matters

Pregel is a graph-computation framework, not just a file-format standard.

That keeps the model reusable across many real systems.

---

## 25. Pregel Architecture from the Paper

### 25.1 Basic Components

The paper describes:

1. **Master**
2. **Workers**
3. **Graph partitions**

### 25.2 Partitioning

The graph is divided into partitions, each containing:

- a set of vertices
- all outgoing edges of those vertices

The paper's default partitioning is based on:

$$
\text{hash(vertexID)} \bmod N
$$

where $N$ is the number of partitions.

### 25.3 Why Partitioning Matters

Partitioning determines:

- how vertices are distributed across workers
- how much cross-machine messaging occurs
- how balanced the workload is

### 25.4 More Than One Partition per Worker

The paper notes that assigning multiple partitions per worker can improve:

- load balance
- parallelism

---

## 26. Execution Flow in Pregel

The paper describes the main execution stages.

### 26.1 Worker and Master Startup

Many instances of the program start on a cluster.

One becomes the **master**, and the others become **workers**.

### 26.2 Graph Loading

Workers read input records and place vertices into their assigned partitions.

If a worker loads a vertex belonging elsewhere, it forwards the appropriate information.

### 26.3 Superstep Execution

For each superstep:

1. the master tells workers to execute
2. each worker processes active vertices
3. outgoing messages are buffered and sent
4. workers report completion and next-step activity

### 26.4 Halting

Execution continues while:

- some vertices remain active, or
- some messages are still in transit

### 26.5 Final Output

After halting, the graph or derived results can be written out.

---

## 27. Fault Tolerance and Checkpointing

### 27.1 Basic Strategy

Pregel uses **checkpointing** for fault tolerance.

At checkpoint boundaries, workers save:

- vertex values
- edge values
- incoming messages

The master also saves aggregator state.

### 27.2 Failure Detection

The master and workers use heartbeat-like ping logic to detect failures.

### 27.3 Recovery Process

If a worker fails:

1. its partition state is considered lost
2. the master reassigns partitions
3. workers reload from the latest checkpoint
4. missing supersteps are re-executed

### 27.4 Confined Recovery

The paper also discusses a more advanced idea: **confined recovery**.

Instead of recomputing everything globally, the system can try to recover only the lost partitions while preserving others.

### 27.5 Determinism Requirement

For confined recovery to work safely, the algorithm should be deterministic.

Randomized algorithms can still be made deterministic by consistent seeding.

---

## 28. Worker Implementation Details

### 28.1 In-Memory Graph State

Each worker stores its assigned graph portion in memory.

Conceptually, for each vertex it stores:

- current vertex value
- outgoing edges and edge values
- incoming message queue
- active / inactive flag

### 28.2 Double Buffering of State for Supersteps

The paper explains that workers keep separate structures for:

- current superstep state
- next superstep state

This is necessary because while superstep $S$ is being computed, messages for $S+1$ are already arriving.

### 28.3 Local vs Remote Messaging

If destination vertex is on another worker:

- buffer the message for remote delivery

If destination vertex is local:

- directly place it into the local incoming queue for the next superstep

### 28.4 Performance Rationale

This design allows:

- asynchronous communication
- batching
- overlap of computation and communication

---

## 29. Master Implementation

### 29.1 What the Master Tracks

The master maintains:

- worker registrations
- partition assignments
- superstep progress
- graph statistics
- aggregator values

### 29.2 Barrier Synchronization

Most master-controlled operations use barriers.

The master sends a request to all relevant workers and waits for all of them to respond before moving on.

### 29.3 Monitoring Support

The paper notes that the master provides monitoring and statistics such as:

- graph size
- out-degree distribution
- number of active vertices
- timing and message traffic across supersteps

This helps both debugging and operational use.

---

## 30. Giraph: Open-Source Pregel-Style System

The slides next move from Google's Pregel paper to **Apache Giraph**.

### 30.1 What Giraph Is

Giraph is an open-source system that implements the Pregel abstraction.

The lecture highlights:

- vertex-centric model
- iterative BSP computation
- Hadoop ecosystem integration

### 30.2 Historical Note in the Slides

The slides list multiple Giraph releases to show that the abstraction became a maintained open-source system over time.

### 30.3 Why Giraph Matters in the Course

Pregel itself is a Google system and paper.

Giraph is important because it shows how the same model was made available in the open-source Hadoop ecosystem.

---

## 31. Giraph Architecture

The lecture gives a concise architecture summary.

### 31.1 Hadoop Map-Only Application Style

Giraph is presented as running over the Hadoop ecosystem, often using a map-only style of job execution.

### 31.2 ZooKeeper

ZooKeeper is responsible for computation state such as:

- partition-to-worker mapping
- global superstep number

### 31.3 Master

The master coordinates:

- partition assignment
- synchronization across workers

### 31.4 Worker

Workers are responsible for:

- holding vertices
- executing `compute()` on active vertices
- sending / receiving messages
- assigning messages to the right local vertices

### 31.5 Checkpointing

The slides also explicitly mention checkpointing of supersteps in Giraph.

So the open-source implementation preserves the fault-tolerant superstep model.

---

## 32. Partitioner in Giraph / Pregel-Style Systems

### 32.1 Purpose

The partitioner maps vertices to partitions operated by workers.

### 32.2 Default Strategy

The slide says the default is a **hash partitioner**.

### 32.3 Dynamic Migration

The slide also notes that partitioning logic may be invoked at the end of each superstep for dynamic migration.

### 32.4 Why This Matters

Partitioning is one of the main performance levers in large-scale graph processing because it affects:

- locality
- communication cost
- load balance

---

## 33. PageRank, Shortest Path, BFS, and WCC in One Unified Lens

These four lecture algorithms illustrate how a wide range of graph problems can fit the same programming model.

### 33.1 WCC

- propagate labels
- converge to a component identifier

### 33.2 BFS

- propagate frontier depth
- first visit determines level

### 33.3 Shortest Path

- propagate improved distance estimates
- only improved vertices keep sending updates

### 33.4 PageRank

- propagate rank mass repeatedly
- aggregate incoming contributions

### 33.5 Main Lesson

Different graph problems look different globally, but Pregel reduces them to the same template:

```text
receive messages -> update local state -> send new messages -> vote/halt
```

---

## 34. Bipartite Matching from the Paper

The paper gives a more advanced example: bipartite matching.

### 34.1 Problem

Given two disjoint sets of vertices with edges only across sets, find a set of non-overlapping matches.

### 34.2 Pregel View

The paper describes a randomized maximal matching algorithm implemented as a multi-phase handshake across supersteps.

### 34.3 Why This Example Matters

It shows that Pregel is not limited to simple propagation tasks. It can also express structured, multi-phase coordination patterns.

---

## 35. Semi-Clustering from the Paper

The paper also presents **semi-clustering**.

### 35.1 What Semi-Clustering Means

A vertex may belong to more than one cluster candidate.

This is useful in social graphs where community boundaries may overlap.

### 35.2 Scoring Function

The paper defines a semi-cluster score using:

- internal edge weight sum
- boundary edge weight sum
- cluster size

The score favors groups that are densely connected internally and relatively lightly connected to the outside.

### 35.3 Why This Example Matters

It shows Pregel can handle more sophisticated graph-mining tasks, not just classical shortest path or PageRank.

---

## 36. K-Means Clustering Slide

The lecture also gives a high-level k-means-on-graphs style example.

### 36.1 Multiple Phases Mentioned in the Slide

- choose $k$ centers
- assign vertices to clusters
- find edge cuts

### 36.2 Mechanisms Mentioned

- multi-source BFS or Euclidean distance for assigning nearest cluster
- `MasterCompute` for global coordination such as:
  - selecting initial vertices
  - calculating edge-cut count
  - deciding termination

### 36.3 Why This Slide Matters

It reinforces that Pregel/Giraph programs often combine:

- local vertex logic
- message propagation
- occasional global coordination

---

## 37. Advantages of Pregel-Style Systems

The slides summarize several advantages.

### 37.1 Easier Distributed Programming

Pregel avoids many explicit concurrency headaches:

- no locks
- no semaphores
- fewer race conditions

### 37.2 Clear Separation of Compute and Communicate

The BSP superstep model provides a disciplined structure.

### 37.3 Vertex-Level Parallelization

The abstraction exposes very high parallelism when many vertices are active.

### 37.4 Stateful In-Memory Computation

The slides note that primarily:

- messages and checkpoints hit disk
- core vertex state is in memory

This supports efficient iterative execution.

---

## 38. Limits and Trade-Offs of Pregel

The paper's conclusion section is explicit that Pregel is powerful, but not universally perfect.

### 38.1 Synchronization Cost

Because Pregel is synchronous, fast workers may need to wait at barriers for slower workers.

### 38.2 Memory Pressure

The paper notes that the full computation state primarily lives in RAM.

For extremely large graphs, spilling or more advanced storage techniques may be needed.

### 38.3 Partitioning Challenges

Good partitioning is difficult, especially when communication patterns do not align cleanly with graph topology.

### 38.4 Sparse-Graph Focus

Pregel is best suited to sparse graphs where communication mostly follows edges.

Dense all-to-all communication patterns are a poor fit.

---

## 39. Experiments and Performance Results

The paper evaluates Pregel primarily using **single-source shortest path (SSSP)**.

### 39.1 Setup Style

Experiments were run on a large cluster of multicore commodity PCs.

### 39.2 Scaling with Workers

The paper reports that as the number of worker tasks increases, runtime drops substantially for very large graphs.

### 39.3 Scaling with Graph Size

The paper also shows runtime growth with increasing graph size, both on:

- binary trees
- more realistic random graphs with heavy-tailed degree distributions

### 39.4 Key Takeaway

Pregel demonstrates practical large-scale performance on graphs with:

- hundreds of millions or billions of vertices
- tens to hundreds of billions of edges

### 39.5 Important Interpretation

The paper itself says the experiments are meant to show **satisfactory performance with relatively little coding effort**, not necessarily fully hand-tuned best possible graph-processing speed.

---

## 40. Pregel vs MapReduce / Spark for Graph Processing

### 40.1 MapReduce / Spark Style

- graph treated as tuples or datasets
- repeated joins and shuffles for iterative algorithms
- graph structure often reprocessed externally each round

### 40.2 Pregel Style

- graph treated natively as vertices and edges
- local vertex state preserved in memory
- communication via messages along graph structure
- iterative supersteps built into the model

### 40.3 Main Trade-Off

Pregel is more specialized than general-purpose dataflow systems, but that specialization makes iterative graph algorithms much more natural.

---

## 41. Pregel vs Graph Databases

### 41.1 Graph Databases

Best for:

- storage of graph-structured data
- query / traversal workloads
- transactional or interactive use cases

### 41.2 Pregel

Best for:

- distributed iterative analytics on huge graphs
- bulk graph algorithms
- repeated propagation-style computations

### 41.3 Exam Distinction

Do not confuse:

- a **graph database** for querying graph data, with
- a **distributed graph analytics engine** for large-scale iterative computation

---

## 42. Pregel vs Asynchronous Systems

The paper positions Pregel as a synchronous alternative to more asynchronous distributed graph processing designs.

### 42.1 Pregel Advantage

- simpler reasoning
- deterministic superstep semantics
- fewer races and deadlocks
- natural barrier-based structure

### 42.2 Cost

- barrier waiting can reduce efficiency if load is imbalanced

### 42.3 Why the Trade-Off Was Worth It

The paper argues that graph computations often have enough parallel slack that the synchronization cost is acceptable.

---

## 43. Why Pregel Was Successful

The paper's conclusion is not just theoretical. It states that many applications were already deployed on Pregel.

### 43.1 User Experience Reported in the Paper

Users found that once they learned to **think like a vertex**, the API became:

- intuitive
- flexible
- easy to use

### 43.2 Important Supporting Features Mentioned

The paper mentions practical tooling such as:

- status pages
- unit testing support
- single-machine mode for prototyping and debugging

This matters because system success depends on usability, not just performance.

---

## 44. High-Yield Exam Distinctions

### 44.1 Vertex-Centric vs Tuple-Centric

- **Vertex-centric**: write logic from a vertex's point of view
- **Tuple-centric**: write transformations over rows / key-value pairs

### 44.2 Superstep vs Streaming Event Processing

- **Pregel superstep**: synchronous iterative round over a graph
- **stream processing micro-batch / event processing**: processing over time-arriving events

### 44.3 Pregel vs Spark Graph Code

- Spark graph code often uses repeated joins and shuffles
- Pregel keeps graph structure local and sends messages instead

### 44.4 WCC vs BFS vs Shortest Path vs PageRank

- **WCC**: propagate labels to identify components
- **BFS**: find minimum hop distance
- **Shortest path**: find weighted path distance
- **PageRank**: compute recursive centrality scores

### 44.5 Combiners vs Aggregators

- **Combiner**: merge multiple messages for the same destination to reduce communication
- **Aggregator**: compute a global reduced value available to all vertices next round

### 44.6 Vote to Halt

- local vertex says it has no more current work
- vertex can still be reactivated by a later message

---

## 45. One-Page Mental Model of Pregel

```text
Graph = vertices + outgoing edges + per-vertex state

Each superstep:
1. active vertices read last round's messages
2. each vertex updates its own state
3. vertices send messages for next round
4. vertices may vote to halt
5. barrier synchronization

Terminate when:
- all vertices are inactive
- no messages remain in transit
```

This is the cleanest way to remember the whole model.

---

## 46. Coverage Checklist Against Slides and Paper

### 46.1 Lecture Slides (69-109)

- [x] graphs are commonplace across web, knowledge graphs, security, IoT, and bioinformatics
- [x] graph problems in smart cities, cybersecurity, and social informatics
- [x] families of graph algorithms: traversals, clustering, centrality
- [x] scale and irregularity challenges of large graphs
- [x] uneven execution cost in graph algorithms
- [x] limits of shared-memory, MapReduce/Spark, and graph databases for large analytics
- [x] PageRank intuition and recursive definition
- [x] Spark PageRank implementation pattern
- [x] Spark graph-processing challenges
- [x] Pregel introduction and BSP motivation
- [x] bulk synchronous parallel model
- [x] vertex-centric BSP and supersteps
- [x] advantages of vertex-centric approach
- [x] Pregel model of computation and vote-to-halt idea
- [x] vertex state machine
- [x] vertex-centric programming perspective
- [x] max vertex example
- [x] WCC example and code
- [x] shortest-path example and code
- [x] PageRank in Pregel and code
- [x] Giraph API slide
- [x] advantages slide
- [x] message-passing slide
- [x] Giraph architecture
- [x] checkpointing slide
- [x] partitioner slide
- [x] combiners slide
- [x] k-means clustering slide
- [x] BFS slide and example

### 46.2 Pregel Paper Coverage

- [x] motivation for scalable graph processing
- [x] limits of custom systems, MapReduce, sequential graph libraries, and earlier parallel libraries
- [x] Pregel computation model
- [x] supersteps and message passing
- [x] vertex state machine and termination
- [x] C++ vertex API foundations
- [x] message-passing semantics
- [x] combiners
- [x] aggregators
- [x] topology mutations
- [x] input and output abstraction
- [x] partitioning and basic architecture
- [x] fault tolerance and checkpointing
- [x] confined recovery
- [x] worker implementation
- [x] master implementation
- [x] PageRank application
- [x] shortest-path application
- [x] bipartite matching
- [x] semi-clustering
- [x] experimental results and scaling
- [x] related-work positioning
- [x] conclusions and limitations

### 46.3 Final Check

All topics requested for the Pregel portion of Module 5 have been covered. The notes intentionally connect the lecture slides and the paper so the model, the API, the system architecture, and the example algorithms are all explained in one continuous narrative.