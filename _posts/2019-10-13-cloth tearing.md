---
layout: post
title: Cloth Tearing
tags: []
---

Dynamic cloth tearing is an unexplored subject but can be an extremelly simple process to implement depending on what 
technique you choose to tear the cloth structure. This is why I decided to write this blog post to help you to implement cloth tearing through a 
very simple vertex splitting technique. 

# Cloth

I assume the simplest physical representation for a cloth. That is, a set of particles connected by springs (or constraints), and triangles. There is 
one spring for each triangle edge.

{:refdef: style="text-align: center;"}
![Cloth Representation](/assets/cloth.png) 
{: refdef}

In a particle system the triangles aren't necessary. They are used solely for rendering the cloth and also for other things like collision and 
ray-casting.

To make the algorithm simple to understand, I assume that you do not maintain a mesh adjacency structure for speeding up the process. 
However, the algorithm showed here can be extended for maintaning such a mesh. 

# Splitting Vertex

In order to implement tearing we will be using a vertex splitting algorithm. If you are familiar with computational geometry you probably know what a vertex splitting 
operation is. 

In this operation you choose a vertex to be split, create a new vertex, choose a set of edges and triangles to connect to the new vertex, and then connect the new vertex 
with the desired edges and triangles.

{:refdef: style="text-align: center;"}
![Single Vertex Splitting](/assets/single_vertex_splitting.png) 
{: refdef}

*I hope you enjoy my coder art. It was done entirely by myself.*

The vertex splitting technique is the simplest technique for tearing a cloth. There are more advanced techniques that involve surface subdivision. 
I have tried edge splitting, but I can't recommend this technique because it can introduce triangles to the cloth that do not contribute to the quality of the simulation.

# Splitting Plane

Now that we know what a vertex split operation is we need to define a splitting plane. In our physics context this plane is typically defined with the 
position of a particle and the particle constraint axis.

{:refdef: style="text-align: center;"}
![Splitting Plane](/assets/splitting_plane.png) 
{: refdef}

In this example there is a tension force acting between particles p1 and p3. The plane is defined as the tension direction and the position of particle p3.

The physical condition for detecting if a particle can be split can be very simple. 
For example, if the length of an edge or the tension force exceeds a threshold value, one of the particles which belong to the edge can be chosen as the splitting particle.

# Partitioning 

We need to determine the set of edges and triangles that will be attached to the new vertex originated from the splitting vertex.

As you can see in the last picture, the splitting plane can be used to separate the triangles in two sets. One set containing the triangles 
that are above the plane and the other containing the triangles that are below the plane. 

In practice those partitions can be created by classifying the center of each triangle that contains the splitting vertex against the splitting plane. 

{:refdef: style="text-align: center;"}
![Triangle Partition](/assets/partition.png) 
{: refdef}

# Vertex Splitting

The first case we need to handle is the case where one of those triangle sets are empty. In this case our vertex splitting routine does nothing. There are no vertices 
to split. Otherwise there is at least one triangle on each side of the plane and we must create a new vertex and start the feature attaching process explained below.

In this algorithm our convention is to attach the edges and triangles that are below the plane to the new vertex. Therefore, after we have 
created the new vertex, we need to process each triangle that is below the separating plane.

For each triangle that is below the plane we must replace the original splitting vertex with the newly created vertex as showed in the picture below. 

{:refdef: style="text-align: center;"}
![Feature Detaching](/assets/detaching.png) 
{: refdef}

However, there is one case we must handle. Suppose a slightly different splitting configuration:

{:refdef: style="text-align: center;"}
![Shared Edge](/assets/shared_edge.png) 
{: refdef}

As you can see, the edge in red of a triangle that is below the plane is connected with a triangle that is above the plane. In this case, we cannot simply make 
the new edge vertex of the triangle below the plane point to the new vertex. Instead, we must create a new edge with the new vertex and do not touch the edge in red.

{:refdef: style="text-align: center;"}
![Edge Creation](/assets/shared_edge_correct.png) 
{: refdef}

