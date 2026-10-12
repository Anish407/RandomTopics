# How Tokenization, Embeddings, and LLMs Work

A practical guide to how text becomes token IDs, how those IDs become vectors, and how a language model learns to predict its next token.

## 1. The big picture

When you type a question such as **“What is an apple?”**, a generative LLM processes it roughly as follows:

```text
Input text: "What is an apple?"
             |
             v
Tokenizer (e.g., tiktoken or a Hugging Face tokenizer)
             |
             v
Token IDs: [illustrative IDs]
             |
             v
Model's learned token embedding matrix
             |
             v
One vector per token (+ positional information)
             |
             v
Transformer layers (attention + learned transformations)
             |
             v
Logits over vocabulary -> token selection
             |
             v
Next token ID -> decoded text
```

**Key distinction:** The **tokenizer** chooses token IDs using a fixed vocabulary and encoding rules. The model's **embedding matrix** uses each ID as a row index to retrieve a learned vector. The tokenizer **does not search the embedding matrix** to discover the token ID.

## 2. What is a tokenizer?

A tokenizer breaks input text into tokens and converts each token to an integer ID. A token might be a whole word, a part of a word, punctuation, or whitespace combined with text. The exact splitting and IDs depend on the tokenizer.

For instance, a hypothetical encoding could give:

| Text piece | Illustrative token ID |
|---|---:|
| `What` | 2061 |
| ` is` | 374 |
| ` an` | 459 |
| ` apple` | 24149 |
| `?` | 30 |

These numbers are **illustrative**, not guaranteed outputs for any particular encoding. To see actual IDs, run the code below with a named tokenizer.

### `tiktoken` example

```bash
pip install tiktoken
```

```python
import tiktoken

encoding = tiktoken.get_encoding("cl100k_base")
text = "What is an apple?"
ids = encoding.encode(text)

print("Token IDs:", ids)
print("Token count:", len(ids))
print("Decoded:", encoding.decode(ids))

for token_id in ids:
    print(token_id, repr(encoding.decode_single_token_bytes(token_id)))
```

`tiktoken` implements encodings used by OpenAI models. Different OpenAI models can use different encodings; for a supported model you can use `tiktoken.encoding_for_model(model_name)`.

**Important:** `tiktoken` is *not* a client for invoking models and *not* an inference engine. It only tokenizes/decodes data.

## 3. What is the embedding matrix?

Suppose a model has a vocabulary of **50,000 tokens** and an embedding width of **768 numbers**. Its learned token embedding matrix has shape:

\[
E \in \mathbb{R}^{50000 \times 768}
\]

Each row corresponds to one token ID; the row contains that token's initial learned vector. In a tiny made-up example:

| Token ID | Token | Learned vector (abbreviated) |
|---:|---|---|
| 100 | `apple` | `[0.21, -0.34, 0.76, ...]` |
| 101 | `orange` | `[0.19, -0.29, 0.71, ...]` |
| 102 | `car` | `[-0.52, 0.61, 0.13, ...]` |

These values are illustrative. In reality, embeddings are *learned during training*, not hand-coded. Individual coordinates rarely have a simple human-readable label such as “fruit.”

A token ID acts like an array index:

```python
import numpy as np

vocab = {"apple": 0, "orange": 1, "car": 2}

# Toy matrix: three tokens, three dimensions (not real model weights)
embedding_matrix = np.array([
    [ 0.21, -0.34,  0.76],  # apple
    [ 0.19, -0.29,  0.71],  # orange
    [-0.52,  0.61,  0.13],  # car
])

token_id = vocab["apple"]
vector = embedding_matrix[token_id]

print("Token ID:", token_id)
print("Embedding:", vector)
```

Expected output:

```text
Token ID: 0
Embedding: [ 0.21 -0.34  0.76]
```

In a neural-network implementation, an embedding layer efficiently performs this lookup for all token IDs in the input.

## 4. Apple the fruit vs. Apple the iPhone company: how context changes meaning

**Your intuition is right, with an important correction about the order of operations:** the tokenizer produces token IDs first; the model looks up their initial embeddings **before** the transformer uses attention. The transformer does **not** decide which embedding row to retrieve by interpreting the word's meaning.

Consider two sentences (lowercase *apple* is deliberate here):

