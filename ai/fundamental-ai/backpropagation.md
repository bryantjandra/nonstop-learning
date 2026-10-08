
## What & Why

**Backpropagation** is the algorithm for efficiently computing the gradient of a loss function w.r.t. every parameter in a model.

Without it, training neural nets at scale would be impossible. Computing gradients numerically (the nudge method) would require one forward pass per parameter — millions of forward passes per step. Backprop gets them all in **one forward pass + one backward pass**.

## The Computation Graph

Any mathematical expression can be drawn as a **graph**:

- **Nodes** = values (inputs, intermediate results, outputs)
- **Edges** = operations connecting them (`+`, `*`, `tanh`, etc.)

Example expression: $e = a \cdot b,; d = e + c,; L = d \cdot f$

```
a ─┐
   ├─[*]─→ e ─┐
b ─┘          ├─[+]─→ d ─┐
        c ────┘          ├─[*]─→ L
                   f ────┘
```

Every neural network — no matter how deep — is just a (much bigger) computation graph. Backprop operates on this graph.

## The Forward Pass

**Goal:** compute the final output (usually the **loss**) from the inputs.

**How:**

1. Start at the input nodes.
2. Walk the graph **left to right**, applying each operation in order.
3. At each node, store the computed value — we'll need it during the backward pass.

For the example above, with $a=2.0,; b=-3.0,; c=10.0,; f=-2.0$:

- $e = a \cdot b = -6.0$
- $d = e + c = 4.0$
- $L = d \cdot f = -8.0$

That's it. The forward pass is just _"do the math."_ Its real job is to (1) produce the loss and (2) **cache intermediate values** so the backward pass has what it needs.

## Loss

The output of the forward pass is typically a single number called the **loss** — a measure of how wrong the model is. Lower = better.

We want to find parameter values that make the loss as small as possible. To do that, we need to know how the loss responds to changes in each parameter — i.e. its gradient.

## The Backward Pass (Backprop)

**Goal:** for every node in the graph, compute $\frac{\partial L}{\partial \text{node}}$ — how much the loss would change if that node were nudged.

**How:**

1. **Seed:** the loss's gradient w.r.t. itself is **1**.
2. Walk the graph **right to left**.
3. At each node, apply the **chain rule**:

$$\text{node.grad} \mathrel{+}= (\text{incoming grad from above}) \times (\text{local derivative})$$

- **Incoming grad** = how much the loss changes w.r.t. the _output_ of this operation.
- **Local derivative** = how much that output changes w.r.t. _this input_ (depends on the operation).
- **`+=`** because a node feeding multiple downstream paths must accumulate contributions from each (multivariable chain rule).

### Local derivatives per operation

Each operation has its own rule for what the local derivative looks like:

|Operation|Output|Local derivatives|
|---|---|---|
|Addition|$f = a + b$|$\frac{\partial f}{\partial a} = 1,; \frac{\partial f}{\partial b} = 1$|
|Multiplication|$f = a \cdot b$|$\frac{\partial f}{\partial a} = b,; \frac{\partial f}{\partial b} = a$|
|Power|$f = a^n$|$\frac{\partial f}{\partial a} = n \cdot a^{n-1}$|
|tanh|$f = \tanh(a)$|$\frac{\partial f}{\partial a} = 1 - f^2$|
|ReLU|$f = \max(0, a)$|$\frac{\partial f}{\partial a} = 1$ if $a > 0$ else $0$|
|exp|$f = e^a$|$\frac{\partial f}{\partial a} = f$|

Notice several operations reuse the _output_ value (tanh, exp) — that's exactly why the forward pass caches everything.

### Walked example

Same graph as the forward pass. Cached values: $e=-6.0,; d=4.0,; L=-8.0$.

**Step 1 — Seed:** $\frac{\partial L}{\partial L} = 1$

**Step 2 — At the `*` node ($L = d \cdot f$):** local derivatives are $f$ and $d$. Incoming grad = 1.

- $\frac{\partial L}{\partial d} = 1 \cdot f = -2.0$
- $\frac{\partial L}{\partial f} = 1 \cdot d = 4.0$

**Step 3 — At the `+` node ($d = e + c$):** local derivatives are 1 and 1. Incoming grad = $-2.0$ (from $d$).

- $\frac{\partial L}{\partial e} = -2.0 \cdot 1 = -2.0$
- $\frac{\partial L}{\partial c} = -2.0 \cdot 1 = -2.0$

**Step 4 — At the `*` node ($e = a \cdot b$):** local derivatives are $b$ and $a$. Incoming grad = $-2.0$ (from $e$).

- $\frac{\partial L}{\partial a} = -2.0 \cdot b = -2.0 \cdot -3.0 = 6.0$
- $\frac{\partial L}{\partial b} = -2.0 \cdot a = -2.0 \cdot 2.0 = -4.0$

**Final gradients:**

|Node|Value|Grad|
|---|---|---|
|L|-8.0|1.0|
|d|4.0|-2.0|
|f|-2.0|4.0|
|e|-6.0|-2.0|
|c|10.0|-2.0|
|a|2.0|6.0|
|b|-3.0|-4.0|

**Interpretation:** nudging $a$ up by a tiny amount changes $L$ six times as fast as the nudge (and in the same direction). $b$ has the same magnitude of influence but opposite sign. $c$, $e$, and $d$ all have weaker influence.

## Why Backprop Is Efficient

The trick is **reusing partial work**. Once you've computed $\frac{\partial L}{\partial d}$, you can compute $\frac{\partial L}{\partial e}$ and $\frac{\partial L}{\partial c}$ without recomputing anything upstream. Information flows backward through the graph the same way values flowed forward — one pass, in reverse.

Cost: **one forward pass + one backward pass**, each roughly the same expense as the forward pass alone. 

## The Training Loop (where backprop fits)

Backprop is one step in a larger loop:

1. **Forward pass** — compute predictions and the loss.
2. **Backward pass (backprop)** — compute the gradient of the loss w.r.t. every parameter.
3. **Update** — nudge each parameter slightly in the **negative gradient** direction (gradient descent):

$$\text{param} \mathrel{-}= \text{learning rate} \times \text{param.grad}$$

4. **Zero gradients** — clear them before the next iteration (otherwise `+=` would keep accumulating across steps).
5. Repeat.

Backprop is steps 1–2. 