As you can see in the illustration above, the red vertex is maintained and a new edge (in green) is created. 

# Implementation

Now that we have explained how to tear a cloth we can write up a simple implementation for the algorithm.

In our implementation we assume the cloth has a list of particles, spring forces acting on a pair of particles, triangles, and functions for creating those entities from definitions.
We also assume we can find a spring given a pair of particles. With those functions we can write our main cloth tearing function.

{% highlight cpp %}

bool Tear()
{
	for (b3SpringForce* s = m_cloth.GetSpringForceList(); s; s = s->GetNext())
	{
		// Get the tension force.
		b3Vec3 tension = s->GetActionForce();

		// This is the maximum tension force in N.
		const scalar kMaxTension = 1000.0f;

		// Has the tension force reached the limit?
		if (b3LengthSquared(tension) <= kMaxTension * kMaxTension)
		{
			continue;
		}

		b3ClothParticle* p1 = s->GetParticle1();
		b3ClothParticle* p2 = s->GetParticle2();

		b3Vec3 x1 = p1->GetPosition();
		b3Vec3 x2 = p2->GetPosition();

		if (p1->GetType() == e_dynamicClothParticle)
		{
			b3Vec3 n = b3Normalize(x2 - x1);
			b3Plane plane(n, x1);

			bool wasSplit = SplitParticle(p1, plane);
			if (wasSplit)
			{
				return true;
			}
		}

		if (p2->GetType() == e_dynamicClothParticle)
		{
			b3Vec3 n = b3Normalize(x1 - x2);
			b3Plane plane(n, x2);

			bool wasSplit = SplitParticle(p2, plane);
			if (wasSplit)
			{
				return true;
			}
		}
	}

	// There are no more particles to split.
	return false;
}

{% endhighlight %}

So that the above function Tear() can be called like so:

{% highlight cpp %}

while (Tear() == true);

{% endhighlight %}

That is, it tries to split a particle until there is no particle to be split.

Let us explain the main portions of code above. It loops over each spring in the cloth and then it checks if the tension force has reached a tolerance.
If this is true then it tries to separate one of the particles by the tension plane. If a particle is not a dynamic particle then it skips that particle. (Decently 
designed cloth simulators have the concept of static, kinematic, and dynamic particles. Splitting kinematic and static particles can create visually 
unpleasing results.) Then it repeats this process until there is no particle to be split.

Now the core of the tearing process lies at the SplitParticle method, which it will be explained in a moment. Before doing so, we must write our triangle 
partition function.

{% highlight cpp %}

void Partition(b3ClothParticle* pSplit, 
	const b3Plane& plane,
	b3Array<b3ClothTriangleShape*>& trianglesAbove,
	b3Array<b3ClothTriangleShape*>& trianglesBelow)
{
	for (b3ClothTriangleShape* t = m_cloth.GetTriangleShapeList(); t; t = t->GetNext())
	{
		b3ClothParticle* p1 = t->GetParticle1();
		b3ClothParticle* p2 = t->GetParticle2();
		b3ClothParticle* p3 = t->GetParticle3();

		// Does the triangle contain the splitting particle pSplit?
		if (p1 != pSplit && p2 != pSplit && p3 != pSplit)
		{
			continue;
		}

		b3Vec3 x1 = p1->GetPosition();
		b3Vec3 x2 = p2->GetPosition();
		b3Vec3 x3 = p3->GetPosition();

		b3Vec3 center = (x1 + x2 + x3) / 3.0f;

		scalar distance = b3Distance(center, plane);
		
		// Is the triangle above or below the plane?
		if (distance > 0.0f)
		{
			trianglesAbove.PushBack(t);
		}
		else
		{
			trianglesBelow.PushBack(t);
		}
	}
}

{% endhighlight %}

The function above is very simple. It loops over the list of triangles in the cloth that contains the splitting particle and classifies the triangle 
center against the splitting plane. 

{% highlight cpp %}

