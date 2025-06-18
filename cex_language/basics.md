---
title: "Basics"
---

## Code Style Guidelines

* `dollar$means_macros`. CEX style uses `$` delimiter as a macro marker, if you see it anywhere in the code this means you are dealing with some sort of macro. `first$` part of name usually linked to a namespace of a macro, so you may expect other macros, type names or functions with that prefix.

* `functions_are_snake_case()`. Lower case for functions

* `MyStruct_c` or `my_struct_s`. Struct types typically expected to be a `PascalCase` with suffix, `_c` suffix indicates there is a code namespace with the same name (i.e. `_c` hints it's a container or kind of object), `_s` suffix means simple data-container without special logic.

* `MyObj.method()` or `namespace.func()`. Namespace names typically lower case, and object specific namespace names reflect type name, e.g. `MyObj_c` has `MyObj.method()`.

* `Enums__double_underscore`. Enum types are defined as `MyEnum_e` and each element looks like `MyEnum__foo`, `MyEnum__bar`.

* `CONSTANTS_ARE_UPPER`. Two notations of constants: `UPPER_CASE_CONST` or `namespace$CONST_NAME`



## Types
CEX provides several short aliases for primitive types and some extra types for covering blank spots in C.

| Type | Description |
| -------------- | --------------- |
| var | automatically inferred variable type |
| bool | boolean type |
| u8/i8 | 8-bit integer |
| u16/i16 | 16-bit integer |
| u32/i32 | 32-bit integer |
| u64/i64 | 64-bit integer |
| f32 | 32-bit floating point number (float) |
| f64 | 64-bit floating point number (double) |
| usize | maximum array size (size_t) |
| isize | signed array size (ptrdiff_t) |
| char* | core type for null-term strings |
| sbuf_c | dynamic string builder type |
| str_s | string slice (buf + len) |
| Exc / Exception | error type in CEX |
| Error.\<some\> | generic error collection |
| IAllocator | memory allocator interface type |
| arr\$(T) | generic type dynamic array |
| hm\$(T) | generic type hashmap|

## Error handling
CEX has a special `Exception` type which is essentially alias for `char*`, and yes all error handling in CEX is based on `char*`. Before you start laughing and rolling on the floor, let me explain the most important part of the `Exception` type, this little `*` part. Exception in CEX is a **pointer** (an address, a number) to a some arbitrary char array on memory.

What if the returned pointer could be always some constant area indicating an error? With that rule, we don't have to match error (string) content, but we can compare only address of the error.

```c

#define CEX_IMPLEMENTATION
#include "cex.h"

const struct _MyCustomError
{
    Exc why_arg_is_one;
} MyError = { .why_arg_is_one = "WhyArgIsOneError" };

Exception
baz(int argc)
{
    if (argc == 1) { return e$raise(MyError.why_arg_is_one, "Why argc is 1, argc = %d?", argc); }
    return EOK;
}

int
main(int argc, char** argv)
{
    e$except (err, baz(argc)) { 
        // NOTE: comparing the address, but not a content!
        if (err == MyError.why_arg_is_one) {
            io.printf("We need moar args!\n");
        }
        return 1; 
    }
    return 0;
}
```

Produces traceback errors:

```sh
[ERROR]   ( main.c:12 baz() ) [WhyArgIsOneError] Why argc is 1, argc = 1?
[^STCK]   ( main.c:35 main() ) ^^^^^ [WhyArgIsOneError] in function call `foo2(argc)`
We need moar args!
```

Check [Error handling ](errors.md) section for more details about error implementation .

## Memory management
CEX tries to adopt allocator-centric approach to memory management, which help to follow those principles:

* **Explicit memory allocation.** Each object (class) or function that may allocate memory has to have an allocator parameter. This requirement, adds explicit API signature hints, and communicates about memory implications of a function without deep dive into documentation or source code.
* **Transparent memory management.** All memory operations are provided by `IAllocator` interface, which can be interchangeable allocator object of different type.
* **Memory scoping**. When possible memory usage should be limited by scope, which naturally regulates lifetimes of allocated memory and automatically free it after exiting scope.
* **UnitTest Friendly**. Allocators allowing implementation of additional levels of memory safety when run in unit test environment. For example, CEX allocators add special poisoned areas around allocated blocks, which trigger address sanitizer when this region accesses with user code. Allocators open door for a memory leak checks, or extra memory error simulations for better out-of-memory error handling.
* **Standard and Temporary allocators**. Sometimes it's useful to have initialized allocator under your belt for short-lived temporary operations. CEX provides two global allocators by default: `mem$` - is a standard heap allocator using `malloc/realloc/free`, and `tmem$` - is dynamic arena allocator of small size (about 256k of per page).  

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

Check [Memory management](memory.md) section for more details about memory handling.


## Strings
There are several types of strings in CEX, each serves its own purpose. 

* `str` - is the most common, it is not a type but a namespace (collection of functions) which work with plain `char*`, but in a safer way than vanilla libc. Inputs are NULL tolerant, and output may return NULL on errors.  
* `sbuf_c` - dynamic string builder container which supports append formatting and dynamic resizing. It's fully `char*` compatible null-terminated string type.
* `str_s` - read-only string view (slice) type, provides `buf+len` strings and has dedicated namespace `str.slice` for dealing with string slices. Strings slices may or may not be null-terminated. This is a goto type when you need dealing with substrings, without memory allocation. It has dedicated `printf` format `%S`.

> [!TIP] 
>
> To get brief cheat sheet on functions list via Cex CLI type `./cex help str` or `./cex help sbuf`

Check [Strings](strings.md) section for more details.

## Data Structures 
There is a lack of support for data structures in C, typically it's up to developer to decide what to do. However, I noticed that many other C projects tend to reimplement over and over again two core data structures, which are used in 90% of cases: dynamic arrays and hashmaps.

Key features of the CEX data structures:

* Allocator based memory management - allowing you to decide memory model and tweak it anytime.
* Type safety and LSP support - each DS must have a specific type and support LSP suggestions.
* Generic types - data structures must be universal.
* Seamless C compatibility - allowing accessing CEX DS as plain C arrays and pass them as pointers.
* Support of any item type.

See more information about [data structures and arrays in CEX](data_structures.md)

## Code Quality Tools
## Typical Project Structure
## Build system
## Project Tools

