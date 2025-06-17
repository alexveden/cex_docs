
---
title: "Error handling"
---

## The problem of error handling in C

C errors always were a mess due to historical reasons and because of ABI specifics. The main curse of C error is mixing values with errors, for example system specific calls return `-1` and set `errno` variable. Some return 0 on error, some NULL, sometimes is an enum, or `MAP_FAILED (which is (void*)-1)`. 

This convention on errors drains a lot of developer energy making him to keep searching docs and figuring out which return values of a function considered errors.

C error handling makes code cluttered with endless `if (ret_code == -1)`pattern.


The code below is a typical error handling pattern in C, however it's illustration for a specific issues:
```c
isize read_file(char* filename, char* buf, usize buf_size) {
    if (buff == NULL || filename == NULL) {
        errno = EINVAL;       /* <1> */
        return -1;
    }

    int fd = open(filename, O_RDONLY);
    if (fd == -1) {
        fprintf(stderr, "Cannot open '%s': %s\n", filename, strerror(errno));  /* <2> */
        return -1;
    }
    isize bytes_read = read(fd, buf, buf_size);
    if (bytes_read == -1) {
        perror("Error reading"); /* <3> */
        return -1;
    }
    return bytes_read; /* <4> */
}
```
1. `errno` is set, but it hard to distinguish by which API call or function argument is failed. 
2. Error message line is located not at the same place as it was reported, so the developer must go through code to check.
3. `errno` is too broad and ambiguous for describing exact reason of failure.
4. `foo` return value is mixing error `-1` and legitimate value of `bytes_read`. The situation gets worse if we need to use non integer return type of a function.

## CEX Error handling goals
CEX made an attempt to re-think general purpose error handling in applications, with the following goals:

* **Errors should be unambiguous** - detaching errors from valid result of a function, there are only 2 states: OK or an error.
* **Error handling should be general purpose** - providing generic code patterns for error handling
* **Error should be easy to report** - avoiding error code to string mapping
* **Error should be bubbling up** - code can pass the same error to the upper caller
* **Error should extendable** - allowing unique error identification
* **Error should be passed as values** - low overhead, error handling
* **Error handling should be natural** - no special constructs required to handle error in C code
* **Error should be forced to check** - no occasional error check skips

## How error handling is implemented
CEX has a special `Exception` type which is essentially alias for `char*`, and yes all error handling in CEX is based on `char*`. Before you start laughing and rolling on the floor, let me explain the most important part of the `Exception` type, this little `*` part. Exception in CEX is a **pointer** (an address, a number) to a some arbitrary char array on memory.

What if the returned pointer could be always some constant area indicating an error? With that rule, we don't have to match error (string) content, but we can compare only address of the error.

### CEX Error in a nutshell
```c
// NOTE: excerpt from cex.h

/// Generic CEX error is a char*, where NULL means success(no error)
typedef char* Exc;

/// Equivalent of Error.ok, execution success
#define EOK (Exc) NULL

/// Use `Exception` in function signatures, to force developer to check return value
/// of the function.
#define Exception Exc __attribute__((warn_unused_result))


/**
 * @brief Generic errors list, used as constant pointers, errors must be checked as
 * pointer comparison, not as strcmp() !!!
 */
extern const struct _CEX_Error_struct
{
    Exc ok; // Success no error
    Exc argument;
    // ... cut ....
    Exc os;
    Exc integrity;
} Error;


// NOTE: user code

Exception
remove_file(char* path)
{
    if (path == NULL || path[0] == '\0') { 
        return Error.argument;  // Empty of null file
    }
    if (!os.path.exists(path)) {
        return "Not exists" // literal error are allowed, but must be handled as strcmp()
    }
    if (str.eq(path, "magic.file")) {
        // Returns an Error.integrity and logs error at current line to stdout
        return e$raise(Error.integrity, "Removing magic file is not allowed!");
    }
    if (remove(path) < 0) { 
        return strerror(errno); // using system error text (arbitrary!)
    }
    return EOK;
}

Exception
main(char* path)
{
    // Method 1: low level handling (no re-throw)
    if (remove_file(path)) { return Error.os; }
    if (remove_file(path) != EOK) { return "bad stuff"; }
    if (remove_file(path) != Error.ok) { return EOK; }

    // Method 2: handling specific errors
    Exc err = remove_file(path);
    if (err == Error.argument) { // <<< NOTE: comparing address not a string contents!
        io.printf("Some weird things happened with path: %s, error: %s\n", path, err);
        return err;
    }
    // Method 3: helper macros + handling with traceback
    e$except(err, remove_file(path)) { // NOTE: this call automatically prints a traceback
        if (err == Error.integrity) { /* TODO: do special case handling */  }
    }

    // Method 4: helper macros + handling unhandled
    e$ret(remove_file(path)); // NOTE: on error, prints traceback and returns error to the caller

    remove_file(path);  // <<< OOPS compiler error, return value of this function unchecked

    return 0;
}


```

