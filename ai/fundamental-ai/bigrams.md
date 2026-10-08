## What is a Bigram?

A bigram is a consecutive pair of characters. A bigram model predicts the next character based solely on the current character. It's the simplest possible language model. 

For the name `emma`:
``` 
. → e
e → m
m → m
m → a
a → .
```

The `.` is a special character that marks the start and end of every word. When we sample `.` as the next character, the name is done. 

This bigram model is essentially learning: **given the current character, how likely is each possible next character?**

---

## Building the Count Matrix

```python
N = torch.zeros((27, 27), dtype=torch.int32)

chars = sorted(list(set("".join(words))))

stoi = {s:i+1 for i,s in enumerate(chars)}

stoi["."] = 0

itos = {i:s for s, i in stoi.items()}
```

- `N` is a 27x27 matrix (26 letters and the `.` token). `N[i,j]` stores how many times character `i` was followed by character `j` across the entire dataset. 
- `stoi` (string → index) maps each character to an integer. `.` gets index `0`; `a–z` get `1–26`.
- `itos` (index → string) is just the reverse lookup.

```python
for w in words:
    chs = ['.'] + list(w) + ['.']
    for ch1, ch2 in zip(chs, chs[1:]):
        ix1 = stoi[ch1]
        ix2 = stoi[ch2]
        N[ix1, ix2] += 1
```
 
For every word, we wrap it in `.` tokens and slide a window of two characters across it. Each pair increments the corresponding cell in `N`. After this loop, `N` is a complete frequency table of every character transition in the dataset.

---

## Converting Counts to Probabilities
 
```python
P = (N+1).float()
P /= P.sum(1, keepdim=True)
```
 
We want each row of `N` to become a probability distribution — the probability of each next character given the current one.
 
**`+1` — smoothing:** Adding 1 to every count before dividing ensures no probability is ever exactly `0`. A zero probability would mean a bigram is "impossible", which would break the log likelihood calculation later (log(0) = -∞). This is called **Laplace smoothing**.
 
**`.float()`:** Converts the integer counts to floats so division works correctly.
 
**Row normalization:** `P.sum(1, keepdim=True)` sums each row, producing a 27×1 tensor. Dividing the 27×27 matrix by this 27×1 tensor broadcasts across columns — every row gets divided by its own total, so each row sums to 1.
 
After this, `P[i]` is a valid probability distribution over all 27 possible next characters when the current character is `i`.

A **probability distribution** is just a list of numbers where:
- Every number is between 0 and 1
- They all add up to exactly 1

So `P[i]` is a row of 27 numbers, each representing the probability that character `j` follows character `i`, and all 27 of them sum to 1.

For example, `P[0]` (the `.` row) might look like:

```
a: 0.08,  b: 0.03,  c: 0.05,  ...  z: 0.01
```

These are the probabilities of each letter appearing as the **first letter** of a name. They all sum to 1 because *something* has to come next and one of those 27 characters will always be chosen.
 
---

```python
g = torch.Generator().manual_seed(214748367)
for i in range(5):
	out = []
	ix = 0
	while True:
		p = P[ix] # p is the probability distribution for the current character P[ix]
		ix = torch.multinomial(p, num_samples=1, replacement=True, generator=g).item()
		out.append(itos[ix])
		if ix == 0:
			break
	print("".join(out))
```

- Start at index `0` (the `.` token). 
- At each step, we look up the probability distribution for the current character `P[ix]`.
-  `torch.multinomial` samples one index from that distribution and characters with higher probability get sampled more often. 
- Append the sampled character to the output, then repeat. 
- We stop when `.` (index `0`) is sampled as that marks the end of the name. 

NOTE: `torch.Generator().manual_seed()` fixes the random seed so sampling is reproducible.

---
## Loss: Negative Log Likelihood
 
```python
log_likelihood = 0.0
n = 0
for w in words:
    chs = ['.'] + list(w) + ['.']
    for ch1, ch2 in zip(chs, chs[1:]):
        ix1 = stoi[ch1]
        ix2 = stoi[ch2]
        prob = P[ix1, ix2]
        logprob = torch.log(prob)
        log_likelihood += logprob
        n += 1
 
nll = -log_likelihood
print(f"{nll/n=}")
```
 
### The goal
 
We want to **maximize the likelihood** of the training data under the model. For a word like `emma`, the likelihood is the product of all its bigram probabilities:
 
$$P(\text{emma}) = P(e|.) \times P(m|e) \times P(m|m) \times P(a|m) \times P(.|a)$$
 
Maximizing the product of many small probabilities across the entire dataset is numerically unstable (numbers get tiny fast). So we take the **log**:
 
$$\log P(\text{emma}) = \log P(e|.) + \log P(m|e) + \log P(m|m) + \log P(a|m) + \log P(.|a)$$
 
Since `log` is monotonically increasing, **maximizing log likelihood is equivalent to maximizing likelihood**, but it's just more numerically stable (products become sums).
 
By convention, ML minimizes losses rather than maximizes objectives, so we negate it:
 
$$\text{NLL} = -\log \text{likelihood}$$
 
Dividing by `n` gives the **average NLL per character** — a single number that measures model quality regardless of dataset size. Lower = better.
 
| Equivalent goals | |
|---|---|
| Maximize likelihood | $\prod P(c_{i+1} \| c_i)$ as large as possible |
| Maximize log likelihood | $\sum \log P(c_{i+1} \| c_i)$ as large as possible |
| Minimize negative log likelihood | $-\sum \log P(c_{i+1} \| c_i)$ as small as possible |
| Minimize average NLL | divide by $n$ to normalise |
 
A **perfect model** would assign probability 1 to every correct next character → log(1) = 0 → NLL = 0. 
A **bad model** assigns low probabilities → log of a small number → large negative → high NLL.