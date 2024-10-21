+++
title = "A type-safe(ish) dynamic library loader in C++20"
date = 2024-01-11
draft = true

[taxonomies]
categories = ["Projects"]
tags = ["c++", "metaprogramming", "c++20"]
+++

# Introduction

I recently had the need to load a dynamic library at runtime in C++. As I'm a huge fan of type safety and compile-time errors, I tried to find a library using C++20 features. Surprisingly, I couldn't find any. So I decided to write my own.

In this article, we will build a library that allows us to load a dynamic library at runtime and call its functions in a type-safe (ish) manner. The meat of the article will not be about loading dynamic libraries, but rather how advanced C++20 features can be used to make this process safer and more convenient.

The whole code is available [here](https://github.com/Guekka/sharlibs). At the time of writing, the library is not ready for cross-plaform consumption, but I nonetheless believe its design can be interesting.

> If you are only interested in more advanced concepts, please jump to [Going further](#going-further).

# The problem

Let's say we have a dynamic library called `libfoo.so` that exports a function called `foo`:

```cpp
#pragma once

extern "C" int foo() {
    return 42;
}
```

On POSIX systems, che usual way to load this library at runtime is to use `dlopen` and `dlsym`:

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
- Error handling can be forgotten.
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

Now, the library will be automatically unloaded when the `dylib` object goes out of scope.

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
            return Error(sharlibs_error_code::invalid_path);
        }
        return dylib(handle);
    }
}
```

Note how the constructor is private, and how we use a static function to create a `dylib` object. This is a common pattern in Rust to handle errors at construction time.

What advantages does this solution have over exceptions? Well, it's a matter of taste and a common debate online. Personally, I like having to handle errors when they occur, and I love the monadic style of error handling that `std::expected` allows.

## Solution #3: Type-safe function pointers

> The function pointer is not type-safe, so we have to manually cast it to the correct type.

The first step we can take is to delegate the cast to the `dylib` class, effectively abstracting it from the user:
```cpp
class dylib {
    std::unique_ptr<void, IntegralFunction<dlclose>> handle_;

    explicit dylib(void *handle) : handle_(handle) {
    }

public:
    template <typename T>
    [[nodiscard]] auto get_function(std::string_view name) const noexcept -> expected<T, Error> {
        auto symbol = dlsym(handle_.get(), name.data());
        if (!symbol) {
            return Error(sharlibs_error_code::invalid_symbol);
        }
        return reinterpret_cast<T>(symbol);
    }
}
```

This is already much better:
```cpp
// ...
auto foo = lib.get_function<int (*)()>("foo");
// using the monadic style
auto result = foo.transform([](auto f) { return f(); });
// using the imperative style
if (result) {
    auto result = *foo();
}
```

Still, the user has to type the function signature everywhere it is used. An alias can be used, but it would not protect us from a confusion between two libraries:
```cpp
using foo_t = int (*)();
using hello_t = char*(*)();

auto lib_hello = dylib::load("libhello.so");
auto lib_foo = dylib::load("libfoo.so");

auto foo = lib_foo.get_function<foo_t>("foo");
// oops, hello signature is not correct
auto hello = lib_hello.get_function<foo_t>("hello");
```

We will see in the next section how we can solve this problem.

## Solution #4: Compile-time strings

> The function name is a string, so we have to manually type it, which is error-prone.

Once again, the fix appears obvious: let's define a struct representing a function, and always use it instead of a string:
```cpp
template <class... Signature>
struct function {
    std::string_view name;
};
```

We can modify `get_function`:
```cpp
template <typename T>
[[nodiscard]] auto get_function(function<T> f) const noexcept -> expected<T, Error> {
    auto symbol = dlsym(handle_.get(), f.name.data());
    if (!symbol) {
        return Error(sharlibs_error_code::invalid_symbol);
    }
    return reinterpret_cast<T>(symbol);
}
```

In use:
```cpp
constexpr auto foo_fn = function<int (*)()>{.name = "foo"};

// ...
auto foo = lib.get_function(foo_fn);
```

# Going further

So, that's it? Did we fix dynamic library loading? Not quite. There are already some libraries that do what we just did, and they are much more complete.
For example, [dylib](https://github.com/martin-olivier/dylib) appears to be a popular choice.

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

We can see the use of exceptions to force the programmer to handle error, RAII to automatically unload the library, and type-safe function pointers. Strings are still used to define the function name, but this is not a big deal.

But we can do better. What if we could define the library interface at compile time, so that it can only be used as intended? At the end of the section, we will have a library that looks like this:

```cpp
    constexpr auto f_tolower = sharlibs::DynamicFunction<"tolower", int(int)>{};
    constexpr auto f_toupper = sharlibs::DynamicFunction<"toupper", int(int)>{};

    const auto lib = sharlibs::DynamicLib<"libc.so.6", f_tolower, f_toupper>::open();
    REQUIRE(lib.has_value());

    std::optional<int> result = lib->call<f_tolower>('A');
    REQUIRE(result.has_value());
    REQUIRE(result.value() == 'a');

    result = lib->call<f_toupper>('a');
    REQUIRE(result.has_value());
    REQUIRE(result.value() == 'A');
