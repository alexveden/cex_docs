---
title: "Getting started with CEX"
---

## What is CEX
Cex is Comprehensively EXtended C Language. CEX was born as alternative answer to a plethora of brand new LLVM based languages which strive to replace old C. CEX still remains C language itself, with small but important tweaks that makes CEX a completely different development experience.

I tried to bring best ideas from the modern languages while maintaining smooth developer experience for writing C code. The main goal of CEX is to provide tools for developers and helping them writing high quality C code in general.

### Core features

- Single header, cross-platform, drop-in C language extension
- No dependencies except C compiler
- Self contained build system: CMake/Make/Ninja no more
- Modern memory management model
- New error handling model
- New strings
- Namespaces
- Code quality oriented tools
- New dynamic arrays and hashmaps with seamless C compatibility


### Solving old C problems

CEX is another attempt to make old C a little bit better. Unlike other new system languages like Rust, Zig, C3 which tend to start from scratch, CEX focuses on evolution process and leverages existing tools provided by modern compilers to make code safer, easy to write and debug.

| C Problem | CEX Solution |
| -------------- | --------------- |
| Bug prone memory management | CEX provides allocator centric and scoped memory allocation. It uses ArenaAllocators and Temporary allocator in `mem$scope()` which decrease probability of memory bugs.  |
| Unsafe arrays |  Address sanitizers are enabled by default, so you'll get your crashes as in other languages. |
| 3rd party build system  |  Integrated build system, eliminates flame wars about what it better. Now you can use Cex to run your build scripts, like in `Zig`  |
| Rudimentary error handling | CEX introduces `Exception` type and compiler forces you to check it. New error handling approach make error checking easy and open cool possibilities like stack traces in C. |
| C is unsafe | Yeah, and it's a cool feature! On other hand, CEX provides unit testing engine and fuzz tester support out of the box.  |
| Bad string support | String operations in CEX are safe, NULL and buffer overflow resilient. CEX has dynamic string builder, slices and C compatible strings. |
| No data structures |  CEX has type-safe generic dynamic array and hashmap types, they cover 80% of all use cases. |
| No namespaces |  It's more about LSP, developer experience and readability. It much better experience to type and read `str.slice.starts_with` than `str_slice_starts_with`. |


## Making new CEX project

You can initialize a working boiler plate project just using a C compiler and the `cex.h` file.

> [!NOTE]
>
> Make sure that you have a C compiler installed, we use `cc` command as a default compiler. You may replace it with gcc or clang.

