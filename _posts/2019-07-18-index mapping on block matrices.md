---
layout: post
title: Index mapping on block matrices
mathjax: true
tags: []
---

Time for some stuff that is worth sharing.

It's common for one to use block matrices for computations. However, when using such conveniences we need to make sure we also have 
functions for getting back the original matrix, reading individual elements, and more. In this post I'll show you how to implement some 
of those functions. The functions in here also applies to sparse matrices stored in a block fashion, which commonly arise in computational physics, in particular 
in problems involving partial differential equations like cloth simulation and finite element simulations.

**Note**: In this post we assume all of our matrices are consistently stored in column-major order.

For example, a (large) 6-by-6 matrix could be stored as a compact 2-by-2 block matrix of 3-by-3 submatrices.

The large matrix is 

$$ 

A = 

\begin{bmatrix}
	 a_{1,1} & a_{1,2} & a_{1,3} & a_{1,4} & a_{1,5} & a_{1,6} \\
     a_{2,1} & a_{2,2} & a_{2,3} & a_{2,4} & a_{2,5} & a_{2,6} \\
     a_{3,1} & a_{3,2} & a_{3,3} & a_{3,4} & a_{3,5} & a_{3,6} \\
     a_{4,1} & a_{4,2} & a_{4,3} & a_{4,4} & a_{4,5} & a_{4,6} \\
     a_{5,1} & a_{5,2} & a_{5,3} & a_{5,4} & a_{5,5} & a_{5,6} \\
     a_{6,1} & a_{6,2} & a_{6,3} & a_{6,4} & a_{6,5} & a_{6,6} \\
\end{bmatrix}

$$

and it's corresponding compact block matrix is

$$

A = 

\begin{bmatrix}
	\boldsymbol{a_{1,1}} & \boldsymbol{a_{1, 2}} \\
	\boldsymbol{a_{2,1}} & \boldsymbol{a_{2, 2}} \\
\end{bmatrix}
	
$$

Now the first question we would like to answer is: How do I create the original matrix from the block matrix?

Any given element in the block matrix can be converted into the corresponding element in the original matrix by using the equations

$$ 
i_0 = 3 * i \\
j_0 = 3 * j
$$

So we can see that creating the original matrix shouldn't be difficult task. That is a matter of looping over the submatrices and put those in the original matrix. 

Here's a complete C++ implementation of the algorithm. Fortunately you can see I aimed self-documented code, so no need for further explanations.

{% highlight cpp %}

float32* a = (float32*)malloc(3 * A.M * 3 * A.N * sizeof(float32));

for (u32 i = 0; i < A.M; ++i)
{
	u32 i0 = 3 * i;
	
	for (u32 j = 0; j < A.N; ++j)
	{
		u32 j0 = 3 * j;
		
		b3Mat33 aij = A(i, j);

		for (u32 I = 0; I < 3; ++I)
		{
			for (u32 J = 0; J < 3; ++J)
			{
				u32 r = i0 + I;
				u32 c = j0 + J;

				a[r + 3 * A.M * c] = aij(I, J);
			}
		}
	}
}

{% endhighlight %}

Again, A is stored in column major order since the index function is 

$$ f(i, j) = i + mj $$

Now the original matrix of the corresponding block matrix can be built. It certainly has some applications sometimes.

The next and last question we would like to answer is: How can I get a particular element of the original matrix from the block matrix given 
the indices of this element in the original matrix?

Solving for $i, j$ in the equation

$$
i_0 = 3 i \\
j_0 = 3 j
$$

yields 

$$
i = \frac {i_0} {3} \\
j = \frac {j_0} {3} 
$$

For example, for row $r$ and column $c$ in the original matrix, the location of the submatrix in the block matrix containing the element at $(r, c)$ is

$$
i = \frac r 3 \\
j = \frac c 3
$$

However, we would like to know the location of the element in the submatrix, and not the location of the submatrix.

We can see in the last implementation we used two functions for mapping from subindices to global indices. Those are

$$
r = 3 i + I \\
c = 3 j + J 
$$

In the current problem we're trying to solve the values $r$ and $c$ are known. Those are the original matrix element indices. 

The subindices $i$ and $j$ are also known,

$$
i = \frac 3r \\
j = \frac 3c 
$$

We are seeking for the values ii and jj. Hence, solving for them, yields 

$$
I = r - 3 i \\
J = c - 3 j 
$$

**Note**: Since we're performing integer math $ 3 (\frac r 3) != r $ because of rounding. The same holds for $c$.

Now a function that returns the element from a block matrix given the original element location can be written.

{% highlight cpp %}

// Return the element in a block matrix given the indices 
// of the element in its corresponding expanded matrix.
static B3_FORCE_INLINE float32 b3GetElement(const b3BlockMat33& A, u32 r, u32 c)
{
	B3_ASSERT(r < 3 * A.M);
	B3_ASSERT(c < 3 * A.N);

	u32 i = r / 3;
	u32 j = c / 3;

	const b3Mat33& a = A(i, j);

	u32 I = r - 3 * i;
	u32 J = c - 3 * j;

	return a(I, J);
}

{% endhighlight %}

That's it. I hope this post can somewhat help you some point if you use block matrices for computations.