---
layout: post
title: Mouse Constraint Derivation
tags: []
---

You can read [here](/assets/mouse_constraint.pdf) how to compute the mouse constraint Jacobian, which I think is the simplest constraint along with a distance constraint. It can be helpfull when you're selecting geometries in the scene using the mouse world space position (as a joint anchor point), since there is lack of controlability by directly applying forces at a particular point of a rigid body shape.

Errata: There is a minus sign before the stabilization term. It should be:

Lambda = (J * M * J^T)-1 * (-J * V - Beta / TimeStep * C)

Note: You can bound excessive impulses to avoid excessive velocity changes.

**Thanks to Dirk Gregorius of Valve for pointing those things out via e-mail.**

Practically, a single mouse constraint equation could be enforced as:

{% highlight cpp %}

	Vec3 P1 = GetWorldAnchorPosition(); // Could be anything static in R^3.
	Vec3 P2 = GetWorldMousePosition();

	Mat33 W2 = Diagonal( Body2->GetInverseMass() );
	Mat33 I2 = Body2->GetWorldInverseInertia();

	Vec3 COM2 = Body2->GetWorldCenterOfMass();
	Vec3 Velocity2 = Body2->GetLinearVelocity();
	Vec3 Omega2 = Body2->GetAngularVelocity();

	Vec3 C = P2 - P1;
	Vec3 Radius2 = P2 - COM2;
	Vec3 Cdot = Velocity2 + Cross(Omega2, Radius2);

	Mat33 R2 = Skew(Radius2);
	Mat33 R2T = Transpose(R);
	Mat33 K = W2 + (R2 * I2 * R2T);
	Mat33 M = Inverse(K);

	Vec3 VelocityBias = Beta / TimeStep * C;

	Vec3 Impulse = M * (-Cdot + VelocityBias);

	Body2->ApplyLinearImpulseAtPoint(Impulse, P2); 

{% endhighlight %}

And here is the result:

<iframe width="560" height="315" src="https://www.youtube.com/embed/CX7ml-p-bx4" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
