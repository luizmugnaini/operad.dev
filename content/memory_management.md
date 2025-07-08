+++
title = "Memory management strategies"
date = 2024-11-25

[taxonomies]
tags = ["memory"]
+++

In this post, I'll be detailing how C and C++ can be programmed in a sane and memory-safe way. We'll
be going through some custom memory allocator strategies that provide simple ways of dissolving
common memory problems.

This post is inspired by my C++ core library called [Presheaf](https://git.sr.ht/~presheaf/presheaf-lib),
whose source code is freely available. The main goal of the library is to deliver the programming
experience I wanted and C wasn't able to fully deliver.

<!-- more -->

Over the last few years C got the fame of being a memory unsafe language due to its lack of
intrinsic constraints that ensure proper object lifetime and ownership management - like you
commonly have in new trendy languages. You might ask why bother learning memory strategies in C if I
can simply use my language Z that has a borrow-checker or a garbage collector: well, language Z is
not suited for all use cases - surely not for me. The borrow-checker is a pain to deal with and most
memory strategies in that land, like ECS, are based in making the compiler shut up by hiding
intricate lifetimes inside of longer living objects. Do I even need to talk about the garbage
collector approach?

# Building your own memory system

When writing any API, one should question himself the scope and the audience of the API. In the case
of a memory management system, the main concern is centred in memory safety. How much safety guard-rails
should we build, and how much hand-holding does our end-user need?

Guard-rails obviously comes with their own performance costs. For instance, if we wanted a very
robust system that disallows for any use-after-free, we wouldn't be returning direct pointers to the
memory to our users. To deal with that, we could use handles and manage their coherence internally
(check [this post](https://floooh.github.io/2018/06/17/handles-vs-pointers.html), by Andre
Weissflog, for more information). This would require every allocation to have a unique ID, every
read request to be checked if ID's match, etc.

In my personal case I don't really need all of this hand-holding and training-wheels. All I need is
a performant system that can be easily managed, and ensures some level of memory safety. The
[Presheaf library](https://git.sr.ht/~presheaf/presheaf-lib) leans into this very philosophy -
obvious protocols exist between caller and callee, if the caller wants to break one of the these
protocol assumptions, the callee won't try to stop the caller fearing stupidity of the programmer.
In summary, the programmer is always treated as an intelligent being that knows what they are doing.
As simple and obvious as this philosophy may seem, the current state of the software industry took a
turn in favor of lazyness, with the excuse that the "developer experience" is the king. I certainly
don't follow this way of thinking, in fact: the end-user is the king, and performance is queen.

# Alignment: rules for memory reads

The CPU memory reads cannot be done willy nilly at any given address. Modern architectures are
optimised to read contiguous memory with a certain alignment - which makes stepping through memory a
regular task (as opposed to jumping around randomly). This alignment is always a power of two and
depends on the size of the memory units (e.g. a struct member) we want to read. In C++ you can query
the memory alignment for a given type `T` using `alignof(T)` (in C, you can use `_Alignof(T)`).

For instance, if we have an array of floats (each float with a size of 4 bytes), the address of the
`n`th element of the array in memory should be of the form `4 n + c` where `c` is the address of the
first element of the array. Hence we say that the alignment of a float is 4 bytes.

For structs, the compiler may need to add paddings in order to satisfy alignment conditions. A lost
art in programming is the arrangement of struct members for optimal alignment. Let's see this in
practice.

Suppose I have a struct `Foo` that has the following memory layout:

```cpp
struct Foo {
    uint8_t* memory;
    uint32_t allocation_count;
    double   some_metric;
    float    some_other_metric;
};
```

From the point of view of the compiler, the actual memory layout of `Foo` has to make sure that the
alignment of each struct member is valid. Thus in reality, the arrangement of bytes composing
`Foo` is laid as follows:

```cpp
struct Foo {
    uint8_t* memory;             // 8 bytes.
    uint32_t allocation_count;   // 4 bytes.
    // ---------------------------> Invisible padding of 4 bytes.
    double   some_metric;        // 8 bytes.
    float    some_other_metric;  // 4 bytes.
    // ---------------------------> Invisible padding of 4 bytes.
};
```

Notice how we're wasting 8 bytes of memory for each instance of our struct.

But really, why does the compiler put those paddings between our members? For the sake of
contradiction, suppose that the compiler didn't put those padding bytes. If we wanted to access the
member `Foo::some_metric` (which has an alignment of 8 bytes), we would start to read `Foo` in the
same address as the first member - then we would advance by our alignment of 8 bytes at a time until
we supposedly reach `Foo::some_metric`. Surprise! We won't be able to reach our destination, in
fact, we would've passed the address of the target by 4 bytes. For this exact reason, the compiler
has to arrange memory for the worst case scenario - the largest alignment in the collection of
members has to be the global alignment of the structure itself. In our case, the compiler sees that
the largest alignment is 8 bytes.

One thing I didn't explain is why the compiler has to put those 4 bytes of padding after
the last structure member. If for some reason you have a contiguous array of `Foo` instances, in
order to traverse through the array we would use the alignment of `Foo` (8 bytes) - and if it wasn't
for those last 4 bytes, we wouldn't reach the next `Foo` in the line! Once again, the compiler has to
account for the worst case scenario.

Fixing our bad memory usage is simple, we just rearrange the members taking into account their
sizes:

```cpp
struct Foo {
    uint8_t* memory;             // 8 bytes.
    double   some_metric;        // 8 bytes.
    uint32_t allocation_count;   // 4 bytes.
    float    some_other_metric;  // 4 bytes.
};
```

Analogously, every time we allocate memory in our custom allocators we'll have to make sure to
account for the necessary alignment restrictions.

# Arena Allocator

The simplest allocator - yet sufficient for almost all use-cases - is the *arena memory allocator*.
Its construction is ridiculously simple: a pointer to the block of memory being managed, the total
maximum capacity of the block, and an offset relative to the start of the block to the free
space:

```cpp
struct Arena {
    uint8_t* memory;
    size_t   capacity;
    size_t   offset;
};
```

Having only this amount of information to deal with, the arena can only be used to accumulate
allocations for a certain period and then free all of the allocated memory at once. This constraint
is perfect for modelling the concept of a lifetime: objects allocated in the same arena, have a
common lifetime end - that is, when the arena has its offset reset. This allows one to trivially
deal with lifetime issues that languages like Rust try to impose in their compiler.

It is to be noted that your style of programming with arenas may differ from the typical
programming you see out there. It is common to see programs where ownership of memory is poorly
defined in the course of the program lifetime. For this exact reason, people in the land of
"Modern C++" resort to "smart" pointers - instead of solving the root cause, this "solution" only
remedy the problem of a poorly designed software by means of runtime cost and, consequently,
absurdly horrible performance.

When programming with arenas, one commonly thinks in groups of allocations (hence lifetime groups),
and chunks of objects. Work is mainly done with these chunks in mind, improving cache spacial and
temporal locality. This way of programming is sometimes named "data oriented programming", but
in reality this is simply the way computers where designed to be used - we don't even need a term
for that, it's merely non-pessimised programming if you think about it.

Many of the made-up problems created by modern software practices are completely dissolved when you
design your program with memory arenas in mind instead of thinking about the program at the
object-level. For instance, ownership and lifetime problems are almost a non-issue and an easy to
solve problem.

## Allocating memory blocks

When allocating a new block of memory in the arena, we have to account for the alignment of the
structure or array that will be allocated. By the simplicity of the arena, computing the next
address that satisfy the required alignment is a pretty simple task:

```cpp
size_t align_forward(uintptr_t ptr_addr, size_t alignment) {
    size_t mod_align = ptr_addr & (alignment - 1); // Same as `ptr_addr % alignment`
    if (mod_align != 0) {
        ptr_addr += alignment - mod_align;
    }
    return ptr_addr;
}
```

where the parameter `alignment` is assumed to be a power of two.

Having this auxiliar function at hand, making a new allocation can be very easily done:

```cpp
uint8_t* arena_alloc_align(Arena* arena, size_t size_bytes, uint32_t alignment) {
    if (arena == nullptr || arena->capacity == 0 || size_bytes == 0) {
        return nullptr;
    }

    uintptr_t memory_addr    = (uintptr_t)arena->buf;
    uintptr_t new_block_addr = align_forward(memory_addr + arena->offset, alignment);

    if (new_block_addr + size_bytes > arena->capacity + memory_addr) {
        // Not enough free memory.
        return nullptr;
    }

    // Commit the new block of memory.
    arena->offset = (size_t)(size_bytes + new_block_addr - memory_addr);

    uint8_t* new_block = (uint8_t*)new_block_addr;
    memset(new_block, 0, size_bytes);

    return new_block;
}
```

You should also create procedures that deal with the following operations: clearing the arena,
resizing an already allocated block of memory, etc.

## Temporary allocations with checkpoints

Having the constraint of only being able to free all memory at once can be a bad restriction once
you want to perform temporary computations that shouldn't be sticking around in the allocator. In
order to overcome this constraint, we can create a checkpoint system that records the current state of
the arena and is capable of restoring the allocator once the memory allocated from the checkpoint
onwards isn't needed anymore. This amounts to a simple implementation like the following:

```cpp
struct ArenaCheckpoint {
    size_t saved_offset;
};

ArenaCheckpoint arena_make_checkpoint(Arena* arena) {
    return ArenaCheckpoint{arena->offset};
}

void arena_restore_state(Arena* arena, ArenaCheckpoint* checkpoint) {
    arena->offset = checkpoint->saved_offset;
}
```

You can certainly extend the behaviour of checkpoints. For instance, you can create a kind
auto-restoring checkpoint with the use of destructors. Moreover, I would recommend creating debug
checks in order to ensure the correctness of the checkpoints being restored (e.g. does it come from
the same arena? is the offset valid? etc.).

# Stack Allocator

A stack allocator is nothing more than a contiguous memory block which we divide in order to offer
memory space to consumers. In order to avoid memory fragmentation, we only allow the last allocated
block to be freed.

## Headers: storing relevant information

Each memory block allocated by our stack allocator will be accompanied by a header that will carry
some basic information about the memory block it's associated with.

```cpp
struct StackAllocHeader {
    size_t padding;
    size_t capacity;
    size_t previous_offset;
};
```

Let me explain what each one of these fields mean:

- `padding`: The offset relative to the *end* of the previously allocated memory block until the start
  of the current memory block. This is here due to the different alignment requirements of each block.
- `capacity`: The total capacity, in bytes, of the current memory block.
- `previous_offset`: The offset relative to the *start* of previously allocated block until the
  start of the current memory block. This will help us to traverse the stack backwards.

You can visualise the header members as follows:

```md
         previous offset              |alignment|              |------capacity------|
                |                     |         |              ^                    ^
                v                     v         v              |                    |
|previous header|previous memory block|+++++++++|current header|current memory block|
                                      ^                        ^
                                      |---------padding--------|
```

## Allocator Structure

On to the stack allocator proper! The basic layout of the allocator looks like this:

```cpp
struct Stack {
    uint8_t* memory;
    size_t   capacity;
    size_t   offset;
    size_t   previous_offset;
};
```
- `memory`: Pointer to the start of the memory block managed by the allocator.
- `capacity`: The maximum capacity in bytes of the allocator's memory block.
- `offset`: The offset, relative to `memory`, to the start of the region available for allocations.
- `previous_offset`: The offset, relative to `memory`, to the start of the last allocated memory block.

This can be visualised by the following diagram:

```md
                                         offset
                                           |
                                           v
  |header 1|memory 1|++++|header 2|memory 2|++free space++|
  ^                               ^                       ^
  |                               |                       |
memory                         previous                  end
  |                             offset                    |
  |                                                       |
  |                                                       |
  |--------------------- capacity ------------------------|
```

The allocator can be either the owner or merely the manager of the memory pointed by `Stack::memory`.
In Presheaf I opted for using allocators as mere managers, so you need to initialise them with a
valid pointer to a previously allocated block of memory.

## Allocating Blocks

Each block provided by the stack allocator consists of an *alignment* offset with respect to the end
of the previous block, a *header*, and the available block of *memory* requested by the user.

```md
      |alignment|    memory
      |         |      |
      v         v      v
| ... |+++++++++|header|memory block| ... |
      ^                ^
      |                |
      |----padding-----|
```

The block of memory is preceded by a *padding* that comprises both the alignment needed for the
memory block and its corresponding header.

In order to compute the padding needed by the block we can implement the following function:

```c
size_t padding_with_header(
    uintptr_t  ptr_addr,
    size_t     alignment,
    size_t     header_size,
    size_t     header_alignment) {
    // Calculate the padding necessary for the new block of memory.
    size_t padding   = 0;
    size_t mod_align = ptr_addr & (alignment - 1);  // Same as `ptr_addr % alignment`.
    if (mod_align != 0) {
        padding += alignment - mod_align;
    }
    ptr_addr += padding;

    // Add the padding necessary for the header alignment.
    size_t mod_header = ptr_addr & (header_alignment - 1);  // Same as `ptr_addr % header_alignment`.
    if (mod_header != 0) {
        padding += header_alignment - mod_header;
    }

    // The padding should at least be able to contain the header.
    padding += header_size;

    return padding;
}
```

It should be stressed that the `alignment` and `header_alignment` parameters should always be powers
of two.

To allocate a new block of memory in the stack, we can proceed as follows:

```cpp
uint8_t* stack_alloc_align(Stack* stack, size_t size_bytes, uint32_t alignment) {
    size_t current_capacity = stack->capacity;
    size_t current_offset   = stack->offset;

    if (current_capacity == 0 || size_bytes == 0) {
        return nullptr;
    }

    uint8_t* free_memory = stack->buf + current_offset;

    size_t padding = padding_with_header(
        (uintptr_t)free_memory,
        alignment,
        sizeof(StackHeader),
        alignof(StackHeader));

    if (padding + size_bytes > current_capacity - current_offset) {
        return nullptr;  // Not enough memory...
    }

    // Address to the start of the new block of memory.
    uint8_t* new_block = free_memory + padding;

    // Write to the header associated with the new block of memory.
    StackHeader* new_header     = (StackHeader*)(new_block - sizeof(StackHeader));
    new_header->padding         = padding;
    new_header->capacity        = size_bytes;
    new_header->previous_offset = stack->previous_offset;

    // Update the stack offsets.
    stack->previous_offset = current_offset + padding;
    stack->offset          = current_offset + padding + size_bytes;

    memset(new_block, size_bytes, 0);
    return new_block;
}
```

# Final comments

Manual memory management systems are actually quite fun and interesting. It is very simple to
construct a consise, easy to use, safe, and performant memory system that will remove most of the
common headaches created by bad software practices. It goes without saying that having a good memory
system frees you from dealing with individual lifetimes, ownership problems, and the paranoia that
`malloc` will inherently generate when used throughout the codebase.

Most of the time, all that you need is an arena. You can also combine the use of the arena with a
stack allocator for more intricate memory arrangements.

If you wish to see the actual implementation of these allocators in the Presheaf library, please
refer to the [source code](https://git.sr.ht/~presheaf/presheaf-lib).

# Further reading material for the nerds

- [Memory Management Reference](https://www.memorymanagement.org/).
- [Untangling Lifetimes: The Arena Allocator](https://www.rfleury.com/p/untangling-lifetimes-the-arena-allocator),
  by Ryan Fleury.
- [Memory Allocation Strategies series](https://www.gingerbill.org/series/memory-allocation-strategies/), by gingerBill.
- [Handles are better than pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html), by
  Andre Weissflog.
- Aaron MacDougall's GDC 2016 talk on [Building a Low-Fragmentation Memory System for 64-bit Games](https://gdcvault.com/play/1023309/Building-a-Low-Fragmentation-Memory).
