# Lecture 3.1: Spark ML (Machine Learning with Spark)

## DS256 - Scalable Systems for Data Science
### Module 3: Machine Learning at Scale

> **Reference**: Ch 10 – Machine Learning with MLlib, Learning Spark, 2nd Edition

---

## 1. Motivation: Why Spark for Machine Learning?

### 1.1 The Problem

Modern data systems (Data Lakes) store raw data at massive scale. Before you can train any ML model, you need to:
1. Pre-process huge volumes of raw data
2. Run ML pipelines across distributed machines

**The key insight**: Spark's RDD/DataFrame model is already ideal for data transformation and preparation — Spark ML simply extends this to learning.

### 1.2 What Spark ML Gives You

- **Reuse Spark concepts**: You don't need to learn a new system — you use DataFrames, transformations, and actions just like before.
- **Scaling to large datasets**: ML algorithms run distributed across the cluster.
- **Easier exploratory analysis**: Unified API for data engineering + ML.

---

## 2. Two Spark ML Libraries

| Library | Abstraction | Status |
|---------|-------------|--------|
| `spark.mllib` | RDD API (older) | Maintenance mode |
| `spark.ml` | DataFrame API (newer) | Active / Recommended |

> **Use `spark.ml`** — it integrates with DataFrames and the Pipeline API.

---

## 3. Comparison: Spark ML vs. Scikit-Learn vs. Keras/TensorFlow/PyTorch

### 3.1 Spark ML vs. Scikit-Learn

Both are for **classical (non-deep) ML training**, but their design goals differ:

| Aspect | Scikit-Learn | Spark ML |
|--------|-------------|---------|
| **Primary Goal** | ML exploration and validation | Scalable ML pipelines (production) |
| **Scale** | Single machine only | Distributed across cluster |
| **ML Libraries** | Very feature-rich (many algorithms) | Fewer built-in algorithms |
| **Visualization** | Integrates with matplotlib, etc. | None built-in |
| **Data Engineering** | Separate from ML | Integrated with DataFrames/RDDs |
| **Use Case** | Exploration, small datasets | Operations, large-scale |

**Key Limitation of Spark ML**: Only algorithms that can be expressed using Spark's RDD/DataFrame model can be parallelized. Not all ML algorithms are amenable to this.

**Decision Rule**:
- Small dataset or exploration → **Scikit-Learn**
- Large-scale or production ML pipeline → **Spark ML**

### 3.2 Spark ML vs. Keras/TensorFlow/PyTorch

| Aspect | Spark ML | Keras/TF/PyTorch |
|--------|----------|-----------------|
| **Type** | Distributed, classical (non-deep) ML | Single-machine (+ optional GPU) deep learning |
| **Focus** | Classical algorithms (regression, clustering, etc.) | Neural networks, deep learning |
| **Distribution** | Native (built for clusters) | Optional (e.g., Distributed TensorFlow) |
| **Best For** | Standard ML at scale | Deep learning |

**Workflow combining both**:
- Use Spark DataFrames for large-scale **data preparation**
- Hand off to TF/PyTorch for **DL training**
- Alternatively: Spark Deep Learning Pipelines for DL training via Spark

---

## 4. Standard ML Training Workflow in Spark ML

The standard workflow has three main stages:

```
Raw Data (DataFrame)
        │
        ▼
1. DATA PREPARATION
   ├── Data quality checks / cleaning (standard DataFrame ops)
   ├── Train/Test split (80:20)
   └── Feature engineering (Transformers like VectorAssembler)
        │
        ▼
2. MODEL TRAINING
   └── Estimator.fit(trainingDF) → Model (Transformer)
        │
        ▼
3. PREDICTION & EVALUATION
   ├── Model.transform(testDF) → predictionDF
   └── Evaluate: RMSE, R²
```

---

## 5. Data Preparation

### 5.1 Train/Test Split

```python
# 80% training, 20% testing, with seed for reproducibility
(trainDF, testDF) = df.randomSplit([0.8, 0.2], seed=42)
```

**Important nuance**: The exact rows in training vs. testing depend on the **number of executors** (because splits happen per partition). Always save training and testing data for reproducibility!

### 5.2 Transformers

**What is a Transformer?**
- Accepts a DataFrame as input
- Returns a **new DataFrame** with one or more columns appended
- Transformers are **lazy** (like RDD transformations) — they don't execute until an action/fit triggers it

**Key Transformer: VectorAssembler**

The most important data preparation transformer. It:
- Takes a list of input columns
- Combines them into a single vector stored in an output column (usually called `"features"`)
- Appends this `"features"` column to a new DataFrame

```python
from pyspark.ml.feature import VectorAssembler

assembler = VectorAssembler(
    inputCols=["col1", "col2", "col3"],
    outputCol="features"
)
featureDF = assembler.transform(rawDF)
# featureDF now has a "features" column with a dense/sparse vector
```

**Why?** Most ML algorithms in Spark ML expect a single vector column as input, not individual columns.

### 5.3 Sparse vs. Dense Vectors

#### Dense Vector
- Stores all elements, including zeros
- Memory: proportional to number of elements
- Example: `[1.0, 0.0, 3.5, 0.0, 0.0]`

#### Sparse Vector
- Stores only **non-zero elements** as `(indices, values)` pairs
- Memory: proportional to number of non-zero elements
- Example: `{5: [0,2], v: [1.0, 3.5]}` → indices `[0, 2]` have values `[1.0, 3.5]`

**Why Sparse Vectors?** Crucial for **One-Hot Encoding (OHE)**:
- OHE converts categorical features to numerical binary vectors
- Example: "color" = {red, blue, green} → vector of length 3 with one 1 and rest 0s
- These vectors are very large but mostly zero → sparse representation saves huge amounts of memory!

#### One-Hot Encoding with Spark ML

```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder

# Step 1: Convert string categories to numeric indices
indexer = StringIndexer(inputCol="color", outputCol="colorIndex")

# Step 2: One-Hot Encode the indices
encoder = OneHotEncoder(inputCols=["colorIndex"], outputCols=["colorVec"])
```

`OneHotEncoder` maps a column of **category indices** to a column of **binary vectors** (as sparse vectors internally).

**Intuition**: OHE avoids a spurious "ordering" that raw integer encoding would introduce (e.g., blue=0, red=1, green=2 would incorrectly imply blue < red < green).

---

## 6. Training a Model: Estimators

### 6.1 What is an Estimator?

An **Estimator** is an algorithm that **learns parameters from training data**. Think of it as the ML algorithm itself (e.g., Linear Regression, Decision Tree).

**Key API**:
```python
model = estimator.fit(trainingDF)
```

- `estimator.fit()` is **eagerly evaluated** (like an RDD action — triggers computation immediately!)
- Returns a **trained Model** object, which is itself a **Transformer**

### 6.2 Training a Linear Regression Model

```python
from pyspark.ml.regression import LinearRegression

# Create estimator
lr = LinearRegression(featuresCol="features", labelCol="label", maxIter=10)

# Train (eager)
lrModel = lr.fit(trainDF)

# Predict (lazy until an action)
predDF = lrModel.transform(testDF)
# predDF has a "prediction" column appended
```

**The workflow**:
```
trainDF → Estimator.fit() → Model (Transformer)
                                    │
                        testDF → Model.transform() → predDF
```

---

## 7. ML Pipeline

### 7.1 Why Pipelines?

Real ML workflows have multiple sequential steps:
1. Handle missing values
2. String indexing (categorical → numeric)
3. One-hot encoding
4. Feature assembly (VectorAssembler)
5. Model training

Without the Pipeline API, you'd chain each step manually and risk errors.

### 7.2 The Pipeline API

```python
from pyspark.ml import Pipeline

# Define stages in order
pipeline = Pipeline(stages=[indexer, encoder, assembler, lr])

# Pipeline is an Estimator — fit trains all stages
pipelineModel = pipeline.fit(trainDF)

# PipelineModel is a Transformer — transform runs predictions
predDF = pipelineModel.transform(testDF)
```

**Key insight**:
- A `Pipeline` is an **Estimator** — calling `.fit()` trains all stages in order
- The result, `PipelineModel`, is a **Transformer** — calling `.transform()` applies all stages for prediction

```
Pipeline (Estimator)
├── Stage 1: StringIndexer (Transformer)
├── Stage 2: OneHotEncoder (Transformer)
├── Stage 3: VectorAssembler (Transformer)
└── Stage 4: LinearRegression (Estimator → becomes Model after fit)

         ↓ .fit(trainDF)

PipelineModel (Transformer)
└── .transform(testDF) → predDF
```

### 7.3 Intuition Behind Pipelines

Think of it like a factory assembly line. Each stage takes a DataFrame, does something, and passes an enhanced DataFrame to the next stage. The Pipeline API manages this chain automatically, making your code cleaner and less error-prone.

---

## 8. Evaluating Models

### 8.1 Root Mean Square Error (RMSE)

$$\text{RMSE} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (y_i - \hat{y}_i)^2}$$

Where:
- $N$ = number of data points
- $y_i$ = true value
- $\hat{y}_i$ = predicted value

**Properties**:
- Relative estimate (depends on the scale of the target variable)
- Sensitive to units — a RMSE of 5 means different things if you're predicting house prices in dollars vs. millions of dollars
- **Lower is better**

### 8.2 R² (Coefficient of Determination)

$$R^2 = 1 - \frac{\sum (y_i - \hat{y}_i)^2}{\sum (y_i - \bar{y})^2}$$

**Interpretation**:
| Value | Meaning |
|-------|---------|
| **1.0** | Perfect prediction — model explains all variance |
| **0.0** | Model is as good as just predicting the mean every time |
| **< 0** | Model is worse than a constant "always predict mean" model |

**Intuition for R²**: How much of the variance in the data does your model explain? An R² of 0.85 means your model explains 85% of the variance.

### 8.3 Code Example

```python
from pyspark.ml.evaluation import RegressionEvaluator

evaluator = RegressionEvaluator(
    labelCol="label",
    predictionCol="prediction",
    metricName="rmse"  # or "r2"
)

rmse = evaluator.evaluate(predDF)
print(f"RMSE: {rmse}")

evaluator.setMetricName("r2")
r2 = evaluator.evaluate(predDF)
print(f"R²: {r2}")
```

---

## 9. Key Concepts Summary Table

| Concept | Role | Trigger | Returns |
|---------|------|---------|---------|
| **Transformer** | Transforms data | `transform(df)` (lazy) | New DataFrame |
| **Estimator** | Learns from data | `fit(df)` (eager) | Transformer (trained model) |
| **Pipeline** | Chains multiple stages | `fit(df)` (eager) | PipelineModel (Transformer) |
| **PipelineModel** | Applies trained pipeline | `transform(df)` (lazy) | Prediction DataFrame |

---

## 10. Summary: Key Takeaways

1. **spark.ml** is the modern DataFrame-based Spark ML library (not mllib).

2. **Spark ML vs. Scikit-Learn**: Scikit-learn for exploration (feature-rich, single machine); Spark ML for large-scale distributed ML pipelines.

3. **Spark ML vs. DL frameworks**: Spark ML handles classical ML distributed; Keras/TF/PyTorch for deep learning (optionally with Spark for data prep).

4. **Transformers are lazy; Estimators (fit) are eager** — understanding this mirrors RDD transformations vs. actions.

5. **VectorAssembler** is the key transformer: combines multiple feature columns into a single `"features"` vector column required by ML algorithms.

6. **Sparse vectors** are essential for memory-efficient representation of OHE features.

7. **Pipeline API** chains Transformers and Estimators into a clean, reproducible workflow.

8. **RMSE**: unit-dependent error metric. **R²**: unit-free (0 to 1), measures variance explained.

---

## References

- Karau, H., et al., "Learning Spark", O'Reilly, 2nd Edition, Ch. 10


# Lecture 3.2: Distributed DNN Training

## DS256 - Scalable Systems for Data Science
### Module 3: Machine Learning at Scale

> **Key Paper**: "Beyond Data and Model Parallelism for Deep Neural Networks", Zhihao Jia, Matei Zaharia, Alex Aiken. MLSys 2019.
>
> **Additional Sources**: CMU 15-418, MIT 6.S965, "Demystifying Parallel and Distributed Deep Learning", Tal Ben-Nun & Torsten Hoefler, ACM Comput. Surv. 2019.

---

## 1. Why is DNN Training So Expensive?

### 1.1 ML Training is Iterative

All ML (including deep learning) follows this iterative loop:

```
1. Initialize model with small random weights
2. Forward pass: predict output for a batch of training data
3. Compute loss (how wrong are we?)
4. Backward pass: compute gradients (how should we update weights?)
5. Update weights using gradients
6. Repeat 2-5 until error is small enough
```

Steps 2-5 are the expensive part: they operate over all training data and use the current model parameters.

### 1.2 The Scale of Modern DNN Training

Modern DNN models are enormous:
- **Several GBs** of model weights
- **Millions of neurons per layer**, millions of edges between layers
- **TBs of training data**
- Example: **Megatron-Turing NLG** (Microsoft/Nvidia):
  - 270 billion tokens from English-language websites
  - 560 Nvidia DGX A100 servers, each with 8 × A100 80GB GPUs
  - Training takes **weeks to months** on this hardware!

**Bottom line**: A single GPU or even a single machine cannot handle this — we need distributed training.

---

## 2. Deep Learning: Forward and Backward Pass

### 2.1 Neural Network Structure

```
Input x → [W₁] → h₁ → [W₂] → h₂ → [W₃] → h₃ → Output y
```

- **Forward Pass**: Map input `x` through layers of learned features `h`, connected by weight matrices `W`, to produce output `y`
- **Backward Pass**: Use the loss gradient to update weights `W` via backpropagation

**Goal**: Find `W = argmin_W L(x, y)` — the weights that minimize the loss function.

### 2.2 Stochastic Gradient Descent (SGD)

**Full batch gradient descent** computes gradients on the entire dataset — too slow.

**SGD** updates weights using a **mini-batch** `B` of training data:

$$W^{(t+1)} = W^{(t)} - \alpha \cdot \nabla_W L(x_B, y_B)$$

Where:
- `W^(t)` = weight vector at time step t
- `α` = learning rate
- `∇L` = gradient of the loss with respect to weights on mini-batch `B`

**Intuition**: Instead of computing the exact gradient direction over all data (expensive), we approximate it using a small batch. This adds noise but makes training much faster per step.

---

## 3. Parallelizing DNN Training

Training a DNN involves two things that can be parallelized:
1. **Data** (training samples)
2. **Model** (the neural network itself)

This naturally leads to the two classical approaches:

```
┌────────────────────────────────────────────────────────┐
│                  Distributed DNN Training               │
├──────────────────────┬─────────────────────────────────┤
│   DATA PARALLELISM   │      MODEL PARALLELISM           │
│   Split training     │   Split the model                │
│   data across GPUs   │   across GPUs                    │
│   Same model on all  │   Different parts of model       │
│   GPUs               │   on different GPUs              │
└──────────────────────┴─────────────────────────────────┘
```

---

## 4. Data Parallelism

### 4.1 Core Idea

Split the training data across GPU workers. **Each GPU holds a complete copy of the model** but trains on its local subset of the data.

```
Training Data
    ├── Batch 1 → GPU 1 (full model copy) → local gradients
    ├── Batch 2 → GPU 2 (full model copy) → local gradients
    ├── Batch 3 → GPU 3 (full model copy) → local gradients
    └── Batch 4 → GPU 4 (full model copy) → local gradients
                            │
                   Synchronize gradients (aggregate)
                            │
                   Update all model copies
```

**Steps**:
1. Training data is partitioned; multiple model replicas are trained in parallel
2. After each mini-batch, model parameters (gradients) are synchronized across all workers

### 4.2 Parameter Server (PS) Architecture

One common approach to synchronize gradients in data parallelism:

```
┌──────────────────┐    ┌──────────────────┐
│  Parameter Server│    │  Parameter Server│
│  w1  w2  w3     │    │  w4  w5  w6     │
└─────────┬────────┘    └─────────┬────────┘
          │                       │
    ┌─────┴────────────────────────┤
    ▼              ▼              ▼
Worker 1        Worker 2       Worker 3
(Data partition) (Data partition) (Data partition)
```

- **Parameter servers** store model parameters and expose them via a **key-value interface** (distributed shared memory)
- **Workers** compute gradients on their local data partition and **push** to PS
- **PS** aggregates gradients from all workers and **pushes** back the updated parameters

**Key property**: Different parts of the model can live on different PS machines (workers retrieve only what they need).

### 4.3 Synchronization Modes (PS)

Three ways workers can coordinate:

| Mode | Description | Use Case |
|------|-------------|----------|
| **Sequential (BSP)** | Tasks execute one by one. Next task starts only after previous finishes. Identical to single-thread behavior. | Maximum correctness |
| **Eventual** | All tasks can start simultaneously. No waiting. | When algorithm is robust to delayed updates |
| **Bounded Delay** | Max delay time is set. A new task is blocked until all tasks from Δ time steps ago have finished. | Balance between correctness and speed |

### 4.4 Problem with Parameter Server: O(N) Bandwidth

As you add more workers, the bandwidth demand on the PS grows linearly:
- Each of N workers contacts the PS every iteration
- PS becomes a **bottleneck** at scale (O(N) communication)

```
N workers × message size per worker = O(N) bandwidth at PS
```

### 4.5 MPI All-Reduce: Solving the Bandwidth Problem

**All-Reduce** is an alternative communication pattern that avoids the centralized PS bottleneck:

```
Instead of all workers → PS → all workers (star topology, O(N)):

Workers form a ring and communicate pairwise:
  W1 ↔ W2 ↔ W3 ↔ W4 ↔ W1 (ring)
  log₂(N) rounds to aggregate everything
```

**Advantage**: Peak bandwidth goes from **O(N)** (PS) to **O(1)** — each worker always communicates with just 2 neighbors.

