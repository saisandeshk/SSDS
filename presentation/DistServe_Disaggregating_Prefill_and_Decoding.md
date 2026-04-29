# DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving

## Paper Presentation Notes
### DS256 / Scalable Systems for Data Science - Module 3 Connection

> **Paper**:
> - Yinmin Zhong et al., "DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving", 2024
>
> **Most relevant course connection**:
> - Module 3 lecture on Transformer inference and ORCA serving

---

## 1. One-line Summary

**Main idea**: DistServe argues that **prefill** and **decode** should not share the same GPUs when we care about latency guarantees. Instead, the system **disaggregates** them onto different GPU groups, then gives each phase its **own batching policy, parallelism strategy, and resource allocation**, which improves the number of requests served per GPU while still meeting both latency targets.

Another way to say it:

- ORCA-style systems ask: **How should we schedule requests well on shared GPUs?**
- DistServe asks a deeper question: **Should prefill and decode even be on the same GPUs at all?**

That architectural shift is the paper's core contribution.

---

## 2. What Problem Is the Paper Solving?

The paper is about **online LLM serving under latency SLOs**.

When a user sends a prompt to an LLM service, the service usually has two latency goals:

- **TTFT (Time To First Token)**: how long the user waits before seeing the first output token
- **TPOT (Time Per Output Token)**: how long each later token takes during generation

Different applications care about these differently:

- A chatbot needs **small TTFT** so it feels responsive.
- A summarization system can tolerate a slower first token, but wants **small TPOT** so long answers are generated quickly.
- A coding assistant wants **both** to be fairly tight.

The paper argues that most prior systems optimize the wrong top-level metric.

- They focus on **throughput**: total tokens/sec.
- DistServe focuses on **goodput**: how many requests/sec can be served **while still satisfying latency SLOs**.

This is a very important distinction.

### 2.1 Throughput vs. Goodput

**Throughput** means the system is producing lots of tokens.

But if many requests miss latency targets, that raw throughput is not very useful for a user-facing service.

So the paper defines **per-GPU goodput** as the maximum request rate per GPU such that a target fraction of requests (for example, 90%) still meets both the TTFT and TPOT SLOs.

In words:

$$
\text{goodput} = \max \{\text{req/s per GPU} : \text{SLO attainment} \ge \alpha\}
$$

where:

- $\alpha$ is the target SLO attainment, such as 90%
- a request counts only if it meets both TTFT and TPOT targets

**Intuition**: If a system is fast on average but frequently makes users wait too long, then it has high throughput but poor goodput.

---

## 3. Background You Need Before Reading the Paper

### 3.1 LLM Inference Happens in Two Phases

When a prompt arrives, LLM inference is not one uniform computation.

It naturally splits into:

1. **Prefill**
2. **Decode**

### 3.2 Prefill

During prefill, the model reads the entire prompt and computes the first output token.

Example:

```text
Prompt: "Summarize this 3-page article"

Prefill work:
- read all prompt tokens
- run them through all transformer layers
- build KV cache for those tokens
- produce the first output token
```

Prefill usually has these properties:

- many input tokens are processed together
- a lot of arithmetic is done in parallel
- it is often **compute-bound**, especially for long prompts

### 3.3 Decode

After the first token, generation continues one token at a time.

Example:

```text
Token 1: "The"
Token 2: "article"
Token 3: "argues"
...
```

Each decode step:

- processes only one new token per request
- reuses the saved KV cache from earlier tokens
- is often **memory-bandwidth-bound**, not compute-bound

### 3.4 KV Cache

The **KV cache** stores the key/value tensors for previous tokens so the model does not recompute them every step.

This is necessary for autoregressive generation, but it creates a systems challenge:

- the cache gets large
- it must be kept in GPU memory
- if moved between GPUs, transfer cost matters

### 3.5 Why the Two Phases Feel So Different

An intuitive analogy:

- **Prefill** is like reading the whole question and building context.
- **Decode** is like speaking the answer word by word.

Those are not the same kind of work.

So DistServe asks:

**Why should we force both onto the same hardware and the same scheduling policy?**

---

## 4. Why Existing Colocated Systems Are Not Enough

Most prior LLM serving systems keep both phases on the same GPUs.

