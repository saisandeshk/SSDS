# Part 1 — Tokenization & Preprocessing
### DS256 A-2 | Team 05 | `saisandeshk` + `juhitharadha`
### Script: `a2p1_v0.1.py` | Deadline: 2026-04-17

---

## Table of Contents
1. [Dataset Analysis](#1-dataset-analysis)
2. [Step 1 — Tokenization](#2-step-1--tokenization)
   - [Implementation](#21-implementation)
   - [Efficiency Analysis](#22-efficiency-analysis--how-to-make-tokenization-fast-with-a-fixed-tokenizer)
   - [EOS Token Handling](#23-eos-token-handling)
   - [What NOT To Do](#24-what-not-to-do)
3. [Step 2 — Preprocessing](#3-step-2--preprocessing)
   - [Why Concatenate and Chunk?](#31-why-concatenate-and-chunk-instead-of-padding)
   - [Implementation](#32-implementation)
   - [Deep Dive: `group_texts`](#33-deep-dive-group_texts)
   - [Efficiency Analysis](#34-efficiency-analysis)
   - [Column Management](#35-column-management-in-map)
   - [What NOT To Do](#36-what-not-to-do)
4. [Pipeline Execution Flow](#4-pipeline-execution-flow)
5. [How to Run in Your Slot](#5-how-to-run-in-your-slot)
6. [Validation & Verification](#6-validation--verification)
7. [Summary](#7-summary)

---

## 1. Dataset Analysis

### Train Dataset (`/mnt/data/ds256_2026/as2/train_dataset`)

| Property | Value |
|---|---|
| Format | Parquet (103 part files + 1 `_SUCCESS` Spark marker) |
| Total size | ~1.01 GB |
| Number of rows | **577,736** |
| Columns | `warc_id`, `url`, `date`, `extracted_text` |
| Text column | `extracted_text` |
| Word length (sample 1000) | min=8, max=9902, mean=438, median=196 |
| Empty texts | 0 (no nulls found) |

> ⚠️ **Gotcha:** The directory also contains a `_SUCCESS` file (0-byte Spark marker from the deduplication pipeline). `load_dataset("parquet", data_dir=...)` crashes on it because it tries to read the schema from a 0-byte file. **Solution:** use `glob.glob("*.parquet")` to explicitly list only the parquet files.

**Commands to explore the train dataset:**
```bash
# Directory overview
ls /mnt/data/ds256_2026/as2/train_dataset/ | head -5
ls /mnt/data/ds256_2026/as2/train_dataset/ | wc -l   # 104: 103 parquets + _SUCCESS

# Total size
du -sh /mnt/data/ds256_2026/as2/train_dataset/

# Inspect schema of one parquet file
/mnt/data/miniconda3/envs/saisandeshk/bin/python -c "
import pyarrow.parquet as pq
f = pq.read_table('/mnt/data/ds256_2026/as2/train_dataset/part-00000-42a96edd-d642-498e-a7e9-22f043032683-c000.snappy.parquet')
print('Schema:', f.schema)
print('Rows in this file:', f.num_rows)
print(f.to_pandas().head(2).to_string())
"

# Row count across all files
/mnt/data/miniconda3/envs/saisandeshk/bin/python -c "
import glob, pyarrow.parquet as pq
files = sorted(glob.glob('/mnt/data/ds256_2026/as2/train_dataset/*.parquet'))
total = sum(pq.read_metadata(f).num_rows for f in files)
print(f'Total rows: {total}')
print(f'Files: {len(files)}')
"
```

---

### Test Dataset (`/mnt/data/ds256_2026/as2/test_dataset`)

| Property | Value |
|---|---|
| Format | HuggingFace Arrow (saved via `save_to_disk`) |
| Files | `data-00000-of-00001.arrow`, `dataset_info.json`, `state.json` |
| Number of rows | **41,401** |
| Columns | `input_ids` (list\<int32\>, len=512), `labels` (list\<int64\>, len=512) |

> ✅ **The test dataset is already fully preprocessed.** It has the final `input_ids + labels` in 512-token blocks. **Do NOT tokenize or preprocess it.** It is used directly in Part 2 for evaluation via `load_from_disk(final_test_dataset)`.

**Commands to inspect the test dataset:**
```bash
/mnt/data/miniconda3/envs/saisandeshk/bin/python -c "
from datasets import load_from_disk
ds = load_from_disk('/mnt/data/ds256_2026/as2/test_dataset')
print('Rows:', len(ds))
print('Columns:', ds.column_names)
print('Features:', ds.features)
print('input_ids length:', len(ds[0]['input_ids']))   # should be 512
print('First 10 tokens:', ds[0]['input_ids'][:10])
print('labels == input_ids:', ds[0]['input_ids'] == ds[0]['labels'])
"
```

---

### GPT-2 Tokenizer

| Property | Value |
|---|---|
| Class | `GPT2TokenizerFast` |
| `is_fast` | **True** (Rust-based `tokenizers` library backend) |
| `eos_token` | `<\|endoftext\|>` |
| `eos_token_id` | **50256** |
| `bos_token_id` | **50256** (same as EOS — by GPT-2 design) |
| `vocab_size` | 50,257 |

**Commands to inspect the tokenizer:**
```bash
/mnt/data/miniconda3/envs/saisandeshk/bin/python -c "
from transformers import AutoTokenizer
tok = AutoTokenizer.from_pretrained('/mnt/data/ds256_2026/as2/gpt2_tokenizer')
print('Class:', type(tok).__name__)
print('is_fast:', tok.is_fast)
print('eos_token:', tok.eos_token, '| id:', tok.eos_token_id)
print('bos_token:', tok.bos_token, '| id:', tok.bos_token_id)
print('vocab_size:', tok.vocab_size)
# Quick test
sample = 'Hello world this is a test.'
ids = tok(sample, add_special_tokens=False)['input_ids']
print('Tokens:', ids, '→ with EOS:', ids + [tok.eos_token_id])
"
```

---

## 2. Step 1 — Tokenization

### 2.1 Implementation

```python
###!@1 START ANSWER STEP 1

def step_1_tokenization():
    ## DO NOT MODIFY ##
    tokenizer = AutoTokenizer.from_pretrained(GPT2_TOKENIZER_DIR)
    print(tokenizer)
    ## DO NOT MODIFY ##

    ## start your edits here  =================

    # Load only .parquet files — skips the 0-byte _SUCCESS marker
    parquet_files = sorted(glob.glob(os.path.join(deduplicated_train_dataset_path, "*.parquet")))
    print(f"Found {len(parquet_files)} parquet files "
          f"({sum(os.path.getsize(f) for f in parquet_files) / 1e9:.2f} GB)")

    train_dataset = load_dataset(
        "parquet",
        data_files={"train": parquet_files},
        split="train",
    )
    print(f"Loaded {len(train_dataset)} training samples | columns: {train_dataset.column_names}")

    def tokenize_fn(examples):
        tokenized = tokenizer(examples["extracted_text"], add_special_tokens=False)
        # Append EOS token (id=50256) to mark document boundaries
        input_ids = [ids + [tokenizer.eos_token_id] for ids in tokenized["input_ids"]]
        return {"input_ids": input_ids}

    num_proc = multiprocessing.cpu_count()
    print(f"Tokenizing with {num_proc} CPU processes (batched, fast tokenizer)...")
    t0 = time.time()
    tokenized_dataset = train_dataset.map(
        tokenize_fn,
        batched=True,
        batch_size=1000,
        num_proc=num_proc,
        remove_columns=["url", "date", "extracted_text"],  # keep warc_id
        desc="Tokenizing",
        writer_batch_size=10000,
    )
    print(f"Tokenization done: {len(tokenized_dataset)} samples in {time.time() - t0:.2f}s")

    tokenized_dataset.save_to_disk(tokenized_train_dataset)
    print(f"Saved tokenized dataset to {tokenized_train_dataset}")

    ## end your edits here  =================

###!@1 END ANSWER STEP 1
```

**Output schema** — satisfies `STEP_1_SCHEMA`:
- `warc_id`: string — preserved identifier per document
- `input_ids`: array\<int\> — variable-length token IDs including the appended EOS

**Why we keep `warc_id`:** The schema validator (`validate_step_1`) explicitly requires it. It also serves as a document-level trace for debugging.

**Why we remove `url`, `date`, `extracted_text`:** The raw text is no longer needed once tokenized. Keeping it doubles disk usage for no reason and would cause schema validation to fail (extra columns not in `STEP_1_SCHEMA`).

---

### 2.2 Efficiency Analysis — How to Make Tokenization Fast with a Fixed Tokenizer

Your doubt: *"Since the tokenizer is already loaded/fixed, how can we implement tokenization efficiently?"*

The key insight: **the tokenizer object is fixed, but how you invoke it over the dataset is entirely up to you.** Three independent levers give the speedup:

---

#### Lever 1: `batched=True` — Activate the Rust Fast-Path

```python
# SLOW: function called once per sample — pure Python overhead × 577,736
dataset.map(lambda x: {"input_ids": tokenizer(x["extracted_text"])["input_ids"]})

# FAST: function called with a dict-of-lists — activates GPT2TokenizerFast's Rust path
dataset.map(lambda batch: {"input_ids": tokenizer(batch["extracted_text"])["input_ids"]},
            batched=True)
```

`GPT2TokenizerFast` is backed by the HuggingFace `tokenizers` library written in **Rust**. When given a **list of strings**, it processes the whole batch inside Rust, releasing the Python GIL. This alone yields ~10–50× speedup over per-sample Python calls.

---

#### Lever 2: `num_proc=48` — True CPU Parallelism (Bypass the GIL)

```python
dataset.map(fn, batched=True, num_proc=multiprocessing.cpu_count())  # 48 on Turing cluster
```

- `datasets.map` with `num_proc > 1` **forks N worker subprocesses**
- The Arrow dataset is split into N equal **shards**, one per worker
- Each subprocess has its own **independent tokenizer copy** (serialized via `dill`, which supports closures unlike plain `pickle`)
- Workers run completely in parallel — zero GIL contention between subprocesses
- Results are written to separate Arrow shard files, then concatenated at the end

```
Arrow dataset (577,736 rows)
  ── shard 0 (12,036 rows) ──► Worker 0 (tokenizer copy) ──► shard_0.arrow
  ── shard 1 (12,036 rows) ──► Worker 1 (tokenizer copy) ──► shard_1.arrow
  ...
  ── shard 47 (12,036 rows) ──► Worker 47 (tokenizer copy) ──► shard_47.arrow
                                                      └─── concat ──► final arrow
```

On the Turing cluster: **48 CPU cores → ~48× wall-clock speedup** over single-process tokenization.

---

#### Lever 3: `writer_batch_size=10000` — Reduce Arrow I/O Flushes

The default is 1,000. Each flush writes buffered rows to the Arrow shard file on disk. Increasing to 10,000 means 10× fewer disk write calls per worker — meaningful on a shared NFS-style cluster where I/O latency matters.

---

#### Full Picture

| Configuration | Throughput (relative) | Notes |
|---|---|---|
| `map(fn)` — no batch, no parallel | 1× (baseline) | Python call overhead × N samples |
| `map(fn, batched=True)` | ~20–50× | Rust batch path in GPT2TokenizerFast |
| `map(fn, batched=True, num_proc=48)` | ~48× further | 48 parallel processes, no GIL |
| `+ writer_batch_size=10000` | marginal extra | Fewer I/O flushes |

---

### 2.3 EOS Token Handling

GPT-2's `eos_token_id = 50256` equals `bos_token_id = 50256` — this is intentional. GPT-2 was trained on documents concatenated and separated by `<|endoftext|>`. Every document must end with this token so the model learns document boundaries during the block-grouping in Step 2.

```python
# CORRECT: append as integer ID after tokenization
input_ids = [ids + [tokenizer.eos_token_id] for ids in tokenized["input_ids"]]

# AVOID: adding as a string before tokenization
text = example["extracted_text"] + "<|endoftext|>"  # fragile, BPE edge cases possible
```

**Why after tokenization, not before:**
- Appending the integer ID is explicit and unambiguous — no risk of the tokenizer splitting the string differently
- `add_special_tokens=False` is set explicitly to prevent the tokenizer from inserting BOS/EOS in unexpected positions (GPT-2 by default adds none, but being explicit future-proofs the code)

---

### 2.4 What NOT To Do

| Anti-pattern | Problem |
|---|---|
| `map(fn)` without `batched=True` | Python function call overhead per sample; no Rust batch fast-path |
| `num_proc=1` or not set | Wastes 47 of 48 available cores |
| `truncation=True, max_length=512` | Truncates sequences at this stage — corrupts the concatenation in Step 2 (blocks would be short at boundaries) |
| `padding="max_length"` | Pads to 512 with pad tokens — these get concatenated in Step 2, poisoning the data stream |
| Not calling `save_to_disk()` | Forces full re-tokenization on every script run (1–10 min wasted) |
| `load_dataset("parquet", data_dir=...)` without filtering | Crashes on the `_SUCCESS` marker file (0 bytes) |

---

## 3. Step 2 — Preprocessing

### 3.1 Why Concatenate and Chunk Instead of Padding?

Transformer models require **uniform-length inputs** for efficient GPU batching. There are two common approaches:

**Option A — Padding (naive):**
```
Doc 1: [t1, t2, ..., t38, EOS, PAD, PAD, ..., PAD]  ← 512 tokens, mostly padding
Doc 2: [t1, t2, ..., t512, t513 ← truncated!]
```
Problems:
- Short documents: most of the 512-token budget is wasted on PAD tokens
- Long documents: truncated at 512, losing content
- GPU computes attention over PAD tokens → wasted compute
- Irregular token distribution distorts learning

**Option B — Concatenate then Chunk (our approach):**
```
Stream: [t1...t38, EOS, t1...t201, EOS, t1...t512, EOS, t1...t89, EOS, ...]
Block 1: [t1...t38, EOS, t1...t201, EOS, t1...t272]   ← 512 tokens, dense
Block 2: [t1...t240, EOS, t1...t89, EOS, t1...t182]   ← 512 tokens, dense
```
Every single token position is a real token. GPU compute is **never wasted on padding**. This is how GPT-2, GPT-3, and all large language models are actually trained.

---

### 3.2 Implementation

```python
###!@2 START ANSWER STEP 2

def step_2_preprocessing():
    print(">>> Starting Step 2: Preprocessing...")

    ## start your edits here  =================

    t0 = time.time()
    tokenized = load_from_disk(tokenized_train_dataset)
    print(f"Loaded tokenized dataset: {len(tokenized)} samples | columns: {tokenized.column_names}")

    def group_texts(examples):
        # Flatten all input_ids in this batch into one continuous token stream
        concatenated_ids = sum(examples["input_ids"], [])
        # Compute the largest multiple of block_size that fits — discard remainder
        total_length = (len(concatenated_ids) // block_size) * block_size
        # Slice into fixed-length blocks
        chunks = [concatenated_ids[i : i + block_size] for i in range(0, total_length, block_size)]
        # For causal LM: target = input shifted by 1 (handled by the model); labels = input_ids copy
        return {"input_ids": chunks, "labels": [c[:] for c in chunks]}

    num_proc = multiprocessing.cpu_count()
    print(f"Grouping into blocks of {block_size} tokens with {num_proc} processes...")
    final_dataset = tokenized.map(
        group_texts,
        batched=True,
        batch_size=1000,
        num_proc=num_proc,
        remove_columns=tokenized.column_names,  # removes warc_id and input_ids (originals)
        desc="Grouping into blocks",
        writer_batch_size=10000,
    )

    print(f"Preprocessing done: {len(final_dataset)} blocks of {block_size} tokens "
          f"in {time.time() - t0:.2f}s")
    final_dataset.save_to_disk(final_train_dataset)
    print(f"Saved final dataset to {final_train_dataset}")

    ## end your edits here  =================

###!@2 END ANSWER STEP 2
```

**Output schema** — satisfies `validate_step_2`:
- `input_ids`: list of exactly **512** integers
- `labels`: identical copy of `input_ids` (causal language modelling target)

---

### 3.3 Deep Dive: `group_texts`

#### Line 1: Flattening with `sum(examples["input_ids"], [])`

```python
concatenated_ids = sum(examples["input_ids"], [])
```

When `batched=True` and `batch_size=1000`, `examples["input_ids"]` is a **list of 1000 lists** (one per tokenized document). `sum(..., [])` is a Python idiom for flattening a list of lists into a single list:

```python
# example input (simplified):
examples["input_ids"] = [[101, 102, 50256], [201, 202, 203, 50256], [301, 50256]]

# after sum(..., []):
concatenated_ids = [101, 102, 50256, 201, 202, 203, 50256, 301, 50256]
```

This creates a **continuous token stream** within the batch. Across the full dataset (577,736 documents, avg ~300 tokens + EOS each), this stream contains approximately **170–200 million tokens**.

> Note: `sum(lists, [])` is simple and correct for typical document sizes. For extremely long individual documents (tens of thousands of tokens), `itertools.chain.from_iterable()` would be more memory-efficient, but for this dataset it's fine.

---

#### Line 2: Truncate to a multiple of `block_size`

```python
total_length = (len(concatenated_ids) // block_size) * block_size
```

This is integer division followed by multiplication — it rounds down to the nearest multiple of 512:

```
concatenated_ids length = 5,137 tokens
block_size = 512
total_length = (5137 // 512) * 512 = 10 * 512 = 5120
→ discards the last 17 tokens
```

**Why discard remainder?** Keeping partial blocks would require padding the last block, reintroducing the very padding waste we avoided. Since we have ~170M tokens total, discarding at most 511 tokens per worker shard is negligible (< 0.001% data loss).

---

#### Line 3: Slicing into blocks

```python
chunks = [concatenated_ids[i : i + block_size] for i in range(0, total_length, block_size)]
```

Standard Python slice-and-step. Each chunk is exactly 512 tokens:

```
range(0, 5120, 512) → [0, 512, 1024, 1536, ..., 4608]
chunks[0] = concatenated_ids[0:512]     → 512 tokens
chunks[1] = concatenated_ids[512:1024]  → 512 tokens
...
chunks[9] = concatenated_ids[4608:5120] → 512 tokens
```

---

#### Line 4: Labels as a copy

```python
return {"input_ids": chunks, "labels": [c[:] for c in chunks]}
```

`c[:]` creates a **shallow copy** of each chunk list (not a reference). Both `input_ids` and `labels` contain the same token IDs.

**Why `labels = input_ids`?** This is the standard setup for **causal language modelling**. The model is trained to predict the next token at each position. During the forward pass, `GPT2LMHeadModel` internally shifts the labels left by one:
```
input_ids:  [t1, t2, t3, ..., t512]
targets:    [t2, t3, t4, ..., t512, (ignored)]  ← shift applied inside the model
```
So providing `labels = input_ids` is correct — the model handles the offset internally.

---

### 3.4 Efficiency Analysis

The same three levers apply as in Step 1, with one key difference: `group_texts` is **pure Python** (no Rust fast-path), so `num_proc` is even more critical here.

#### Why `num_proc` matters more here

GPT-2's fast tokenizer releases the GIL inside Rust. `group_texts` has no such escape — it's Python list operations throughout. Without `num_proc`, this step runs entirely single-threaded in CPython.

```python
# WITHOUT num_proc: entire 577k-row dataset processed sequentially in Python
tokenized.map(group_texts, batched=True)

# WITH num_proc=48: 48 shards processed in parallel, true CPU concurrency
tokenized.map(group_texts, batched=True, num_proc=48)
```

#### `batch_size=1000` for block grouping

The batch size in `group_texts` determines how many documents are concatenated per call. With `batch_size=1000` and avg ~300 tokens/doc:
- ~300,000 tokens per batch → ~585 blocks of 512 per function call
- Good balance: enough tokens for meaningful chunking, not so much that RAM spikes

If `batch_size` were 1 (i.e., one document per call), you'd get ~300 tokens per call — almost never a full 512-token block, generating almost no output. **Batch size must be large enough to span multiple blocks.**

#### `remove_columns` order of operations

HuggingFace `datasets.map` applies `remove_columns` to the **output** *before* merging the function's return dict. So:

```
Input columns: ["warc_id", "input_ids"]
↓ remove_columns=["warc_id", "input_ids"] (originals stripped from output)
↓ merge function return: {"input_ids": blocks_of_512, "labels": ...}
Output columns: ["input_ids", "labels"]   ← only the new block data ✅
```

This is why naming `remove_columns=tokenized.column_names` is safe even though the function also returns `input_ids` — the original variable-length `input_ids` is stripped, and the new fixed-512 `input_ids` from the function takes its place.

---

### 3.5 Column Management in `map()`

After Step 1, `tokenized.column_names = ["warc_id", "input_ids"]`:
- `warc_id`: document ID — meaningless for 512-token blocks that may span multiple docs
- `input_ids`: variable-length token sequences — being replaced by fixed-length blocks

Using `remove_columns=tokenized.column_names` removes both, leaving only what the function returns.

---

### 3.6 What NOT To Do

| Anti-pattern | Problem |
|---|---|
| `batch_size=1` in `group_texts` | ~300 tokens/call — almost never fills a 512-token block; output would be nearly empty |
| Not using `num_proc` | `group_texts` is pure Python; 48× slower without multiprocessing |
| Applying padding in this step | Contradicts the entire point of block-based preprocessing; corrupts training |
| Keeping `warc_id` in final dataset | `validate_step_2` / `default_data_collator` will fail on a non-tensor string column |
| Not using `c[:]` for labels copy | `labels` would reference the same list object as `input_ids`; safe here but bad practice |
| `truncation=True` in Step 1 + block grouping | Sequences already truncated to 512 before grouping means every document wastes the remainder of its block |

---

## 4. Pipeline Execution Flow

```
/mnt/data/ds256_2026/as2/train_dataset/   (103 parquet files, 577,736 rows)
        │
        │  step_1_tokenization()
        │  - glob.glob("*.parquet")  → skip _SUCCESS
        │  - load_dataset("parquet")
        │  - tokenizer(batch, add_special_tokens=False)
        │  - append eos_token_id (50256) to each sequence
        │  - remove url, date, extracted_text
        │  - save_to_disk()
        ▼
/scratch/saisandeshk/train_dataset/tokenized/   (577,736 rows: warc_id + input_ids[variable])
        │
        │  main: load_from_disk() → validate_step_1() → schema check (warc_id + input_ids)
        │
        │  step_2_preprocessing()
        │  - load_from_disk()
        │  - group_texts: sum(batch_ids, []) → chunk to 512 → labels = copy
        │  - remove warc_id + original input_ids
        │  - save_to_disk()
        ▼
/scratch/saisandeshk/train_dataset/final/   (~500k–700k rows: input_ids[512] + labels[512])
        │
        │  main: load_from_disk() → validate_step_2() → DataLoader shape check (1, 512)
        ▼
        ✅  Part 1 complete — ready for a2p2_v0.1.py
```

---

## 5. How to Run in Your Slot

### Step 0 — Check your slot
```bash
/apps/myslot.sh
```

Your team (team05) slot: **odd days, 08:00–12:00**. Look for:
```
✅ Your slot is ACTIVE now!
   Reservation: team05_20260409_0800

   Run your job with:
   srun -N <num_gpus> --ntasks=<num_gpus> --partition=ds256 --qos=ds256_qos --reservation=team05_20260409_0800 -t <HH:MM:SS> /apps/run_wrapper.sh <your_script.py>
```

If not active:
```
⏳ Your slot is NOT active yet.
   Next reservation: team05_20260411_0800
   Starts at: 2026-04-11T08:00:00
```

---

### Step 1 — Ensure scratch directory exists (can be done outside slot)
```bash
mkdir -p /scratch/saisandeshk/train_dataset/tokenized
mkdir -p /scratch/saisandeshk/train_dataset/final
ls /scratch/saisandeshk/
```

---

### Step 2 — Submit the Part 1 job (must be inside slot, N=1 mandatory)

```bash
srun -N 1 \
    --ntasks=1 \
    --partition=ds256 \
    --qos=ds256_qos \
    --reservation=<your_reservation_name> \
    -t 02:00:00 \
    /apps/run_wrapper.sh /home/saisandeshk/A-2/a2p1_v0.1.py
```

> Replace `<your_reservation_name>` with the active reservation (e.g., `team05_20260411_0800`).
> `-t 02:00:00` = 2-hour time limit. Adjust to your slot's remaining time if needed (slot is 4h total).
> **Do not change `-N 1 --ntasks=1`** — the preprocessing must run on a single node per the guidelines.

---

### Step 3 — Monitor progress (in another terminal)
```bash
# Watch SLURM queue
watch squeue -u saisandeshk

# Check output files as they appear
watch ls -lh /scratch/saisandeshk/train_dataset/tokenized/
watch ls -lh /scratch/saisandeshk/train_dataset/final/
```

---

## 6. Validation & Verification

Run these after the job completes (no slot needed — CPU only):

```bash
/mnt/data/miniconda3/envs/saisandeshk/bin/python -c "
from datasets import load_from_disk
from transformers import default_data_collator
from torch.utils.data import DataLoader

print('=== Step 1 Validation ===')
tok = load_from_disk('/scratch/saisandeshk/train_dataset/tokenized')
print('Rows:', len(tok))                             # expect 577,736
print('Columns:', tok.column_names)                  # expect ['warc_id', 'input_ids']
print('Sample warc_id:', tok[0]['warc_id'])
print('Sample input_ids[:10]:', tok[0]['input_ids'][:10])
print('EOS at end:', tok[0]['input_ids'][-1] == 50256)  # expect True

print()
print('=== Step 2 Validation ===')
final = load_from_disk('/scratch/saisandeshk/train_dataset/final')
print('Blocks:', len(final))                         # expect 500k–700k
print('Columns:', final.column_names)                # expect ['input_ids', 'labels']
print('input_ids length:', len(final[0]['input_ids']))  # expect 512
print('labels == input_ids:', final[0]['input_ids'] == final[0]['labels'])  # expect True

dl = DataLoader(final, batch_size=1, collate_fn=default_data_collator)
batch = next(iter(dl))
print('DataLoader input_ids shape:', batch['input_ids'].shape)  # expect torch.Size([1, 512])
print('DataLoader labels shape:   ', batch['labels'].shape)     # expect torch.Size([1, 512])
print()
print('ALL VALIDATIONS PASSED ✅')
"
```

---

### Expected output sizes

| Dataset | Expected rows | Format | Location |
|---|---|---|---|
| Raw train | 577,736 | 103 parquet files | `/mnt/data/ds256_2026/as2/train_dataset/` |
| Tokenized train | 577,736 | HuggingFace Arrow | `/scratch/saisandeshk/train_dataset/tokenized/` |
| Final train (blocks) | ~500k–700k | HuggingFace Arrow | `/scratch/saisandeshk/train_dataset/final/` |
| Test (pre-made) | 41,401 | HuggingFace Arrow | `/mnt/data/ds256_2026/as2/test_dataset/` |

**Estimating final block count:**
```
577,736 docs × avg ~300 tokens/doc × 1.1 (EOS overhead) ≈ 190M tokens
190M tokens ÷ 512 tokens/block ≈ ~371,000 blocks
```
Actual number depends on true document length distribution — expect 300k–500k blocks.

---

## 7. Summary

| | Step 1: Tokenization | Step 2: Preprocessing |
|---|---|---|
| **Input** | 577,736 raw text rows (103 parquet files) | 577,736 tokenized rows (Arrow) |
| **Core operation** | GPT-2 tokenizer + EOS append | Flatten → chunk into 512 → discard remainder |
| **Output** | 577,736 rows: `warc_id` + variable-length `input_ids` | ~300k–500k blocks: `input_ids`[512] + `labels`[512] |
| **Efficiency technique** | `batched=True` (Rust fast-path) + `num_proc=48` + `writer_batch_size=10000` | `batched=True` + `num_proc=48` (critical: pure Python op) + `writer_batch_size=10000` |
| **Key constraint** | Append EOS (id=50256); no truncation/padding | `batch_size` must be large enough to span blocks; discard remainder |
| **Output path** | `/scratch/saisandeshk/train_dataset/tokenized/` | `/scratch/saisandeshk/train_dataset/final/` |
| **Estimated runtime** | ~5–15 min with 48 cores | ~5–10 min |
| **SLURM constraint** | N=1, ntasks=1 (mandatory) | Same |
| **Validation** | Schema check: `warc_id` + `input_ids` columns | DataLoader shape: `(1, 512)` |