1. Make a project directory
```sh
mkdir project_dir
cd project_dir
```
2. Download [cex.h](https://raw.githubusercontent.com/alexveden/cex/refs/heads/master/cex.h)
3. Make a seed program

At this step we are compiling a special pre-seed program that will create a template project at the first run
```sh
cc -D CEX_NEW -x c ./cex.h
```
4. Run cex program for project initilization

Cex program automatically creating a project structure with sample app and unit tests. Also it recompiles itself to become universal build system for the project. You may change its logic inside `cex.c` file, this is your build script now.
```sh
./cex
```
5. Now your project is ready to go 

Now you can lauch a sample program or run its unit tests.
```sh
./cex test run all
./cex app run myapp
```

6. This is how to check your environment and build variables
```sh
> ./cex config

cexy$* variables used in build system, see `cex help 'cexy$cc'` for more info
* CEX_LOG_LVL               4
* cexy$build_dir            ./build
* cexy$src_dir              ./examples
* cexy$cc                   cc
* cexy$cc_include           "-I."
* cexy$cc_args_sanitizer    "-fsanitize-address-use-after-scope", "-fsanitize=address", "-fsanitize=undefined", "-fsanitize=leak", "-fstack-protector-strong"
* cexy$cc_args              "-Wall", "-Wextra", "-Werror", "-g3", "-fsanitize-address-use-after-scope", "-fsanitize=address", "-fsanitize=undefined", "-fsanitize=leak", "-fstack-protector-strong"
* cexy$cc_args_test         "-Wall", "-Wextra", "-Werror", "-g3", "-fsanitize-address-use-after-scope", "-fsanitize=address", "-fsanitize=undefined", "-fsanitize=leak", "-fstack-protector-strong", "-Wno-unused-function", "-Itests/"
* cexy$ld_args
* cexy$fuzzer               "clang", "-O0", "-Wall", "-Wextra", "-Werror", "-g", "-Wno-unused-function", "-fsanitize=address,fuzzer,undefined", "-fsanitize-undefined-trap-on-error"
* cexy$debug_cmd            "gdb", "-q", "--args"
* cexy$pkgconf_cmd          "pkgconf"
* cexy$pkgconf_libs
* cexy$process_ignore_kw    ""
* cexy$cex_self_args
* cexy$cex_self_cc          cc

Tools installed (optional):
* git                       OK
* cexy$pkgconf_cmd          OK ("pkgconf")
* cexy$vcpkg_root           Not set
* cexy$vcpkg_triplet        Not set

Global environment:
* Cex Version               0.14.0 (2025-06-05)
* Git Hash                  07aa036d9094bc15eac8637786df0776ca010a33
* os.platform.current()     linux
* ./cex -D<ARGS> config     ""
```

## Meet Cexy build system
`cexy$` is a build system integrated with Cex, which helps to manage your project, run tests, find symbols and getting help. 


```sh
> ./cex --help
Usage:
cex  [-D] [-D<ARG1>] [-D<ARG2>] command [options] [args]

CEX language (cexy$) build and project management system

help                Search cex.h and project symbols and extract help
process             Create CEX namespaces from project source code
new                 Create new CEX project
stats               Calculate project lines of code and quality stats
config              Check project and system environment and config
libfetch            Get 3rd party source code via git or install CEX libs
test                Test running
fuzz                Generic fuzz tester
app                 Generic app build/run/debug

You may try to get help for commands as well, try `cex process --help`
Use `cex -DFOO -DBAR config` to set project config flags
Use `cex -D config` to reset all project config flags to defaults
```


## Code example
### Hello world in CEX
```c
#define CEX_IMPLEMENTATION
#include "cex.h"

int
main(int argc, char** argv)
{
    io.printf("MOCCA - Make Old C Cexy Again!\n");
    return 0;
}
```

### Holistic function
```c
// CEX has special exception return type that forces the caller to check return type of calling
//   function, also it provides support of call stack printing on errors in vanilla C
Exception
cmd_custom_test(u32 argc, char** argv, void* user_ctx)
{
    // Let's open temporary memory allocator scope (var name is `_`)
    //  it will free all allocated memory after any exit from scope (including return or goto)
    mem$scope(tmem$, _)
    { 
        e$ret(os.fs.mkpath("tests/build/")); // make directory or return error with traceback
        e$assert(os.path.exists("tests/build/")); // evergreen assertion or error with traceback

        // auto type variables
        var search_pattern = "tests/os_test/*.c";

        // Trace with file:<line> + formatting
        log$trace("Finding/building simple os apps in %s\n", search_pattern);

        // Search all files in the directory by wildcard pattern
        //   allocate the results (strings) on temp allocator arena `_`
        //   return dynamic array items type of `char*`
        arr$(char*) test_app_src = os.fs.find(search_pattern, false, _);

        // for$each works on dynamic, static arrays, and pointer+length
        for$each(src, test_app_src)
        {
            char* tgt_ext = NULL;
            char* test_launcher[] = { cexy$debug_cmd }; // CEX macros contain $ in their names

            // arr$len() - universal array length getter 
            //  it supports dynamic CEX arrays and static C arrays (i.e. sizeof(arr)/sizeof(arr[0]))
            if (arr$len(test_launcher) > 0 && str.eq(test_launcher[0], "wine")) {
                // str.fmt() - using allocator to sprintf() format and return new char*
                tgt_ext = str.fmt(_, ".%s", "win");
            } else {
                tgt_ext = str.fmt(_, ".%s", os.platform.to_str(os.platform.current()));
            }

            // NOTE: cexy is a build system for CEX, it contains utilities for building code
            // cexy.target_make() - makes target executable name based on source
            char* target = cexy.target_make(src, cexy$build_dir, tgt_ext, _);

            // cexy.src_include_changed - parses `src` .c/.h file, finds #include "some.h",
            //   and checks also if "some.h" is modified
            if (!cexy.src_include_changed(target, src, NULL)) {
                continue; // target is actual, source is not modified
            }

            // Launch OS command and get interactive shell
            // os.cmd. provides more capabilities for launching subprocesses and grabbing stdout
            e$ret(os$cmd(cexy$cc, "-g", "-Wall", "-Wextra", "-o", target, src));
        }
    }

    // CEX provides capabilities for generating namespaces (for user's code too!)
    // For example, cexy namespace contains
    // cexy.src_changed() - 1st level function
    // cexy.app.run() - sub-level function
    // cexy.cmd.help() - sub-level function
    // cexy.test.create() - sub-level function
    return cexy.cmd.simple_test(argc, argv, user_ctx);
}
```
### Code FAQ (TODO links)
* Q: Why so many `$` - TODO
* A: Because in CEX style (TODO LINK) it indicates a macro function call, this helps easily distinguish macros in code without OVER_CAPSING().

---

* Q: What is `mem$scope(tmem$)` and what is `_`
* A: It opens temporary memory scope using built-in temporary arena allocator, all allocations used within this scope will be automatically free after exit. Learn more about how it works (TODO)

---

* Q: What are `Exception` `e$ret` + how errors are handled
* A: They are parts of CEX error handling (TODO LINK) which simplifies code flow, allows traceback error printing and simlifies error handling. `e$ret` returns the error of inside call and exits the fuction with the same error, this is a part of error traceback mechanism.

---

* Q: How `arr$(char*)` works
* A: It's a generic dynamic array type initialization, allowing you to declare dynamic arrays of any type and work with them seamleassly in C (as with regular pointers). More info (TODO LINK)

---

* Q: How namespace calls work e.g. `cexy.target_make()`
* A: Technically speaking namespaces are big structures of constant function pointers. They help you to organize code, and support 1 level of sub-names. There is no performance hit it production build, because modern compilers are smart enough to replace them by direct calls. (TODO more info)

---

### More examples
* TODO


## Supported compilers/platforms

### Tested compilers / Libc support

* GCC - 10, 11, 12, 13, 14, 15
* Clang - 13, 14, 15, 16, 17, 18, 19, 20
* MSVC - unsupported, probably never will
* LibC tested - glibc (linux), musl (linux), ucrt/mingw (windows), macos

### Tested platforms / architectures
* Linux - x32 / x64 (glibc, gcc + clang),
* Alpine linux - (libc musl, gcc) on architectures x86_64, x86, aarch64, armhf, armv7, loongarch64, ppc64le, riscv64, and s390x (big-endian)
* Windows (via MSYS2 build) - x64 (mingw64 + clang), libc mscrt/ucrt
* Macos - x64 / arm64 (clang)

## Resources
* [GitHub Repo](https://github.com/alexveden/cex)
* [Ask a question on GitHub](https://github.com/alexveden/cex/discussions)


## Learn more
TODO: links!

* All CEX features in one place
* CEX Language in-depth
* CEX Build system
* CEX Tools

