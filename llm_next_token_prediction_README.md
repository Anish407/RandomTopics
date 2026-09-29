# How an LLM Predicts the Next Token

This README explains, step by step, how a Large Language Model (LLM) predicts the next token.

It also clarifies the important difference between:

- **Vector similarity in RAG/search**
- **Next-token prediction inside an LLM**

---

## 1. RAG/Search vs LLM Generation

These are related, but they work differently.

### RAG / Semantic Search

In a RAG system, a user question is converted into a vector and compared with vectors stored in a vector database.

```text
Question
   ↓
Embedding model
   ↓
Question vector
   ↓
Compare with document vectors
   ↓
Cosine similarity / Euclidean distance / dot product
   ↓
Retrieve the most relevant documents
```

For example:

```text
Question:
"How can I run containers without managing EC2?"
```

The embedding model might generate:

```text
[0.14, -0.72, 0.81, 0.33, ...]
```

Stored document vectors might look like:

```text
Document 1 → [0.18, -0.68, 0.75, 0.30, ...]
Document 2 → [-0.52, 0.11, 0.09, 0.27, ...]
Document 3 → [0.91, 0.42, -0.63, 0.12, ...]
Document 4 → [0.16, -0.71, 0.83, 0.31, ...]
```

Similarity scores might be:

```text
Question ↔ Document 1 = 0.89
Question ↔ Document 2 = 0.34
Question ↔ Document 3 = 0.07
Question ↔ Document 4 = 0.97
```

The RAG system retrieves Document 4 because its vector is most similar to the question vector.

---

## 2. LLMs Do Not Predict the Next Token by Searching for the Nearest Meaning

An LLM does not usually work like this:

```text
Input vector
   ↓
Find nearest word/vector
   ↓
Return that token
```

Instead, it works more like this:

```text
Input text
   ↓
Tokenization
   ↓
Embedding lookup
   ↓
Transformer layers
   ↓
Final contextual representation
   ↓
Scores for every possible next token
   ↓
Softmax probabilities
   ↓
Select the next token
```

---

# Example: "AWS is a cloud"

Suppose the user types:

```text
AWS is a cloud
```

The model wants to predict what comes next.

A sensible continuation could be:

```text
provider
```

The model arrives at that result through several steps.

---

## 3. Step 1 — Tokenization

The text is split into tokens.

A fake example:

```text
"AWS"    → 81
" is"    → 24
" a"     → 10
" cloud" → 56
```

The model receives:

```text
[81, 24, 10, 56]
```

These numbers are only token IDs.

They do not contain semantic meaning by themselves.

---

## 4. Step 2 — Embedding Lookup

The model contains an embedding matrix.

Each token ID corresponds to one row in that matrix.

Example:

```text
token 81 → [0.8, 0.2, -0.4]
token 24 → [0.1, 0.3,  0.2]
token 10 → [0.0, 0.1,  0.1]
token 56 → [0.7, 0.4, -0.2]
```

So the input becomes a matrix:

```text
[
  [0.8, 0.2, -0.4],
  [0.1, 0.3,  0.2],
  [0.0, 0.1,  0.1],
  [0.7, 0.4, -0.2]
]
```

Each row is a token vector.

The important point is:

> The model does not search the embedding matrix for the nearest meaning.

The tokenizer already produced the token ID, so the model directly retrieves the correct row.

Conceptually:

```text
token ID 81
   ↓
embedding_matrix[81]
   ↓
[0.8, 0.2, -0.4]
```

---

## 5. Step 3 — Transformer Layers Process Context

The initial embedding is only a starting representation.

The transformer layers modify the representation based on the surrounding tokens.

For example:

```text
AWS is a cloud
```

The token `cloud` should be understood in relation to `AWS`.

Compare that with:

```text
A cloud is forming in the sky
```

The initial token embedding for `cloud` may start from the same vector, but after attention and transformer processing, its contextual representation becomes different.

Conceptually:

```text
Initial "cloud" embedding
        ↓
[0.7, 0.4, -0.2]
        ↓
Attention considers "AWS"
        ↓
Other transformer operations
        ↓
More transformer layers
        ↓
Context-aware representation
```

---

## 6. Attention Uses Dot Products

Inside a transformer, each token representation is used to create:

```text
Query
Key
Value
```

Usually written as:

```text
Q = XWq
K = XWk
V = XWv
```

Attention uses a dot product between queries and keys:

```text
Q · K
```

This helps the model determine how strongly different token representations should interact.

For example, in:

```text
AWS is a cloud
```

the representation of `cloud` may strongly attend to `AWS`.

---

## 7. Step 4 — The Model Produces a Final Context Vector

After many transformer layers, suppose the model produces a final vector:

```text
h = [0.9, 0.1, 0.8]
```

This does not literally mean:

```text
"provider"
```

A better mental model is:

> This vector represents the model's internal state for predicting what should come next.

---

## 8. Step 5 — Score Every Possible Next Token

Suppose our fake model only has these possible output tokens:

```text
provider
banana
service
company
dog
```

Each possible output token has learned weights.

Example:

```text
provider = [ 0.8, 0.2,  0.9]
banana   = [-0.7, 0.5, -0.6]
service  = [ 0.7, 0.3,  0.6]
company  = [ 0.5, 0.1,  0.4]
dog      = [-0.3, 0.6, -0.2]
```

The final context vector is:

```text
h = [0.9, 0.1, 0.8]
```

The model computes a dot product between `h` and each possible output token.

### Provider

```text
0.9(0.8) + 0.1(0.2) + 0.8(0.9)
= 0.72 + 0.02 + 0.72
= 1.46
```

