---
layout: post
title: Cone Limit in Quaternion Space
mathjax: true
tags: []
---

Welcome back to the joint articles.

Today's problem is to implement a stable cone joint in quaternion space. This is an elegant way of implementing rotational constraints. 
In this post I'll show you how this can be done in joint space. I also show how to apply a twist limit along the cone axis.

Commonly the conical joint limit is modeled as a constraint 

$$
C = u_2 \cdot u_1 - \cos{\theta} > 0 \\
\dot{C} = u_2 \cdot { {\omega}_1 \times u_1 } + u_1 \cdot { {\omega}_2 \times u_2 }
$$

where $u_1$ and $u_1$ are the cone axis stored in the frame of body $A$ and $B$ respectively mapped to world space and $\theta$ is the half cone angle.

The Jacobians can be identified via inspection:

Rewrite:

$$
\dot{C} = 
J v =
{ u_1 \times u_2 } \cdot { {\omega}_1 } + { u_2 \times u_1 } \cdot { {\omega}_2 } = 
{ -u_2 \times u_1} \cdot { {\omega}_1 } + { u_2 \times u_1 } \cdot {\omega}_2 \\
$$

Identify:

$$

J = 

\begin{bmatrix}
	0 & -n & 0 & n
\end{bmatrix}

$$

where $ n = u_2 \times u_1 $.

Of course in practice we can derive the position constraint using $\text {atan2}$ since it returns the full cone angle and checks for sine division by zero:

$$
C = \theta - \text{atan2}( {\|u_2 \times u_1\| }, { u_2 \cdot u_1 } ) > 0
$$

Using quaternions things are more complex but still very easy to visualize. 

Here we use quaternions in block form. It has a vector part $\boldsymbol v$ and a scalar part $s$: 

$$
q = 

\begin{bmatrix}
	\boldsymbol v & s
\end{bmatrix}

=

\begin{bmatrix}
	v_x & v_y & v_z & s
\end{bmatrix}

$$

Also, the quaternion inverse denotes the conjugate quaternion. 

The joint rotation is:

$$
q = {q_0}^{-1} {f_A}^{-1} f_B
$$

$f_A$ and $f_B$ are the orientations of both joint frames in world space.

It's well known that any rotation can be decomposed into a swing and twist rotation. The swing can happen before the twist. However, 
typically the swing comes after the twist. Therefore,

$$
q = q_{\text{swing}} q_{\text{twist}}
$$

Here $q_{\text{swing}}$ is a swing rotation and $q_{\text{twist}}$ is a twist rotation. 

Having those quaternions computed, we can apply the twist and swing limit to those and use the above formula to recover the clamped quaternion.

In non-singular scenarios, if we chose the twist quaternion to be the $x$-axis, the twist can be computed as:

$$

q_{\text{twist}} = 
\frac {1} {s}
\begin{bmatrix}
	q_{v_x} & 0 & 0 & q_s
\end{bmatrix}

$$

where

$$
s = \sqrt { {q_{v_x}}^2 + {q_s}^2 }
$$

which is simply taking a quaternion with the $x$ and $s$ components of $q$ and normalizing the resulting quaternion.

Solving for the swing we have:

$$
q_{\text{swing}} = q {q_{\text{twist}}}^{-1}
$$

With this quaternion the full twist angle can be extracted with the help of $\text{atan2}$:

$$
\alpha = 2 \text{atan2}(q_{v_x}, q_s)
$$

Now that we have both quaternions and twist angle we can apply the limits.

The twist limit along the $x$-axis can be applied by checking $\alpha$ is in the range $[lo, hi]$ and clamping it to that range. 
Therefore, 

$$

{ { q_{\text{twist}} }_v }_x = \sin { \frac {lo} 2 } \\
{ q_{\text{twist}} }_s = \cos { \frac {lo} 2 } \\

\text{if} \\

\alpha \leq lo

$$

or

$$

{ { q_{\text{twist}} }_v }_x = \sin { \frac {hi} 2 } \\
{ q_{\text{twist}} }_s = \cos { \frac {hi} 2 } \\

\text{if} \\

\alpha \geq hi

$$

The swing limit is a little more complicated. 

