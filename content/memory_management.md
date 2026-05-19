+++
title = "Practical Memory Management"
date = 2024-11-25

[taxonomies]
tags = ["programming"]
+++

This post covers custom memory allocator strategies — arenas and stack allocators — that eliminate
most common memory management problems in C and C++. The snippets shown here are simplified for
clarity, drawn from a production codebase I work on that has to deal with real-time constraints.

I need to make some things clear from the start:
- None of what I'll be talking about is new. Many large codebases adopt this exact memory-management
  design from the ground up.
- My only goal here is to make this folk knowledge available to anyone looking to incorporate these
  strategies.

Over the last few years, C has been widely criticised for being memory unsafe — it lacks compiler
constraints for object lifetime and ownership semantics that newer languages provide. Although that's
true, you can still build different allocation strategies that will deal with most of these memory
issues for free. We'll provide memory safety through deliberate design rather than language-level
enforcement. Most importantly, it doesn't have to be painful! The memory allocators I'll present are
extremely easy to implement and manage. Rather than adding more complexity to your codebase, you'll
be simplifying it. The cost of such approaches comes through discipline, and may change the way you
handle problems in favour of batched processing, which is almost always a good thing to do.

# Building your own memory system

When writing any API, one should consider the scope and the audience of the API. In the case of a
memory management system, the main concern is centred on memory safety. How many safety guard-rails
should we build?

Guard-rails obviously come with their own performance costs. For instance, if we wanted a very
robust system that disallows any use-after-free, we wouldn't be returning direct pointers to the
memory to our users. To deal with that, we could use handles and manage their coherence internally
(check [this post](https://floooh.github.io/2018/06/17/handles-vs-pointers.html), by Andre
Weissflog, for more information). This would require every allocation to have a unique ID, every
read request to be checked if IDs match, etc.

In the majority of cases, my applications have strict requirements for real-time performance —
adding a new layer of indirection is not an option. In order to meet these requirements we have to
make a compromise: procedures establish protocols between the caller and callee, and correctness is
upheld by convention rather than enforced at runtime. The allocator won't validate that you're using
it correctly — that responsibility falls on the programmer. This is a deliberate trade-off: we give
up runtime safety checks in exchange for predictable, minimal-overhead memory operations. On the
other hand, of course you can always write validation checks that are enabled only in debug mode ---
in fact, this is a common practice in our codebase, all protocols are established as code blocks
that assert all assumptions that the function makes when dealing with the input.

One warning regarding the allocators presented in this post: they are **not** implemented to be
thread-safe. You can obviously implement a thread-safe version easily, however I prefer using these
allocators in a per-thread basis rather than cross-threads --- which avoids contentions, race
conditions, and memory sync issues.

# Alignment: rules for memory reads

The CPU memory reads cannot be done willy nilly at any given address. Modern architectures are
optimised to read contiguous memory with a certain alignment --- which makes stepping through memory a
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
    uint8_t *memory;
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
    uint8_t *memory;             // 8 bytes.
    uint32_t allocation_count;   // 4 bytes.
    uint8_t  padding1[4];        // Inserted padding of 4 bytes.
    double   some_metric;        // 8 bytes.
    float    some_other_metric;  // 4 bytes.
    uint8_t  padding2[4];        // Inserted padding of 4 bytes.
};
```

Notice how we're wasting 8 bytes of memory for each instance of our struct.

But really, why does the compiler insert padding between struct members? Most modern CPU
architectures require data to be aligned according to its size. Misaligned access can lead to
performance penalties on some platforms (like x86), and in other cases it can even cause hardware
exceptions to be thrown.

One thing I didn't explain is why the compiler has to put those 4 bytes of padding after the last
structure member. If for some reason you have a contiguous array of `Foo` instances, in order to
traverse through the array we would use the alignment of `Foo` (8 bytes) --- and if it wasn't for
those last 4 bytes, the address of the next `Foo` instance would be misaligned! Once again, the
compiler has to account for the worst case scenario.

Fixing our bad memory usage is simple, we just rearrange the members taking into account their
sizes:

```cpp
struct Foo {
    uint8_t *memory;             // 8 bytes.
    double   some_metric;        // 8 bytes.
    uint32_t allocation_count;   // 4 bytes.
    float    some_other_metric;  // 4 bytes.
};
```

Analogously, every time we allocate memory in our custom allocators we'll have to make sure to
account for the necessary alignment restrictions.

# Arena Allocator

The simplest allocator --- yet sufficient for almost all use-cases --- is the *arena memory allocator*.
Its construction is ridiculously simple: a pointer to the block of memory being managed, the total
maximum capacity of the block, and an offset relative to the start of the block to the free
space:

```cpp
struct Arena {
    uint8_t *buffer;
    size_t   capacity;
    size_t   offset;
};
```

Having only this amount of information to deal with, the arena can only be used to accumulate
allocations for a certain period and then free all of the allocated memory at once. This constraint
is perfect for modelling the concept of a lifetime: objects allocated in the same arena, have a
common lifetime end --- that is, when the arena has its offset reset. This allows one to trivially
deal with lifetime issues that languages like Rust enforce at the compiler level.

It is to be noted that your style of programming with arenas may differ from the typical programming
you see out there. It is common to see programs where ownership of memory isn't strictly defined in
the course of the program execution. For this exact reason, modern C++ codebases often use
`shared_ptr` and reference counting — besides the overhead, this approach makes reasoning about
lifetimes harder, not easier.

When programming with arenas, one commonly thinks in groups of allocations (hence lifetime groups),
and chunks of objects. Work is mainly done with these chunks in mind, improving cache spatial and
temporal locality. This way of programming is sometimes called "data-oriented programming" — it aligns with how
memory hierarchies are designed to be used.

Many problems that arise from object-level memory management are eliminated when you
design your program with memory arenas in mind. For instance, ownership and lifetime problems are
almost a non-issue and an easy-to-solve problem.

## Allocating memory blocks

When allocating a new block of memory in the arena, we have to account for the alignment of the
structure or array that will be allocated. By the simplicity of the arena, computing the next
address that satisfies the required alignment is a pretty simple task:

```cpp
uintptr_t align_forward(uintptr_t ptr_addr, size_t alignment) {
    uintptr_t mod_align = ptr_addr & (alignment - 1); // Same as `ptr_addr % alignment`
    if (mod_align != 0) {
        ptr_addr += alignment - mod_align;
    }
    return ptr_addr;
}
```

where the parameter `alignment` is assumed to be a power of two.

Having this auxiliary function at hand, making a new allocation can be very easily done:

```cpp
uint8_t *arena_alloc_align(Arena *arena, size_t size_bytes, size_t alignment) {
    if (arena == nullptr || arena->capacity == 0 || size_bytes == 0) {
        return nullptr;
    }

    uintptr_t memory_addr    = (uintptr_t)arena->buffer;
    uintptr_t new_block_addr = align_forward(memory_addr + arena->offset, alignment);

    if (new_block_addr + size_bytes > arena->capacity + memory_addr) {
        // Not enough free memory.
        return nullptr;
    }

    // Commit the new block of memory.
    arena->offset = (size_t)(size_bytes + new_block_addr - memory_addr);

    uint8_t *new_block = (uint8_t *)new_block_addr;
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
    Arena *arena;
    size_t saved_offset;
};

