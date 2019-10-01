---
layout: post
title: How to transform a plane, with scale
tags: []
---

In this post you will learn how to scale and transform a plane. The scale matrix can be a non-uniform scale matrix.

We first start by learning how to scale a plane with a non-uniform scale matrix. Then we will learn how to apply a full transformation to a plane.

## Scale 

Let us define a plane in local space (space 1) by a normal and any point in that plane:

{% highlight cpp %}
P1 := (n1, w1)
{% endhighlight %}

where n1 is the plane normal and w1 is the plane offset. The offset is the signed distance from the origin to any point in the plane.

A point x1 is on the plane if

{% highlight cpp %}
n1^T * x1 = w1
{% endhighlight %}

Remember that the dot product is equivalent to vector multiplication by the transpose.

Now we want to apply a non-uniform scale to the plane. Call that matrix S. S is a diagonal matrix.

With S we can convert a point from local space to scaled space (space 2): 

{% highlight cpp %}
x2 = S * x1
{% endhighlight %}

Any normal vector in local space can be converted to scaled space:

{% highlight cpp %}
n2 = (S^-1 * n1) / norm(S^-1 * n1)
{% endhighlight %}

Under uniform scale the plane normal doesn't change. 
However, for non-uniformely scaling normals we must apply the inverse scale and normalize the result. 

Now solving for x1 and n1 is easy. Let's solve for x1.

{% highlight cpp %}
x1 = S^-1 * x2
{% endhighlight %}

Now for n1.

{% highlight cpp %}
n2 = (S^-1 * n1) / norm(S^-1 * n1) <=>
n2 * norm(S^-1 * n1) = S^-1 * n1 <=> Premultiply both sides by S to toss out S^-1 from the right side
n1 = S * n2 * norm(S^-1 * n1)
{% endhighlight %}

Now substitute x1 and n1 into the local plane equation we defined earlier.

Define 

{% highlight cpp %}
a = norm(S^-1 * n1)
{% endhighlight %}

Then

{% highlight cpp %}
(S * n2 * a)^T * (S^-1 * x2) = w1 <=>
a * [(S * n2)^T * (S^-1 * x2)] = w1
{% endhighlight %}

We can rewrite the expression above using the following matrix product transpose property:

{% highlight cpp %}
(A * B)^T = B^T * A^T
{% endhighlight %}

Therefore,

{% highlight cpp %}
a * [n2^T * S^T * S^-1 * x2] = w1
{% endhighlight %}

Since S is a diagonal matrix, S^T = S. Hence, 

{% highlight cpp %}
a * [n2^T * I * x2] = w1 <=>
a * [n2^T * x2] = w1
{% endhighlight %}

Now we can identify and solve for the scaled plane offset w2:

{% highlight cpp %}
a * w2 = w1
w2 = w1 / a
{% endhighlight %}

The equation above is handy because sometimes the scaled point in the plane is unknown.

## Scale, Rotate, then Translate

Now that you learned how to scale a plane let us learn how to transform a plane. 

Say we have a transform A 

{% highlight cpp %}
A := (R, p)
{% endhighlight %}

where R is a orthogonal rotation matrix and p is a translation vector. 

Now say want to apply a local scale represented by a matrix S. With our transform A and scale S we can convert any point in local space (1) 
to world space (2) with the scale applied in local space.

{% highlight cpp %}
x2 = R * S * x1 + p
{% endhighlight %}

As we did in the last chapter, we can trivially solve for x1.

{% highlight cpp %}
x1 = S^-1 * R^T * (x2 - p)
{% endhighlight %}

Now for transforming a normal vector we must apply the *inverse transpose*. Look for inverse transpose on Google and you will get many hits. This is how 
graphics programmers transform the vertex normals of a mesh from model space to world space in their engines.

Define 

{% highlight cpp %}
M = R * S
{% endhighlight %}

We want the inverse transpose of that matrix. For simplifying things around, we can use the matrix product inverse property. Therefore,

{% highlight cpp %}
M^-1 = S^-1 * R^T
{% endhighlight %}

We can use the transpose property for finding the inverse transpose matrix:

{% highlight cpp %}
(M^-1)^T = (R^T)^T * (S^-1)^T = R * S^-1
{% endhighlight %}

We are now ready to transform a normal vector to world space:

{% highlight cpp %}
n2 = (R * S^-1 * n1) / norm(R * S^-1 * n1)
{% endhighlight %}

Similarly, we can now solve for n2:

{% highlight cpp %}
n2 * norm(R * S^-1 * n1) = R * S^-1 * n1 <=>  Premultiply both sides by R^T to eliminate R from to right side
R^T * n2 * norm(R * S^-1 * n1) = S^-1 * n1 <=> Premultiply both sides by S to remove S^-1 from to right side
n1 = S * R^T * n2 * norm(R * S^-1 * n1)
{% endhighlight %}

Now we substitute x1 and n1 into the local plane equation for finding the scaled and transformed plane offset. 

Let us define 

{% highlight cpp %}
a = norm(R * S^-1 * n1)
{% endhighlight %}

Therefore,

{% highlight cpp %}
(S * R^T * n2 * a)^T * (S^-1 * R^T * (x2 - p)) = w1 <=>
a * [(S * R^T * n2)^T * (S^-1 * R^T * (x2 - p))] = w1 
{% endhighlight %}

Let us focus in on the terms inside the brackets. We will get back to the full equation above later. 

Expanding those terms we will get:

{% highlight cpp %}
(S * R^T * n2)^T * (S^-1 * R^T * (x2 - p)) <=>
(S * R^T * n2)^T * (S^-1 * R^T * x2 - S^-1 * R^T * p)) <=>
(S * R^T * n2)^T * (S^-1 * R^T * x2) - (S * R^T * n2)^T * (S^-1 * R^T * p)
{% endhighlight %}

Now we need to simplify the last expression.

Let us define two temporary matrices A and B:

{% highlight cpp %}
A = S * R^T
B = S^-1 * R^T
{% endhighlight %}

Rewriting the last expression using A and B becomes:

{% highlight cpp %}
(A * n2)^T * (B * x2) - (A * n2)^T * (B * p)
{% endhighlight %}

Again, we use the transpose multiplication property ((A * B)^T = B^T * A^T)

{% highlight cpp %}
n2^T * A^T * B * x2 - n2^T * A^T * B * p
{% endhighlight %}

The matrix A^T can be written as: 

{% highlight cpp %}
A^T = (S * R^T)^T = (R^T)^T * S^T = R * S
{% endhighlight %}

Replace A^T with R * S:

{% highlight cpp %}
n2^T * R * S * B * x2 - n2^T * R * S * B * p
{% endhighlight %}

Replace B and some terms get cancelled out:

{% highlight cpp %}
n2^T * R * S * S^-1 * R^T * x2 - n2^T * R * S * S^-1 * R^T * p <=>
n2^T * R * I * R^T * x2 - n2^T * R * I * R^T * p <=>
n2^T * I * x2 - n2^T * I * p <=> 
n2^T * x2 - n2^T * p <=>
w2 - n2^T * p
{% endhighlight %}

As you can see w2 has been identified. Now we can write back our simplified bracketed expression to solve for w2:

{% highlight cpp %}
a * [w2 - n2^T * p] = w1 <=>
w2 - n2^T * p = w1 / a <=>
w2 = w1 / a + n2^T * p
{% endhighlight %}

We have finally arrived at our very compact formula for the locally scaled and transformed plane normal and offset. Usefull, isn't it? 

Note: We can use this recipe for any transformation. 