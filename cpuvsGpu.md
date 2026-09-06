# Why GPUs Matter for Running Large AI Models

Yes — at a basic level, a CPU is designed to execute instructions in sequence, but modern CPUs also do some work in parallel.

The important difference for understanding AI models is not simply **“CPU = sequential, GPU = parallel.”** It is:

> **A CPU is optimized for doing a relatively small number of complicated tasks very quickly. A GPU is optimized for doing a huge number of similar mathematical operations simultaneously.**

That distinction is exactly why GPUs are so important for large neural networks.

---

## 1. CPU vs GPU

Imagine a CPU with 8–16 powerful cores. Each core can execute instructions extremely quickly and does clever things such as:

- Branch prediction
- Out-of-order execution
- Caching
- Vector instructions
- Multiple threads

So CPUs are not literally doing one instruction at a time across the whole processor.

But suppose you need to perform:

```text
y = W × x
```

where `W` is a gigantic matrix containing millions or billions of model weights.

For a tiny example:

```text
Weights:

[ 0.2  0.4  0.1 ]
[ 0.7  0.3  0.9 ]
[ 0.5  0.8  0.2 ]

Input:

[ 5 ]
[ 2 ]
[ 7 ]
```

You calculate:

```text
0.2×5 + 0.4×2 + 0.1×7
0.7×5 + 0.3×2 + 0.9×7
0.5×5 + 0.8×2 + 0.2×7
```

Notice something important:

These multiplications are mostly independent.

You do not need to calculate:

```text
0.2×5
```

before calculating:

```text
0.4×2
```

They can happen at the same time.

That is where GPUs shine.

### Simplified CPU

```text
Core 1  ───── calculation
Core 2  ───── calculation
Core 3  ───── calculation
Core 4  ───── calculation
Core 5  ───── calculation
Core 6  ───── calculation
Core 7  ───── calculation
Core 8  ───── calculation
```

### Simplified GPU

```text
core   core   core   core   core   core   core ...
core   core   core   core   core   core   core ...
core   core   core   core   core   core   core ...
core   core   core   core   core   core   core ...
...
thousands of execution units
```

So instead of doing a few matrix operations at once, the GPU can perform enormous numbers of multiply/add operations concurrently.

This maps extremely well to neural networks.

---

## 2. Connecting This to Billions of Parameters

Suppose a model has:

```text
10 billion parameters
```

For simplicity, imagine each parameter occupies 2 bytes using FP16 or BF16.

Just storing the weights requires roughly:

```text
10 billion × 2 bytes = 20 GB
```

Already you have a problem on many laptops.

Your CPU might have 32 GB system RAM, so perhaps the model technically fits.

But fitting in memory does not mean it runs fast.

During inference, every transformer layer performs calculations resembling:

```text
input
   ↓
matrix multiplication
   ↓
activation
   ↓
matrix multiplication
   ↓
attention calculations
   ↓
matrix multiplication
   ↓
next layer
```

And those matrices are enormous.

A simplified transformer layer may look like:

```text
Token vectors
      │
      ▼
┌──────────────┐
│ Q projection │ ── huge matrix multiplication
└──────────────┘
      │
┌──────────────┐
│ K projection │ ── huge matrix multiplication
└──────────────┘
      │
┌──────────────┐
│ V projection │ ── huge matrix multiplication
└──────────────┘
      │
      ▼
    Attention
      │
      ▼
┌──────────────┐
│ Feed-forward │ ── even more huge matrix multiplications
└──────────────┘
```

And there might be:

```text
30
40
80
100+
```

transformer layers.

So generating one token can require billions of arithmetic operations.

---

## 3. LLM Generation Is Still Sequential

Even with a GPU, the model itself still has some sequential dependency.

Suppose the model generates:

```text
AWS uses EC2 for ...
```

To generate that output:

```text
token 1 → AWS
token 2 → uses
token 3 → EC2
token 4 → for
```

the model normally cannot generate token 4 before it has generated token 3.

So token generation is autoregressive:

```text
Input
  ↓
Generate token 1
  ↓
Input + token 1
  ↓
Generate token 2
  ↓
Input + token 1 + token 2
  ↓
Generate token 3
```

That part is sequential.

But the huge amount of mathematics required inside each of those steps is massively parallel.

That is where the GPU helps.

```text
                  Sequential
                     ↓
             Generate next token
                     │
                     ▼
      ┌───────────────────────────┐
      │ Millions/billions of math│
      │ operations               │
      │                           │
      │  × × × × × × × × × ×   │
      │  + + + + + + + + + +   │
      │  × × × × × × × × × ×   │
      └───────────────────────────┘
                     ↑
              massively parallel
                     │
                    GPU
```

So GPUs do not eliminate the sequential nature of language generation.

They make each individual step much faster.

---

## 4. Why a Powerful Laptop CPU Is Still Not Enough

There are three major bottlenecks.

### 4.1 Memory Capacity

The weights have to live somewhere.

For approximate FP16 storage:

| Model Size | Approximate Weight Memory |
|---|---:|
| 1B | ~2 GB |
| 7B | ~14 GB |
| 13B | ~26 GB |
| 30B | ~60 GB |
| 70B | ~140 GB |

Quantization can dramatically reduce this.

For example, a 70B model at 4-bit precision needs roughly:

```text
70B × 0.5 bytes ≈ 35 GB
```

plus some additional runtime overhead.

This is why tools such as Ollama and llama.cpp can run surprisingly large models locally: they often use quantized weights.

---

### 4.2 Memory Bandwidth

This is extremely important and often overlooked.

When generating tokens, the processor repeatedly has to read enormous amounts of weight data.

Suppose your model has:

```text
20 GB of weights
```

and your memory system can effectively deliver:

```text
100 GB/s
```

Ignoring all other overhead:

```text
20 GB / 100 GB/s = 0.2 seconds
```

That gives a rough upper bound of only about:

```text
5 weight-streaming passes per second
```

for that simplified example.

Now imagine a GPU with:

```text
1500 GB/s
```

of memory bandwidth.

The same conceptual workload can be supplied much faster.

Modern AI accelerators often use very high-bandwidth memory such as HBM.

So for LLM inference:

> **Memory bandwidth can be just as important as raw compute.**

---

### 4.3 Parallel Computation

Consider a matrix operation like:

```text
A × B = C
```

There may be millions of individual multiply-add operations.

A CPU has relatively few powerful cores:

```text
CPU:

[Core]
[Core]
[Core]
[Core]
[Core]
[Core]
[Core]
[Core]
```

A GPU provides huge numbers of simpler parallel execution lanes:

```text
GPU:

[][][][][][][][][][][][][][][][][]
[][][][][][][][][][][][][][][][][]
[][][][][][][][][][][][][][][][][]
[][][][][][][][][][][][][][][][][]
[][][][][][][][][][][][][][][][][]
...
```

This is why GPU architecture fits neural networks so well.

---

## 5. Tensor Cores and AI-Specific Hardware

AI-oriented GPUs add another optimization: specialized matrix hardware.

NVIDIA calls these **Tensor Cores**.

Instead of treating matrix multiplication as generic arithmetic, the chip contains hardware heavily optimized for operations like:

```text
D = A × B + C
```

which appears constantly in neural-network workloads.

A useful hierarchy is:

```text
CPU
↓
excellent general-purpose processor
few powerful cores

GPU
↓
thousands of parallel execution units
very good at matrix/vector math

AI-oriented GPU
↓
GPU
+
Tensor Cores / matrix units
+
very high-bandwidth memory
+
lower-precision arithmetic
```

---

## 6. Why Lower Precision Helps

Instead of always using FP32:

```text
FP32 = 32 bits
```

AI workloads often use:

```text
FP16
BF16
FP8
INT8
INT4
```

Less precision can mean:

```text
less memory
+
less data moved
+
more operations per second
```

For neural networks, that can be a huge performance advantage.

---