### Error tracebacks and custom errors in CEX
CEX error system was designed to help in debugging, this is a simple example of deep call stack printing in CEX.

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

Exception
bar(int argc)
{
    e$ret(baz(argc));
    return EOK;
}

Exception
foo2(int argc)
{
    io.printf("MOCCA - Make Old C Cexy Again!\n");
    e$ret(bar(argc));
    return EOK;
}

int
main(int argc, char** argv)
{
    (void)argv;
    e$except (err, foo2(argc)) { 
        if (err == MyError.why_arg_is_one) {
            io.printf("We need moar args!\n");
        }
        return 1; 
    }
    return 0;
}
```

```sh
MOCCA - Make Old C Cexy Again!
[ERROR]   ( main.c:12 baz() ) [WhyArgIsOneError] Why argc is 1, argc = 1?
[^STCK]   ( main.c:19 bar() ) ^^^^^ [WhyArgIsOneError] in function call `baz(argc)`
[^STCK]   ( main.c:27 foo2() ) ^^^^^ [WhyArgIsOneError] in function call `bar(argc)`
[^STCK]   ( main.c:35 main() ) ^^^^^ [WhyArgIsOneError] in function call `foo2(argc)`
We need moar args!
```

## Rewriting initial C example to CEX

Main benefits of using CEX error handling system:

1. Error messages come with `source_file.c:line` and `function()` for easier to debugging
2. Easier to do quick checks with `e$assert`
3. Easier to re-throw generic unhandled errors inside code
4. Unambiguous return values: OK or error.
5. Unlimited variants of returning different types of errors (`Error.argument`, `"literals"`, `strerror(errno)`, `MyCustom.error`)
6. Easy to log - Exceptions are just `char*` strings
7. Traceback support when chained via multiple functions

::: {.panel-tabset}
### CEX 
```c
Exception read_file(char* filename, char* buf, isize* out_buf_size) {
    e$assert(buff != NULL);  /* <1> */
    e$assert(filename != NULL && "invalid filename");

    int fd = 0;
    e$except_errno(fd = open(filename, O_RDONLY)) { return Error.os; } /* <2> */
    e$except_errno(*out_buf_size = read(fd, buf, *out_buf_size)) { return Error.io; } /* <3> */
    return EOK; /* <4> */
}
```
1. Returns error with printing out internal expression: `[ASSERT]  ( main.c:26 read_file() ) buff != NULL`. `e$assert` is an Exception returning assert, it doesn't abort your program, and these asserts are not stripped in release builds.
2. Handles typical `-1 + errno` check with print: `[ERROR]   ( main.c:27 read_file() ) fd = open("foo.txt", O_RDONLY) failed errno: 2, msg: No such file or directory`
3. Result of a function returned by reference to the `out` parameter.
4. Unambiguous return code for success.


### C
```c
isize read_file(char* filename, char* buf, usize buf_size) {
    if (buff == NULL || filename == NULL) {
        errno = EINVAL;
        return -1;
    }

    int fd = open(filename, O_RDONLY);
    if (fd == -1) {
        fprintf(stderr, "Cannot open '%s': %s\n", filename, strerror(errno));
        return -1;
    }
    isize bytes_read = read(fd, buf, buf_size);
    if (bytes_read == -1) {
        perror("Error reading");
        return -1;
    }
    return bytes_read;
}
```
:::

## Helper macros `e$...`
CEX has a toolbox of macros with `e$` prefix, which are dedicated to the `Exception` specific tasks. However, it's not mandatory to use them, and you can stick to regular control flow constructs from C.

In general, `e$` macros provide location logging (source file, line, function), which is a building block for error traceback mechanism in CEX.

`e$` macros mostly designed to work with functions that return `Exception` type.

### Returning the `Exc[eption]`
Errors in CEX are just plain string pointers. If the `Exception` function returns `NULL` or `EOK` or `Error.ok` this is indication of successful execution, otherwise any other value is an error.

Also you may return with `e$raise(error_to_return, format, ...)` macro, which prints location of the error in the code with message formatting.

```c
Exception error_sample1(int a) {
    if (a == 0) return Error.argument; // standard set of errors in CEX
    if (a == -1) return "Negative one";   // error literal also works, but harder to handle
    if (a == -2) return UserError.neg_two; // user error
    if (a == 7) return e$raise(Error.argument, "Bad a=%d", a); // error with logging
    
    return EOK; // success
    // return Error.ok; // success
    // return NULL; // success
}
```

### Handling errors
Error handling in CEX supports two ways:

* Silent handling - which suppresses error location logging, this might be useful for performance critical code, or tight loops. Also, this is a general way of returning errors for CEX standard lib.
* Loud handling with logging - this way is useful for one shot complex functions which may return multiple types of errors for different reasons. This is the way if you wanted to incorporate tracebacks for your errors.

#### Silent handling example
> [!NOTE]
> 
> Avoid using e$raise() in called functions if you need silent error handling, use plain `return Error.*`

```c
Exception foo_silent(void) {
    // Method 1: quick and dirty checks
    if (error_sample1(0)) { return "Error"; /* Silent handling without logic */ }
    if (error_sample1(0)) { /* Discarding error of a call */ }

    // Method 2: silent error condition
    Exc err = error_sample1(0);
    if (err) {
        if (err == Error.argument) {
            /* Handling specific error here */
        }
        return err;
    }

    // Method 3: silent macro, with temp error value
    e$except_silent(err, error_sample1(0)) {
        // NOTE: nesting is allowed!
        e$except_silent(err, error_sample1(-2)) {
            return err; // err = UserError.neg_two
        }

        // err = Error.argument now
        if (err == Error.argument) {
            /* Handling specific error here */
        }
        // break; // BAD! See caveats section below
    }
    return EOK;
}