**Trade-off**: Requires O(log N) steps instead of O(1) steps (PS is 1 round trip).

**Used by**: Horovod, NCCL, MPI-based systems.

### 4.6 The Critical Problem with Data Parallelism

**The entire model must fit in each worker's GPU memory.**

Example: A 350 GB model requires 350 GB on each GPU in data parallelism. Since modern GPUs only have 40–80 GB memory, the model cannot fit on a single GPU. This motivates model parallelism, where the model is split across multiple GPUs.

---

## 5. Model Parallelism

### 5.1 Core Idea

Instead of replicating the model on every GPU, **partition the model** and assign different parts to different GPUs.

```
GPU 1       GPU 2       GPU 3       GPU 4
┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐
│Layers │   │Layers │   │Layers │   │Layers │
│ 1-25  │──▶│ 26-50 │──▶│ 51-75 │──▶│ 76-100│
└───────┘   └───────┘   └───────┘   └───────┘
```

**Benefits**:
- Allows training models that exceed single GPU memory
- Example: 350GB model across 8 GPUs = 43.75 GB per GPU (fits in 80 GB A100)

### 5.2 Types of Model Parallelism

1. **Inter-layer (Layer-wise) parallelism**: Split the model across layers. GPU 1 gets layers 1-N, GPU 2 gets layers N+1 to 2N, etc.
2. **Intra-layer (Tensor parallelism)**: Within a single layer, split the weight matrix across neurons/channels on different GPUs.

```
Inter-layer:              Intra-layer:
GPU1: [Layer 1]          GPU1: [Part of Layer 1]
GPU2: [Layer 2]          GPU2: [Part of Layer 1]
GPU3: [Layer 3]          (both work on same layer simultaneously)
```

**Complexity**: Placement of model parts on GPUs can be very complex — especially for non-linear models like RNNs with complex dependency graphs.

### 5.3 Problem with Naive Model Parallelism: Low Utilization

With layer-wise model parallelism, at any given time only **one GPU is active** — the rest are idle:

```
Time →
GPU1: [==Forward1==][idle][idle][idle][==Backward1==][idle]
GPU2: [idle][==Fwd2==][idle][idle][idle][==Bkwd2==][idle]
GPU3: [idle][idle][==Fwd3==][idle][idle][idle][==Bkwd3==]
GPU4: [idle][idle][idle][==Fwd4==][idle][idle][idle]

Only ~25% GPU utilization in a 4-GPU setup!
```

---

## 6. Pipeline Parallelism (GPipe)

### 6.1 The Key Insight

**Overlap computation** from different mini-batches across the pipeline stages to increase GPU utilization.

**How**: Split each mini-batch into **micro-batches**. While GPU2 processes micro-batch 1, GPU1 can start on micro-batch 2.

```
Without pipelining:
GPU1: [MB1-fwd][idle    ][MB1-bkwd][idle    ]
GPU2: [idle   ][MB1-fwd][idle    ][MB1-bkwd]

With pipeline parallelism (GPipe):
GPU1: [μB1-fwd][μB2-fwd][μB3-fwd][μB4-fwd][ bubble ][μB4-bkwd]...
GPU2: [idle   ][μB1-fwd][μB2-fwd][μB3-fwd][μB4-fwd][  bubble ][μB4-bkwd]...
```

### 6.2 GPipe Details

- Split each mini-batch into **m micro-batches**
- **Pipeline** the forward and backward computations across micro-batches
- **Idle time** (pipeline "bubble") reduced to **(p-1)/m** where:
  - `p` = number of pipeline stages (GPUs)
  - `m` = number of micro-batches

As `m` grows relative to `p`, bubble time decreases → higher utilization.

**Cost**: Need to store **intermediate activations** for all micro-batches before back-propagation can start → increased memory usage.

---

## 7. Combining All: 3D Parallelism (DeepSpeed)

Modern systems like **DeepSpeed** (Microsoft) combine all three forms of parallelism:

```
┌──────────────────────────────────────────────────────────┐
│                    3D Parallelism                         │
│                                                           │
│  DATA PARALLELISM: Multiple replicas of the pipeline     │
│       ↕                                                   │
│  PIPELINE PARALLELISM: Stages of the model across GPUs   │
│       ↕                                                   │
│  TENSOR PARALLELISM: Within each stage, split layers      │
│                       across GPUs                         │
└──────────────────────────────────────────────────────────┘
```

**Why combine?**
- Data parallelism alone: model doesn't fit in one GPU
- Model parallelism alone: low utilization
- Pipeline parallelism alone: still limited scaling
- **Together**: each addresses a different bottleneck

---

## 8. SOAP: Beyond Data and Model Parallelism (The Paper)

> **Paper**: Jia, Zaharia, Aiken, "Beyond Data and Model Parallelism for Deep Neural Networks", MLSys 2019

### 8.1 The Problem with Existing Approaches

Existing parallelism strategies are **suboptimal** because they only explore limited dimensions:

| Approach | Dimensions Explored | Limitations |
|----------|--------------------|--------------------|
| **Data Parallelism** | Sample (S) only | Inefficient for layers with many params (e.g., embeddings); model must fit on one GPU |
| **Model Parallelism** | Operator (O) + Parameter (P) | Limited parallelism; sequential operator execution |
| **Expert-designed** (e.g., Krizhevsky 2014) | S + P for CNNs | Suboptimal; specific to one model type |
| **OptCNN** | S, A, P | Only works for linear CNNs |
| **ColocRL** | O only | Only learns device placement (not full tensor partitioning) |

### 8.2 The SOAP Search Space

**SOAP** = **S**ample · **O**perator · **A**ttribute · **P**arameter

These are the **four dimensions** in which a DNN can be parallelized:

| Dimension | What it means | Example |
|-----------|--------------|---------|
| **Sample (S)** | Partition training samples across devices | Data parallelism: each device gets a subset of the batch |
| **Operator (O)** | Assign different operators (layers) to different devices | Model parallelism: layer 1 on GPU1, layer 2 on GPU2 |
| **Attribute (A)** | Partition within a sample's attributes | For images: partition along height/width dimensions |
| **Parameter (P)** | Partition model parameters across devices | Split weight matrix channels across GPUs |

**Intuition**: Think of a convolution layer's output tensor as a 4D array: `[batch, height, width, channel]`. You can split it along any of these dimensions across devices:
- Split along `batch` → Sample parallelism
- Split along `height/width` → Attribute parallelism
- Split along `channel` → Parameter parallelism
- Assign entire layers to different devices → Operator parallelism

### 8.3 Why SOAP is More Comprehensive

Example with a 1D convolution:

```
┌─────────────────────────────────────────────────────────┐
│                                                          │
│  Data Parallelism (S):  Split batch across GPUs         │
│  [batch1→GPU1] [batch2→GPU2]                            │
│                                                          │
│  Model Parallelism (P): Split channels across GPUs      │
│  [channels1-512→GPU1] [channels513-1024→GPU2]           │
│                                                          │
│  Hybrid (S+P): Split both batch AND channels            │
│  GPU1: [batch1, chan1-512]  GPU2: [batch1, chan513-1024] │
│  GPU3: [batch2, chan1-512]  GPU4: [batch2, chan513-1024] │
│                                                          │
│  SOAP includes ALL of these + more combinations!        │
└─────────────────────────────────────────────────────────┘
```

**Key insight**: For different operators, different combinations of SOAP dimensions are optimal. Data parallelism is efficient for compute-intensive operators with few parameters (like convolutions) but inefficient for parameter-heavy operators with little computation (like embedding layers).

### 8.4 Parallelizable Dimensions by Operator Type

| Operator | Parallelizable Dimensions (S, A, P) |
|----------|-------------------------------------|
| 1D Pooling | Sample, Length |
| 1D Convolution | Sample, Length, Channel |
| 2D Convolution | Sample, Height×Width, Channel |
| Matrix Multiplication | Sample, Channel |

---

## 9. FlexFlow: Automatic Parallelization

### 9.1 Overview

**FlexFlow** is a deep learning engine that:
1. Takes a DNN model (as an **operator graph**) and a cluster configuration (as a **device topology graph**) as input
2. Uses the **SOAP search space** to find the optimal parallelization strategy
3. Executes the strategy using a distributed runtime

```
┌─────────────────────┐    ┌─────────────────────┐
│   Operator Graph G  │    │  Device Topology D   │
│  (DNN Architecture) │    │  (Cluster hardware)  │
└──────────┬──────────┘    └──────────┬──────────┘
           └──────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Execution Optimizer  │
              │   (MCMC Search Alg.)  │◀──────────────┐
              └───────────┬───────────┘               │
                          │ Candidate Strategy         │
                          ▼                           │
              ┌───────────────────────┐               │
              │  Execution Simulator  │──Simulated────┘
              │  (Predicts perf.)     │   Performance
              └───────────┬───────────┘
                          │ Best Found Strategy
                          ▼
              ┌───────────────────────┐
              │  Distributed Runtime  │
              │  (Actual execution)   │
              └───────────────────────┘
```

### 9.2 Operator Graph and Device Topology

**Operator Graph G = (nodes, edges)**:
- **Node** `oᵢ`: an operator (e.g., matrix multiplication, convolution)
- **Edge** `(oᵢ, oⱼ)`: a tensor (n-dimensional array) that is output of `oᵢ` and input of `oⱼ`

**Device Topology D = (devices, connections)**:
- **Node** `dᵢ`: a device (CPU or GPU)
- **Edge** `(dᵢ, dⱼ)`: hardware connection (NVLink, PCIe, Infiniband)
- Edges labeled with **bandwidth** and **latency**

### 9.3 Parallelization Configurations

For each operator `oᵢ`, a **parallelization configuration** `cᵢ` specifies:
- The **degree of parallelism** in each SOAP dimension (how many pieces to split along that dimension)
- The **device assignment** for each resulting task

Example for matrix multiplication `U = V × W`:
```
Degree(Sample) = 2, Degree(Channel_out) = 2
→ Partitioned into 4 tasks:
  - Task 1: [batch 1-half, channel_out 1-half] → GPU1
  - Task 2: [batch 1-half, channel_out 2-half] → GPU2
  - Task 3: [batch 2-half, channel_out 1-half] → GPU3
  - Task 4: [batch 2-half, channel_out 2-half] → GPU4
```

A **parallelization strategy** `S` = one configuration `cᵢ` for **each** operator in the graph. Each operator's configuration can be chosen independently.

**Key point**: The number of possible strategies is **exponential** in the number of operators → cannot brute-force search.

---

## 10. Execution Simulator

### 10.1 Why a Simulator?

To find the best SOAP strategy, FlexFlow needs to **evaluate many candidate strategies** without running them all on real hardware (too slow — profiling each strategy would take hours).

The simulator predicts performance **three orders of magnitude faster** than real execution.

### 10.2 Simulator Assumptions

The simulator relies on four key assumptions:

| Assumption | What it means |
|-----------|---------------|
| **A1** | Execution time of each task is predictable (low variance, independent of input content) |
| **A2** | Transferring tensor of size `s` over bandwidth `b` takes `s/b` time |
| **A3** | Each device uses FIFO scheduling (first task ready = first task started) |
| **A4** | Runtime overhead is negligible |

### 10.3 Task Graph

The simulator builds a **task graph** `T = (tasks, dependencies)`:

1. For each operator `oᵢ` with configuration `cᵢ`, add `|cᵢ|` **normal tasks** (one per partition)
2. For each pair of tasks with shared tensors:
   - **Same device**: add a dependency edge (no communication needed)
   - **Different devices**: add a **communication task** (data transfer) between them

```
Operator o1 (on GPU1) → Tensor → Operator o2 (on GPU2)
                                       ↕
                    Communication Task (data transfer GPU1→GPU2)
```

Properties tracked per task:
- `exeTime`: measured by running the task once and caching (assumption A1)
- `device`: which GPU/CPU it's assigned to
- `readyTime`, `startTime`, `endTime`: computed during simulation

### 10.4 Full Simulation Algorithm

Uses a variant of **Dijkstra's shortest-path algorithm**:

```
1. Build task graph (as above)
2. Initialize a priority queue (sorted by readyTime)
3. Put all tasks with no predecessors into the queue
4. While queue not empty:
   a. Dequeue task t with smallest readyTime
   b. startTime(t) = max(readyTime(t), last_endTime of t's device)
   c. endTime(t) = startTime(t) + exeTime(t)
   d. For each successor task n:
      - readyTime(n) = max(readyTime(n), endTime(t))
      - If all predecessors done → add n to queue
5. Return max endTime over all tasks
```

**FIFO guarantee**: When a task is dequeued, all earlier tasks have already been scheduled — maintains correct FIFO ordering.

### 10.5 Delta Simulation Algorithm (Key Optimization)

**Problem**: MCMC search proposes a new strategy by changing **one operator's configuration** at a time. The full simulation would re-simulate everything from scratch — wasteful!

**Insight**: If only operator `oᵢ`'s config changes, only tasks derived from `oᵢ` and their downstream dependencies need re-simulation. The rest of the timeline is unchanged.

**Delta simulation**:
1. Update only the tasks and dependencies that changed
2. Enqueue modified tasks into a priority queue
3. Propagate updates forward (like Bellman-Ford shortest path)
4. Only re-simulate the "grey area" (affected tasks)

**Speed improvement**: Delta simulation is **2.2–6.9× faster** than full simulation, with greater speedup as the number of devices grows (more tasks unchanged).

---

## 11. Execution Optimizer: MCMC Search

### 11.1 The Problem

Finding the optimal parallelization strategy is **NP-hard** (reducible to minimum makespan scheduling). The search space is exponential in the number of operators.

### 11.2 MCMC Sampling (Metropolis-Hastings Algorithm)

FlexFlow uses **Markov Chain Monte Carlo (MCMC)** with the Metropolis-Hastings algorithm:

**Current strategy**: `S`
**Propose new strategy**: `S*` by randomly changing one operator's config
**Accept `S*` with probability**:

$$\alpha(S^* | S) = \min\left(1, \exp\left(\beta \cdot (\text{cost}(S) - \text{cost}(S^*))\right)\right)$$

Where `cost(S)` = simulated execution time of strategy `S`.

**Behavior**:
- If `S*` is better (lower cost): always accepted
- If `S*` is worse: accepted with some probability (helps escape local minima)
- As β increases: more greedy (less exploration)

### 11.3 Search Algorithm

```
1. Initialize with known strategies (data parallelism, random strategies)
2. For each initial strategy:
   a. Propose new candidate by randomly changing one operator's config
   b. Evaluate using execution simulator (fast!)
   c. Accept/reject using Metropolis-Hastings
   d. Track the best strategy found
   e. Stop when: time budget exhausted OR no improvement for half the budget
3. Return the best strategy overall
```

**Key advantage over ColocRL**:
- ColocRL requires running actual hardware executions as reward signals → 12-27 hours on 160 nodes
- FlexFlow uses simulation → finds strategy in **seconds to minutes on a single node**

---

## 12. FlexFlow Runtime

**Problem**: Existing DL frameworks (TensorFlow, PyTorch, Caffe2, MXNet) only natively support data parallelism (sample dimension).

**Solution**: FlexFlow implements its distributed runtime using **Legion** (a high-performance parallel runtime for distributed heterogeneous systems).

- Uses Legion's **high-dimensional partitioning interface** to support splitting tensors along any SOAP dimension
- Uses **cuDNN** and **cuBLAS** as underlying GPU libraries for DNN operators
- All strategies found in the SOAP space are executable through this runtime

---

## 13. Evaluation Results

### 13.1 Experimental Setup

- **6 DNN benchmarks**: AlexNet, Inception-v3, ResNet-101 (CNNs); RNNTC, RNNLM, NMT (RNNs)
- **2 GPU clusters**:
  - P100 cluster: 4 nodes, 4× Tesla P100 per node, 100 GB/s Infiniband
  - K80 cluster: 16 nodes, 4× Tesla K80 per node, 56 GB/s Infiniband
- Search time budget: **30 minutes** (search terminates in minutes in most cases)

### 13.2 Key Results

**Training throughput improvement** over data parallelism and expert-designed strategies:
- Up to **3.3× speedup** in per-iteration training throughput
- Up to **5× reduction** in communication costs (data transfers per iteration)
- Better **scalability** as number of GPUs increases

**vs. ColocRL** (RL-based device placement):
- FlexFlow: **3.4–3.8× faster** strategy found in seconds vs. hours