## 7. What Happens When You Ask an LLM a Question

Suppose you ask:

```text
What is AWS Lambda?
```

The model does not search for some specific group of “AWS weights” and execute only those.

A simplified flow is:

```text
"What is AWS Lambda?"
        ↓
Tokenizer
        ↓
tokens
        ↓
Embeddings
        ↓
vectors
        ↓

Transformer layer 1
  ↓
matrix multiplication using layer 1 weights

Transformer layer 2
  ↓
matrix multiplication using layer 2 weights

Transformer layer 3
  ↓

...

Transformer layer N
        ↓
probabilities over vocabulary
        ↓
next token
```

Most of the model's weights participate in the computation.

That means a 70B model may need to move and use an enormous amount of parameter data for every generated token.

That is the real reason hardware matters so much.

---

## 8. A Useful Mental Model

Think of a CPU as:

> **A small team of extremely capable engineers doing complicated jobs.**

Think of a GPU as:

> **A massive factory with thousands of workers doing similar calculations simultaneously.**

An LLM mostly says:

> **“I need an enormous number of very similar matrix calculations done.”**

So the factory wins.

A giant model therefore creates two separate problems:

1. **Can I fit all the weights into memory?**
2. **Can I move and process those weights quickly enough?**

A machine can have enough RAM to load a model and still generate tokens painfully slowly.

---

## 9. The Core Idea to Remember

Do not reduce the distinction to:

```text
CPU = sequential
GPU = parallel
```

A better mental model is:

```text
CPU
→ optimized for low-latency, general-purpose, branch-heavy work
→ relatively few very powerful cores

GPU
→ optimized for throughput
→ huge amounts of similar math in parallel
→ enormous memory bandwidth

LLM
→ massive matrix multiplications
→ billions of weights
→ repeated for every generated token
```

That combination is why GPUs are so effective for modern AI workloads.

---

## 10. What Is VRAM?

**VRAM (Video RAM)** is memory that belongs directly to the GPU.

A simple way to think about a computer is:

```text
CPU
│
└── System RAM
    e.g. 16 GB / 32 GB / 64 GB

GPU
│
└── VRAM
    e.g. 8 GB / 16 GB / 24 GB / 48 GB / 80 GB
```

System RAM is primarily used by the CPU.

VRAM is primarily used by the GPU.

For AI workloads, VRAM is extremely important because the GPU needs fast access to:

```text
model weights
+
current activations
+
KV cache
+
temporary calculation buffers
```

Ideally, these stay in VRAM while the model is running.

---

### Why Not Just Use Normal RAM?

Suppose your laptop has:

```text
System RAM = 32 GB
GPU VRAM   = 8 GB
```

and you try to run a model that needs 12 GB of GPU-accessible memory.

The GPU cannot fit everything into its 8 GB VRAM.

Some data may have to remain in system RAM:

```text
System RAM
    │
    │ transfer over PCIe / shared interconnect
    ▼
GPU VRAM
    │
    ▼
GPU calculation
```

This transfer is much slower than accessing data already inside VRAM.

The GPU is extremely fast, but if it constantly has to wait for model data to arrive from system RAM, much of that compute power is wasted.

A useful analogy is:

```text
VRAM
= tools already on your workbench

System RAM
= tools stored in another room
```

If everything you need is on the workbench, you work quickly.

If you have to walk into another room before every operation, the process becomes much slower.

---

## 11. VRAM Capacity vs VRAM Bandwidth

There are two separate VRAM properties that matter.

### VRAM Capacity

This answers:

> How much data can the GPU hold at once?

Example:

```text
RTX-class GPU
VRAM = 8 GB
```

versus:

```text
High-end GPU
VRAM = 24 GB
```

versus AI/data-center GPUs with much larger memory capacities.

A larger VRAM capacity allows larger models to stay completely on the GPU.

For example, consider a rough FP16 model:

```text
7B parameters × 2 bytes
≈ 14 GB
```

An 8 GB GPU cannot store all 14 GB of weights in VRAM.