bool HasSpring(const b3Array<b3ClothTriangleShape*>& triangles,
		b3ClothParticle* pSplit, b3ClothParticle* pOther)
{
	for (u32 i = 0; i < triangles.Count(); ++i)
	{
		b3ClothTriangleShape* triangle = triangles[i];

		b3ClothParticle* tp1 = triangle->GetParticle1();
		b3ClothParticle* tp2 = triangle->GetParticle2();
		b3ClothParticle* tp3 = triangle->GetParticle3();

		// 1, 2
		if (tp1 == pSplit && tp2 == pOther)
		{
			return true;
		}

		// 2, 1
		if (tp2 == pSplit && tp1 == pOther)
		{
			return true;
		}

		// 2, 3
		if (tp2 == pSplit && tp3 == pOther)
		{
			return true;
		}

		// 3, 2
		if (tp3 == pSplit && tp2 == pOther)
		{
			return true;
		}

		// 3, 1
		if (tp3 == pSplit && tp1 == pOther)
		{
			return true;
		}

		// 1, 3
		if (tp1 == pSplit && tp3 == pOther)
		{
			return true;
		}
	}

	return false;
}

{% endhighlight %}

The last function checks if the given list of triangles has a triangle which is connected by a spring (edge). 

With this function we can finally write the SplitParticle routine. I won't explain this function as the code comments tell what it does.

{% highlight cpp %}

