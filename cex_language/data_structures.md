---
title: "Data structures and arrays"
---

## Data structures in CEX
There is a lack of support for data structures in C, typically it's up to developer to decide what to do. However, I noticed that many other C projects tend to reimplement over and over again two core data structures, which are used in 90% of cases: dynamic arrays and hashmaps.

Key requirements of the CEX data structures:

* Allocator based memory management - allowing you to decide memory model and tweak it anytime.
* Type safety and LSP support - each DS must have a specific type and support LSP suggestions.
* Generic types - DS must be generic.
* Seamless C compatibility - allowing accessing CEX DS as plain C arrays and pass them as pointers.
* Support any item type including overaligned.

## Dynamic arrays
Dynamic arrays (a.k.a vectors or lists) are designed specifically for developer convenience and based on ideas of Sean Barrett's STB DS.

### What is dynamic array in CEX
Technically speaking it's a simple C pointer `T*`, where `T` is any generic type. The memory for that pointer is allocated by allocator, and its length is stored at some byte offset before the address of the dynamic array head.

With this type representation we can get some useful benefits:

* Array access with simple indexing, i.e. `arr[i]` instead of `dynamic_arr_get_at(arr, i)`
* Passing by pointer into vanilla C code. For example, a function signature `my_func(int* arr, usize arr_len)`  is compatible with `arr$(int*)`, so we can call it as `my_func(arr, arr$len(arr))`
* Passing length information integrated into single pointer, `arr$len(arr)` extracts length from dynamic array pointer
* Type safety out of the box and full LSP support without dealing with `void*`


### `arr$` namespace
`arr$` is completely macro-driven namespace, with generic type support and safety checks.

`arr$` API:
 
| Macro | Description |
| -------------- | --------------- |
| arr\$(T) | Macro type definition, just for indication that it's a dynamic array |
| arr\$new(arr, allocator, kwargs...) | Initialization of the new instance of dynamic array | 
| arr\$free(arr) | Dynamic array cleanup (if HeapAllocator was used) |
| arr\$clear(arr) | Clearing dynamic array contents |
| arr\$push(arr, item) | Adding new item to the end of array |
| arr\$pushm(arr, item, item1, itemN) | Adding many new items to the end of array |
| arr\$pusha(arr, other_arr, \[other_arr_len\]) | Adding many new item to the end of array |
| arr\$pop(arr) | Returns last element and removes it |
| arr\$at(arr, i) | Returns element at index with boundary checks for i |
| arr\$last(arr) | Returns last element |
| arr\$del(arr, i) | Removes element at index (following data is moved at the i-th position) |
| arr\$delswap(arr, i) | Removes element at index, the removed element is replaced by last one |
| arr\$ins(arr, i, value) | Inserts element at index |
| arr\$grow_check(arr, add_len) | Grows array by `add_len` if needed |
| arr\$sort(arr, qsort_cmp) | Sorting array with `qsort` function |


### Examples

::: {.panel-tabset}
#### Initialization
```c
arr$(int) arr = arr$new(arr, mem$); /* <1> */
arr$push(arr, 1);  /* <2> */
arr$pushm(arr, 2, 3, 4); /* <3> */
int static_arr[] = { 5, 6 };
arr$pusha(arr, static_arr /*, array_len (optional) */); /* <4> */

io.printf("arr[0]=%d\n", arr[0]); // prints arr[0]=1

// Iterate over array: prints lines 1 ... 6
for$each (v, arr) { /* <5> */
    io.printf("%d\n", v); 
}

arr$free(arr); /* <6> */
```

1. Initialization and allocator 
2. Adding single element
3. Adding multiple elements via vargs.
4. Adding arbitrary array, supports static arrays, dynamic CEX arrays or `int*`+arr_len 
5. Array iteration via `for$each` is common and compatible with all arrays in Cex (dynamic, static, pointer+len)
6. Deallocating memory (only needed when HeapAllocator is used)

#### Overaligned structs
```c

test$case(test_overaligned_struct)
{
    struct test32_s
    {
        alignas(32) usize s;
    };

    arr$(struct test32_s) arr = arr$new(arr, mem$);
    struct test32_s f = { .s = 100 };
    tassert(mem$aligned_pointer(arr, 32) == arr);

    for (u32 i = 0; i < 1000; i++) {
        f.s = i;
        arr$push(arr, f);

        tassert_eq(arr$len(arr), i + 1);
    }
    tassert_eq(arr$len(arr), 1000);

    for (u32 i = 0; i < 1000; i++) {
        tassert_eq(arr[i].s, i);
        tassert(mem$aligned_pointer(&arr[i], 32) == &arr[i]);
    }

    arr$free(arr);
    return EOK;
}
```

