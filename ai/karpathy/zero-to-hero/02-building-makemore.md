
## Strategy 1: Count-Based Bigram Model

Core Idea: Build a 27×27 count matrix of every character transition in the dataset, normalize each row into a probability distribution, then sample from it directly.
More detailed in: [[bigrams]]

## Strategy 2: Neural Network Approach (Simple)

### What kind of network is this one?

It is a **single-layer neural network with no hidden layers**. It just takes an input, a weight matrix, and a softmax output.  It is the simplest possible neural net and it is essentially a linear classifier (more on [[linearity]]. 

---
### Training Data

```python
xs, ys = [], []
for w in words:
    chs = ['.'] + list(w) + ['.']
    for ch1, ch2 in zip(chs, chs[1:]):
        ix1 = stoi[ch1]
        ix2 = stoi[ch2]
        xs.append(ix1)
        ys.append(ix2)
xs = torch.tensor(xs)  # inputs  — current character indices
ys = torch.tensor(ys)  # labels  — correct next character indices
```
Same bigram loop as before where every consecutive character pair across every word. `xs` is the input (current character) and `ys` is the label (correct next character) the network should learn to predict.

### Weight Matrix & One-Hot Encoding
```python
W = torch.randn(27, 27, generator=g, requires_grad=True)
xenc = F.one_hot(xs, num_classes=27).float()
```

**W** is a 27x27 matrix of learnable parameters (first randomly initialized). Each row corresponds to one input character, and each column corresponds to one possible next character. 

Through training, W learns to store log-counts for every character transition.

**Why One-Hot Encoding?** 
- We can't feed raw integer indices into the network as integers carry implied numerical relationships (index 13 > index 5 doesn't mean `m` is "greater than" `e`). 
- One-hot encoding converts each integer into a vector of 27 zeros with a single `1` at the character's position, treating every character as its own independent category with no relationship to any other.
```
index 5  (e) → [0, 0, 0, 0, 0, 1, 0, 0, ...]
index 13 (m) → [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, ...]
```

When `xenc @ W` is computed, the single `1` in the one-hot vector **selects the corresponding row of W** — all other rows contribute nothing. So `xenc @ W` is really just a differentiable way of doing a row lookup.


### Forward Pass: Logits --> Softmax --> Probabilities

```python
logits = xenc @ W              # log-counts
counts = logits.exp()          # equivalent to our N array
probs = counts / counts.sum(1, keepdims=True)  # normalize to probabilities
```

**Step 1 - Logits:** `xenc @ W` produces a row of 27 numbers for each example. Each one is a number per possible next character. These are called logits (log-counts) and they can be any real number (positive or negative). 

**Step 2 - Exponentiate**: `logits.exp()` converts log-counts to counts, making all values strictly positive (just like the raw count matrix `N` from the bigram model). 

**Step 3 — Normalize:** Dividing by the row sum converts counts into a valid probability distribution, exactly like we did with `P` in the bigram model.

Steps 2 and 3 together are the **softmax** function:
$\text{softmax}(x_i) = \frac{e^{x_i}}{\sum_j e^{x_j}}$

Softmax takes any vector of real numbers and squashes it into a probability distribution that sums to 1.

### Loss Function

```python
loss = -probs[torch.arange(num), ys].log().mean() + 0.01*(W**2).mean()
```
#### Negative Log Likelihood
`probs[torch.arange(num), ys]` picks, for each training example, the probability the network assigned to the **correct** next character. We then:

- Take the log of those probabilities
- Average them (`.mean()`)
- Negate (minimize NLL instead of maximize log likelihood)

This is identical to the NLL we computed for the bigram model.

#### Regularization (similar to Laplace Smoothing)
```python
+ 0.01*(W**2).mean()
```

**L2** regularization adds a penalty for large weights. It forces the optimizer to keep weights small and close to zero. 