**vs. OptCNN** (dynamic programming for linear CNNs):
- FlexFlow: **1.2–1.6× faster** on non-linear networks (RNNs, etc.)
- Same performance on linear CNNs (OptCNN's strength)

**End-to-end training**: FlexFlow reduces Inception-v3 training time by **38%** compared to TensorFlow on 16 P100 GPUs.

### 13.3 How FlexFlow Achieves Better Performance

Two main advantages:

1. **Reducing communication costs**:
   - Data parallelism always synchronizes all weights → O(model_size) communication per iteration
   - FlexFlow finds strategies that only synchronize what's necessary → fewer/smaller transfers
   - For NMT on 64 K80 GPUs: **2–5.5× less data transferred** per iteration

2. **Reducing task computation time**:
   - Parallelizing different dimensions can reduce an operator's total computation
   - Example: Parallelizing matrix multiplication in the channel (parameter) dimension rather than batch (sample) dimension reduces total compute by **38%** for NMT

### 13.4 Simulator Accuracy

- For all measured executions, relative difference between real and simulated time: **< 30%**
- More importantly: simulated execution time **preserves ordering** of real execution time → simulator reliably identifies better strategies even if absolute values differ

---

## 14. Case Studies

### 14.1 Inception-v3 on 4 P100 GPUs

FlexFlow discovers a strategy that:
- Uses **intra-operator parallelism** for operators on the **critical path** (most important bottleneck)
- Uses a **combination of intra- and inter-operator parallelism** for operators on different branches
- Result: reduces parameter synchronization costs by **75%** and per-iteration execution time by **12%** vs. data parallelism

**Asymmetric cluster adaptation**: On K80 cluster (GPUs with asymmetric connections), FlexFlow tends to parallelize operators on adjacent GPUs with direct connections to minimize communication costs. This topology-awareness is something human-designed strategies miss.

### 14.2 NMT on 4 P100 GPUs

FlexFlow discovers a 3-tier strategy:
1. **Layers with many params but little compute** (embedding layers): compute on a **single GPU** — no parameter synchronization overhead
2. **Layers with many params and heavy compute** (softmax): use **parameter dimension parallelism** — each device computes for a subset of parameters, reducing sync while maintaining balance
3. **Multiple recurrent layers** (LSTM, attention): use **concurrency across layers** + **within-operator parallelism** — reduces param sync while balancing load

---

## 15. Why Data/Model Parallelism Alone are Suboptimal — Summary

| Issue | Data Parallelism | Model Parallelism |
|-------|-----------------|-------------------|
| **Memory** | Model must fit in one GPU ❌ | Model can be distributed ✓ |
| **Communication** | O(model_size) sync every iteration ❌ | No param sync (but data transfer at split points) ✓ |
| **Utilization** | High (all GPUs always busy) ✓ | Low (sequential layer execution) ❌ |
| **Generality** | Works for any model ✓ | Complex placement for non-linear models ❌ |
| **Scalability** | Bandwidth bottleneck at PS ❌ | Limited by sequential depth ❌ |

**SOAP / FlexFlow addresses all these** by searching over a much larger space of strategies that can mix and match different parallelism types per-operator.

---

## 16. Complete Summary: Key Takeaways

1. **DNN training is iterative and expensive**: Forward pass + backprop + weight update, repeated thousands of times on TBs of data with GB-scale models.

2. **Data Parallelism**: Each GPU gets a full model copy but trains on subset of data. Requires gradient synchronization (via PS or AllReduce). Bottleneck: O(N) bandwidth at PS, or O(log N) with AllReduce. **Critical limitation**: model must fit in one GPU.

3. **Parameter Server**: Stores model weights, workers push/pull gradients. Bandwidth grows linearly with workers → bottleneck. Synchronization modes: Sequential (BSP), Eventual, Bounded Delay.

4. **MPI All-Reduce**: Communication ring reduces PS bandwidth bottleneck from O(N) to O(1) but requires O(log N) steps. Used by Horovod.

5. **Model Parallelism**: Split model across GPUs. Enables training large models. Naive version has low GPU utilization (~25% for 4 GPUs).

6. **Pipeline Parallelism (GPipe)**: Split mini-batches into micro-batches, overlap their execution across pipeline stages. Reduces idle time from 1-(1/p) to (p-1)/m. Trade-off: more activation memory.

7. **3D Parallelism (DeepSpeed)**: Combines data + pipeline + tensor parallelism. Each addresses different bottleneck.

8. **SOAP (FlexFlow paper)**: 4D search space — Sample, Operator, Attribute, Parameter. Data parallelism = S only; model parallelism = O+P; FlexFlow = all four + hybrids.

9. **FlexFlow**: Uses MCMC search + execution simulator to find best SOAP strategy automatically. 3.3× better throughput, 5× less communication than data/model parallelism.

10. **Delta Simulation**: Key optimization — re-simulate only changed tasks (2.2–6.9× faster than full simulation). Critical for making MCMC search practical.

---

## References

1. Jia, Z., Zaharia, M., Aiken, A., "Beyond Data and Model Parallelism for Deep Neural Networks", MLSys 2019
2. Ben-Nun, T., Hoefler, T., "Demystifying Parallel and Distributed Deep Learning", ACM Comput. Surv. 2019
3. CMU 15-418: Parallel Computer Architecture and Programming
4. MIT 6.S965: TinyML and Efficient Deep Learning Computing


# Lecture 3.3: Federated Learning

## DS256 - Scalable Systems for Data Science
### Module 3: Machine Learning at Scale

---

## 1. Cloud Computing

Cloud computing provides on-demand access to large-scale computing infrastructure without the need to own or manage physical hardware.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLOUD COMPUTING MODEL                         │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │              Large Data Centers                               │  │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │
│   │   │  Rack 1  │  │  Rack 2  │  │  Rack 3  │  │  Rack N  │   │  │
│   │   │ 100s srv │  │ 100s srv │  │ 100s srv │  │ 100s srv │   │  │
│   │   └──────────┘  └──────────┘  └──────────┘  └──────────┘   │  │
│   │       1000s of racks → 100k+ servers                         │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   Key Properties:                                                    │
│   • On-demand: Rent VMs by the minute                               │
│   • 100 machines for 10 mins == 1 machine for 1000 mins (same cost) │
│   • Economies of scale → much cheaper than on-premises              │
│   • Reduced operations costs (energy, personnel)                    │
│   • Capital costs (servers) are fully utilized                      │
│                                                                      │
│   Traditional Approach: Move data → Cloud → Process/Store           │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Advantages:**
- **Cost Efficiency**: Pay-per-use model; economies of scale reduce cost
- **Elasticity**: Scale up/down on demand
- **Fully Utilized Infrastructure**: Shared infrastructure ensures no idle servers
- **Reduced Operational Burden**: No physical hardware management

---

## 2. Challenges in Centralized Training

Traditional ML training requires all data to be centralized in a single cluster. This creates serious problems:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CENTRALIZED TRAINING PROBLEMS                      │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Central Server / Cluster                  │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │  ← ALL DATA MUST FLOW HERE           │
│             ┌───────────────┼───────────────┐                       │
│             ▼               ▼               ▼                       │
│      ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│      │ Hospital A │  │ Hospital B │  │  Phone 1   │                │
│      │ patient    │  │ patient    │  │ keyboard   │                │
│      │ records    │  │ records    │  │   logs     │                │
│      └────────────┘  └────────────┘  └────────────┘                │
│                                                                      │
│  Problem 1: DATA PRIVACY                                            │
│   • Medical records, financial data — legally/ethically cannot      │
│     leave the source                                                │
│   • Keyboard autocorrection — highly personal text                  │
│                                                                      │
│  Problem 2: GRADIENT LEAKAGE                                        │
│   • Sharing gradient loss can leak private raw data from clients    │
│                                                                      │
│  Problem 3: PARAMETER SERVER LIMITATIONS                            │
│   • Parameter Server works well in a data center                    │
│   • Poor for wide-area distribution across 1000s of mobile devices  │
└─────────────────────────────────────────────────────────────────────┘
```

**Core Challenges:**
- **Data Privacy**: Personal/sensitive data cannot be transmitted to a central location (e.g., medical records, smartphone keyboard input)
- **Gradient Leakage**: Even sharing gradient updates can indirectly expose private training data
- **Scale Mismatch**: Parameter Server architecture designed for data centers performs poorly across thousands of geographically distributed edge devices

---

## 3. Edge & Fog Computing

Edge and fog computing bring compute resources closer to the data source, reducing latency and enabling local processing.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     COMPUTING HIERARCHY                              │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                       CLOUD                                  │  │
│   │              Large centralized data centers                  │  │
│   └───────────────────────────┬─────────────────────────────────┘  │
│                               │                                      │
│   ┌───────────────────────────▼─────────────────────────────────┐  │
│   │                   FOG / MICRO DATA CENTERS                   │  │
│   │  • Accelerated workstations (e.g., NVIDIA TX1)               │  │
│   │  • Small clusters serving a community or city                │  │
│   │  • Like a Content Distribution Network (CDN)                 │  │
│   └─────────────┬──────────────────────┬───────────────────────┘   │
│                 │                      │                             │
│   ┌─────────────▼──────────┐  ┌───────▼──────────────────────┐    │
│   │       EDGE DEVICES     │  │      EDGE DEVICES             │    │
│   │  • IoT micro-controllers│  │  • Smart phones               │    │
│   │  • RPi-class gateways  │  │  • Edge TPU/GPU accelerators  │    │
│   │  • Sensor motes        │  │  • Almost free compute        │    │
│   └────────────────────────┘  └───────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Characteristics:**

| Layer | Examples | Role |
|-------|----------|------|
| **Edge** | IoT sensors, RPi gateways, smartphones, Edge TPU/GPU | Local data collection and lightweight processing |
| **Fog** | Accelerated workstations, small clusters (TX1) | Mid-tier aggregation, serves community/city |
| **Cloud** | Large data centers | Global aggregation, heavy computation |

- Edge and fog computing **complement** cloud computing — they don't replace it
- Edge devices are often **nearly free** (captive, already deployed)
- Fog nodes serve as intermediaries between edge and cloud (like CDN for compute)

---

## 4. Training on the Edge: Federated Learning

The abundance of powerful edge devices creates an opportunity: instead of sending data to the cloud, we can **train models locally on edge devices** and only share model updates.

**Use Cases:**
- **Keyboard auto-suggestion**: User corrections stay on device
- **Siri/Alexa suggestions**: Voice command corrections learned locally
- **Photo/video tagging**: Entity labeling from personal media

**Core Idea:**
```
┌─────────────────────────────────────────────────────────────────────┐
│                  MOTIVATION FOR FEDERATED LEARNING                   │
│                                                                      │
│  Instead of:  Data  →  Cloud  →  Train                              │
│                                                                      │
│  Do:          Train locally on edge devices                         │
│               Share only model updates (not raw data)               │
│               Aggregate updates in the cloud                        │
│                                                                      │
│  Benefits:                                                           │
│   ✓ Opportunistic use of captive compute (already on device)        │
│   ✓ Data privacy preserved — raw data never leaves the device       │
│   ✓ Avoids centralization of sensitive personal data                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Federated Learning: Explanation

Federated Learning is a distributed machine learning paradigm where multiple clients **collaboratively train a shared global model** without sharing raw data.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   FEDERATED LEARNING OVERVIEW                        │
│                                                                      │
│                        ┌────────────┐                               │
│                         │   Server   │                               │
│                        └─────┬──────┘                               │
│                              │                                       │
│              ① Download global model from server                    │
│                              │                                       │
│         ┌────────────────────┼─────────────────────┐               │
│         ▼                    ▼                      ▼               │
│   ┌──────────┐        ┌──────────┐          ┌──────────┐           │
│   │ Client 1 │        │ Client 2 │          │ Client N │           │
│   │Local Data│        │Local Data│          │Local Data│           │
│   └──────────┘        └──────────┘          └──────────┘           │
│         │                    │                      │               │
│         └────────────────────┼─────────────────────┘               │
│                              │                                       │
│   ② (Re)train local model using global model + local data           │
│   ③ Upload local models → Server aggregates into global model       │
│                                                                      │
│   Repeat until convergence                                          │
└─────────────────────────────────────────────────────────────────────┘
```

**In each iteration:**
1. Clients **download** the current global model from the server
2. Each client **retrains** locally using its own private data
3. Clients **upload** updated local models to the server
4. Server **aggregates** all local models into a new global model
5. Process **repeats** for multiple rounds until convergence

---

## 6. Federated Machine Learning: Step-by-Step

### Step 1: Forward Pass
```
┌─────────────────────────────────────────────────────────────────────┐
│  • All edges start with same initial (random) model                  │
│  • Perform Forward Pass of training using local data (mini-batch)   │
│                                                                      │
│        Central Server                                               │
│             │                                                        │
│      ┌──────┴────────┐                                              │
│      │  Global Model │  ──────────────── Gradients ──────────────── │
│      └───────────────┘                                              │
│             │ Forward Pass                                           │
│    ┌────────┼────────┬────────┐                                     │
│   Edge0   Edge1   Edge2    Edge3                                    │
│  Data 0  Data 1  Data 2   Data 3                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 2: Backward Pass
```
┌─────────────────────────────────────────────────────────────────────┐
│  • Apply the gradients to the local model (Backward Pass)           │
│  • Each edge now has a different local model trained on its data    │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 3: Repeat for Local Epochs
```
┌─────────────────────────────────────────────────────────────────────┐
│  • Train on each edge for a fixed number of local epochs            │
│  • Each edge's model diverges from others (local specialization)    │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: Aggregate at Server
```
┌─────────────────────────────────────────────────────────────────────┐
│  • After fixed epochs: transfer all local models to central server  │
│  • Server averages all local models → new global model              │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 5: Distribute and Repeat
```
┌─────────────────────────────────────────────────────────────────────┐
│  • Server sends aggregated global model back to all edges           │
│  • Edges repeat local training with new global model                │
│  • Process continues for several rounds until convergence           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Federated Learning with Model Averaging (FedAvg)

**FedAvg** is the foundational algorithm for Federated Learning, introduced by McMahan et al. It uses **weighted model averaging** across clients.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FedAvg ALGORITHM                                 │
│                                                                      │
│  Server:                                                             │
│    Initialize global model w₀                                       │
│    For each round t = 1, 2, ..., T:                                 │
│      Select subset S of clients                                      │
│      Broadcast wₜ to all clients in S                               │
│      For each client k in S (in parallel):                          │
│        wₜ₊₁ᵏ ← ClientUpdate(k, wₜ)     ← local training           │
│      wₜ₊₁ ← Σₖ (nₖ/n) · wₜ₊₁ᵏ          ← weighted average        │
│                                                                      │
│  ClientUpdate(k, w):                                                 │
│    Partition local data into mini-batches B                         │
│    For each local epoch e = 1, ..., E:                              │
│      For each batch b in B:                                         │
│        w ← w − η · ∇ℓ(w; b)             ← SGD step                │
│    Return w                                                          │
│                                                                      │
│  Weight: nₖ = # samples on client k,  n = total samples            │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Parameters:**
- **C**: Fraction of clients selected per round
- **E**: Number of local training epochs per round
- **B**: Local mini-batch size
- **η**: Learning rate

---

## 8. Cross-Silo vs Cross-Device Settings

Federated Learning operates in two fundamentally different deployment settings:

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                     CROSS-SILO vs CROSS-DEVICE COMPARISON                            │
├─────────────────────┬──────────────────────────────┬──────────────────────────────┤
│ Feature             │ Cross-Silo FL                │ Cross-Device FL              │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Clients             │ Institutions (hospitals,     │ Mobile devices, IoT sensors  │
│                     │ banks, enterprises)          │                              │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ # of Clients        │ Small (10–100)               │ Large (1000s to millions)    │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Availability        │ Stable (always online)       │ Intermittent connections     │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Data Size           │ Large datasets per client    │ Small datasets per device    │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Compute Power       │ High (data centers, cloud)   │ Low (resource-constrained)   │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Data Distribution   │ More homogeneous             │ Highly heterogeneous         │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Participation       │ Regular and consistent       │ Sporadic and unpredictable   │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Network             │ Stable, high bandwidth       │ Unstable, low bandwidth      │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Privacy Concerns    │ Regulatory (GDPR, HIPAA)     │ Device-level (personal data) │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Resource Limits     │ Few (high-end servers)       │ Many (low CPU, battery)      │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Fault Tolerance     │ Easier (fewer clients)       │ Harder (frequent dropouts)   │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Security Threats    │ Data breaches, espionage     │ Poisoning, adversarial upd.  │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Client Selection    │ All clients participate      │ Client sampling by avail.    │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Applications        │ Healthcare, finance, gov.    │ Mobile apps, IoT, smart home │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Typical Models      │ Large, complex (NNs)         │ Lightweight (MobileNets)     │
├─────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Infrastructure      │ Data centers, stable cloud   │ Edge devices, IoT, 5G        │
└─────────────────────┴──────────────────────────────┴──────────────────────────────┘
```

---

## 9. Horizontal FL vs Vertical FL

The two main data partitioning paradigms in Federated Learning differ in **how data is split** across participants:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  HORIZONTAL FEDERATED LEARNING (HFL)                 │
│                                                                      │
│  Feature Space (columns) — SAME across all clients                  │
│  Sample Space (rows) — DIFFERENT on each client                     │
│                                                                      │
│        Client A:          Client B:         Client C:               │
│       ┌───┬───┬───┐      ┌───┬───┬───┐    ┌───┬───┬───┐           │
│       │F1 │F2 │F3 │      │F1 │F2 │F3 │    │F1 │F2 │F3 │           │
│       ├───┼───┼───┤      ├───┼───┼───┤    ├───┼───┼───┤           │
│       │r1 │ . │ . │      │r4 │ . │ . │    │r7 │ . │ . │           │
│       │r2 │ . │ . │      │r5 │ . │ . │    │r8 │ . │ . │           │
│       │r3 │ . │ . │      │r6 │ . │ . │    │r9 │ . │ . │           │
│       └───┴───┴───┘      └───┴───┴───┘    └───┴───┴───┘           │
│          Labels             Labels           Labels                  │
│                                                                      │
│  Example: Multiple hospitals with same health record schema          │
│           but different patient populations                          │
│  Algorithm: FedAvg — same model structure, aggregate updates        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                  VERTICAL FEDERATED LEARNING (VFL)                   │
│                                                                      │
│  Feature Space (columns) — DIFFERENT across clients                 │
│  Sample Space (rows) — SAME (same users/entities)                   │
│                                                                      │
│        Client A (Bank):   Client B (Insurance):                     │
│       ┌───┬───┐           ┌───┬───┬───┐                            │
│       │F1 │F2 │           │F3 │F4 │F5 │                            │
│       ├───┼───┤           ├───┼───┼───┤                            │
│       │r1 │ . │           │r1 │ . │ . │  ← same users              │
│       │r2 │ . │           │r2 │ . │ . │                            │
│       │r3 │ . │           │r3 │ . │ . │                            │
│       └───┴───┘           └───┴───┴───┘                            │
│                                    Labels (one party holds them)    │
│                                                                      │
│  Example: Bank + insurance co. sharing data about same clients      │
│           but with different attributes                              │
│  Algorithm: Exchange intermediate (encrypted) results for joint     │
│             model updates using Homomorphic Encryption (HE)         │
└─────────────────────────────────────────────────────────────────────┘
```

**Comparison Table:**

| Feature | Horizontal FL (HFL) | Vertical FL (VFL) |
|---------|--------------------|--------------------|
| Data Partitioning | Same schema, different rows | Different columns, same rows |
| Feature Overlap | Full overlap | No overlap |
| Sample Overlap | No overlap | Full overlap |
| Privacy Technique | Differential privacy, secure aggregation | Homomorphic Encryption (HE) |
| Training Method | FedAvg — same model structure | Secure computation, encrypted result exchange |
| Challenge | Client drift, data heterogeneity | High communication overhead, sample ID alignment |
| Use Case | Multi-hospital, multi-bank (same domain) | Cross-industry (bank + e-commerce) |
| Applications | Financial services, healthcare | Marketing, customer analytics, retail |

---

## 10. Aggregation Strategies

### 10.1 Synchronous Aggregation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNCHRONOUS AGGREGATION                           │
│                                                                      │
│  Server waits for ALL selected clients to finish before aggregating │
│                                                                      │
│  Timeline:                                                           │
│                                                                      │
│  Server:  ─────────────────[WAIT]─────────────────[AGGREGATE]──►   │
│                            ↑                       ↑                │
│  Edge 1:  ──────[TRAIN]────┘ (fast)               │                │
│  Edge 2:  ─────────────[TRAIN]────────────────────┘ (slow)         │
│  Edge 3:  ──────────[TRAIN]──────────────────────→ (medium)        │
│                                                                      │
│  • Round-time = slowest client's time (straggler bottleneck)        │
│  • Global model sees all clients' updates before proceeding         │
│  • FedAvg is the most popular synchronous strategy                  │
│                                                                      │
│  Pros:  Convergence guarantees, simpler analysis                    │
│  Cons:  Bottlenecked by slowest client (stragglers)                 │
│         Heterogeneous data → different label distributions          │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.2 Asynchronous Aggregation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ASYNCHRONOUS AGGREGATION                          │
│                                                                      │
│  Server updates global model immediately when ANY client finishes   │
│                                                                      │
│  Timeline:                                                           │
│                                                                      │
│  Server:  ──[UPDATE]──[UPDATE]──[UPDATE]──[UPDATE]──────────────►   │
│                ↑          ↑         ↑         ↑                     │
│  Edge 1:  ──[T]──────────────────────────────────── (updates often) │
│  Edge 2:  ─────────────[T]──────────────────────── (medium)        │
│  Edge 3:  ─────────────────────────────[T]──────── (straggler)     │
│                                                                      │
│  • No waiting → faster round completion                             │
│  • Straggler updates based on stale global model                    │
│  • Fast clients dominate → biased data distribution                 │
│  • FedAsync is the most popular asynchronous strategy               │
│                                                                      │
│  Pros:  Faster progress, no straggler blocking                      │
│  Cons:  Model divergence from stale updates                         │
│         Fast clients over-represented → skewed distribution         │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.3 Tiered Aggregation

```
┌─────────────────────────────────────────────────────────────────────┐
│                      TIERED AGGREGATION                              │
│                                                                      │
│  Clients grouped into tiers based on a metric (latency, data dist.) │
│                                                                      │
│       ┌───────────────────────────────────────────────┐            │
│       │                   SERVER                       │            │
│       └────────────────────┬──────────────────────────┘            │
│                            │ Aggregate tiers (sync or async)        │
│       ┌────────────────────┼────────────────────┐                  │
│       ▼                    ▼                    ▼                   │
│  ┌─────────┐          ┌─────────┐         ┌─────────┐             │
│  │ Tier 1  │          │ Tier 2  │         │ Tier 3  │             │
│  │ (fast)  │          │ (med.)  │         │ (slow)  │             │
│  └────┬────┘          └────┬────┘         └────┬────┘             │
│       │ sync               │ sync               │ sync             │
│  ┌────┴────┐          ┌────┴────┐         ┌────┴────┐             │
│  │Clients  │          │Clients  │         │Clients  │             │
│  │(similar │          │(similar │         │(similar │             │
│  │latency) │          │latency) │         │latency) │             │
│  └─────────┘          └─────────┘         └─────────┘             │
│                                                                      │
│  TiFL: Tiers by response latency → reduces straggler effect         │
│  HACCS: Tiers by data similarity → ensures label coverage per round │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Points:**
- Clients within a tier train **synchronously**
- Updates across tiers may be aggregated **synchronously or asynchronously**
- **TiFL**: Groups clients by similar latencies; selects one tier per round → no straggler problem
- **HACCS**: Groups by dataset similarity; selects few from each tier → global model sees all labels every round

### 10.4 Hierarchical Aggregation

```
┌─────────────────────────────────────────────────────────────────────┐
│                   HIERARCHICAL AGGREGATION                           │
│                                                                      │
│  Three levels: Global Server → Edge Nodes → Clients                 │
│                                                                      │
│              ┌──────────────────────────┐                           │
│              │       GLOBAL SERVER       │                           │
│              │  Aggregates edge models   │                           │
│              │  periodically             │                           │
│              └────────────┬─────────────┘                           │
│                           │ Global aggregation                       │
│         ┌─────────────────┼──────────────────┐                     │
│         ▼                 ▼                  ▼                      │
│   ┌───────────┐    ┌───────────┐    ┌───────────┐                  │
│   │ Edge Node │    │ Edge Node │    │ Edge Node │                  │
│   │    1      │    │    2      │    │    3      │                  │
│   │(agg. tier)│    │(agg. tier)│    │(agg. tier)│                  │
│   └─────┬─────┘    └─────┬─────┘    └─────┬─────┘                  │
│         │ Local agg.      │ Local agg.      │ Local agg.            │
│    ┌────┴───┐       ┌────┴───┐       ┌────┴───┐                    │
│   C1   C2  C3     C4   C5  C6     C7   C8  C9                      │
│  (local area)    (local area)    (local area)                       │
│                                                                      │
│  Clients → sync train → edge node (local agg.) → server (global)   │
│  Simplest implementation: HierFAVG (sync at both levels)            │
└─────────────────────────────────────────────────────────────────────┘
```

**How It Works:**
1. Clients in each tier train synchronously and send updates to their assigned edge node
2. Edge node performs **local aggregation** of its clients' models
3. Server periodically aggregates edge node models into a new global model
4. Supports sync or async aggregation between edge nodes and server

### 10.5 Aggregation Strategy Comparison

| Strategy | Waiting Behaviour | Straggler Impact | Convergence | Key Algorithm |
|----------|-------------------|------------------|-------------|---------------|
| **Synchronous** | Waits for ALL clients | High (bottlenecked) | Well-studied guarantees | FedAvg |
| **Asynchronous** | No waiting | Low (stragglers ignored) | Risk of divergence | FedAsync |
| **Tiered** | Sync within tier, async/sync across tiers | Reduced (similar-latency tiers) | Better label coverage | TiFL, HACCS |
| **Hierarchical** | Sync within tier, periodic server agg. | Managed at edge node level | HierFAVG guarantees | HierFAVG |

---

## 11. Client Selection Strategies in FL

### 11.1 Why Client Selection?

In realistic FL deployments, the number of clients ranges from **thousands to millions**. Not all clients can or should participate in every round.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CLIENT SELECTION CHALLENGES                        │
│                                                                      │
│  Problem:                                                            │
│  • Selecting all clients → massive communication bottleneck         │
│  • Selecting too few → global model misses some data labels         │
│  • Clients vary widely in data, compute, and reliability            │
│                                                                      │
│  Three Dimensions of Heterogeneity:                                 │
│                                                                      │
│  1. DATA HETEROGENEITY (Statistical Heterogeneity)                  │
│     • Varying label distributions across clients (non-IID data)     │
│     • Different # samples per label and overall data volume         │
│                                                                      │
│  2. DEVICE HETEROGENEITY (System Heterogeneity)                     │
│     • Compute capability varies by orders of magnitude              │
│       (e.g., Samsung Galaxy M04 vs Galaxy S23 Ultra)                │
│     • Capabilities change over time (battery, thermal throttling)   │
│                                                                      │
│  3. DEVICE UNRELIABILITY                                             │
│     • Unstable network connections                                   │
│     • Clients can crash or drop out mid-round                       │
│     • Availability changes (charging, network, usage state)         │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 Client Sampling Strategies

#### OORT
```
┌─────────────────────────────────────────────────────────────────────┐
│  OORT (Guided Participant Selection for Training)                    │
│                                                                      │
│  Defines a "utility" metric for each client:                        │
│    Utility(k) = f(loss_k, latency_k)                                │
│                                                                      │
│  • Higher loss → model learns more from this client → higher util.  │
│  • Lower latency → faster round completion → higher utility         │
│  • Selects subset with the HIGHEST utility every round              │
│                                                                      │
│  Goal: Balance training efficiency and statistical utility           │
└─────────────────────────────────────────────────────────────────────┘
```

#### TiFL (Tiered Federated Learning)
```
┌─────────────────────────────────────────────────────────────────────┐
│  TiFL                                                                │
│                                                                      │
│  • Tiers clients by RESPONSE LATENCY                                │
│  • Clients with similar latencies → same tier                       │
│  • One tier selected per round                                      │
│  • All clients in the selected tier participate                     │
│                                                                      │
│  Effect: Reduces straggler problem (all clients in a tier are       │
│          similarly fast); consistent round duration                  │
└─────────────────────────────────────────────────────────────────────┘
```

#### HACCS (Heterogeneity-Aware Client-Constrained Selection)
```
┌─────────────────────────────────────────────────────────────────────┐
│  HACCS                                                               │
│                                                                      │
│  • Tiers clients by DATASET SIMILARITY                              │
│  • Groups clients with similar data distributions together          │
│  • Selects a few clients from EACH tier every round                 │
│                                                                      │
│  Effect: Ensures global model sees ALL available labels each round  │
│          → Approximates IID training even with non-IID data         │
└─────────────────────────────────────────────────────────────────────┘
```

#### REFL (Resource-Efficient Federated Learning)
```
┌─────────────────────────────────────────────────────────────────────┐
│  REFL                                                                │
│                                                                      │
│  • Selects clients based on PROJECTED AVAILABILITY                  │
│  • Clients with the LOWEST probability of being available are       │
│    selected in the current round                                     │
│                                                                      │
│  Rationale: Prioritize rare-availability clients now; commonly      │
│  available clients can be scheduled in future rounds                │
│  Effect: Better utilization of all client data over time            │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.3 Client Sampling Strategies Summary

| Strategy | Selection Criterion | Addresses | Key Effect |
|----------|---------------------|-----------|------------|
| **OORT** | Highest utility (loss + latency) | Data + Device heterogeneity | Balances efficiency and learning gain |
| **TiFL** | Same-latency tier per round | Device heterogeneity (stragglers) | Consistent round duration |
| **HACCS** | Tiered by data similarity, sample from each | Data heterogeneity (non-IID) | Label coverage every round |
| **REFL** | Lowest projected availability | Device unreliability | Utilizes rare-availability clients |

---

## 12. Google's Federated Learning Approach

Google's production FL system is designed for **Android clients** at massive scale, powering products like **Gboard (keyboard autocorrect)**.

```
┌─────────────────────────────────────────────────────────────────────┐
│               GOOGLE FEDERATED LEARNING SYSTEM                       │
│                                                                      │
│  Data:    Stored on phone (never leaves device)                     │
│  Train:   TensorFlow executes training locally on device            │
│  Agg:     Federated aggregation on cloud servers                    │
│  Privacy: Secure aggregation (individual updates are masked)        │
│  Deploy:  Aggregated global model pushed back to phone              │
│                                                                      │
│  Design Focus:                                                       │
│   • Synchronous federated learning (trend away from async)          │
│   • Privacy protocols often rely on sync training                   │
│   • Mitigate performance effects of synchronous aggregation         │
│                                                                      │
│  Frameworks Available:                                               │
│   • Google TensorFlow Federated (TFF)                               │
│   • Android Private Compute Services / Federated Compute Platform   │
└─────────────────────────────────────────────────────────────────────┘
```

### Other Notable FL Frameworks

| Framework | Organization |
|-----------|-------------|
| Flower | Cambridge University |
| FedML | USC |
| FATE | Webank |
| PySyft | OpenMined / Oxford |
| Flotilla | IISc |
| TensorFlow Federated | Google |
| OpenFL | Intel |
| Federated Compute Platform | Google / Android |

---

## 13. Federated Learning at Scale: Protocol

### 13.1 High-Level Protocol

```
┌─────────────────────────────────────────────────────────────────────┐
│                   FL PROTOCOL OVERVIEW                               │
│                                                                      │
│  Key Concepts:                                                       │
│   • FL Population: The learning problem / application               │
│     (e.g., "Gboard next-word prediction")                           │
│   • FL Task: A specific computation for the FL population           │
│     (e.g., "one round of training")                                 │
│                                                                      │
│  Flow:                                                               │
│   1. Devices announce to FL server they can run an FL task          │
│   2. Server selects ~100s of devices from 10,000s that announce    │
│   3. Selected devices download the FL plan (TF graph) + global model│
│   4. Devices perform local computation on local data                │
│   5. Devices send FL checkpoint (output) back to server             │
│   6. Server applies updates to global state                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.2 Detailed Protocol: Selection → Configuration → Reporting

```
┌─────────────────────────────────────────────────────────────────────┐
│                   3-PHASE PROTOCOL                                   │
│                                                                      │
│  ┌─────────────┐   ┌─────────────────┐   ┌──────────────────┐     │
│  │  SELECTION  │──►│  CONFIGURATION  │──►│    REPORTING     │     │
│  └─────────────┘   └─────────────────┘   └──────────────────┘     │
│                                                                      │
│  SELECTION Phase:                                                    │
│   • Devices meeting eligibility criteria check in with server       │
│     (e.g., phone is charging, on unmetered WiFi)                    │
│   • Opens bi-directional network connection for liveness monitoring │
│   • Server uses RESERVOIR SAMPLING to pick k devices from n        │
│     (n is not known a priori — devices arrive dynamically)          │
│   • Unselected devices are asked to reconnect later                 │
│                                                                      │
│  CONFIGURATION Phase:                                                │
│   • Server sends FL plan (TF computation graph) to selected devices │
│   • Sends current global model parameters and FL checkpoints        │
│   • Devices begin local computation                                 │
│                                                                      │
│  REPORTING Phase:                                                    │
│   • Devices send updated FL checkpoints (model updates) to server  │
│   • Server aggregates updates into new global model                 │
│   • Round completes only if SUFFICIENT devices report               │
│   • If insufficient reports → round is DISCARDED                   │
│   • Stragglers are IGNORED                                          │
│   • Selection and Reporting use different time windows              │
│     (participant count, timeout, reporting count thresholds)        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 14. Pace Steering

Pace Steering manages the **rate at which devices reconnect** to the FL server, enabling graceful scaling.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PACE STEERING                                 │
│                                                                      │
│  Problem: Device availability is highly variable                    │
│   • Too few devices → cannot form a valid round                     │
│   • Too many devices → "thundering herd" overwhelms server          │
│   • Diurnal oscillations (day/night usage patterns)                 │
│                                                                      │
│  Solution: Server tells devices WHEN to reconnect                   │
│                                                                      │
│  Mechanisms:                                                         │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │ Scale UP   → Few devices available                            │ │
│   │             → Ask them to concurrently reconnect              │ │
│   │             → Ensure critical mass for a round                │ │
│   └──────────────────────────────────────────────────────────────┘ │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │ Scale DOWN → Many devices available                           │ │
│   │             → Force randomization across time windows         │ │
│   │             → Avoid "thundering herd" on server               │ │
│   └──────────────────────────────────────────────────────────────┘ │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │ Diurnal Mgmt → Handles day/night device availability swings  │ │
│   └──────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 15. Device Architecture

The on-device FL architecture separates application data from the FL runtime for security and isolation:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      DEVICE ARCHITECTURE                             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                   Application Data Store                      │  │
│  │       (e.g., SQLite DB with training samples)                 │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                Boundary     │  (strict separation)                   │
│  ┌──────────────────────────▼───────────────────────────────────┐  │
│  │                    FL Runtime                                  │  │
│  │  • Periodic trigger to become available for FL job            │  │
│  │  • Execute FL plan (TF computation graph)                     │  │
│  │  • Report results to server                                   │  │
│  │                                                                │  │
│  │  Supports multiple FL task types:                             │  │
│  │   • Training tasks                                            │  │
│  │   • Evaluation/validation tasks (validate trained models)     │  │
│  │                                                                │  │
│  │  Multi-tenancy:                                               │  │
│  │   • Multiple FL applications run on same device               │  │
│  │   • Prevents overloading across different FL populations      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Secure Aggregation (Privacy):                                       │
│   • Multi-party secure operations (masked models)                   │
│   • Aggregation performed over masked models                        │
│   • Final global model revealed only in the last step               │
│   • Protects individual client updates from server inspection       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 16. Operational Profile

The operational profile describes how Google's FL system **behaves in production**:

### 16.1 Device Availability

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DEVICE AVAILABILITY PATTERNS                     │
│                                                                      │
│  Device Count                                                        │
│      │                                                               │
│  High│      ██████████                                               │
│      │    ██          ██    (Night peak — charging, idle)            │
│      │  ██              ██                                           │
│  Low │██                  ████████████  (Day — user interaction)    │
│      └──────────────────────────────────────── Time (24h)           │
│                                                                      │
│  • Device availability follows diurnal (day/night) oscillations     │
│  • Drop-out rate is HIGHER during the day (user interaction)        │
│  • Eligibility changes based on: charging status, network,          │
│    user activity (screen off required)                              │
│  • Impacts: round completion rate                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.2 Round Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ROUND MANAGEMENT                                │
│                                                                      │
│  Overselection Strategy:                                             │
│   • Server selects MORE devices than needed for each round          │
│   • Excess devices are ABORTED once the goal is met                 │
│   • Compensates for dropout and device unavailability               │
│                                                                      │
│  Round Duration:                                                     │
│   • Round run time ≈ majority of device participation times         │
│   • Device participation time is CAPPED (hard timeout)              │
│   • Prevents slow stragglers from blocking indefinitely             │
│   • Round ends when enough devices complete (not all)               │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.3 Communication Profile

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMMUNICATION PROFILE                             │
│                                                                      │
│  Download (Server → Device):                                         │
│   • Downloads FL task plan (TF computation graph)                  │
│   • Downloads current global model parameters                       │
│   • Both are roughly similar in size                                │
│                                                                      │
│  Upload (Device → Server):                                           │
│   • Uploads updates (deltas) to the global model                    │
│   • Inherently MORE COMPRESSIBLE than the full model                │
│   • Upload size << Download size                                     │
│                                                                      │
│  Key Insight:                                                        │
│   DOWNLOAD dominates UPLOAD in bandwidth cost                       │
│   → Compression and efficient model distribution is critical        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 18. Practical Insight: FedAvg is Often Good Enough

A key takeaway from research and experimentation with FL:

> **"Strategies in literature don't work as claimed — FedAverage is often good enough!"**

Despite the many sophisticated client selection and aggregation strategies (OORT, TiFL, HACCS, REFL, FedAsync, tiered aggregation, etc.), empirical evaluations (e.g., from the Flotilla framework at IISc) show that:

- Complex strategies often fail to replicate their claimed improvements in practice
- Simple **FedAvg** (synchronous weighted averaging) is frequently competitive or better
- The overhead of complex strategies (communication, coordination) can outweigh their benefits

**Lesson**: Do not over-engineer the aggregation strategy until FedAvg is proven insufficient. This is analogous to how in data center ML, simple SGD with good hyperparameters often beats complex optimizers.

---

## 19. FL Server Architecture (Actor Model) — From the Paper

The Google FL server is built around the **Actor Programming Model** to handle massive scale (tens of millions of devices).

### 19.1 Actor Programming Model

**Actors** are independent concurrent computation units that communicate only via **message passing**:
- Each actor handles messages **strictly sequentially** → simple programming model, no shared memory bugs
- Multiple actor instances of the same type → natural horizontal scaling
- Actors can be co-located on one machine or **distributed across data centers**
- Ephemeral actors: created just for a round, destroyed after → dynamic resource management

### 19.2 The Four Key Actor Types

```
┌────────────────────────────────────────────────────────────────┐
│                    FL SERVER ARCHITECTURE                        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    COORDINATOR                           │   │
│  │  • Top-level actor; one per FL population               │   │
│  │  • Manages global synchronization and round advancement │   │
│  │  • Registers in shared locking service (single owner)   │   │
│  │  • Knows how many devices are connected to each Selector│   │
│  │  • Spawns Master Aggregators for each FL task           │   │
│  └──────────┬───────────────────────────┬──────────────────┘   │
│             │ creates                    │ coordinates           │
│  ┌──────────▼──────────┐     ┌──────────▼──────────────────┐   │
│  │   MASTER AGGREGATOR  │     │        SELECTOR(S)           │   │
│  │  • Manages rounds    │     │  • Accept device connections │   │
│  │    for one FL task   │     │  • Globally distributed      │   │
│  │  • Spawns Aggregators│     │    (close to devices)        │   │
│  │  • No state written  │     │  • Forward devices to        │   │
│  │    to disk until     │     │    Aggregators               │   │
│  │    fully aggregated  │     │  • Local accept/reject       │   │
│  └──────────┬───────────┘     │    decisions per population  │   │
│             │ creates         └─────────────────────────────┘   │
│  ┌──────────▼──────────┐                                        │
│  │    AGGREGATOR(S)     │                                        │
│  │  • Receives updates  │                                        │
│  │    from a subset of  │                                        │
│  │    devices           │                                        │
│  │  • Runs Secure Agg.  │                                        │
│  │    per group         │                                        │
│  │  • Ephemeral: lives  │                                        │
│  │    only for one round│                                        │
│  └─────────────────────┘                                        │
└────────────────────────────────────────────────────────────────┘
```

**Key Properties**:
- **No persistent logs of per-device updates** — state kept in-memory, aggregated ephemerally → prevents data center attacks on individual updates
- Selectors are globally distributed (deployed close to devices) to minimize latency
- Coordinator uses a shared locking service → exactly one Coordinator per FL population (prevents split-brain)

### 19.3 Pipelining of Rounds

An important optimization: the **Selection phase of round i+1 runs in parallel** with the Configuration/Reporting phases of round i.

```
Round i:     [Selection][Configuration ──── Reporting]
Round i+1:              [Selection][Configuration ──── Reporting]
                               ↑
                   Overlap! Selection runs continuously
                   (Selector actors are always running)
```

This is "free" — no extra complexity added, just a natural consequence of Selectors running continuously.

### 19.4 Failure Modes

The server continues making progress in all failure cases:

| Failure | Impact | Recovery |
|---------|--------|---------|
| **Aggregator crashes** | Only devices connected to it are lost | Round may still succeed with remaining devices |
| **Master Aggregator fails** | Current round fails | Coordinator restarts a new round from last committed checkpoint |
| **Coordinator dies** | Selectors detect it and respawn it | Exactly once, via shared locking service |

**In all cases**: no data is lost and the system restarts from the last committed checkpoint.

### 19.5 Device Attestation

To prevent malicious devices from corrupting the model:
- Uses **Android's remote attestation mechanism** to verify genuine devices and apps
- Provides protection against **data poisoning** via compromised devices
- Does NOT verify user identity (anonymous participation) — uses hardware attestation instead

---

## 20. Secure Aggregation — From the Paper

**Secure Aggregation** (Bonawitz et al., 2017) is a cryptographic protocol that makes individual device updates **uninspectable** by the server — the server only sees the aggregate sum, never individual updates.

### 20.1 What It Protects Against

- Protects from **"honest but curious"** attackers with access to Aggregator memory
- Individual device updates remain encrypted **even in memory** within the server
- Final global model is only revealed at the last step

### 20.2 Four-Round Protocol

```
┌─────────────────────────────────────────────────────────────────────┐
│              SECURE AGGREGATION — 4-ROUND PROTOCOL                   │
│                                                                       │
│  ROUND 1 & 2: PREPARE PHASE                                          │
│  ─────────────────────────────────────────────────────────────────  │
│  • Devices and server establish shared secrets (key agreement)       │
│  • Devices that drop out in this phase → NOT included in aggregation │
│  • Prepares cryptographic material for masking                       │
│                                                                       │
│  ROUND 3: COMMIT PHASE                                                │
│  ─────────────────────────────────────────────────────────────────  │
│  • Devices upload CRYPTOGRAPHICALLY MASKED model updates             │
│  • Server accumulates a sum of the masked updates                    │
│  • All devices who complete this round WILL have their update        │
│    included in the final aggregate (or the entire round fails)       │
│                                                                       │
│  ROUND 4: FINALIZATION PHASE                                          │
│  ─────────────────────────────────────────────────────────────────  │
│  • Devices reveal cryptographic secrets to allow server to UNMASK    │
│    the aggregated update                                              │
│  • Not all committed devices need to complete (robust to dropouts)  │
│  • If sufficient survivors → protocol succeeds                       │
│  • Server sees only: SUM(all_device_updates) — no individual updates │
└─────────────────────────────────────────────────────────────────────┘
```

### 20.3 Scaling Limitation and Solution

**Problem**: Secure Aggregation has **quadratic cost** with number of users (especially server-side computation). In practice, this limits a single Secure Aggregation group to **hundreds of users**.

**Solution**: Run one Secure Aggregation instance **per Aggregator actor**:
- Each Aggregator handles ≥ k devices securely → intermediate encrypted sum
- Master Aggregator combines intermediate sums without Secure Aggregation
- Result: scales to thousands of devices per round

---

## 21. Analytics and Monitoring — From the Paper

### 21.1 Why Analytics Matters

In FL, individual training examples on devices are **never inspectable** by the server. This creates an observability challenge — you can't debug by looking at the data.

Solution: Collect **aggregate health statistics** (no PII) and **training state logs**.

### 21.2 What is Monitored

**Device-side logging** (no PII):
- Device state when training was activated
- How often and how long training ran
- Memory usage
- Errors detected
- Phone model / OS / FL runtime version

**Server-side logging**:
- Devices accepted and rejected per round
- Timing of each protocol phase
- Upload/download throughput
- Error counts and types

### 21.3 Session Shape Visualizations

Training round outcomes are encoded as compact "session shape" strings:

| Symbol | Meaning |
|--------|---------|
| `-` | Checked in to FL server |
| `v` | Downloaded FL plan |
| `[` | Training started |
| `]` | Training completed |
| `+` | Upload started |
| `^` | Upload completed (success) |
| `#` | Upload rejected (late — round already closed) |
| `!` | Interrupted (device left idle state) |

**Example session shapes from production**:

| Session Shape | Meaning | Count | % |
|--------------|---------|-------|---|
| `-v[]+^` | Complete success | 1,116,401 | 75% |
| `-v[]+#` | Trained but upload rejected (straggler) | 327,478 | 22% |
| `-v[!` | Interrupted during training | 29,771 | 2% |

**Insight**: 75% of clients complete successfully; 22% are stragglers rejected after deadline. This justifies the overselection strategy (selecting 130% of target devices).

---

## 22. Tools and Workflow — From the Paper

### 22.1 Model Engineer Workflow

```
Development Environment:              Production Environment:
┌──────────────────┐                  ┌──────────────────┐
│  Model Program   │                  │    FL Server     │
│  (Python + TF)   │──generate plan──▶│                  │
│                  │                  │  ─download──▶    │
│  ─simulate──▶    │                  │  plan & model    │
│  (proxy data +   │                  │                  │
│  simulated FL    │◀──upload──────── │  ◀──upload──     │
│  server)         │  model & metrics │  model & metrics │
└──────────────────┘                  └──────────────────┘
```

### 22.2 FL Plan

Each FL task is compiled into an **FL Plan** — a data structure encoding all necessary computation:

**FL Plan has two parts**:
1. **Device portion**: TensorFlow graph, data selection criteria from example store, batching instructions, number of epochs, node labels for weight loading/saving
2. **Server portion**: Aggregation logic (how to combine device updates)

**Why not Python directly?** FL plans must run on devices (Android) and servers without a Python interpreter — the plan describes computation independently of Python.

### 22.3 Simulation with Proxy Data

During development, models are tested using **proxy data**:
- Real on-device data can't be inspected (privacy)
- Proxy data has the same **shape/structure** but different distribution
- Example: Wikipedia text as proxy for mobile keyboard text
- Simulation runs the same code as on-device, communicates with a simulated FL server

### 22.4 Versioning Challenge

On-device ML faces a unique versioning problem:
- Devices may run a TF runtime **months older** than the FL plan
- Old runtime may be missing operators or have incompatible signatures
- Solution: generate **versioned FL plans** — each derived from the default plan via graph transformations for compatibility
- Versioned plans pass the same release tests as the default (semantically equivalent)
- About **one incompatible change fixable per 3 months** in practice

### 22.5 Deployment Requirements

An FL task is only accepted for deployment if:
1. Built from **auditable, peer-reviewed code**
2. Bundled **test predicates** that pass in simulation
3. Resource consumption within **safe range** for target population
4. Tests pass on **every supported TF runtime version** (tested in Android emulator)

---

## 23. Applications — From the Paper

FL applies best when:
- On-device data is more relevant / recent than server data
- Data is privacy-sensitive or infeasible to transmit
- Labels are inferred from user behavior (clicks, typed words)

### 23.1 On-Device Item Ranking

Apps select/rank items from on-device inventory (e.g., settings search on Google Pixel).
- Ranking happens on-device → no server round-trip, lower latency, private queries
- User interactions become labeled training data

### 23.2 Content Suggestions for Keyboards (Gboard)

- Gboard uses FL to train models for **triggering** suggestion features and **ranking** suggested items
- All keyboard text stays on device

### 23.3 Next Word Prediction (Key Example with Numbers)

Gboard trained an **RNN for next-word prediction** using the FL platform:

| Metric | Value |
|--------|-------|
| Model parameters | ~1.4 million |
| FL rounds to convergence | 3,000 rounds |
| Time to convergence | ~5 days (2-3 min per round) |
| Unique users | ~1.5 million |
| Sentences processed | 6 × 10⁸ |
| Top-1 recall (n-gram baseline) | 13.0% |
| Top-1 recall (FL model) | **16.4%** |
| Comparison | Matches server-trained RNN on same task |

**Key result**: FL model outperforms both the n-gram baseline and the server-trained RNN in live A/B experiments — **despite never having access to raw user data**.

**Note**: FL training was ~7× slower than comparable datacenter training, but this is acceptable since the goal is to train on data that cannot be in the datacenter.

---

## 24. Operational Profile — Detailed Numbers From the Paper

### 24.1 Scale

- **Total FL population**: ~10 million daily active devices across several applications
- **Simultaneous participants**: up to **10,000 devices** at any time (filtered by eligibility + pace steering)
- **Diurnal variation**: up to **4× difference** between day and night participation (US-centric population)

### 24.2 Dropout and Overselection

- **Device dropout rate**: 6–10% (computation errors, network failures, eligibility changes)
- **Overselection factor**: Server selects **130% of target** devices to compensate for dropouts
- Excess devices are aborted once the target count is reached

### 24.3 Round Completion

- **75%** of devices complete training and upload successfully
- **22%** complete training but are **rejected** (upload arrives after the reporting window closes)
- **2%** are **interrupted** before completing (device left idle state)

### 24.4 Network Asymmetry

- **Download dominates upload**: each device downloads both the FL plan **and** the global model (similar sizes)
- Upload contains only model **updates** (deltas) — inherently more compressible than the full model
- Model updates (∆) are more amenable to compression: `∆_k = n_k × (w_trained - w_initial)`

---

## 25. FedAvg Algorithm — Formal Pseudocode (From Paper Appendix)

```
Server executes:
    Initialize w₀
    For each round t = 1, 2, ...:
        Select 1.3K eligible clients  ← overselect to compensate dropouts
        Wait for updates from K clients

        For client k:  (∆_k, n_k) ← ClientUpdate(w_t)

        w̄_t = Σ_k ∆_k         ← sum of weighted updates
        n̄_t = Σ_k n_k          ← sum of weights
        ∆_t = w̄_t / n̄_t       ← average weighted update
        w_{t+1} = w_t + ∆_t    ← apply update

ClientUpdate(w):
    B ← local data divided into mini-batches
    n ← |B|              ← number of local examples (used as weight)
    w_init ← w
    For each batch b in B:
        w ← w - η ∇ℓ(w; b)     ← local SGD step
    ∆ ← n · (w - w_init)        ← weighted update (NOT the weights themselves)
    Return (∆, n) to server
```

**Important**: The server aggregates **weighted updates** ∆_k (= n_k × weight_change), not the raw weights. This is:
- More numerically stable
- More compressible (sparse updates vs. dense model weights)
- Naturally proportional to data size on each device

---

## 26. Open Problems and Future Work — From the Paper

### 26.1 Bias in Participation

**Problem**: FL assumes all devices are equally likely to participate. In practice:
- Only devices that are **charging + on unmetered WiFi** participate
- In some countries, few people have access to unmetered networks
- Only devices with **recent Android + ≥ 2GB RAM** participate
- This may cause the global model to reflect a specific demographic

**Current mitigation**: Models are evaluated via live **A/B experiments** after training; if bias causes inferior models, it's detected there.

### 26.2 Convergence Time

- FL is **~7× slower** than equivalent datacenter training (for the Gboard next-word prediction model)
- Current algorithms (FedAvg) can only efficiently use **hundreds of devices** in parallel — but millions are available
- Need new algorithms that scale parallelism to much larger cohorts

### 26.3 Device Scheduling

- Multi-tenant scheduler uses a simple worker queue — blind to app usage patterns
- Can end up training repeatedly on old data for some apps while neglecting fresh data for frequently-used apps

### 26.4 Bandwidth

- Some model updates (e.g., RNN language model) can be **larger than the raw data** they're trained on
- Solution: compression techniques (quantization, sketched updates)

### 26.5 Federated Computation (Beyond ML)

**Federated Computation** generalizes the FL architecture beyond machine learning:
- Same basic principles (bring computation to data)
- But supports **general MapReduce-like workloads**, not just TF model training

**Federated Analytics**: Monitor aggregate device statistics without sending raw device logs to the cloud — a key near-term application of Federated Computation.

**Analogy**: FL is to Gboard what MapReduce is to Google Search — the platform enables a class of applications, not just one.

---

## Summary: Federated Learning End-to-End

```
┌─────────────────────────────────────────────────────────────────────┐
│                  FEDERATED LEARNING — BIG PICTURE                    │
│                                                                      │
│  WHY: Privacy, data locality, regulatory compliance                 │
│  WHAT: Collaborative training without sharing raw data              │
│  HOW:  Local training + model update aggregation                    │
│                                                                      │
│  Key Design Choices:                                                 │
│  ┌───────────────┬──────────────────────────────────────────────┐  │
│  │ Dimension     │ Options                                        │  │
│  ├───────────────┼──────────────────────────────────────────────┤  │
│  │ FL Setting    │ Cross-Silo (10s of orgs) vs Cross-Device (M) │  │
│  │ Data Split    │ Horizontal FL vs Vertical FL                  │  │
│  │ Aggregation   │ Sync → Async → Tiered → Hierarchical         │  │
│  │ Client Select │ Random → OORT → TiFL → HACCS → REFL          │  │
│  │ Privacy       │ Secure Aggregation, Differential Privacy, HE  │  │
│  └───────────────┴──────────────────────────────────────────────┘  │
│                                                                      │
│  Production Example (Google Gboard):                                │
│   Phone (TF local train) → Masked update → Cloud Aggregation       │
│   → New global model → Push back to phones                         │
│   Governed by: Eligibility, Pace Steering, Overselection           │
│                                                                      │
│  Paper's Server: Actor model (Coordinator → Selector →              │
│   Master Aggregator → Aggregator). All ephemeral, in-memory.       │
│                                                                      │
│  Bottom Line: FedAvg is often good enough!                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

*Reference: Towards Federated Learning at Scale: System Design, Keith Bonawitz et al., SysML 2019*
*Lecture Source: DS256 Module 3 — Machine Learning at Scale, Slides 63–116*


# Lecture 3.4: Distributed Training of Graph Neural Networks

## DS256 - Scalable Systems for Data Science
### Module 3: Machine Learning at Scale

> **Prepared by**: Pranjal Naman
> **Reference**: DistDGL paper — https://arxiv.org/pdf/2010.05337

---

## 1. Why Graphs? (Motivation)

Before diving into GNNs, it's worth understanding *why* graphs matter as a data structure and why they require special treatment.

### 1.1 Graphs Are Everywhere

A **graph** is a mathematical structure consisting of:
- **Nodes (vertices)**: entities in the system
- **Edges**: relationships or connections between entities

Almost everything around us can be represented as a graph:

| Domain | Nodes | Edges |
|--------|-------|-------|
| Social Networks | Users | Friendships/follows |
| Payment Networks | Accounts | Transactions |
| Chemical Networks (Molecules) | Atoms | Chemical bonds |
| Citation Networks | Papers | Citations |
| Traffic Networks | Intersections | Roads |
| Program Flows | Code blocks | Control flow |

**Even "non-graph" data can be modeled as graphs!**
- **Images**: Each pixel is a node, connected to its spatial neighbors → forms a grid graph
- **Text**: Tokens (words/characters) are nodes, connected sequentially → forms a chain/path graph
- **Molecules**: Atoms as nodes, bonds as edges — crucial for drug discovery

> **Key Insight**: Graphs are the most *general* data structure — lists, sequences, and grids are all special cases of graphs. This generality is why GNNs are so powerful.

### 1.2 Industrial Popularity

GNNs are not just academic — they power real products:

| Company | Application |
|---------|-------------|
| Amazon | Recommendation systems, Fraud detection |
| Uber Eats | Food recommendations (graph of users ↔ restaurants) |
| Alibaba | Product recommendations |
| LinkedIn | Social recommendations |
| Pinterest | Item tagging (PinSAGE model) |
| Google Maps | Traffic prediction |
| Pharma | Drug discovery algorithms |

GNNs are the **6th most common keyword at ICLR 2023** — one of the top ML conferences.

---

## 2. The Problem with Traditional ML on Graphs

### 2.1 What Traditional ML Expects

Standard ML models (CNNs, RNNs, DNNs) are designed for **structured, regular data**:
- **CNNs**: Require grid-structured input (images are H×W×C tensors)
- **RNNs**: Require sequential input (text is a sequence of tokens)
- **DNNs**: Require fixed-size rectangular arrays

### 2.2 Why Graphs Are Fundamentally Different

Graphs have two properties that break traditional ML:

1. **Arbitrary size**: A graph can have any number of nodes and edges — you can't just flatten it into a fixed-size tensor.

2. **Complex topological structure (no spatial locality)**: Unlike an image where nearby pixels are always spatial neighbors, in a graph the "neighborhood" of a node depends on the edge structure, which is completely arbitrary.

### 2.3 The Naive Approach: Adjacency Matrix + DNN

A seemingly obvious approach: represent a graph as its **adjacency matrix A** (where A[i][j] = 1 if edge exists between nodes i and j) and feed it to a DNN.

**Why this doesn't work well:**

The adjacency matrix depends on the **ordering of nodes**. If you number nodes differently:
- Node 1 ↔ Node 2 connection appears at position (1,2) in one ordering
- But at position (3,5) in another ordering

**DNNs are not permutation invariant** — changing the node ordering produces a completely different input tensor, so the model would need to relearn the same graph structure from scratch for each different ordering. This is fundamentally wrong.

> **What we need**: A model that produces the *same output* regardless of how we number the nodes. This is called **permutation invariance** — a core property of GNNs.

---

## 3. Graph Neural Networks: Core Concepts

### 3.1 The Key Idea: Encoder-Decoder Framework

Graph learning methods follow an **Encoder-Decoder** approach:

```
Raw Graph (nodes + edges)
         │
         ▼
    ENCODER
    ─────────────────────────────────────────
    Map each node → low-dimensional vector
    (called an "embedding" or "representation")

    Constraint: Similar nodes in the graph
    should have similar embeddings!
    (preserve graph topology in embedding space)
    ─────────────────────────────────────────
         │
         ▼
    Node Embeddings (dense vectors)
         │
         ▼
    DECODER
    ─────────────────────────────────────────
    Measure similarity in embedding space
    (e.g., dot product, cosine similarity)
    ─────────────────────────────────────────
```

The embeddings capture both:
- **Structural information**: Where a node sits in the graph topology
- **Feature information**: The combined features of the node and its neighbors

### 3.2 Message Passing: The Heart of GNNs

**Message Passing** is the fundamental operation in GNNs. It's a generalization of convolutions to non-Euclidean (graph) data.

**The Core Idea**: A node's representation is updated by *aggregating information from its neighbors*.

Think of it like this: In a social network, what defines you? Partly your own attributes, but also the people around you. A node's embedding is informed by its neighborhood.

**The GNN Layer Update Equation:**

$$h_v^{(k+1)} = \text{UPDATE}\!\left(h_v^{(k)},\ \text{AGGREGATE}\!\left(\{h_u^{(k)} : u \in \mathcal{N}(v)\}\right)\right)$$

Where:
- $h_v^{(k)}$ = embedding of node $v$ at layer $k$
- $\mathcal{N}(v)$ = set of neighbors of node $v$
- **AGGREGATE**: Combines embeddings from all neighbors (e.g., mean, sum, max)
- **UPDATE**: Combines the aggregated neighborhood info with the node's own embedding (an MLP)
- Both AGGREGATE and UPDATE are **differentiable** functions (learned via backprop)

**Visualizing a 2-layer GNN:**

```
Layer 0 (input):    Each node has its raw features h_v^(0) = x_v

Layer 1:            Node A gathers info from its direct neighbors (1-hop)
                    h_A^(1) = UPDATE(h_A^(0), AGGREGATE({h_B^(0), h_C^(0), ...}))

Layer 2:            Node A gathers info from 2-hop neighbors
                    h_A^(2) = UPDATE(h_A^(1), AGGREGATE({h_B^(1), h_C^(1), ...}))
                    (but h_B^(1) already contains B's neighbors' info!)
```

> **Deeper = More Context**: A $k$-layer GNN allows each node to "see" up to $k$ hops away in the graph. This is analogous to increasing the receptive field in CNNs.

**Why this is permutation invariant**: AGGREGATE operations like mean/sum/max don't care about the order of neighbors — you get the same result regardless of node numbering. ✓

### 3.3 Full Batch Training

The basic GNN training formulation uses the entire graph:

**Data Structures:**
- **Adjacency Matrix A** (N×N): Encodes graph structure
- **Feature Matrix X** (N×d): Each row is the feature vector of a node

**Layer Computation (full batch):**
$$H^{(L+1)} = A \cdot H^{(L)} \cdot W^{(L)}, \quad H^{(0)} = X$$

Where $W^{(L)}$ is the weight matrix at layer $L$ (what gets trained).

**Problem**: For large graphs (millions of nodes), matrices A and X don't fit in GPU memory → need mini-batch training.

---

## 4. Types of GNN Tasks

GNNs can be applied at three granularities:

### 4.1 Node-Level Tasks
- Concerned with properties of **individual nodes**
- Examples:
  - **Node classification**: Is this web page spam or not? Is this user a bot?
  - **Node clustering**: Group similar users together
- Real use: Classifying documents, videos, web pages into categories

### 4.2 Edge-Level Tasks
- Concerned with **relationships between nodes**
- Examples:
  - **Link prediction**: Will user A follow user B? Should we recommend product P to user U?
  - **Missing link prediction**: Find hidden relationships in knowledge graphs
- Real use: Recommendation systems in social networks

### 4.3 Graph-Level Tasks
- Concerned with properties of the **entire graph**
- Examples:
  - **Graph classification**: How might a molecule smell? Is this molecule toxic?
  - **Graph regression**: How many rings are present in this molecule?
- Real use: Drug discovery, molecular property prediction

---

## 5. Types of GNN Training Settings

| Setting | Labeled Data | Unlabeled Data | Graph Structure |
|---------|-------------|----------------|-----------------|
| **Supervised** | All nodes labeled | — | Available |
| **Semi-Supervised** | Small subset labeled | Most nodes unlabeled | Available for all |
| **Unsupervised** | None | All nodes | Available |

**Semi-supervised** is very common in practice — labeling every node in a large graph is expensive, but the graph structure itself provides useful signal for learning.

---

## 6. Mini-Batch Training and the Neighborhood Explosion Problem

### 6.1 Why Mini-Batches Are Needed

Real-world graphs have millions of nodes — you can't load the full adjacency matrix and feature matrix into GPU memory. Solution: train on **subgraphs** (mini-batches).

**How it works**: Select a small batch of target nodes. Build the "computation graph" — the subgraph containing those nodes and all nodes they need for their GNN computation.

### 6.2 The Neighborhood Explosion Problem

**This is a critical challenge specific to GNNs** (unlike regular DNN training):

```
Layer 0 (target):   1 node
Layer 1:            ~10 neighbors (1-hop)
Layer 2:            ~100 neighbors (2-hop)
Layer 3:            ~1000 neighbors (3-hop) ← this is huge!
```

Each hop multiplies the number of nodes needed. With a 3-layer GNN and average degree 10, a single target node pulls in ~1000 nodes just for its computation!

This is called **neighborhood explosion** — the computation graph grows exponentially with depth.

### 6.3 Neighborhood Sampling

**Solution**: Instead of using *all* neighbors at each hop, randomly sample a *fixed number* of neighbors.

```
Without sampling (degree 10, 3 layers):   1 → 10 → 100 → 1000 nodes
With sampling (sample 5 per hop):         1 → 5  → 25  → 125  nodes
```

This dramatically reduces memory usage while still capturing neighborhood structure.

> **Trade-off**: Sampling introduces randomness/noise into training, but in practice this has regularization benefits and helps generalization.

---

## 7. Challenges in Large-Scale GNN Training

| Challenge | Description |
|-----------|-------------|
| **Scale** | Real-world graphs are too large to fit on a single machine |
| **Dynamic graphs** | Social/traffic networks change over time |
| **Heterogeneous graphs** | Different types of nodes and edges (e.g., user nodes + product nodes) |
| **Model depth** | Deeper GNNs → more context, but more neighborhood explosion |
| **Over-smoothing** | Very deep GNNs make all node embeddings converge to same value |
| **Memory growth** | Memory requirements increase with depth due to intermediate activations |
| **Communication overhead** | Distributed training requires moving node features across workers |

---

## 8. Large-Scale Distributed GNN Training

Like DNN training, large-scale GNN training uses both **Data Parallelism** and **Model Parallelism**.

### 8.1 Key Difference from DNN Data Parallelism

In standard DNN training:
- Each worker gets an **independent** mini-batch of data
- Workers train independently, then sync gradients
- Data on different workers has **no dependencies**

In GNN training:
- Nodes are **interconnected** — a node's computation depends on its neighbors
- If the graph is partitioned across workers, some nodes' neighbors live on **other workers**
- → Cross-worker communication is required during *every* forward pass (not just gradient sync)

This makes distributed GNN training fundamentally harder than distributed DNN training.

### 8.2 Data Parallelism for GNNs

There are two forms:

#### (a) Graph Partition Parallelism
Used when a **single large graph** doesn't fit on one machine.

```
Large Graph
│
▼ Partition
┌──────────────┬──────────────┐
│  Worker 1    │  Worker 2    │
│  Subgraph 1  │  Subgraph 2  │
└──────────────┴──────────────┘
        ↕ (cross-partition edges → communication)
```

**Goals of partitioning:**
1. **Minimize edge cuts**: Fewer edges crossing partition boundaries = less cross-worker communication
2. **Balance load**: Roughly equal number of nodes/training nodes per worker

#### (b) Mini-Batch Parallelism
Used for training where the **input is a collection of graphs** (e.g., molecular property prediction):
- Each mini-batch is a set of independent graphs → very similar to standard DNN mini-batch training
- Workers process different mini-batches independently, sync gradients afterward

When training a single large graph:
- Build computation subgraphs from sampled nodes
- Workers may jointly process a single batch (processing different parts)
- Can sync gradients synchronously or asynchronously

**Most real-world systems combine both**: Graph partition + mini-batch parallelism.

### 8.3 Model Parallelism for GNNs

**Key difference from DNN model parallelism**: GNN models are typically **small** (few layers, modest parameter count) — they can usually fit on a single machine's memory.

However, model parallelism is still used to:
- **Increase throughput** (process more nodes simultaneously)
- **Full-batch training acceleration** (split the large computation across devices)

Note: Model parallelism for GNNs is more advanced and out of scope for this course.

---

## 9. Frameworks for GNN Training

| Framework | Distributed Training | Sampling | Backend | Notes |
|-----------|---------------------|----------|---------|-------|
| **PyTorch Geometric (PyG)** | ✗ Not out-of-the-box | Limited | PyTorch only | Best for research/small graphs |
| **DGL (Deep Graph Library)** | ✓ Built for it | Multiple samplers | PyTorch, TF, MXNet | Production-grade distributed GNN |

**DGL** is the system studied in this course. Key features:
- Purpose-built for distributed GNN training
- Great sampling support (multiple samplers for neighborhood sampling)
- Provides interface with `networkx` (popular graph library)
- Backend agnostic — works with PyTorch, TensorFlow, and MXNet

---

## 10. Case Study: DistDGL

**DistDGL** is the distributed GNN training system built on top of DGL.
- Paper: https://arxiv.org/pdf/2010.05337
- Combines **Graph Partition Parallelism** + **Mini-Batch Parallelism**

### 10.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    DistDGL System                        │
│                                                          │
│  Step 1: Graph Partitioning (done once before training)  │
│  ┌──────────────────────────────────────────────────┐    │
│  │  METIS Partitioner                               │    │
│  │  - Minimize edge cuts                            │    │
│  │  - Balance training nodes across partitions      │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Step 2: Distributed Training Loop (repeated)            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │  Worker 1  │  │  Worker 2  │  │  Worker N  │         │
│  │ Partition 1│  │ Partition 2│  │ Partition N│         │
│  │            │  │            │  │            │         │
│  │  Sampling  │  │  Sampling  │  │  Sampling  │         │
│  │  Fetching  │  │  Fetching  │  │  Fetching  │         │
│  │  Forward   │  │  Forward   │  │  Forward   │         │
│  │  Backward  │  │  Backward  │  │  Backward  │         │
│  └────────────┘  └────────────┘  └────────────┘         │
│       │               │               │                  │
│       └───────────────┴───────────────┘                  │
│                Ring All-Reduce (gradient sync)            │
└─────────────────────────────────────────────────────────┘
```

### 10.2 Step 1: Graph Partitioning (Pre-processing)

Before training starts, the graph is partitioned **once** using the **METIS partitioner**.

**METIS goals:**
1. **Minimize edge cuts**: Edge cuts = edges that cross partition boundaries. Each cut edge requires cross-worker communication during training → fewer cuts = less communication.
2. **Balance training nodes**: Each partition has roughly equal number of training nodes → prevents some workers from being idle while others are still computing.

**Halo Vertices**: When edges are cut, the nodes connected by cut edges are called **halo vertices** (also known as "boundary nodes" or "ghost nodes").

```
Partition 1:          Partition 2:
  A ─── B ─ ─ ─ ─ ─ ─ C ─── D
              (cut edge)

  Worker 1 owns A, B   Worker 2 owns C, D
  B's neighbor C is in Worker 2 → B needs C's features during GNN computation
```

**Important optimization**: Each worker stores the **features of its own nodes only**, NOT the features of halo nodes. Features of halo nodes are fetched on-demand during training (during the Fetching phase).

**Why not replicate halo node features?** In dynamic graphs or large-feature graphs, replicating would use too much memory and require synchronization.

### 10.3 The 4 Phases of DistDGL Training

Every training iteration has 4 sequential phases:

#### Phase 1: Sampling

**Goal**: Build the computation graph for a batch of training nodes.

```
Worker 1's training batch: [Node A, Node B, Node C]
│
▼ Neighborhood Sampling
For each training node, sample its k-hop neighborhood:
- If neighbor is LOCAL (on this worker) → sample locally
- If neighbor is REMOTE (on another worker) → make remote call to that worker
                                               to get sampled subgraph
│
▼ Merge all local + remote subgraphs
→ Final computation graph for this batch
```

This phase requires **cross-worker communication** for nodes near partition boundaries.

#### Phase 2: Fetching

**Goal**: Gather the actual feature vectors for all nodes in the computation graph.

```
Computation graph built in Phase 1 contains:
- Local nodes → features already available locally
- Remote nodes (halo vertices) → must fetch from owning worker
                                  via remote procedure call (RPC)
│
▼ After all fetches complete
Worker has all features needed for forward pass
```

This is a second round of **cross-worker communication** — fetching features is often the bottleneck in distributed GNN training because feature vectors can be large (high-dimensional).

#### Phase 3: Forward Pass

**Goal**: Compute GNN layer outputs and the loss.

```
All features present locally (fetched in Phase 2)
│
▼ GNN Layer 1: AGGREGATE neighbors → UPDATE node embeddings
▼ GNN Layer 2: AGGREGATE neighbors → UPDATE node embeddings
...
▼ Final embeddings → Prediction head (MLP)
▼ Compute loss (cross-entropy for classification, MSE for regression)
│
▼ Begin backward pass
```

No cross-worker communication in this phase! All required data is already local.

#### Phase 4: Backward Pass

**Goal**: Compute gradients and synchronize across all workers.

```
Each worker computes gradients from its local loss
│
▼ Ring All-Reduce
All workers collectively compute the SUM of gradients
(bandwidth-optimal, O(1) peak bandwidth)
│
▼ Gradient update: W ← W - lr × avg_gradient
All workers now have identical updated model weights
```

> **Note**: Only model parameters (W matrices) need gradient sync — node features don't have gradients (they're inputs, not parameters).

