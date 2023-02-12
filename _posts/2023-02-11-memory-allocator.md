---
layout: post
title: Understanding `malloc()` - Dive into Memory Allocators
---

Memory allocators are a buzz word for everybody interested in systems programming. Today, we'll dig into `malloc()` and write our own version of it in C. The principles in other languages will be similar. Even in garbage collected languages, such as Java and Golang, much of the concepts will apply, with the important addition of compaction in Java for more effective space reclamation. But that's a different topic.

On most systems your program will by default link against glibc's `malloc()` implementation and it will use `mmap()` or `sbrk()` to allocate larger chunks of memory which are then handed out piecewise on every `malloc()` call. That's fine for most applications, but since memory allocation is such a fundamental aspect of a program commercial applications tend to override this allocator with their own implementation.

Even custom implementations usually follow the same principles and have to solve the same problems using various tradeoffs. Let's first define a rough interface we are trying to achieve:

```c
void * malloc(size_t bytes);
void free(void *);
```

There are three classical approaches to implement this:
- slab allocators
- pool allocators
- for very large allocations we can pass through to mmap() directly

## Slab Allocators

With all approaches we end up going to the kernel to request a big chunk of memory (our "slab"). We don't want to request too much because we might not need that much memory, but we want large allocations to amortize the syscall overhead. We then hand out a small piece of memory of arbitrary size on every allocation.

We will somewhere need to store the size of the memory handed out. Either we allocate a small section before every chunk or we only allow specific sizes we have in our pool allocators and round up any smaller allocation to the next biggest aligned one. Both approaches imply *internal fragmentation*, meaning we cannot use all the memory with 100% efficiency. Both TCMalloc and the Golang allocator go for the latter solution of rounding up. They then allow looking up the size of an element via a radix tree.

### Free list and coalescing

Handing out the memory was easy but if we do not free all memory at the same time we end up with external fragmentation, meaning we might be able to fulfill large allocations because we have many small free chunks which are not adjacent. Free'd elements are typically stored in a free list (a sorted linked list) and adjacent free elements can be coalesced to get bigger chunks.

Should the whole buffer become free we can release it back to the kernel.

## Pool Allocators

We saw that it is messy to serve allocations of different sizes from the same slab. It is much easier to use a single slab if all allocations are of the same size.

Again, we will need a way to determine the size of the allocation on free. TCMalloc uses a *radix tree*, but it is also possible to encode it in the interface of malloc if we change it a little bit:

```c
struct element;
struct memory_pool;

memory_pool*
create_memory_pool(size_t size_of_element,
                   size_t num_elements_in_pool);

void * malloc(memory_pool*);
void free(memory_pool*, void *);
```

As long as we keep the memory pool object around we only need to store the information about size in a single place.

We can use a free list to implement to store pointers to the next chunk or we can use a bit vector. This is less efficient than using a radix tree but hey, this is a teaching exercise, so don't use this in production.

```c
// note: this pool allocator is not taking care of alignment
//       and does not zero memory

struct PoolAllocator
{

size_t const numElements = 0;
size_t const elemSize = 0;
vector<bool> freeList; /* true -> element is used */

void * memoryChunk = nullptr;

PoolAllocator(size_t numElements_, size_t elemSize_) :
    numElements(numElements_),
    elemSize(elemSize_),
    freeList(numElements, false)
{
    memoryChunk = malloc(numElements_ * elemSize_);
}

~PoolAllocator()
{
    free(memoryChunk);
}

void * map()
{
    for(size_t i = 0; i < numElements; ++i) {
        if (!freeList[i]) {
            freeList[i] = true;
            return (void*)((char*)memoryChunk + i*elemSize);
        }
    }

    return nullptr;
}

void unmap(void * addr)
{
    size_t idx = ((char*)addr - (char*)memoryChunk)/elemSize;
    freeList[idx] = false;
}
};

```

## Multithreading

The implementations we discussed work fine on a single thread but will require synchronization once we end up going parallel. Beefy machines can have 100s of CPUs, making this a real bottleneck if synchronization means using a global mutex. Google and Facebook both wrote and open-sourced their own memory allacators mainly to improve scaling across many cores.

In practice, memory allocators use the concept of **arenas**, sometimes also referred to as **heaps**, to solve this problem. Every arena will be specific to a logical core (i.e. a hyperthread on x86), meaning no locking is required as long as we can serve from the arena. We will only require locking to transfer buffers between cores or to get a new buffer from the kernel.

## Re: My program is spending too much time in `malloc()`

I mentioned that Google came up with TCMalloc to improve scaling, but it turns out that just measuring the time spent on allocation and deallocation is not necessarily the best metric to optimize for. If you're curious, you can read more about it on [Google's blog](https://cloud.google.com/blog/topics/systems/trading-off-malloc-costs-and-fleet-efficiency?hl=en).

## References

- there's a section on implementing a memory allocator in the C book by Kerninghan and Ritchie
- Google's TCMalloc has a pretty detailed [design section](https://google.github.io/tcmalloc/design)
- The golang allocator is a [good read](https://go.dev/src/runtime/malloc.go)
- [jemalloc](https://jemalloc.net/), mostly developed at [Facebook](https://engineering.fb.com/2011/01/03/core-data/scalable-memory-allocation-using-jemalloc/)
- if you're interested in automatic memory management I recommend the ["bible of garbage collection"](https://gchandbook.org/)