```

> [!NOTE]
> 
> `e$except_silent` will print error log when code runs under unit test or inside CEX build system, this helps a lot with debugging.

#### Loud handling with logging

If you write some general purpose code with debugability in mind, the logged error handling can be a breeze. It allows traceback error logging, therefore deep stack errors now easier to track and reason about.

There are special error handling macros for this purpose:

1. `e$except(err, func_call()) { ... }` - error handling scope which initialize temporary variable `err` and logs if there was an error returned by `func_call()`. `func_call()` must return `Exception` type for this macro.
2. `e$except_errno(sys_func()) { ... }` - error handling for system functions, returning `-1` and setting `errno`.
3. `e$except_null(ptr_func()) { ... }` - error handling for `NULL` on error functions.
4. `e$except_true(func()) { ... }` - error handling for functions returning non-zero code on error.
5. `e$ret(func_call());` - runs the `Exception` type returning function `func_call()`,  and on error it logs the traceback and re-return the same return value. This is a main code shortcut and driver for all CEX tracebacks. Use it if you don't care about precise error handling and fine to return immediately on error.
6. `e$goto(func_call(), goto_err_label);` - runs the `Exception` type function, and does `goto goto_err_label;`. This macro is useful for resource deallocation logic, and intended to use for typical C error handling pattern `goto fail`.
7. `e$assert(condition)` or `e$assert(condition && "What's wrong")` or `e$assertf(condition, format, ...)`  - quick condition checking inside `Exception` functions, logs a error location + returns `Error.assert`. These asserts remain in release builds and do not affected by `NDEBUG` flag. 

```c
Exception foo_loud(int a) {
    e$assert(a != 0);
    e$assert(a != 11 && "a is suspicious");
    e$assertf(a != 22, "a=%d is something bad", a);

    char* m = malloc(20);
    e$assert(m != NULL && "memory error"); // ever green assert

    e$ret(error_sample1(9)); // Re-return on error

    e$goto(error_sample1(0), fail); // goto fail and free the resource

    e$except(err, error_sample1(0)) {
        // NOTE: nesting is allowed!
        e$except(err, error_sample1(-2)) {
            return err; // err = UserError.neg_two
        }
        // err = Error.argument now
        if (err == Error.argument) {
            /* Handling specific error here */
        }

        // continue; // BAD! See caveats section below
    }

    // For these e$except_*() macros you can use assignment expression
    // e$except_errno(fd = open(..))
    // e$except_null(f = malloc(..))
    // e$except_true (sqlite3_open(db_path, &db))
    FILE* f;
    e$except_null(f = fopen("foo.txt", "r")) {
        return Error.io;
    }

    return EOK;
    
fail:
    free(m);
    return Error.runtime;
}

