---
layout: post
title: How to transform a plane, with scale
mathjax: true
tags: []
---

In this post you will learn how to scale and transform a plane. The scale matrix can be a non-uniform scale matrix.

We first start by learning how to scale a plane with a non-uniform scale matrix. Then we will learn how to apply a full transformation to a plane.

## Scale 

Let us define a plane in local space (space 1) by a normal and any point in that plane:

$$
P_1 := (n_1, w_1)
$$

where $n_1$ is the plane normal and $w_1$ is the plane offset. The offset is the signed distance from the origin to any point in the plane.

A point $x_1$ is on the plane if

$$
n_1^T x_1 = w_1
$$

Remember that the dot product is equivalent to vector multiplication by the transpose.

Now suppose we want to apply a non-uniform scale to the plane. Call that matrix $S$. $S$ is a diagonal matrix.

With $S$ we can convert a point from local space to scaled space (space 2): 

$$
x_2 = S x_1
$$

Any normal vector in local space can be converted to scaled space:

$$
n_2 = \frac { S^{-1} n_1 } { \|S^{-1} n_1\| }
$$

Under uniform scale the plane normal doesn't change. 
However, for non-uniformely scaling normals we must apply the inverse scale and normalize the result. 

Now solving for $x_1$ and $n_1$ is easy. Let's solve for $x_1$.

$$
x_1 = S^{-1} x_2
$$

Now for $n_1$.

$$
n_2 = \frac { S^{-1} n_1 } { \|S^{-1} n_1\| } \\
n_2 { \|S^{-1} n_1\| } = S^{-1} n_1 \\
n_1 = S n_2 { \|S^{-1} n_1\| }
$$

Now substitute $x_1$ and $n_1$ into the local plane equation we defined earlier.

Define 

$$
a = { \|S^{-1} n_1\| }
$$

Then

$$
(S n_2 a)^T (S^{-1} x_2) = w_1 \\
a [(S n_2)^T (S^{-1} x_2)] = w_1
$$

We can rewrite the expression above using the following matrix product transpose property:

$$
(A B)^T = B^T A^T
$$

Therefore,

$$
a [n_2^T S^T S^{-1} x_2] = w_1
$$

Since $S$ is a diagonal matrix, we have $S^T = S$. Hence, 

$$
a [n_2^T I x_2] = w_1 \\
a [n_2^T x_2] = w_1
$$

Now we can identify and solve for the scaled plane offset $w_2$:

$$
a w_2 = w_1 \\
w_2 = \frac {w_1} {a}
$$

The equation above is handy because sometimes the scaled point in the plane is unknown.

## Scale, Rotate, then Translate

Now that you learned how to scale a plane let us learn how to transform a plane. 

Say we have a transform $A$ 

$$
A := (R, p)
$$

where $R$ is a orthogonal rotation matrix and $p$ is a translation vector. 

Now say want to apply a local scale represented by a matrix $S$. With our transform $A$ and scale $S$ we can convert any point in local space (1) 
to world space (2) with the scale applied in local space.

$$
x_2 = R S x_1 + p
$$

As we did in the last chapter, we can trivially solve for $x_1$.

$$
x_1 = S^{-1} R^T (x_2 - p)
$$

Now for transforming a normal vector we must apply the *inverse transpose*. Look for inverse transpose on Google and you will get many hits. This is how 
graphics programmers transform the vertex normals of a mesh from model space to world space in their engines.

Define 

$$
M = R S
$$

We want the inverse transpose of that matrix. For simplifying things around, we can use the matrix product inverse property. Therefore,

$$
M^{-1} = S^{-1} R^T
$$

We can use the transpose property for finding the inverse transpose matrix:

$$
(M^{-1})^T = (R^T)^T (S^{-1})^T = R S^{-1}
$$

We are now ready to transform a normal vector to world space:

$$
n_2 = \frac { R S^{-1} n_1 } {\|R S^{-1} n_1\|}
$$

Similarly, we can now solve for $n_1$.

$$
n_2 {\|R S^{-1} n_1\|} = R S^{-1} n_1  
$$

Premultiply both sides by $R^T$ to eliminate $R$ from the right side:

$$
R^T n_2 {\|R S^{-1} n_1\|} = S^{-1} n_1 
$$

Premultiply both sides by $S$ to remove $S^{-1}$ from the right side:

$$
n1 = S R^T n_2 {\|R S^{-1} n_1\|}
$$

Now we substitute $x_1$ and $n_1$ into the local plane equation for finding the scaled and transformed plane offset. 

Let us define 

$$
a = {\|R S^{-1} n_1\|}
$$

Therefore,

$$
(S R^T n_2 a)^T (S^{-1} R^T (x_2 - p)) = w_1 \\
a [(S R^T n2)^T (S^{-1} R^T (x_2 - p))] = w_1 
$$

Let us focus in on the terms inside the brackets. We will get back to the full equation above later. 

Expanding those terms we will get:

$$
(S R^T n_2)^T (S^{-1} R^T (x_2 - p)) \\
(S R^T n_2)^T (S^{-1} R^T x_2 - S^{-1} R^T p)) \\
(S R^T n_2)^T (S^{-1} R^T x_2) - (S R^T n_2)^T (S^{-1} R^T p)
$$

Now we need to simplify the last expression.

Let us define two temporary matrices $A$ and $B$:

$$
A = S R^T \\
B = S^{-1} R^T
$$

Rewriting the last expression using $A$ and $B$ becomes:

$$
(A n_2)^T (B x_2) - (A n_2)^T (B p)
$$

Again, we use the transpose multiplication property $ (A B)^T = B^T A^T $

$$
n_2^T A^T B x_2 - n_2^T A^T B p
$$

The matrix $A^T$ can be written as: 

$$
A^T = (S R^T)^T = (R^T)^T S^T = R S
$$

Replace $A^T$ with $R S$:

$$
n_2^T R S B x_2 - n_2^T R S B p
$$

Replace $B$ and some terms get cancelled out:

$$
n_2^T R S S^{-1} R^T x_2 - n_2^T R S S^{-1} R^T p = \\
n_2^T R I R^T x_2 - n_2^T R I R^T p = \\
n_2^T I x_2 - n_2^T I p = \\
n_2^T x_2 - n_2^T p = \\
w_2 - n_2^T p
$$

As you can see $w_2$ has been identified. Now we can write back our simplified bracketed expression to solve for $w_2$:

$$
a [w_2 - n_2^T p] = w_1 \\
w_2 - n_2^T p = \frac { w_1 } { a } \\
w_2 = \frac { w_1 } { a } + n_2^T p
$$

We have finally arrived at our very compact formula for the locally scaled and transformed plane normal and offset. Usefull, isn't it? 

Note: We can use this recipe for any transformation. 