That is called **colocation**.

They then use batching to maximize hardware utilization.

This sounds reasonable, but DistServe says it creates two major problems:

1. **Prefill-decode interference**
2. **Resource and parallelism coupling**

### 4.1 Problem 1: Prefill-Decode Interference

Suppose one request has a long prompt and another request is already decoding.

If they share the same GPU batch:

- the long prefill step takes much longer
- the decode step gets delayed behind it
- TPOT becomes much worse

At the same time, adding decode work into a prefill-heavy batch also hurts TTFT.

So the phases damage each other's latency.

### 4.2 Intuitive Example

Imagine a restaurant kitchen where:

- one station handles **ingredient prep**
- the same station also handles **final plating**

Now a huge prep task arrives. Small plating tasks that should have finished quickly get stuck waiting.

That is exactly the paper's point:

- **prefill** is the big prep task
- **decode** is the small but latency-sensitive plating task

Sharing the same station creates interference.

### 4.3 Problem 2: Coupled Resource Decisions

Even worse, colocated systems force both phases to share:

- the same GPU pool
- the same model parallelism decisions
- the same memory capacity budget

But the two phases want different things.

**Prefill wants**:

- low execution time
- often smaller batches
- sometimes more aggressive intra-op parallelism

**Decode wants**:

- large batches to avoid low utilization
- enough memory for many active requests and KV caches
- a different scaling policy

If both phases are glued together, the system cannot optimize them independently.

### 4.4 Why Chunked Prefill Is Not a Full Fix

The paper also discusses **chunked prefill with piggyback** (for example, SARATHI / DeepSpeed-MII style ideas).

The idea is:

- split a long prefill into smaller chunks
- sneak decode tokens into the same batch

This helps somewhat, but DistServe argues it still does not solve the root problem.

Why?

1. Decode still competes with prefill for the same GPU.
2. Prefill still gets slowed by decode work.
3. If chunks are too small, GPU utilization drops.
4. If chunks are too large, piggyback opportunities shrink.
5. Chunking increases KV-cache rereads.

The paper makes a strong systems point here:

**Chunking is a scheduling workaround; disaggregation is an architectural fix.**

---

## 5. The Key Insight of DistServe

DistServe proposes to **disaggregate** the two phases.

That means:

- **prefill runs on prefill instances**
- **decode runs on decoding instances**
- the intermediate state, mainly KV cache plus first token information, is transferred from prefill to decode

### 5.1 What Is an "Instance"?

In the paper, an **instance** is a resource unit that holds one full copy of model weights.

Important detail:

- one instance may use **multiple GPUs** if model parallelism is applied

So an instance is not necessarily one GPU. It is one serving unit for one copy of the model.

### 5.2 Request Flow in DistServe

```text
Incoming request
    |
    v
Prefill instance
    - reads prompt
    - computes first token
    - builds KV cache
    |
    | transfer KV cache + first-token state
    v
Decoding instance
    - continues token-by-token generation
    |
    v
Return full output to client
```

### 5.3 Why This Helps

Because once the phases are separated:

- prefill no longer blocks decode on the same GPU
- decode can batch many active requests without worrying about long prompt processing
- each phase can use different parallelism strategies
- GPU allocation can match each phase's own bottleneck

This is the core reason DistServe improves **goodput**, not just raw throughput.

---

## 6. A Concrete Example from the Paper

The paper gives a simple motivating example with a 13B model on A100 GPUs.

Under latency constraints and 90% SLO attainment:

- a colocated system achieves about **1.6 req/s per GPU**
- a prefill-only setup can achieve about **5.6 req/s per GPU**
- a decode-only setup can achieve about **10 req/s per GPU**

The paper then argues:

- if we use **2 GPUs for prefill** and **1 GPU for decode**
- overall goodput becomes about **10 req/s total**
- which is about **3.3 req/s per GPU**

That is about **2.1x better** per GPU than the colocated design.

This example captures the entire paper in one picture:

**the bottleneck is not just computation; it is the bad coupling between two very different computations.**

---

## 7. DistServe's Tradeoff Analysis

The paper does not stop at saying "separate the phases." It also asks:

**Once we separate them, how should each phase be configured?**

