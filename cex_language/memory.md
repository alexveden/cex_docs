---
title: "Memory management"
---

## The problem of memory management in C
C has a long-lasting history of memory management issues. Many modern languages proposed multiple solutions for these issues: RAII, borrow checkers, garbage collection, allocators, etc. All of them work and solve the memory problem to some extent, but sometimes adding new sets of problems in different places. 

From my prospective, the root cause of the C memory problem is hidden memory allocation. When developer works with a function which does memory allocation, it's hard to remember its behavior without looking into source code or documentation. Absence of explicit indication of memory allocation lead to the flaws with memory handling, for example: memory leaks, use after free, or performance issues.

While C remains system and low-level language, it's important to have precise control over code behavior and memory allocations. So in my opinion, RAII and garbage collection are alien approaches to C philosophy, but on the other hand modern languages like `Zig` or `C3` have allocator centric approach, which is more explicit and suitable for C.

## Modern way of memory management in CEX
### Allocator-centric approach
CEX tries to adopt allocator-centric approach to memory management, which help to follow those principles:

* **Explicit memory allocation.** Each object (class) or function that may allocate memory has to have an allocator parameter. This requirement, adds explicit API signature hints, and communicates about memory implications of a function without deep dive into documentation or source code.
* **Transparent memory management.** All memory operations are provided by `IAllocator` interface, which can be interchangeable allocator object of different type.
* **Memory scoping**. When possible memory usage should be limited by scope, which naturally regulates lifetimes of allocated memory and automatically free it after exiting scope.
* **UnitTest Friendly**. Allocators allowing implementation of additional levels of memory safety when run in unit test environment. For example, CEX allocators add special poisoned areas around allocated blocks, which trigger address sanitizer when this region accesses with user code. Allocators open door for a memory leak checks, or extra memory error simulations for better out-of-memory error handling.
* **Standard and Temporary allocators**. Sometimes it's useful to have initialized allocator under your belt for short-lived temporary operations. CEX provides two global allocators by default: `mem$` - is a standard heap allocator using `malloc/realloc/free`, and `tmem$` - is dynamic arena allocator of small size (about 256k of per page).  

### Example
This is a small example of key memory management concepts in CEX:

```c
mem$scope(tmem$, _) /* <1> */
{
    arr$(char*) incl_path = arr$new(incl_path, _); /* <2> */
    for$each (p, alt_include_path) {
        arr$push(incl_path, p);  /* <3> */
        if (!os.path.exists(p)) { log$warn("alt_include_path not exists: %s\n", p); }
    }
} /* <4> */
```
1. Initializes a temporary allocator (`tmem$`) scope in `mem$scope(tmem$, _) {...}` and assigns it as a variable `_` (you can use any name).
2. Initializes dynamic array with the scoped allocator variable `_`, allocates new memory.
3. May allocate memory
4. All memory will be freed at exit from this scope

### Lifetimes and scopes
Use of memory scopes naturally regulates lifetime of initialized memory. From the example above you can't use `incl_path` variable outside of `mem$scope`. And more to say, that memory will be automatically freed after exiting scope. This design approach significantly reduces surface for use after free errors in general.


### Temporary memory allocator
Dealing with lots of small memory allocations always was a pain in C, because we need to deallocate them at the end, also because of potential overhead each individual memory allocation might have. Temporary allocator in CEX works as a small-page (around 256kb) memory arena, which can be dynamically resized when needed. The most important feature of temporary arena allocator it does the full cleanup at the `mem$scope` exit automatically.

Temporary allocator is always available via `tmem$` global variable and can be used anytime at the program lifetime. It allowed to be used only inside `mem$scope`, with support of up to 32 levels of `mem$scope` nesting. At the end of the program, CEX will automatically finalize and free all allocated memory.

You can find more technical details about implementation below in this article.

## Memory management in CEX
### Allocators
Allocators add many benefits into codebase design and development experience:

