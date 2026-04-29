# Slide Material: DistServe Presentation

**Recommended deck:** 9 slides.  
This satisfies the professor's required slide categories and uses one extra methods slide so the architecture and the placement/runtime ideas do not get crammed together.

---

## Slide 0 - Title Slide

**Slide title**

**DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving**

**On-slide content**

- Yinmin Zhong et al., 2024
- Scalable Systems for Data Science
- Presented by: *Your Name*
- One-line thesis: **separate prefill and decode to improve per-GPU goodput under latency SLOs**

**What to present**

- Start with the paper's core claim: existing LLM serving systems usually keep prefill and decode together, but this paper argues that is the wrong architecture when latency guarantees matter.
- Say the paper is not just about better batching; it is about a higher-level architectural change.
- Preview the punchline: by disaggregating the two phases, DistServe can serve more requests per GPU while still meeting TTFT and TPOT targets.

**Suggested visual**

- Clean title slide only, with a small subtitle: **"From shared-engine scheduling to phase-level disaggregation."**

---

## Slide 1 - Overall Context

**On-slide content**

- LLM inference has **two phases**:
  - **Prefill**: process the full prompt and generate the first token
  - **Decode**: generate later tokens one by one using the **KV cache**
- User-facing latency is measured by:
  - **TTFT** = time to first token
  - **TPOT** = time per output token
- Different apps emphasize them differently:
  - chatbots -> low TTFT
  - summarization -> low TPOT
  - coding assistants -> both matter

**What to present**

- Connect this to transformer inference from class: autoregressive generation naturally splits inference into prompt processing and token-by-token generation.
- Explain that this split matters because the two phases have different bottlenecks: prefill is often more compute-heavy, while decode is often more memory-bandwidth sensitive.
- Frame the systems question: if the computation is already split logically, should the serving architecture also split physically?

**Suggested visual**

- A simple two-stage pipeline:
  `Prompt -> Prefill -> KV cache -> Decode -> Output stream`

---

## Slide 2 - Motivation and Gaps

**On-slide content**

- **Problem 1: Prefill-decode interference**
  - long prefill jobs delay decode steps
  - decode traffic also hurts TTFT
- **Problem 2: Coupled resource decisions**
  - same GPU pool
  - same parallelism plan
  - same memory budget
- Existing fixes like **continuous batching / ORCA / chunked prefill** help scheduling, but **do not remove the shared-resource conflict**
- Throughput is not enough -> the right metric is **goodput**

**What to present**

- Say DistServe's criticism of prior work is not that batching is useless; it is that smarter batching still operates inside a colocated design.
- Mention ORCA carefully: ORCA improved iteration-level scheduling in shared serving, but DistServe argues the deeper issue is that prefill and decode still compete for the same hardware.
- Use a concrete intuition: a long prompt is like a large setup job, while decode is a stream of short latency-sensitive jobs; mixing them creates tail latency problems.

**Suggested visual**

- Use **Figure 1** idea from the paper or draw a cartoon showing long prefill blocking many decode steps.

---

## Slide 3 - Problem Definition

**On-slide content**

- **Goal:** maximize **per-GPU goodput**
- **Goodput:** max request rate per GPU such that at least the target fraction of requests meets **both** TTFT and TPOT SLOs
- Inputs to the optimization:
  - model size
  - prompt/output length distribution
  - request rate
  - latency SLOs
  - GPU memory and cluster bandwidth
- Output decisions:
  - number of prefill instances
  - number of decode instances
  - parallelism choice for each
  - placement in the cluster

**What to present**

- Emphasize the paper's key shift from raw throughput to goodput.
- Explain why this matters operationally: a system can produce many tokens per second and still be bad for users if too many requests miss latency targets.
- Mention the paper's 90% SLO attainment target as the main evaluation setting.

**Suggested visual**

- Put this formula in a box:  
  `goodput = max req/s/GPU subject to >= alpha of requests meeting TTFT and TPOT`

---

## Slide 4 - Methods 1: Architecture

**On-slide content**

- **Disaggregate the phases**
  - prefill runs on **prefill instances**
  - decode runs on **decoding instances**
- A request flow:
  1. request arrives
  2. prefill instance computes first token + KV cache
  3. KV cache is transferred
  4. decode instance continues generation
  5. output is streamed to the client
- Each instance may span **multiple GPUs**
- Separate queues and separate scaling for the two phases

**What to present**

- Stress that an "instance" in the paper is a serving unit with one model replica, not necessarily one GPU.
- The architecture's main benefit is isolation: prefill no longer directly blocks decode on the same GPU.
- The second benefit is configurability: each phase can now choose a different batching policy and parallelism strategy.

**Suggested visual**