### 10.4 Ring All-Reduce in DistDGL

DistDGL uses **Ring All-Reduce** for gradient aggregation (same as we saw in distributed DNN training).

**How it works** (example with 3 workers A, B, C, each with 3 gradient chunks):

```
Initial state:
Worker A: [5, 13] [8, 19] [42, 1]
Worker B: [9, 27] [3, 15] [8,  4]
Worker C: [8, 11] [4,  2] [7,  7]

After Ring All-Reduce:
All workers have: [Σ=22, 51] [Σ=15, 36] [Σ=50, 12]
```

Each worker communicates with its two ring neighbors $(N-1)$ times total.

**Why Ring All-Reduce?**
- **Bandwidth optimal**: Each worker sends/receives a total of $(N-1)/N \times \text{data size}$ — the network is used at full capacity
- **No bottleneck**: Unlike Parameter Server where the PS is a bottleneck, Ring All-Reduce distributes load equally
- See L3.2 notes for full analysis

---

## 11. Summary: Full DistDGL Training Flow

```
 Pre-training:
 ┌─────────────────────────────────────────────────────┐
 │ Graph → METIS Partitioner → N partitions             │
 │ Each worker loads its partition (own nodes + structure)│
 │ All workers initialize same model W                  │
 └─────────────────────────────────────────────────────┘
                     │
                     ▼ (repeat for each epoch/batch)
 ┌─────────────────────────────────────────────────────┐
 │ PHASE 1: SAMPLING                                    │
 │ - Select batch of training nodes                    │
 │ - Sample k-hop neighborhood (local + remote calls)  │
 │ - Build computation graph                           │
 └─────────────────────────────────────────────────────┘
                     │
                     ▼
 ┌─────────────────────────────────────────────────────┐
 │ PHASE 2: FETCHING                                    │
 │ - Fetch features for all nodes in computation graph │
 │ - Local nodes: read from local storage              │
 │ - Remote nodes: RPC call to owner worker            │
 └─────────────────────────────────────────────────────┘
                     │
                     ▼
 ┌─────────────────────────────────────────────────────┐
 │ PHASE 3: FORWARD PASS                                │
 │ - GNN layers: AGGREGATE → UPDATE                    │
 │ - Compute predictions and loss                      │
 │ [No cross-worker communication]                     │
 └─────────────────────────────────────────────────────┘
                     │
                     ▼
 ┌─────────────────────────────────────────────────────┐
 │ PHASE 4: BACKWARD PASS                               │
 │ - Compute local gradients                           │
 │ - Ring All-Reduce: sync gradients across workers    │
 │ - Update model weights W                            │
 └─────────────────────────────────────────────────────┘
```