* All memory allocating functions or objects become explicit, because they require `IAllocator` argument 
* Logic of the code become detached from memory model, the same dynamic array can be backed by heap, arena, or stack based static char buffer with the same allocator interface. The same piece of code may work on Linux OS or embedded device without changes to memory allocation model.
* Allocators may add testing capabilities, i.e. simulating out-of-mem errors in unit tests, or adding memory checks or extra integrity checks of memory allocations
* There are multiple memory allocation models (heap, arenas, temp allocation), so you can find the best type of allocator for your needs and use case.
* It's easier to trace and doing memory benchmarks with allocators.
* Automatic garbage collection with `mem$scope` and arena allocators - you'll get everything freed on scope exit

### Allocator interface
The allocator interface is represented by `IAllocator` type, which is an interface structure of function pointers for generic operations. Allocators in CEX support `malloc/realloc/calloc/free` functions similar to their analogs in C, the only optional parameter is alignment for requested memory region.


```c
#define IAllocator const struct Allocator_i* 

typedef struct Allocator_i
{
    // >>> cacheline
    alignas(64) void* (*const malloc)(IAllocator self, usize size, usize alignment);
    void* (*const calloc)(IAllocator self, usize nmemb, usize size, usize alignment);
    void* (*const realloc)(IAllocator self, void* ptr, usize new_size, usize alignment);
    void* (*const free)(IAllocator self, void* ptr);
    const struct Allocator_i* (*const scope_enter)(IAllocator self);   /* Only for arenas/temp alloc! */
    void (*const scope_exit)(IAllocator self);    /* Only for arenas/temp alloc! */
    u32 (*const scope_depth)(IAllocator self);  /* Current mem$scope depth */
    struct {
        u32 magic_id;
        bool is_arena;
        bool is_temp;
    } meta;
    //<<< 64 byte cacheline
} Allocator_i;

```

### `mem$` API

You shouldn't use allocator interface directly (it's less convenient), so it's better to use memory specific macros:

* `mem$malloc(allocator, size, [alignment])` - allocates uninitialized memory with `allocator`, `size` in bytes, `alignment` parameter is optional, by default it's system specific alignment (up to 64 byte alignment is supported)
* `mem$calloc(allocator, nmemb, size, [alignment])` - allocates zero-initialized memory with `allocator`, `nbemb` elements of `size` each, `alignment` parameter is optional, by default it's system specific alignment (up to 64 byte alignment is supported)
* `mem$realloc(allocator, old_ptr, size, [alignment])` - reallocates previously initialized `old_ptr` with `allocator`, `alighment` parameter is optional and must match initial alignment of a `old_ptr`
* `mem$free(allocator, old_ptr)` - frees `old_prt` and implicitly set it to `NULL` to avoid use-after-free issues.
* `mem$new(allocator, T)` - generic allocation of new instance of `T` (type), with respect of its size and alignment.

Allocator scoping:

* `mem$arena(page_size) { ... }` - enters new instance of allocator arena with the `page_size`.
* `mem$scope(arena_or_tmem$, scope_var) { ... }` - opens new memory scope (works only with arena allocators or temp allocator)


### Dynamic Arenas
Dynamic arenas using an array of dynamically allocated pages, each page has static size and allocated on heap. When you allocate memory on arena and there is enough room on page, the arena allocates this chunk of memory inside page (simply moving a pointer without real allocation). If your memory request is big enough, the arena creates new page while keeping all old pages untouched and manages new allocation on the new page.

Arenas are designed to work with `mem$scope()`, this allowing you create temporary memory allocation, without worrying about cleanup. Once scope is left, the arena will deallocate all memory and return to the initial state. This approach allowing to use up to 32 levels of `mem$scope()` nesting. Essentially it is exact mechanism that fuels `tmem$` - temporary allocator in CEX.

Working with arenas:

::: {.panel-tabset}

#### Direct initialization 
```c
IAllocator arena = AllocatorArena.create(4096); /*<1>*/
u8* p = mem$malloc(arena, 100);  /* <2> */

mem$scope(arena, tal) /*<3>*/
{
    u8* p2 = mem$malloc(tal, 100000);  /*<4>*/

    mem$scope(arena, tal)
    {
        u8* p3 = mem$malloc(tal, 100);
    } /*<5>*/
} /*<6>*/

AllocatorArena.destroy(arena); /*<7>*/
```
1. New arena with 4096 byte page
2. Allocating some memory from arena
3. Entering new memory scope
4. Allocation size exceeds page size, new page will be allocated then. `p` address remain the same!
5. At scope exit `p3` will be freed, `p2` and `p` remain
6. At scope exit `p2` will be freed, excess pages will be freed, `p` remains
7. Arena destruction, all pages are freed, `p` is invalid now.

