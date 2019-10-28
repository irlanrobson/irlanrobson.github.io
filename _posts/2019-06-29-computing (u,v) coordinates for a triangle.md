---
layout: post
mathjax: true
title: Computing (u, v) coordinates for a triangle
tags: []
---

I've implemented the cloth simulation described in [1] in 3D. This continuum formulation requires that we define the $$(u, v)$$ coordinates of a triangle vertices 
in order to measure streching, compression and shearing.

In this post I'll show you a robust way of computing these values in a per-triangle basis. This is very usefull if you want to define the cloth mesh using only vertices and triangles and don't have 
the $$(u, v)$$ coordinates available in the cloth mesh. The $$(u, v)$$ coordinates we define here satisfy the requirements of the paper.

The first step in our derivation is to choose the $$u$$ axis to form an $$(u, v)$$ space. For a triangle those can be the directions of the $$\vec{AB}$$, $$\vec{AC}$$, or $$\vec{BC}$$ edges. Here we chose the direction of $$\vec{AB}$$ as the $$u$$ axis. The $$v$$ axis is an axis orthogonal to the $$u$$ axis. The origin $$(0, 0)$$ of our $$(u, v)$$ space is $$A$$. See the image below.

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/uv.png) 
{: refdef}

Looking at the image, the $$(u, v)$$ coordinates of $$A$$ is obviously 

$$(u, v) = (0, 0)$$ 

The $$(u, v)$$ coordinates of the vertex $$B$$ is 

$$(u, v) = (\|\vec {AB}\|, 0)$$

For computing the $$u$$ coordinate of $$C$$ we can project the segment $$\vec{AC}$$ into the direction of $$\vec{AB}$$. Therefore

$$u = \frac {\vec{AC} \cdot \vec{AB} } {\|\vec{AB}\|} $$

Now we need the $$v$$ coordinate of the vertex $$C$$. By looking at the picture we can see that this is simple the height $$h$$ of the triangle. So we need to find the 
height of the triangle. We can do this using the formula for the area of a triangle.

Remember from high school that the formula for the area of a triangle is

$$a = \frac {bh} {2} $$

Solving for the height yields

$$ h = \frac {2a} {b} $$

In our example the base of the triangle is $$b = \|AB\|$$. Finally the twice of the area of our triangle is defined as 

$$ 2a = \| {\vec{AB}} \times {\vec{AC}} \| $$

## Appendix

Now I'll show an usefull simplification to use in your implementation. 

Another step in the simulation is to map the from plane coordinates to world space using the matrix 

$$ 

A = 

\begin{bmatrix} 
	du_1 & du_2 \\
	dv_1 & dv_2
\end{bmatrix}^{-1}

$$

The inverse of this 2-by-2 matrix can be computed as 

$$

A^{-1} = 

\frac {1} {\det A}

\begin{bmatrix} 
	dv_2 & -du_2 \\
	-dv_1 & du_1
\end{bmatrix}

$$

As in the paper, the world coordinates is

$$

\begin{bmatrix} 
	w_u & w_v
\end{bmatrix} 
= 
\begin{bmatrix} 
	dx_1 & dx_2
\end{bmatrix}

{A^{-1}}

$$

Now we define $$ a_{i, j} = {A^{-1}}_{i, j} $$. Hence, 

$$

\begin{bmatrix} 
	w_u & w_v
\end{bmatrix} 

= 

\begin{bmatrix} 
	dx_{1_x} a_{1,1} + dx_{2_x} a_{2,1}	& dx_{1_x} a_{1,2} + dx_{2_x} a_{2,2} \\
	dx_{1_y} a_{1,1} + dx_{2_y} a_{2,1}	& dx_{1_y} a_{1,2} + dx_{2_y} a_{2,2} \\
	dx_{1_z} a_{1,1} + dx_{2_z} a_{2,1}	& dx_{1_z} a_{1,2} + dx_{2_z} a_{2,2} \\
\end{bmatrix} 

$$

This can be written compactly as 

$$

\begin{bmatrix} 
	w_u & w_v
\end{bmatrix} 

= 

\begin{bmatrix} 
	dx_1 a_{1,1} + dx_2 a_{2,1} & dx_1 a_{1,2} + dx_2 a_{22} 
\end{bmatrix} 

$$

This equation can be further simplified to 

$$

\begin{bmatrix} 
	w_u & w_v
\end{bmatrix} 

= 

\frac {1} {\det A}

\begin{bmatrix} 
	dx_1 dv_2 - dx_2 dv_1 & -dx_1 du_2 + dx_2 du_1
\end{bmatrix}

$$

## References

[1] David Baraff, Andrew Witkin. [Large Steps in Cloth Simulation](https://www.cs.cmu.edu/~baraff/papers/sig98.pdf).