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

## Strings
There are several types of strings in CEX, each serves its own purpose. 

* `str` - is the most common, it is not a type but a namespace (collection of functions) which work with plain `char*` but in a safer way than vanilla libc. Inputs are NULL tolerant, and output may return NULL on errors.  
* `sbuf_c` - dynamic string builder container which supports append formatting and dynamic resizing. 
* `str_s` - read-only string view (slice) type, provides `buf+len` strings and has dedicated namespace `str.slice` for dealing with string slices. Strings slices may or may not be null-terminated. This is a goto type when you need dealing with substrings, without memory allocation. It has dedicated `printf` format `%S`.

> [!TIP] 
>
> To get brief cheat sheet on functions list via Cex CLI type `./cex help str` or `./cex help sbuf`

## Data Structures 
## Working with arrays
## Code Quality Tools
## Typical Project Structure
## Build system
## Project Tools