---

## 12. GNN vs DNN: Key Comparison

| Aspect | DNN Training | GNN Training |
|--------|--------------|--------------|
| **Data structure** | Independent samples (images, text) | Interconnected graph nodes |
| **Mini-batch independence** | ✓ Fully independent | ✗ Cross-batch dependencies (neighbors) |
| **Model size** | Can be very large (billions of params) | Typically small (fits on one GPU) |
| **Communication** | Only gradient sync | Gradient sync + feature fetch + sampling |
| **Data parallelism** | Partition dataset | Partition graph (minimize cuts) |
| **Key challenge** | Parameter sync bandwidth | Neighborhood explosion + cross-partition fetch |
| **Framework** | PyTorch, TF, JAX | DGL, PyG |

---

## 13. Summary: Key Takeaways

1. **Graphs generalize all data structures** — images, sequences, molecules, social networks are all graphs. GNNs are needed because traditional ML models assume grid/sequence structure.

2. **Message Passing is the core GNN operation**: Each node gathers (aggregates) information from its neighbors, then updates its embedding. Repeating for $k$ layers gives $k$-hop context.

3. **Permutation invariance**: GNNs produce the same output regardless of node ordering — unlike DNNs applied to adjacency matrices.

4. **Neighborhood explosion** is the key challenge unique to GNNs: a $k$-layer GNN with degree $d$ needs up to $d^k$ nodes per training sample. **Neighborhood sampling** is the standard solution.

