# datasketch

[![Build Status](https://travis-ci.org/ekzhu/datasketch.svg?branch=master)](https://travis-ci.org/ekzhu/datasketch)

datasketch gives you probabilistic data structures that can process
vary large amount of data super fast, with little loss of accuracy.

This package contains the following data sketches:

| Data Sketch   | Usage                                |
|---------------|--------------------------------------|
| MinHash       | estimate resemblance and cardinality |
| b-Bit MinHash | estimate resemblance                 |
| HyperLogLog   | estimate cardinality                 |
| HyperLogLog++ | estimate cardinality                 |

datasketch must be used with Python 2.7 or above and NumPy.

## Install

To install datasketch using `pip`:

    pip install datasketch -U

This will also install NumPy as dependency.

## MinHash

MinHash lets you estimate the Jaccard similarity
(resemblance)
between datasets of
arbitrary sizes in linear time using a small and fixed memory space.
It can also be used to compute Jaccard similarity between data streams.
MinHash is introduced by Andrei Z. Broder in this
[paper](http://cs.brown.edu/courses/cs253/papers/nearduplicate.pdf)

```python
from hashlib import sha1
from datasketch import MinHash

data1 = ['minhash', 'is', 'a', 'probabilistic', 'data', 'structure', 'for',
        'estimating', 'the', 'similarity', 'between', 'datasets']
data2 = ['minhash', 'is', 'a', 'probability', 'data', 'structure', 'for',
        'estimating', 'the', 'similarity', 'between', 'documents']

m1, m2 = MinHash(), MinHash()
for d in data1:
	m1.digest(sha1(d.encode('utf8')))
for d in data2:
	m2.digest(sha1(d.encode('utf8')))
print("Estimated Jaccard for data1 and data2 is", m1.jaccard(m2))

s1 = set(data1)
s2 = set(data2)
actual_jaccard = float(len(s1.intersection(s2)))/float(len(s1.union(s2)))
print("Actual Jaccard for data1 and data2 is", actual_jaccard)
```

You can adjust the accuracy by customizing the number of permutation functions
used in MinHash.

```python
# This will give better accuracy than the default setting (128).
m = MinHash(num_perm=256)
```

The trade-off for better accuracy is slower speed and higher memory usage.
Because using more permutation functions means 1) more CPU instructions
for every hash digested and 2) more hash values to be stored.
The speed and memory usage of MinHash are both linearly proportional
to the number of permutation functions used.

![MinHash Benchmark](https://github.com/ekzhu/datasketch/blob/master/minhash_benchmark.png)

You can union two MinHash object using the `merge` function.
This makes MinHash useful in parallel MapReduce style data analysis.

```python
# The makes m1 the union of m2 and the original m1.
m1.merge(m2)
```

MinHash can be used for estimating the number of distinct elements, or cardinality.
The analysis is presented in [Cohen 1994](http://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=365694).

```python
# Returns the estimation of the cardinality of
# all elements digested so far.
m.count()
```

## b-Bit MinHash

[b-Bit MinHash](http://research.microsoft.com/pubs/120078/wfc0398-liPS.pdf)
is created by Ping Li and Arnd Christian König.
It is a compression of MinHash - it stores only the lowest b-bits of each
minimum hashed values in the MinHash, allowing one to trade accuracy for
less storage cost.

When the actual Jaccard similarity, or resemblance, is large (>= 0.5),
b-Bit MinHash's estimation for Jaccard has very small loss of accuracy
comparing to the original MinHash.
On the other hand, when the actual Jaccard is small, b-Bit MinHash gives
bad estimation for Jaccard, and it tends to over-estimate.

![b-Bit MinHash Benchmark](https://github.com/ekzhu/datasketch/blob/master/b_bit_minhash_benchmark.png)

To create a b-Bit MinHash object from an existing MinHash object:

```python
from datasketch import bBitMinHash

# minhash is an existing MinHash object.
bm = bBitMinHash(minhash)
```

To estimate Jaccard similarity using two b-Bit MinHash objects:

```python
# Estimate Jaccard given bm1 and bm2, both must have the same
# value for parameter b.
bm1.jaccard(bm2)
```

The default value for parameter b is 1.
Estimation accuracy can be improved by keeping more bits -
increasing the value for parameter b.

```python
# Using higher value for b can improve accuracy, at the expense of
# using more storage space.
bm = bBitMinHash(minhash, b=4)
```

**Note:** for this implementation,
using different values for the parameter b
won't make a difference in the in-memory
size of b-Bit MinHash, as the underlying storage is a NumPy integer array.
However, the size of a serialized b-Bit MinHash is determined by the parameter
b (and of course the number of permutation functions in the original MinHash).

Because b-Bit MinHash only retains the lowest b-bits of the minimum hashed
values in the original MinHash, it is not mergable.
Thus it has no `merge` function.

## HyperLogLog

HyperLogLog is capable of estimating the cardinality (the number of
distinct values) of dataset in a single pass, using a small and fixed
memory space.
HyperLogLog is first introduced in this
[paper](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)
by Philippe Flajolet, Éric Fusy, Olivier Gandouet and Frédéric Meunier.

```python
from hashlib import sha1
from datasketch import HyperLogLog

data1 = ['hyperloglog', 'is', 'a', 'probabilistic', 'data', 'structure', 'for',
'estimating', 'the', 'cardinality', 'of', 'dataset', 'dataset', 'a']

h = HyperLogLog()
for d in data1:
  h.digest(sha1(d.encode('utf8')))
print("Estimated cardinality is", h.count())

s1 = set(data1)
print("Actual cardinality is", len(s1))
```

As in MinHash, you can also control the accuracy of HyperLogLog by changing
the parameter p.

```python
# This will give better accuracy than the default setting (8).
h = HyperLogLog(p=12)
```

Interestingly, there is no speed penalty for using higher p value.
However the memory usage is exponential to the p value.

![HyperLogLog Benchmark](https://github.com/ekzhu/datasketch/blob/master/hyperloglog_benchmark.png)

As in MinHash, you can also merge two HyperLogLogs to create a union HyperLogLog.

```python
h1 = HyperLogLog()
h2 = HyperLogLog()
h1.digest(sha1('test'.encode('utf8')))
# The makes h1 the union of h2 and the original h1.
h1.merge(h2)
# This will return the cardinality of the union
h1.count()
```

## HyperLogLog++

[HyperLogLog++](http://research.google.com/pubs/pub40671.html)
is an enhanced version of HyperLogLog by Google with the following
changes:
* Use 64-bit hash values instead of the 32-bit used by HyperLogLog
* A more stable bias correction scheme based on experiments
on many datasets
* Sparse representation (not implemented here)

HyperLogLog++ object shares the same interface as HyperLogLog.
So you can use all the HyperLogLog functions in HyperLogLog++.

```python
from datasketch import HyperLogLogPlusPlus

# Initialize an HyperLogLog++ object.
hpp = HyperLogLogPlusPlus()
# Everything else is the same as HyperLogLog
```

## Serialization

All data sketches supports efficient serialization
using Python's `pickle` module.
For example:

```python
import pickle

m1 = MinHash()
# Serialize the MinHash objects to bytes
bytes = pickle.dumps(m1)
# Reconstruct the serialized MinHash object
m2 = pickle.loads(bytes)
# m1 and m2 should be equal
print(m1 == m2)
```

Additionally, you can check the byte size of any data sketch using the
`bytesize` function.

```python
# Print out the serialized size of m1 in number of bytes.
print(m1.bytesize())
```
