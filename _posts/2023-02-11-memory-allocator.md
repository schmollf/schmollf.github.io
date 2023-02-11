---
layout: post
title: Understanding malloc() - Dive into Memory Allocators
---

# Understanding `malloc()` - Dive into Memory Allocators

Memory allocators are a buzz word for everybody interested in systems programming. Today, we'll dig into `malloc()` and write our own version of it in C. The principles in other languages will be similar. Even in garbage collected languages, such as Java and Golang, much of the concepts will apply, with the important addition of compaction in Java for more effective space reclamation. But that's a different topic.

On most systems your program will by default link against glibc's `malloc()` implementation and it will use `mmap()` to allocate larger chunks of memory which are then handed out piecewise on every `malloc()` call. That's fine for most applications, but since memory allocation is such a fundamental aspect of a program commercial applications tend to override this allocator with their own implementation.

// maybe write why they override their own implementation, e.g. that syscalls are expensive?

Even custom implementations usuallyl follow the same principles. Let's first define a rough interface we are trying to achieve:

```c
void * malloc(size_t bytes);
void free(void *);
```

There are three classical approaches to implement this:
- slab allocators
- pool allocators
- for very large allcoations, we can pass through to mmap() directly

## Slab Allocators

With all approaches we end up going to the kernel to request a big chunk of memory (our "slab"). We don't want to request too much because we might not need that much, but we want large allocations to amortize the syscall overhead. We then can hand out a small piece of memory of arbitrary size on every allocation.

We will somewhere need to store the size of the memory handed out. Either we allocate a small section before every chunk (leading to internal fragmentation) or we only allow specific sizes we have in our pool allocators and round up any smaller allocation to the next biggest aligned one. This is what TCMalloc and the Golang allocator do. They then allow looking up the size of an element via a lookup in a radix tree. // specific example as explanation?

### Free list and coalescing

Handing out the memory was easy but if we will not free all memory at the same time we end up with (external/internal?) fragmentation. We store free'd elements in a free list (a sorted linked list) and coalesce adjacent free elements to get bigger chunks.

Should the whole buffer become free we can release it back to the kernel.

### when to use slab allocators vs pool allocators

## Pool Allocators

We saw that it is messy to serve allocations of different sizes from the same slab. It is much easier to use a single slab if all allocations are of the same size.

Again, we will need a way to determine the size of the allocation on free. TCMalloc uses a radix_tree, but it is also possible to encode it in the interface of malloc if we change it a little bit.

```c
custom_memory_pool create_memory_pool(sizeof(element), elements_in_pool);

void * malloc(custom_memory_pool);
void free(custom_memory_pool, void *);
```

As long as we keep the memory pool object around we only need to store the information about size in a single place.

We can use a free list to implement this by using the unused memory to store pointers to the next chunk or we can use a bit vector.

```c
// This pool allocator is not taking care of alignment.

struct PoolAllocator {
        vector<bool> freeList;
        void * memoryChunk;
        int const numElements;
        int const elemSize;

        PoolAllocator(int numElements_, int elemSize_) : numElements(numElements_), elemSize(elemSize_), freeList(numElements, false) {
                memoryChunk = malloc(numElements_ * elemSize_);
                cout << reinterpret_cast<uint64_t>(memoryChunk) << endl;
        }

        ~PoolAllocator() {
                free(memoryChunk);
        }

        void * map() {
                for(int i = 0; i < numElements; ++i) {
                        if(!freeList[i]) {
                                freeList[i] = true;
                                return (void*)(((char*)memoryChunk) + i*elemSize);
                        }
                }

                return nullptr;
        }

        void unmap(void * addr) {
                int idx = ((char*)addr - (char*)memoryChunk)/elemSize;
                freeList[idx] = false;
        }
};

```

## Multithreading

The examples we wrote work fine on a single thread but will require synchronization once we end up going parallel. Beefy machines can have 100s of CPUs, making this a real bottleneck. Google and Facebook both wrote and open-sourced their own memory allcators to improve scaling across many cores.

That's why memory allocators in practice use the concept of **arenas**, sometimes also referred to as **heaps**. Every arena will be specific to a logical core (i.e. a hyperthread on x86), meaning no locking is required as long as we can serve from the arena. We will only require locking to transfer buffers between cores or to get a new buffer from the kernel.

// here could be a nice picture of transferring a buffer between CPUs/arena's

## Re: My program is spending too much time in `malloc()`

## References

- there's a chapter on implementing a memory allocator in the C book by Kerninghan
- garbage collection book
- there's TCMalloc by Google (announcement https://abseil.io/blog/20200212-tcmalloc), and it has a design section (https://google.github.io/tcmalloc/design)
- the golang allocator is a good read https://go.dev/src/runtime/malloc.go
- jemalloc, mostly developed at facebook https://engineering.fb.com/2011/01/03/core-data/scalable-memory-allocation-using-jemalloc/