When W is close to zero, `logits=0` for all entries, and `exp(0) = 1` for every count. This means all 27 next characters appear equally likely. **This is a uniform distribution.**

This is exactly what Laplace smoothing did in the bigram model: adding `+1` to every count pulled probabilities away from zero and *toward a more uniform distribution*, preventing the model from being infinitely confident about any rare or unseen transitions.

|Method|Mechanism|Effect|
|---|---|---|
|Laplace smoothing|Add 1 to every count|Pulls probabilities toward uniform|
|L2 regularization|Penalize large weights|Pulls logits toward 0 → pulls probabilities toward uniform|

Both are doing the same thing conceptually of **preventing the model from being too confident.** The `0.01` coefficient controls the strength here where a larger value forces weights closer to zero and a more uniform distribution, a smaller value lets the model be more confident.

### Backward Pass & Update
```python
W.grad = None       # zero out gradients (equivalent to p.grad = 0.0 in micrograd)
loss.backward()     # backprop — fills W.grad
W.data += -50 * W.grad  # gradient descent step
```

`W.grad = None` is PyTorch's preferred way to zero gradients. It is slightly more memory efficient than setting to zero. Everything else is the same training loop pattern from micrograd.


---

## Strategy 3: MLP Language Model 

The core upgrade over the bigram neural net is **context**: instead of predicting the next character from just one previous character, we use the previous **3 characters** as context, each represented as a learned **10-dimensional embedding**.

---

## Building the Dataset

```python
def build_dataset(words):
    block_size = 3  # context length: how many characters do we take to predict the next one?
    X, Y = [], []
    for w in words:
        context = [0] * block_size
        for ch in w + '.':
            ix = stoi[ch]
            X.append(context)
            Y.append(ix)
            context = context[1:] + [ix]  # crop and append

    X = torch.tensor(X)
    Y = torch.tensor(Y)
    return X, Y
```

The function converts a list of words into (input → target) training pairs using a **sliding window** of 3 characters.

- **`context = [0] * block_size`** — every word starts with a context of `[0, 0, 0]`, i.e. three `.` padding tokens. This lets the model learn what characters words *begin* with.
- **`for ch in w + '.'`** — a `.` is appended to each word so the model also learns where words *end*.
- **Each iteration records one training example**: the current 3-character context goes into `X`, and the actual next character `ix` goes into `Y` (the label).
- **`context = context[1:] + [ix]`** — the window slides forward: drop the oldest character, append the one just seen.

### Example: the word "emma"

| Context (X) | Target (Y) |
| ----------- | ---------- |
| `. . .`     | `e`        |
| `. . e`     | `m`        |
| `. e m`     | `m`        |
| `e m m`     | `a`        |
| `m m a`     | `.`        |
One word of length n produces **n + 1 examples**. Across ~32k names this yields ~228k  examples, so `X` has shape `(228146, 3)` and `Y` has shape `(228146,)`.

---

## Data Splits

```python
random.shuffle(words)
n1 = int(0.8 * len(words))
n2 = int(0.9 * len(words))

Xtr,  Ytr  = build_dataset(words[:n1])      # 80% — training
Xdev, Ydev = build_dataset(words[n1:n2])    # 10% — validation
Xte,  Yte  = build_dataset(words[n2:])      # 10% — test
```

The dataset is split into three non-overlapping parts:

| Split            | Size | Purpose                                                                           |
| ---------------- | ---- | --------------------------------------------------------------------------------- |
| Training         | 80%  | Optimize model **parameters** (W1, b1, W2, b2, C) via gradient descent            |
| Validation (dev) | 10%  | Tune **hyperparameters** (embedding size, hidden size, learning rate, block size) |
| Test             | 10%  | Final honest evaluation                                                           |

---

## Network Architecture

```python
C  = torch.randn((27, 10))    # embedding table
W1 = torch.randn((30, 200))   # hidden layer weights
b1 = torch.randn(200)         # hidden layer biases
W2 = torch.randn((200, 27))   # output layer weights
b2 = torch.randn(27)          # output layer biases
```

