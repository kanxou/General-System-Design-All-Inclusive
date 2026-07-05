# SimHash & MinHash

# 1. Why Do We Need Them?

Large search engines crawl billions of webpages.

Many pages are:

* Exact duplicates
* Near duplicates (small edits)
* Mirror sites
* Mobile/Desktop versions
* Syndicated news
* Pages differing only by ads, timestamps or formatting

Comparing every page with every other page is computationally impossible.

To solve this, search engines compute **compact signatures (fingerprints)** that preserve document similarity.

---

# 2. Exact Duplicate vs Near Duplicate

## Exact Duplicate

Uses cryptographic hashes.

Examples:

* MD5
* SHA-256

Same content

↓

Same hash

Even one character changes

↓

Completely different hash

Useful only for **exact duplicates**.

---

## Near Duplicate

Need algorithms where

Small document change

↓

Small signature change

Two major algorithms:

* SimHash
* MinHash

Both belong to the **Locality Sensitive Hashing (LSH)** family.

---

# 3. Locality Sensitive Hashing (LSH)

LSH algorithms are designed so that:

> Similar inputs produce similar outputs.

Unlike MD5 or SHA,

LSH preserves similarity.

Examples

* SimHash
* MinHash

---

# 4. Important Terminology

## Token

Smallest unit extracted from text.

Example

```text
The quick brown fox
```

Tokens

```text
The
quick
brown
fox
```

---

## Feature

Any information extracted from a document.

Possible features

* Words
* Word n-grams
* Character n-grams
* Shingles
* HTML elements
* URLs
* Metadata

Algorithms compare features—not raw HTML.

---

## N-Gram

A contiguous sequence of **N consecutive tokens**.

Examples

Unigram

```text
machine
```

Bigram

```text
machine learning
```

Trigram

```text
deep machine learning
```

Can also be character-based.

---

## Shingle

A shingle is simply an **n-gram used for document similarity**.

Difference

* N-gram → NLP terminology
* Shingle → Information Retrieval terminology

Every shingle is an n-gram.

Not every n-gram is referred to as a shingle.

---

# 5. Why Use N-Grams?

Words alone lose context.

Example

Words

```text
New
York
```

Phrase

```text
New York
```

Bigrams/trigrams preserve relationships between words.

Trade-off

Small n

* More robust
* Less context

Large n

* Better context
* More sensitive to edits

---

# SIMHASH

# 6. What is SimHash?

SimHash is a Locality Sensitive Hashing algorithm that approximates **Cosine Similarity**.

Output

Typically

* 64 bits
* 128 bits

Documents with similar content generate fingerprints with small Hamming Distance.

---

# 7. Core Idea

Represent every document as a **weighted feature vector**.

Example vocabulary

```text
apple
banana
orange
grape
mango
```

Document

```text
apple apple banana orange
```

Vector

```text
[2,1,1,0,0]
```

Each dimension represents one feature.

The vector can be viewed as an arrow in high-dimensional space.

---

# 8. Cosine Similarity

Measures

> Similarity between vector directions.

Formula

Cos(A,B)

=

(A · B)

/

(|A||B|)

Where

* A·B = Dot Product
* |A| = Magnitude

---

## Why Angle?

Document size should not affect similarity.

Vectors

```text
[1,1,1]

[2,2,2]
```

Represent the same topic.

Their lengths differ.

Their directions are identical.

Cosine similarity = 1.

Thus cosine measures **feature distribution**, not document length.

---

# 9. Dot Product

For vectors

```text
A=[a1,a2,...]

B=[b1,b2,...]
```

Dot product

Σ(ai × bi)

Each dimension is compared with the **same dimension**.

Apple compares with Apple.

Banana compares with Banana.

Never Apple with Banana.

---

# 10. Why Not Compute Cosine Directly?

Real document vectors may have millions of dimensions.

Comparing them directly for billions of pages is extremely expensive.

Need compression.

---

# 11. SimHash Algorithm

## Step 1

Extract features.

Possible choices

* Words
* Word shingles
* Character shingles

---

## Step 2

Assign weights.

Usually

* Frequency
* TF-IDF

---

## Step 3

Hash every feature individually.

Example

```
hash(feature)

↓

64-bit value
```

Common hash functions

* MurmurHash
* xxHash
* FarmHash
* CityHash

Important

The **entire document vector is NOT hashed**.

Every feature is hashed separately.

---

## Step 4

Create Score Vector.

For 64-bit SimHash

```
[0,0,0,...64 positions]
```

Each position stores an integer vote.

This is NOT the feature vector.

---

## Step 5

Weighted Voting

For every feature hash

Bit = 1

↓

+weight

Bit = 0

↓

-weight

Repeat for every feature.

---

## Step 6

Generate Fingerprint

Positive score

↓

1

Negative score

↓

0

Final result

64-bit fingerprint.

---

# 12. Why SimHash Works

Each hash bit behaves like a **random hyperplane** in the original high-dimensional feature space.

For each hyperplane,

the algorithm asks:

> Which side of the hyperplane does the document vector lie on?

One side

↓

Bit = 1

Other side

↓

Bit = 0

Repeating this 64 or 128 times creates the fingerprint.

Documents with small angles between their vectors usually fall on the same side of most hyperplanes.

Therefore

Small cosine angle

↓

Small Hamming Distance.

---

# 13. Comparing SimHashes

Use **Hamming Distance**.

Definition

