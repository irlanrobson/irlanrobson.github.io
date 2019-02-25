---
layout: post
title: Building a Half-Edge Mesh from a Triangle List
tags:
- Computational Geometry
- Math
---
The problem is to build a polygonal mesh from a triangle list efficiently. A solution is important specially when building or testing physics engines. The method below shows a possible solution. This can be done without needing the additional storage and processing cost of keeping a list of half-edges for a given face during its addition which would be linked at the end of the addition in that case.

{% highlight cpp %}

#define NULL_FEATURE 255

void Hull::AddFaces(u8 V, const Vec3* vs, u8 F, const Face* fs)
{
	// Hull statistics
	// E ~ 3V - 6, F ~ 2V - 4 
	// Mesh statistics
	// E ~ 3V, F ~ 2V, HE = 2 * E 
	u32 HE = 2 * 3 * V;
	ASSERT(V < 256);
	ASSERT(HE < 256);
	ASSERT(F < 256);

	memcpy(vertices, vs, V * sizeof(Vec3));
	vertexCount = V;

	edgeCount = 0;

	for (u8 i = 0; i < F; ++i) 
	{
		faces[i].edge = NULL_FEATURE;
	}
	faceCount = F;

	struct Edge
	{
		u8 v1, v2;
		u8 e12, e21;
	};

	const u32 edgePairCapacity = 256;
	Edge edgePairs[edgePairCapacity];
	u32 edgePairCount = 0;

	// Loop over all faces.
	for (u8 i = 0; i < faceCount; ++i) 
	{
		// Add face
		const Face* face = fs + i;
		u8 faceVertexCount = face->vertexCount;
		const u8* indices = face->indices;
		
		// Loop over all edges if the current face.
		for (u8 j = 0; j < faceVertexCount; ++j) 
		{
			// Add edge
			u8 v1 = indices[j];
			u8 v2 = j + 1 < faceVertexCount ? indices[j + 1] : indices[0];

			// Check if two half edges for this edge were created before.
			i32 edgeIndex = -1;
			bool flip = false;
			for (u32 k = 0; k < edgePairCount; ++k)
			{
				Edge* e = edgePairs + k;
				if (e->v1 == v1 && e->v2 == v2)
				{
					edgeIndex = k;
					break;
				}
				if (e->v1 == v2 && e->v2 == v1)
				{
					edgeIndex = k;
					flip = true;
					break;
				}
			}

			if (edgeIndex >= 0)
			{
				ASSERT(flip);
				u8 e12 = edgePairs[edgeIndex].e21;
				u8 e21 = edgePairs[edgeIndex].e12;

				HalfEdge* edge12 = edges + e12;
				HalfEdge* edge21 = edges + edge12->twin;
				
				ASSERT(e12 == edge21->twin);
				ASSERT(e21 == edge12->twin);

				// The mesh is probably a non-manifold one.
				ASSERT(edge12->face == NULL_FEATURE);

				// Add the half-edge to this face circular list of half-edges
				edge12->face = i;
				
				if (faces[i].edge == NULL_FEATURE)
				{
					faces[i].edge = e12;
					edges[e12].prev = e12;
					edges[e12].next = e12;
				}
				else
				{
					u8 tail = faces[i].edge;
					u8 head = edges[tail].next;

					edges[head].prev = e12;
					edges[tail].next = e12;

					edges[e12].prev = tail;
					edges[e12].next = head;

					faces[i].edge = e12;
				}
			}
			else 
			{
				// Add two half-edges
				u8 e12 = edgeCount++;
				u8 e21 = edgeCount++;

				edges[e12].origin = v1;
				edges[e12].twin = e21;
				edges[e12].face = i;
				edges[e12].prev = NULL_FEATURE;
				edges[e12].next = NULL_FEATURE;

				edges[e21].origin = v2;
				edges[e21].twin = e12;
				edges[e21].face = NULL_FEATURE;
				edges[e21].prev = NULL_FEATURE;
				edges[e21].next = NULL_FEATURE;

				// Add half-edge to this face circular list of half-edges
				if (faces[i].edge == NULL_FEATURE)
				{
					faces[i].edge = e12;
					edges[e12].prev = e12;
					edges[e12].next = e12;
				}
				else
				{
					u8 tail = faces[i].edge;
					u8 head = edges[tail].next;

					edges[head].prev = e12;
					edges[tail].next = e12;
					
					edges[e12].prev = tail;
					edges[e12].next = head;
					
					faces[i].edge = e12;
				}
				
				// Add half-edges to map
				ASSERT(edgePairCount < edgePairCapacity);
				edgePairs[edgePairCount++] = { v1, v2, e12, e21 };
			}
		}
	}

	Validate();
}

{% endhighlight %}

The method above works because the final face edge index is used as a "tail pointer" for the face's circular list of half-edges. It is not necessary to have the face at this index pointing to a particular edge unless required. It can point to any half-edge as long as the edge envolves the face.