5. **Distributed GNN training ≠ Distributed DNN training**: Graph nodes are *dependent* across partitions, requiring cross-worker communication not just for gradients but also for sampling and feature fetching.

6. **DistDGL** combines Graph Partition Parallelism (METIS) + Mini-Batch Parallelism. Training has 4 phases: Sampling → Fetching → Forward Pass → Backward Pass.

7. **METIS partitioner** minimizes edge cuts (reduce communication) and balances training nodes (load balance) — done once before training begins.

8. **Halo vertices**: Nodes adjacent to cross-partition cut edges. Their features are NOT pre-replicated; they are fetched on-demand during training.

9. **Ring All-Reduce**: Used in the backward pass for bandwidth-optimal gradient aggregation — same as in DNN distributed training.

10. **GNN model sizes are small** (fit on one machine), unlike very large DNNs — model parallelism is less critical; data/graph parallelism is the main scaling strategy.

---

## References

- DistDGL: Zheng et al., "DistDGL: Distributed Graph Neural Network Training for Billion-Scale Graphs", 2020. https://arxiv.org/pdf/2010.05337
- Besta et al., "Parallel and Distributed Graph Neural Networks: An In-Depth Concurrency Analysis", 2022. https://arxiv.org/pdf/2205.09702
- Hamilton, W.L., "Graph Representation Learning", 2020.
- DGL Documentation: https://docs.dgl.ai
- GNN Introduction (visual): https://distill.pub/2021/gnn-intro/


