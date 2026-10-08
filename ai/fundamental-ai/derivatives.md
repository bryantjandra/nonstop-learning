
## Key Idea

A derivative answers: _when I nudge the input a tiny bit, how much does the output change, and in which direction?_ It measures how sensitive an output is to a small change in the input.

## Terminology

- $f$ = the function (e.g. $f(x) = 3x^2 - 4x + 5$)
- $x$ = the input
- $h$ = a tiny nudge
- **slope / derivative** = $\frac{f(x+h) - f(x)}{h}$

As $h$ shrinks toward zero, this ratio settles on one number — the derivative at that point.

**Sign (+/−)**

- Positive → output rises when input rises
- Negative → output falls when input rises
- Zero → flat spot (e.g. bottom of a parabola)

**Magnitude**

- Large → output reacts strongly to nudges
- Small → output ba rely reacts

## Two Ways to Compute Derivatives

### 1. Numerical (the "nudge" method)

$$\text{slope} \approx \frac{f(x+h) - f(x)}{h}$$

This is the _approximation_. The exact definition takes the limit:

$$f'(x) = \lim_{h \to 0} \frac{f(x+h) - f(x)}{h}$$

Useful as a **sanity check**, not for real training (slow + has floating-point noise from $h \neq 0$).

### 2. Analytical (using rules)

|Rule|Statement|
|---|---|
|Constant|$\frac{d}{dx}[c] = 0$|
|Power|$\frac{d}{dx}[x^n] = n \cdot x^{n-1}$|
|Constant multiple|$\frac{d}{dx}[c \cdot f(x)] = c \cdot f'(x)$|
|Sum|$\frac{d}{dx}[f + g] = f' + g'$|
|Product|$\frac{d}{dx}[f \cdot g] = f'g + fg'$|

This is **exact** and what neural net training actually uses (chained together via the chain rule).

## Example: Single Input

$f(x) = 3x^2 - 4x + 5$, find slope at $x = 3$.

**Numerical** (with $h = 0.0001$):

- $f(3) = 3(9) - 12 + 5 = 20$
- $f(3.0001) \approx 20.0014$
- slope $\approx \frac{20.0014 - 20}{0.0001} = 14.0003$

**Analytical:**

- $f'(x) = 6x - 4$
- $f'(3) = 18 - 4 = 14$ ✓

The `.0003` was just noise — the true derivative is 14.

## Example: Multiple Inputs (Partial Derivatives)

When a function has several inputs, each gets its own derivative. To find the derivative w.r.t. one input, nudge only that input and hold the others fixed. This is a **partial derivative**, written $\frac{\partial f}{\partial x}$.

For $d = a \cdot b + c$:

$$\frac{\partial d}{\partial a} = b \qquad \frac{\partial d}{\partial b} = a \qquad \frac{\partial d}{\partial c} = 1$$

Each input influences the output differently:

- nudging $a$ → output changes by an amount governed by $b$
- nudging $b$ → output changes by an amount governed by $a$
- nudging $c$ → output changes by 1

**Code pattern (numerical check):**

```python
d1 = a*b + c
a += h
d2 = a*b + c
slope_a = (d2 - d1) / h   # should ≈ b
```



## The Chain Rule

The engine of backpropagation.

### Setup

When functions are composed — output of one feeds into the next:

$$y = f(g(x))$$

Let $u = g(x)$, so $y = f(u)$.

### Statement

$$\frac{dy}{dx} = \frac{dy}{du} \cdot \frac{du}{dx}$$

### Intuition

A chain of conversion rates. If $u$ changes 3× as fast as $x$, and $y$ changes 5× as fast as $u$, then $y$ changes $3 \cdot 5 = 15$× as fast as $x$.

### Worked Example

$y = (3x + 1)^2$. Let $u = 3x + 1$, so $y = u^2$.

- $\frac{du}{dx} = 3$
- $\frac{dy}{du} = 2u = 2(3x+1)$
- $\frac{dy}{dx} = 2(3x+1) \cdot 3 = 6(3x+1)$

**Verify** by expanding first: $y = 9x^2 + 6x + 1$, so $\frac{dy}{dx} = 18x + 6 = 6(3x+1)$ ✓