1. **"I ate an apple."** — Here, *apple* is a fruit.
2. **"I bought an apple phone."** — Here, *apple* refers to the company that makes the iPhone.

If both sentences contain the **exact same token** for ` apple` (including its leading space), the tokenizer assigns it the **same token ID** in both sentences. The model retrieves the **same initial embedding vector** from the same row of its embedding matrix. Other tokens around it are different, so the context is different.

**Important nuance:** Tokenizers often distinguish capitalization, leading spaces, and different text fragments. `apple` and `Apple` are **not guaranteed** to have the same ID; a word may also split into multiple tokens. The same-ID claim applies to an **identical token in the same encoding**, not to every spelling or occurrence of a word.

### The precise sequence

```text
"I ate an apple."                "I bought an apple phone."
         |                                  |
    Tokenizer                            Tokenizer
         |                                  |
 Token IDs, including                Token IDs, including
   ID for " apple"                     ID for " apple"
         |                                  |
         +----------- same ID -------------+
                         |
                    Embedding lookup
                         |
              Same INITIAL vector for
                 this token ID
                         |
             Transformer layers use
             attention + other learned
             transformations WITH CONTEXT
                       /   \
                      v     v
                Fruit-related   Company-related
                  contextual      contextual
                representation  representation
```

This diagram is conceptual: each **full sentence** goes through the model independently. Its contextual representation for the ` apple` token can differ between the sentences, even when the initial embedding is identical.

### What attention actually does

Self-attention allows each token's representation (subject to the model's attention mask) to draw on relevant information from other tokens. In a causal language model, a position attends to **earlier tokens and itself**, not future tokens. Other learned transformations, including feed-forward layers, also contribute to the changing representations.

The transformer does **not** normally choose between a permanent "fruit embedding" and "company embedding" stored as separate rows for one identical token ID. It progressively computes **contextual hidden-state vectors** that reflect the text around a token. Meaning is also distributed throughout the model's learned weights, not exclusively in the input embedding matrix.

### Try it with `tiktoken`

```python
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")

for sentence in ["I ate an apple.", "I bought an apple phone."]:
    ids = enc.encode(sentence)
    pieces = [enc.decode_single_token_bytes(i) for i in ids]
    print(sentence)
    print(list(zip(ids, pieces)))

# Demonstrate that capitalization can produce different IDs:
print("apple:", enc.encode(" apple"))
print("Apple:", enc.encode(" Apple"))
```

Inspect the decoded pieces and IDs to confirm the actual boundaries in the chosen encoding. This script shows **tokenization only**; it does not run a transformer or compute contextual embeddings.

**In one sentence:** Same token ID → same initial embedding row; different sentence context → potentially different contextual representation after transformer processing.

## 5. How does training create the numbers?

A simplified next-token training loop:

1. **Tokenize** training text into token IDs.
2. **Look up embeddings** for those IDs.
3. **Run the transformer** to produce predictions for the next tokens.
4. **Calculate loss** by comparing predictions with actual next-token targets.
5. **Backpropagate gradients** through the network.
6. **Update weights**, including token embeddings (when trainable), using an optimizer.
7. Repeat across many sequences.

Over training, the parameters learn statistical and semantic patterns useful for token prediction. There is no separate lookup table that declares `apple = fruit` and stores the whole model's knowledge.

## 6. Where do next-token probabilities come from?

After transformer processing, the model produces **logits** (scores) for candidate tokens in its vocabulary. Applying **softmax** converts those logits to a probability distribution:

\[
P(\text{next token}=i) = \frac{e^{z_i}}{\sum_j e^{z_j}}
\]

A decoding strategy (greedy selection, sampling, etc.) chooses a token. Its ID is appended to the sequence, and the process repeats until stopping.

This is different from searching for the closest word embedding. Standard generative LLM decoding uses output logits and a token-selection strategy rather than simply selecting the nearest embedding vector.

## 7. `tiktoken` vs. Hugging Face vs. Ollama

| Tool | What it does | Typical use |
|---|---|---|
| **`tiktoken`** | Encodes/decodes text with supported token encodings | Inspect or count tokens for compatible OpenAI models |
| **Hugging Face `AutoTokenizer`** | Loads the tokenizer associated with a specific model | Tokenize inputs for local Transformers models |
| **Hugging Face `AutoModelForCausalLM`** | Loads a model and its trained weights | Run next-token generation in Python |
| **Ollama** | Runs models behind a local API and handles tokenization internally | Build local model-backed applications |

