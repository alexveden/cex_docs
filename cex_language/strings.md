---
title: "Strings"
---

## Problems with strings in C

Strings in C are historically endless source of problems, bugs and vulnerabilities. String manipulation in standard lib C is very low level and sometimes confusing. But in my opinion, the most of the problems with string in C is a result of poor code practices, rather than language issues itself.

With modern tooling like Address Sanitizer it's much easier to catch these bugs, so we are starting to face developer experience issues rather than security complications.

Problems with C `char*` strings:

* No length information included, which leads to performance issues with overuse of `strlen`
* Null terminator is critical for security, but not all libc functions handle strings securely
* String slicing is impossible without copy and setting null-terminator at the end of slice
* libc string functions behavior sometimes is implementation specific and insecure

## Strings in CEX

There are 3 key string manipulation routines in general:

1. General purpose string manipulation - uses vanilla `char*` type, with null-terminator, with dedicated `str` namespace. The main purpose is to make strings easy to work with, and keeping them C compatible. `str` namespace uses allocators for all memory allocating operations, which allows us to use temporary allocations with `tmem$`.
2. String slicing - sometimes we need to obtain and work with a part of existing string, so CEX use `str_s` type for defining slices. There is dedicated sub-namespace `str.slice` which is specially designed for working with slices. Slices may or may not be null-terminated, they carry pointer and length. Typically is a quick and non-allocating way of working of string view representation.
3. String builder - in the case if we need to build string dynamically we may use `sbuf_c` type and `sbuf` namespace in CEX. This type is dedicated for dynamically growing strings backed by allocator, that are always null-terminated and compatible with `char*` without casting.

Cex strings follow these principles:

* Security first - all strings are null-terminated, all buffer related operations always checking bounds.
* NULL-tolerant - all strings may accept NULL pointers and return NULL result on error. This significantly reduces count of `if(s == NULL)` error checks after each function, allowing to chain string operations and check `NULL` at the last step.
* Memory allocations are explicit - if string function accepts `IAllocator` this is indication of allocating behavior.
* Developer convenience - sometimes it's easier to allocate and make new formatted string on `tmem$` for example `str.fmt(_, "Hello: %s", "CEX")`, or use builtin pattern matching engine `str.match(arg[1], "command_*_(insert|delete|update))")`, or work with read-only slice representation of constant strings.


> [!TIP] 
>
> To get brief cheat sheet on functions list via Cex CLI type `./cex help str` or `./cex help sbuf`

## General purpose strings

Use `str` for general purpose string manipulation, this namespace typically returns `char*` or NULL on error, all function are tolerant to NULL arguments of `char*` type and re-return NULL in this case. Each allocating function must have `IAllocator` argument, also return NULL on memory errors.

```c
    char*           str.clone(char* s, IAllocator allc);
    Exception       str.copy(char* dest, char* src, usize destlen);
    bool            str.ends_with(char* s, char* suffix);
    bool            str.eq(char* a, char* b);
    bool            str.eqi(char* a, char* b);
    char*           str.find(char* haystack, char* needle);
    char*           str.findr(char* haystack, char* needle);
    char*           str.fmt(IAllocator allc, char* format,...);
    char*           str.join(char** str_arr, usize str_arr_len, char* join_by, IAllocator allc);
    usize           str.len(char* s);
    char*           str.lower(char* s, IAllocator allc);
    bool            str.match(char* s, char* pattern);
    int             str.qscmp(const void* a, const void* b);
    int             str.qscmpi(const void* a, const void* b);
    char*           str.replace(char* s, char* old_sub, char* new_sub, IAllocator allc);
    str_s           str.sbuf(char* s, usize length);
    arr$(char*)     str.split(char* s, char* split_by, IAllocator allc);
    arr$(char*)     str.split_lines(char* s, IAllocator allc);
    Exc             str.sprintf(char* dest, usize dest_len, char* format,...);
    str_s           str.sstr(char* ccharptr);
    bool            str.starts_with(char* s, char* prefix);
    str_s           str.sub(char* s, isize start, isize end);
    char*           str.upper(char* s, IAllocator allc);
    Exception       str.vsprintf(char* dest, usize dest_len, char* format, va_list va);

```