A 24 GB GPU potentially can.

This is one reason why VRAM size matters so much for local LLMs.

---

### VRAM Bandwidth

Capacity tells us how much can fit.

Bandwidth tells us how quickly the GPU can read and write that memory.

For example:

```text
GPU A:
24 GB VRAM
500 GB/s bandwidth

GPU B:
24 GB VRAM
1,500 GB/s bandwidth
```

Both can hold the same amount of data.

But GPU B can feed data into its compute units much faster.

For LLM inference, this can make a major difference because the model repeatedly reads large weight matrices.

So:

```text
VRAM capacity
= how big the warehouse is

VRAM bandwidth
= how quickly goods can leave the warehouse
```

You need both.

---

## 12. Why LLMs Consume VRAM

When running an LLM, VRAM is not used only for weights.

A simplified breakdown is:

```text
VRAM usage

├── Model weights
├── Activations
├── KV cache
└── Temporary buffers / runtime overhead
```

Let's look at each one.

### 12.1 Model Weights

This is usually the largest fixed component.

Suppose:

```text
Model = 7 billion parameters
Precision = FP16
```

FP16 requires approximately 2 bytes per parameter:

```text
7B × 2 bytes
≈ 14 GB
```

So approximately 14 GB is required just for the weights.

This is before considering the rest of the runtime memory.

---

### 12.2 Activations

As discussed earlier:

```text
weights
= learned values stored in the model

activations
= temporary values produced by the current input
```

During a forward pass:

```text
input
 ↓
layer
 ↓
activation
 ↓
next layer
 ↓
activation
 ↓
next layer
```

Those intermediate vectors require memory.

During inference, many temporary activations can be discarded once they are no longer needed.

During training, however, many activations must be preserved so gradients can later be calculated during backpropagation.

That is one reason training requires dramatically more GPU memory than inference.

---

## 13. The KV Cache

The **KV cache** is another major source of VRAM usage during LLM inference.

KV stands for:

```text
K = Key
V = Value
```

These come from the transformer's attention mechanism.

Suppose your conversation contains:

```text
User: Explain AWS Lambda.
Assistant: AWS Lambda is ...
User: How is it different from ECS?
```

When generating the next token, the model needs information from previous tokens.

Without caching, it would repeatedly recompute some attention information for all previous tokens.

Instead, it stores previously calculated Keys and Values:

```text
Token 1 ── Key + Value ─┐
Token 2 ── Key + Value ─┤
Token 3 ── Key + Value ─┤
Token 4 ── Key + Value ─┤
...                      │
                         ▼
                      KV Cache
```

For each new token:

```text
new token
   ↓
calculate new Query
   ↓
compare with cached Keys
   ↓
use cached Values
   ↓
generate next token
```

This makes generation much faster.

But there is a cost:

> The longer the context, the larger the KV cache becomes.

So a model running with:

```text
2,000 token context
```

needs less KV-cache memory than the same model running with:

```text
32,000 token context
```

or:

```text
128,000 token context
```

This is one reason long-context models can consume much more GPU memory.

---

## 14. Why Quantization Helps VRAM

Quantization reduces how many bits are used to represent model weights.

For example:

```text
FP32
= 32 bits
= 4 bytes per parameter

FP16
= 16 bits
= 2 bytes per parameter

INT8
= 8 bits
= 1 byte per parameter

INT4
= 4 bits
= 0.5 bytes per parameter
```

For a 7B model:

```text
FP32:
7B × 4 bytes
≈ 28 GB

FP16:
7B × 2 bytes
≈ 14 GB

8-bit:
7B × 1 byte
≈ 7 GB

4-bit:
7B × 0.5 bytes
≈ 3.5 GB
```

There is additional overhead in real implementations, so these are simplified estimates.

But the key idea remains:

```text
lower precision
→ smaller model
→ less VRAM needed
→ larger models can run on consumer GPUs
```

This is why a 7B model that would not fit into an 8 GB GPU at FP16 may fit when quantized to 4-bit.

---

