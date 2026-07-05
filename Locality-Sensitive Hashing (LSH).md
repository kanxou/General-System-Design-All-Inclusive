# SimHash -

## Important: SimHash Uses Two Different Vectors

This is the most common source of confusion.

There are **two completely different vectors** involved in SimHash.

They serve different purposes.

---

# 1. Feature Vector (Conceptual)

The **Feature Vector** is the mathematical representation of a document.

Its purpose is to explain **Cosine Similarity**.

Suppose the vocabulary is:

```text
apple
banana
orange
grape
mango
```

Document:

```text
apple apple banana orange
```

Feature weights (frequency):

| Feature | Weight |
| ------- | -----: |
| apple   |      2 |
| banana  |      1 |
| orange  |      1 |
| grape   |      0 |
| mango   |      0 |

Feature Vector:

```text
[2,1,1,0,0]
```

Each dimension corresponds to one feature.

This vector can be viewed as an arrow in a high-dimensional space.

Cosine Similarity compares the directions of these feature vectors.

**Important**

The Feature Vector is a **conceptual representation** used to explain cosine similarity.

A SimHash implementation does **not** have to explicitly build or store this vector.

---

# 2. Score Vector (Actual SimHash)

The **Score Vector** is the actual data structure used by the SimHash algorithm.

Suppose we are generating a 64-bit SimHash.

Initially:

```text
[0,0,0,0,...64 positions]
```

Each position corresponds to one bit in the final fingerprint.

Each position stores an integer vote.

---

# SimHash Algorithm (Correct Version)

## Step 1

Extract features.

Possible features:

* Words
* Word shingles
* Character shingles

---

## Step 2

Assign a weight to every feature.

Weights may be:

* Frequency
* TF-IDF
* Custom importance

Example:

```text
apple  = 2
banana = 1
orange = 1
```

These weights are conceptually equivalent to the Feature Vector, but the algorithm only needs the individual feature weights.

---

## Step 3

Hash each feature independently.

Example:

```text
hash(apple)
↓

10101100
```

```text
hash(banana)
↓

11001001
```

```text
hash(orange)
↓

00101111
```

The entire document is **never hashed**.

The Feature Vector is **never hashed**.

Only individual features are hashed.

---

## Step 4

Create the Score Vector.

For an 8-bit example:

```text
[0,0,0,0,0,0,0,0]
```

---

## Step 5

Update the Score Vector.

Process one feature at a time.

Example:

Feature

```text
apple
```

Weight

```text
2
```

Hash

```text
10101100
```

Rule:

Bit = 1

↓

Add the feature weight.

Bit = 0

↓

Subtract the feature weight.

Contribution:

```text
[+2,-2,+2,-2,+2,+2,-2,-2]
```

Add this contribution to the Score Vector.

Repeat the same process for every feature.

---

## Step 6

Generate the Final Fingerprint.

After processing every feature, suppose the Score Vector becomes:

```text
[3,-1,2,-2,4,1,-1,0]
```

Convert every position:

Positive

↓

1

Negative

↓

0

Zero

↓

Typically 1 (implementation-dependent)

Final SimHash:

```text
10101101
```

---

# Relationship Between the Two Vectors

The two vectors have completely different purposes.

### Feature Vector

Example:

```text
[2,1,1,0,0]
```

Purpose:

Represents the document mathematically.

Used to define Cosine Similarity.

One dimension per feature.

Usually conceptual.

---

### Score Vector

Example:

```text
[3,-1,2,-2,4,...]
```

Purpose:

Intermediate accumulator during SimHash computation.

One position per fingerprint bit.

Always constructed by the algorithm.

---

# Mental Model

```
Document
     │
     ▼
Extract Features
     │
     ▼
Feature Weights
(apple=2, banana=1, orange=1)
     │
     ├──────────────► Conceptually forms the Feature Vector
     │                     [2,1,1,0,0]
     │
     │                     Used to explain
     │                     Cosine Similarity
     │
     ▼
Hash Each Feature
     │
     ▼
Weighted Bit Voting
     │
     ▼
Score Vector
[3,-1,2,-2,4,...]
     │
     ▼
Convert Signs
     │
     ▼
64-bit SimHash Fingerprint
```

---

# Interview Clarification

If asked:

> "Does SimHash first build the Feature Vector?"

A strong answer is:

> "Not necessarily. The Feature Vector is the mathematical representation of the document used to explain cosine similarity. In practice, SimHash streams through the extracted features one at a time. For each feature, it uses the feature's weight, hashes that feature, and immediately updates the Score Vector. The Feature Vector does not need to be explicitly materialized."

---

# Summary Table

| Feature Vector                    | Score Vector                                |
| --------------------------------- | ------------------------------------------- |
| Represents the document           | Intermediate data structure used by SimHash |
| One dimension per feature         | One position per hash bit (64/128)          |
| Used to explain cosine similarity | Used to generate the fingerprint            |
| Usually conceptual                | Actually created by the algorithm           |
| Example: `[2,1,1,0,0]`            | Example: `[3,-1,2,-2,4,...]`                |
| Not hashed                        | Built using hashes of individual features   |
