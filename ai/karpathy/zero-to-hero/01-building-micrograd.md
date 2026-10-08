## What is Micrograd?

Micrograd is a tiny scalar-valued (every `Value` object holds a single number) autograd engine. It is a minimal implementation of the core mechanism behind every deep learning framework (PyTorch, Tensorflow, JAX). 

It essentially does two things:

1. Builds a computation graph as you do math on `Value` objects. 
2. Runs backpropagation through that graph in order to compute each node's gradients.

Frameworks like PyTorch operate on tensors, but the underlying graph structure and math is identical.

--- 

## The Value Object

The `Value` object wraps a single number and it also knows where this number came from, what its gradient is, and how to propagate that gradient backward. 

**Fields**
```python
class Value:
    def __init__(self, data, _children=(), _op="", label=""):
        self.data = data
        self._prev = set(_children)
        self._backward = lambda: None
        self._op = _op
        self.label = label
        self.grad = 0.0
```

| Field       | Purpose                                                                                                          |
| ----------- | ---------------------------------------------------------------------------------------------------------------- |
| `data`      | The actual number that this `Value` instance holds                                                               |
| `_prev`     | Parent nodes (the `Value`'s that were combined in order to produce this one)                                     |
| `_backward` | Function that computes and assigns the parents' gradients                                                        |
| `_op`       | What operation created this `Value` instance (mainly used for visualization)                                     |
| `label`     | Also mainly used for visualization                                                                               |
| `grad`      | $\frac{\partial L}{\partial \text{this Value instance}}$ - starts at 0.0, but then filled up by backpropagation. |
*Notes about gradient*:  `_backward` always uses `+=` to accumulate gradients. This is because a node (a `Value` instance) can be fed into multiple other nodes downstream, and thus each path contributes a gradient that must be summed (multivariable chain rule). Starting at `0.0` makes that accumulation possible also. 

**Complete Code**:
```python
class Value:
    def __init__(self, data, _children=(), _op="", label=""):
        self.data = data
        self._prev = set(_children)
        self._backward = lambda: None
        self._op = _op
        self.label=label
        self.grad = 0.0
    
    def __repr__(self):
        return f"Value(data={self.data})"

    def __add__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data + other.data, (self, other), "+")
        def _backward():
            self.grad += 1.0 * out.grad
            other.grad +=  1.0 * out.grad
        out._backward = _backward
        return out
    
    def __mul__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data * other.data, (self, other), "*")
        def _backward():
            self.grad += other.data * out.grad
            other.grad += self.data * out.grad
        out._backward = _backward
        return out
    
    def __pow__(self, other):
        assert isinstance(other, (int, float)), "only supporting int/float powers for now"
        out = Value(self.data**other, (self, ), f'**{other}')
        def _backward():
            self.grad += other * (self.data ** (other - 1)) * out.grad
        out._backward = _backward

        return out
    
    def __rmul__(self, other):
        return self * other
    
    def __radd__(self, other):
        return self + other

    def __truediv__(self, other):
        return self * other**-1
    
    def __neg__(self):
        return self * - 1
    
    def __sub__(self, other):
        return self + (-other)
    
    def tanh(self):
        x = self.data
        t = (math.exp(2*x) - 1)/(math.exp(2*x) + 1)
        out = Value(t, (self, ), "tanh")
        def _backward():
            self.grad += (1 - t**2) * out.grad
        out._backward = _backward
        return out

    def exp(self):
        x = self.data
        out = Value(math.exp(x), (self, ), "exp")

        def _backward():
            self.grad += out.data * out.grad
        out._backward = _backward
        return out
            

    def backward(self):
        # building a topological ordering of our nodes 
        # a certain node will essentially be in the topological order once all its childrens nodes are already inserted first.
        topo = []
        visited = set()
        def build_topo(v):
            if v not in visited:
                visited.add(v)
                for child in v._prev:
                    build_topo(child)
                topo.append(v)
        build_topo(self) 

        self.grad = 1.0
        for node in reversed(topo):
            node._backward()

a = Value(2.0, label="a")
b = Value(-3.0, label="b")
c = Value(10.0, label="c")
e = a*b; e.label="e"
d = e+c; d.label="d"
f=Value(-2.0); f.label="f"

L = d*f; L.label="L"
L
```

--- 
## Building the Computation Graph

Every time a certain operation like (`+` or `*`) is performed on `Value` objects, a new `Value` is created and the parent nodes are recorded in `_prev`. It also defines a `_backward` closure. 

The computation graph essentially builds itself silently as a side effect of doing math. 

The `_backward` pattern is always the same and it will be:

`parent.grad += (local derivative w.r.t. parent) * out.grad`

- out.grad = This is the incoming gradient from above (chain rule multiplier)
- local derivative = how this specific operation (e.g. `+` or `*`) affects this specific parent. 

### Addition `__add__`

```python
def __add__(self, other):
    other = other if isinstance(other, Value) else Value(other)
    out = Value(self.data + other.data, (self, other), "+")
    def _backward():
        self.grad += 1.0 * out.grad
        other.grad += 1.0 * out.grad
    out._backward = _backward
    return out
```

$$\frac{\partial(a+b)}{\partial a} = 1 \qquad \frac{\partial(a+b)}{\partial b} = 1$$

Gradient **passes through unchanged** to both parents.

### Multiplication `__mul__`

```python
def __mul__(self, other):
    other = other if isinstance(other, Value) else Value(other)
    out = Value(self.data * other.data, (self, other), "*")
    def _backward():
        self.grad += other.data * out.grad
        other.grad += self.data * out.grad
    out._backward = _backward
    return out
```

$$\frac{\partial(a \cdot b)}{\partial a} = b \qquad \frac{\partial(a \cdot b)}{\partial b} = a$$

Each parent's local derivative is the **other parent's value** — they swap.

### Power `__pow__`

```python
def __pow__(self, other):
    out = Value(self.data**other, (self,), f'**{other}')
    def _backward():
        self.grad += other * (self.data ** (other - 1)) * out.grad
    out._backward = _backward
    return out
```

$$\frac{\partial(x^n)}{\partial x} = n \cdot x^{n-1}$$

### tanh

```python
def tanh(self):
    x = self.data
    t = (math.exp(2*x) - 1) / (math.exp(2*x) + 1)
    out = Value(t, (self,), "tanh")
    def _backward():
        self.grad += (1 - t**2) * out.grad
    out._backward = _backward
    return out
```

$$\frac{\partial \tanh(x)}{\partial x} = 1 - \tanh(x)^2$$

Reuses `t` (the cached output) — exactly why the forward pass stores intermediate values.

### exp

```python
def exp(self):
    x = self.data
    out = Value(math.exp(x), (self,), "exp")
    def _backward():
        self.grad += out.data * out.grad
    out._backward = _backward
    return out
```

$$\frac{\partial e^x}{\partial x} = e^x = \text{out.data}$$

The derivative of $e^x$ is itself — so it simply reuses the already-computed output.

### Convenience operations (no new `_backward` needed)

These express themselves in terms of `*`, `+`, `**` — so they automatically inherit the correct backward behavior:

```python
def __rmul__(self, other): return self * other        # 2 * Value  (int on left)
def __radd__(self, other): return self + other        # 2 + Value  (needed for sum())
def __truediv__(self, other): return self * other**-1 # a/b = a * b^(-1)
def __neg__(self): return self * -1                   # -a = a * -1
def __sub__(self, other): return self + (-other)      # a-b = a + (-b)
```

> **`__rmul__` / `__radd__` pattern:** Python tries `int.__add__(Value)` first. When that fails (int doesn't know about Value), it falls back to `Value.__radd__(int)`. Without these, expressions like `2 * a` or `sum(...)` crash.

> **`isinstance` check in operations:** `other = other if isinstance(other, Value) else Value(other)` — this lets you write `Value(2.0) + 3` where `3` is a plain Python number. It wraps raw numbers in `Value` automatically.

---

## Manual Backpropagation

Building the graph:

```python
a = Value(2.0);  b = Value(-3.0)
c = Value(10.0); f = Value(-2.0)
e = a * b   # e = -6.0
d = e + c   # d =  4.0
L = d * f   # L = -8.0
```

```
a ─┐
   ├─[*]─→ e ─┐
b ─┘          ├─[+]─→ d ─┐
        c ────┘          ├─[*]─→ L
                   f ────┘
```

Applying the chain rule manually at each node, right to left:

**Seed:** $\frac{\partial L}{\partial L} = 1$

**At `L = d * f`** — local derivatives swap: $$\frac{\partial L}{\partial d} = f = -2.0 \qquad \frac{\partial L}{\partial f} = d = 4.0$$

**At `d = e + c`** — gradient flows through unchanged, scaled by incoming $-2.0$: $$\frac{\partial L}{\partial e} = -2.0 \cdot 1 = -2.0 \qquad \frac{\partial L}{\partial c} = -2.0 \cdot 1 = -2.0$$

**At `e = a * b`** — local derivatives swap, scaled by incoming $-2.0$: $$\frac{\partial L}{\partial a} = -2.0 \cdot b = -2.0 \cdot -3.0 = 6.0$$ $$\frac{\partial L}{\partial b} = -2.0 \cdot a = -2.0 \cdot 2.0 = -4.0$$

**Final gradients:**

|Node|data|grad|
|---|---|---|
|L|-8.0|1.0|
|d|4.0|-2.0|
|f|-2.0|4.0|
|e|-6.0|-2.0|
|c|10.0|-2.0|
|a|2.0|6.0|
|b|-3.0|-4.0|

---

## Automated Backprop: `backward()`

Instead of computing gradients by hand, `backward()` automates the process. Two steps:

### Step 1 — Topological sort

```python
topo = []
visited = set()
def build_topo(v):
    if v not in visited:
        visited.add(v)
        for child in v._prev:
            build_topo(child)
        topo.append(v)
build_topo(self)
```

A topological sort orders the graph so that every node appears **only after all nodes it depends on**. This is critical because `_backward()` at any node needs the incoming `out.grad` to already be computed before it can push gradients to parents.

For our graph: `[a, b, e, c, d, f, L]`

Reversed: `[L, f, d, c, e, b, a]` — the correct order to run backward.

### Step 2 — Seed and sweep

```python
self.grad = 1.0
for node in reversed(topo):
    node._backward()
```

Set the output's grad to `1.0`, then iterate through every node in reverse topological order. Each node runs its own `_backward()` — the closure that was defined when that node was created by its operation (`+`, `*`, `tanh`, etc.). That closure computes `incoming grad * local derivative` and accumulates the result into each parent's `.grad`. By the time the sweep is done, every node in the graph holds the correct gradient.

--- 

## Building a Neural Network Library

This library is build on top of our `Value` class and it is 3 classes stacked on each other: `Neuron` --> `Layer` --> `MLP`.

```python
class Neuron:
    def __init__(self, nin):
        self.w = [Value(random.uniform(-1, 1)) for _ in range(nin)]
        self.b = Value(random.uniform(-1, 1))

    def __call__(self, x):
        act = sum((wi * xi for wi, xi in zip(self.w, x)), self.b)
        out = act.tanh()
        return out

    def parameters(self):
        return self.w + [self.b]
```

A single neuron computes: $\text{out} = \tanh\left(\sum_i w_i x_i +b\right)$

- `__init__`: Creates one `Value` weight per input, plus one bias, all randomly initialized between -1 and 1. 
- `__call__`: computes the weighted sum starting from `self.b`, then squashes the result through `tanh` to keep it between -1 and 1. 
- `parameters()`: returns every learnable value in this neuron (all weights and the bias). 

### Layer

```python
class Layer:
    def __init__(self, nin, nout):
        self.neurons = [Neuron(nin) for _ in range(nout)]

    def __call__(self, x):
        outs = [n(x) for n in self.neurons]
        return outs[0] if len(outs) == 1 else outs

    def parameters(self):
        return [p for neuron in self.neurons for p in neuron.parameters()]
```

A layer is just `nout` neurons all receiving the same input `x`. It is an array of neurons. 

- `__call__`: runs every neuron on `x`, collects their outputs. What we specify to `nin` is essentially the number of neurons acting as input to this layer, and what we specify to `nout` is the number of neurons in this layer.  
- E.g. Layer (3,4) builds a layer where 3 input neurons are fully connected to 4 output neurons.
- If there's only one neuron (the output layer), it unwraps the list and returns a single `Value`, otherwise the loss function would get a list instead of a `Value` and crash.
- `parameters()`: flattens all neuron parameters into one list.

### MLP (Multi-Layer Perceptron)

```python
class MLP:
    def __init__(self, nin, nouts):
        sz = [nin] + nouts
        self.layers = [Layer(sz[i], sz[i+1]) for i in range(len(nouts))]

    def __call__(self, x):
        for layer in self.layers:
            x = layer(x)
        return x

    def parameters(self):
        return [p for layer in self.layers for p in layer.parameters()]
        
n = MLP(3, [4,4,1])
```

It is an array of layers and it stacks layers end-to-end. `MLP(3, [4, 4, 1])` builds:
```text
sz = [3, 4, 4, 1]

Layer(3→4)   Layer(4→4)   Layer(4→1)
  [4 neurons]  [4 neurons]  [1 neuron]
```
Precisely, `n` is a network that takes 3 inputs, has 2 hidden layers of 4 neurons each, and outputs a single value.
- Hidden Layer 1 - 4 neurons each receiving 3 inputs.
- Hidden Layer 2 - 4 neurons each receiving 4 inputs. 
- Output Layer - 1 neuron, receiving 4 inputs.

- `__call__`: passes input through each layer in sequence. `x` gets overwritten at each step as  the output of one layer becomes the input to the next.
- `parameters()`: one flat list of every `Value` in the entire network. This is what the training loop iterates over.

---

## Training Loop

### The dataset

```python
xs = [
    [2.0, 3.0, -1.0],
    [3.0, -1.0, 0.5],
    [0.5, 1.0, 1.0],
    [1.0, 1.0, -1.0]
]
ys = [1.0, -1.0, -1.0, 1.0]  # desired outputs
```

4 training examples, each with 3 inputs. `ys` is what the network should output for each one and they are binary labels of `+1` or `-1`.

### The loss function

```python
loss = sum((yout - ygt)**2 for ygt, yout in zip(ys, ypred))
```

**Mean Squared Error (MSE)** — for each example, square the difference between the prediction and the target, then sum them all. The closer predictions are to targets, the smaller the loss.

|Prediction|Target|$(yout - ygt)^2$|
|---|---|---|
|0.9|1.0|0.01|
|-0.8|-1.0|0.04|
|0.5|-1.0|2.25|

A high loss = predictions are far off. A low loss = network is doing well. **The entire goal of training is to minimize this number.**

### The loop

```python
for k in range(20):
    # 1. forward pass
    ypred = [n(x) for x in xs]
    loss = sum((yout - ygt)**2 for ygt, yout in zip(ys, ypred))

    # 2. backward pass
    for p in n.parameters():
        p.grad = 0.0
    loss.backward()

    # 3. update
    for p in n.parameters():
        p.data += -0.05 * p.grad

    print(k, loss.data)
```

Each iteration does three things:

**1. Forward pass** — run every input through the network (`n`) and compute the loss.
**2. Backward pass** — zero all gradients first (otherwise they accumulate across steps), then call `loss.backward()` to fill in `.grad` for every parameter.
**3. Update (gradient descent)** — nudge every parameter slightly in the direction that reduces the loss:

The negative sign ensures we go _against_ the gradient. If the gradient is negative, increasing the weights will decrease our loss. If the gradient is positive, decreasing our weights will decrease our loss.

`0.05` is the learning rate and this controls the step size. Too large and training overshoots; too small and it crawls.

### Output

```
0  3.790
1  2.818
5  0.498
10 0.171
19 0.067
```

Loss drops from `3.79` → `0.067` over 20 steps. The network's predictions are getting closer to the targets with every iteration. This is the entire training process, it is essentially nothing more than repeatedly applying these three steps above.

## Related Topics

[[derivatives]]
[[backpropagation]]
