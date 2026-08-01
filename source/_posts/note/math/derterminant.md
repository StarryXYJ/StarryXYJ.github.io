---
title: 更合理的导出行列式的计算
date: 2026-3-12 13:15:44
tags:
- 笔记
- 数学
- 线性代数
categories:
- [笔记，数学]
---

## Definition

To derive the determinant rigorously from the concept of $n$ -dimensional volume, we define a function

$$f: (\mathbb{R}^n)^n \to \mathbb{R}$$

that maps $n$ vectors $(\mathbf{v}_1, \mathbf{v}_2, \dots, \mathbf{v}_n)$ to a scalar representing the **signed volume** of the parallelotope they span. For this function to behave like a measure of volume in Euclidean space, it must satisfy three fundamental axioms:


1. **Multilinearity:** For any $i$, $f$ is linear with respect to the $i$-th vector:
    
    $$f(\dots, c\mathbf{v}_i + \mathbf{u}, \dots) = cf(\dots, \mathbf{v}_i, \dots) + f(\dots, \mathbf{u}, \dots)$$
    
2. **Alternating Property:** If any two vectors are equal, the volume is zero (the shape collapses):
    
    $$f(\mathbf{v}_1, \dots, \mathbf{v}_i, \dots, \mathbf{v}_i, \dots, \mathbf{v}_n) = 0$$
    
3. **Normalization:** The volume of the unit hypercube spanned by the standard basis $\{\mathbf{e}_1, \dots, \mathbf{e}_n\}$ is unity:
    
    $$f(\mathbf{e}_1, \mathbf{e}_2, \dots, \mathbf{e}_n) = 1$$
    

From the **alternating property**, we can immediately derive that swapping any two vectors negates the result. Consider $f(\dots, \mathbf{v}_i + \mathbf{v}_j, \dots, \mathbf{v}_i + \mathbf{v}_j, \dots) = 0$. By expanding this using multilinearity and canceling the terms $f(\dots, \mathbf{v}_i, \dots, \mathbf{v}_i, \dots)$ and $f(\dots, \mathbf{v}_j, \dots, \mathbf{v}_j, \dots)$ which are zero, we arrive at:

$$f(\dots, \mathbf{v}_i, \dots, \mathbf{v}_j, \dots) = -f(\dots, \mathbf{v}_j, \dots, \mathbf{v}_i, \dots)$$

Now, let $A$ be an $n \times n$ matrix where the $j$-th column is $\mathbf{v}_j$. We express each column vector in the standard basis: $\mathbf{v}_j = \sum_{i=1}^n a_{ij} \mathbf{e}_i$. Substituting these into the volume function $f$, we use multilinearity to expand the expression completely:

$$f(\mathbf{v}_1, \dots, \mathbf{v}_n) = f\left(\sum_{i_1=1}^n a_{i_1 1} \mathbf{e}_{i_1}, \dots, \sum_{i_n=1}^n a_{i_n n} \mathbf{e}_{i_n}\right) = \sum_{i_1, \dots, i_n = 1}^n a_{i_1 1} \dots a_{i_n n} f(\mathbf{e}_{i_1}, \dots, \mathbf{e}_{i_n})$$

Due to the alternating property, any term in this summation where the indices $(i_1, \dots, i_n)$ are not distinct vanishes. Therefore, we only sum over sequences that are **permutations** $\sigma$ of the set $\{1, \dots, n\}$. For any such permutation, the alternating property implies:

$$f(\mathbf{e}_{\sigma(1)}, \dots, \mathbf{e}_{\sigma(n)}) = \text{sgn}(\sigma) f(\mathbf{e}_1, \dots, \mathbf{e}_n)$$

Applying the normalization axiom $f(\mathbf{e}_1, \dots, \mathbf{e}_n) = 1$, the expression simplifies to the **Leibniz formula**, which serves as the formal definition of the determinant:

$$\det(A) = \sum_{\sigma \in S_n} \text{sgn}(\sigma) \prod_{j=1}^n a_{\sigma(j), j}$$

This proof demonstrates that the determinant is the unique function that satisfies the geometric requirements of a signed volume scaling factor.