## String slices 

CEX has a special type and namespace for slices, which are dedicated struct of `(len, char*)` fields, which intended for working with parts of other strings, or can be a representation of a null-terminated string of full length.

### Creating string slices

```c
char* my_cstring = "Hello CEX";

// Getting a sub-string of a C string
str_s my_cstring_sub = str.sub(my_cstring, -3, 0); // Value: CEX, -3 means from end of my_cstring

// Created from any other null-terminated C string
str_s my_slice = str.sstr(my_cstring);

// Statically initialized slice with compile time known length
str_s compile_time_slice = str$s("Length of this slice created compile time"); 

// Making slice from a buffer (may not be null-terminated)
char buf[100] = {"foo bar"}; 
str_s my_slice_buf = str.sbuf(buf, arr$len(buf));

```

> [!NOTE]
> 
> `str_s` types are always passed by value, it's a 16-byte struct, which fits 2 CPU registers on x64


### Using slices

Once slice is created and you see `str_s` type, it's only safe to use special functions which work only with slices, because null-termination is not guaranteed anymore.

There are plenty of operations which can be made only on string view, without touching underlying string data.

```c

char*           str.slice.clone(str_s s, IAllocator allc);
Exception       str.slice.copy(char* dest, str_s src, usize destlen);
bool            str.slice.ends_with(str_s s, str_s suffix);
bool            str.slice.eq(str_s a, str_s b);
bool            str.slice.eqi(str_s a, str_s b);
isize           str.slice.index_of(str_s s, str_s needle);
str_s           str.slice.iter_split(str_s s, char* split_by, cex_iterator_s* iterator);
str_s           str.slice.lstrip(str_s s);
bool            str.slice.match(str_s s, char* pattern);
int             str.slice.qscmp(const void* a, const void* b);
int             str.slice.qscmpi(const void* a, const void* b);
str_s           str.slice.remove_prefix(str_s s, str_s prefix);
str_s           str.slice.remove_suffix(str_s s, str_s suffix);
str_s           str.slice.rstrip(str_s s);
bool            str.slice.starts_with(str_s s, str_s prefix);
str_s           str.slice.strip(str_s s);
str_s           str.slice.sub(str_s s, isize start, isize end);
```

> [!NOTE]
> 
> All Cex formatting functions (e.g. `io.printf()`, `str.fmt()`) support special format `%S` dedicated for string slices, allowing to work with slices naturally.

```c
char* my_cstring = "Hello CEX";
str_s my_slice = str.sstr(my_cstring);
str_s my_sub = str.slice.sub(my_slice, -3, 0);

io.printf("%S - Making Old C Cexy Again\n", my_sub);
io.printf("buf: %c %c %c len: %zu", my_sub.buf[0], my_sub.buf[1], my_sub.buf[2], my_sub.len);
```

### Error handling
On error all slice related routines return empty `(str_s){.buf = NULL, .len = 0}`, all routines check if `.buf == NULL` therefore it's safe to pass empty/error slice multiple times without need for checking errors after each call. This allows operations chaining like this:

```c
str_s my_sub = str.slice.sub(my_slice, -3, 0);
my_sub = str.slice.remove_prefix(my_sub, str$s("pref"));
my_sub = str.slice.strip(my_sub);
if (!my_sub.buf) {/* OOPS error */}
```

## String conversions

When working with strings, conversion from string into numerical types become very useful. Libc conversion functions are messy end error prone, CEX uses own implementation, with support for both `char*` and slices `str_s`. 

You may use one of the functions above or pick typesafe/generic macro `str$convert(str_or_slice, out_var_pointer)`

