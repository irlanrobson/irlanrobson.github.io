---
layout: post
title: Strech Forces in Cloth Simulation
tags: []
---

If you're implementing the continuum approach to cloth simulation described in [1] you will find this article pretty useful.

From now on, I assume you're familiar with the paper [1]. 
Here we will see how we can reliably include the strech forces into the system formulated in there without introducing problems into the system.

For simplicity, I won't be using LaTex.
I will write equations in form of code. 
This is much less time-consuming for me and it doesn't make it difficult for one to understand the explanation.

Let us start the explanation.

In the paper there are three forces that are included in order to simulate the internal cloth behaviour.
Those are stretch, shear, and bend forces. 
Here we will focus in on the strech force. 
We might talk about other forces into the future as their safe inclusion into the system can become more 
involved and require an excellent mathematical background. 

The stretch condition function is defined as

{% highlight cpp %}

C(x) = alpha * [||w_u(x)|| - b_u]
               [||w_v(x)|| - b_v]

{% endhighlight %}

Now we describe the equation above.

In this equation alpha is the initial triangle area in a (u, v) space.

w_u and w_v are the mapped world coordinates from the (u, v) space. I showed how we can reliably define the u and v values in triangle plane space and 
how to convert from plane coordinates to world coordinates in 
[**this post**](https://irlanrobson.github.io/2019/06/29/computing-(u,v)-coordinates-for-a-triangle/). 

b_u and b_v are the desired stretchiness in the u, and v directions.

Now we can make some observations. 

The first thing to notice in the expression above is that they pack two constraint equations. One equation for the u direction, and other equation for the v direction.

{% highlight cpp %}

Cu = alpha * (||w_u(x)|| - b_u)  
Cv = alpha * (||w_v(x)|| - b_v)

{% endhighlight %}

Therefore, their corresponding forces can (and should) be included into the system independently of each other. 
In all the papers and implementations for the simulation I could read on the Internet people include the forces for those equations together into the system.
This is not correct as the condition in a direction might make the system indefinite, but the other might not. 
We will talk about this shortly in a moment. 

This separation also makes possible for us to control the stiffness coefficients anisotropically as we do for strechiness. 
We can define two stiffness coefficients ks_u, and ks_v both which are the coefficients for the u, and v directions. 
Similarly we can define two damping coefficients kd_u, and kd_v.

The Jacobians and force derivatives for the stretch forces are described in [3]. 
However, if you're following this paper notice the forces are not handled independently neither introduced correctly in the author's engine, 
namely the project [freecloth](http://davidpritchard.org/freecloth/). 

[VEGA FEM](http://run.usc.edu/vega/) also incorrectly includes the stretch force into the system.

Here's how one computes the Jacobians of the constraint Cu:

{% highlight cpp %}

b3Vec3 dCudx[3];
for (u32 i = 0; i < 3; ++i)
{
	dCudx[i] = alpha * dwudx[i] * n_wu;
}

{% endhighlight %}

In the code above 

{% highlight cpp %}

n_wu = wu / ||wu|| 

{% endhighlight %}

dwdudx is a constant term and can be cached. It's defined as:

{% highlight cpp %}

b3Vec3 dwudx;
dwudx[0] = inv_det * (dv1 - dv2);
dwudx[1] = inv_det * dv2;
dwudx[2] = -inv_det * dv1;

{% endhighlight %}

Similarly we can define the Jacobians for Cv.

Here's how one computes the nine force derivatives in the u direction:

{% highlight cpp %}

b3Mat33 K[3][3];
for (u32 i = 0; i < 3; ++i)
{
	for (u32 j = 0; j < 3; ++j)
	{
		b3Mat33 d2Cuxij = (alpha * inv_len_wu * dwudx[i] * dwudx[j]) * (I - b3Outer(n_wu, n_wu));

		b3Mat33 Kij = -m_ks_u * (b3Outer(dCudx[i], dCudx[j]) + Cu * d2Cuxij);

		K[i][j] = Kij;
	}
}

{% endhighlight %}

Similarly we can define the force derivatives for Cv.

We now can make some observations.

The Conjugate Gradient (CG) method converges for matrices which are symmetric and positive definite. 

When including the stretch forces symmetry is guaranteed unconditionally.

The first term of Kij is symmetric because the Jacobians are paralell to each other. 
Remember that the outer product between two paralell vectors is a symmetric matrix. 
Therefore Kij is a symmetric matrix because the second term d2Cuxij is also a symmetric matrix.
The sum of two symmetric matrices yields a symmetric matrix.

However, positive definiteness is not guaranteed under some conditions.
More precisely handling a compression force might turn the system matrix into indefinite since Cu is negative and can be arbitrarly large independently of the time-step.
This can result in the simulation blowing up, exploding, CG not converging at all.
The second term is important because it introduces large positive eigenvalues in the system matrix. So we cannot toss it out from the equation to avoid breaking positive-definiteness.
In particular we cannot remove it from the strech force because that is the main cloth internal force.

All that said, we should only include the strech forces in the u direction only if the following condition is satisfied:

{% highlight cpp %}

||w_u(x)|| > b_u

{% endhighlight %}

to ensure Cu > 0.

Similarly we can include the forces for the condition in the v direction if

{% highlight cpp %}

||w_v(x)|| > b_v

{% endhighlight %}

This ensures Cv > 0.

Of course the Jacobians are not degenerate under the following conditions.

{% highlight cpp %}

||w_u(x)|| > 0
||w_v(x)|| > 0

{% endhighlight %}

If you need somewhat to include compression forces in your cloth simulation, the work of [2] is usually recommended as a good approach to the problem.

(Then someone asks...)

**Dude, where is the complete source code?**

Dude, fortunately it happens that theres a complete logic for the inclusion of the strech force
implemented [**here**](https://github.com/irlanrobson/bounce/blob/master/src/bounce/cloth/forces/strech_force.cpp).
(You can browse the project files for seeing an implementation for a modified version of the entire cloth simulator described in [1].)
In this implementation all the edge cases for the inclusion of the stretch forces are handled correctly which results in good and trustable cloth simulation.
The only thing is missing is supporting anisotropic stiffness coefficients. I might add that in the future.

I think that's a wrap. 

Hopefully this post will help someone to reliably implement Baraff's cloth simulation.

## References

[1] David Baraff, Andrew Witkin. [Large Steps in Cloth Simulation](https://www.cs.cmu.edu/~baraff/papers/sig98.pdf).

[2] Kwang-Jin Choi, Hyeong-Seok Ko. [Stable but Responsive Cloth](http://graphics.snu.ac.kr/~kjchoi/publication/cloth.pdf).

[3] David Pritchard. [Implementing Baraff & Witkin's Cloth Simulation](http://davidpritchard.org/freecloth/docs/report.pdf).