```

### Caveats

Most of `e$excep_*` macros are backed by `for()` loop, so you have to be careful when you nest them inside outer loops and try to `break`/`continue` outer loop on error. 

In my opinion using `e$except_` inside loops is generally bad idea, and you should consider:

1. Factoring error emitting code into a separate function
2. Using `if(error_sample(i))` instead of `e$except`

#### Bad example!

```c
Exception foo_err_loop(int a) {
    for (int i = 0; i < 10; i++) {
        e$except(err, error_sample1(i)) {
            break; // OOPS: `break` stops `e$except`, not outer for loop
        }
    }
    return EOK;
}
```


## Standard `Error`
CEX implements a standard `Error` namespace, which typical for most common situations if you might need to handle them.

```c
const struct _CEX_Error_struct Error = {
    .ok = EOK,                       // Success
    .memory = "MemoryError",         // memory allocation error
    .io = "IOError",                 // IO error
    .overflow = "OverflowError",     // buffer overflow
    .argument = "ArgumentError",     // function argument error
    .integrity = "IntegrityError",   // data integrity error
    .exists = "ExistsError",         // entity or key already exists
    .not_found = "NotFoundError",    // entity or key already exists
    .skip = "ShouldBeSkipped",       // NOT an error, function result must be skipped
    .empty = "EmptyError",           // resource is empty
    .eof = "EOF",                    // end of file reached
    .argsparse = "ProgramArgsError", // program arguments empty or incorrect
    .runtime = "RuntimeError",       // generic runtime error
    .assert = "AssertError",         // generic runtime check
    .os = "OSError",                 // generic OS check
    .timeout = "TimeoutError",       // await interval timeout
    .permission = "PermissionError", // Permission denied
    .try_again = "TryAgainError",    // EAGAIN / EWOULDBLOCK errno analog for async operations
}

Exception foo(int a) {
    e$except(err, error_sample1(0)) {
        if (err == Error.argument) {
            return Error.runtime; // Return another error
        }
    }
    return Error.ok; // success
}
```

## Making custom user exceptions
### Extending with existing functionality

Probably you only need to make custom errors when you need specific needs of handling, which is rare case. In common case you might need to report details of the error and forget about it. Before we dive into customized error structs, let's consider what simple instruments do we have for error customization without making another entity in the code:

1. You may try to return string literals as a custom error, these errors are convenient options when you don't need to handle them (e.g. for rare/weird edge cases)
```c
Exception foo_literal(int a) {
    if (a == 777999) return "a is a duplicate of magic number";
    return EOK;
}
```
2. You may try to return standard error + log something with `e$raise()` which support location logging and custom formatting.
```c
Exception foo_ret(int a) {
    if (a == 777999) return e$raise(Error.argument, "a=%d looks weird", a);
    return EOK;
}
```

### Custom error structs
If you need custom handling, you might need to create a new dedicated structure for errors.

Here are some requirements for a custom error structure:

1. It has to be a constant global variable
2. All fields must be initialized, uninitialized fields are NULL therefore they are **success** code.

```c
// myerr.h
extern const struct _MyError_struct
{
    Exc foo;
    Exc bar;
    Exc baz;
} MyError;

// myerr.c
const struct _MyError_struct MyError = {
    .foo = "FooError",
    .bar = "BarError",
    // WARNING: missing .baz - which will be set to NULL => EOK
}

// other.c
#include "cex.h"
#include "myerr.h"

Exception foo(int a) {
    e$except(err, error_sample1(0)) {
        if (err == Error.argument) {
            return MyError.foo;
        }
    }
    return Error.ok; // success
}

