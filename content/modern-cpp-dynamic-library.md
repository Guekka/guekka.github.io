+++
title = "A type-safe dynamic library loader in C++20"
date = 2024-01-11
draft = true

[taxonomies]
categories = ["Projects"]
tags = ["c++", "metaprogramming", "c++20"]
+++

# Introduction

I recently had the need to load a dynamic library at runtime in C++. As I'm a huge fan of type safety and compile-time errors, I tried to find a library using C++20 features. Surprisingly, I couldn't find any. So I decided to write my own.

In this article, we will build a library that allows us to load a dynamic library at runtime and call its functions in a type-safe (ish) manner. The meat of the article will not be about loading dynamic libraries, but rather how advanced C++20 features can be used to make this process safer and more convenient. Metaprogramming fans will be happy.

The whole code is available [here](https://github.com/Guekka/sharlibs). At the time of writing, the library is not ready for cross-plaform consumption, but I still believe it can be useful.



# The problem

Let's say we have a dynamic library called `libfoo.so` that exports a function called `foo`:

```cpp
#pragma once

extern "C" int foo() {
    return 42;
}
```

The usual way to load this library at runtime is to use `dlopen` and `dlsym`:

```cpp
#include <dlfcn.h>

int main() {
    void *lib = dlopen("libfoo.so", RTLD_NOW);
    if (!lib) {
        // handle load error
    }

    auto foo = reinterpret_cast<int (*)()>(dlsym(lib, "foo"));
    if (!foo) {
        // handle symbol not found
    }

    auto result = foo();
    dlclose(lib);
}
```

We can immediately see a few problems with this approach:
- `dlclose` has to be called manually, which is error-prone.
- Error handling is not mandatory. It is easy to forget to check for errors.
- The function pointer is not type-safe, so we have to manually cast it to the correct type.
- The function name is a string, so we have to manually type it, which is error-prone.

Most of these issues boils down to the fact that this API allows the programmer to make mistakes.

Let's solve each of these problems one by one.

# Solutions

## Solution #1: RAII

RAII (Resource Acquisition Is Initialization), also known as SBRM (Scope-Bound Resource Management), is one of the most important advantages of C++ over C. It allows us to manage resources in a safe and convenient manner.

Most of you are already familiar with RAII: smart pointers, `std::lock_guard` or even `std::string` are all examples of RAII.

Let's see how we can use RAII to solve the first problem:

```cpp
#include <dlfcn.h>

class dylib {
public:
    explicit dylib(std::string_view path) {
        handle = dlopen(path.c_str(), RTLD_NOW);
        if (!handle) {
            throw load_error(dlerror());
        }
    }

    ~dylib() {
        dlclose(handle);
    }
```





# Existing solutions: dylib

Some libraries already exist to solve this problem. For example, [dylib](https://github.com/martin-olivier/dylib) appears to be a popular choice.

Here's how we would use it:

```cpp
#include <dylib.hpp>

int main() {
    try {
        dylib lib("libfoo.so");
        auto foo = lib.get_function<int()>("foo");
        auto result = foo();
    } catch (const dylib::load_error &load_error) {
        // handle load error
    } catch (const dylib::symbol_error &symbol_error) {
        // handle symbol not found
    }
}
```

This is already much better. The library is automatically unloaded when the `dylib` object goes out of scope thanks to RAII, and error handling is mandatory. It uses exceptions, which is not my preferred way of handling errors, but it's not a big deal.

Here's a simple implementation of `dylib`:

```cpp
class dylib {
public:
    explicit dylib(std::string_view path) {
        handle = dlopen(path.c_str(), RTLD_NOW);
        if (!handle) {
            throw load_error(dlerror());
        }
    }

    ~dylib() {
        dlclose(handle);
    }

    template <typename T>
    T get_function(const std::string &name) {
        auto symbol = dlsym(handle, name.c_str());
        if (!symbol) {
            throw symbol_error(dlerror());
        }
        return reinterpret_cast<T>(symbol);
    }
}
```

Still, this library does not protect us from some mistakes.

## Mistake #1: Calling a function after the library has been unloaded

```cpp
using foo_t = int (*)();

auto load_foo() -> foo_t {
    try {
        dylib lib("libfoo.so");
        return lib.get_function<int()>("adder");
    } catch (const dylib::load_error &load_error) {
        // handle load error
    } catch (const dylib::symbol_error &symbol_error) {
        // handle symbol not found
    }
}

int main() {
    auto foo = load_foo();
    auto result = foo(); // oops, the library has already been unloaded
}
```

