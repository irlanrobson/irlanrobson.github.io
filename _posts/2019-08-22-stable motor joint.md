---
layout: post
title: Stable Motor Joint
mathjax: true
tags: []
---

The problem is to implement a stable motor joint. Commonly a motor joint is modeled as constraint which controls the relative motion of two bodies.

Therefore, one could write a constraint in terms of quaternion differences such as:

$$
q_C = q_0 {q_A}^{-1} q_B
$$

where $q_A$ and $q_B$ are the orientations of body $A$ and body $B$ and $q_0$ is the angular offset which is the orientation of body $B$ in body $A$'s frame (the 
target relative orientation).

Here I assume the inverse symbol represents the conjugate of the quaternion.

Then we could build the Jacobians using a projector operator or some simplification of some kind as usual. 
However, I will show that this is not necessary if you are solving this constraint at the velocity level. 
The velocity constraint has the following form:

$$
\dot{C} = {\omega}_B - {\omega}_A - {\omega}_0
$$

where ${\omega}_A$ and ${\omega}_B$ are the angular velocities of the bodies.

Here ${\omega}_0$ is the target relative angular velocity and we need to find it.

Therefore, we can pose the problem as follows. Find the relative angular velocity given the body rotations and angular offset. For this we can apply the finite difference method:

Define 

$$
q_1 = {q_A}^{-1} q_B \\
q_2 = q_0
$$

Then check the polarity, flip one of the quaternions if it is needed case and use the formula

$$
q_{\omega} = \frac { 2 (q_2 - q_1) {q_1}^{-1} } { h }
$$

where $h$ is the time step. 

The vector part of $q_{\omega}$ is the angular velocity that will rotate $q_1$ to $q_2$ over the time step $h$. 
Substitute the angular velocity in $\dot{C}$.

**Note**: The angular velocity is in body $A$'s frame. Therefore, we must convert it to world space.

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