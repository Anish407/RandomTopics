# Ollama vs vLLM

## Overview

Ollama and vLLM are both tools used to run Large Language Models (LLMs), but they are designed with different priorities.

A useful mental model is:

```text
Ollama = developer-friendly model runtime and local model manager

vLLM   = high-performance LLM inference and serving engine
```

Ollama is **not a wrapper around vLLM**.

They are separate runtimes that can both load a model, run inference, expose APIs, and return generated tokens.

---

## The Basic Architecture

When you run an LLM, there are two different things involved:

```text
Model
  |
  | loaded by
  v
Inference Runtime
  |
  | called by
  v
Application
```

For example:

```text
Llama / Mistral / Qwen / Gemma
              |
              v
       Ollama OR vLLM
              |
              v
      Python / C# Backend
```

The model contains the learned parameters/weights.

Ollama or vLLM is the software responsible for loading those weights and executing the calculations required for inference.

---

# Ollama

Ollama is designed to make running LLMs easy, especially on a developer machine.

For example:

```bash
ollama run llama3.1
```

From the developer's perspective, this is very simple.

Behind the scenes, Ollama handles several things for you:

```text
1. Find/download the model
2. Store the model locally
3. Load the model into RAM and/or GPU memory
4. Start the inference runtime
5. Tokenize the prompt
6. Run the model
7. Generate tokens
8. Return the response
```

Ollama also exposes an HTTP API.

Conceptually:

```text
My Application
      |
      | HTTP
      v
+-------------+
|   Ollama    |
+-------------+
      |
      v
+-------------+
| Llama Model |
+-------------+
```

For example, a Python, C#, or JavaScript application can send prompts to the Ollama API instead of directly dealing with model loading and GPU execution.

---

## What Ollama Gives You

Ollama provides convenience around running models.

It handles things such as:

- downloading models
- storing models locally
- loading/unloading models
- model configuration
- prompt templates
- local API serving
- command-line interaction
- model packaging through `Modelfile`
- CPU/GPU inference management

Because of this, Ollama often feels like a wrapper.

But it is not a wrapper over vLLM.

Ollama has its own runtime architecture and has historically relied heavily on the llama.cpp ecosystem for model inference.

---

# vLLM

vLLM is primarily designed for efficient, high-throughput LLM inference.

Instead of focusing mainly on:

> "How easily can a developer run this model?"

vLLM focuses heavily on:

> "How efficiently can I serve this model to many requests?"

For example:

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct
```

The architecture could look like:

```text
User 1 ----User 2 -----User 3 ------> vLLM Server ---> GPU ---> Llama Model
User 4 -----/
User 5 ----/
```

vLLM manages many concurrent inference requests efficiently.

---

# Why vLLM Is Useful in Production

Suppose you have a GPU server running a Llama model.

Without good inference scheduling, requests might behave roughly like:

```text
Request A
   |
   v
GPU processing
   |
   v
finish

Request B
   |
   v
GPU processing
```

This wastes potential GPU capacity.

An inference server such as vLLM tries to combine and schedule work efficiently.

Conceptually:

```text
Request A ----Request B -----Request C ------> vLLM Scheduler ---> GPU
Request D -----/
```

The GPU can therefore serve multiple generation workloads much more efficiently.

---

# KV Cache

One major concern when serving LLMs is the **KV cache**.

During token generation, transformers repeatedly use information calculated from previous tokens.

Instead of recalculating everything from scratch for every generated token, the model stores intermediate Key and Value tensors.

This is called the KV cache.

Conceptually:

```text
Prompt:

"What is AWS Lambda?"

Tokens:

What -> is -> AWS -> Lambda -> ?
```

During generation:

```text
Token 1 calculations
        |
        v
     KV Cache

Token 2 calculations
        |
        +---- reuse previous KV values
        |
        v
     KV Cache

Token 3 calculations
        |
        +---- reuse previous KV values
```

This makes generation significantly faster.

The problem is that KV caches can consume a large amount of GPU memory when many users are generating responses at the same time.

---

# PagedAttention in vLLM

One of the technologies vLLM became well known for is **PagedAttention**.

The idea is similar to virtual memory paging in operating systems.

Instead of requiring every request's KV cache to occupy one large contiguous block of GPU memory, vLLM divides KV-cache memory into smaller blocks/pages.

Conceptually:

```text
GPU Memory

+---------+
| Page 1  | -> Request A
+---------+
| Page 2  | -> Request C
+---------+
| Page 3  | -> Request A
+---------+
| Page 4  | -> Request B
+---------+
| Page 5  | -> Request C
+---------+
```

This reduces memory waste and helps vLLM serve more concurrent requests.

---

# Ollama vs vLLM

| Area | Ollama | vLLM |
|---|---|---|
| Primary goal | Easy model execution | High-performance model serving |
| Typical use | Local development | Production inference |
| Setup | Very easy | More infrastructure-oriented |
| Model management | Built in | More limited / external |
| API | Yes | Yes |
| GPU support | Yes | Yes |
| CPU support | Strong local-use support | Primarily optimized for accelerators/GPU workloads |
| High concurrency | Possible, but not its main focus | Major design goal |
| Continuous batching | Not the main abstraction exposed to users | Core capability |
| KV-cache optimization | Runtime-dependent | Major optimization area |
| PagedAttention | No | Yes |
| Model downloading | Very convenient | Usually integrates with model repositories such as Hugging Face |
| Local developer experience | Excellent | More technical |
| Large-scale serving | Not its strongest use case | Excellent |

---

# Example: Local Development with Ollama

Imagine you are developing an AI application locally.

```text
Laptop