```

## Advanced topics
### Performance
#### Errors are pointers
Using strings as error value carrier may look controversial at the first glance. However let's remember that strings in C are `char*`, and essentially `*` part means that it's a `size_t` integer value of a memory address. Therefore CEX approach is to have set of pre-defined and constant memory addresses that hold standard error values (see `Standard Error` section above).

So for error handling we need to compare return value with `EOK|NULL|Error.ok` to check if error was returned or not. Then we check address of returned error and compare it with the address of the standard error.

With this being said, performance of typical error handling in CEX is one assembly instruction that compares a register with `NULL` and one instruction for comparing **address** of an error with some other constant address when handling returned error type.

> [!NOTE]
> 
> CEX uses direct pointer comparison `if (err == Error.argument)`, instead of string content comparison `if(strcmp(err, "ArgumentError") == 0) /* << BAD */`

#### Branch predictor control
All CEX `e$` macros uses `unlikely` a.k.a. `__builtin_expect` to shape assembly code in the way of favoring happy path, for example this is a `e$assert` source snippet:
```c
#    define e$assert(A)                                                                             \
        ({                                                                                          \
            if (unlikely(!((A)))) {                                                                 \
                __cex__fprintf(stdout, "[ASSERT] ", __FILE_NAME__, __LINE__, __func__, "%s\n", #A); \
                return Error.assert;                                                                \
            }                                                                                       \
        })
```

The `unlikely(!(A))` hints the compiler to place assembly instructions in a way of favoring happy path of the `e$assert`, which is a performance gain when you have multiple error handling checks and/or big blocks for error handling.

### Compatibility
Be careful if you need to expose CEX exception returning functions to an API. Sometimes, if you are working with different shared libraries, the addresses of the same errors might be different. If user code is intended to check and handle API errors, maybe it's better to stick to C-compatible approach instead of CEX errors.

CEX Exceptions work best when you use them in single address space of an app or a library. If you need to cross this boundary, do your best assessment for pros and cons.

## Useful code patterns

### Escape `main()` when possible

CEX approach is to keep `main()` function separated and as short as possible. This opens capabilities for full code unit testing, unity builds, and tracebacks. This is a typical example app:
```c
// app_main.c file
#include "cex.h"
Exception
app_main(int argc, char** argv)
{
    bool my_flag = false;
    argparse_c args = {
        .description = "New CEX App",
        argparse$opt_list(
            argparse$opt_help(),
            argparse$opt(&my_flag, 'c', "ctf", .help = "Capture the flag"),
        ),
    };
    if (argparse.parse(&args, argc, argv)) { return Error.argsparse; }
    io.printf("MOCCA - Make Old C Cexy Again!\n");
    io.printf("%s\n", (my_flag) ? "Flag is captured" : "Pass --ctf to capture the flag");
    return EOK;
}

// main.c file
#define CEX_IMPLEMENTATION   // this only appears in main file, before #include "cex.h"
#include "cex.h"
#include "app_main.c"  // NOTE: include .c, using unity build approach

int
main(int argc, char** argv)
{
    if(app_main(argc, argv)) { return 1; }
    return 0;
}
```

### Inversion of error checking
Instead of doing `if` nesting, try an opposite approach, check an error and exit. In CEX you can also use `e$assert()` for a quick and dirty checking with one line.

::: {.panel-tabset}
#### CEX 
```c
Exception
app_main(int argc, char** argv)
{
    e$assert(argc == 2);  // assert shortcut
    if (str.eq(argv[1], "MOCCA")) { return Error.integrity; }

    io.printf("MOCCA - Make Old C Cexy Again!\n");
    return EOK;
}
```
#### Nested errors
```c
Exception
app_main(int argc, char** argv)
{
    if (argc > 1) {
        if (str.eq(argv[1], "MOCCA")) {
            io.printf("MOCCA - Make Old C Cexy Again!\n");
        } else {
            return Error.integrity;
        }
    } else {
        return Error.argument;
    }
    return EOK;
}
```
:::


### Resource cleanup
Sometimes you need to open resources, manage your memory, and carry error code. Or maybe we have to use legacy API inside function, with some incompatible error code calls. Here is a CEX flavored implementation of common `goto fail` C code pattern.

::: {.panel-tabset}
#### Cleanup
```c

Exception
print_zip(char* zip_path, char* extract_dir)
{
    Exc result = Error.runtime; // NOTE: default error code, setting to error by default

    // Open the ZIP archive
    int err;
    struct zip* archive = NULL;
    e$except_null (archive = zip_open(zip_path, 0, &err)) { goto end; }

    i32 num_files = zip_get_num_entries(archive, 0);

    for (i32 i = 0; i < num_files; i++) {
        struct zip_stat stat;
        if (zip_stat_index(archive, i, 0, &stat) != 0) {
            result = Error.integrity;  // NOTE: we can substitute error code if needed
            goto end;
        }

        // NOTE: next may return error on buffer overflow -> goto end then
        char output_path[64];
        e$goto(str.sprintf(output_path, sizeof(output_path), "%s/%s", extract_dir, stat.name), end);

        io.printf("Element: %s\n", output_path);
    }

    // NOTE: success when no `goto end` happened, only one happy outcome
    result = EOK;

end:
    // Cleanup and result
    zip_close(archive);
    return result;
}
```

#### On fail
```c
MyObj
MyObj_create(char* path, usize buf_size)
{
    MyObj self = {0};

    e$except_null (self.file = fopen(path, "r")) { goto fail; }

    self.buf = malloc(buf_size);
    if (self.buf == NULL) { goto fail; }

    e$goto(fetch_data(&self.data), fail);
    
    // MyObj was initialized and in consistent state
    return self;

fail:
    // On error - do a cleanup of initialized stuff
    if (self.file) { fclose(self.file); }
    if (self.buf) { free(self.buf); }
    memset(&self, 0, sizeof(MyObj));
    return self;
}
```

:::
