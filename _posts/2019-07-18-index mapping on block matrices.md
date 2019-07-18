---
layout: post
title: Index mapping on block matrices
tags: []
---

Time for some tricks that are worth sharing.

It's common for one to use block matrices for computations. However, when using such conveniences we need to make sure we also have 
functions for getting back the original matrix, reading individual elements, and more. In this post I'll show you how to implement some 
of those functions. The functions in here also applies to sparse matrices stored in a block fashion, which commonly arise in computational physics, in particular 
in problems involving partial differential equations like cloth simulation and finite element simulations.

**Note**: In this post we assume all of our matrices are consistently stored in column-major order.

For example, a (large) 6-by-6 matrix could be stored as a compact 2-by-2 block matrix of 3-by-3 submatrices.

The large matrix is 

{% highlight cpp %}

A = [a11 a12 a13 a14 a15 a16] 
    [a21 a22 a23 a24 a25 a26]
    [a31 a32 a33 a34 a35 a36]
    [a41 a42 a43 a44 a45 a46]
    [a51 a52 a53 a54 a55 a56]
    [a61 a62 a63 a64 a65 a66]

{% endhighlight %}

and it's corresponding compact block matrix is

{% highlight cpp %}

A = [a11 a12]
    [a21 a22]
	
{% endhighlight %}

Now the first question we would like to answer is: How do I create the original matrix from the block matrix?

Any given element in the block matrix can be converted into the corresponding element in the original matrix by using the equations

{% highlight cpp %}

i0 = 3 * i
j0 = 3 * j

{% endhighlight %}

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

		for (u32 ii = 0; ii < 3; ++ii)
		{
			for (u32 jj = 0; jj < 3; ++jj)
			{
				u32 row = i0 + ii;
				u32 col = j0 + jj;

				a[row + 3 * A.M * col] = aij(ii, jj);
			}
		}
	}
}

{% endhighlight %}

Again, A is stored in column major order since the index function is f(i, j) = i + m * j.

Now the original matrix of the corresponding block matrix can be built. It certainly has some applications sometimes.

The next and last question we would like to answer is: How can I get a particular element of the original matrix from the block matrix given 
the indices of this element in the original matrix?

Solving for i, j in the equation

{% highlight cpp %}

i0 = 3 * i
j0 = 3 * j

{% endhighlight %}

yields 

{% highlight cpp %}

i = i0 / 3
j = j0 / 3

{% endhighlight %}

For example, for row *row* and column *col* in the original matrix, the location of the submatrix in the block matrix containing the element at (*row*, *col*) is

{% highlight cpp %}

i = row / 3
j = col / 3

{% endhighlight %}

However, we would like to know the location of the element in the submatrix, and not the location of the submatrix.

We can see in the last implementation we used two functions for mapping from subindices to global indices. Those are

{% highlight cpp %}

row = 3 * i + ii 
col = 3 * j + jj 

{% endhighlight %}

In the current problem we're trying to solve the values row and col are known. Those are the original matrix element indices. 

The subindices i and j are also known,

{% highlight cpp %}

i = row / 3
j = col / 3

{% endhighlight %}

We are seeking for the values ii and jj. Hence, solving for them, yields 

{% highlight cpp %}

ii = row - 3 * i 
jj = col - 3 * j 

{% endhighlight %}

**Note**: Since we're performing index math 3 * (row / 3) != row because of rounding. The same holds for col.

Now a function that returns the element from a block matrix given the original element location can be written.

{% highlight cpp %}

// Return the element in a block matrix given the indices 
// of the element in its corresponding expanded matrix.
static B3_FORCE_INLINE float32 b3GetElement(const b3BlockMat33& A, u32 row, u32 col)
{
	B3_ASSERT(row < 3 * A.M);
	B3_ASSERT(col < 3 * A.N);

	u32 i = row / 3;
	u32 j = col / 3;

	const b3Mat33& a = A(i, j);

	u32 ii = row - 3 * i;
	u32 jj = col - 3 * j;

	return a(ii, jj);
}

{% endhighlight %}

That's it. I hope this post can somewhat help you some point if you use block matrices for computations.