bool SplitParticle(b3ClothParticle* pSplit, const b3Plane& plane)
{
	// Separate the triangles in two sets.
	b3StackArray<b3ClothTriangleShape*, 256> trianglesAbove;
	b3StackArray<b3ClothTriangleShape*, 256> trianglesBelow;
	Partition(pSplit, plane, trianglesAbove, trianglesBelow);

	// There must be at least one triangle on each side of the plane.
	if (trianglesAbove.Count() == 0 || trianglesBelow.Count() == 0)
	{
		// The particle can't be split.
		return false;
	}

	// Define the new particle.
	b3ClothParticleDef pdNew;
	pdNew.type = pSplit->GetType();
	
	// Shift the particle along the plane normal.
	pdNew.position = pSplit->GetPosition() - 0.2f * plane.normal;

	// Create the new particle.
	b3ClothParticle* pNew = m_cloth.CreateParticle(pdNew);
	
	// Process each triangle that is below the plane. This is the ataching process covered in this tutorial.
	for (u32 i = 0; i < trianglesBelow.Count(); ++i)
	{
		b3ClothTriangleShape* triangle = trianglesBelow[i];

		b3ClothParticle* p1 = triangle->GetParticle1();
		b3ClothParticle* p2 = triangle->GetParticle2();
		b3ClothParticle* p3 = triangle->GetParticle3();

		// Destroy the triangle.
		m_cloth.DestroyTriangleShape(triangle);
		
		// Check what particle on the triangle is the splitting particle.
		if (p1 == pSplit)
		{
			// Define the new triangle.
			b3ClothTriangleShapeDef tdNew;
			tdNew.p1 = pNew;
			tdNew.p2 = p2;
			tdNew.p3 = p3;
			tdNew.v1 = pNew->GetPosition();
			tdNew.v2 = p2->GetPosition();
			tdNew.v3 = p3->GetPosition();

			// Create the new triangle.
			m_cloth.CreateTriangleShape(tdNew);

			// Note that the springs that are shared among the triangles that are below the plane can 
			// be destroyed in the splitting process, so the pointer can be null.
			b3SpringForce* sf1 = m_cloth.FindSpring(p1, p2);
			if (sf1)
			{
				// Define the new spring for the particles pNew and p2 using the parameters of the current spring.
				b3SpringForceDef sNew;
				sNew.Initialize(pNew, p2, sf1->GetStructuralStiffness(), sf1->GetDampingStiffness());
				
				// Create the new spring using the spring definition sNew.
				m_cloth.CreateForce(sNew);
				
				// If the triangles above the plane contain the current spring then the spring must not be destroyed.
				if (HasSpring(trianglesAbove, p1, p2) == false)
				{
					m_cloth.DestroyForce(sf1);
				}
			}
			
			// Do the same for the edge p3-p1...
			b3SpringForce* sf2 = m_cloth.FindSpring(p3, p1);
			if (sf2)
			{
				b3SpringForceDef sNew;
				sNew.Initialize(p3, pNew, sf2->GetStructuralStiffness(), sf2->GetDampingStiffness());
				
				m_cloth.CreateForce(sNew);
				
				if (HasSpring(trianglesAbove, p3, p1) == false)
				{
					m_cloth.DestroyForce(sf2);
				}
			}
		}
		
		// The same applies to the other particles.
		if (p2 == pSplit)
		{
			b3ClothTriangleShapeDef tdNew;
			tdNew.p1 = p1;
			tdNew.p2 = pNew;
			tdNew.p3 = p3;
			tdNew.v1 = p1->GetPosition();
			tdNew.v2 = pNew->GetPosition();
			tdNew.v3 = p3->GetPosition();

			m_cloth.CreateTriangleShape(tdNew);

			b3SpringForce* sf1 = m_cloth.FindSpring(p1, p2);
			if (sf1)
			{
				b3SpringForceDef sNew;
				sNew.Initialize(p1, pNew, sf1->GetStructuralStiffness(), sf1->GetDampingStiffness());
				
				m_cloth.CreateForce(sNew);
				
				if (HasSpring(trianglesAbove, p1, p2) == false)
				{
					m_cloth.DestroyForce(sf1);
				}
			}

			b3SpringForce* sf2 = m_cloth.FindSpring(p2, p3);
			if (sf2)
			{
				b3SpringForceDef sNew;
				sNew.Initialize(pNew, p3, sf2->GetStructuralStiffness(), sf2->GetDampingStiffness());
				m_cloth.CreateForce(sNew);
				
				if (HasSpring(trianglesAbove, p2, p3) == false)
				{
					m_cloth.DestroyForce(sf2);
				}
			}
		}
		
		if (p3 == pSplit)
		{
			b3ClothTriangleShapeDef tdNew;
			tdNew.p1 = p1;
			tdNew.p2 = p2;
			tdNew.p3 = pNew;
			tdNew.v1 = p1->GetPosition();
			tdNew.v2 = p2->GetPosition();
			tdNew.v3 = pNew->GetPosition();

			m_cloth.CreateTriangleShape(tdNew);
			
			b3SpringForce* sf1 = m_cloth.FindSpring(p2, p3);
			if (sf1)
			{
				b3SpringForceDef sNew;
				sNew.Initialize(p2, pNew, sf1->GetStructuralStiffness(), sf1->GetDampingStiffness());
				
				m_cloth.CreateForce(sNew);
				
				if (HasSpring(trianglesAbove, p2, p3) == false)
				{
					m_cloth.DestroyForce(sf1);
				}
			}

			b3SpringForce* sf2 = m_cloth.FindSpring(p3, p1);
			if (sf2)
			{
				b3SpringForceDef sNew;
				sNew.Initialize(pNew, p1, sf2->GetStructuralStiffness(), sf2->GetDampingStiffness());
				m_cloth.CreateForce(sNew);
				
				if (HasSpring(trianglesAbove, p3, p1) == false)
				{
					m_cloth.DestroyForce(sf2);
				}
			}
		}
	}

	// The particle has been split.
	return true;
}

{% endhighlight %}

# Results

Here's a video showing the visually pleasing results of the technique described in this tutorial in practice:

<iframe width="560" height="315" src="https://www.youtube.com/embed/bjf5vbp2meM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

I hope you have enjoyed this small tutorial!

While I can't find a simple comment system to add to this blog you're welcome to report any errors using the issue tracker in this repository:

[https://github.com/irlanrobson/irlanrobson.github.io](https://github.com/irlanrobson/irlanrobson.github.io)