+---------------------+
| Python FastAPI App  |
+----------+----------+
           |
           | HTTP
           v
+---------------------+
|       Ollama        |
+----------+----------+
           |
           v
+---------------------+
|     Llama 8B        |
+---------------------+
```

Your backend does not need to understand how to:

- allocate GPU memory
- load GGUF model files
- initialize the inference engine
- manage the model process

It simply calls Ollama.

For example:

```text
POST http://localhost:11434/api/chat
```

This makes Ollama very convenient for development.

---

# Example: Production with vLLM

Now imagine the same application becomes a production system with thousands of users.

```text
                 Internet
                     |
                     v
              Load Balancer
                     |
                     v
              AI Backend API
                     |
                     v
              +-------------+
              |    vLLM     |
              +------+------+
                     |
             +-------+-------+
             |               |
            GPU             GPU
             |
             v
        Llama Model
```

Now the requirements are different.

You care about:

```text
requests/second
tokens/second
GPU utilization
latency
concurrent users
batching
KV-cache memory
model parallelism
```

These are areas where vLLM is particularly useful.

---

# Both Expose APIs

A very important point is that both Ollama and vLLM can sit behind an API.

For Ollama:

```text
Application
    |
    | HTTP
    v
Ollama API
    |
    v
Model
```

For vLLM:

```text
Application
    |
    | HTTP
    v
vLLM API
    |
    v
Model
```

So from the perspective of your backend application, they can look surprisingly similar.

The important difference is what happens **inside the inference server**.

---

# OpenAI-Compatible APIs

vLLM commonly exposes an OpenAI-compatible API.

Conceptually:

```text
POST /v1/chat/completions
```

That means an application written for an OpenAI-style client can often point at a vLLM server by changing the base URL.

The application architecture could therefore be:

```text
Python / C# Application
          |
          | OpenAI-style API
          v
       vLLM
          |
          v
      Llama 3
```

Ollama also provides HTTP APIs and OpenAI-compatible interfaces for many common use cases.

---

# Ollama Is More Than an Inference Engine

This distinction is important.

Ollama is not merely doing matrix multiplication.

It provides a higher-level developer experience around model execution.

Conceptually:

```text
             Ollama
               |
    +----------+----------+
    |          |          |
Model      Model       HTTP
Download   Storage      API
    |          |          |
    +----------+----------+
               |
        Inference Runtime
               |
               v
             Model
```

It therefore combines:

```text
model management
+
runtime configuration
+
inference
+
API serving
```

---

# vLLM Is More Focused on Serving

vLLM's architecture is more focused on inference efficiency.

Conceptually:

```text
                 vLLM
                   |
        +----------+----------+
        |          |          |
    Scheduler   Batching   KV Cache
        |          |          |
        +----------+----------+
                   |
                   v
                  GPU
                   |
                   v
                 Model
```

Its major concern is efficiently turning many incoming requests into GPU workloads.

---

# The Most Important Mental Model

Do not think:

```text
Application
    |
    v
Ollama
    |
    v
vLLM
    |
    v
Model
```

That is generally incorrect.

Instead think:

```text
                 Model
                   |
          +--------+--------+
          |                 |
          v                 v
       Ollama              vLLM
          |                 |
          v                 v
    Local App         Production App
```

They are alternative ways of running/serving models.

---

# Model vs Runtime

This distinction is fundamental when learning LLM infrastructure.

A model might be:

```text
Llama
Mistral
Qwen
Gemma
DeepSeek
```

A runtime/inference server might be:

```text
Ollama
vLLM
llama.cpp
TensorRT-LLM
Hugging Face TGI
```

So when somebody says:

> "We run Llama."

an important infrastructure question is:

> "What runtime are you using to serve it?"

Because:

```text
Llama
 |
 +--> Ollama
 |
 +--> vLLM
 |
 +--> llama.cpp
 |
 +--> TensorRT-LLM
 |
 +--> TGI
```

The model and the software executing the model are different things.

---

# Simple Analogy

Think about a database file and a database engine.

The model weights are somewhat like the stored data.

The inference runtime is the engine that knows how to operate on it.

Conceptually:

```text
Database world:

Data
 |
 v
Database Engine
 |
 v
Application
```

LLM world:

```text
Model Weights
 |
 v
Inference Runtime
 |
 v
Application
```

Ollama and vLLM are two different implementations of that runtime/serving layer, with different priorities.

---

# Summary

The simplest distinction is:

```text
Ollama
    ↓
Easy model management and local inference
    ↓
Excellent developer experience


vLLM
    ↓
High-performance LLM inference
    ↓
Excellent production serving performance
```

Ollama is **not built on top of vLLM**.

Both can:

```text
load model weights
tokenize input
execute transformer inference
generate tokens
expose an API
```

But their emphasis is different:

```text
Ollama → simplicity

vLLM → throughput, concurrency, and GPU efficiency
```

For local AI application development, Ollama is often the easier choice.

For serving a model to many concurrent users on GPUs, vLLM is often the more appropriate architecture.