## 15. What Happens When the Model Does Not Fit in VRAM?

Suppose you have:

```text
Model runtime requirement = 14 GB
GPU VRAM                 = 8 GB
System RAM               = 32 GB
```

Different frameworks can handle this in different ways.

One possibility is **CPU/GPU offloading**.

For example:

```text
Model layers

Layer 1   ┐
Layer 2   │
Layer 3   │
Layer 4   ├── GPU VRAM
Layer 5   │
Layer 6   ┘

Layer 7   ┐
Layer 8   │
Layer 9   ├── System RAM / CPU
Layer 10  ┘
```

When processing reaches a layer stored outside the GPU, data must move between CPU memory and GPU memory.

Conceptually:

```text
GPU
 ↓
PCIe / interconnect
 ↓
CPU RAM
 ↓
PCIe / interconnect
 ↓
GPU
```

That can work.

But it is usually much slower than keeping the entire model in VRAM.

So there is a big difference between:

```text
"the model can run"
```

and:

```text
"the model can run efficiently"
```

---

## 16. Unified Memory Systems

Some computers use **unified memory** instead of completely separate CPU RAM and GPU VRAM.

A simplified picture:

```text
Traditional PC

CPU ── System RAM

GPU ── VRAM
```

versus:

```text
Unified-memory system

        Shared Memory
        /           \
      CPU           GPU
```

This can make it easier for CPU and GPU to access the same memory pool.

However, memory capacity and bandwidth still matter.

A machine having:

```text
64 GB unified memory
```

does not automatically mean it performs like a dedicated AI GPU with:

```text
64 GB high-bandwidth VRAM
```

The architecture and memory bandwidth are different.

---

## 17. Training Uses Much More VRAM Than Inference

During inference, you mostly need:

```text
weights
+
KV cache
+
current activations
+
temporary buffers
```

Training needs considerably more:

```text
model weights
+
activations
+
gradients
+
optimizer state
+
temporary buffers
```

For example, training conceptually requires:

```text
forward pass
     ↓
store activations
     ↓
calculate loss
     ↓
backpropagation
     ↓
calculate gradients
     ↓
update weights
```

The gradients themselves consume memory.

Optimizers such as Adam also maintain additional values for each parameter.

That is why:

```text
Model can run inference on one GPU
```

does **not** imply:

```text
Model can be trained on that GPU
```

A GPU that can comfortably run a model may still have nowhere near enough VRAM to train it.

---

## 18. Putting CPU, RAM, GPU and VRAM Together

The full picture now looks like:

```text
                   COMPUTER
                       │
        ┌──────────────┴──────────────┐
        │                             │
       CPU                           GPU
        │                             │
        ▼                             ▼
   System RAM                      VRAM
        │                             │
general programs                model weights
OS                              activations
application state               KV cache
some model data                 GPU buffers
        │                             │
        └────────── transfer ─────────┘
```

Ideally for fast LLM inference:

```text
             GPU
              │
              ▼
        ┌───────────┐
        │   VRAM    │
        │           │
        │ weights   │
        │ KV cache  │
        │ activations│
        └───────────┘
              │
              ▼
     thousands of parallel
        compute units
              │
              ▼
         next token
```

The GPU can repeatedly access everything locally at very high speed.

---

## 19. The Most Important VRAM Mental Model

Imagine the GPU is a factory.

```text
GPU compute units
= factory workers

VRAM
= warehouse attached directly to the factory

VRAM bandwidth
= conveyor belts between warehouse and workers

System RAM
= another warehouse farther away
```

If the factory has thousands of workers but the attached warehouse is too small:

```text
workers wait for materials
```

If the warehouse is large but its conveyor belts are slow:

```text
workers still wait for materials
```

For AI workloads you therefore care about:

```text
GPU compute
+
VRAM capacity
+
VRAM bandwidth
```

not merely the number of GPU cores.

---

## 20. Final Combined Mental Model

When you run an LLM:

```text
Prompt
  ↓
Tokenizer
  ↓
Embeddings
  ↓
GPU loads/uses model weights from VRAM
  ↓
Massively parallel matrix calculations
  ↓
Activations
  ↓
Attention
  ↓
KV cache stores previous attention information
  ↓
More transformer layers
  ↓
Next-token probabilities
  ↓
Generate one token
  ↓
Repeat
```

The important hardware relationship is:

```text
Model size
        ↓
How much memory is required?
        ↓
VRAM capacity

Huge matrix operations
        ↓
How quickly can they be calculated?
        ↓
GPU compute

Billions of weights repeatedly accessed
        ↓
How quickly can they reach the compute units?
        ↓
VRAM bandwidth

Long conversation/context
        ↓
More cached Key/Value tensors
        ↓
More KV-cache VRAM
```

So when evaluating whether a computer can run a large model, do not ask only:

> "How powerful is the GPU?"

Ask:

```text
1. How much VRAM does it have?
2. What is its VRAM bandwidth?
3. How much compute does the GPU provide?
4. What precision/quantization is the model using?
5. How large is the context window?
6. Does the entire model fit in VRAM?
```

Those factors together determine whether a model merely runs or runs well.

---

## 21. What Problem Does VRAM Solve Compared with Normal RAM?

VRAM solves a very specific problem:

> **The GPU needs extremely fast, local access to the model data it is constantly using.**

A model is not loaded once and then left alone. During inference, the GPU repeatedly reads and writes large amounts of data while generating every token.

A simplified flow looks like:

```text
Prompt
  ↓
Embeddings
  ↓
Transformer layer
  ↓
Read model weights
  ↓
Matrix calculations
  ↓
Write activations
  ↓
Attention
  ↓
Read/write KV cache
  ↓
Next transformer layer
  ↓
...
  ↓
Generate next token
```

The important point is that the GPU needs to access:

```text
model weights
+
activations
+
KV cache
+
temporary buffers
```

over and over again.

### If the Data Is in Normal RAM

Normal system RAM is primarily connected to the CPU.

If the GPU needs data stored there, that data has to travel across an interconnect such as PCIe:

```text
System RAM
    │
    ▼
PCIe / interconnect
    │
    ▼
GPU
    │
    ▼
calculation
```

The GPU may be capable of enormous parallel compute, but it cannot calculate on data it has not received yet.

If it constantly waits for model weights or intermediate data to arrive from system RAM, much of the GPU's compute capacity sits idle.

That is the bottleneck VRAM is designed to reduce.

---

### Why VRAM Helps

VRAM is memory located directly next to the GPU and designed to feed large amounts of data to the GPU's execution units at very high bandwidth.

Think of it like this:

```text
System RAM
= central warehouse

VRAM
= warehouse attached directly to the factory floor

GPU cores
= factory workers
```

A GPU has thousands of parallel execution units.

If those units constantly need:

```text
weights
activations
KV-cache values
temporary tensors
```

then keeping that data in VRAM means the GPU can access it much faster.

Instead of:

```text
System RAM
   ↓
interconnect
   ↓
GPU
```

the normal fast path becomes:

```text
VRAM
  ↕
GPU
```

This is one of the biggest reasons VRAM is so important for AI workloads.

---

## 22. What Happens Internally During Model Inference?

Consider one simplified transformer layer.

The current token representations enter the layer:

```text
input activations
       ↓
```

The model then performs several operations using learned weight matrices:

```text
input
 ├── multiply with Q weights
 ├── multiply with K weights
 └── multiply with V weights
```

This creates new temporary values:

```text
Q activations
K activations
V activations
```

Then attention calculations happen:

```text
Q
│
├── compare with K
│
▼
attention scores
│
▼
use V
│
▼
new activations
```

Those activations then go through more matrix operations in the feed-forward part of the transformer:

```text
activations
    ↓
matrix multiplication
    ↓
activation function
    ↓
matrix multiplication
    ↓
new activations
```

This process repeats through many transformer layers.

During all of this, the GPU is constantly reading and writing data.

That is why it is useful for the following to stay close to the GPU:

```text
weights
activations
KV cache
temporary tensors
```

---

## 23. Why Sending Intermediate Results Back to RAM Would Be Slow

Imagine every operation had to work like this:

```text
GPU calculates something
        ↓
send result to system RAM
        ↓
read it back from system RAM
        ↓
continue next operation
```

The GPU would spend too much time waiting for memory transfers.

Instead, with VRAM:

```text
VRAM
  ↕
GPU
  ↕
VRAM
  ↕
GPU
```

The working data stays close to the compute hardware.

This is especially important because neural-network workloads consist of many large tensor and matrix operations that happen one after another.

---

## 24. VRAM and the KV Cache

The KV cache is another good example of why VRAM matters.

For attention, the model creates:

```text
K = Key
V = Value
```

for previous tokens.

Instead of recomputing them for every new token, the model stores them.

Conceptually:

```text
Token 1 → K + V ┐
Token 2 → K + V ├── KV cache
Token 3 → K + V ┤
Token 4 → K + V ┘
```

When a new token is processed:

```text
new token
   ↓
create Query
   ↓
compare Query with cached Keys
   ↓
use cached Values
   ↓
continue attention calculation
```

If the KV cache is in VRAM, the GPU can access it quickly.

If the context becomes very long, the KV cache grows.

That is why longer context windows can consume significantly more VRAM.

---

## 25. What Happens When the Model Does Not Fit in VRAM?

Suppose the GPU does not have enough VRAM for:

```text
all model weights
+
KV cache
+
activations
+
runtime buffers
```

Then some parts of the model may need to stay in system RAM.

A runtime may perform CPU/GPU offloading:

```text
Some layers
   ↓
VRAM / GPU

Other layers
   ↓
System RAM / CPU
```

This allows a larger model to run.

But the downside is additional movement:

```text
System RAM
    ↕
interconnect
    ↕
GPU VRAM
```

That is much slower than keeping everything on the GPU.

So there is a major difference between:

```text
the model fits somewhere in the computer's memory
```

and:

```text
the model fits completely in GPU VRAM
```

The second usually gives much better inference performance.

---

## 26. VRAM Capacity Is Not the Only Important Thing

When thinking about model performance, three GPU characteristics matter:

```text
GPU compute
+
VRAM capacity
+
VRAM bandwidth
```

### GPU Compute

Determines how much parallel mathematical work the GPU can perform.

### VRAM Capacity

Determines how much of the model and runtime state can stay on the GPU.

### VRAM Bandwidth

Determines how quickly weights, activations, and cached data can be supplied to the GPU compute units.

A GPU can have enormous compute capability but still underperform if its execution units are constantly waiting for data.

---

## 27. VRAM Compared with CPU Caches

The same general principle exists inside CPUs.

A CPU does not rely only on normal RAM.

It uses:

```text
System RAM
   ↓
L3 cache
   ↓
L2 cache
   ↓
L1 cache
   ↓
CPU core
```

The closer data is to the processor, the faster it can be accessed.

The GPU has a similar hierarchy:

```text
System RAM
    ↓
VRAM
    ↓
GPU caches
    ↓
GPU execution units
```

VRAM is therefore not just "extra RAM for graphics."

For modern AI workloads, it acts as the large high-speed working memory that keeps the GPU supplied with model data.

---

## 28. Best Mental Model for VRAM

Think of a GPU as a huge factory.

```text
GPU execution units
= thousands of workers

VRAM
= warehouse attached directly to the factory

VRAM bandwidth
= conveyor belts between the warehouse and workers

System RAM
= warehouse farther away
```

If the factory has thousands of workers but the attached warehouse is too small, data has to be fetched from farther away.

If the warehouse is large but its conveyor belts are slow, workers still have to wait.

So for LLMs:

```text
GPU compute
= how many calculations can be done

VRAM capacity
= how much model data can stay close to the GPU

VRAM bandwidth
= how quickly that data reaches the GPU
```

That is the real reason VRAM matters so much for large models.