ArenaCheckpoint arena_make_checkpoint(Arena *arena) {
    return ArenaCheckpoint{arena, arena->offset};
}

void arena_restore_checkpoint(ArenaCheckpoint checkpoint) {
    checkpoint.arena->offset = checkpoint.saved_offset;
}
```

You can certainly extend the behaviour of checkpoints. For instance, you can create a kind of
auto-restoring checkpoint with the use of destructors if you are into that.

Although extremely simple, checkpoints are one of the most useful constructs you can build on top of
an arena allocator. Here I'll list two patterns that I use all of the time:

- **Temporary work.** Sometimes, in order to make some computation, you may require dynamic allocation:
  ```cpp
  ComputeResult compute_foo(Bar *bar, int bar_count, Arena *work_arena) {
      ArenaCheckpoint work_checkpoint = arena_make_checkpoint(work_arena);

      // `foobar` is used temporarily to compute `result`.
      Array<int> foobar = array_make<int>(work_arena, bar_count);
      for (int i = 0; i < bar_count; ++i) {
          // Compute foobar[i].
      }

      ComputeResult result = {};
      foobar_result(&foobar, &result);

      arena_restore_checkpoint(work_checkpoint);
      return result;
  }
  ```
  If you don't like the fact that you have to remember to restore the checkpoint, you can implement
  either a destructor for the checkpoint or add a `defer` macro that handles this.
- **Lifetime as a parameter.** When using an arena allocator, you can mask from the callee the
  actual lifetime of the object they will allocate. The callee only has to know two things: does
  the object's lifetime start and end within my scope (temporary work), or does it start in my
  scope and end in the caller's?

  Consider, for instance, that you have an opaque pointer to the `GPUContext` of an application. The
  actual definition of `GPUContext` has to be deferred to their corresponding backend implementation
  file (e.g. `gpu_context_vulkan.cpp`) --- hence the caller cannot simply instantiate `GPUContext`
  on the stack, it has to be allocated at runtime. In order to create a new GPU context, the caller
  passes an arena to `gpu_context_make` as the lifetime of the `GPUContext` itself:
  ```cpp
  GPUContext *gpu_context_make(Arena *persistent_arena) {
      GPUContext *context = (GPUContext *)arena_alloc_align(persistent_arena, sizeof(GPUContext), alignof(GPUContext));

      // Initialise the context ...

      return context;
  }
  ```

# Stack Allocator

A stack allocator is nothing more than a contiguous memory block which we divide in order to offer
memory space to consumers. In order to avoid memory fragmentation, we only allow the last allocated
block to be freed.

## Headers: storing relevant information

Each memory block allocated by our stack allocator will be accompanied by a header that will carry
some basic information about the memory block it's associated with.

```cpp
struct StackHeader {
    size_t padding;
    size_t capacity;
    size_t previous_offset;
};
```

Let me explain what each one of these fields mean:

- `padding`: The offset relative to the *end* of the previously allocated memory block until the start
  of the current memory block. This is here due to the different alignment requirements of each block.
- `capacity`: The total capacity, in bytes, of the current memory block.
- `previous_offset`: The offset, relative to `buffer`, to the start of the previously allocated
  memory block. This allows us to traverse the stack backwards.

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
    uint8_t *buffer;
    size_t   capacity;
    size_t   offset;
    size_t   previous_offset;
};
```
- `buffer`: Pointer to the start of the memory block managed by the allocator.
- `capacity`: The maximum capacity in bytes of the allocator's memory block.
- `offset`: The offset, relative to `buffer`, to the start of the region available for allocations.
- `previous_offset`: The offset, relative to `buffer`, to the start of the last allocated memory block.