That leads to one of the most important parts of the paper: the analysis of **prefill-only** and **decode-only** instances.

### 7.1 Prefill Instance Analysis

After disaggregation, a prefill instance only needs to optimize **TTFT**.

#### Batching behavior

Prefill is often compute-intensive.

The paper shows that for long enough prompts, even **one request can already saturate the GPU**. After that point, adding more requests to the batch does not help much and may only increase total batch time.

The paper defines a threshold $L_m$:

- if prompt length is above $L_m$, the GPU is already near compute saturation
- then large prefill batches are usually not a good idea

**Intuition**: If one big prompt already keeps the GPU fully busy, batching more prompts only makes everyone wait longer.

#### Parallelism behavior

For prefill, the paper compares:

- **intra-op parallelism**: split heavy operators across GPUs
- **inter-op parallelism**: pipeline layers/stages across GPUs

The paper's key observation is:

- at **low arrival rates** or under **very tight TTFT SLOs**, intra-op is often better because it reduces the service time of each request
- at **higher arrival rates**, inter-op can become better because it increases effective rate capacity and reduces queuing pressure

The paper models this with queuing theory and shows that the best choice depends on both:

- execution time
- queuing delay

So there is no single "best" parallelism plan for prefill.

### 7.2 Decode Instance Analysis

After disaggregation, a decoding instance only needs to optimize **TPOT**.

Decode has the opposite character:

- one decode job by itself underutilizes the GPU
- batching is very important
- the phase is mostly memory-bandwidth-bound

This is where disaggregation becomes especially powerful.

In a colocated design, making decode batches larger often hurts TTFT, because prefill and decode compete on the same GPUs.

In DistServe, decode runs alone, so the system can:

- batch many decode requests together
- improve GPU utilization
- raise throughput in the decode phase
- do this **without damaging prefill TTFT**

This is one of the cleanest benefits of separation.

### 7.3 Communication Overhead

Disaggregation is not free.

You must transfer the KV cache from prefill to decode.

The paper gives a concrete example:

- for OPT-66B with a 512-token request, KV cache is about **1.13 GB**
- at 10 requests/sec, that means about **11.3 GB/s**, or about **90 Gbps** to hide the transfer overhead

That sounds large, but the paper argues modern clusters often have enough bandwidth:

- high-end InfiniBand can handle it
- inside a node, NVLINK is much faster

So the communication cost is real, but manageable if placement is done carefully.

---

## 8. DistServe's Placement Problem

Once prefill and decode are separated, the system must decide:

1. How many prefill instances to deploy
2. How many decode instances to deploy
3. Which parallelism each type should use
4. Where those instances should be placed in the cluster

The paper calls this overall decision a **placement**.

### 8.1 Why Placement Is Hard

The answer depends on several interacting factors:

- model size
- prompt length distribution
- output length distribution
- arrival rate
- TTFT SLO
- TPOT SLO
- SLO attainment target
- GPU memory size
- intra-node bandwidth
- cross-node bandwidth

This is too complicated for one simple analytic formula.

So DistServe uses **simulation plus search**.

---

## 9. High-Bandwidth Placement Algorithm

The paper first considers a cluster where cross-node communication is fast enough that KV-cache transfer across nodes is not a serious problem.

In that case, DistServe does a two-level optimization.

### 9.1 Step 1: Optimize Prefill Separately

It enumerates feasible prefill parallelism configurations and estimates, via simulation, the maximum rate that still meets the TTFT SLO attainment target.

### 9.2 Step 2: Optimize Decode Separately

It does the same for decoding, but now with respect to TPOT.

### 9.3 Step 3: Replicate to Reach Total Traffic

Once the best per-instance configuration for each phase is found, DistServe replicates those instances as needed to serve the target request rate.

### 9.4 Why a Simulator Is Needed

The paper explicitly says that real workloads are messy:

- prompt lengths vary
- output lengths vary
- arrivals are irregular

So instead of deriving everything by formula or profiling every candidate in the real cluster, DistServe:

- fits workload distributions from historical traces
- resamples traces
- simulates each candidate placement
- uses binary search to find the highest rate that still satisfies the SLO attainment target

The simulator uses latency models based on:

- FLOPs
- memory accesses
- communication behavior

