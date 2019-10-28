---
layout: post
title: Fast Segment-AABB Collision Detection
mathjax: true
tags:
- Collision
- Math
---

Let's say we need to compute the intersection point and normal from a line segment to an axis-aligned bounding box (AABB) bounds, and we neither want to perform expensive plane computations nor keep the AABB boundaries as an intersection of 6 axis-aligned planes (as it is a waste of memory). A quick way to do that is by evaluating the slab plane equation for each axis-aligned plane of the AABB.

### Derivation

The plane equation for a slab is simpler than the plane's in that the signed distance $d_i$ from any point to the slab plane in a certain direction is just the point coordinate in that direction $p_i$ minus the slab plane offset from the origin in that direction $w_i$.

$$ w_i = {n_i} \cdot {p_i} \\ 
w_i = s p_i \\
i = 1..6 $$

Here the sign variable $s$ is either $-1$ or $1$. Either the positive slab direction or the negative. If we store the lower and upper bounds of the AABB then we can get the intersection fraction from a slab plane to a point without creating the six planes of an AABB.

Therefore, the signed distance in the i-th direction is

$$ d_i = s p_i - w_i $$

A common segment definition is  

$$ S = P + t D $$

Where $D = Q - P$. Let us plug that definition in the plane equation. 

$$ 
{n} \cdot (P + t D) = w
$$

Solving for $t$ (unbounded), we get:

$$
{ n \cdot P } + t {n \cdot D} = w \\
t = \frac { w - { n \cdot P } } { n \cdot D }
$$

Now simplifying using the slab equation, for the i-th plane:

$$ t_i = \frac { w_i - s P_i } { s D_i } $$

For example, the intersection fraction for the upper bound $u$ of an AABB in the world up direction to a ray, is:

$$ t_y = \frac { u_y - P_y } { D_y } $$

Of course, if $D_y = 0$ then the ray is paralell to the upper plane and we should do something in this case.

For the intersection test, we can basically use the same method showed on Christer Ericson's excelent book Real Time Collision Detection, at page 180, for testing a segment with a convex polyhedron.┬áThe method finds the minimum intersection fraction with the ray given that during the process it performs simple comparison operations to maintain the logical intersections along the ray. I'm not going to explain the method here, but it is very popular and I hopefully believe that it can be found around the web pretty quickly.

### Implementation

Here's my version of the method showed in the book, with a couple of improvements and explanations as well:

{% highlight cpp %}

// Returns false for a unbounded fraction.
bool Bounds3::RayCast(const Vec3& p1, const Vec3& p2, float maxFraction, float& minFraction) const
{
	Vec3 d = p2 - p1;

	float lower = 0.0f;
	float upper = maxFraction;

	// Solve segment to slab plane.
	// s = p1 + w * d
	// Slab plane equation:
	// dot(n[i], s) = offset[i]
	// Solution:
	// dot(n[i], p1 + w * d) = offset[i]
	// dot(n[i], p1) + w * dot(n[i], d) = offset[i]
	// w * dot(n[i], d) = offset[i] - dot(n[i], p1)
	// w = (offset[i] - dot(n[i], p1)) / dot(n[i], d)

	for (uint i = 0; i < 3; ++i)
	{
		// Evaluate equations for the near and far plane of the current axis.
		float numerators[2], denominators[2];
		// Note the negative signs used as dot products.
		// The sign (-m_lower[i]) below is for because is the
		// plane offset along the plane normal
		// and not aways along the positive axis!
		// We're basically evaluating the plane equation
		// for any corner on this near plane
		// (e.g. dot(corner[i], normal[i]) = offset).

		numerators[0] = (-m_lower[i]) - (-p1[i]);
		numerators[1] = (m_upper[i]) - (p1[i]);
		denominators[0] = -d[i];
		denominators[1] = d[i];

		// For each plane...
		for (uint j = 0; j < 2; ++j)
		{
			float numerator = numerators[j];
			float denominator = denominators[j];

			if (denominator == 0.0f)
			{
				// s is parallel to this half-space.
				if (numerator < 0.0f)
				{
					// s is outside of this half-space.
					// dot(n, p1) and dot(n, p2) &amp;lt; 0 is guaranteed.
					return false;
				}
			}
			else
			{
				float fraction = numerator / denominator;
				if (denominator < 0.0f)
				{
					// s enters this half-space.
					if (fraction > lower)
					{
						// Increase lower.
						lower = fraction;
					}
				}
				else
				{
					// s exits the half-space.
					if (fraction < upper)
					{
						// Decrease upper.
						upper = fraction;
					}
				}
				// Exit if intersection becomes empty.
				if (upper < lower)
				{
					return false;
				}
			}
		}
	}

	Assert(lower >= 0.0f && lower <= maxFraction);
	
	// Output minimum intersection fraction.
	// You must evaluate the segment equation to get the closest point on the bounds.
	minFraction = lower;

	return true;
}

{% endhighlight %}

If divisions are a bottleneck to the processor, then we can substitute the division at line 48 by optimized comparisons using just multiplications.

{% highlight cpp %}

//...
if (denominator < 0.0f)
{
	// s enters this half-space.
	if (numerator < lower * denominator)
	{
		// Increase lower.
		lower = numerator / denominator;
	}
}
else
{
	// s exits the half-space.
	if (numerator < upper * denominator)
	{
		// Decrease upper.
		upper = numerator / denominator;
	}
}
//...

{% endhighlight %}

### Conclusions

Finally, it happens that the code for polyhedron-segment intersection test is not much different from this one, and I'm really in favor of claricity-generality over performance in some cases.

In the polyhedron's case, we would be doing this operation for all the polyhedron hull planes. Except that we replace the numerator and denominator variables, for the i-th plane, as follows.

{% highlight cpp %}

float numerator = planes[i].m_offset - dot(planes[i].normal, p1)
float denominator = dot(planes[i].normal, d)

{% endhighlight %}

That's all.

By the way: Happy New Year to everyone that reads the blog!