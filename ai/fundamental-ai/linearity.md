## Linear Classifier

A linear classifier makes predictions by computing a **weighted sum of the inputs**. In matrix form:

$$\text{output} = xW$$

Every output is just a straight linear combination of the inputs. 

The name comes from geometry — the decision boundaries it can draw between classes are always **straight lines** (or flat hyperplanes in higher dimensions). It can only separate data that is **linearly separable**.

---

## Why Non-Linearity is Needed

Consider a deeper network with two weight matrices W1 and W2. Without non-linearity between them:

$$\text{output} = (xW_1)W_2 = x(W_1W_2) = xW_3$$

Two linear layers collapse into one. We can stack as many layers as we want and they're mathematically equivalent to a single linear layer. 

Real-world data like images, language, and audio, has deeply non-linear structure. A straight line cannot separate a cat from a dog in pixel space. We need curved, complex decision boundaries (example below for XOR).

![[XORNonLinearity.png]]

---

## Introducing Non-Linearity

We insert an **activation function** between layers and that is a simple non-linear function applied element-wise:

```python
# without non-linearity 
out = x @ W1 @ W2

# with non-linearity 
out = torch.tanh(x @ W1) @ W2
```


Common activation functions:

|Function|Output range|Used where|
|---|---|---|
|`tanh`|(-1, 1)|micrograd, older nets|
|`ReLU`|(0, ∞)|most modern networks|
|`sigmoid`|(0, 1)|binary outputs|
|`softmax`|(0, 1), sums to 1|final layer for classification|