This is a classic systems move: **approximate the design space fast enough to search it**.

---

## 10. Low-Bandwidth Placement Algorithm

The paper also handles a more realistic case: cross-node bandwidth is limited.

In that case, blindly putting prefill on one node and decode on another may make KV transfer too slow.

### 10.1 Key Insight

KV cache transfer happens between **corresponding model stages**.

So if model parallelism divides the model into stages, DistServe can colocate:

- the prefill stage segment
- and the matching decode stage segment

on the same node.

That way, transfer uses **intra-node NVLINK** instead of slow cross-node links.

### 10.2 Intuitive Picture

```text
Node 1: Prefill Stage 1 + Decode Stage 1
Node 2: Prefill Stage 2 + Decode Stage 2
Node 3: Prefill Stage 3 + Decode Stage 3
...
```

This is clever because it preserves disaggregation **at the phase level** while still making communication local **at the stage level**.

Important clarification:

- **same node** does **not** mean **same instance**
- it also does **not** mean prefill and decode are forced onto the exact same GPUs
- it means the node hosts a **prefill segment** and the **matching decode segment** side by side, typically on different GPU subsets of that node

So DistServe is still disaggregated in the architectural sense:

- prefill and decode remain different serving instances
- they keep separate queues and separate scheduling roles
- they still communicate by transferring KV cache from prefill to decode

What changes in the low-bandwidth case is that the placement becomes **more constrained**. DistServe can no longer place the two phases arbitrarily across the cluster; it must align matching stages so the KV-cache handoff uses fast local links.

This is why the low-bandwidth design is best viewed as **constrained disaggregation**, not as a return to the fully colocated design.

### 10.3 Why This Matters

This part of the paper is important because it shows that DistServe is not just an abstract architecture idea. It is aware of real cluster topology.

The paper's message is:

**disaggregation only works well if you also respect the network.**

One subtle detail from the paper: in the low-bandwidth algorithm, the design space is indeed narrower than in the ideal high-bandwidth case. Matching prefill/decode stages must be colocated, and the placement search must respect per-node GPU limits. So this version gives up some freedom in exchange for much cheaper KV-cache transfer.

---

## 11. Runtime System Design

DistServe is not only an offline planner. It also provides an online serving runtime.

### 11.1 High-Level Architecture

```text
Requests
   |
   v
Central controller
   |
   +--> Prefill instance with shortest queue
            |
            | KV cache + first-token state
            v
       Least-loaded decoding instance
            |
            v
         Stream output
```

### 11.2 Scheduling Policy

The online scheduler is intentionally simple:

- requests first go to the prefill instance with the shortest queue
- then they are assigned to the least-loaded decode instance

The paper uses **FCFS** as the baseline runtime scheduling policy.

This simplicity is a feature: DistServe's main contribution is the architecture and placement optimization, not a very complicated online scheduler.

### 11.3 Reducing Pipeline Bubbles

Real prompts are not all the same length.

That creates **pipeline bubbles**, especially for inter-op parallelism, because some stages finish earlier than others.

DistServe tries to reduce bubbles by batching requests so that each batch has a balanced amount of new-token work.

For prefill:

- batch multiple short prompts together until the total length is near the GPU-saturating threshold $L_m$
- if one prompt is already long enough, run it alone

For decode:

- use the largest appropriate batch size to keep the GPU busy

### 11.4 Handling Bursty Traffic with Pull-based KV Transfer

This is a nice systems detail.

If many prefill jobs finish at once, they may all try to push large KV caches to decode instances and overwhelm decode memory.

So DistServe uses **pull**, not push:

- prefill stores the KV cache temporarily
- decode fetches it when ready

This makes prefill memory act like a buffer queue.

That is a simple but practical design choice.

### 11.5 Replanning

Workloads change over time.

So DistServe periodically monitors:

- average arrival rate
- average input length
- average output length

If the workload shifts enough, DistServe reruns its placement search.

This is important because the "best" prefill/decode split is workload-dependent.

---

## 12. Implementation Details

The system is fully implemented, not just simulated.

### 12.1 Codebase Size

- about **6.5K lines of Python** for placement, frontend, and orchestration
- about **8.1K lines of C++/CUDA** for the parallel execution engine

