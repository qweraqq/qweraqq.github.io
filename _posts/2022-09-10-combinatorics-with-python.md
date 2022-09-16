---
layout: post
title: "Counting with Python"
nav_order: 4
date: 2022-09-10 00:00:00 +0800
author: xiangxiang
categories: python
tags: [python math combinatorics]
---

counting有个高大上的名字叫组合数学

  <script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
        inlineMath: [['$','$']]
      }
    });
  </script>
  <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
  
* auto-gen TOC:
{:toc}

## 0x01 Sum rule
### 1.1 Sum rule
Generalized Rule of Sum
$$|A \cup B| = |A| + |B| - |A \cap B|$$
- If there are $n$ objects of the first type and there are $k$ objects of the second type, then there are $n+k$ objects of one of two types.
- when applying the sum rule, one should ensure that no object belongs to both classes

### 1.2 Code
- In practice, use **set**

```python
a = {1, 2, 3}
b = {2, 3, 4}

c = a.union(b)
c = a | b      # the union operation is also denoted by | in Python
```


## 0x02 Product rule
### 2.1 Product rule

$$A \times B = \{(a, b): a \in A, b\in B\}$$

$$|A \times B| = |A| \times |B|$$

$$A_1 \times ... \times A_k = \{(a_1, ...,a_k): a_1 \in A_1, ..., a_k \in A_k\}$$

$$|A_1 \times ... \times A_k| =|A_1| \times ... \times |A_k|$$

### 2.2 Code
```python
from itertools import product

a = (1, 2, 3)
b = (5, 6, 7)

a_times_b = product(a, b)
print(list(a_times_b))
```


## 0x03 Tuples
### 3.1 Number of tuples
The number of sequences of length $k$ composed out of $n$ symbols is $n^k$

### 3.2 Code
```python
from itertools import product

a = (1, 2, 3)

a_tuples = product(a, repeat=2)
print(list(a_tuples))
```

## 0x04 Permutations
### 4.1 k-permutations
The number of sequences of length &k& with no repetitions composed out of $n$ symbols is
$k$-permutations over $n$ symbols = $n(n-1)(n-2)...(n-k+1)$

### 4.2 Code
```python
from itertools import permutations
a = (1, 2, 3)
two_permutations = permutations(a, 2)
print(list(two_permutations))
```


## 0x05 Combinations
### 5.1 n choose k
- For a set $S$, its $k$-combination is a subset of $S$ of size $k$
- Each subset is counted $k!$ times if using $k$-permutations, make sure that each object is counted once
- The number of $k$-combinations of an $n$ element set is denoted by ${n \choose k}$
- ${n \choose k}$ Pronounced "n choose k"

$${n \choose k} = \frac{n!}{k!(n-k)!}$$

### 5.2 Code
```python
from itertools import combinations

a = (1, 2, 3)
two_combinations = combinations(a, 2)
print(list(two_combinations))
```

## 0x06 Pascal's Triangle

$${n \choose k} = {n-1 \choose k-1} + {n-1 \choose k}$$

### 6.1 Proof
Proof: choose k students from n students to form a team ${n \choose k}$

Fix one of the students, call her Alice
There are two types of teams:
1. Teams with Alice: ${n-1 \choose k-1}$
2. Teams without Alice : ${n-1 \choose k}$

Hence ${n \choose k} = {n-1 \choose k-1} + {n-1 \choose k}$

### 6.2 Picture
![](/img/Pascal-Triangle.jpg){:width="512px"}

$$
\begin{array}{ccccccccccc}
n=0 :   &    &    &    &    &  1 &    &    &    &    &   \cr
n=1 :   &    &    &    &  1 &    &  1 &    &    &    &   \cr
n=2 :   &    &    &  1 &    &  2 &    &  1 &    &    &   \cr
n=3 :   &    &  1 &    &  3 &    &  3 &    &  1 &    &   \cr
n=4 :   &  1 &    &  4 &    &  6 &    &  4 &    &  1 &   \cr
n=5 : 1 &    &  5 &    & 10 &    & 10 &    &  5 &    & 1 \cr
\dots \cr
\end{array}
$$

### 6.3 Pascal’s Triangle is Symmetric

$${n \choose k} = {n \choose n-k}$$

### 6.4 Row sums
- Can be proved by induction

$${n \choose 0} + {n \choose 1} +...+ {n \choose n} = 2^n$$


- Can be proved by construct a one-to-one correspondence between odd size subsets and even size subsets

For n > 0, 

$$\sum_{k=0}^{n}{(-1)^k {n \choose k} } = 0$$

Need to show that

$${n \choose 1} + {n \choose 3} + ... =  {n \choose 0} + {n \choose 2} + ...$$

Construct a one-to-one correspondence between odd size subsets and even size subsets
1. Fix any element x (can do this since n > 0)
2. Partition all subsets into pairs $(A, B)$ where $A = B \cup {x}$
3. One of A, B has odd size, the other one has even size

### 6.5 Binomial Theorem

$$(a+b)^n = \sum_{k=0}^{n}{ {n \choose k} a^{n-k}b^k}$$

Consequences
1. set $a=1, b=1$,  ${n \choose 0} + {n \choose 1} +...+ {n \choose n} = 2^n$

2. set $a=1, b=-1$, $\sum_{k=0}^{n}{(-1)^k{n \choose k}} = 0$

3. set $a=1, b=2$,  $3^n = {n \choose 0} + {n \choose 1}2 + {n \choose 2}2^2 + ... + {n \choose n}2^n$
Combinatorial proof:
- $3^n$ is the number of words of length $n$ over the alphabet {x, y, z}
- ${n \choose 0}$ is the number of words consisting of the letter x only
- ${n \choose 1}2$ is the number of words with exactly $n − 1$ letters x
- ${n \choose 2}2^2$ is the number of words with exactly $n − 2$ letters x

### 6.6 Code
```python
C = dict()  # C[n, k] is equal to n choose k

max_n = 8

for n in range(max_n):
    C[n, 0] = 1
    C[n, n] = 1

    for k in range(1, n):
        C[n, k] = C[n-1, k-1] + C[n-1, k]


print(C[4, 3])
```

## 0x07 Tricks
### 7.1 Bijection rule
A bijection between two sets $A$ and $B$ proves that the cardinalities of $A$ and $B$ are the same.

Example
```text
Consider 10 points on a circle. 
A: One can draw many triangles whose vertices are chosen among these points.
B: Also one can draw many 7-gons whose vertices are chosen among these points.

triangle (3 points) <---> 7-gons(the other 7 poins)

|A| = |B|
```

### 7.2 Complement rule
The number of objects satisfying a certain property is equal to the total number of objects minus the number of objects that do not satisfy the property

if $A \subseteq U$, then 
$|A| = |U| - |U \setminus A|$