- Best option: redraw **Figure 6** simply:
  `Client requests -> controller -> shortest-queue prefill -> KV handoff -> least-loaded decode -> streamed output`

---

## Slide 5 - Methods 2: Phase-specific Techniques

**On-slide content**

- **Prefill optimization target = TTFT**
  - often compute-bound
  - very long prompts may already saturate the GPU
  - batch short prompts up to a saturation threshold `L_m`; run very long prompts alone
- **Decode optimization target = TPOT**
  - often memory-bandwidth-bound
  - larger decode batches improve utilization
- **Parallelism can differ by phase**
  - prefill may prefer lower-latency choices under tight TTFT
  - decode may prefer throughput-oriented scaling under TPOT constraints

**What to present**

- This is the most important methods slide after the architecture slide.
- Say DistServe does not stop at "separate them"; it independently tunes them because the two phases have different bottlenecks.
- Mention the paper's tradeoff: prefill may prefer intra-op parallelism when latency is very tight, but inter-op can win at higher arrival rates because it reduces queuing.

**Suggested visual**

- A 2-column table:

| Prefill | Decode |
|---|---|
| optimize TTFT | optimize TPOT |
| compute-heavy | bandwidth-heavy |
| smaller / saturation-aware batches | larger batches |
| latency-sensitive execution plan | utilization-focused batching |

---

## Slide 6 - Methods 3: Placement and Runtime

**On-slide content**

- **Placement problem:** choose GPU split + parallelism + location
- **Search strategy:** simulator + enumeration + binary search
- **High-bandwidth case:** optimize prefill and decode separately, then replicate
- **Low-bandwidth case:** place matching prefill/decode stages on the same node to use **NVLINK** for KV transfer
- Runtime system:
  - centralized controller
  - **shortest-queue** prefill dispatch
  - **least-loaded** decode dispatch
  - **pull-based** KV transfer
  - periodic replanning

**What to present**

- Explain that disaggregation creates a new systems problem: communication overhead and topology-aware placement.
- The paper's answer is bandwidth-aware placement: keep phase separation, but make KV handoff use fast links whenever possible.
- Mention the nice runtime detail: decode pulls KV cache when ready, so prefill memory acts like a buffer and bursty traffic does not overflow decode memory.

**Suggested visual**

- A compact diagram with three boxes: **Simulator/Search -> Placement -> Runtime Controller**

---

## Slide 7 - Sample Results

**On-slide content**

- **Headline:** up to **7.4x higher request rate** or **12.6x tighter SLOs**
- Chatbot (ShareGPT): **2.0x-4.6x** higher rate than vLLM
- Code completion (OPT-66B): **5.7x** higher rate than vLLM
- Summarization (OPT-66B): **4.3x** higher rate and **12.6x** tighter SLO than vLLM
- Communication overhead is small:
  - KV transfer is **< 0.1%** of total latency in one breakdown
  - **>95%** of transfers are under **30 ms**

**What to present**

- Present the headline result first, then explain why the gains differ by workload.
- Chatbot and summarization benefit a lot because colocated systems suffer when long prefill work slows decode.
- Code completion benefits because TTFT is very strict, so faster prefill tuning matters.
- Close this slide by saying the communication-overhead result is critical: DistServe works because the network cost is controlled by bandwidth-aware placement.

**Suggested visual**

- Main figure: **Figure 8** or **Figure 9**
- Small callout bubble: **Figure 10 -> transmission is not the bottleneck**

---

## Slide 8 - Key Takeaways and Course Connections

**On-slide content**

- **Main takeaway:** DistServe improves **goodput** by disaggregating prefill and decode
- **Transformer connection:** prefill/decode split and KV cache are the foundation of the whole design
- **ORCA connection:**  
  - ORCA -> better scheduling inside a shared serving engine  
  - DistServe -> asks whether the two phases should share resources at all
- **Distributed systems patterns**
  - workload-aware specialization
  - resource disaggregation
  - topology-aware placement
  - simple runtime control with periodic replanning
- Best fit: **latency-sensitive, multi-GPU serving clusters**

**What to present**

- This is the slide where you connect the paper back to class.
- The clean comparison is: ORCA improves batching and scheduling, while DistServe moves one level up and changes the serving architecture itself.
- End with one strong sentence: when two phases have different bottlenecks and latency goals, separating them can be more valuable than trying to schedule them better on the same hardware.

**Suggested visual**

- A small comparison table:

| Topic from class | DistServe's connection |
|---|---|
| Transformer inference | prefill vs decode, KV cache |
| ORCA | scheduling within colocated serving |
| Distributed systems | disaggregation, placement, control/runtime design |

---

## Short closing line for the actual presentation

**"DistServe's core insight is that better scheduling is not always enough: if prefill and decode want different hardware behavior, the right systems move is to separate them and optimize each phase on its own terms."**