# Lectures 3.5 & 3.6: Transformer Attention + ORCA Serving System

## DS256 - Scalable Systems for Data Science
### Module 3: Machine Learning at Scale

> **References**:
> - vLLM: An Efficient Inference Engine for Large Language Model, Kwon, Woosuk (UC Berkeley)
> - Yu et al., "ORCA: A Distributed Serving System for Transformer-Based Generative Models", OSDI 2022

---

## Part 1: A Primer on Transformer Models & Attention (L3.5)

---

## 1. Transformer Architecture Overview

The **Transformer** is the foundational architecture behind modern large language models (GPT, BERT, T5, etc.). It processes sequences of tokens (words, sub-words) using **stacked layers of attention and feed-forward networks**.

```
Input tokens (e.g., "I think this")
         │
         ▼
  [Embedding Layer]
         │
         ▼
  [Transformer Layer 1]  ─── Attention + MLP
         │
  [Transformer Layer 2]  ─── Attention + MLP
         │
       ... (N layers)
         │
  [Output Head]
         │
         ▼
  Output tokens (e.g., "is great <EOS>")
```

**Key components of a single Transformer layer** (GPT-style):
1. **LayerNorm** — normalize input for stability
2. **QKV Linear** — project input into Query, Key, Value vectors
3. **Attention** — compute weighted relationships between tokens
4. **Attn Out Linear** — project attention output back to model dimension
5. **Add** (residual connection)
6. **LayerNorm**
7. **MLP (FFN)** — GeLU → Linear non-linearity
8. **Add** (residual connection)

---

## 2. Attention: The Heart of the Transformer

### 2.1 Intuition: What Does Attention Do?

Attention allows each token in a sequence to **look at all other tokens** and decide which ones are relevant for understanding its meaning.

**Classic example**: "it is closely related to *cat*" (first sentence) vs. "it refers to *milk*" (second sentence). The same word "it" must attend to different words depending on context.

Attention **relates words in the input to each other** and helps the model pay attention to the most relevant prior words for prediction.

### 2.2 Multi-Headed Attention

Multiple **attention heads** can focus on different types of relationships simultaneously:
- One head might focus on syntactic relations (subject → verb)
- Another on semantic relations (pronoun → referent)
- Another on positional relations

This gives a **richer understanding of context** than a single attention computation.

### 2.3 Self-Attention: Q, K, V

Self-attention takes three inputs derived from the same sequence:

| Component | Role | Analogy |
|-----------|------|---------|
| **Query (Q)** | The current token "asking" for context | A search query |
| **Key (K)** | All tokens being searched over | Search index labels |
| **Value (V)** | The actual content of each token | Search result content |

**How it works**: Each token projects itself to Q, K, V spaces using learned weight matrices:
$$Q = X W_Q, \quad K = X W_K, \quad V = X W_V$$

### 2.4 Self-Attention Computation (Step by Step)

$$\text{Attention}(Q, K, V) = \text{softmax}\!\left(\frac{Q K^\top}{\sqrt{d_k}}\right) V$$

**Step-by-step:**

```
1. Convert word to embedding → project to Q, K, V spaces (using trained weights)

2. Compute match scores: Q · Kᵀ
   (How well does this query token match each key token?)

3. Scale: divide by √d_k
   (Prevent dot products of large vectors from creating extreme softmax gradients)

4. Softmax → attention weights
   (Convert raw scores into probability distribution summing to 1)

5. Weighted average of Values: weights × V
   (Select and blend the most relevant content tokens)
   → Final output embedding for this token
```

> **Intuition**: If you're computing the embedding for the word "ball" in "the blue ball", the attention scores will be high for "blue" and "holding" but low for "boy" — the output is a context-aware weighted blend of all token values.

### 2.5 Autoregressive Generation

For generative models (GPT), the model generates output **one token at a time**:

1. **Initiation phase** (Prefill): Process all input tokens at once in parallel → generate first output token
2. **Increment phase** (Decode): Feed the last generated token → generate next token, repeat until `<EOS>`

```
Input: "I think this"
   ↓ Initiation phase (all 3 tokens processed in parallel)
Token 4: "is"
   ↓ Increment phase (one token at a time)
Token 5: "great"
   ↓
Token 6: "<EOS>" (stop)
```

Each full run through all N layers of the model = **one iteration**.

**Key property**: Attention at position $t$ requires keys and values from **all previous positions** $1, 2, \ldots, t-1$.

---

## 3. Computational Characteristics

### 3.1 Arithmetic Intensity (AI) and the Roofline Model

**Arithmetic Intensity (AI)** = FLOPs performed / bytes transferred from memory

- **High AI → Compute Bound**: The bottleneck is the number of floating-point operations
- **Low AI → Memory Bandwidth Bound**: The bottleneck is reading/writing data to/from memory

**GPU hardware skews toward compute speed** over memory bandwidth:
- NVIDIA H100: ~1,000 TFLOPS of BF16 compute, but only 3.35 TB/s memory bandwidth
- **Roofline threshold**: AI > 330 FLOPs/byte → compute bound on H100; else **memory bound**

> **Implication for LLM inference**: The decode phase (one token at a time) is often memory-bound because the computation is small but requires loading large model weights from memory.

### 3.2 Number Format Precision

| Format | Size | Exponent | Mantissa | Notes |
|--------|------|----------|----------|-------|
| **FP32** (IEEE) | 4 bytes | 8 bits | 23 bits | Standard, highest precision |
| **BF16** (Brain Float, Google) | 2 bytes | 8 bits | 7 bits | Mixed precision; retains FP32's exponent range |
| **TF32** (Nvidia Tensor Core) | ~2.4 bytes | 8 bits | 10 bits | Used internally in Tensor Cores |

**BF16**: Half the memory of FP32, same exponent range → can represent same range of values, just less precisely. Used extensively in LLM training and inference.

### 3.3 Prefill vs. Decode Phases

| Phase | What happens | Compute profile | Complexity |
|-------|-------------|-----------------|-----------|
| **Prefill (Initiation)** | Process all $n$ input tokens at once | Compute-intensive | $O(n^2 \cdot d)$ — matrix $QK^\top$ is $n \times n$ |
| **Decode (Increment)** | Process one token at a time | Memory-intensive | $O(t)$ per token $t$ (with KV cache) |

