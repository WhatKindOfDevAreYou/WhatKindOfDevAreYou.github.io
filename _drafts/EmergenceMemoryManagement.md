---
layout: post
title: 'Emergence: Memory management'
categories: [Emergence, Development Log]
tags: [Emergence, C++]
---

It is very important to have consistent memory model for your project from the start, because it is almost impossible
to change it later. In the post I will talk about [Emergence](https://github.com/KonstantinTomashevich/Emergence)
memory management library and its design decisions.

### Motivation

Having the right approach to managing memory is crucial for performance: it impacts both allocation cost and
cache coherency. Of course, you can just allocate all the memory on heap by using new, but it will lead to slow
allocation, cache misses and memory fragmentation. In my opinion, game engine without consistent memory management
model is like giant with legs made of straw. Therefore, I decided that 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) must be built on top of consistent memory library
from the very beginning. Furthermore, I've decided to write my own memory management library to have full control
on how memory allocation works.

Writing your own memory management library is not a walk in the park and it's quite easy to get lost in details.
For example, if you're committed enough, you can try to ignore the existence of C standard library allocation
functions and write your own platform-specific versions. But it is a tedious and troublesome task. I've decided
to be less thorough about that and primarily focus on the API instead of implementation details: once API is done
right and whole project uses it, it shouldn't be difficult to change implementation details under the hood.