The network has three stages:

### Stage 1 — Embedding lookup

```python
emb = C[Xtr[ix]]   # (32, 3, 10)
```

Each of the 3 context characters gets looked up in `C`, the embedding table, giving it a 10-dimensional vector. The result is shape `(32, 3, 10)` — 32 training examples, each with 3 context characters, and 10 dimensions per character! 

### Stage 2 — Hidden layer

```python
h = torch.tanh(emb.view(-1, 30) @ W1 + b1)   # (32, 200)
```

The 3 embeddings are flattened into a single vector of 30 numbers (`3 × 10 = 30`), then passed through a hidden layer of 200 neurons. Each neuron computes a weighted sum over all 30 inputs, adds its bias, and squashes the result through `tanh`. The output is 200 numbers per example and this is the hidden layer's representation of the context.

### Stage 3 — Output layer

```python
logits = h @ W2 + b2   # (32, 27)
```

The 200 hidden values are projected down to 27 logits (one per possible next character). No activation here as the softmax is handled inside the loss function.

---

## Cross Entropy Loss

```python
loss = F.cross_entropy(logits, Ytr[ix])
```

`F.cross_entropy` combines softmax + negative log likelihood in a single operation. It is mathematically identical to what we computed manually in the bigram model, but:

- **More numerically stable** — avoids computing `exp()` of very large or small numbers directly
- **More memory efficient** — doesn't need to materialise the full probability distribution

It takes raw logits and the correct label indices directly, and returns the average NLL across the batch.

---

## Training Loop

```python
for i in range(200000):
    # minibatch
    ix = torch.randint(0, Xtr.shape[0], (32,))

    # forward pass
    emb = C[Xtr[ix]]
    h = torch.tanh(emb.view(-1, 30) @ W1 + b1)
    logits = h @ W2 + b2
    loss = F.cross_entropy(logits, Ytr[ix])

    # backward pass
    for p in parameters:
        p.grad = None
    loss.backward()

    # update
    lr = 0.1 if i < 100000 else 0.01
    for p in parameters:
        p.data += -lr * p.grad
```

### Minibatch

At each step, 32 random training examples are sampled. The gradient computed over these 32 examples is a noisy but unbiased estimate of the true gradient over the full dataset. This is far cheaper per step and the noise can help escape poor local minima.

### Learning Rate Decay

```python
lr = 0.1 if i < 100000 else 0.01
```

The learning rate starts at `0.1` for the first 100,000 steps and this is large enough to make fast progress. It then drops to `0.01` for the remaining 100,000 steps to fine-tune near the minimum without overshooting it. This is called **step decay** and it's the simplest form of learning rate scheduling.

---

## Parameters vs Hyperparameters

| Examples            | How they're set                                                                        |                                                            |
| ------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Parameters**      | C, W1, b1, W2, b2                                                                      | Learned automatically by gradient descent                  |
| **Hyperparameters** | embedding size (10), hidden size (200), block size (3), learning rate, batch size (32) | Set manually by the developer, tuned using the **dev set** |

The dev set exists precisely to tune hyperparameters. You try different values, evaluate on dev, and keep what works.

---

## Evaluating the Model

```python
# Training loss — how well the model fits the data it trained on
emb = C[Xtr]
loss = F.cross_entropy(h @ W2 + b2, Ytr)

# Dev loss — how well it generalises to unseen data
emb = C[Xdev]
loss = F.cross_entropy(h @ W2 + b2, Ydev)
```

Comparing these two numbers tells you where you are:

| Training loss | Dev loss | Diagnosis                                                                                                                        |
| ------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------- |
| High          | High     | **Underfitting** — model too simple, we gotta increase network size or embedding size. More about underfitting: [[underfitting]] |
| Low           | High     | **Overfitting** — model is memorising training data, reduce complexity or get more data                                          |
| Low           | Low      | **Good fit** — model is generalising well                                                                                        |