### 12.2 Major Components

- placement algorithm module
- OpenAI-compatible REST frontend
- orchestration layer
- parallel execution engine

### 12.3 Communication Mechanisms

- **NCCL** for cross-node GPU communication
- **asynchronous CudaMemcpy** for intra-node transfer

The asynchronous transfer detail matters because it avoids blocking GPU computation while communication happens.

### 12.4 Modern LLM Optimizations Used

DistServe is not reinventing all inference optimizations from scratch. It integrates existing techniques such as:

- continuous batching
- FlashAttention
- PagedAttention

This is an important presentation point:

**DistServe complements lower-level inference optimizations. It does not replace them.**

---

## 13. Evaluation Setup

### 13.1 Cluster

- 4 nodes
- 32 A100 80GB GPUs total
- 8 GPUs per node with NVLINK inside the node
- cross-node bandwidth: 25 Gbps

Because cross-node bandwidth is limited, the paper mainly evaluates the bandwidth-aware low-bandwidth placement algorithm.

### 13.2 Models

- OPT-13B
- OPT-66B
- OPT-175B

The paper deliberately uses OPT with standard multi-head attention to put real pressure on KV-cache transfer. It also notes that newer GQA/MQA-style models would likely make DistServe look even better because they shrink KV cache size.

### 13.3 Workloads and SLOs

| Application | Model | TTFT SLO | TPOT SLO | Dataset |
|-------------|-------|----------|----------|---------|
| Chatbot | OPT-13B | 0.25 s | 0.10 s | ShareGPT |
| Chatbot | OPT-66B | 2.5 s | 0.15 s | ShareGPT |
| Chatbot | OPT-175B | 4.0 s | 0.20 s | ShareGPT |
| Code completion | OPT-66B | 0.125 s | 0.20 s | HumanEval |
| Summarization | OPT-66B | 15 s | 0.15 s | LongBench |

The paper uses Poisson arrivals because the datasets do not contain timestamps.

---

## 14. Main Results

### 14.1 Headline Result

Across models and applications, DistServe can achieve up to:

- **7.4x higher request rate**, or
- **12.6x tighter SLOs**

while still satisfying the latency constraints for more than 90% of requests.

That is the main empirical takeaway.

### 14.2 Chatbot Results

On ShareGPT:

- DistServe sustains about **2.0x to 4.6x** higher request rate than vLLM
- DistServe sustains about **1.6x to 7.4x** higher request rate than DeepSpeed-MII
- DistServe handles about **1.8x to 3.2x** tighter SLOs than vLLM

Why?

- chatbots care strongly about responsiveness
- TPOT violations from shared prefill/decode execution hurt colocated systems badly
- DistServe isolates decode from long prefill jobs

### 14.3 Code Completion Results

For OPT-66B on HumanEval:

- DistServe sustains about **5.7x** higher request rate than vLLM
- DistServe sustains about **1.6x** higher request rate than DeepSpeed-MII
- DistServe handles about **1.4x** tighter SLOs than both baselines

The paper explains that code completion is very TTFT-sensitive, so DistServe benefits from being able to make prefill faster with its own parallelism choices.

### 14.4 Summarization Results

For OPT-66B on LongBench:

- DistServe achieves about **4.3x** higher request rate than vLLM
- DistServe achieves about **1.8x** higher request rate than DeepSpeed-MII
- DistServe handles about **12.6x** tighter SLOs than vLLM
- DistServe handles about **2.6x** tighter SLOs than DeepSpeed-MII

This setting has long prompts, so prefill is heavy, but TTFT is relatively loose. That makes TPOT especially important. Colocated systems suffer here because long prefill work damages decode latency.

---

## 15. Is Communication Overhead a Deal Breaker?

The paper directly checks this.

For OPT-175B on ShareGPT, the latency breakdown shows:

- KV-cache transmission is **less than 0.1%** of total latency
- more than **95%** of requests see transfer delay under **30 ms**

This is one of the paper's most important validation points.

The entire DistServe idea would be weak if communication dominated. The experiments show it usually does not, provided placement is bandwidth-aware.

---

## 16. Ablation and Additional Findings

### 16.1 Disaggregation Matters More Than Just Better Tuning