```c
Exception       str.convert.to_f32(char* s, f32* num);
Exception       str.convert.to_f32s(str_s s, f32* num);
Exception       str.convert.to_f64(char* s, f64* num);
Exception       str.convert.to_f64s(str_s s, f64* num);
Exception       str.convert.to_i16(char* s, i16* num);
Exception       str.convert.to_i16s(str_s s, i16* num);
Exception       str.convert.to_i32(char* s, i32* num);
Exception       str.convert.to_i32s(str_s s, i32* num);
Exception       str.convert.to_i64(char* s, i64* num);
Exception       str.convert.to_i64s(str_s s, i64* num);
Exception       str.convert.to_i8(char* s, i8* num);
Exception       str.convert.to_i8s(str_s s, i8* num);
Exception       str.convert.to_u16(char* s, u16* num);
Exception       str.convert.to_u16s(str_s s, u16* num);
Exception       str.convert.to_u32(char* s, u32* num);
Exception       str.convert.to_u32s(str_s s, u32* num);
Exception       str.convert.to_u64(char* s, u64* num);
Exception       str.convert.to_u64s(str_s s, u64* num);
Exception       str.convert.to_u8(char* s, u8* num);
Exception       str.convert.to_u8s(str_s s, u8* num);

```

For example:
```c
i32 num = 0;
s = "-2147483648";

// Both are equivalent
e$ret(str.convert.to_i32(s, &num));
e$ret(str$convert(s, &num));
```

## Dynamic strings / string builder

If you need to build string dynamically you can use `sbuf_c` type, which is simple alias for `char*`, but with special logic attached. This type implements dynamic growing / shrinking, and formatting of strings with null-terminator.


### Example
```c
sbuf_c s = sbuf.create(5, mem$); /* <1> */

char* cex = "CEX";
e$ret(sbuf.appendf(&s, "Hello %s", cex)); /* <2> */
e$assert(str.ends_with(s, "CEX")); /* <3> */

sbuf.destroy(&s);
```
1. Creates new dynamic string on heap, with 5 bytes initial capacity
2. Appends text to string with automatic resize (memory reallocation)
3. `s` variable of type `sbuf_c` is compatible with any `char*` routines, because it's an alias of `char*`

> [!TIP]
> 
> If you need one-shot format for string try to use `str.fmt(allocator, format, ...)` inside temporary allocator `mem$scope(tmem$, _)`

### `sbuf` namespace

```c
    Exception       sbuf.append(sbuf_c* self, char* s);
    Exception       sbuf.appendf(sbuf_c* self, char* format,...);
    Exception       sbuf.appendfva(sbuf_c* self, char* format, va_list va);
    u32             sbuf.capacity(sbuf_c* self);
    void            sbuf.clear(sbuf_c* self);
    sbuf_c          sbuf.create(u32 capacity, IAllocator allocator);
    sbuf_c          sbuf.create_static(char* buf, usize buf_size);
    sbuf_c          sbuf.create_temp(void);
    sbuf_c          sbuf.destroy(sbuf_c* self);
    Exception       sbuf.grow(sbuf_c* self, u32 new_capacity);
    bool            sbuf.isvalid(sbuf_c* self);
    u32             sbuf.len(sbuf_c* self);
    void            sbuf.shrink(sbuf_c* self, u32 new_length);
    void            sbuf.update_len(sbuf_c* self);
```

## String formatting in CEX
All CEX routines  with format strings (e.g. `io.printf()`/`log$error()`/`str.fmt()`) use special formatting function with extended features:

* `%S` format specifier is used for printing string slices of `str_s` type
* `%S` format has a sanity checks in the case if simple string is passed to its place, it will print `(%S-bad/overflow)` in the text. However, it's not guaranteed behavior, and depends on platform.
* `%lu`/`%ld` - formats are dedicated for printing 64-bit integers, they are not platform specific

