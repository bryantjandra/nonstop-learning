## What is a normal distribution?

Imagine you measure the heights of 10,000 people. Most people are somewhere around average. Very few people are extremely tall or extremely short. If you plot how many people fall into each height range, you get a bell curve — tall in the middle, tapering off symmetrically on both sides.

That shape is a **normal distribution** (also called Gaussian). It shows up everywhere in nature — heights, weights, test scores, measurement errors — because it's what you naturally get when many small independent factors add up.

---

## What is mean?

The **mean** is just the average — where the center of the bell sits.

```
mean = 0          mean = 5
    ▲                   ▲
    │                   │
────┼────        ────────────────
   0                   5
```

Same shape, just shifted left or right.

---

## What is standard deviation (std)?

Std measures **how spread out** the values are — how wide or narrow the bell is.

Think of it as the "typical distance from the center." If std = 1, most values are within 1 unit of the mean. If std = 3, most values are within 3 units.

```
std = 1 (narrow)      std = 3 (wide)

     ████                  ██
    ██████               ██████
   ████████           ██████████
──────────────    ──────────────────
  -3  0  3          -9    0    9
```

Same area under the curve (probabilities always sum to 1) — but one is tightly packed, the other is spread out.

**A concrete example:** exam scores with mean=70, std=10 means:

- ~68% of students scored between 60 and 80 (within 1 std of mean)
- ~95% scored between 50 and 90 (within 2 std)
- ~99.7% scored between 40 and 100 (within 3 std)

This 68-95-99.7 rule holds for **any** normal distribution regardless of mean or std.

---

## Standard normal distribution

`torch.randn` specifically samples from the **standard** normal — the special case where:

- Mean = 0 (centered at zero)
- Std = 1 (most values between -3 and +3)

This is the left histogram in Karpathy's plot. Tight bell, centered at 0.

---

## What happens after matrix multiplication

```python
x = torch.randn(1000, 10)   # std = 1.0
w = torch.randn(10, 200)    # std = 1.0
y = x @ w                   # std = 3.15  ← grew!
```

Each value in `y` is computed as:

```
y = x[0]*w[0] + x[1]*w[1] + x[2]*w[2] + ... + x[9]*w[9]
```

You're summing 10 products. Each individual product is still centered at 0, but **when you add multiple random values together, their spreads accumulate.** Mathematically:

$$\text{std}_{output} = \sqrt{n} \times \text{std}_{input} = \sqrt{10} \approx 3.16$$

The bell curve shape is preserved (still Gaussian — right histogram still looks like a bell) but it's now 3× wider. The values that were mostly between -3 and +3 are now mostly between -9 and +9.

---

## Why this is a problem for neural networks

One matrix multiply: std 1.0 → 3.16

Two layers: 3.16 → 10.0

Three layers: 10.0 → 31.6

By the time you're 10 layers deep the typical activation value is in the **tens of thousands**. Feed that into tanh and you get ±1 every single time — fully saturated, zero gradient, nothing learns.

---

## The fix

Scale the weights down by $\frac{1}{\sqrt{n}}$ where $n$ is the number of inputs:

```python
w = torch.randn(10, 200) * (1 / 10**0.5)
#                          ↑ ≈ 0.316
```

Now each product is smaller, and when you sum 10 of them the std stays ≈ 1.0 instead of growing to 3.16. The bell coming out of the layer looks the same as the bell going in and this stays true no matter how many layers deep you go.

This is the core idea behind **Kaiming initialization**: calculate exactly the right scale factor per layer so the std stays at 1 all the way through the network.