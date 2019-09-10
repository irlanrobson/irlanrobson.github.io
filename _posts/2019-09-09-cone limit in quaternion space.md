---
layout: post
title: Cone Limit in Quaternion Space
tags: []
---

Welcome back to the joint articles.

Today's problem is to implement a stable cone joint in quaternion space. This is an elegant way of implementing rotational constraints. 
In this post I'll show you how this can be done in joint space. I also show how to apply a twist limit along the cone axis.

Commonly the conical joint limit is modeled as a constraint 

{% highlight cpp %}

C = dot(u2, u1) - cos(angle / 2) > 0
Cdot = dot(u2, omega1 x u1) + dot(u1, omega2 x u2)

{% endhighlight %}

where u1 and u1 are the cone axis stored in the frame of body A and B respectively mapped to world space.

By far the Jacobians are identified via inspection:

{% highlight cpp %}

dot(u1 x u2, omega1) + dot(u2 x u1, omega2) = 
dot(-u2 x u1, omega1) + dot(u2 x u1, omega2)
n = u2 x u1
J = [0 -n 0 n]

{% endhighlight %}

Of course in practice we can derive the position constraint using atan2 since it returns the full cone angle and checks for sine division by zero:

{% highlight cpp %}
C = angle / 2 - atan2( norm(u2 x u1), dot(u2, u1) ) > 0
{% endhighlight %}

Using quaternions things are more complex but still very easy to visualize. The joint rotation is:

{% highlight cpp %}
q = conjugate(reference_rotation) * conjugate(fA) * fB
{% endhighlight %}

where fA and fB are the orientations of both joint frames in world space.

It's well known that any rotation can be decomposed into a swing and twist rotation. The swing can happen before the twist. However, 
typically the swing comes after the twist. Therefore,

{% highlight cpp %}
q = qs * qt
{% endhighlight %}

Here qs is a swing rotation and qt is a twist rotation. 

Having those quaternions computed, we can apply the twist and swing limit to those and use the above formula to recover the clamped quaternion.

In non-singular scenarios, if we chose the twist quaternion to be the x-axis, the twist can be computed as:

{% highlight cpp %}

s = sqrt(q.v.x * q.v.x + q.s * q.s)

qt.v.x = q.v.x / s;
qt.v.y = 0
qt.v.z = 0
qt.s = q.s / s;

{% endhighlight %}

which is simply taking a quaternion with the x and s components of q and normalizing the resulting quaternion.

Solving for the swing we have:

{% highlight cpp %}
qs = q * conjugate(qt)
{% endhighlight %}

With this quaternion the full twist angle can be extracted with the use of atan2:

{% highlight cpp %}
angle = 2 * atan2(q.v.x, q.s)
{% endhighlight %}

Now that we have both quaternions and twist angle we can apply the limits.

The twist limit along the x-axis can be applied by checking if the current twist angle is in the range *[lower, upper]* and clamping it to that range. 
The following code shows how this can be done basically.

{% highlight cpp %}
float32 angle = 2 * atan2(q.v.x, q.s)

if (angle <= lower)
{
	float32 theta = 0.5f * lower;
	qt.v.x = sin(theta);
	qt.s = cos(theta);
}
else if (angle >= upper)
{
	float32 theta = 0.5f * upper;
	qt.v.x = sin(theta);
	qt.s = cos(theta);
}
{% endhighlight %}

The swing limit is a little more complicated. 

The swing cone limit can be applied by clamping the 2D swing vector part (here, (y, z)) of the quaternion to a 2D circle and more generally to a 2D ellipse.
However, the latter is more complicated because we will have to find the closest point from a point to an ellipse. Which means analitically solving a 4-th order polynomial 
or numerically solving for it using a suitable root finding algorithm.
Clamping to other shapes numerically can be done using GJK.
Therefore, I hope you know how to clamp a 2D point to a 2D circle (:.

The circle radius is 

{% highlight cpp %}
r = sin(half_cone_angle / 2)
{% endhighlight %}

The circle center is at the origin of course.

{% highlight cpp %}

// Half cone angle
float32 angle = 0.5f * m_coneAngle;

// Circle radius
float32 r = sin(0.5f * angle);

// Clamp p to circle with radius r at the origin
b3Vec2 p(y, z);
if (b3LengthSquared(p) > r * r)
{
	p.Normalize();
	
	p *= r;

	y = p.x;
	z = p.y;
}

{% endhighlight %}

Now since we have both quaternions clamped, we can recover the clamped quaternion:

{% highlight cpp %}
q = qs * qt
{% endhighlight %}

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
Sometimes it is much simpler to comment the code and provide an explanation than writing Latex scripts.

**Note**: If you are looking for a more general framework for applying limits in quaternion space I recommend this nice project by Gino from which I got some inspiration
to write this post:

[https://github.com/dtecta/motion-toolkit/](https://github.com/dtecta/motion-toolkit/)