#### Array of strings

```c
test$case(test_array_char_ptr)
{
    arr$(char*) array = arr$new(array, mem$);
    arr$push(array, "foo");
    arr$push(array, "bar");
    arr$pushm(array, "baz", "CEX", "is", "cool");
    for (usize i = 0; i < arr$len(array); ++i) { io.printf("%s \n", array[i]); }
    arr$free(array);

    return EOK;
}

```

#### Array inside `tmem$`
```c
mem$scope(tmem$, _) /* <1> */
{
    arr$(char*) incl_path = arr$new(incl_path, _, .capacity = 128); /* <2> */
    for$each (p, alt_include_path) {
        arr$push(incl_path, p);  /* <3> */
        if (!os.path.exists(p)) { log$warn("alt_include_path not exists: %s\n", p); }
    }
} /* <4> */
```
1. Initializes a temporary allocator (`tmem$`) scope in `mem$scope(tmem$, _) {...}` and assigns it as a variable `_` (you can use any name).
2. Initializes dynamic array with the scoped allocator variable `_`, allocates with specific capacity argument.
3. May allocate memory
4. All memory will be freed at exit from this scope


:::

## Hashmaps

Hashmaps (`hm$`) in CEX are backed by structs with `key` and `value` fields, essentially they are backed by plain dynamic arrays of structs (iterable values) with hash table part for implementing keys hashing.

Hashmaps in CEX are also generic, you may use any type of keys or values. However, there are special handling for string keys (`char*`, or `str_s` CEX slices). Typically string keys are not copied by hashmap by default, and stored by reference, so you'll have to keep their allocation stable.

Hashmap initialization is similar to the dynamic arrays, you should define type and call `hm$new`.

### Array compatibility
Hashmaps in CEX are backed by dynamic arrays, which leads to the following developer experience enhancements:

* `arr$len` can be applied to hashmaps for checking number of available elements
* `for$each/for$eachp` can be used for iteration over hashmap key/values pairs
* Hashmap items can be accessed as arrays with index

### Initialization
There are several ways for declaring hashmap types:

1. Local function hashmap variables
```c
    hm$(char*, int) intmap = hm$new(intmap, mem$);
    hm$(const char*, int) ap = hm$new(map, mem$);
    hm$(struct my_struct, int) map = hm$new(map, mem$);

```

2. Global hashmaps with special types
```c

// NOTE: struct must have .key and .value fields
typedef struct
{
    int key;
    float my_val;
    char* my_string;
    int value;
} my_hm_struct;

void foo(void) {
    // NOTE: this is equivalent of my_hm_struct* map = ...
    hm$s(my_hm_struct) map = hm$new(map, mem$);
}

void my_func(hm$s(my_hm_struct)* map) {
    // NOTE: passing hashmap type, parameter
    int v = hm$get(*map, 1);

    // NOTE: hm$set() may resize map, because of this we use `* map` argument, for keeping pointer valid!
    hm$set(*map, 3, 4);

    // Setting entire structure
    hm$sets(*map, (my_hm_struct){ .key = 5, .my_val = 3.14, .my_string = "cexy", .value = 98 }));
}

```

3. Declaring hashmap as type

```c
typedef hm$(char*, int) MyHashMap;

struct my_hm_struct {
    MyHashmap hm;
};

void foo(void) {
    // Initialing  new variable
    MyHashMap map = hm$new(map, mem$);
    
    // Initialing hashmap as a member of struct
    struct my_hm_struct hs = {0};
    hm$new(hs.hm, mem$);

}
```



### Hashmap API
| Macro | Description |
| --------------- | --------------- | --------------- |
| hm\$new(hm, allocator, kwargs...) |  Initialization of hashmap |
| hm\$set(hm, key, value) | Set element |
| hm\$setp(hm, key, value) | Set element and return pointed to the newly added item inside hashmap |
| hm\$sets(hm, struc_value...) | Set entire element as backing struct |
| hm\$get(hm, key) | Get a value by key (as a copy) |
| hm\$getp(hm, key) | Get a value by key as a pointer to hashmap value |
| hm\$gets(hm, key) | Get a value by key as a pointer to a backing struct |
| hm\$clear(hm) | Clears contents of hashmap |
| hm\$del(hm, key) | Delete element by key |
| hm\$len(hm) | Number of elements in hashmap / `arr$len()` also works |

