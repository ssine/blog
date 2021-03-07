---
title: TAGE Branch Predictor
date: '2018-11-24'
math: true
tags:
    - Architecture
    - Course
categories:
    - Course
---

Introduce the powerful branch predictor TAGE. I haven't seen anyone writing related articles, probably going to read the papers.

<!-- more -->

The TAGE predictor is called the TAgged GEometric history length branch predictor. Since the first appearance on CBP-2 in 2006, it has not left the championship list. The winners of several competitions are all variants of the TAGE predictor. The TAGE predictor is a PPM-like predictor, the biggest difference being the way the status is updated.

TAGE Paper: A. Seznec, P. Michaud, "[A case for (partially) tagged Geometric History Length Branch Prediction](http://www.irisa.fr/caps/people/seznec/JILP-COTTAGE.pdf)", Journal of Instruction Level Parallelism, Feb. 2006

PPM Paper: P. Michaud, "[A PPM-like tag-based predictor](http://www.jilp.org/vol7/v7paper10.pdf)", Journal of Instruction Level Parallelism, April 2005

The above two papers cover everything you need to implement a TAGE predictor, which is what this article will cover next.

# Branch predictor

First explain the concept of the branch predictor. Modern processors usually have deep pipelines. When they encounter branch instructions, they are not predicted to cause a loss of performance and cause serious performance loss. Even simple predictions always jump more than wait for the branch results to appear. Efficient. The success rate of the branch predictor has a large impact on the performance of the processor.

The branch predictor can be regarded as an object, which determines whether the instruction is a branch instruction and whether it will jump according to the current PC value. In the processor, the next instruction is fetched during the decoding stage of the previous instruction. It refers to the operation, so the information that the branch predictor can get is only the current PC value. Whether the instruction is a branch instruction according to the PC address can be implemented by a relatively independent table, so it is mainly considered to determine whether the current instruction will jump.

After the branch predictor gives the prediction result, the processor continues to execute according to the result. When the result of the jump instruction is determined, the processor notifies the branch predictor of the result, and the predictor updates its state.

Therefore, a branch predictor object has two main interfaces:

1. Predict interface: Enter the PC and output the predicted result.
2. Update interface: Enter the actual jump result and update the predictor.

# Parts and their functions

Below are the various components of the TAGE predictor and their corresponding functions.

## Basic components

### GHR

GHR: Global History Register Global History Register. This is a shift register and the length is adjusted according to the predictor requirements. Each time a branch result is determined, GHR will shift one bit to the left and the latest result to the rightmost one.

### Saturate Counter

Saturation counter. The saturation counter has a value range of $[0, n]$ Out of scope modifications have no effect. Each branch actually jumps +1, and the branch does not actually jump -1. The value of the saturation counter when getting the prediction $>n/2$ Then predict to jump, $<n/2$ The prediction does not jump.

## Base Predictor

The base predictor is used to give a default prediction result if the tag predictor is missing. The basic predictor in the paper is a simple 2-bits historical predictor. Several low predictor PC as an index value, each entry is a saturating counter , the PC command flag location where jump.

## Tagged Predictor

The tag predictor is also a table. The table entry consists of a saturation counter, a tag, and a usage tag. Using the hash of the PC value and the GHR as the index of the table, the token in the entry is another hash of the PC and the GHR, which is a means of preventing collisions. Use tags to indicate whether the entry is useful or not, and is used when updating the state.

The TAGE predictor contains multiple tag predictors, the difference being that the GHR history length used is different. Suppose there is $n$ Predictor $T_1, T_2, \cdots, T_n$, then the length of the history records they use constitutes an equal series $L_n=a_0*q^{n-1}$. For example, for a TAGE with four tag predictors, the GHR bits used in the hashing of the index and tag can be $GHR[0:5], GHR[0:10], GHR[0:20], GHR[0:40]$.

# Get predictions

When obtaining the prediction, query all the entries of the marker predictor. If there is a hit, select the result of the table with the longest history. If all of the marker predictors are not hit, use the results given by the base predictor.

# update status

The logic to update the state is more complex, which is the main reason why TAGE performance exceeds the original PPM-like predictor.

When obtaining predictions and may have more than one hit the table, we call last resort (ie use the oldest) is the result of the **predicted value** , the predicted value Provider **provider** , said that it would adopt when the provider misses results of **candidate values** , the provider of the candidate value for the **provider candidates** .

---

Specific steps for the update operation

1. If the predicted value is different from the candidate value, whether the candidate provider's usage flag is correctly increased or decreased according to the prediction result.
2. Update the provider's saturation counter based on the actual jump results.
3. If the prediction is incorrect, the tag predictor entry is assigned according to the rules below.

First find all the corresponding entries that use the tag predictor whose history is longer than the provider.

If all of the tag predictor counterparts are occupied, then their usage flags are all decremented by one and the operation ends.

If only one item is empty, assign it to the current situation.

If there are multiple items that are empty, the current situation is assigned to one of the tag predictors by probability. The rule is: suppose there are $n$ Predictors can be assigned, sorted from small to large according to historical length, $1,2,\cdots,n$, then the $i$th Probability that the predictors are assigned is $P(i) = 2P(i+1)$.

# Implementation details

When implementing the TAGE predictor in C++, the logic part is not very difficult, but the more ambiguous point in the paper is the calculation of the hash function.

The longer history and PC values ​​are hashed to a small range to be used as an index/marker. The method used in the paper is a "folding" operation. definition of $fold$ Function is

$$
fold(H_h,m)=\underset{j\ge0}{\bigoplus}H_h\times2^{-jm}\,\mod 2^m
$$

This means that a large number will be $H_h$ Fold into a value of only m binary bits.

## Index hash calculation

For the base predictor, its index is the simple n bits after the PC.

The index of the marker predictor is more complicated, assuming that the binary length of an index is $L_t$, then the hash value of the index is

$$
(PC >> L_t)\oplus PC \oplus fold(H_h, m)
$$

among them $H_h$ is The historical length used by the current tag predictor, $m$ The length of the index.

## Tag hash calculation

The hash of the token is calculated as

$$
PC\oplus fold(H_h,m)\oplus\left(2\times fold(H_h,m-1)\right)
$$

## Ring shift register CSR

In order to calculate these hashes, we need an efficient hardware calculator of $fold$ function. The PPM paper states that this can be done with a circular shift register. Here is the diagram I drew:

![CSR](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/CSR_shift.png)
