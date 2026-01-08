**Overview**

Over the holidays, I deployed a recent open-weight LLM locally on my laptop and pushed it across context length, KV cache usage, and concurrency to understand where inference actually breaks down in practice.

This repo documents empirical observations, metrics, and mental models from that exploration — with a focus on how long-context inference behaves on real hardware, not benchmarks or theory.

The goal is not to optimize performance, but to build intuition for:

	•	How KV cache scales with context
	•	Where latency really comes from
	•	Why long-context inference feels slow even on powerful machines
	•	What matters (and what doesn’t) when serving models in production

**Quick Summary Sheet:**

<img width="1197" height="385" alt="image" src="https://github.com/user-attachments/assets/dd041b72-62f9-4064-b2a9-e76134322959" />



**Hardware & Environment**

Machine

	•	Apple MacBook Pro
	•	M4 Pro
	•	48 GB unified memory

Runtime

	•	macOS Metal backend (GPU acceleration via Apple Metal)
	•	Ollama (local model serving)
	•	Single-node, single-GPU setup
	•	No distributed execution
	•	No external memory or paging tricks

Model

	•	qwen3:4b-instruct
	•	Fully GPU-resident (all layers on GPU)
	•	KV cache enabled
	•	Context window tested up to 128k tokens
	•	Single-request unless otherwise noted


**Learning Goals**

This exploration focused on answering a few concrete questions:
	1.	What actually dominates latency in long-context inference?
	2.	How does KV cache impact memory, GPU time, and throughput?
	3.	Why does increasing context length hurt latency more than memory?
	4.	How does concurrency interact with KV cache reuse?
	5.	What production trade-offs become obvious when running this locally?



**Key Concepts (Quick Definitions)**

	•	Prefill (Prompt Eval)
The phase where input tokens are ingested and their Key/Value tensors are computed and stored in the KV cache.

	•	Decode (Generation Eval)
The phase where the model generates new tokens, attending over the entire existing KV cache for every step.

	•	KV Cache
Stores per-token attention keys and values. Scales linearly with context length and dominates memory usage.

	•	TTFT (Time to First Token)
User-perceived latency. Dominated by prefill at long context lengths.



**Test Matrix**

Experiments were run across multiple context sizes:

	•	~2k tokens
	•	~10k tokens (truncated and non-truncated)
	•	~100k tokens
	•	Context window set to 128k to avoid truncation effects


Metrics collected:

	•	Prompt eval count and duration
	•	Decode duration and throughput
	•	Total end-to-end latency
	•	GPU time (Activity Monitor)
	•	Peak RSS memory
	•	Disk reads/writes
	•	Server logs (KV cache behavior)



**High-Level Results**

1. Long Context Is Prefill-Dominated

At large context sizes:

	•	80%+ of total latency comes from prefill
	•	TTFT becomes unusable at 100k tokens
	•	Decode speed degrades, but far less dramatically than prefill

Long-context inference is not “slow generation” — it is slow ingestion.


2. Memory Grows Sublinearly, GPU Time Does Not

Observed scaling:

	•	Context length: ~25× (4k → 100k)
	•	Memory usage: ~7× (≈3 GB → ≈22 GB)
	•	GPU time: ~30× (≈43s → ≈22 min)

Why:

	•	KV cache is pre-allocated based on max context
	•	Attention compute scales with active tokens, not allocated memory
	•	Every new token attends to all previous tokens


3. KV Cache Saves Compute, Not Attention Cost

KV cache:

	•	Eliminates recomputing keys/values for cached tokens
	•	Does not eliminate attention over those tokens

Even with a warm cache:

	•	Each decode step still performs attention across the full context
	•	GPU time remains high at long context

KV cache reduces prefill cost, not decode attention cost.


4. Concurrency ≠ Batching

Observed behavior with two simultaneous clients:

	•	Second request reused KV cache
	•	Prefill became extremely fast
	•	Decode time remained unchanged
	•	Total latency increased due to scheduling

Interpretation:

	•	Concurrency trades memory for parallelism
	•	Batching trades latency for throughput
	•	KV cache helps concurrency only for shared prefixes


5. Context Window Size Is a Real Production Cost

Setting a 128k context window:

	•	Reserves large KV buffers even if unused
	•	Increases memory pressure
	•	Reduces safe concurrency
	•	Does not improve performance unless fully utilized

**Takeaway**:

Context length should be treated like a memory allocation decision, not a model capability toggle.



**What This Changes About Mental Models**

Common assumption:

“Long context is mostly a memory problem.”

Reality:

Long-context inference is attention-bound, not compute-bound.

Implications:

	•	Faster GPUs help less than expected
	•	More GPUs help more than faster ones
	•	Memory bandwidth and attention kernels dominate
	•	Prefill optimization matters more than decoding optimization at scale



**Why This Matters**

If you care about:

	•	Long-context RAG
	•	Agent memory
	•	Multi-turn conversations
	•	Serving LLMs at scale

Then understanding KV cache behavior, attention cost, and prefill dominance is foundational.

Theory helps. Running it locally makes it obvious.

Thank you for reading :)