**Never assume token IDs are interchangeable between unrelated model families.** A model must receive IDs from the vocabulary/encoding it was trained to use.

### Hugging Face: matched tokenizer and model

```bash
pip install torch transformers
```

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

model_id = "openai-community/gpt2"  # small demonstration model

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id)

inputs = tokenizer("Machine learning is", return_tensors="pt")
print("Input IDs:", inputs["input_ids"])

outputs = model.generate(
    **inputs,
    max_new_tokens=20,
    do_sample=False,
    pad_token_id=tokenizer.eos_token_id,
)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

For **chat/instruct models**, use their proper **chat templates** (`tokenizer.apply_chat_template(...)`) rather than assuming that a raw user string is the correct prompt format.

### Ollama: Python calls the server, not the tokenizer

With Ollama installed, first pull and serve a model (the local service must be running):

```bash
ollama pull llama3.2
```

```python
import requests

response = requests.post(
    "http://localhost:11434/api/generate",
    json={
        "model": "llama3.2",
        "prompt": "Explain machine learning",
        "stream": False,
    },
    timeout=120,
)
response.raise_for_status()
data = response.json()

print(data["response"])
print("Prompt tokens:", data.get("prompt_eval_count"))
print("Generated tokens:", data.get("eval_count"))
```

Ollama tokenizes the text internally according to the model. You generally do not send arbitrary `tiktoken` IDs to its text-generation API.

## 8. Can `tiktoken` token IDs go directly into a local model?

**Only when the token ID mapping and special-token conventions are compatible with that model.** GPT-2 is a useful educational example: `tiktoken`'s `gpt2` encoding matches the GPT-2 vocabulary/ID mapping for ordinary text.

```python
import torch
import tiktoken
from transformers import AutoModelForCausalLM

encoding = tiktoken.get_encoding("gpt2")
model = AutoModelForCausalLM.from_pretrained("openai-community/gpt2")
model.eval()

prompt = "Artificial intelligence is"
ids = encoding.encode(prompt)
input_ids = torch.tensor([ids], dtype=torch.long)

with torch.no_grad():
    generated = model.generate(
        input_ids,
        max_new_tokens=20,
        do_sample=False,
        pad_token_id=model.config.eos_token_id,
    )

print(encoding.decode(generated[0].tolist()))
```

Do **not** swap in `cl100k_base` or `o200k_base` for a GPT-2 model. Likewise, do not assume that those encodings work for Llama or Qwen.

## 9. Token embeddings vs. RAG embeddings

These concepts are related but not the same:

- **Generative model's token embeddings:** Map individual token IDs to learned starting vectors, used inside a language model.
- **RAG / retrieval embeddings:** An embedding model converts a text passage or query into a vector designed for comparing semantic similarity between *whole pieces of text*.

A retrieval system may compare vectors using cosine similarity or dot product. A generative LLM instead processes a sequence of token embeddings with transformer layers and predicts output tokens from logits. These are different stages and different use cases.

## 10. Key takeaways

1. The tokenizer maps **text → token IDs** with its own vocabulary and encoding rules.
2. The model uses **token IDs → embedding matrix rows → vectors**.
3. The embedding vectors and many other matrices are **learned during training**.
4. Transformer layers develop **context-sensitive representations**, so a word can mean different things in different sentences.
5. A generative model predicts next-token **logits**, from which tokens are chosen; it does not simply find the closest word embedding.
6. Tokenizers must **match the model**; `tiktoken` is not a universal bridge to any Hugging Face or Ollama model.

## Further reading

- [OpenAI tiktoken](https://github.com/openai/tiktoken)
- [Hugging Face — Tokenizers](https://huggingface.co/docs/transformers/main_classes/tokenizer)
- [Hugging Face — Chat templates](https://huggingface.co/docs/transformers/chat_templating)
- [Hugging Face — Text generation](https://huggingface.co/docs/transformers/main_classes/text_generation)
- [Ollama API — Generate](https://docs.ollama.com/api/generate)