The test loss is only evaluated once the model is fully finalized.

### Several Issues with Strategy #3 Model.

### Issue 1: Saturated Tanh

**What is it?**
`tanh` squashes any input it receives into (-1, 1). When the input is large in magnitude (e.g. +5 or -8), the output is pinned to almost exactly +1 or -1. 

When `tanh(x) ≈ ±1`, then `1 - (±1)² ≈ 0`. The gradient is essentially zero.

During backprop:
`self.grad += (1 - t**2) * out.grad`

No gradient will flow back through that neuron and the weights feeding into it stop updating. We call this a **dead neuron** as it's not learning anything.

**Why does it happen at initialization?**
With `W1 = torch.randn((30, 200))`, weights have std ≈ 1. The pre-activation is `emb @ W1 + b1` — a sum of 30 products of random values. Summing 30 random values causes the result to blow up in magnitude, pushing most neurons straight into the flat tails of tanh.

**The fix** is to scale our weights down by multplying them by a factor: $W1 * 0.1$. 

This keeps the pre-activations as close to 0 as possible --> $tanh(0) \approx 0$, which is right in the active region where the derivative is approximately 1. 


## Kaiming Initialization

```python
W1 = torch.randn((30, 200), generator=g) * (5/3) / (30**0.5)
```

This replaces the rough `* 0.1` hack from before with a principled formula.

---

## What it does

It scales the randomly initialized weights so that the **std of activations stays at 1.0 through the layer**.

The formula is:

$$W = \text{randn} \times \frac{\text{gain}}{\sqrt{n_{in}}}$$

Breaking down each part:

**`30**0.5` = $\sqrt{n_{in}}$** — this is the core fix. As established, multiplying `n` random values together causes std to grow by $\sqrt{n}$. Dividing by $\sqrt{n_{in}}$ exactly cancels that growth out, keeping std ≈ 1.

**`5/3` = the gain** — this is a correction factor specific to `tanh`. tanh isn't perfectly linear as it squashes values, which slightly shrinks the std passing through it. The `5/3` compensates for that squashing so std stays at 1 even after the tanh is applied. Different activation functions have different gains:

|Activation|Gain|
|---|---|
|tanh|5/3 ≈ 1.667|
|ReLU|√2 ≈ 1.414|
|Linear / none|1.0|

---

## Why it's needed over `* 0.1`

`* 0.1` was a guess and it happened to work for this specific network size. But if you changed the layer from 30 inputs to 100 inputs, `* 0.1` would no longer be the right scale. We'd have to manually re-tune it.

Kaiming initialization **automatically computes the correct scale** for any layer size. It doesn't matter if `n_in` is 30 or 300 as the formula adapts and the std stays at 1 regardless.


### Batch Normalization

Batch normalization is a technique that **forces the pre-activations of a layer to have mean=0 and std=1** at every step of training automatically. This is regardless of what the weights are doing. 

It directly solves the saturation problem at its root rather than carefully tuning initialization (what we did above). 

**Why it's needed?**
Even with Kaiming initialization, as training progresses and weights changes. There's no guarantee that these weights stay in a good range. These weights can drift and cause pre-activations to grow large again. This then re-saturates `tanh`. Kaiming effectively only fixes the problem at the initialization step. 

---
**How does batch norm work?**

```python
bnmeani = hpreact.mean(0, keepdim=True)
bnstdi  = hpreact.std(0, keepdim=True)
hpreact = bngain * (hpreact - bnmeani) / bnstdi + bnbias
```

Step 1 - Normalize
- For each of the 200 neurons, take the 32 pre-activation values in the minibatch and:
	- Subtract the mean --> shifts the distribution to be centered at 0. 
	- Divide by std --> squashes the distribution to have a spread of 1. 