The paper creates a stronger baseline, "vLLM++", which tries different parallelism settings instead of just using vLLM defaults.

Result:

- vLLM++ ends up essentially the same as vLLM in their setting

Interpretation:

**The main problem is not that prior systems picked the wrong parallelism. The main problem is that prefill and decode were still colocated.**

### 16.2 Simulator Accuracy

The simulator's SLO-attainment estimates differ from the real system by less than 2% in the experiments shown.

That supports the paper's search-based optimization approach.

### 16.3 Algorithm Running Time

The placement search runs in **minutes**, not hours, and parallelizes well across CPU cores.

That makes periodic replanning realistic.

---

## 17. Limitations and When DistServe May Not Be Best

The paper is fairly honest that DistServe is not the right answer for every setting.

### 17.1 Throughput-Only Offline Workloads

If a workload is not latency-sensitive, then the goal may be pure throughput rather than goodput.

In that case, colocated systems with chunked-prefill style batching may be more attractive because they can keep every iteration highly utilized.

### 17.2 Small or Resource-Constrained Deployments

If you only have a few GPUs, or even one GPU, the design space is much smaller.

Then the complexity of disaggregation may not pay off, and simpler non-disaggregated systems may be better operationally.

### 17.3 Missing Features in the Current System

The paper does not fully implement:

- advanced preemption
- full fault tolerance

Those are left as future work.

### 17.4 Long Contexts

Interestingly, the paper argues DistServe still looks promising for very long contexts.

- KV transfer grows roughly linearly with context length
- prefill compute grows much faster
- the mismatch between prefill and decode becomes even larger

So long-context serving may actually strengthen the motivation for disaggregation.

---

## 18. How This Paper Connects to the Course

This paper fits the course extremely well. It connects directly to several Module 3 ideas.

### 18.1 Connection 1: Transformer Inference and KV Cache

From the course lecture on transformer attention:

- prefill processes many prompt tokens together
- decode generates one token at a time
- KV cache is necessary to avoid recomputation

DistServe is essentially a systems paper built on top of exactly that computational structure.

Without understanding:

- autoregressive generation
- prefill vs. decode
- KV cache behavior

the DistServe paper does not make sense.

### 18.2 Connection 2: Distributed DNN Parallelism

From the module on distributed DNN systems, you saw different forms of parallelism.

DistServe uses the serving-side versions of those ideas:

- **intra-op parallelism** is similar in spirit to tensor parallelism
- **inter-op parallelism** is similar in spirit to pipeline parallelism

But here the goal is not training throughput. The goal is **serving goodput under latency SLOs**.

That is a nice course connection: the same distributed systems tools are reused, but under a different optimization objective.

### 18.3 Connection 3: ORCA

This is the most important presentation connection.

ORCA, from your lecture, addressed a real serving problem for transformer generation:

- requests finish at different times
- new requests arrive asynchronously
- fixed request-level batches waste work and increase latency

ORCA's key ideas were:

- **iteration-level scheduling**
- **selective batching**
- better utilization during autoregressive generation

That was a major step forward.

But DistServe says that even ORCA-class systems still leave a deeper issue unsolved.

---

## 19. Why ORCA Is Not Enough for DistServe's Goal

The cleanest way to say it is:

**ORCA solves a scheduling problem inside a shared engine. DistServe solves an architectural resource-separation problem.**

### 19.1 What ORCA Solves

ORCA improves how requests are admitted and processed over iterations.

That helps with:

- early-finished requests
- late-arriving requests
- dynamic batching efficiency
- overall serving utilization

Those are real wins.

### 19.2 What ORCA Still Assumes

Even with iteration-level scheduling or continuous batching, ORCA-style systems still fundamentally **colocate prefill and decode on the same serving resources**.

The DistServe paper explicitly places ORCA in the family of prior systems that still use colocated prefill/decode execution.

That means ORCA still inherits two problems:

1. **Prefill-decode interference remains**
2. **Resource/parallelism coupling remains**

### 19.3 Why Scheduling Alone Cannot Remove the Interference

Suppose a long prefill request and many small decode steps share the same GPUs.

Even if the scheduler is very smart:

- they still compete for the same compute resources
- they still compete for the same memory bandwidth
- they still share the same model-parallel setup

