Underfitting occurs when our model is too simple to capture the patterns in the data. It performs poorly on both training and validation sets. Thus, it hasn't even learned the training data properly. 

## Mitigation Strategies #1: Making the Network More Complex

```python
# before
W1 = torch.randn((6,100)) # 100 hidden neurons


# after
W1 = torch.randn((30, 200)) # 200 hidden neurons
```

More neurons = more parameters = more capacity to learn complex patterns. The model literally cannot represent the function that is mapping the inputs to outputs, and so, adding neurons gives it the room learn more nuanced relationships between certain characters. 

## Mitigation Strategies #1: Making the Embedding Space Larger

```python
# before
C = torch.randn((27, 2))   # 2D embeddings

# after
C = torch.randn((27, 10))  # 10D embeddings
```

With only small dimensional embeddings like 2D, we are forcing each input to be described by just 2 numbers. 

For example, the model may want to encode that `e` is a vowel, appears at the start of names, and often follows certain consonants, etc. BUT it has no room to store all that information. 

With a higher embedding space like 10D, each input gets 10 numbers and the model is able to encode multiple independent properties simultaneously. Similar inputs or similar characters can be close in some dimensions while being different in others. 

The richer representation gives the rest of the network better information to work with.

