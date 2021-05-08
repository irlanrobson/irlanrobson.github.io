---
layout: post
title: Strech Forces in Cloth Simulation
mathjax: true
tags: []
---

If you're implementing the continuum approach to cloth simulation described in [1] you will find this article pretty useful.

From now on, I assume you're familiar with the paper [1]. 
Here we will see how we can reliably include the stretch forces into the system formulated in there without introducing problems into the system.

Let us start the explanation.

In the paper there are three forces that are included in order to simulate the internal cloth behaviour.
Those are stretch, shear, and bend forces. 
Here we will focus in on the strech force. 
We might talk about other forces into the future as their safe inclusion into the system can become more 
involved and require an excellent mathematical background. 

The stretch condition function is defined as

$$

C(\boldsymbol x) = 

\alpha 

\begin{bmatrix}
	\|{\boldsymbol w}_u\| - b_u \\
	\|{\boldsymbol w}_v\| - b_v
\end{bmatrix}

$$

Now we describe the equation above.

In this equation $\alpha$ is typically the initial triangle area in a $(u, v)$ space.

${\boldsymbol w}_u$ and ${\boldsymbol w}_v$ are the mapped world coordinates from the $(u, v)$ space. I showed how we can reliably define the $u$ and $v$ values in triangle plane space and 
how to convert from plane coordinates to world coordinates in 
[**this post**](https://irlanrobson.github.io/2019/06/29/computing-(u,v)-coordinates-for-a-triangle/). 

$b_u$ and $b_v$ are the desired stretchiness in the $u$, and $v$ directions.

Now we can make some observations. 

The first thing to notice in the expression above is that they pack two constraint equations. One equation for the $u$ direction, and other equation for the $v$ direction.

$$

C_u(\boldsymbol x) = 

\alpha 

\begin{bmatrix}
	\|{\boldsymbol w}_u\| - b_u \\
\end{bmatrix}

\\

C_v(\boldsymbol x) = 

\alpha 

\begin{bmatrix}
	\|{\boldsymbol w}_v\| - b_v
\end{bmatrix} \\

$$

Therefore, their corresponding forces must be included into the system independently of each other. 
In all the papers and implementations for the simulation I could read on the Internet people include the forces for those equations together into the system.
This is not correct as the condition in a direction might make the system indefinite, but the other might not. 
We will talk about this shortly in a moment. 

This separation also makes possible for us to control the stiffness coefficients anisotropically as we do for stretchiness. 
We can define two stiffness coefficients $ k_{s_u} $, and $ k_{s_v} $ both which are the coefficients for the $u$, and $v$ directions. 
Similarly we can define two damping coefficients $k_{d_u}$, and $k_{d_v}$.

The Jacobians and force derivatives for the stretch forces are described in [3]. 
However, if you're following this paper notice that the forces are not introduced correctly in the author's engine, 
namely the project [freecloth](http://davidpritchard.org/freecloth/). 

[VEGA FEM](http://run.usc.edu/vega/) also incorrectly includes the stretch force into the system.

**Note**: I am assuming that those engines are using the Conjugate Gradient method (CG) which converges for symmetric and positive definite matrices. 
*freecloth* supports adaptive integration and *VEGA* different linear system solvers. So the authors may be aware of the problems and may have tried to solve them in a 
different way.

The constraint Jacobian is:

$$
{ \frac {\partial C_u} {\partial {\boldsymbol x}_i} } = \alpha { \frac {\partial {\boldsymbol w}_u} {\partial {\boldsymbol x}_i} } {\hat {\boldsymbol w}_u}
$$

where

$$

{ \hat {\boldsymbol w}_u } = \frac { {\boldsymbol w}_u } {\|{\boldsymbol w}_u\|} \\

{ \frac {\partial {\boldsymbol w}_u} {\partial \boldsymbol{x}} } =

\frac {1} {\det A}

\begin{bmatrix}
	dv_1 - dv_2 & dv_2 & -dv_1
\end{bmatrix} \\

$$

The force Jacobian is:

$$

\frac {\partial {\boldsymbol f}_i} { \partial {\boldsymbol x}_j } = 
K_{i,j} = 
-k_{s_u} 
( { \frac {\partial C_u} {\partial {\boldsymbol x}_i} } { \frac {\partial C_u} {\partial {\boldsymbol x}_j} }^T + 
C_u 
\frac {\partial^2{C_u}} {\partial {\boldsymbol x}_i \partial {\boldsymbol x}_j })

$$

where 

$$

\frac {\partial^2{C_u}} { \partial {\boldsymbol x}_i \partial {\boldsymbol x}_j } = 

\alpha 

( 
{ \frac {1} {\|{\boldsymbol w}_u \|} } 
{ \frac {\partial {\boldsymbol w}_u} {\partial {\boldsymbol{x}}_i} }
{ \frac {\partial {\boldsymbol w}_u} {\partial {\boldsymbol{x}}_j} } 
) 

(
\boldsymbol{I} - {\hat {\boldsymbol w}_u} {\hat {\boldsymbol w}_u^T}
)

$$

Similarly we can define the Jacobians for $C_v$.

We now can make some observations.

CG converges for matrices which are symmetric and positive definite. 

When including the stretch forces symmetry is guaranteed unconditionally.

The first term of $K_{i,j}$ is symmetric because the Jacobians are paralell to each other. 
Remember that the outer product between two paralell vectors is a symmetric matrix. 
Therefore $K_{i,j}$ is a symmetric matrix because the second term $ \frac {\partial^2{C(\boldsymbol {x})}} { \partial \boldsymbol x_i \partial \boldsymbol x_j } $ is also a symmetric matrix.
The sum of two symmetric matrices yields a symmetric matrix.

However, positive definiteness is not guaranteed under some conditions.
More precisely handling a compression force might turn the system matrix into indefinite since $C_u$ is negative and can be arbitrarly large independently of the time-step.
This can result in the simulation blowing up, exploding, CG not converging at all.
The second term is important because it introduces large positive eigenvalues in the system matrix. So we cannot toss it out from the equation to avoid breaking positive-definiteness.
Also, we cannot remove it from the strech force because that is the main cloth internal force. We could, but this would lead to instabilities for large stretch or large time-step.

All that said, we should only include the strech forces in the $u$ direction only if the following condition is satisfied:

$$
\|{\boldsymbol w}_u(\boldsymbol {x})\| > b_u
$$

to ensure $C_u > 0$.

Similarly we can include the forces for the condition in the $v$ direction if

$$
\|{\boldsymbol w}_v(\boldsymbol {x})\| > b_v
$$

This ensures $C_v > 0$.

Of course the constraint Jacobians are not degenerate under the following conditions.

$$
\|{\boldsymbol w}_u(\boldsymbol {x})\| > 0 \\
\|{\boldsymbol w}_v(\boldsymbol {x})\| > 0
$$

If you need somewhat to include compression forces in your cloth simulation, the work of [2] is usually recommended as a good approach to the problem.

I think that's a wrap. 

Hopefully this post will help someone to reliably implement Baraff's cloth simulation.

## References

[1] David Baraff, Andrew Witkin. [Large Steps in Cloth Simulation](https://www.cs.cmu.edu/~baraff/papers/sig98.pdf).

[2] Kwang-Jin Choi, Hyeong-Seok Ko. [Stable but Responsive Cloth](http://graphics.snu.ac.kr/~kjchoi/publication/cloth.pdf).

[3] David Pritchard. [Implementing Baraff & Witkin's Cloth Simulation](http://davidpritchard.org/freecloth/docs/report.pdf).