#### Arena scope
```c
mem$arena(4096, arena)
{
    // This needs extra page
    u8* p2 = mem$malloc(arena, 10040);
    mem$scope(arena, tal)
    {
        u8* p3 = mem$malloc(tal, 100);
    }
}
```

#### Temp allocator
```c
mem$scope(tmem$, _) /* <1> */
{
    u8* p2 = mem$malloc(_, 110241024); /* <2> */

    mem$scope(tmem$, _) /* <3> */
    {
        u8* p3 = mem$malloc(_, 100);
    } /* <4>*/
} /* <5> */
```
1. Initializes a temporary allocator (`tmem$`) scope in `mem$scope(tmem$, _) {...}` and assigns it as a variable `_` (you can use any name). `_` is a pattern for temp allocator in CEX.
2. New page for temp allocator created, because size exceeds existing page size
3. Nested scope is allowed
4. Scope exit `p3` automatically cleaned up
5. Scope exit `p2` cleaned up + extra page freed.

:::


### Standard allocators

There are two general purpose allocators globally available out of the box for CEX:

* `mem$` - is a heap allocator, the same old `malloc/free` type of allocation, with extra alignment support. In unit tests this allocator provides simple memory leak checks even without address sanitizer enabled.
* `tmem$` - dynamic arena, with 256kb page size, used for short lived temporary operations, cleans up pages automatically at program exit. Does page allocation only at the first allocation, otherwise remain global static struct instance (about 128 bytes size). Thread safe, uses `thread_local`.


### Caveats
#### Do cross-scope memory access carefully
Never reallocate memory from one scope, in the nested scope, which will automatically lead to use-after-free issue. This is a bad example:
```c
mem$scope(tmem$, _)
{
    u8* p2 = mem$malloc(_, 100); /* <1> */

    mem$scope(tmem$, _)
    {
        p2 = mem$realloc(_, p2, 110241024); /* <2> */
    } /* <3>*/

    if(p2[128] == '0') { /* OOPS */} /* <4> */
} 
```
1. Initially allocation at first scope
2. `realloc` uses different scope depth, this might lead to assertion in CEX unit test
3. `p2` automatically freed, because now it belongs to different scope
4. You'll face use-after-free, which typically expressed use-after-poison in temp allocator.

> [!TIP]
> 
> CEX does its best to catch these cases in unit test mode, it will raise an assertion at the `mem$realloc` line with some meaningful error about this. Standard CEX collections like dynamic arrays `arr$` and hashmap `hm$` also get triggered then they need to resize in a different level of `mem$scope`.

#### Be careful with reallocations on arenas
CEX arenas are designed to be always growing, if your code pattern is based on heavily reallocating memory, the arena growth may lead to performance issues, because each reallocation may trigger memory copy with new page creation. Consider pre-allocate some reasonable capacity for your data when working with arenas (including temp allocator). However, if you're reallocating the **exact** last pointer, the arena might do it in place on the same page.

#### Unit Test specific behavior
When run in test mode (or specifically `#ifdef CEX_TEST` is true) the memory allocation model in CEX includes some extra safety capabilities:

1. Heap based allocator (`mem$`) starts tracking memory leaks, comparing number of allocations and frees.
2. `mem$malloc()` - return uninitialized memory with `0x7f` byte pattern
3. If Address Sanitizer is available all allocations for arenas and heap will be surrounded by poisoned areas. If you see use-after-poison errors, it's likely a sign of use-after-free or out of bounds access in `tmem$`. Try to switch your code to the `mem$` allocator if possible to triage the exact reason of the error.
4. Allocators do sanity checks at the end of the each unit test case



## Advanced topics
### Performance
### When to use Arena or Heap allocators
### UnitTesting and Memory Leaks
### Out-of-bounds access / poisoning

## Code patterns
