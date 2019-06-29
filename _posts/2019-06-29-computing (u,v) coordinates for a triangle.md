---
layout: post
title: Computing (u, v) coordinates for a triangle
tags: []
---

I've implemented the cloth simulation described in [1] in 3D. This continuum formulation requires that we define the (u, v) coordinates of a triangle vertices 
in order to measure streching, compression and shearing.

In this post I'll show you a robust way of computing these values in a per-triangle basis. This is very usefull if you want to define the cloth mesh using only vertices and triangles and don't have 
the (u, v) coordinates available in the cloth mesh. The (u, v) coordinates we define here satisfy the requirements of the paper.

The first step in our derivation is to choose the u axis to form an (u, v) space. For a triangle those can be the directions of the AB, AC, or BC edges. Here we chose the direction of AB as the u axis. The v axis is an axis orthogonal to the u axis. The origin (0, 0) of our (u, v) space is A. See the image below.

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/uv.png) 
{: refdef}

Looking at the image, the (u, v) coordinates of A is obviously (u, v) = (0, 0).

The (u, v) coordinates of the B vertex is (u, v) = (\|\|AB\|\|, 0)

For computing the u coordinate of C we can project the segment AC into the direction of AB. Therefore,

u = dot(AC, AB / \|\|AB\|\|)

Now we need the v coordinate of the vertex C. By looking at the picture we can see that this is simple the height h of the triangle. So we need to find the 
height of the triangle. We can do this using the formula for the area of a triangle.

Remember from high school that the formula for the area of a triangle is

A  = b * h / 2

Solving for the height yields

h = (A * 2) / b

In our example the base is b = \|\|AB\|\|. Finally the twice of the area of our triangle can be computed as 

A * 2 = \|\|cross(AB, AC)\|\|

## Appendix

Now I'll show an usefull simplification to use in your implementation. 

Another step in the simulation is to map the from plane coordinates to world space using the matrix 

{% highlight cpp %}

A = [du1 du2]^-1
    [dv1 dv2]

{% endhighlight %}

The inverse of this 2-by-2 matrix can be computed as 

{% highlight cpp %}

A^-1 = det(A)^-1 * [dv2  -du2]
                   [-dv1  du1]

{% endhighlight %}
				
As in the paper, the world coordinates is

{% highlight cpp %}

[wu wv] = [dx1 dx2] * A^-1

{% endhighlight %}

Now we define aij = A^-1_ij.

{% highlight cpp %}

[wu wv] = [dx1.x * a11 + dx2.x * a21	dx1.x * a12 + dx2.x * a22]
          [dx1.y * a11 + dx2.y * a21	dx1.y * a12 + dx2.y * a22]
          [dx1.z * a11 + dx2.z * a21	dx1.z * a12 + dx2.z * a22]

{% endhighlight %}

This can be written as 

{% highlight cpp %}

[wu wv] = [a11 * dx1 + a21 * dx2	a12 * dx1 + a22 * dx2]

{% endhighlight %}

This equation can be further simplified to 

{% highlight cpp %}

[wu wv] = [det(A)^-1 * (dv2 * dx1 - dv1 * dx2)	det(A)^-1 * (-du2 * dx1 + du1 * dx2)]

{% endhighlight %}

## References

[1] David Baraff, Andrew Witkin. [Large Steps in Cloth Simulation](https://www.cs.cmu.edu/~baraff/papers/sig98.pdf).