### Initialization params
`hm$new` accepts optional params which may help you to adjust hashmap key behavior:

* `.capacity=16` - initial capacity of the hashmap, will be rounded to closest power of 2 number
* `.seed=` - initial seed for hashing algorithm
* `.copy_keys=false` - enabling copy of `char*` keys and storing them specifically in hashmap
* `.copy_keys_arena_pgsize=0` - enabling using arena for `copy_keys` mode

Example:
```c
test$case(test_hashmap_string_copy_arena)
{
    hm$(char*, int) smap = hm$new(smap, mem$, .copy_keys = true, .copy_keys_arena_pgsize = 1024);

    char key2[10] = "foo";

    hm$set(smap, key2, 3);
    tassert_eq(hm$len(smap), 1);
    tassert_eq(hm$get(smap, "foo"), 3);
    tassert_eq(hm$get(smap, key2), 3);
    tassert_eq(smap[0].key, "foo");

    // Initial buffer gets destroyed, but hashmap keys remain the same
    memset(key2, 0, sizeof(key2));

    tassert_eq(smap[0].key, "foo");
    tassert_eq(hm$get(smap, "foo"), 3);

    hm$free(smap);
    return EOK;
}

```

### Examples
::: {.panel-tabset}
#### Hashmap basic use
```c
hm$(char*, int) smap = hm$new(smap, mem$);
hm$set(smap, "foo", 3);
hm$get(smap, "foo");
hm$len(smap);
hm$del(smap, "foo");
hm$free(smap);

```

#### String key hashmap
```c
test$case(test_hashmap_string)
{
    char key_buf[10] = "foobar";

    hm$(char*, int) smap = hm$new(smap, mem$);

    char* k = "foo";
    char* k2 = "baz";

    char key_buf2[10] = "foo";
    char* k3 = key_buf2;
    hm$set(smap, "foo", 3);

    tassert_eq(hm$len(smap), 1);
    tassert_eq(hm$get(smap, "foo"), 3);
    tassert_eq(hm$get(smap, k), 3);
    tassert_eq(hm$get(smap, key_buf2), 3);
    tassert_eq(hm$get(smap, k3), 3);

    tassert_eq(hm$get(smap, "bar"), 0);
    tassert_eq(hm$get(smap, k2), 0);
    tassert_eq(hm$get(smap, key_buf), 0);

    tassert_eq(hm$del(smap, key_buf2), 1);
    tassert_eq(hm$len(smap), 0);

    hm$free(smap);
    return EOK;
}

```

#### Iterating elements
```c
test$case(test_hashmap_basic_iteration)
{
    hm$(int, int) intmap = hm$new(intmap, mem$);
    hm$set(intmap, 1, 10);
    hm$set(intmap, 2, 20);
    hm$set(intmap, 3, 30);

    tassert_eq(hm$len(intmap), 3);  // special len
    tassert_eq(arr$len(intmap), 3); // NOTE: arr$len is compatible

    // Iterating by value (data is copied)
    u32 nit = 1;
    for$each (it, intmap) {
        tassert_eq(it.key, nit);
        tassert_eq(it.value, nit * 10);
        nit++;
    }

    // Iterating by pointers (data by reference)
    for$eachp(it, intmap)
    {
        isize _nit = intmap - it; // deriving index from pointers
        tassert_eq(it->key, _nit);
        tassert_eq(it->value, _nit * 10);
    }

    hm$free(intmap);

    return EOK;
}

```

:::

## Working with arrays

Arrays are probably most used concept in any language, with C arrays may have many different forms. Unfortunately, the main problem of working with arrays in C is a specialization of methods and operations, each type of array may require special iteration macro, or function for getting array length or element.

Collection types in C:

* Static arrays `i32 arr[10]`
* Dynamic arrays as pointers `(i32* arr, usize arr_len)`
* Custom dynamic arrays `dynamic_array_push_back(&int_array, &i);`
* Char buffers `char buf[1024]`
* Null-terminated strings and slices
* Hashmaps

Cex tries to solve this by unification of all arrays operations around standard design principles, without getting too far away from standard C.

### `arr$len` unified length

`arr$len(array)` macro is a ultimate tool for getting lengths of arrays in CEX. It supports: static arrays, char buffers, string literals, dynamic arrays of CEX `arr$` and hashmaps of CEX `hm$`. Also it's a NULL resilient macro, which returns 0 if `array` argument is NULL.

