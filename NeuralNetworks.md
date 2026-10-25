# Neural Networks From First Principles: Neurons, Weights, Biases, Layers, and Learning

A beginner-friendly but technically accurate guide to how a **feed-forward neural network** computes predictions, why weighted sums and biases are needed, how layers combine information, and how training changes the network. The core walkthrough follows 3Blue1Brown's [*But what is a neural network?* (Deep Learning, Chapter 1)](https://www.youtube.com/watch?v=aircAruvnKk), with additional worked examples, mathematical notation, training context, and NumPy exercises.

> **Scope:** The video primarily explains the *forward pass* of a simple fully connected neural network. This guide labels further details about optimization, other network types, and Transformers as **additional context**. It does not treat hypothetical edge/loop detectors as a guaranteed account of what trained neurons learn.

## Table of contents

1. [Video overview and chapter-by-chapter summary](#1-video-overview-and-chapter-by-chapter-summary)
2. [What is a neural network?](#2-what-is-a-neural-network)
3. [Representing handwritten digits as numbers](#3-representing-handwritten-digits-as-numbers)
4. [Neurons, activation values, and layers](#4-neurons-activation-values-and-layers)
5. [Why use many layers?](#5-why-use-many-layers)
6. [Why use a weighted sum?](#6-why-use-a-weighted-sum)
7. [What exactly does bias do?](#7-what-exactly-does-bias-do)
8. [Why an activation function is necessary](#8-why-an-activation-function-is-necessary)
9. [Step-by-step pattern detection example](#9-step-by-step-pattern-detection-example)
10. [Does each neuron have different weights and biases?](#10-does-each-neuron-have-different-weights-and-biases)
11. [Parameter counting and the video's 13,002 parameters](#11-parameter-counting-and-the-videos-13002-parameters)
12. [Matrix multiplication: a complete layer in one equation](#12-matrix-multiplication-a-complete-layer-in-one-equation)
13. [Forward propagation through a network](#13-forward-propagation-through-a-network)
14. [How neural networks learn (additional context)](#14-how-neural-networks-learn-additional-context)
15. [Different architectures and parameter sharing (additional context)](#15-different-architectures-and-parameter-sharing-additional-context)
16. [Connection to Transformers and LLM parameters (additional context)](#16-connection-to-transformers-and-llm-parameters-additional-context)
17. [Try it with NumPy](#17-try-it-with-numpy)
18. [Common misconceptions](#18-common-misconceptions)
19. [Practice exercises with answers](#19-practice-exercises-with-answers)
20. [Reference resources and next steps](#20-reference-resources-and-next-steps)

---

## 1. Video overview and chapter-by-chapter summary

**Video:** [3Blue1Brown — But what is a neural network? | Deep learning chapter 1](https://www.youtube.com/watch?v=aircAruvnKk) (Grant Sanderson, October 5, 2017). The video uses handwritten digit recognition to explain neural network structure and computation.

**Companion written lesson:** https://www.3blue1brown.com/lessons/neural-networks/

The video's major sections (timestamps from its published chapter list):

| Time | Subject | Main idea |
|---|---|---|
| 0:00 | Handwriting recognition | A neural network can classify handwritten digits 0–9. |
| 1:07 | Series preview | Neural network computation and training are introduced as a sequence of lessons. |
| 2:42 | Neurons | A neuron is represented by a numerical activation. |
| 3:35 | Layers | The demonstration network has 784 input neurons, two hidden layers of 16 neurons each, and 10 output neurons. |
| 5:31 | Why layers? | Multiple stages might combine pixels into edges, then shapes, then digits. This is an intuition, not a claim that every actual neuron neatly learns an assigned feature. |
| 8:38 | Edge detection | A weighted sum gives different pixel locations different positive or negative influence. Bias adjusts when a neuron becomes active. |
| 11:34 | Counting parameters | The demonstration network has **13,002** weights and biases. |
| 12:30 | Learning | These adjustable parameters determine the network's behavior; their learning is explored in the next lesson. |
| 13:26 | Notation and linear algebra | Matrices and vectors express many weighted-sum calculations at once. |
| 15:17 | Recap | Activations are computed from the previous layer using weights, biases, and nonlinear functions. |
| 16:27 | Final observations | Neural networks can be understood in terms of concrete numerical operations. |
| 17:03 | ReLU versus sigmoid | Sigmoid saturates for large positive/negative inputs; ReLU is a useful alternative in many architectures. |

**Core message:** A neural network isn't handed explicit rules for recognizing digits. Its behavior is determined by many adjustable numbers, called **parameters**, which influence how signals propagate from one layer to the next.

### What the video does and does not teach

- **Covered directly:** Input pixel activations, neurons, dense layers, weighted connections, bias, sigmoid, brief comparison with ReLU, matrix notation, forward computation, and parameter counting.
- **Intuitively motivated:** A hierarchy from local edges to loops/lines to digit identity. Such a hierarchy is *possible* but not guaranteed in any particular trained dense network.
- **Primarily in later lessons:** How a loss function is minimized, gradient descent, and backpropagation. Section 14 below introduces these to complete the learning picture, but they are **not** a detailed summary of Chapter 1.

---

## 2. What is a neural network?

An artificial neural network is a parameterized mathematical function built from successive computational stages called **layers**. It maps inputs, such as pixel intensities, to outputs, such as scores for possible digit classes.

In a simple feed-forward network:

```text
Input image -> Input layer -> Hidden layer 1 -> Hidden layer 2 -> Output layer
                  pixels        features         features      digit scores
```

Every dense neuron after the input layer combines outputs from the previous layer using **weights**, adds its **bias**, and applies an **activation function**.

Neural networks are loosely inspired by biological neurons, but artificial neurons are mathematical computations rather than faithful simulations of brain cells.

## 3. Representing handwritten digits as numbers

In the video's example, each grayscale image is **28 × 28 pixels**:

```text
28 rows × 28 columns = 784 pixels
```

Flattening the image produces 784 input values. After normalization, a pixel might be represented from **0 (dark)** to **1 (bright)**. A pixel at intensity 0.5 is neither fully dark nor fully bright.

```text
Image (28 × 28)
       ↓ flatten
x = [x1, x2, x3, ..., x784]   (784 numbers)
       ↓
Network computes 10 output activations/scores
       ↓
Predicted class = index of the largest output
```

For example, if output neuron 8 has the highest activation, the network predicts the digit **8**. In this sigmoid-based demonstration, output activations are not automatically a normalized probability distribution: they need not sum to 1.

## 4. Neurons, activation values, and layers

A neuron's output is often called its **activation**.

The core formula for one neuron is:

$$
\boxed{z = \sum_{i=1}^{n} w_i x_i + b}
$$

$$
\boxed{a = f(z)}
$$

Where:

| Symbol | Meaning |
|---|---|
| $x_i$ | Activation/input received from previous neuron $i$ |
| $w_i$ | Weight of the connection from input $i$ to this neuron |
| $b$ | This neuron's bias |
| $z$ | Weighted sum **plus** bias, before activation |
| $f$ | Activation function such as sigmoid or ReLU |
| $a$ | Neuron's output activation |

A layer is a collection of neurons computing their outputs from the previous layer. The input layer in this example stores pixel values; it does not have incoming learned weights or biases.

**Important:** A neuron's activation does not have to be exactly 0 or 1. Sigmoid outputs values *between* 0 and 1; ReLU can output 0 or any positive number. Other activations can produce negative values too.

## 5. Why use many layers?

An intuitive approach to image recognition is to detect increasingly abstract information:

```text
Pixels → possible edges → possible corners/loops/lines → evidence for a digit
```

For example, recognizing an **8** may depend on detecting two loops. Multiple layers let learned combinations of simpler signals feed into later computations.

However, **there is no code assigning “vertical line” to neuron 4 or “loop” to layer 3**. Training tunes all parameters, and learned internal representations can be distributed across many neurons. The layer-by-layer illustration explains *why depth could help*, not what every real neuron must represent.

More depth can provide successive nonlinear transformations. Wider layers allow more features to be represented at a particular stage. Deep networks can express complicated input-output functions, but extra depth or width alone does not guarantee better results.

## 6. Why use a weighted sum?

Plain addition treats every input as equally important. A **weighted sum** allows each incoming connection to contribute differently:

$$
z = w_1x_1+w_2x_2+\dots+w_nx_n+b
$$

- A **positive weight** makes a positive input tend to increase the pre-activation $z$.
- A **negative weight** makes a positive input tend to decrease $z$.
- A weight near **zero** makes that input contribute relatively little.
- A **large-magnitude** weight means the input has a stronger effect, all else equal.

These are learned coefficients, **not hardcoded statements that an input is true or false**. Weights can be negative, fractional, or larger than 1. The weight pattern defines what combination of inputs excites or suppresses a neuron.

A weighted sum alone is still a linear combination. The activation function adds nonlinear behavior (Section 8).

## 7. What exactly does bias do?

Bias shifts the weighted sum before applying the activation:

$$z = w\cdot x+b$$

Think of the bias as controlling the neuron's baseline/threshold, while the weights control how the inputs affect the neuron.

For sigmoid, $\sigma(0)=0.5$, so the neuron has activation above 0.5 whenever $z>0$ (and below 0.5 whenever $z<0$). A negative bias means the weighted inputs must provide enough positive contribution to overcome it.

Example: when the weighted sum is 2.5 and bias is −2:

$$z=2.5-2=0.5$$

If the bias changes to −4 with all inputs and weights unchanged:

$$z=2.5-4=-1.5$$

The same input then results in a much lower sigmoid activation.

**Crucial distinction:** Bias does not directly change the stored input pixel values or the weights. It changes the neuron's pre-activation result.

## 8. Why an activation function is necessary

After computing $z$, the neuron applies a nonlinear function $a=f(z)$.

### Sigmoid (used in the video)

$$
\sigma(z)=\frac{1}{1+e^{-z}}
$$

| $z$ | Approximate sigmoid output |
|---:|---:|
| −2.0 | 0.119 |
| −1.2 | 0.231 |
| −0.3 | 0.426 |
| 0 | 0.500 |
| 0.3 | 0.574 |
| 0.5 | 0.622 |
| 2.0 | 0.881 |

It smoothly maps every real input to a value strictly between **0 and 1**. The activation can be read as a strength of response, but **it is not automatically a calibrated probability**.

### ReLU (mentioned in the video)

$$
\operatorname{ReLU}(z)=\max(0,z)
$$

- $z=-1.5$ → 0
- $z=0$ → 0
- $z=2.5$ → 2.5

Sigmoid can become very flat at large positive or negative values, resulting in very small gradients. ReLU often helps with this issue on its positive side (but can also suffer from inactive or “dead” units). Other modern functions include **GELU** and **SiLU**.

### Why can't we omit nonlinear activation functions?

A chain of dense layers with no nonlinear activation only computes another **affine transformation**:

$$
W_2(W_1x+b_1)+b_2 = (W_2W_1)x+(W_2b_1+b_2)
$$

Thus, stacking such layers without nonlinearities does not create a more expressive nonlinear mapping than a single affine layer. Activations allow the network to represent more complicated decision boundaries and feature interactions.

## 9. Step-by-step pattern detection example

Consider a **toy** neuron receiving brightness values for three vertically aligned pixels:

```text
[1]   top pixel
[1]   middle pixel
[1]   bottom pixel
```

Take learned (illustrative) parameters:

$$x=[1,1,1],\quad w=[0.8,0.9,0.8],\quad b=-2$$

### Pattern A: all three pixels active

$$
\begin{aligned}
z &= 1(0.8)+1(0.9)+1(0.8)-2 \\
  &= 0.5
\end{aligned}
$$

$$a=\sigma(0.5)\approx 0.622$$

### Pattern B: only the top and middle pixels active

$$x=[1,1,0]$$

$$z=0.8+0.9+0-2=-0.3$$

$$a=\sigma(-0.3)\approx0.426$$

### Pattern C: only the top pixel active

$$x=[1,0,0]$$

$$z=0.8-2=-1.2,\quad a\approx0.231$$

### Pattern D: no pixels active

$$x=[0,0,0]$$

$$z=0-2=-2,\quad a\approx0.119$$

| Pixel pattern | Sum before bias | $z$ after bias | Sigmoid activation |
|---|---:|---:|---:|
| `[1,1,1]` | 2.5 | 0.5 | 0.622 |
| `[1,1,0]` | 1.7 | −0.3 | 0.426 |
| `[1,0,0]` | 0.8 | −1.2 | 0.231 |
| `[0,0,0]` | 0 | −2 | 0.119 |

The neuron responds most strongly to the full set of active pixels. This illustrates weighted evidence plus a threshold shift, **not a complete or robust vertical-line detector**: with all-positive weights, it could also activate for non-line arrangements if it receives those inputs. A practical filter often includes negative weights for pixels that *should be dark* and learns a richer spatial pattern.

### Small two-input calculation

$$x=[1,1],\quad w=[0.5,0.8],\quad b=-1$$

$$z=1(0.5)+1(0.8)-1=0.3$$

$$a=\sigma(0.3)\approx\boxed{0.574}$$

Notice that **0.3 is the pre-activation; 0.574 is the neuron output**.

## 10. Does each neuron have different weights and biases?

For a **fully connected (dense) layer**:

> **Each connection has its own weight. Each neuron in the receiving layer has its own bias.**

Take three inputs and two receiving neurons:

```text
x1 ── w11 ──┐
x2 ── w12 ──┼──> neuron 1 (+ b1) ──> a1
x3 ── w13 ──┘

x1 ── w21 ──┐
x2 ── w22 ──┼──> neuron 2 (+ b2) ──> a2
x3 ── w23 ──┘
```

The equations are:

$$z_1=w_{11}x_1+w_{12}x_2+w_{13}x_3+b_1$$

$$z_2=w_{21}x_1+w_{22}x_2+w_{23}x_3+b_2$$

$$a_1=f(z_1),\qquad a_2=f(z_2)$$

Both neurons receive the same inputs but can learn **different combinations** because their weights and biases are independently trainable. For example, one might become responsive to an edge orientation while another responds to something else.

If two identical neurons have identical weights and biases, receive the same inputs, and are subject to symmetric learning updates, their outputs and updates can remain identical. This is one reason symmetry-breaking initialization matters.

This small layer contains **3 × 2 = 6 weights**, plus **2 biases**, for **8 trainable parameters**.

### A wording nuance

Saying “each neuron has its own weights” is useful shorthand, but more precisely: each **incoming connection** to a dense neuron has a weight. The bias is associated with the receiving neuron. The input layer itself has no incoming learned connection weights in this simple example.

## 11. Parameter counting and the video's 13,002 parameters

For a dense layer with $n_{in}$ input features and $n_{out}$ neurons:

$$
\boxed{\text{weights}=n_{in}\times n_{out}}
$$

$$
\boxed{\text{biases}=n_{out}}
$$

$$
\boxed{\text{total parameters}=(n_{in}+1)n_{out}}
$$

### Example A: 4 → 5 → 3 → 2 network

| Connection | Weights | Biases | Parameters |
|---|---:|---:|---:|
| Input (4) → Hidden 1 (5) | $4\times5=20$ | 5 | 25 |
| Hidden 1 (5) → Hidden 2 (3) | $5\times3=15$ | 3 | 18 |
| Hidden 2 (3) → Output (2) | $3\times2=6$ | 2 | 8 |
| **Total** | **41** | **10** | **51** |

### Example B: video architecture, 784 → 16 → 16 → 10

| Connection | Weights | Biases | Parameters |
|---|---:|---:|---:|
| 784 → 16 | 12,544 | 16 | 12,560 |
| 16 → 16 | 256 | 16 | 272 |
| 16 → 10 | 160 | 10 | 170 |
| **Total** | **12,960** | **42** | **13,002** |

That is the video's **13,002 knobs and dials**. Each can affect how the network transforms input pixels into outputs.

**Parameters are not the number of training images or the number of stored rules.** They are the learned numerical quantities making up the model.

## 12. Matrix multiplication: a complete layer in one equation

Many individual weighted sums can be expressed together:

$$
\boxed{\mathbf z=W\mathbf x+\mathbf b}
$$

$$
\boxed{\mathbf a=f(\mathbf z)}
$$

For a layer with **3 inputs** and **2 output neurons**:

$$
W=\begin{bmatrix}w_{11}&w_{12}&w_{13}\\w_{21}&w_{22}&w_{23}\end{bmatrix}
\quad (2\times3)
$$

$$
\mathbf x=\begin{bmatrix}x_1\\x_2\\x_3\end{bmatrix}\quad(3\times1),\qquad
\mathbf b=\begin{bmatrix}b_1\\b_2\end{bmatrix}\quad(2\times1)
$$

$$
W\mathbf x+\mathbf b=
\begin{bmatrix}
w_{11}x_1+w_{12}x_2+w_{13}x_3+b_1\\
w_{21}x_1+w_{22}x_2+w_{23}x_3+b_2
\end{bmatrix}
$$

Here $f$ is applied **elementwise**. The result has shape **2 × 1**, giving one activation for each output neuron.

### Why matrices matter

A matrix stores all connection weights for a layer. Matrix multiplication evaluates all weighted sums efficiently, especially for many examples processed as a batch on GPUs. In ML libraries, the weight matrix may be stored transposed depending on the convention; the underlying calculation is the same.

For a batch $X$ of shape `(batch_size, input_features)` with rows as samples, a common implementation is:

```python
Z = X @ W.T + b   # W has shape (output_features, input_features)
A = activation(Z)
```

## 13. Forward propagation through a network

The **forward pass** means calculating outputs using the *current* weights and biases, without changing them:

```text
Input pixels x
    │
    ▼
z1 = W1 x + b1
    │
    ▼
a1 = activation(z1)
    │
    ▼
z2 = W2 a1 + b2
    │
    ▼
a2 = activation(z2)
    │
    ▼
z3 = W3 a2 + b3
    │
    ▼
output = activation(z3)
    │
    ▼
predicted digit = index with highest output activation
```

Each layer receives the preceding layer's **activations**, not the raw inputs from every earlier layer (in this basic sequential architecture). The same computational pattern can be repeated layer after layer. A forward pass is used during **both training and inference**.

## 14. How neural networks learn (additional context)

**This is a brief overview extending beyond the first 3Blue1Brown video.** See Chapter 2 (gradient descent) and Chapters 3–4 (backpropagation) for full explanations.

The network does **not** come with perfectly selected weights and biases. During supervised training:

1. **Initialize parameters:** Begin with an initialization scheme that usually gives weights small/random values and biases initial values. Different initialization schemes are used for different activations and architectures.
2. **Forward pass:** Run an input image through the network to get predictions.
3. **Compare with known label:** For an image labelled 8, compare output with target 8 using a chosen **loss function**. This is mathematical comparison against labelled data, not a person manually checking every prediction.
4. **Backpropagation:** Apply the chain rule to compute gradients of the loss with respect to individual weights and biases.
5. **Optimization:** An optimizer updates parameters in a loss-reducing direction (locally), e.g. gradient descent.
6. **Repeat:** Train over many examples and iterations, and evaluate on held-out data to test generalization.

A conceptual gradient descent update is:

$$
w\leftarrow w-\eta\frac{\partial L}{\partial w}
$$

$$
b\leftarrow b-\eta\frac{\partial L}{\partial b}
$$

Here $L$ is loss and $\eta$ is the learning rate. An individual update is not guaranteed to lower the loss on every example; real optimization is more nuanced.

### Training versus inference

| Stage | What happens to parameters? |
|---|---|
| Training | Weights and biases are updated using gradient-based optimization. |
| Inference | Current parameters are used for a forward pass; ordinary inference does not change them. |

## 15. Different architectures and parameter sharing (additional context)

The per-connection rule above describes **dense layers**, not all neural network designs.

| Architecture | Parameter organization |
|---|---|
| **Dense / fully connected** | Each input-to-output connection in the layer has its own weight; one bias per output neuron in the usual formulation. |
| **Convolutional (CNN)** | A filter's weights are reused at many spatial positions; often one bias per output channel. |
| **Recurrent (RNN)** | The same recurrent transformation weights are reused across sequence time steps. |
| **Transformer** | Attention and feed-forward projection matrices are reused across token positions within a layer; separate blocks usually have their own parameters, though some designs share them. |

This distinction prevents the mistaken conclusion that **every activation at every location or time step has a unique independently learned weight**.

## 16. Connection to Transformers and LLM parameters (additional context)

Neural networks for language models use the same foundational operations — **vectors, matrix multiplication, learned weights, nonlinear transformations** — but a Transformer is more sophisticated than the handwritten-digit feed-forward example.

A simplified decoder-only Transformer flow is:

```text
Text -> tokenizer -> token IDs -> embedding vectors
     -> repeated Transformer blocks (attention + feed-forward layers)
     -> final representation -> vocabulary logits
     -> probabilities / token selection -> next token
```

A dense feed-forward sublayer inside a Transformer still contains learned matrices that mix and transform features. Attention adds contextual interactions between token positions; it is not equivalent to the digit recognition network's simple dense layers.

When an LLM has billions of **parameters**, those are principally learned numbers in embedding/projection/feed-forward matrices, along with other trainable parameters where included. Parameter count is not a count of independently stored facts, deterministic rules, or neurons. More parameters can improve capacity but do not by themselves ensure correctness or quality.

## 17. Try it with NumPy

This example exactly recreates the two-neuron / three-input dense layer computation and lets you inspect shapes:

```python
import numpy as np

# Three input activations
x = np.array([1.0, 1.0, 1.0])   # shape (3,)

# Two neurons, three incoming weights each
W = np.array([
    [0.8, 0.9, 0.8],
    [-0.5, 0.2, -0.4],
])                                  # shape (2, 3)

# Each output neuron has its own bias
b = np.array([-2.0, 0.1])        # shape (2,)

def sigmoid(z):
    return 1.0 / (1.0 + np.exp(-z))

z = W @ x + b
activations = sigmoid(z)

print('W shape:', W.shape)
print('x shape:', x.shape)
print('b shape:', b.shape)
print('pre-activation z:', z)
print('neuron outputs:', activations)
print('parameter count:', W.size + b.size)
```

Expected output (approximately):

```text
W shape: (2, 3)
x shape: (3,)
b shape: (2,)
pre-activation z: [ 0.5 -0.6]
neuron outputs: [0.62245933 0.35434369]
parameter count: 8
```

Try these changes:

1. Change `x` to `[1, 0, 0]`. Inspect both neurons' outputs.
2. Change the first neuron's bias from `-2.0` to `-4.0`, leaving weights untouched.
3. Change the first neuron's middle weight from `0.9` to `-0.9` and interpret the result.
4. Implement `relu = lambda z: np.maximum(0, z)` and compare it to sigmoid.
5. Add a second dense layer: `W2` of shape `(2, 2)` and `b2` of shape `(2,)`. Feed the first layer's activations into it.

## 18. Common misconceptions

| Misconception | More accurate explanation |
|---|---|
| "A neuron only has a bias." | A typical dense neuron has **multiple incoming connection weights and one bias**. |
| "All neurons in a layer have the same weights and bias." | Dense neurons have **independently learned** weights and biases; architectures with sharing are different. |
| "Every neuron outputs only 0 or 1." | Activations often output continuous values; allowed ranges depend on the function. |
| "A bias directly multiplies or changes each input." | Bias is **added after the weighted sum**, before activation. |
| "Weights must be between 0 and 1." | Weights may be negative, zero, fractional, or greater than 1. |
| "Sigmoid output is automatically a probability." | Sigmoid outputs a value in (0,1); its calibration/interpretation depends on the model and training objective. |
| "A layer is explicitly programmed to detect lines or circles." | Training learns parameters; feature specialization is not manually assigned or guaranteed. |
| "All parameter values are chosen manually." | Most weights and biases are learned from data through optimization. |
| "The network compares outputs to known labels during ordinary inference." | Known labels are used for computing training/evaluation loss, not usually needed for inference. |
| "Stacking only weighted sums gives unlimited expressiveness." | Without nonlinearities, sequential affine layers collapse into one affine transformation. |
| "A 13,002-parameter model remembers 13,002 facts." | Parameters are learned numerical coefficients, not a fact inventory. |

## 19. Practice exercises with answers

### Exercise 1: Calculate a neuron activation

Given $x=[1,1]$, $w=[0.5,0.8]$, $b=-1$, and sigmoid activation, find $z$ and $a$.

**Answer:** $z=0.5+0.8-1=0.3$ and $a=\sigma(0.3)\approx0.574$.

### Exercise 2: Count parameters

A network has layer sizes **4 → 5 → 3 → 2**. Assume all connections between successive layers are dense, and each non-input neuron has a bias.

**Answer:** Weights $4(5)+5(3)+3(2)=41$; biases $5+3+2=10$; total **51**.

### Exercise 3: Predict the effect of a bias change

If a neuron's weights and inputs stay fixed but its bias decreases by 2, what happens to its pre-activation? What happens to its sigmoid activation?

**Answer:** Pre-activation decreases by exactly 2, and sigmoid activation decreases (because sigmoid is strictly increasing).

### Exercise 4: Explain why two neurons can detect different patterns

Two neurons in the same dense layer receive the same input vector. Why might they produce different outputs?

**Answer:** They have different incoming connection weights and/or biases, so their weighted sums and activations differ.

### Exercise 5: Compute matrix dimensions

A layer accepts 784 inputs and outputs 16 activations. Using column-vector convention, what are the shapes of $W$, $x$, $b$, and the output?

**Answer:** $W:(16,784)$, $x:(784,1)$, $b:(16,1)$, output $(16,1)$.

## 20. Reference resources and next steps

### Main video / primary source

- [3Blue1Brown — But what is a neural network? (Chapter 1)](https://www.youtube.com/watch?v=aircAruvnKk)
- [3Blue1Brown — Written companion lesson](https://www.3blue1brown.com/lessons/neural-networks/)

### Follow-on learning

- [3Blue1Brown — Gradient descent, how neural networks learn (Chapter 2)](https://www.3blue1brown.com/lessons/gradient-descent/)
- [3Blue1Brown — Neural networks topic page](https://www.3blue1brown.com/topics/neural-networks)
- [Michael Nielsen — Neural Networks and Deep Learning](http://neuralnetworksanddeeplearning.com/)

### Recommended learning order

1. Compute several neurons manually using **weighted sums and biases**.
2. Compare **sigmoid vs ReLU** and explain why nonlinearity is necessary.
3. Master **dot products and matrix dimensions** for dense layers.
4. Implement a forward pass using **NumPy**.
5. Learn how a **loss function**, **derivatives**, **gradient descent**, and **backpropagation** update parameters.
6. Connect those ideas to **Transformer feed-forward networks and attention**.

---

## One-page mental model

```text
                  A FEED-FORWARD NEURAL NETWORK

 Raw inputs (e.g., 784 pixel brightness values)
                       │
                       ▼
               Input vector x
                       │
                       ▼
            For each dense layer:
              z = W @ x + b
              a = activation(z)
                       │
                       ▼
              Next layer uses a
                       │
                       ▼
             Final output values
                       │
                       ▼
          Class decision (e.g., digit 8)

 WEIGHTS:        learned coefficients on connections
 BIASES:         learned offsets for receiving neurons
 ACTIVATIONS:    nonlinear transformed output values
 LAYERS:         successive stages of computation
 PARAMETERS:     learned numerical weights + biases (+ others)
 TRAINING:       update parameters based on loss gradients
 INFERENCE:      use existing parameters to make predictions
```

**Core equation to remember:**

$$\boxed{\mathbf a=f(W\mathbf x+\mathbf b)}$$

If you understand every symbol, the dimensions, and why the activation function is nonlinear, you have the mathematical foundation needed to understand dense neural networks — and an essential building block for modern AI models.