- After this, every neuron's pre-activations across the batch have exactly mean=0 and std=1. This is called **standardization**.
- Concretely, if neuron 7's pre-activations across 32 examples were `[-8, 3, 15, -2, ...]`, after normalization they become something like `[-0.9, 0.1, 1.2, -0.05, ...]` — all nicely centered and scaled.

Step 2 - Scale and Shift
- If we only left off at step 1, we would have taken away the freedom of the network and all pre-activations are forced into the same range regardless of what the layer learned. 
So we introduce two **learnable parameters per neuron**:
- `bngain` (initialized to 1) — lets the network learn to scale the std back up if needed
- `bnbias` (initialized to 0) — lets the network learn to shift the mean if needed

These get trained via backprop alongside the weights. So the network can learn "neuron 7 works best with std=2 and mean=0.5" and bngain/bnbias will adjust to that. At initialization they do nothing (×1 +0), but over time they adapt.

This is why `b1` is removed and `bnbias` replaces it. Having both would be redundant since bnbias already shifts the mean.


### What it looks like before vs after

**Without batch norm** — pre-activations spread all over the place, many values large, tanh saturates:

```
pre-activations: [-12, 0.3, 8.5, -6.1, 15, 0.1, -9, ...]
after tanh:      [-1,  0.3, 1.0, -1.0,  1, 0.1, -1, ...]  ← lots of ±1
```

**With batch norm** — pre-activations normalized before tanh:

```
pre-activations: [-0.8, 0.1, 1.2, -0.5, 1.8, 0.0, -0.9, ...]
after tanh:      [-0.7, 0.1, 0.8, -0.5, 0.9, 0.0, -0.7, ...] 
```

---

### The running statistics problem

There's a catch. During training, batch norm uses the **mean and std of the current minibatch**. But at inference time (when you're sampling names), you're passing in one example at a time and you can't compute a meaningful mean and std from a single example.

The solution is to track a **running estimate** of the true mean and std across all batches seen during training:
```python
with torch.no_grad():
    bnmean_running = 0.999 * bnmean_running + 0.001 * bnmeani
    bnstd_running  = 0.999 * bnstd_running  + 0.001 * bnstdi
```

This is an **exponential moving average**,  it slowly updates the running estimate each step, weighting recent batches slightly more than old ones. `0.999` means "keep 99.9% of what you knew before, add 1% of what you just saw."

By the end of training, `bnmean_running` and `bnstd_running` are stable estimates of the mean and std across the whole dataset. At inference time, you use these fixed values instead of computing from the batch.

`torch.no_grad()` means these updates don't participate in backprop because they aren't learnable parameters. 


### Issue 2: Softmax is Confidently Wrong.

At initialization, the network hasn't learned anything, so it should be maximally uncertain and assign equal probability $\approx$ 1/27 to every next character. 

That should then give an ideal starting loss of -log(1/27) = log(27) = 3.296. 

But with `W2 = torch.randn((200, 27))`, the 27 logits come out wildly unequal with something like `[-5, 8, -3, 2, -7, ...]`. Softmax will see the value `8` and it confidently assigns most of the probability to that one character.

It's almost certainly wrong, so the loss explodes (> 15) instead of 3.296.

This causes a **massive gradient** on step 1, an enormous destabilizing weight update, and then the loss drops sharply back down to a normal range. That's the **hockey stick** shape in the loss curve.

**Why is the hockey stick bad?**

Those first N steps were entirely wasted as the network spent them correcting a bad initialization rather than actually learning patterns from the data. We want learning to start from step 1 and not step 1000.

**The fix: `W2 * 0.01`, `b2 * 0`**

With near-zero weights, all 27 logits come out close to 0. Softmax of values close to 0 like `[0.98, 0.40, 0, ..., 0]` means it's almost a perfectly uniform distribution with every character geting a probability close to 1/27. 


