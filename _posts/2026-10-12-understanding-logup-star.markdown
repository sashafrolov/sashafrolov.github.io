---
layout: post
published: true
math: true
title:  "Understanding LogUp*: A Royal Road"
date:   2026-10-12 00:00:00
categories: zkproofs
---

Lookup arguments are a commonly used technique in zk-SNARKs which allow a prover to show that some set of values appears in a public table. We also often want to show something stronger: showing that some values appear at certain specific indices in a table. This is called an *indexed lookup*. 

[LogUp\*](https://eprint.iacr.org/2025/946), by Lev Soukhanov, is a lookup argument for indexed lookups for small tables. Its main benefits are that the prover can avoid committing to the entire vector of lookup results, that it works over small characteristic (read: binary) fields and that the prover avoids committing to large field elements as much as possible.

This post is inspired by [Understanding LogUp: A Royal Road](https://building-babylon.net/2024/02/14/a-royal-road-to-logup/), which I thought was a good explanation of ordinary LogUp. It will (hopefully) not require too much prior knowledge of LogUp.

## What are we trying to prove?

Suppose we have a table $$T$$ with $$M$$ entries and an array of indices $$I$$ with $$N$$ entries. Looking up each index produces an array $$V$$, where $$V[i]=T[I[i]]$$ for all indices $$i$$ in $$0$$ through $$N$$.

For example,

$$
T=(t_0,t_1,t_2,t_3), \qquad I=(2,0,2,1)
$$

gives

$$
V=(t_2,t_0,t_2,t_1).
$$

We assume the prover has already committed (via a polynomial commitment scheme) to $$T$$ and $$I$$ (or that $$T$$'s multilinear extension can be evaluated cheaply, if it is sufficiently structured). 

One way to prove the validity of a set of lookups is to commit to $$V$$ too, and then use ordinary LogUp to show that every pair $$(I[i],V[i])$$ appears in the table of pairs $$(j,T[j])$$. Including the index in each pair is the standard trick to convert an unindexed lookup argument like LogUp into an indexed one.

This works (though only in finite fields whose characteristic is greater than the number of lookups $$N$$) but it requires committing to $$N$$ additional field elements. Consider proving $$\sim 2^{20}$$ lookups into a table with $$\sim 2^8$$ entries (a common parameter regime for LogUp-style lookup arguments). Even though the table is small, the prover now has to commit to an additional vector of $$\sim 2^{20}$$ lookup results. If the indices are small and the table values are large extension field elements, that extra commitment can be much more expensive than the commitment to $$I$$. In contexts like elliptic curve-based PCSs, or binary field-based ones, the bitlengths of the values that we commit to are very relevant to the costs of the scheme.

The [LogUp\* paper](https://eprint.iacr.org/2025/946) targets the cost of committing to $$I$$, and is tailored to a binary field context, where field characteristic is small and committing to short field elements is cheaper.

## What do we need to do with the lookup result V?

In proof systems based on the sumcheck protocol (i.e. basically all of the current state of the art ones), when proving some property of a vector $$V$$, the "hard part" of working with $$V$$ is often just proving an evaluation claim $$V(r)=e.$$

Here $$V(r)$$ is the multilinear extension of the vector $$V$$ evaluated at a random point $$r$$ derived from the surrounding protocol. We can ask for this evaluation even when $$V$$ has not been committed on its own (this is called virtualization). To avoid committing to $$V(r)$$ directly, LogUp\* will prove this evaluation using the existing commitments/evaluation access to $$T$$ and $$I$$.

A big part of LogUp* is showing how to evaluate the multilinear extension of $$V$$ in terms of the extensions of $$T$$ and $$I$$. Let $$N=2^n$$, and let the bits of each row index $$i$$ be $$(i_0,\ldots,i_{n-1})$$. For $$r\in\mathbb F^n$$, define

$$
\operatorname{eq}(r,i)
=
\prod_{k=0}^{n-1}
\big(r_ki_k+(1-r_k)(1-i_k)\big).
$$

When the values $$i_k$$ are bits, each factor "selects" either $$r_k$$ or $$1-r_k$$. These equality polynomials can be used for interpolation over the Boolean hypercube, so

$$
V(r)=\sum_{i=0}^{N-1}\operatorname{eq}(r,i)V[i]
=\sum_{i=0}^{N-1}\operatorname{eq}(r,i)T[I[i]].
$$

Thus, the evaluation of $$V$$ we want is a weighted sum of table entries of $$T$$. The weights for each entry of $$T$$ are determined by $$r$$ and which lookups are fo that entry. 

To illustrate these weights, consider the same, previously discussed lookup table instance with 4 entries. The values of the equality polynomial for interpolating the extension of a table with 4 entries are:

$$
\begin{aligned}
a&=(1-r_0)(1-r_1), & b&=r_0(1-r_1),\\
c&=(1-r_0)r_1,     & d&=r_0r_1,
\end{aligned}
$$

Now, computing the multilinear extension of $$V$$ for the previous example of a lookup instance:

$$
V(r)=at_2+bt_0+ct_2+dt_1.
$$

Both $$a$$ and $$c$$ correspond to lookups for the same table entry, so we can collect them:

$$
V(r)=bt_0+dt_1+(a+c)t_2.
$$

Now we have one weight per table entry. These weights are:

$$
Y=(b,d,a+c,0).
$$

Note that the last entry is zero because there were no lookups to $$t_3$$, but $$t_2$$'s weight is the sum of multiple evaluations of the $$\operatorname{eq}$$ polynomial, because it had multiple lookups.

## Collecting the weights

This same regrouping works for any index array. Define

$$
Y_j=\sum_{i:I_i=j}\operatorname{eq}(r,i).
$$

When expressing the evaluation of the multilinear extension of $$V$$ in terms of a weighted sum of entries of $$T$$, $$Y_j$$ is the weight assigned to table entry $$j$$ in this sum. To compute $$Y$$, start with $$M$$ zeros. Visit each row $$i$$ and add its weight $$\operatorname{eq}(r,i)$$ to entry $$I_i$$. Now, rearranging sums:

$$
\begin{aligned}
V(r)
&=\sum_{i=0}^{N-1}\operatorname{eq}(r,i)T_{I_i}\\
&=\sum_{j=0}^{M-1}T_j
   \left(\sum_{i:I_i=j}\operatorname{eq}(r,i)\right)\\
&=\sum_{j=0}^{M-1}T_jY_j
=\langle T,Y\rangle.
\end{aligned}
$$

This suggests a cheaper way of proving an evaluation of $$V$$: commit to $$Y$$ and show that $$\langle T,Y\rangle=e$$. This additional polynomial commitment is to a vector of $$M$$ entries, independent of the number of lookups.

The inner product claim can be proved via a standard use of the sumcheck protocol. Assuming $$M=2^m$$, we view $$T$$ and $$Y$$ as multilinear polynomials and run sumcheck on

$$
\sum_{y\in\{0,1\}^m}T(y)Y(y)=e.
$$

Sumcheck reduces this claim to evaluations of $$T$$ and $$Y$$ at a random point. The prover can then open polynomial commitments to these vectors.

We can now translate this regrouping into the paper's terminology. I personally found the paper's terminology somewhat confusing, and prefer the previously explained treatement. The lookup array $$V$$ is called the **pullback** of $$T$$ along $$I$$, written $$I^*T$$. The collected weights $$Y$$ are the **pushforward** of $$\operatorname{eq}(r,\cdot)$$ along $$I$$, written $$I_*\operatorname{eq}(r,\cdot)$$. More generally, for any row weights $$X$$,

$$
(I_*X)_j=\sum_{i:I_i=j}X_i,
\qquad
\langle I^*T,X\rangle=\langle T,I_*X\rangle.
$$

The paper calls this *pullback-pushforward duality*. It is the regrouping we just did, where we collected the sums of the equality polynomial and committed to these, rather than committing to the looked up table values.

Of course, one more step remains. We have not yet specified a protocol for checking that $$Y$$ is valid. The prover could commit to an arbitrary $$Y$$, and we still need to check that each $$Y_j$$ is the sum of the weights assigned to table position $$j$$.

## Checking the collected weights with LogUp

Ordinary LogUp uses the "LogUp identity" to check the validity of a set of lookups:

$$
\sum_i\frac{1}{C-w_i}
=
\sum_j\frac{m_j}{C-t_j},
$$

where $$m_j$$ records how many times table value $$t_j$$ was used. $$C$$ is a random challenge. For simplicity, suppose the table values are distinct. It should be obvious to see that this identity holds if the $$m_j$$'s correctly count the number of lookups to each value, and it is not hard to show that converse, that this equation fails with high probability if the $$m_j$$'s are invalid.

Our goal is to check the validity of the weights $$Y$$ using a LogUp-like identity, so it is natural to replace the quantities in the numerators on either side of the equation with values of $$\operatorname{eq}$$ rather than $$1$$ and $$m_j$$. Likewise, we put table indices in the denominators.

We need distinct field elements to label the table positions. Write $$\iota(j)$$ for the label of position $$j$$. Over a prime field whose characteristic exceeds $$M$$, we can use $$\iota(j)=j$$. We'll return to binary fields below. Some works on lookup arguments avoid this formalism, assume high field characteristic, and just jump between $$j$$ as an index and $$j$$ as a field element.

The rewritten LogUp identity to check is

$$
\sum_{i=0}^{N-1}
\frac{\operatorname{eq}(r,i)}{C-\iota(I_i)}
=
\sum_{j=0}^{M-1}
\frac{Y_j}{C-\iota(j)}
\tag{LogUp* Identity}
\label{eq:logup-star}
$$

In our small example, the left hand side is

$$
\frac{a}{C-\iota(2)}
+\frac{b}{C-\iota(0)}
+\frac{c}{C-\iota(2)}
+\frac{d}{C-\iota(1)}.
$$

Collecting the two fractions with denominator $$C-\iota(2)$$ gives exactly the right-hand side for $$Y=(b,d,a+c,0)$$.

Why does this force the prover to use the correct $$Y$$? Treating $$C$$ as a formal variable, and looking for an equality of rational functions, let

$$
\Delta_j=Y_j-\sum_{i:I_i=j}\operatorname{eq}(r,i)
$$

be the error at table position $$j$$. After subtracting the left hand side from the right hand side of the LogUp* identity, we get

$$
\sum_{j=0}^{M-1}\frac{\Delta_j}{C-\iota(j)}=0.
$$

Then, multiply through by $$D(C)=\prod_{j=0}^{M-1}(C-\iota(j))$$. If we then substitute $$C=\iota(k)$$ for any $$k$$, every term except the one containing $$\Delta_k$$ vanishes, leaving

$$
\Delta_k\prod_{j\ne k}\big(\iota(k)-\iota(j)\big)=0.
$$

The labels are $$\iota$$ are distinct, so every factor in the product is nonzero. This shows that $$\Delta_k=0$$ for all $$k$$, so the LogUp* identity shows that all the weights were collected correctly. Conversely, if any $$\Delta _k$$ is not zero, then the two sides of the LogUp* identity are not equal with high probability.

Notice that the table's values don't appear in this check. LogUp\* uses the indices to check $$Y$$ and uses $$T$$ in the separate inner product check.

## Replacing the formal variable with a random challenge

The LogUp* verifier cannot check the LogUp* identity by looking at functions as formal rational functions. Instead, as is common in SNARKs, after the prover commits to $$Y$$, the verifier samples a random field element $$c$$ and checks the LogUp* identity at a random challenge point $$C=c$$.

If $$Y$$ is wrong, clearing the denominators gives a nonzero polynomial. As usual, the probability this polynomial evaluates to zero is low. The paper gives a conservative soundness error from this random check of $$(N+M)/\lvert\mathbb{F}\rvert$$ for this step. The LogUp* verifier should also reject challenges which make some index in the LogUp* identity have a zero denominator.

## Proving a sum of fractions

We have now reduced the problem to showing the equality of two sums of fractions. We now need some protocol to prove the evaluations of two sums of fractions. This is basically the same thing that needs to be shown in regular LogUp.

The GKR version of LogUp, from [Papini and Haböck](https://eprint.iacr.org/2023/1284), proves this sum with a specialized GKR circuit. It arranges the additions of fractions in a binary tree. Recall grade school fraction addition:

$$
\frac{a_0}{b_0}+\frac{a_1}{b_1}
=
\frac{a_0b_1+a_1b_0}{b_0b_1}.
$$

Consider a circuit to evaluate the sum of a bunch of fractions. The GKR version of LogUp's circuit does this by storing each fraction as a numerator,denominator pair. It combines adjacent pairs and their results, continuing up a binary tree, using the standard fractional addition identity. Every internal node uses only additions and multiplications. The root contains one numerator and one denominator for the entire sum.

For LogUp\*, the two trees of fractional additions have leaves

$$
\left(\operatorname{eq}(r,i),\,c-\iota(I_i)\right)
\qquad\text{and}\qquad
\left(Y_j,\,c-\iota(j)\right).
$$

corresponding to the numerators and denominators in the LogUp* identity. To begin the GKR protocol, the prover sends the two root pair results, say $$(P_L,Q_L)$$ and $$(P_R,Q_R)$$. The verifier checks

$$
P_LQ_R=P_RQ_L,
\qquad Q_L\ne0,\quad Q_R\ne0.
$$

Nonzero checks are important, since they prevent soundness issues from dividing by zero.

As usual for the GKR protocol, the specialized instance of GKR checks the arithmetic in each tree by working from the root down to the leaves, with an application of the sumcheck protocol at each layer. The prover computes the intermediate values, but does not need to commit to them. At the leaves, GKR has reduced proving the computation to a few polynomial evaluations:

- On the lookup side, the verifier can evaluate $$\operatorname{eq}(r,\cdot)$$ itself efficiently. Evaluating the denominator requires an opening proof of the committed index column $$\widehat I_i=\iota(I_i)$$.
- On the table side, the numerator requires an opening of $$Y$$. The denominator comes from the public table position labels, whose multilinear extension can be evaluated directly with properly chosen encodings.

Together with the earlier product sumcheck for $$\langle T,Y\rangle=e$$, this proves the lookup evaluation. The only additional committed vector is $$Y$$.

## The protocol in order

Putting the pieces together, suppose $$T$$ and the encoded index column $$I$$ are already committed, and we must prove $$(I^*T)(r)=e$$ for the surrounding protocol.

1. The prover computes $$Y_j=\sum_{i:I_i=j}\operatorname{eq}(r,i)$$ and commits to $$Y$$.
2. The verifier samples $$c$$.
3. The prover and verifier engage in the GKR protocol to check that the LogUp* identity holds for $$Y$$ evaluated at $$c$$.
4. The prover and verifier run a degree 2 product sumcheck to show $$\langle T,Y\rangle=e$$.
5. The prover provides openings for the evaluations of the encoded index column $$I$$, $$T$$, and $$Y$$ required by those reductions.

## Why small characteristic is okay

The other cool feature of LogUp* is that it works in small characteristic fields. Ordinary LogUp has a problem in small characteristic fields. In a field of characteristic $$p$$, a value that is looked up $$p$$ times contributes

$$
\underbrace{\frac{1}{C-w}+\cdots+\frac{1}{C-w}}_{p\text{ times}}
=
\frac{p}{C-w}=0.
$$

to its sum. If $$w$$ is absent from the table, the identity is still valid! In characteristic two, just two lookups are enough to trigger this issue. This is why the LogUp requires the witness length to be smaller than the field characteristic.

LogUp\* checks sums of weights over fields. Our proof that every $$\Delta_j$$ is zero used only distinct labels, so it works in every characteristic. The numerators are distinct, random evaluations of the $$\operatorname{eq}$$ polynomial, and integer counts don't appear anywhere, so the previously mentioned type of cancellation doesn't happen.

Eagen and Haböck's [Bypassing the characteristic bound in LogUp](https://eprint.iacr.org/2024/2067) also solves this problem. LogUp*'s approach has a connection to their approach. The Eagen/Haböck approach gives witness occurrences algebraic weights (rather than 1) to prevent their counts from vanishing. The functions $$\operatorname{eq}(r,i)$$, viewed as polynomials in $$r$$, form a multilinear Lagrange basis. A sum over any nonempty set of rows is therefore a nonzero polynomial, even in characteristic two (it must evaluate to $$1$$ at the summed Boolean rows).

Such a polynomial can still vanish at a random point, but this is now a low probability event, guaranteed by the Schwarz-Zippel lemma. 

There is one more important detail for binary fields. The labels $$0,1,2,\ldots$$ passed to the $$\iota$$ functions cannot mean repeated addition of $$1$$, since $$2=0$$ in binary fields. Instead, the bits of a table index should be encoded using field elements $$b_0,\ldots,b_{m-1}$$ that are linearly independent over $$\mathbb F_2$$:

$$
\iota(j)=\sum_{k=0}^{m-1}j_kb_k.
$$

Most binary field field libraries/implementations have some standard convenient mapping from bits to a binary field element which looks like this. This gives distinct labels to all $$2^m$$ table positions. The multilinear extension of these indices is the linear polynomial $$J(z)=\sum_k b_kz_k$$, so the verifier can evaluate the table-side denominator as $$c-J(z)$$. 

## Several lookups into one table

Many applications of LogUp care about having multiple columns (i.e. multiple lookup vectors which we wish to prove valid). Many papers on LogUp specify how to batch checking the validity of multiple lookup columns into one check. The same type of trick is possible for LogUp*. If several ($$L$$) index arrays use the same table, we can collect all their weights into one $$Y$$. Suppose their evaluation claims are

$$
(I_\ell^*T)(r_\ell)=e_\ell,
\qquad 0\le\ell<L.
$$

After these claims are fixed, sample a batching challenge $$\gamma$$ and define

$$
Y_j=
\sum_{\ell=0}^{L-1}\gamma^\ell
\sum_{i:(I_\ell)_i=j}\operatorname{eq}(r_\ell,i).
$$

The inner product check becomes

$$
\langle T,Y\rangle=\sum_{\ell=0}^{L-1}\gamma^\ell e_\ell.
$$

For the fractional check, the lookup-side fractions for $$L$$ lookup columns need to be summed, with the numerators for array $$\ell$$ scaled by $$\gamma^\ell$$. The table side still uses the single array $$Y$$. This is a very standard random batching trick in SNARKs, which adds an additional soundness error of at most $$(L-1)/\lvert\mathbb{F}\rvert$$.

## Conclusion/total savings

LogUp* achieves a few cool things. It has a savings in terms of what gets committed. A commitment to an vector of length $$N$$ of lookup results is replaced by a commitment to a vector of length $$M$$ of weights. Many lookups into a small table can therefore share a small auxiliary commitment, even when the table values live in a large field.

LogUp* is particularly beneficial when coupled with a PCS which has costs which scale in terms of the bitlengths of the coefficients of polynomials they commit to. The index vector $$I$$ can be made to have values of small bitlengths, and it is the only length $$N$$ vector committed by LogUp*. The weight vector $$Y$$ does have full length field elements, but its length only scales in $$M$$.

And lastly, LogUp* works nicely over binary fields :). I found it somewhat hard to understand at first, but I think it is a cool modification of LogUp which achieves a few cool properties at the same time. 

## References

- [Understanding LogUp: A Royal Road](https://building-babylon.net/2024/02/14/a-royal-road-to-logup/), Benjamin Wilson.
- [Multivariate lookups based on logarithmic derivatives](https://eprint.iacr.org/2022/1530), Ulrich Haböck.
- [Improving logarithmic derivative lookups using GKR](https://eprint.iacr.org/2023/1284), Shahar Papini and Ulrich Haböck.
- [Bypassing the characteristic bound in LogUp](https://eprint.iacr.org/2024/2067), Liam Eagen and Ulrich Haböck.
- [LogUp*: faster, cheaper LogUp argument for small-table indexed lookups](https://eprint.iacr.org/2025/946), Lev Soukhanov.