The swing cone limit can be applied by clamping the 2D swing vector part (here, $(q_{v_y}, q_{v_z})$) of $q$ to a 2D circle and more generally to a 2D ellipse.
However, the latter is more complicated because we will have to find the closest point from a point to an ellipse. Which means analitically solving a 4-th order polynomial 
or numerically solving for it using a suitable root finding algorithm.
Clamping to other shapes numerically can be done using GJK.
Therefore, I hope you know how to clamp a 2D point to a 2D circle (:

The circle radius is 

$$
r = \sin { \frac { \beta} {2} }
$$

where $\beta$ is the half cone angle. The circle center is at the origin $(0, 0)$ of course. Therefore, 

$$

{ { { q_{\text{swing}} }_v }_y } = r 
\frac { { { q_{\text{swing}} }_v }_y } { \sqrt { { { { q_{\text{swing}} }_v }_y }^2 + { { { q_{\text{swing}} }_v }_z }^2 } } \\

{ { { q_{\text{swing}} }_v }_z } = r 
\frac { { { q_{\text{swing}} }_v }_z } { \sqrt { { { { q_{\text{swing}} }_v }_y }^2 + { { { q_{\text{swing}} }_v }_z }^2 } } \\

\text{if} \\

{ { { q_{\text{swing}} }_v }_y }^2 + { { { q_{\text{swing}} }_v }_z }^2 > r^2

$$

Now we have both quaternions clamped we can recompose the clamped joint quaternion:

$$
q = q_{\text{swing}} q_{\text{twist}}
$$

Finally, we can write a working implementation for rotational cone-twist joint limits in quaternion space which supports full angles:

{% highlight cpp %}

b3Quat q1 = b3Conjugate(m_referenceRotation) * b3Conjugate(fA) * fB;

b3Quat q = q1;

// Make sure the scalar part is positive.
if (q.s < 0.0f)
{
	// Negating it still represents the same orientation.
	q = -q;
}

// Twist
float32 x, xs;

// Swing
float32 y, z;

float32 s = b3Sqrt(q.v.x * q.v.x + q.s * q.s);
if (s < B3_EPSILON)
{
	// Swing by 180 degrees is singularity.
	// Assume the twist is zero.
	x = 0.0f;
	xs = 1.0f;
	
	y = q.v.y;
	z = q.v.z;
}
else
{
	float32 r = 1.0f / s;

	x = q1.v.x * r;
	xs = q1.s * r;

	y = (q.s * q.v.y - q.v.x * q.v.z) * r;
	z = (q.s * q.v.z + q.v.x * q.v.y) * r;
}

// Solve twist
{
	// Joint angle
	float32 angle = 2.0f * atan2(q1.v.x, q1.s);
	
	if (b3Abs(m_upperAngle - m_lowerAngle) < B3_EPSILON)
	{
		float32 theta = 0.5f * m_lowerAngle;

		x = sin(theta);
		xs = cos(theta);
	}
	else if (angle <= m_lowerAngle)
	{	
		float32 theta = 0.5f * m_lowerAngle;

		x = sin(theta);
		xs = cos(theta);
	}
	else if (angle >= m_upperAngle)
	{
		float32 theta = 0.5f * m_upperAngle;

		x = sin(theta);
		xs = cos(theta);
	}
}

// Clamped twist
b3Quat qt(x, 0.0f, 0.0f, xs);

// Solve swing
{
	// Half angle
	float32 angle = 0.5f * m_coneAngle;

	// Circle radius
	float32 r = sin(0.5f * angle);

	// Circle clamp
	b3Vec2 p(y, z);
	if (b3LengthSquared(p) > r * r)
	{
		p.Normalize();
		
		p *= r;

		y = p.x;
		z = p.y;
	}
}

// Clamped swing
// The scalar part is positive so no need to multiply the scalar part by a sign
b3Quat qs(0.0f, y, z, b3Sqrt(b3Max(0.0f, 1.0f - y * y - z * z)));

b3Quat q2 = qs * qt;

{% endhighlight %}

In physics such a routine is usefull since eventually some constraints requires that we compute the quaternion (or angular velocity) that will rotate the 
current joint rotation to a target constrained rotation. Cheaply we can use finite differences. This is not as exact as quaternion constraints but can be 
elegant and simple to implement. The Jacobians and constraint effective masses of the quaternion constraints are not so trivial to derive. 

That's it. I hope this post was not very confusing. 

**Note**: If you are looking for a more general framework for applying limits in quaternion space I recommend this nice project by Gino from which I got some inspiration
to write this post:

[https://github.com/dtecta/motion-toolkit/](https://github.com/dtecta/motion-toolkit/)