So the scheduler can reduce inefficiency, but it cannot change the fact that the two phases want different hardware behavior.

That is DistServe's criticism.

### 19.4 ORCA vs. DistServe in One Table

| Aspect | ORCA | DistServe |
|--------|------|-----------|
| Main target | Better batching/scheduling for generative serving | Better goodput under TTFT and TPOT SLOs |
| Core granularity | Iteration-level scheduling | Phase-level disaggregation + placement search |
| Prefill/decode placement | Colocated | Disaggregated |
| Main problem addressed | Request scheduling inefficiency | Prefill-decode interference and resource coupling |
| Resource decision | Mostly within a shared engine | Separate GPU allocation and parallelism per phase |
| Network-awareness | Not the main story | Explicitly modeled in placement |

### 19.5 The Best Way to Present the Relationship

Do not present DistServe as "ORCA was wrong."

A better framing is:

- ORCA improved **how** shared serving resources are scheduled.
- DistServe questions **whether the two phases should share resources in the first place**.

So DistServe is best seen as a **next architectural step** beyond ORCA for latency-sensitive LLM serving.

### 19.6 DistServe Still Benefits from ORCA-like Techniques

This is an important nuance.

DistServe is not anti-batching or anti-ORCA.

In fact, the implementation integrates **continuous batching** and other inference optimizations. So the right mental model is:

- ORCA-style techniques help **inside** each serving phase
- DistServe adds a higher-level architectural layer **around** them

That is why the two papers fit together so well in the course.

---

## 20. Presentation-Friendly Narrative

If you need a clean story for the presentation, use this flow:

1. **Start from user-facing latency**
   Explain TTFT and TPOT, and why both matter.

2. **Explain prefill vs. decode**
   Prefill is heavy and often compute-bound. Decode is incremental and often bandwidth-bound.

3. **Show why colocation is bad**
   Long prefill jobs hurt decode latency, and shared resource settings force bad compromises.

4. **Introduce DistServe's main idea**
   Separate the two phases onto different GPUs.

5. **Explain the systems challenge**
   Once separated, you must solve GPU allocation, parallelism selection, and placement under network constraints.

6. **Explain the solution structure**
   Simulator + search + bandwidth-aware placement + simple runtime scheduler.

7. **Show the payoff**
   Up to 7.4x higher rate or 12.6x tighter SLOs.

8. **Connect back to ORCA**
   ORCA fixed dynamic batching in colocated systems. DistServe goes one level up and separates the phases entirely.

---

## 21. If You Remember Only 10 Things

1. DistServe is about **goodput**, not just throughput.
2. LLM serving has two phases with very different behavior: **prefill** and **decode**.
3. Prefill is often **compute-bound**; decode is often **memory-bandwidth-bound**.
4. Colocating the two phases causes **prefill-decode interference**.
5. Colocation also forces **one shared resource and parallelism plan** for two different workloads.
6. DistServe fixes this by **disaggregating** prefill and decode onto different GPU groups.
7. After disaggregation, DistServe independently optimizes each phase's batching, parallelism, and scale.
8. The paper uses **simulation plus search** to find a good placement under SLO and bandwidth constraints.
9. KV-cache transfer overhead is small in practice if placement respects fast links like NVLINK.
10. Relative to ORCA, DistServe's message is: **better scheduling is good, but separating the phases is even better when latency SLOs matter.**

---

## 22. Final Takeaway

The deepest insight of the paper is not just that prefill and decode are different.

It is this:

**When two stages of a workload have different bottlenecks, latency goals, and scaling behavior, a shared-resource design can become the real bottleneck.**

DistServe's contribution is to turn that observation into a concrete serving architecture, a placement optimizer, and a working system.

For this course, the paper is a natural continuation of the ORCA lecture:

- ORCA taught how to schedule generative serving better.
- DistServe teaches when a better scheduler is still not enough, and why architecture-level disaggregation can be the next systems lever.

---

## References

- Yinmin Zhong et al., "DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving", 2024
- Gyeong-In Yu et al., "ORCA: A Distributed Serving System for Transformer-Based Generative Models", OSDI 2022
- The course note on transformer attention and ORCA serving