### Service

```text
0.9(0.7) + 0.1(0.3) + 0.8(0.6)
= 0.63 + 0.03 + 0.48
= 1.14
```

### Banana

```text
0.9(-0.7) + 0.1(0.5) + 0.8(-0.6)
= -0.63 + 0.05 - 0.48
= -1.06
```

The resulting scores might be:

```text
provider   1.46
service    1.14
company    0.78
dog       -0.37
banana    -1.06
```

These raw scores are called:

```text
logits
```

---

## 9. Why the Dot Product Helps

During training, the model adjusts its weights so that useful contexts produce high scores for appropriate next tokens.

Suppose training data contains many examples like:

```text
AWS is a cloud provider
Azure is a cloud provider
GCP is a cloud provider
```

If the model predicts:

```text
banana
```

instead of:

```text
provider
```

the training process calculates an error and adjusts the model's weights slightly.

After huge amounts of training data, the network learns transformations that make useful next tokens receive higher scores.

---

## 10. Step 6 — Softmax Converts Logits to Probabilities

Logits are not probabilities.

For example:

```text
provider   1.46
service    1.14
company    0.78
dog       -0.37
banana    -1.06
```

Softmax converts them into a probability distribution.

A simplified example:

```text
provider   43%
service    31%
company    20%
dog         4%
banana      2%
```

Conceptually, the model is estimating:

```text
P(next token | previous tokens)
```

or:

```text
Probability of a token being next,
given everything already in the context.
```

---

## 11. Step 7 — Select the Next Token

The model may now select:

```text
provider
```

The text becomes:

```text
AWS is a cloud provider
```

Then the model repeats the process.

It now predicts another token.

For example:

```text
AWS is a cloud provider
                   ↓
                  that
```

Then:

```text
AWS is a cloud provider that
                        ↓
                       offers
```

Then:

```text
AWS is a cloud provider that offers
                               ↓
                              ...
```

LLMs generate text one token at a time.

---

# 12. A Second Example

Suppose the input is:

```text
The capital of Sweden is
```

After processing the context, the model might produce logits like:

```text
Stockholm    12.8
Gothenburg    6.2
Sweden        4.1
Paris        -1.4
banana       -5.7
```

After softmax:

```text
Stockholm   99.7%
Gothenburg   0.2%
Sweden       0.1%
others      ~0%
```

So `Stockholm` will have a very high probability.

But consider this input:

```text
I visited Sweden and then traveled to
```

The model's contextual representation will be different.

It might produce something like:

```text
Norway       4.8
Denmark      4.5
Finland      4.3
Stockholm    3.0
```

So the same word `Sweden` does not always cause the same output.

Context changes the prediction.

---

# 13. Why Matrices Matter

A real model may have a vocabulary with more than 100,000 possible tokens.

The model does not manually compute:

```text
score token 1
score token 2
score token 3
...
score token 100000
```

Instead, it stores output weights in a large matrix.

Conceptually:

```text
W = [
  token_1_weights
  token_2_weights
  token_3_weights
  ...
  token_100000_weights
]
```

Then it performs a matrix multiplication:

```text
logits = h × Wᵀ
```

This produces all token scores efficiently.

The result might look like:

```text
[1.46, 1.14, 0.78, -0.37, -1.06, ...]
```

This is one of the main reasons matrix math is so important in AI.

GPUs are extremely good at performing these matrix operations in parallel.

---

# 14. Important: The Model Does Not Always Pick the Highest Probability

Suppose softmax produces:

```text
provider    40%
service     30%
platform    20%
company     10%
```

The model does not necessarily always choose `provider`.

Text-generation systems can sample from several likely tokens.

Settings such as these affect the choice:

```text
temperature
top-k
top-p
```

In general:

```text
Lower temperature
→ more deterministic and predictable

Higher temperature
→ more varied and random
```

---

# 15. Full Next-Token Prediction Pipeline

```text
User:
"AWS is a cloud"

        ↓

Tokenizer

        ↓

[81, 24, 10, 56]

        ↓

Embedding lookup

        ↓

Token vectors

        ↓

Transformer layer 1
        ↓
Transformer layer 2
        ↓
Transformer layer 3
        ↓
...
Transformer layer N

        ↓

Final context vector h

        ↓

h × output weight matrix

        ↓

Logits

provider  1.46
service   1.14
company   0.78
banana   -1.06

        ↓

Softmax

        ↓

Probabilities

provider  43%
service   31%
company   20%
banana     2%

        ↓

Select one token

        ↓

"provider"

        ↓

New context:

"AWS is a cloud provider"

        ↓

Predict the next token again
```

---

# 16. The Key Difference to Remember

## RAG / Vector Search

```text
Question
   ↓
Embedding vector
   ↓
Compare with stored vectors
   ↓
Cosine similarity / Euclidean distance / dot product
   ↓
Find semantically relevant data
```

## LLM Generation

```text
Input tokens
   ↓
Embedding lookup
   ↓
Transformer layers
   ↓
Contextual representation
   ↓
Matrix multiplication with output weights
   ↓
Logits
   ↓
Softmax probabilities
   ↓
Next token
```

---

# 17. Core Mental Model

The most important idea is:

> An LLM does not search for the nearest word. It repeatedly transforms vectors through learned matrices. At the end, it uses the resulting context vector to calculate a score for every possible next token.

That is the connection between:

- vectors
- matrices
- dot products
- attention
- logits
- softmax
- next-token prediction

These concepts form the mathematical foundation for understanding modern neural networks and transformer-based LLMs.