This can be visualised by the following diagram:

```md
                                         offset
                                           |
                                           v
  |header 1|memory 1|++++|header 2|memory 2|++free space++|
  ^                               ^                       ^
  |                               |                       |
buffer                         previous                  end
  |                             offset                    |
  |                                                       |
  |                                                       |
  |--------------------- capacity ------------------------|
```

The allocator can be either the owner or merely the manager of the memory pointed by `Stack::buffer`.
In my case, I opted for using allocators as mere managers, so you need to initialise them with a
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
    uintptr_t ptr_addr,
    size_t    alignment,
    size_t    header_size,
    size_t    header_alignment) {
    if (alignment < header_alignment) {
        alignment = header_alignment;
    }

    size_t padding = 0;
    uintptr_t mod_align = ptr_addr & (alignment - 1);  // Same as `ptr_addr % alignment`.
    if (mod_align != 0) {
        padding = alignment - mod_align;
    }

    // Ensure there's enough space for the header before the block.
    if (padding < header_size) {
        size_t needed = header_size - padding;
        padding += alignment * ((needed + alignment - 1) / alignment);
    }

    return padding;
}
```

It should be stressed that the `alignment` and `header_alignment` parameters should always be powers
of two.

To allocate a new block of memory in the stack, we can proceed as follows:

```cpp
uint8_t *stack_alloc_align(Stack *stack, size_t size_bytes, uint32_t alignment) {
    size_t current_capacity = stack->capacity;
    size_t current_offset   = stack->offset;

    if (current_capacity == 0 || size_bytes == 0) {
        return nullptr;
    }

    uint8_t *free_memory = stack->buffer + current_offset;

    size_t padding = padding_with_header(
        (uintptr_t)free_memory,
        alignment,
        sizeof(StackHeader),
        alignof(StackHeader));

    if (padding + size_bytes > current_capacity - current_offset) {
        return nullptr;  // Not enough memory...
    }

    // Address to the start of the new block of memory.
    uint8_t *new_block = free_memory + padding;

    // Write to the header associated with the new block of memory.
    StackHeader *new_header     = (StackHeader *)(new_block - sizeof(StackHeader));
    new_header->padding         = padding;
    new_header->capacity        = size_bytes;
    new_header->previous_offset = stack->previous_offset;

    // Update the stack offsets.
    stack->previous_offset = current_offset + padding;
    stack->offset          = current_offset + padding + size_bytes;

    return new_block;
}
```

# Final comments

Manual memory management systems are actually quite fun and interesting. It is very simple to
construct a concise, easy to use, safe, and performant memory system that will remove most of the
common headaches that arise from ad-hoc memory management. It goes without saying that having a good memory
system frees you from dealing with individual lifetimes, ownership problems, and the paranoia that
`malloc` will inherently generate when used throughout the codebase.

Most of the time, all that you need is an arena. You can also combine the use of the arena with a
stack allocator for more intricate memory arrangements.

# Further reading material for the nerds

- [Memory Management Reference](https://www.memorymanagement.org/).
- [Untangling Lifetimes: The Arena Allocator](https://www.rfleury.com/p/untangling-lifetimes-the-arena-allocator),
  by Ryan Fleury.
- [Memory Allocation Strategies series](https://www.gingerbill.org/series/memory-allocation-strategies/), by gingerBill.
- [Handles are better than pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html), by
  Andre Weissflog.
- Aaron MacDougall's GDC 2016 talk on [Building a Low-Fragmentation Memory System for 64-bit Games](https://gdcvault.com/play/1023309/Building-a-Low-Fragmentation-Memory).
