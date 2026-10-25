# Transformer Fundamentals — Embeddings, Training, Matrix Shapes, and Context Windows

> A practical revision guide based on our discussion while watching 3Blue1Brown's **Transformers (GPT), the tech behind LLMs**: https://www.youtube.com/watch?v=wjZofJX0v4M
>
> Audience: a backend developer learning the internals of Transformer language models. Examples are deliberately small and illustrative.

## Contents

1. [The full picture](#1-the-full-picture)
2. [Tokens, token IDs, and the vocabulary](#2-tokens-token-ids-and-the-vocabulary)
3. [What is an embedding matrix?](#3-what-is-an-embedding-matrix)
4. [Is the embedding matrix created or adjusted during training?](#4-is-the-embedding-matrix-created-or-adjusted-during-training)
5. [Why does a token have multiple embedding dimensions?](#5-why-does-a-token-have-multiple-embedding-dimensions)
6. [How gradient descent updates an embedding](#6-how-gradient-descent-updates-an-embedding)
7. [Does learning a new topic add a new dimension?](#7-does-learning-a-new-topic-add-a-new-dimension)
8. [Understanding the mathematics notation](#8-understanding-the-mathematics-notation)
9. [Embedding matrix versus input embedding matrix](#9-embedding-matrix-versus-input-embedding-matrix)
10. [Context window versus actual token count](#10-context-window-versus-actual-token-count)
11. [Why self-attention needs a context](#11-why-self-attention-needs-a-context)
12. [How Query, Key, and Value matrix shapes work](#12-how-query-key-and-value-matrix-shapes-work)
13. [Next-token prediction and generation](#13-next-token-prediction-and-generation)
14. [Context limits in real systems, including RAG](#14-context-limits-in-real-systems-including-rag)
15. [Training, fine-tuning, and inference](#15-training-fine-tuning-and-inference)
16. [Hands-on NumPy exercises](#16-hands-on-numpy-exercises)
17. [Common misconceptions](#17-common-misconceptions)
18. [Cheat sheet](#18-cheat-sheet)
19. [Self-test](#19-self-test)

---

## 1. The full picture

A decoder-only Transformer roughly follows this sequence:

```text
Input text: "The capital of France is"
         |
         v
Tokenizer -> token IDs
         |
         v
Look up each token ID in the learned embedding matrix
         |
         v
Token vectors + positional information
         |
         v
Transformer blocks: causal self-attention + feed-forward layers
         |
         v
Final hidden vector at the last input position
         |
         v
Output projection -> logits for vocabulary tokens
         |
         v
Softmax / sampling -> next token (e.g., " Paris")
         |
         v
Append token and repeat
```

**Important separation:** training adjusts model parameters; inference uses the trained parameters. The current prompt/conversation is *context*, not new permanent training.

## 2. Tokens, token IDs, and the vocabulary

A **tokenizer** converts text into token IDs. A token is not necessarily a complete word: it may be a word, punctuation, a subword, or part of whitespace. The exact splitting depends on the tokenizer.

Hypothetical vocabulary:

| Text token | Token ID |
|---|---:|
| `apple` | 10 |
| `banana` | 11 |
| `car` | 12 |

The IDs are identifiers, **not** semantic magnitudes. ID 11 is not intrinsically closer in meaning to ID 10 than ID 12 is. Embeddings provide trainable numerical representations associated with token IDs.

## 3. What is an embedding matrix?

Suppose:

- Vocabulary size `V = 10,000`
- Embedding/model dimension `d_model = 4`

Then the token embedding table has shape:

```text
E.shape = (10_000, 4)
```

Mathematically: **E ∈ ℝ^(10000×4)**.

| Token | Dim 1 | Dim 2 | Dim 3 | Dim 4 |
|---|---:|---:|---:|---:|
| apple | 0.20 | -0.10 | 0.30 | 0.50 |
| banana | 0.10 | -0.20 | 0.40 | 0.60 |
| car | 0.80 | 0.30 | -0.10 | 0.20 |
| python | -0.30 | 0.50 | 0.20 | 0.10 |

Only four of 10,000 rows are shown; values are made up. Looking up an ID retrieves **a row containing four values**.

```python
vector = E[token_id]  # shape: (4,)
```

The vocabulary size sets the number of **rows**; embedding dimension sets the number of **columns**.

## 4. Is the embedding matrix created or adjusted during training?

**Both, in order:**

1. When a model is initialized, the architecture allocates its embedding parameters, normally initialized with random small numbers according to an initialization strategy.
2. During pretraining, backpropagation computes gradients and an optimizer adjusts the trainable embedding values along with other weights.
3. When you load a pretrained model, those learned embeddings are already part of its weights.
4. Normal inference uses the learned embeddings without updating them.
5. Fine-tuning may update embeddings **if they are trainable**; if frozen, only other trainable parameters change.

The tokenizer determines token IDs; it is not the same object as the trainable embedding matrix.

## 5. Why does a token have multiple embedding dimensions?

One number (the token ID) gives the network no useful continuous representation of meaning or usage. A vector provides multiple coordinates on which neural network computations can operate.

For **intuition only**, imagine a hand-designed 3D space:

| Token | Fruit-related | Transport-related | Technology-related |
|---|---:|---:|---:|
| apple | 0.9 | 0.0 | 0.1 |
| banana | 0.8 | 0.0 | 0.0 |
| car | 0.0 | 0.9 | 0.5 |

These labels are **not how real LLM embedding columns work**. The model does not reserve column 1 for fruit or column 3 for technology. In a trained neural network, individual coordinates generally lack simple human-readable meanings. A feature can be distributed across many dimensions, and each coordinate can contribute to many features.

A token's *initial* embedding is not the whole contextual meaning. For example, a token relating to **Apple the company** versus **apple the fruit** can receive different context-dependent hidden representations after attention layers process the surrounding tokens. Exact token IDs also depend on case and tokenizer behavior.

## 6. How gradient descent updates an embedding

Suppose a token embedding starts as:

```text
old embedding = [0.20, 0.40, 0.60]
```

Training on text calculates a loss (typically based on probabilities assigned to the actual next tokens). Backpropagation calculates how sensitive that loss is to each trainable value. For this example, assume:

```text
gradient = [0.10, -0.20, 0.30]
learning rate = 0.1
```

Simplified SGD update:

```text
new_embedding = old_embedding - learning_rate * gradient
```

Calculate dimension by dimension:

| Dimension | Old | Gradient | Update | New |
|---|---:|---:|---:|---:|
| 1 | 0.20 | 0.10 | -0.01 | 0.19 |
| 2 | 0.40 | -0.20 | +0.02 | 0.42 |
| 3 | 0.60 | 0.30 | -0.03 | 0.57 |

Result: `[0.19, 0.42, 0.57]`. **Three dimensions before; three after.**

Mathematically:

**E_new = E_old − η ∂L/∂E**

- `E`: embedding parameters
- `η` (eta): learning rate
- `L`: loss
- `∂L/∂E`: gradient of loss with respect to embedding parameters

In actual training, gradients are calculated from batches and networks with many layers; optimizers such as AdamW use more elaborate update rules. This toy gradient is supplied manually to demonstrate the arithmetic.

Not every embedding row necessarily receives a direct input-lookup gradient in every batch; details also depend on whether input/output weights are tied and on the training architecture.

## 7. Does learning a new topic add a new dimension?

**No.** Learning about medical terminology or quantum computing generally changes values in existing trainable parameters (embeddings, attention projections, feed-forward layers, etc.). It does **not** add a column called `medicine` or `quantum physics`.

For a fixed model architecture:

```text
Before: E.shape = (10000, 4)
After more training: E.shape = (10000, 4)
```

Different situation: adding a **new vocabulary token** can require adding a **row** to the embedding table, not a column:

```text
Before adding token: E.shape = (10000, 4)
After adding one token: E.shape = (10001, 4)
```

The tokenizer must also be updated, and the vocabulary-facing output parameters may need resizing. Unfamiliar *words* often can already be represented by existing subword tokens, so adding vocabulary entries is not usually required just to discuss a new topic.

**Knowledge is distributed**: factual and conceptual patterns are not necessarily saved in a single token embedding or a single dimension. Transformer blocks also learn important parameters.

## 8. Understanding the mathematics notation

The expression from our discussion:

**W_Q ∈ ℝ^(768×768)**

Read as: “W sub Q is a real-valued matrix with 768 rows and 768 columns.”

| Symbol | Meaning |
|---|---|
| `W` | Weight matrix |
| subscript `Q` | Used for the *Query* projection in attention |
| `∈` | “Is an element of” / “belongs to” |
| `ℝ` | Real numbers |
| `ℝ^3` | Three-dimensional real vector space |
| `ℝ^(m×n)` | Set of `m`-row, `n`-column real matrices |
| `×` in a shape | Rows by columns |
| `Wᵀ` | Transpose (swap rows and columns) |
| `Σ` | Summation |
| `∇` | Gradient |
| `∂` | Partial derivative |

A `768 × 768` weight matrix contains `768*768 = 589,824` numerical weights.

In NumPy:

```python
W_Q.shape == (768, 768)
```

**Note:** `∈ ℝ^(768×768)` states a matrix's *shape and number domain*, not its values. In real software, values are stored in finite-precision types such as float32, bfloat16, etc.

## 9. Embedding matrix versus input embedding matrix

These are different things:

**Learned vocabulary embedding table:**

```text
E.shape = (10000, 4)
```

**Embedding sequence produced for four input tokens:**

```text
Tokens: "I" | "love" | "machine" | "learning"
X.shape = (4, 4)
```

`X` contains *four selected rows* of `E`, in sequence order (with repeats if the same token occurs repeatedly). Position information may be added or introduced through attention, depending on the architecture.

The general rule for an unbatched embedding lookup:

```text
X.shape = (actual_number_of_input_tokens, embedding_dimension)
```

For a batch, the usual shape is `(batch_size, sequence_length, embedding_dimension)`.

## 10. Context window versus actual token count

**Context window = maximum supported number of token positions.**

**Actual token count = number of tokens in this input.**

Example model:

```text
Vocabulary = 10,000
Embedding dimension = 4
Context window = 16
```

| Actual input tokens | Lookup result shape | Is it within the 16-token window? |
|---:|---|---|
| 4 | `(4, 4)` | Yes |
| 8 | `(8, 4)` | Yes |
| 12 | `(12, 4)` | Yes |
| 16 | `(16, 4)` | Yes |
| 20 | `(20, 4)` in a naive lookup, but sequence exceeds model's supported window | No |

**This corrects an easy misunderstanding:** increasing the *supported context* from 8 to 16 **does not itself make an eight-token input into a `16 × 4` matrix**. The shape becomes `16 × 4` only when you actually supply 16 token positions, or an implementation pads a shorter sequence to 16.

Therefore:

```text
8 tokens, d_model=4  => X.shape=(8,4)
16 tokens, d_model=4 => X.shape=(16,4)
```

A model does **not** always process its full maximum window. Each model/configuration has a supported context length, and runtime software (e.g., Ollama) can impose a lower configured limit. Position encodings and long-context training also affect reliable use of long sequences. A large context does not automatically mean better reasoning.

## 11. Why self-attention needs a context

Consider:

```text
"The cat sat on the mat"
```

Each input token starts with a token embedding (plus appropriate positional information). But a standalone token embedding does not tell the model exactly how `sat`, `cat`, and `mat` relate **in this sentence**.

Self-attention lets positions incorporate information from other permitted positions. In a GPT-like *causal* Transformer, a position may attend to itself and earlier positions, but not future positions.

For six tokens, causal access looks like this (`✓` = permitted, `—` = future blocked):

```text
               Key positions ->
                1  2  3  4  5  6
Query token 1:  ✓  —  —  —  —  —
Query token 2:  ✓  ✓  —  —  —  —
Query token 3:  ✓  ✓  ✓  —  —  —
Query token 4:  ✓  ✓  ✓  ✓  —  —
Query token 5:  ✓  ✓  ✓  ✓  ✓  —
Query token 6:  ✓  ✓  ✓  ✓  ✓  ✓
```

The symbols represent *allowed comparisons*, not the actual attention strengths.

## 12. How Query, Key, and Value matrix shapes work

Let:

```text
number of tokens n = 8
model width d_model = 4
one attention head with d_k = d_v = 4
X.shape = (8,4)
```

Then (ignoring optional biases):

```text
Q = X @ W_Q      shape: (8,4)
K = X @ W_K      shape: (8,4)
V = X @ W_V      shape: (8,4)

scores = Q @ K.T shape: (8,8)
```

Why `(8,8)`? Eight query positions are compared with eight key positions. The causal mask prevents future-position interactions. Softmax produces normalized attention weights and those weights are multiplied by `V`:

```text
attention_weights.shape = (8,8)
attention_output.shape  = (8,4)
```

A standard scaled dot-product attention expression is:

```text
Attention(Q,K,V) = softmax((Q @ K.T)/sqrt(d_k) + causal_mask) @ V
```

In real multi-head architectures the head dimensions and projection shapes can differ; `768×768` was one *illustrative* choice, not an invariant of every Transformer.

**Important distinction:** the attention score matrix (`n×n`) is *computed for the input*, not a fixed trainable embedding matrix. The projection matrices `W_Q`, `W_K`, and `W_V` are learned weights.

## 13. Next-token prediction and generation

Suppose the tokenizer splits:

```text
"The capital of France is"
```

into five tokens (illustrative only). With model width 768:

```text
Input embedding sequence: (5, 768)
Transformer output:        (5, 768)
Last-position hidden state:(768,)
Output vocabulary logits:  (vocabulary_size,)
```

The last-position representation is mapped to scores (logits) over possible next tokens. Those scores define probabilities after softmax; a token is selected or sampled, appended, and the model repeats. During training, multiple next-token losses can be calculated in parallel at different sequence positions using causal masking.

The Transformer does not search the embedding matrix for the nearest token as its whole prediction mechanism. Learned attention, feed-forward transformations, and an output projection produce the next-token scores.

## 14. Context limits in real systems, including RAG

Imagine a model with a maximum context of **8,192 tokens** and a RAG prompt with:

| Input component | Token count |
|---|---:|
| System instructions | 500 |
| Question | 100 |
| Retrieved document 1 | 3,000 |
| Retrieved document 2 | 4,000 |
| Retrieved document 3 | 3,000 |
| **Total input** | **10,600** |

`10,600 > 8,192`: the complete prompt cannot fit. The application needs to reduce/rank/filter chunks, summarize, split work, or choose a longer-context model. It must also budget output tokens where the runtime/API has a combined context limit.

A window does not grow simply because the application has more documents. **Vector database storage** is external retrieval storage; **context** is the tokens provided to the model for this pass; **weights** are parameters learned during training.

### Attention cost

Full self-attention computes pairwise attention scores over `n` positions, conceptually `n × n` scores per head:

| n | n² score entries |
|---:|---:|
| 512 | 262,144 |
| 1,024 | 1,048,576 |
| 2,048 | 4,194,304 |
| 4,096 | 16,777,216 |

Doubling `n` quadruples the number of score pairs, but **does not necessarily quadruple total inference time or memory**: optimized attention kernels need not materialize all scores, and autoregressive generation may use a **KV cache** to reuse past Keys and Values.

### If the context limit is exceeded

Behavior depends on the application and runtime: reject the request, truncate history, use a sliding-window strategy, or summarize selected older content. These mechanisms are not all intrinsic to every Transformer.

## 15. Training, fine-tuning, and inference

| Phase | Embedding table | Other model weights | Context tokens |
|---|---|---|---|
| Initial model construction | Initialized | Initialized | None yet |
| Pretraining | Usually updated | Updated | Training batches |
| Full fine-tuning | Updated if unfrozen | Updated if trainable | Fine-tuning batches |
| Parameter-efficient fine-tuning | Often frozen | Selected adapter/other parameters updated | Fine-tuning batches |
| Normal inference | Not updated | Not updated | Current prompt and generated tokens |

**Adding a new topic does not expand the number of embedding dimensions.** If vocabulary is explicitly extended, embedding rows and possibly output parameters may need resizing.

## 16. Hands-on NumPy exercises

### Exercise A — Token ID lookup vs input sequence shape

```python
import numpy as np

rng = np.random.default_rng(42)
V, d_model = 10, 4
E = rng.normal(size=(V, d_model))

token_ids = np.array([2, 4, 1, 2])  # four tokens; first repeats
X = E[token_ids]

print('Embedding table:', E.shape)    # (10, 4)
print('Input embeddings:', X.shape)   # (4, 4)
print('Repeated tokens match:', np.allclose(X[0], X[3]))  # True
```

### Exercise B — Manually update a token embedding (toy SGD)

```python
import numpy as np

embedding = np.array([0.20, 0.40, 0.60])
gradient = np.array([0.10, -0.20, 0.30])
learning_rate = 0.1

embedding -= learning_rate * gradient
print(embedding)        # [0.19 0.42 0.57]
print(embedding.shape)  # (3,)
```

This is one hand-supplied update, not automatic gradient calculation.

### Exercise C — Matrix multiplication and Q/K/V shapes

```python
import numpy as np

rng = np.random.default_rng(7)
n_tokens, d_model = 8, 4
X = rng.normal(size=(n_tokens, d_model))
W_Q = rng.normal(size=(d_model, d_model))
W_K = rng.normal(size=(d_model, d_model))
W_V = rng.normal(size=(d_model, d_model))

Q = X @ W_Q
K = X @ W_K
V = X @ W_V
scores = Q @ K.T / np.sqrt(d_model)

# Prevent each token position from attending to future positions.
future_positions = np.triu(np.ones((n_tokens, n_tokens), dtype=bool), k=1)
masked_scores = np.where(future_positions, -np.inf, scores)

# Row-wise stable softmax.
exps = np.exp(masked_scores - np.max(masked_scores, axis=-1, keepdims=True))
weights = exps / exps.sum(axis=-1, keepdims=True)
output = weights @ V

print('Q:', Q.shape)             # (8, 4)
print('scores:', scores.shape)   # (8, 8)
print('weights:', weights.shape) # (8, 8)
print('output:', output.shape)   # (8, 4)
print('First position attention:', weights[0])  # only first entry nonzero
```

### Exercise D — Context size is a ceiling, not a row count

```python
context_limit = 16
d_model = 4
for actual_tokens in [4, 8, 12, 16]:
    X = np.zeros((actual_tokens, d_model))
    assert actual_tokens <= context_limit
    print(actual_tokens, 'tokens =>', X.shape)

# 4 tokens => (4, 4)
# 8 tokens => (8, 4)
# 12 tokens => (12, 4)
# 16 tokens => (16, 4)
```

## 17. Common misconceptions

| Misconception | Correction |
|---|---|
| Every topic creates a new embedding dimension. | The existing numerical weights adapt; no topic-specific column is automatically created. |
| A vocabulary token ID measures word meaning. | IDs are identifiers; learned embeddings carry numerical representations. |
| Embedding dimension 1 means “fruit.” | Real learned dimensions generally have no fixed human-readable topic labels. |
| Adding a new word always adds a token. | Most tokenizers can represent unfamiliar words via existing subwords. |
| The embedding table stores all model knowledge. | Knowledge is distributed throughout many model parameters. |
| Context size equals the number of input rows. | Context size is the **maximum**; input row count usually equals **actual tokens**. |
| A model with a 16-token window always makes a `(16,4)` input matrix. | Four actual tokens normally make a `(4,4)` lookup result; padding is implementation-dependent. |
| Context window means permanent memory. | Context is temporary available input; trained weights and external stores are different. |
| A longer window always makes a model smarter. | It enables longer input, but quality of long-context use varies. |
| Attention scores are static learned weights. | Attention scores are computed per input; projection weights are learned. |

## 18. Cheat sheet

| Concept | Shape / rule | What to remember |
|---|---|---|
| Embedding table | `(vocab_size, d_model)` | Learned per-token-ID vectors |
| Input embeddings | `(actual_tokens, d_model)` | Actual tokens, not maximum context |
| Batched embeddings | `(batch, tokens, d_model)` | One sequence per batch member |
| Maximum context | `tokens ≤ max_context` | Model/runtime-supported upper limit |
| Query projection | `Q = X @ W_Q` | Learns a representation for matching |
| Key projection | `K = X @ W_K` | Used to compare positions |
| Value projection | `V = X @ W_V` | Information mixed by attention weights |
| Score matrix | `Q @ K.T`, shape `(n,n)` in self-attention | Pairwise token-position comparisons |
| Attention output | `softmax(masked scores) @ V` | Context-mixed representations |
| Next-token prediction | Last hidden state -> vocabulary logits | Not nearest-neighbor lookup of embeddings |
| Toy gradient update | `w_new = w_old - lr * grad` | Values change, tensor shape usually doesn't |

### One-line takeaway

**The embedding table determines the vector width per token; the input token count determines how many rows are looked up; the context window limits how many positions can be processed; and training changes learned numerical values without automatically changing those shapes.**

## 19. Self-test

1. A model has `V=50,000`, `d_model=768`, and `max_context=1,024`. What is the shape of its token embedding table?
2. If you send exactly 500 tokens, what is the unbatched embedding lookup shape?
3. If you send 1,500 tokens, does that fit the supported 1,024-token window?
4. If the model learns medical concepts, does `d_model` automatically increase to 769?
5. What is different between a *new topic* and a *new vocabulary token*?
6. If `X.shape=(8,4)` and `W_Q.shape=(4,4)`, what is `Q.shape`?
7. If `Q.shape=(8,4)` and `K.shape=(8,4)`, what is `(Q @ K.T).shape`?
8. Is the attention score matrix itself the trainable embedding table?
9. Does normal inference update the pretrained embedding values?
10. Why might a RAG app retrieve useful information but still fail to fit it into the model prompt?

<details>
<summary>Answers</summary>

1. `(50000, 768)`.
2. `(500, 768)` (before adding any optional batch axis).
3. No; the application/runtime must handle the excess.
4. No. The width remains fixed absent a deliberate architecture change.
5. A topic changes existing parameters when trained; adding a vocabulary token can add a row and may require output-vocabulary changes.
6. `(8, 4)`.
7. `(8, 8)`.
8. No; it is computed dynamically from Q and K.
9. No, not in normal inference.
10. Retrieved documents consume prompt tokens; the combined input and output budget may exceed the model's supported context window.

</details>

---

### Video and next learning steps

- [3Blue1Brown — Transformers (GPT), the tech behind LLMs](https://www.youtube.com/watch?v=wjZofJX0v4M)
- [3Blue1Brown — Attention in Transformers, Visually Explained](https://www.youtube.com/watch?v=eMlx5fFNoYc)
- Next concepts to study: dot product -> matrix multiplication -> Q/K/V -> causal mask -> softmax -> multi-head attention -> feed-forward layers -> backpropagation -> KV caching.

*This file consolidates our explanations and adds a few clarifying implementation details; the numerical training examples are educational, not measurements from the video or a real pretrained model.*