> [!NOTE]
> 
> Not all array pointers are supports by `arr$len` (only dynamic arrays or hashmaps are valid), however in debug mode `arr$len` will raise an assertion/ASAN crash if you passed wrong pointer type there.

Example:
```c
test$case(test_array_len)
{
    arr$(int) array = arr$new(array, mem$);
    arr$pushm(array, 1, 2, 3);

    // Works with CEX dynamic arrays
    tassert_eq(arr$len(array), 3);

    // NULL is supported, and emits 0 length
    arr$free(array);
    tassert(array == NULL); 
    tassert_eq(arr$len(array), 0); // NOTE: NULL array - len = 0

    // Works with static arrays
    char buf[] = {"hello"}; 
    tassert_eq(arr$len(buf), 6); // NOTE: includes null term

    // Works with arrays of given capacity
    char buf2[10] = {0};
    tassert_eq(arr$len(buf2), 10);

    // Type doesn't matter
    i32 a[7] = {0};
    tassert_eq(arr$len(a), 7);

    // Works with string literals
    tassert_eq(arr$len("CEX"), 4); // NOTE: includes null term

    // Works with CEX hashmap
    hm$(int, int) intmap = hm$new(intmap, mem$);
    hm$set(intmap, 1, 3);
    tassert_eq(arr$len(intmap), 1);

    hm$free(intmap);

    return EOK;
}

```

### Accessing elements of array is unified

```c
test$case(test_array_access)
{
    arr$(int) array = arr$new(array, mem$);
    arr$pushm(array, 1, 2, 3);

    // Dynamic array access is natural C index
    tassert_eq(array[2], 3);
    // tassert_eq(arr$at(array, 3), 3); // NOTE: this is bounds checking access, with assertion 
    arr$free(array);

    // Works with static arrays
    char buf[] = {"hello"}; 
    tassert_eq(buf[1], 'e'); 

    // Works with CEX hashmap
    hm$(int, int) intmap = hm$new(intmap, mem$);
    hm$set(intmap, 1, 3);
    hm$set(intmap, 2, 5);
    tassert_eq(arr$len(intmap), 2);

    // Accessing hashmap as array
    // NOTE: hashmap elements are ordered until first deletion
    tassert_eq(intmap[0].key, 1);
    tassert_eq(intmap[0].value, 3);

    tassert_eq(intmap[1].key, 2);
    tassert_eq(intmap[1].value, 5);

    hm$free(intmap);

    return EOK;
}
```

### CEX way of iteration over arrays

CEX introduces an unified `for$*` macros which helps with dealing with looping, these are typical patters for iteration:

* `for$each(it, array, [array_len])` - iterates over array, `it` represents value of array item. `array_len` is optional and uses `arr$len(array)` by default, or you might explicitly set it for iterating over arbitrary C pointer+len arrays.
* `for$eachp(it, array, [array_len])` - iterates over array, `it` represent a pointer to array item. `array_len` is inferred by default.
* `for$iter(it_val_type, it, iter_funct)` - a special iterator for non-indexable collections or function based iteration, tailored for customized iteration of unknown length.
* `for(usize i = 0; i < arr$len(array); i++)` - classic also works :)

```c
test$case(test_array_iteration)
{
    arr$(int) array = arr$new(array, mem$);
    arr$pushm(array, 1, 2, 3);

    i32 nit = 0; // it's only for testing
    for$each(it, array) {
        tassert_eq(it, ++nit);
        io.printf("el=%d\n", it);
    }
    // Prints: 
    // el=1
    // el=2
    // el=3

    nit = 0;
    // NOTE: prefer this when you work with bigger structs to avoid extra memory copying
    for$eachp(it, array) {
        // TIP: making array index out of `it`
        usize i = it - array;
        tassert_eq(i, nit);

        // NOTE: it now is a pointer
        tassert_eq(*it, ++nit);
        io.printf("el[%zu]=%d\n", i, *it);
    }
    // Prints: 
    // el[0]=1
    // el[1]=2
    // el[2]=3

    // Static arrays work as well (arr$len inferred)
    i32 arr_int[] = {1, 2, 3, 4, 5};
    for$each(it, arr_int) {
        io.printf("static=%d\n", it);
    }
    // Prints:
    // static=1
    // static=2
    // static=3
    // static=4
    // static=5


    // Simple pointer+length also works (let's do a slice)
    i32* slice = &arr_int[2];
    for$each(it, slice, 2) {
        io.printf("slice=%d\n", it);
    }
    // Prints:
    // slice=3
    // slice=4

    arr$free(array);
    return EOK;
}
```