One important part about memory management model is that it should not only be consistent and easy to use,
but also provide tools for memory usage analysis, for example custom profiler. Having such tools is important
because they allow us to visualize how we're using memory and provide us with insight on what can be improved.
When I was searching for such tool to integrate into my memory management library, I've mostly encountered tools
that track memory allocations by file name or by one of the static predefined tags. It looked like it is not
good enough for [Emergence](https://github.com/KonstantinTomashevich/Emergence): I wanted to be able to build
dynamic memory usage flame graph and none of the tools I found was fully capable of doing it. So I've decided
that I need to write my own tool.

To summarize, I've decided to follow these principles:

- The most important part is consistent and easy-to-use API.
- C standard library functions is a good enough backend for the beginning.
- Library must provide API and a tool for memory usage profiling and analysis.

### Features

It is important to have a rich enough feature set. It should be neither too small nor too big: it should provide
enough control on how memory is allocated, but user must not sink in the sea of details. I've decided to go with
this feature set for [Emergence](https://github.com/KonstantinTomashevich/Emergence) memory management library:

- Unordered pool allocator for lighting-fast allocation and deallocation of objects of the same type.
- Ordered pool allocator, that has slow deallocation operation, but is more cache coherent and provides iteration
  over allocated chunks.
- Stack allocator for lighting-fast cache-coherent allocation of temporary trivial objects.
- Heap allocator for general purpose allocation.
- String interning implementation for ASCII strings.

It's better to provide memory usage profiler features as separate list to avoid blending with memory management
features:

- Memory usage is tracked for logical groups that can be created at any moment during runtime.
- Memory usage snapshot for all logical groups can be taken at any moment of time. 
  It makes sampling-based profiling possible.
- Profiler should be able to represent all memory operations as continuous set of events.
  It makes instrumentation-based profiling possible.
- There should be an API for serializing and deserializing memory usage tracks: initial memory snapshot and 
  continuous set of events that happened after this snapshot, 
- There should be a tool for viewing memory tracks.

Initially I also wanted to implement client-server profiler like [Tracy](https://github.com/wolfpld/tracy), but I've
decided to postpone it. It is not so difficult, but it requires time and I've decided that it is better to spend this 
time on other [Emergence](https://github.com/KonstantinTomashevich/Emergence) libraries.

### Allocators

I will start from describing implemented allocators and their implementation details.

#### Pool

Pool allocators manage memory in terms of chunks and pages. Chunk is a memory block of predefined size which is
usually selected during allocator construction. Page is a large continuous block of chunks. Pages are organized into
linked list: each page contains pointer to the next one. Chunks can either be used or unused: used chunks contain
user-defined data and free chunks are organized into free-chunk linked list -- every free chunk contains pointer
to the next one. To sum up, referencing looks like this:

![Pool allocator pages and pointers](/assets/img/EmergenceMemoryManagement/PoolAllocator.png)

When user requests new chunk from the allocator, we start by checking whether free chunk list is empty. If it's not
empty, we can just pop the first chunk out of this list and return it to user. If there is no free chunks left, we
need to allocate new page and put all the chunks from this page into free chunk list. 

When user reports that he no longer uses given chunk, we can just put this chunk back into free chunks list and that's
all! But there is one important question: are putting this chunk into the beginning of free chunk list or into other 
place? This question defines main algorithmic difference between ordered and unordered pools. Basically, ordered pool 
guarantees that page list and free chunk list are sorted by addresses in ascending order. You can see that image above
illustrates ordered pool. This ordering provides several benefits:

- Memory allocation is more cache coherent as free chunk list ordering minimizes amount of holes between used chunks.
- Pool shrinking (deallocation of fully unused pages) can be implemented effectively: it can be done in 
  O(pageCount + freeChunkCount) instead of O(pageCount * pageCapacity * freeChunkCount), because we do not need
  to iterate over all free chunk list to determinate whether chunk is free.
- Iteration over used chunks can be implemented effectively for the same reason.

However, ordering is not a silver bullet, because it has one important flaw: it makes release operation 
(when user informs that chunk is no longer used) O(freeChunksCount) -- we need to find suitable place for new chunk
in the list. This can introduce significant performance drops if we're releasing lots of chunks in a random order.
Therefore, usage of ordered or unordered pool is always a trade-off and memory usage strategy should be considered
thoroughly before selecting one pool type over another.

[Emergence::Memory service](https://github.com/KonstantinTomashevich/Emergence/tree/e8c37b6/Service/Memory) provides
both type of pools as separate classes: `OrderedPool` and `UnorderedPool`. This allows user to select pool algorithm
with zero runtime overhead. They both provide this set of operations:

- `Acquire` requests new chunk from the pool.
- `Release` returns chunk that is no longer used to the pool.
- `IsEmpty` checks whether pool has any used chunk.
- `Clear` deallocates all pages altogether.

In addition, `OrderedPool` supports these operations:

- Iteration over acquired (used) chunks through `BeginAcquired`/`EndAcquired`.
- `Shrink` deallocates all pages that have no used chunks.

It is worth mentioning that both pools support custom chunk alignment that can be specified during allocator creation.

In the end, let's go over the pros and cons of pool allocator usage. 

Pros:

- Generally faster allocation and deallocation: operations on free chunk list are much more lightweight that
  operations on complex heap allocator structures.
- Protection from memory fragmentation: on the top level allocator works with big memory blocks (pages) instead
  of small blocks and that reduces risk of memory fragmentation by a lot.
- Cache coherency: chunks are allocated on continuous blocks of memory (pages), therefore in most cases logically
  adjacent data will be stored in adjacent memory addresses.

Cons:

- You need to manually shrink/clear pool allocators if your memory usage strategy is not stable. For example,
  you may allocate lots of chunks for temporary objects during level loading and you will no longer need this data
  after level loading.
- Pool allocators only work for fixed size chunks with size greater or equal to the size of pointer. If you need
  variable-size cache coherent allocations or need to allocate smaller blocks of memory you need to use other approach.

#### Stack

Stack allocators usually operate on preallocated block of memory and provide memory in a stack-like fashion by
increasing stack top pointer. At the beginning stack top pointer looks at first byte of allocator-managed memory
region. Each allocation is done by moving stack top pointer forward. Deallocations are done by moving this pointer
backward. You can already see the problem in this pattern: deallocation can not be done in random order. That's true:
there is no canonical deallocation in stack allocator API, instead there is ReturnToCheckpoint operation, that moves
stack top pointer to saved position from the past. This operation deallocates everything that was allocated after
checkpoint.

![Stack allocator block](/assets/img/EmergenceMemoryManagement/StackAllocator.png)

[Emergence::Memory service](https://github.com/KonstantinTomashevich/Emergence/tree/e8c37b6/Service/Memory) provides
implementation of this type of allocator through `Stack` class. It preallocates memory block of given capacity and 
operates on top of this block. `Stack` supports following operations:

- `Acquire` -- allocates given amount of bytes with given alignment and moves stack top forward.
- `Head` -- returns stack top pointer that can be saved and used as checkpoint later.
- `Release` -- moves stack top pointer back to given checkpoint.
- `Clear` -- moves stack top pointer back to the beginning.
- `GetFreeSpace` -- returns how much space is left to be used.

Stack allocators are simple and powerful, but also usually a niche solution. Let's go over its pros and cons.

Pros:

- Lighting-fast allocation: nothing can be faster than one simple pointer operation.
- Lighting-fast mass-deallocation: we can deallocate all object in one pointer operation without needing to update
  anything except stack top pointer.
- Do not cause memory defragmentation: stack only preallocates memory during the construction and never actually
  allocates memory from heap after that.

Cons:

- Stack allocator is only suitable for trivially-destructible objects due to its mass-deallocation approach.
- Stack operates on fixed amount of memory and can never grow bigger that predefined capacity.
- Stack does not support selective deallocation: you either deallocate everything by reverting to checkpoint or
  deallocate nothing.

To sum up, stack allocator is one of the best examples of niche solutions. It is almost completely unusable for
general purpose situations, but is very helpful for some special cases. For example, it is a usual solution for
temporary trivial objects with one-frame lifetime: during frame all systems than need such objects request memory
for them though allocator and when frame ends these objects are easily deallocated all-at-once.

#### Heap

Heap allocators theme is quite interesting from technological point of view and it deserves a separate article for sure!
But in [Emergence](https://github.com/KonstantinTomashevich/Emergence) I've decided to avoid inventing my own bicycle,
which is quite rare as I'm almost always inventing my own bicycles, and use `malloc`, `realloc` and `free` under the
hood of `Heap` wrapper-class. Heap allocators are universal generic solution, but there are reasons for the existence
of other allocators, so let's go over pros and cons of this allocator type.

Pros:

- Universal: allocate whatever you want whenever you want.

Cons:

- Slower than specialized allocators, like pool and stack allocators.
- Has high risk of memory fragmentation when lots of small objects are allocated and deallocated.


### String interning

String interning is an important technique that allows to both save the memory and improve application performance
if you use it right. Interned strings are stored in special pool and are never duplicated. That makes equality check,
hash calculating and string copying lighting fast, because interned string object is essentially just a pointer
to real string in interned strings pool. But it also slows down string creation from raw values, because it adds
string interning pass to construction routine.

[Emergence::Memory service](https://github.com/KonstantinTomashevich/Emergence/tree/e8c37b6/Service/Memory) provides
string interning implementation for ASCII strings through `UniqueString` class. Let's take a quick look at how interned 
strings pool is organized there.

![String interning memory structure](/assets/img/EmergenceMemoryManagement/StringInterning.png)

Instead of spreading strings all over the memory (which would cause memory fragmentation), we're storing them in
special memory structure: pool of string stacks. It is essentially pool allocator of stack allocators that are used
to allocate interned string values. When current string stack is filled we allocate new stack and continue to allocate
string through it. Interned strings are never deallocated as it makes interning logic more performant and easy-to-use.
To check whether string value is already interned or not we keep separate hash set of strings that hashes interned
strings by their values instead of their pointers. It allows us to do this check in almost O(1), but, of course,
eats additional memory.

As every other memory-related solution, string interning is not a silver bullet and has its pros and cons.

Pros:

- Provides lighting-fast equality check, hash calculation and string copying.
- Guarantees that strings are not duplicated in memory.

Cons:

- Most implementations store interned values for whole program execution, therefore temporary strings should never
  be interned.
- Interning is not a fast operation, so it shouldn't be called every frame and should not be called on temporary
  strings.

Implementing your own string interning library is much easier than it sounds! If you wanna try, check out 
[my article]({% post_url 2022-07-12-TutorialStringInterning %}) about implementation of trivial string 
interning routine.

### Profiling

...