Number of differing bits.

Lower distance

↓

Higher similarity.

Typical 64-bit thresholds

0–3

Almost identical

4–6

Very similar

7–10

Possibly related

Above 10

Usually different

Threshold depends on the application.

---

# MINHASH

# 14. What is MinHash?

MinHash is a Locality Sensitive Hashing algorithm that approximates **Jaccard Similarity**.

Unlike SimHash,

MinHash ignores frequency.

Documents are treated as **sets**.

---

# 15. Jaccard Similarity

Definition

Intersection

/

Union

Formula

|A ∩ B|

/

|A ∪ B|

Measures

> Percentage of common unique elements.

---

# 16. Why Not Compute Jaccard Directly?

Suppose

Each document contains

20,000 shingles.

Computing

Intersection

*

Union

for billions of documents is too expensive.

Need compression.

---

# 17. MinHash Algorithm

## Step 1

Generate shingles.

Usually

* Word shingles
* Character shingles

Convert document into a **set**.

Duplicate shingles are ignored.

---

## Step 2

Apply many independent hash functions.

Examples

h1

h2

h3

...

hK

---

## Step 3

For every hash function

Hash every shingle.

Keep only

The **minimum hash value**.

---

## Step 4

Repeat

for all hash functions.

If

K = 100

Signature

```
100 integers
```

instead of

20,000 shingles.

---

# 18. Why Minimum?

Think of each hash function as producing a random ordering of all shingles.

The minimum hash corresponds to the first shingle in that random order.

Beautiful theorem

Probability

(Minimums match)

=

Jaccard Similarity.

One hash function

↓

One random experiment.

Many hash functions

↓

Reliable estimate.

---

# 19. Estimating Similarity

Suppose

100 hash functions.

Document A

100 values.

Document B

100 values.

If

82 positions match

Estimated Jaccard

=

82%

No intersection or union is computed during comparison.

---

# 20. Choosing Number of Hash Functions

Trade-off

More hash functions

* Better estimate
* More CPU
* More memory

Typical production values

100–200

Common choice

128

Large systems often derive many virtual hash functions from one or two base hashes.

---

# CONTENT EXTRACTION

Before SimHash or MinHash,

search engines first remove boilerplate.

Downloaded HTML contains

* Navigation
* Header
* Footer
* Ads
* Sidebars
* Cookie banners

Only the main content should be compared.

Methods

* HTML semantic tags (<article>, <main>)
* DOM tree analysis
* Text density
* Link density
* Repeated page structure
* Machine learning

Pipeline

Download HTML

↓

Parse DOM

↓

Remove boilerplate

↓

Extract main content

↓

Generate features

↓

SimHash / MinHash

---

# DISTRIBUTED WEB CRAWLER

Crawler nodes do NOT compare pages directly.

Pipeline

Crawler Node

↓

Download page

↓

Extract content

↓

Compute fingerprint

↓

Send fingerprint to distributed duplicate index

↓

Duplicate index compares fingerprints

↓

Duplicate?

YES

Skip or keep canonical page.

NO

Index page.

---

# SIMHASH vs MINHASH

| Aspect            | SimHash                      | MinHash                                |
| ----------------- | ---------------------------- | -------------------------------------- |
| Input             | Weighted feature vector      | Set of shingles/features               |
| Frequency matters | Yes                          | No                                     |
| Preserves         | Cosine Similarity            | Jaccard Similarity                     |
| Output            | One 64/128-bit fingerprint   | Signature of K minimum hash values     |
| Comparison        | Hamming Distance             | Fraction of matching signature entries |
| Best for          | Weighted document similarity | Set overlap                            |

---

# Complexity

## SimHash

Time

O(Number of Features)

Space

O(Hash Length)

---

## MinHash

Time

O(Number of Shingles × Number of Hash Functions)

Space

O(Number of Hash Functions)

---

# End-to-End Pipeline

```
Discover URL

↓

Download HTML

↓

Parse DOM

↓

Remove Boilerplate

↓

Extract Main Content

↓

Normalize Text

↓

Generate Features/Shingles

↓

Compute SimHash or MinHash

↓

Send Fingerprint to Distributed Duplicate Index

↓

Find Similar Fingerprints

↓

Duplicate?

YES → Skip / Canonicalize

NO → Store and Index
```

---

# When Should You Use Which?

Use **SimHash** when:

* Feature frequencies matter.
* You want to preserve document semantics approximately.
* You need a very compact fingerprint.
* Comparing massive numbers of documents efficiently.

Use **MinHash** when:

* You care about overlap of unique shingles.
* Frequency is irrelevant.
* Estimating Jaccard similarity is sufficient.
* Working with sets rather than weighted vectors.

---

# Interview Cheat Sheet

## SimHash

* Locality Sensitive Hashing (LSH)
* Approximates Cosine Similarity
* Uses weighted features
* Hashes each feature individually
* Uses weighted bit voting
* Produces one 64/128-bit fingerprint
* Compared using Hamming Distance

## MinHash

* Locality Sensitive Hashing (LSH)
* Approximates Jaccard Similarity
* Uses sets of shingles
* Ignores frequency
* Applies many hash functions
* Stores minimum hash from each
* Compared by signature agreement

## One-Line Difference

**SimHash compresses a weighted feature vector so that Hamming Distance approximates Cosine Similarity.**

**MinHash compresses a set of shingles so that matching signature values approximate Jaccard Similarity.**
