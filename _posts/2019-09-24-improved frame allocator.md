---
layout: post
title: Improved Frame Allocator
tags: []
---

Among all types of allocators, the frame allocator is the fastest, highly debuggable and easiest to implement. 

This allocator stores an array of bytes and maintains a pointer to the current memory address you can allocate in the next call to allocate. 
The pointer can never go backwards. It only moves forwards. This is why it is so fast.

However, in typical implementation, the allocation function never fallbacks to a parent allocator such as malloc if it runs out of memory. 
This can happen often because it is not possible to free memory with this allocator. This is one practical rule of thumb of allocators. 
If you can't allocate memory using one allocator then fallback to another allocator!

Therefore, a simple solution is to not allocate just memory but also allocate more information, or as we call here, a memory block. 
The memory block should contain a flag indicating the memory came out from a parent allocator or from the frame allocator. 
Here's an example of how one could declare this kind of improved frame allocator. 

(I hope the types are self-documented, and please ignore the charming namespace prefixes, I prefer them over namespaces for clarity)

{% highlight cpp %}

// Allocate 1 MiB from the stack. 
// Increase as you want.
const u32 b3_maxFrameSize = B3_MiB(1);

// A small frame allocator.
class b3FrameAllocator
{
public:
	b3FrameAllocator();
	~b3FrameAllocator();

	// Allocate a block of memory.
	// Allocate using malloc if the memory is full.
	void* Allocate(u32 size);
	
	// Free the memory if it was allocated by malloc.
	void Free(void* p, u32 size);

	// Resets this allocator. This function must be called at the beginning of a frame.
	void Reset();
private:
	struct Block
	{
		void* p;
		u32 size;
		bool parent;
	};

	u8 m_memory[b3_maxFrameSize];
	u8* m_p;
	u32 m_allocatedSize;
};

{% endhighlight %}

Notice the user must supply the allocated memory size in bytes to the allocation and free functions. 
This is because in our implementation we ensure this block structure is followed by the allocated memory.
So we need the offset. This allows fast checking.
Of course here you could use some kind of pointer math or whatever, but there seems to be platform dependency 
here when playing with pointers so I even tried that. The extension I purpose below is much clear and reliable than 
the other versions.

Here's the implementation: 

{% highlight cpp %}

b3FrameAllocator::b3FrameAllocator()
{
	m_p = m_memory;
	m_allocatedSize = 0;
}

b3FrameAllocator::~b3FrameAllocator()
{
}

void* b3FrameAllocator::Allocate(u32 size)
{
	u32 totalSize = size + sizeof(Block);

	if (m_allocatedSize + totalSize > b3_maxFrameSize)
	{
		u8* p = (u8*)malloc(totalSize);
		
		Block* block = (Block*)(p + size);
		block->p = p;
		block->size = size;
		block->parent = true;
		return p;
	}

	u8* p = m_p;
	m_p += totalSize;
	m_allocatedSize += totalSize;
	
	Block* block = (Block*)(p + size);
	block->p = p;
	block->size = size;
	block->parent = false;
	return p;
}

void b3FrameAllocator::Free(void* p, u32 size)
{
	Block* block = (b3Block*)((u8*)p + size);
	B3_ASSERT(block->p == p);
	B3_ASSERT(block->size == size);
	if (block->parent)
	{
		free(p);
	}
}

void b3FrameAllocator::Reset()
{
	m_p = m_memory;
	m_allocatedSize = 0;
}

{% endhighlight %}

This way an allocation practically never fails. Of course here the parent allocator could be replaced by another type of allocator so we have a composition of 
allocators. There's room for showing up creativity here. There are combinations of allocators that can work really fast! 