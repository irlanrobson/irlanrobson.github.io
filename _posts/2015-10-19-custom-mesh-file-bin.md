---
layout: post
title: Custom Mesh File (.bin)
tags:
- Game Programming
---

Binary files are the game industry standard format for streaming data from disk to memory at engine run-time because they can be easily interpreted.

I've adopted a custom binary mesh file format in my in-house game engine recently and also written an .fbx to .bin converter. For those who don't know, binary files contain contiguous bytes which will be preferably loaded linearly at some point by the final game. The image below shows that is possible to visualize a blob in an hexadecimal editor and actually understand a little better how they are used.

{:refdef: style="text-align: center;"}
![Mesh file format on a text editor](/assets/mesh_file.png) 
{: refdef}

In the image above, there is a 4-meshes-indicator initially, followed by the mesh string "Body_Tri" total bytes which is 8, followed by the string itself "Body_Tri", followed by the total of vertex positions, followed by a float array, and so forth.

Generally, binary files have been choosen by game programmers since many years ago to be interpreted by the final game executable because they've a couple of advantages, some of which listed below:

* There's no need of neither parsing text formats or validating them because that is the engine data compiler's responsability (the tools side of an engine). This is important and makes the engine code a lot of simpler which is good for whatever engine.
	
* It's pretty easy to paste more data into it from a tool.
	
* It is possible to write a binary file so that we can cast it into a C++ class/struct once! (See [here](http://bitsquid.blogspot.com.br/2010/02/blob-and-i.html) for more details.)
	
* It lead us to think more of POD data and data-flow instead of objects.

If you're interested in writing your own file format, there is an example below showing a pseudo-code for reading a standard **v-n-i** mesh structure from a stream of bytes loaded from a bin-file. Note that the only responsability of the byte stream class is to receive a blob and reinterpret these bytes as integral types when requested; it doesn't manages memory in any kind (altough is pseudo).

{% highlight cpp %}

struct SubMesh
{
  Array<Vec3> positions;
  Array<Vec3> normals;
  Array<u16> indices;
}; 

void Cast(ByteStream& stream, SubMesh& sm)
{
  sm.positions.Alloc(stream.ReadUInt32());
  stream.ReadFloats(sm.positions.Base(), 3 * sizeof(float) * sm.positions.Count());

  sm.normals.Alloc(stream.ReadUInt32());
  stream.ReadFloats(sm.normals.Base(), 3 * sizeof(float) * sm.normals.Count());

  sm.indices.Alloc(stream.ReadUInt16());
  stream.ReadUInts16(sm.indices.Base(), sizeof(u16) * sm.indices.Count());
} 

struct ByteStream
{
  ByteStream(u8* begin, u32 size) : m_begin(begin), m_size(size), m_marker(begin) { }

  u8* m_begin;
  u8* m_marker;
  u32 m_size;

  float ReadFloat()
  {
     float out = *(float*) m_marker;
     m_marker += sizeof(float);
     return out;
  }

  u16 ReadUInt16()
  {
     u16 out = *(u16*)m_marker;
     m_marker += sizeof(u16);
     return out;
  }
  ...
};

{% endhighlight %}

That's all. I hope this post helps.