**Why decode is memory-bound**: Each new token requires reading the entire KV cache from memory, but performs relatively little computation (just one token's attention).

**Problem with decode batching**: Even with multiple requests batched together, the per-token computation in decode is too small to saturate GPU compute cores. You'd need thousands of requests batching together — typically infeasible.

---

## 4. Key-Value (KV) Cache

### 4.1 Why KV Cache Exists

During autoregressive generation, each new token's Attention operation needs the keys and values of **all previous tokens**. Recomputing them from scratch at every step is wasteful.

**KV Cache**: Store and **reuse** the key and value tensors from all previous tokens → avoid redundant computation.

### 4.2 KV Cache Memory Cost

Memory per token in the KV cache:
$$\text{bytes} = 2 \times L \times H \times d \times \text{dtype\_size}$$

Where:
- $L$ = number of layers
- $H$ = number of attention heads
- $d$ = head dimension
- Factor 2 = one for key, one for value

**Example**: 100 layers, 8 key-value heads, 128 head dimension:
- Per token: $2 \times 100 \times 8 \times 128 \times 2\text{ bytes (BF16)} = 400\text{ KB}$
- Request with 100,000 tokens: **40 GB KV cache** — more than most GPUs' VRAM!

### 4.3 KV Cache Challenges

1. **Memory bandwidth**: Accessing the growing KV cache for each new token consumes significant memory bandwidth
2. **Memory fragmentation**: KV cache grows dynamically with the number of tokens, and dynamic allocation causes fragmentation
3. **Trade-off**: Larger batches → better GPU utilization AND higher KV cache memory usage → tension between throughput and memory

---

## 5. LLM Inference Performance Metrics

| Metric | Definition | Bottleneck | Interactive target |
|--------|-----------|-----------|-------------------|
| **TTFT** (Time-to-First-Token) | Time from request submission until first output token | Prefill phase (compute-bound) | ~2 seconds |
| **TPOT** (Time-per-Output-Token) | Time between consecutive tokens in decode phase | Decode phase (memory-bound) | ~200 ms |

**TTFT is affected by**:
- Prompt length (more compute)
- Queuing delays (earlier requests in system)
- Prefill batching with ongoing decode phases

**TPOT is affected by**:
- Batch size (more compute per step)
- Sequence length seen so far (larger KV cache to access)

---

## Part 2: ORCA — A Distributed Serving System for Transformer-Based Generative Models (L3.6)

> **Paper**: Yu et al., OSDI 2022

---

## 6. Problem: Existing LLM Serving Systems

### 6.1 How Current Systems Work (e.g., Triton + FasterTransformer)

```
Client requests
     │
     ▼
  [Endpoint]
     │
  [Scheduler] ──── creates batch of requests from queue
     │
     ▼
  [Execution Engine] ──── processes entire batch over multiple iterations
     │
     ▼
  Returns generated text to all clients (only when ALL requests in batch finish)
```

The scheduler operates at **request granularity**: once a batch is dispatched to the engine, the engine runs until *all* requests in the batch complete.

### 6.2 Challenge 1: Early-Finished and Late-Joining Requests

**The problem**: Different requests need different numbers of tokens. In a fixed batch:

```
Request x₁: "I think" → needs to generate: "this is great" (3 tokens, 3 iterations)
Request x₂: "I love"  → needs to generate: "you" (1 token, 1 iteration)

Iteration 1: x₁ generates "this", x₂ generates "you" → x₂ DONE!
Iteration 2: x₁ generates "is",   x₂ is IDLE (but engine still runs it!)
Iteration 3: x₁ generates "great", x₂ is IDLE
             ↑ x₂ must wait, and we waste compute on it
```

**Two consequences**:
1. **Early-finished requests** cannot return to clients immediately — they must wait for the entire batch to finish → increased latency
2. **Late-arriving requests** must wait for the entire current batch to finish before being admitted → increased queuing delay

### 6.3 Challenge 2: Batching Arbitrary Requests

Batching is critical for GPU efficiency (reuses model weights across requests). But Transformer layers expect input tensors of shape $[B, L, H]$ where $B$ = batch size, $L$ = sequence length (same for all requests), $H$ = hidden size.

**Three cases where two requests CANNOT be batched together:**

| Case | Why | Example |
|------|-----|---------|
| **Both in initiation phase, different input lengths** | Tensor length $L$ differs | $x_3$ has 2 tokens, $x_4$ has 3 tokens |
| **Both in increment phase, at different token indices** | Attention KV tensor shape differs | $x_1$ at position 3, $x_2$ at position 1 |
| **One in initiation, one in increment** | Initiation processes all tokens at once; increment processes one | $x_1$ (increment) vs $x_3$ (initiation) |

This severely limits batching opportunities in practice, especially as requests arrive asynchronously.

---

## 7. ORCA's Solutions

### 7.1 Solution 1: Iteration-Level Scheduling

**Core idea**: Schedule execution at the **granularity of one iteration** (one forward pass through all layers), not the full request lifetime.

**The scheduling loop:**
```
LOOP:
  1. SELECT: Choose a set of requests from the request pool
  2. INVOKE: Run engine for exactly ONE iteration on the selected requests
  3. RECEIVE: Get output tokens for this iteration
  4. CHECK: Did any requests finish (hit <EOS>)?
     → YES: Return their output immediately to clients, remove from pool
     → NO:  Append generated token to request, return to pool
  GOTO LOOP
```

**Benefits**:
- **Immediate Return**: Finished requests returned to clients after every iteration check — no waiting
- **Fast Admission**: Newly arrived requests can be selected at the very next iteration — minimal queuing delay
- **Full flexibility**: Scheduler has complete control over which requests to include in each iteration

> **Analogy**: Request-level scheduling is like a bus that only departs when all passengers have finished their trips. Iteration-level scheduling is like a bus that makes frequent stops, letting passengers on and off at every stop.

### 7.2 Solution 2: Selective Batching

**The insight**: Not all operations in a Transformer layer have the same batching constraints. Operations can be split into two groups:

**Group 1 — Non-Attention operations** (Linear, LayerNorm, MLP, Add, GeLU):
- Do NOT need to distinguish which tokens belong to which request
- Can be applied token-wise on a **flattened** batch tensor of shape $[\sum L, H]$
- **Benefit from batching**: reuse model parameters loaded from GPU memory

**Group 2 — Attention operation**:
- Needs to know which tokens belong together (same request) to apply causal masking
- Typically requires the batch dimension $[B, L, H]$ with fixed $L$
- **Does NOT benefit from model parameter reuse** (Attention has no learnable parameters beyond QKV projections)
- Does NOT have locality in KV cache

**Selective Batching Strategy**:
```
For a batch of requests (x₁, x₂, x₃, x₄) with total 7 tokens:

[7, H] tensor (all tokens concatenated)
    │
    ▼
Non-Attention ops applied on full [7, H] tensor (token-wise batching)
    │
    ▼ Split operation
┌──────┬──────┬──────┬──────┐
│ x₁   │ x₂   │ x₃   │ x₄   │   (split by request)
│[1,H] │[1,H] │[2,H] │[3,H] │
└──────┴──────┴──────┴──────┘
    │ Each request processed individually through Attention
    ▼
Attention K/V Manager: maintains KV cache per request
    │
    ▼ Merge operation
[7, H] tensor (rejoin)
    │
    ▼
Continue with non-Attention ops (batched again)
```

**The Attention K/V Manager**: Maintains K and V tensors separately for each active request. For requests in the increment phase, it provides the stored K/V from all previous tokens along with the current token's Q, K, V.

---

## 8. ORCA System Architecture

### 8.1 Components

```
Clients
   │  requests / responses
   ▼
[Endpoint] (HTTPS or gRPC)
   │
   ▼
[Request Pool]  ←──────────────────────┐
   │  (stores all active requests)     │
   ▼                                   │
[Scheduler]                            │ (finished: remove)
   │  selects requests per iteration   │ (unfinished: re-add with new token)
   ▼                                   │
[Execution Engine (Engine Master)]─────┘
   │
   ├── [Worker 1: inter-layer partition]
   │       └── [GPU + Controller thread]
   │
   └── [Worker 2: inter-layer partition]
           └── [GPU + Controller thread]
```

**Request Pool**: A pool (not a simple queue) that holds all active requests. Each request stores its accumulated tokens (both input and all generated tokens so far).

**Scheduler**: Selects at most `max_bs` requests per iteration from the pool using FCFS order. Manages KV memory reservation.

**Engine Master**: Receives the scheduled batch from the scheduler, sends token information to the first worker process to begin the pipeline.

**Worker Process**: One per inter-layer partition. Each worker manages one or more CPU threads (Controllers), each controlling one GPU.

**Controller**: Parses batch information from the Engine Master, issues GPU kernel calls to its GPU. Also asynchronously forwards the control message to the next Worker's Controller.

### 8.2 Distributed Architecture: Parallelism

ORCA uses both parallelism strategies from DNN training:

**Intra-layer parallelism (Tensor Parallelism)**:
- Splits linear and Attention matrix multiplications **within a single layer** across multiple GPUs
- Based on Megatron-LM / Mesh-TensorFlow approach
- Communication via **NCCL** (GPU-to-GPU, for tensor data)

**Inter-layer parallelism (Pipeline Parallelism)**:
- Splits Transformer layers across GPUs
- E.g., layers 1–48 on Worker 1, layers 49–96 on Worker 2
- Equal number of Transformer layers per GPU (load balancing)
- Communication via **gRPC** (CPU-to-CPU, for control messages and tokens)

**Why separate communication channels?**
- Existing systems (FasterTransformer) use NCCL for *all* communication, including control messages → CPU-GPU sync overhead at every iteration
- ORCA separates: **gRPC for control** (CPU threads, no GPU sync), **NCCL for tensor data** (GPU-to-GPU only for intermediate activations)
- Result: reduced CPU-GPU synchronization overhead

**Example (4-layer GPT model, 6 GPUs):**
```
           Layer1  Layer2  Layer3  Layer4
GPU1  ─── [▓▓▓▓▓] [▓▓▓▓▓] [     ] [     ]  ← Worker 1
GPU2  ─── [▓▓▓▓▓] [▓▓▓▓▓] [     ] [     ]  ← Worker 1
GPU3  ─── [▓▓▓▓▓] [▓▓▓▓▓] [     ] [     ]  ← Worker 1
                                              (intra-layer partition within Worker 1)
GPU4  ─── [     ] [     ] [▓▓▓▓▓] [▓▓▓▓▓]  ← Worker 2
GPU5  ─── [     ] [     ] [▓▓▓▓▓] [▓▓▓▓▓]  ← Worker 2
GPU6  ─── [     ] [     ] [▓▓▓▓▓] [▓▓▓▓▓]  ← Worker 2
          └─ inter-layer split ─┘
```
- Inter-layer: Layers 1-2 → Worker 1, Layers 3-4 → Worker 2
- Intra-layer: Within each worker, 3 GPUs share each layer's matrices

### 8.3 Execution Flow (One Iteration)

```
Scheduler ──①── selects requests (x₁, x₂, x₃, x₄)
              ↓
Engine Master ──②── sends (tokens, request IDs, token indices, token counts)
              ↓                                to Worker 1's Controller
Worker 1 Controller ──③── issues GPU kernels on GPU1/2/3
              │                  (uses KV manager for Attention K/V)
              │   ──④──  forwards control message ASYNC to Worker 2's Controller
              ↓
Worker 2 Controller ──⑤── issues GPU kernels on GPU4/5/6
              │              waits for Worker 1 outputs
              │   ──⑥──  sends output tokens back to Engine Master
              ↓
Engine Master ──⑦── updates request pool
                      finished requests → notify Endpoint → return to clients
                      unfinished requests → append token → remain in pool
```

### 8.4 Scheduling Algorithm

**Algorithm 1 (ORCA Scheduler)**:

```
Parameters: n_workers (pipeline depth), max_bs (max batch size), n_slots (KV memory slots)

n_scheduled = 0   // number of currently running batches
n_rsrv = 0        // number of reserved KV memory slots

LOOP:
  batch, n_rsrv = Select(request_pool, n_rsrv)  // pick next batch
  schedule engine: run 1 iteration on batch

  foreach req in batch: mark req.state = RUNNING
  n_scheduled += 1

  IF n_scheduled == n_workers:  // pipeline is full
    wait for return from engine
    foreach req in returned_batch:
      req.state = INCREMENT
      IF finished(req):
        n_rsrv -= req.max_tokens  // free reserved KV slots
      n_scheduled -= 1

SELECT(pool, n_rsrv):
  pool = {req ∈ pool | req.state ≠ RUNNING}  // only non-running requests
  SortByArrivalTime(pool)  // FCFS ordering
  foreach req in pool:
    if batch.size() == max_bs: break
    if req.state == INITIATION:
      new_n_rsrv = n_rsrv + req.max_tokens
      if new_n_rsrv > n_slots: break  // not enough KV memory
      n_rsrv = new_n_rsrv
    batch = batch ∪ {req}
  return batch, n_rsrv
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| **Iteration-level FCFS** | Earlier-arrived requests processed ≥ as many iterations as later ones; fair ordering |
| **max_bs** | Caps batch size to trade off throughput vs. latency — tunable by operator |
| **n_slots reservation** | When scheduling a new INITIATION-phase request, pre-reserve KV memory for `max_tokens` tokens → prevents OOM deadlock during generation |
| **Pipeline parallelism** | Scheduler doesn't wait after dispatching each batch; keeps `n_workers` batches running concurrently (one per worker) |

**Pipeline Parallelism (ORCA vs FasterTransformer):**

```
ORCA pipeline (3 workers, max_batch=2):
Worker1: [A₁B₁][C₁D₁][E₁F₁][A₂B₂][C₂D₂][E₂F₂]
Worker2:     [A₁B₁][C₁D₁][E₁F₁][A₂B₂][C₂D₂]
Worker3:         [A₁B₁][C₁D₁][E₁F₁][A₂B₂]
(Xᵢ = i-th iteration of request X)

FasterTransformer pipeline (3 partitions, microbatch=A,B):
Partition1: [A₁][B₁]   [A₂][B₂]   [A₃]
Partition2:    [A₁][B₁]   [A₂][B₂]
Partition3:       [A₁][B₁]   [A₂][B₂]
```

ORCA's advantage: Because iteration-level scheduling allows arbitrary request selection at each iteration, there is no need to split batches into microbatches (which FasterTransformer requires). This avoids the **bubble time** problem of pipeline parallelism (see L3.2 notes).

---

## 9. Implementation

- **13,000 lines of C++**, CUDA ecosystem
- **gRPC** for control plane communication (control messages between workers)
- **NCCL** for data plane (inter-layer and intra-layer tensor communication)
- **Fused CUDA kernels**: LayerNorm, Attention, GeLU — fused into fewer kernel launches
  - Attention: dot products Q·K, softmax, weighted sum of V all fused into single kernel
  - Split Attention operators: kernel thread blocks from different requests concatenated → single kernel call for all requests despite processing them separately
- Models supported: original Transformer [Vaswani et al. 2017], GPT, T5 variants

---

## 10. Evaluation

### 10.1 Setup

- **Hardware**: Azure ND96asr A100 VMs — up to 4 VMs, each with 8× NVIDIA A100 40GB GPUs (NVLink), 200Gbps Infiniband between VMs
- **Models**: GPT family — 13B, 101B, 175B, 341B parameters
- **Baseline**: NVIDIA FasterTransformer (state-of-the-art distributed inference engine)

**Model configurations:**

| # Params | # Layers | Hidden size | # Inter-partitions | # Intra-partitions |
|----------|----------|-------------|-------------------|-------------------|
| 13B | 40 | 5120 | 1 | 1 |
| 101B | 80 | 10240 | 1 | 8 |
| 175B | 96 | 12288 | 2 | 8 |
| 341B | 120 | 15360 | 4 | 8 |

### 10.2 Engine Microbenchmark (No Scheduler)

Comparison of ORCA engine vs. FasterTransformer engine in isolation (not testing scheduling):
- Both receive the same homogeneous batch
- **ORCA performs similarly** to FasterTransformer (slightly worse for small models)
- **Why slightly worse?**: ORCA does not batch the Attention operation; FasterTransformer batches everything
- **Why the gap is small**: Attention operation has no model parameters → no benefit from parameter reuse across requests → the difference is minimal
- For 175B model (2 inter-layer partitions): ORCA outperforms FasterTransformer by **up to 47%** due to control-data plane separation advantage

**FasterTransformer limitation**: Fixed memory pre-allocation for KV cache based on `max_tokens` → OOM errors at moderate batch sizes (e.g., batch=16 for 13B model, batch=16 for 101B model).

### 10.3 End-to-End Performance (With Scheduler)

**Workload**: Synthesized trace — input lengths from $U(32, 512)$, generation lengths from $U(1, 128)$.

**Key result**: For the **175B model** at a median normalized latency of 190ms/token:
- FasterTransformer: **0.185 req/s** throughput
- ORCA: **6.81 req/s** throughput
- **= 36.9× throughput improvement** at the same level of latency!

For the **341B model** (32 GPUs): ORCA outperforms FasterTransformer under **every level of load** in both latency and throughput — sometimes by **an order of magnitude**.

**Why such large gains?**: FasterTransformer (with request-level scheduling) wastes compute on:
1. Early-finished requests (idle but still batch-occupying)
2. Late-joined requests (waiting in queue unnecessarily)
3. Fixed microbatch sizes causing pipeline bubbles

All three inefficiencies are eliminated by ORCA's iteration-level scheduling.

**Varying batch size**: Increasing `max_bs` in ORCA improves throughput without increasing latency (because early-finishing resolves the early-finish/late-join problem). There is no such free lunch in FasterTransformer.

---

## 11. Summary: Concept Map

```
TRANSFORMER BACKGROUND
├── Architecture: N stacked [LayerNorm → QKV → Attention → MLP] layers
├── Self-Attention: Q·Kᵀ/√d → softmax → ×V (captures token relationships)
├── Multi-head attention: multiple perspectives simultaneously
├── Autoregressive generation: initiation (prefill, all tokens) + increment (decode, 1 token/iter)
├── Arithmetic Intensity: high AI → compute bound; low AI → memory bound
├── BF16: 2 bytes, 8-bit exponent (same range as FP32), reduced mantissa
├── Prefill: O(n²d), compute-bound; Decode: O(t), memory-bound
├── KV Cache: store K,V for all previous tokens → avoid recomputation
│             cost = 2×L×H×d per token (e.g., 400KB/token for large models)
└── Metrics: TTFT (first token latency, prefill-dominated) + TPOT (decode speed)

ORCA SERVING SYSTEM
├── Problem: Request-level scheduling → early-finish waste + late-join delay
│
├── Challenge 1: Early-finished/late-joining requests
│   └── Solution 1: ITERATION-LEVEL SCHEDULING
│       • Scheduler runs one iteration per scheduling decision
│       • Immediately returns finished requests
│       • Admits new requests every iteration
│       • FCFS at iteration granularity
│
├── Challenge 2: Batching arbitrary requests with incompatible tensor shapes
│   └── Solution 2: SELECTIVE BATCHING
│       • Non-Attention ops: token-wise batching on flattened [ΣL, H] tensor
│       • Attention op: processed per-request (no batching)
│       • Attention K/V Manager: per-request KV cache storage
│
├── Architecture
│   ├── Request Pool → Scheduler → Engine Master → Workers
│   ├── Workers: one per inter-layer partition
│   ├── Intra-layer parallelism (NCCL, tensor ops across GPUs within worker)
│   ├── Inter-layer parallelism (gRPC, pipeline across workers)
│   └── Separated control plane (gRPC) + data plane (NCCL) → reduced sync overhead
│
├── Scheduling Algorithm
│   ├── FCFS iteration-level ordering
│   ├── max_bs: cap batch size (throughput vs. latency knob)
│   ├── n_slots: KV memory reservation for initiation-phase requests
│   └── Pipeline parallelism: n_workers batches running concurrently
│
└── Results: 36.9× throughput improvement vs. FasterTransformer at same latency
```

---

## 12. Key Takeaways

1. **Transformer attention** computes $\text{softmax}(QK^\top / \sqrt{d_k})V$ — allows every token to attend to all others. Scaling by $\sqrt{d_k}$ prevents gradient instability from large dot products.

2. **Multi-headed attention** runs multiple attention functions in parallel, capturing different types of relationships (syntax, semantics, position).

3. **Generative models (GPT)** are autoregressive: **initiation phase** processes all input tokens at once (compute-bound, $O(n^2d)$); **increment phase** generates one token per iteration (memory-bound, $O(t)$).

4. **KV cache** avoids recomputing keys/values for previous tokens but consumes substantial GPU memory (400KB/token for large models → 40GB for 100K-token context).

5. **TTFT** = latency for first token (prefill-dominated); **TPOT** = per-token decode speed. Both critical for interactive applications.

6. **Problem with request-level scheduling**: Fixed batches cause early-finished requests to wait and late-arriving requests to queue unnecessarily — catastrophic for variable-length generation.

7. **Iteration-level scheduling** (ORCA's key contribution): Schedule at the granularity of *one forward pass* → immediate return for finished requests + fast admission for new requests.

8. **Selective batching** (ORCA's second contribution): Apply token-wise batching to non-Attention ops (which benefit from parameter reuse) but process Attention per-request (which cannot be batched across different token positions anyway).

9. **ORCA achieves 36.9× throughput** over FasterTransformer at the same latency level for large models — the gains come almost entirely from eliminating the waste of request-level scheduling.

10. **Separation of control plane (gRPC) and data plane (NCCL)** avoids the CPU-GPU synchronization overhead that plagues systems like FasterTransformer which use NCCL for everything.

---

## 13. Related Work

**BatchMaker** (Gao et al., EuroSys 2018) is the most closely related prior work:
- Serves **RNNs** at the granularity of RNN cells (not full requests) — analogous to ORCA's iteration-level scheduling
- Breaks the dataflow graph into RNN cells and schedules batching at cell granularity
- Can batch identical cells from different requests together

**Why BatchMaker doesn't work for Transformers**:
- Transformer tokens at different indices each require a *different* set of Attention Keys/Values → $L$ different cell types per request (where $L$ = total input + generated tokens)
- This makes cells of the same type extremely rare → most execution is serialized, not batched
- BatchMaker also lacks support for large models requiring model/pipeline parallelism

**ORCA's key principle** (different from BatchMaker): Maximize computation per round of model parameter reads. Reading model weights from GPU global memory is the major bottleneck for large-scale models. ORCA uses iteration-level scheduling and selective batching to process as many "ready" tokens as possible per parameter read — regardless of whether those tokens can be batched (non-Attention) or not (Attention).

**Other inference systems** (FasterTransformer, LightSeq, TurboTransformers, EET):
- All operate as execution engines with request-level scheduling delegated to serving systems like Triton/TF-Serving
- The interface between serving system and execution engine is too restricted for the multi-iteration generative workload
- ORCA tightly integrates scheduler and engine to enable iteration-level control

---

## References

- Vaswani et al., "Attention is All You Need", NeurIPS 2017
- Yu et al., "ORCA: A Distributed Serving System for Transformer-Based Generative Models", OSDI 2022
- Kwon, Woosuk, "vLLM: An Efficient Inference Engine for Large Language Model", UC Berkeley PhD Dissertation
- NVIDIA Developer Blog: "Mastering LLM Techniques: Inference Optimization"
