---
layout: post
title: Stable Motor Joint
tags: []
---

The problem is to implement a stable motor joint. Commonly a motor joint is modeled as constraint which controls the relative motion of two bodies.

Therefore, one could write a constraint in terms of quaternion differences such as:

qC = conjugate(angular_offset) * conjugate(qA) * qB

where qA and qB are the orientations of both bodies and angular_offset is the angular offset which is the orientation of body B in body A's frame (the 
target relative orientation).
Then we could build the Jacobians using a projector operator or some simplification of some kind as usual. 
However, I will show that this is not necessary if you are solving this constraint at the velocity level. 
The velocity constraint has the following form:

{% highlight cpp %}

Cdot = (wB - wA) - angular_velocity

{% endhighlight %}

where wA and wB are the angular velocities of the bodies.

Here angular_velocity is the target relative angular velocity and we need to find it.

Therefore, we can pose the problem as follows. Find the relative angular velocity given the body rotations and angular offset. For this we can apply the finite difference method:

Define 

{% highlight cpp %}

q1 = conjugate(qA) * qB
q2 = angular_offset

{% endhighlight %}

Then check the polarity, flip one of the quaternions if it is needed case and use the formula

{% highlight cpp %}

qW = inv_h * 2.0f * (q2 - q1) * conjugate(q1)

{% endhighlight %}

where inv_h is the inverse time step. 

The vector part of qW is the angular velocity will rotate q1 to q2 over the time step h. 
Substitute the angular velocity in Cdot.
Note: The angular velocity is in body A's frame. So we need to convert it to world space.

In code that could be written as:

{% highlight cpp %}

b3Quat q1 = b3Conjugate(qA) * qB;
b3Quat q2 = m_angularOffset;

float32 sign = b3Sign(b3Dot(q1, q2));

q1 = sign * q1;

// Apply a finite difference
b3Quat qw = inv_h * 2.0f * (q2 - q1) * b3Conjugate(q1);

// Angular velocity that will rotate q1 to q2 over h
m_angularVelocity.x = qw.x;
m_angularVelocity.y = qw.y;
m_angularVelocity.z = qw.z;

// Convert the relative angular velocity to world space
m_angularVelocity = b3Rotate(qA, m_angularVelocity);

m_angularVelocity *= m_correctionFactor;

// The corresponding effective mass is just the sum of the world space inverse inertia tensors
m_angularMass = m_iA + m_iB;

{% endhighlight %}