```
Trying to call a function that is not in the library will result in a compile-time error.

## Compile-time strings

*Credits goes to [vector-of-bool](https://vector-of-bool.github.io/2021/10/22/string-templates.html) for the concept.*

Since C++20, we can define strings as template parameters. This is a powerful feature that can be used to define compile-time strings. Here's a simple implementation:

```cpp
template<size_t Length>
struct FixedString
{
    constexpr FixedString(const char (&arr)[Length + 1]) noexcept
    {
        for (size_t i = 0; i < Length; ++i)
            chars[i] = arr[i];
    }

    char chars[Length + 1] = {}; // +1 for null terminator

    [[nodiscard]] constexpr auto size() const noexcept -> size_t { return Length; }

    [[nodiscard]] constexpr auto operator[](size_t index) const noexcept -> char { return chars[index]; }
};

template<size_t N>
FixedString(const char (&arr)[N]) -> FixedString<N - 1>; // Drop the null terminator
```

This class can be used like this:
```cpp
constexpr auto str = FixedString("hello");
static_assert(str.size() == 5);
static_assert(str[0] == 'h');
```

Compile-time strings!

## Compile-time function names

We can now define a struct representing a function:
```cpp
template<detail::FixedString function_name, class Prototype>
struct DynamicFunction
{
    constexpr DynamicFunction() noexcept = default;

    using function_type                       = Prototype;
    constexpr static detail::FixedString name = function_name;
};
```

This struct stores both the name of the function and its prototype. It can replace the `function` struct we defined earlier.
```cpp
constexpr auto f_tolower = DynamicFunction<"tolower", int(int)>{};

static_assert(f_tolower.name.size() == 6);
static_assert(std::is_same_v<decltype(f_tolower)::function_type, int(int)>);
```

We're also going to replace `get_function` with `call`: this way, we can no longer store functions independently from the library instance, which could have resulted in use after free.

This `DynamicFunction` struct is simple, but it makes `call` dramatically more complex:
```cpp
    template<DynamicFunction func, class... Args, class Func = typename decltype(func)::function_type>
    [[nodiscard]] auto call(Args &&...args) const noexcept
        -> std::optional<std::conditional_t<std::is_void_v<std::invoke_result_t<Func, Args...>>,
                                            std::monostate,
                                            std::invoke_result_t<Func, Args...>>>
        requires std::is_invocable_v<Func, Args...>
    {
        if (auto symbol = get_symbol(func.name.chars); symbol != nullptr)
        {
            auto function = reinterpret_cast<std::add_pointer_t<Func>>(symbol);
            if constexpr (std::is_void_v<std::invoke_result_t<Func, Args...>>)
            {
                std::invoke(function, std::forward<Args>(args)...);
                return std::monostate{};
            }
            else
            {
                return std::invoke(function, std::forward<Args>(args)...);
            }
        }

        return std::nullopt;
    }
```
What's happening here?
- We alias the function prototype to `Func`
- `get_symbol` is equivalent to `dlsym`
- We access the function name using `func.name.chars`
- We cast the symbol to the correct function pointer type. `std::add_pointer_t` is used to handle function references.
- We invoke the function using `std::invoke` using perfect forwarding of the arguments.
- We return an `std::optional` with the result of the function. If the function returns `void`, we return `std::monostate`.

The `requires` clause, which is part of the Concepts feature, ensures that the function can be called with the given arguments.

## Compile-time library name

Similarly, we can make the library name a template parameter:
```cpp
template<detail::FixedString library_name>
struct DynamicLib
{
    // ...
    [[nodiscard]] static auto open() noexcept -> std::optional<DynamicLib>
    {
        if (auto handle = dlopen(library_name.chars, RTLD_NOW); handle != nullptr)
        {
            return DynamicLib{handle};
        }

        return std::nullopt;
    }
};
```

## Requiring functions to be predeclared

In order to force the function to be predeclared, we're going to need a helper, or a `trait` in C++ terms:
```cpp
template<class What, class... Args>
struct is_present : std::disjunction<std::is_same<What, Args>...>
{
};

template<class What, class... Args>
constexpr auto is_present_v = is_present<What, Args...>::value;
```
    
This trait can be used to check if a type is part of a list of types:
```cpp
static_assert(is_present_v<int, int, char, float>); // true, since int is part of the list
static_assert(!is_present_v<int, char, float>); // false, since int is not part of the list
```

If we modify `DynamicLib` to require the functions to be predeclared:
```cpp
template<detail::FixedString library_name, DynamicFunction... functions>
struct DynamicLib
{
    // ...
};
```
We can now add a new `requires` clause to `call`: the function must be part of the list of functions:
```cpp
    template<DynamicFunction func, class... Args, class Func = typename decltype(func)::function_type>
    [[nodiscard]] auto call(Args &&...args) const noexcept
        -> std::optional<std::conditional_t<std::is_void_v<std::invoke_result_t<Func, Args...>>,
                                            std::monostate,
                                            std::invoke_result_t<Func, Args...>>>
        requires std::is_invocable_v<Func, Args...> && is_present_v<decltype(func), decltype(functions)...>
    {
        // ...
    }
```

## Conclusion

I am not going to put all this code together, as it is available in [sharlibs](https://github.com/Guekka/sharlibs).

In this post, we saw how advanced C++20 metaprogramming features can be useful. Now, should we actually use this kind of library in production? I don't know. It's a fun exercise, but it might be overkill for most projects. It's up to you to decide if the benefits outweigh the complexity. Furthermore, it is likely such templates will slow down compilation times.

I hope you enjoyed this article. If you have any questions or suggestions, feel free to reach out to me or post a comment below.

