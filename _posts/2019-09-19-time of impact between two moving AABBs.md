---
layout: post
title: Time of Impact between two moving AABBs
mathjax: true
tags: []
---

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/touching_1.png) 
{: refdef}

I recently needed a function for computing the time of impact (TOI) between two moving AABBs (Axis Aligned Bounding Box). 

So quickly I scanned through [1] expecting a full implementation of this algorithm or at least some direction into solving this problem.

I found a good direction, not the implementation. This is why I am writing this short blog post today. 

Of course, I found other implementation in the game development forums. However, they are incomplete and their explanation was not clear. This showed lack of understanding of 
the authors. Therefore, I decided to dissect the algorithm in [1] and write up my own version.

Here is my version of the algorithm:

{% highlight cpp %}

struct b3TOIOutput
{
	enum State
	{
		e_overlapped,
		e_touching,
		e_separated
	};

	State state;
	float32 t;
};

b3TOIOutput b3TimeOfImpact(const b3AABB& A, const b3Vec3& dA, const b3AABB& B, const b3Vec3& dB)
{
	// If the shapes are overlapped, we give up on continuous collision.
	if (b3TestOverlap(A, B))
	{
		// Failure!
		b3TOIOutput output;
		output.state = b3TOIOutput::e_overlapped;
		output.t = 0.0f;
		return output;
	}
	
	b3Vec3 d = dB - dA;

	b3Vec3 t1(0.0f, 0.0f, 0.0f);
	b3Vec3 t2(1.0f, 1.0f, 1.0f);

	for (u32 i = 0; i < 3; ++i)
	{
		if (d[i] == 0.0f)
		{
			if (A.lowerBound[i] > B.upperBound[i] || A.upperBound[i] < B.lowerBound[i])
			{
				// Victory!
				b3TOIOutput output;
				output.state = b3TOIOutput::e_separated;
				output.t = 1.0f;
				return output;
			}
		}

		if (d[i] < 0.0f)
		{
			if (B.upperBound[i] < A.lowerBound[i])
			{
				// Victory!
				b3TOIOutput output;
				output.state = b3TOIOutput::e_separated;
				output.t = 1.0f;
				return output;
			}
			else
			{
				t2[i] = (A.lowerBound[i] - B.upperBound[i]) / d[i];
			}
		
			if (B.lowerBound[i] > A.upperBound[i])
			{
				t1[i] = (A.upperBound[i] - B.lowerBound[i]) / d[i];
				
				if (t1[i] >= 1.0f)
				{
					// Victory!
					b3TOIOutput output;
					output.state = b3TOIOutput::e_separated;
					output.t = 1.0f;
					return output;
				}
			}
		}

		if (d[i] > 0.0f)
		{	
			if (B.lowerBound[i] > A.upperBound[i])
			{
				// Victory!
				b3TOIOutput output;
				output.state = b3TOIOutput::e_separated;
				output.t = 1.0f;
				return output;
			}
			else
			{
				t2[i] = (A.upperBound[i] - B.lowerBound[i]) / d[i];
			}
		
			if (B.upperBound[i] < A.lowerBound[i])
			{
				t1[i] = (A.lowerBound[i] - B.upperBound[i]) / d[i];

				if (t1[i] >= 1.0f)
				{
					// Victory!
					b3TOIOutput output;
					output.state = b3TOIOutput::e_separated;
					output.t = 1.0f;
					return output;
				}
			}
		}
	}

	float32 t1_max = b3Max(t1.x, b3Max(t1.y, t1.z));
	float32 t2_min = b3Min(t2.x, b3Min(t2.y, t2.z));

	if (t1_max > t2_min)
	{
		// Failure!
		b3TOIOutput output;
		output.state = b3TOIOutput::e_separated;
		output.t = 0.0f;
		return output;
	}

	// Victory!
	b3TOIOutput output;
	output.state = b3TOIOutput::e_touching;
	output.t = t1_max;
	return output;
}

{% endhighlight %}

Let us start explaining the algorithm above.

Fist of all, as in any TOI problem, the TOI for AABBs is the problem of finding the fraction on the displacement at the largest distance the objects can travel before overlapping. 

Let us call those bounding boxes $A$ and $B$. 

The first step in the algorithm is to check if the AABBs are overlapping. If they aren't the TOI can be 
determined. Otherwise we can quit the algorithm and return false because the TOI can't be determined.

If the two AABBs have displacements $d_A$ and $d_B$ of their center of masses then we can reduce this problem to the problem of finding the TOI 
between a dynamic AABB $B$ and a static AABB $A$ by putting $d_B$ relative to $d_A$.

Fortunately, for two AABBs, the TOI along a cardinal axis can be computed using simple interval algebra. 

Here, we call $t_1$ the TOI at which $B$ begins overlapping $A$. We name $t_2$ the TOI at which $B$ ends overlapping $A$.

Then for each axis we need to test against some conditions. 
The first case we test is when there is no displacement in the axis. In this case we need to check if the AABBs are overlapping in the axis. 
If they are, we must continue our search. Here is how this could look like in a touching configuration. 
The line at the center represents the displacement vector.

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/touching_3.png) 
{: refdef}

Otherwise the AABBs are clearly separated. Below is one configuration in which this happens.

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/separating_2.png) 
{: refdef}

If $t_1$ or $t_2$ can be computed, the formulas for the (positive) TOIs along an axis depend on the sign of $d$ 
along that axis and the positions of the intervals. So there are a few if checks to ensure the resulting TOI is positive. 
Otherwise, if we can't compute the TOIs for that axis they will be set to the minimum or maximum possible TOI, that is, $0$ or $1$.

If we can't compute $t_2$ for an axis, we check if the interval $B$ is moving away from interval $A$. If it is the AABBs are clearly 
separated on that axis and we can quit the algorithm.

If we can compute $t_1$ for an axis, then we can check if it exceeds the maximum possible TOI: $1$. If it does, then the AABB $B$ is separated along the direction and 
we can quit the algorithm returning false.

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/separating_3.png) 
{: refdef}

Now that we have computed $t_1$ and $t_2$ for each axis, we need to find the TOI at the *largest distance B can travel before overlapping*. 
This is the output TOI if the AABBs will touch during the motion. This means taking the maximum of all $t_1$ values.

We also need to find the minimum TOI at which $B$ ends overlapping $A$, choosing the minimum of all $t_2$ values.

With $t_1$ and $t_2$, we can test whether the AABBs will overlap during the motion. 
For this, we check if the maximum TOI at which $B$ begins overlapping $A$ is greater than the minimum TOI at which $B$ ceases overlapping $A$. 
If it is, the AABBs are separated. 
The following picture illustrates this idea. As before, it uses lines for representing maximum $t_1$ and minimum $t_2$. 

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/separating_1.png) 
{: refdef}

Otherwise the AABBs will touch during the motion and we can return true and the TOI. This can be seen in the picture below:

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/touching_2.png) 
{: refdef}

That's it. I hope this short post helps if you're writing this function (:

**Note**: Christer's code can be faster as it tracks the maximum and minimum TOI but it doesn't handle cases where the displacement is zero. 

But whatever... Just like any code in that book, it must first be understood and then fixed and used.

## References

[1] Christer Ericson. *Real Time Collision Detection*