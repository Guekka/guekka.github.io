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

In this article, we will build a library that allows us to load a dynamic library at runtime and call its functions in a type-safe (ish) manner. The meat of the article will not be about loading dynamic libraries, but rather how advanced C++20 features can be used to make this process safer and more convenient.

The whole code is available [here](https://github.com/Guekka/sharlibs). At the time of writing, the library is not ready for cross-plaform consumption, but I still believe its design is interesting.



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

> `dlclose` has to be called manually, which is error-prone.

RAII (Resource Acquisition Is Initialization), also known as SBRM (Scope-Bound Resource Management), is one of the most important advantages of C++ over C. It allows us to manage resources in a safe and convenient manner.

Most of you are already familiar with RAII: smart pointers, `std::lock_guard` or even `std::string` are all examples of RAII.

Let's see how we can use RAII to solve the first problem:

```cpp
class dylib {
    void *handle;
public:
    explicit dylib(std::string_view path) : handle(dlopen(path.c_str(), RTLD_NOW)) {
    }

    ~dylib() {
        dlclose(handle);
    }
```

This can be further simplified using `std::unique_ptr`:
```cpp
class dylib {
    // decltype(lambda) in this context only works since C++20
    std::unique_ptr<void, decltype([](native_handle handle) { dlclose(handle); })> handle_;

public:
    explicit dylib(std::string_view path) : handle(dlopen(path.c_str(), RTLD_NOW)) {
    }
}
```

Another way with `unique_ptr`, a bit more complex but shorter:
```cpp
// ...

template<auto Function>
using IntegralFunction = std::integral_constant<decltype(Function), Function>;

class dylib {
    std::unique_ptr<void, IntegralFunction<dlclose>> handle_;
    // ...
}
```

If you are interested in the different ways to use `unique_ptr` with custom deleters, this [reddit thread](https://www.reddit.com/r/cpp/comments/18zyae6/how_to_wrap_unsigned_char_in_unique_ptr_with/) is quite informative.

## Solution #2: Error handling

> Error handling is not mandatory. It is easy to forget to check for errors.

### Exceptions

This one is easy: we can use exceptions to handle errors. This will avoid forgetting to check for errors.
```cpp
class sharlibs_error : public std::runtime_error {
public:
    explicit sharlibs_error(const std::string &message) : std::runtime_error(message) {
    }
};

class load_error : public sharlibs_error {
public:
    explicit load_error(const std::string &message) : sharlibs_error(message) {
    }
};

class dylib {
    std::unique_ptr<void, IntegralFunction<dlclose>> handle_;

public:
    explicit dylib(std::string_view path) : handle(dlopen(path.c_str(), RTLD_NOW)) {
        if (!handle) {
            throw load_error(dlerror());
        }
    }
}
```

Now, a programmer can forget to check for errors, but they will have to handle the exception at some point.

### `std::expected`

However, exceptions are not the only way to handle errors. Rust error handling is very appreciated by the community. To put it simply, Rust uses a type called `Result` to represent the result of an operation that can fail. This type can either be `Ok` (the operation succeeded) or `Err` (the operation failed). This is called a discriminated union, or more formally a [sum type](https://en.wikipedia.org/wiki/Tagged_union).

This is a very powerful concept, and it is possible to implement it in C++ using `std::variant`:
```cpp
template <typename Value, typename Error>
class expected {
    std::variant<Value, Error> value_or_error;

    // add constructors, assignment operators, etc.
};
```

The `expected` type is already implemented in libraries such as [tl::expected](https://github.com/TartanLlama/expected), and is part of the standard library in C++23.

I have not found any recommendations for defining the `Error` type, so my personal convention is the following:
```cpp
class Error : public std::error_code {
    std::source_location loc_;

public:
    explicit Error(std::error_code ec, std::source_location loc = std::source_location::current()) noexcept
        : std::error_code(ec), loc_(loc) {
    }

    [[nodiscard]] auto location() const noexcept -> std::source_location {
        return loc_;
    }
```

This allows us to keep track of where the error was created, which is very useful for debugging.

The downside of `std::error_code` is all the boilerplate it requires. For example, in our case:
```cpp
enum class sharlibs_error_code {
    success,
    invalid_path,
    invalid_handle,
    invalid_symbol
};

namespace std {
    template <>
    struct is_error_code_enum<sharlibs_error_code> : true_type {
    };
}

auto make_error_code(sharlibs_error_code ec) noexcept -> std::error_code {
    return std::error_code(static_cast<int>(ec), load_error_category());
}

class sharlibs_error_category : public std::error_category {
public:
    [[nodiscard]] auto name() const noexcept -> const char * override {
        return "sharlibs";
    }

    [[nodiscard]] auto message(int ev) const -> std::string override {
        switch (static_cast<sharlibs_error_code>(ev)) {
            case sharlibs_error_code::success:
                return "success";
            case sharlibs_error_code::invalid_path:
                return "invalid path";
            case sharlibs_error_code::invalid_handle:
                return "invalid handle";
            case sharlibs_error_code::invalid_symbol:
                return "invalid symbol";
            default:
                return "unknown error";
        }
    }

    [[nodiscard]] auto default_error_condition(int ev) const noexcept -> std::error_condition override {
        switch (static_cast<sharlibs_error_code>(ev)) {
            case sharlibs_error_code::success:
                return std::errc::success;
            case sharlibs_error_code::invalid_path:
                return std::errc::invalid_argument;
            case sharlibs_error_code::invalid_handle:
                return std::errc::bad_file_descriptor;
            case sharlibs_error_code::invalid_symbol:
                return std::errc::no_such_file_or_directory;
            default:
                return std::error_condition(ev, *this);
        }
    }
};

[[nodiscard]] auto sharlibs_error_category() noexcept -> const sharlibs_error_category & {
    static sharlibs_error_category instance;
    return instance;
}
```
The [Outcome documentation](https://ned14.github.io/outcome/motivation/plug_error_code/) has a good explanation of creating error codes.

This is a lot of code for a simple error code. Note that `std::error_code` is *not mandatory* for `std::expected` to work, this is just my personal convention.

Now that we have defined our error type, we can use it in our `dylib` class:
```cpp
class dylib {
    std::unique_ptr<void, IntegralFunction<dlclose>> handle_;

    explicit dylib(void *handle) : handle_(handle) {
    }

public:
   [[nodiscard]] static auto load(std::string_view path) noexcept -> expected<dylib, Error> {
        auto handle = dlopen(path.c_str(), RTLD_NOW);
        if (!handle) {
            return Error(load_error_code::invalid_path);
        }
        return dylib(handle);
    }
}
```

Note how the constructor is private, and how we use a static function to create a `dylib` object. This is a common pattern in Rust to handle errors at construction time.

What advantages does this solution have over exceptions? Well, it's a matter of taste. I personally prefer this solution because it forces the programmer to handle errors, and it is easier to reason about the control flow of the program.

## Solution #3: Type-safe function pointers

> The function pointer is not type-safe, so we have to manually cast it to the correct type.

## Solution #4: Compile-time strings

> The function name is a string, so we have to manually type it, which is error-prone.

# Side note: existing solutions

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

