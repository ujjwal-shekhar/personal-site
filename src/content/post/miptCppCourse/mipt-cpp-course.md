---
title: "Notes from the Master's Course in C++ (MIPT, 2025-2026)"
description: "An advanced deep dive into a lot of the C++ internals. I also work out all the assignments here."
publishDate: "26 Oct 2025"
updatedDate: "26 Oct 2025"
tags: ["c++", "compilers", "lang-features"]
pinned: true
---

In this course, we will assume that we are enabling a good level of compiler optimizations. We will also work mainly with C++23 and sometimes C++26.

## Lecture 1 : The Very Soul of C++

> Watch the lecture on YouTube [here](https://www.youtube.com/watch?v=X6GVR_3FCHU)

Some jargon that is always thrown around with cpp:

- General purpose a.k.a. the second best language for everything ;)
- Statically typed (tbd) 
- Statically compiled: the compiled -> machine code file is end all be all

The language standard is NOT the final approach to implementation, it ONLY guarantees the translation. It is an agreement between the compiler devs (producers of the language translation implmentation: gcc, clang, msvc..) and the programmers (consumers of the compiler!). Thus, the language standard either translates the program or issues a diagnostic, it should not act like a static analyzer and freely issue false positives.

:::caution
There are caveats, like linker diagnostics being used for missing forward declarations. More on that later. The primary idea is to clarify what the language standard aims to and does NOT aim to do.
:::

### Vectors of development

Consider an implementation of something like a `find_msb`. The most C-like way of doing this is to have a for loop. Since we don't have templates/generics in C, we use a macro on the parameter type:

```cpp
int find_msb(MSB_TYPE msb) {
    int MAX_BIT = sizeof(MSB_TYPE) * CHAR_BIT;
    for (int i = 0; i < MAX_BIT; i++) {
        if (msb & (1 << i)) {
            return i;
        }
    }
    return -1;
}
```

This is okay, but we wish to avoid the loop and here we see the vector of development from **common idiom to machine instructions**. The value of `i` above can be determined using a single `__builtin_clz()` call which is implemented on certain compilers and machines and translates to the equivalent of a `clzw` **single machine instruction**. This is much faster (no branch prediction needed, less memory overhead of loop variable, fewer clock cycles, cleaner code... need I say more:p).

```cpp
int find_msb(MSB_TYPE msb) {
    int total_bits = sizeof(MSB_TYPE) * CHAR_BIT;
    int leading_zeros = __builtin_clz(msb);
    return total_bits - leading_zeros - 1;
}
```

Besides, a use of macro here implies that the either us (the programmers) or the language standards (the compiler devs) have missed something... which takes us to the next vector of development **compiler instrinsics to standard implementation**.

Subtle, but the entire reason why something like `__builtin_clz` exists as a compiler/machine dependent behaviour is that it is useful. It is a natural next step for the programmer to expect from the compiler dev that this feature become a standard language feature instead i.e., from `__builtin_clz` to `template<class T> countl_zero`. Note that the `countl_zero` function can only accept unsigned types. We get:

```cpp
template<typename T>
int find_msb(T value) {
    using U = std::make_unsigned_t<T>;
    const auto digits = std::numeric_limits<U>::digits;
    const auto leading_zeros = std::countl_zero(static_cast<U>(digits));
    return digits - leading_zeros - 1;
}
```

This is okay, but but but, we can always generalize this further. The version below accepts anything that can only take in integral types. We would like to make our function accept ANY non-integral type so long as it can logically find an MSB (*think, atcoder's ModInt*). Concepts to the rescue:

```cpp
int find_msb(std::integral auto value) {
    using U = std::make_unsigned_t<decltype(value)>;
    const auto digits = std::numeric_limits<U>::digits;
    const auto leading_zeros = std::countl_zero(static_cast<U>(digits));
    return digits - leading_zeros - 1;
}
```

:::note
Notice that this version still has some type deduction machinery (`auto` uses the template type deduction machinery inside the compiler anyways!), just that we increase (but continue to restrict) the domain of the incoming input types. Cleanly imposes requirements on the incoming type by adding a layer of indirection.
:::

### Users can insert UB for compiler opts as well (+contracts FTW)

![Generated code for an unsigned char template type deduction](image.png)

To help us remove the extra LOC, avoid the `int`-promotion and avoid the slightly expensive `cmov`. ~~The grandmaster will sacrifice the exchange:P~~ We will manually insert UB to eradicate the leading path to that point and optimize awat with it any extra case handling.

:::caution
To avoid unleashing cthulhu we add a pre-condition contract to the function
:::

```cpp
int find_msb(std::integral auto value) 
pre(value != 0)
{
    if (value == 0) std::unreachable(); // -> UB, but improves generated asm
    using U = std::make_unsigned_t<decltype(value)>;
    const auto digits = std::numeric_limits<U>::digits;
    const auto leading_zeros = std::countl_zero(static_cast<U>(digits));
    return digits - leading_zeros - 1;
}
```

### Abstract Machine of cpp

- Non-deterministic (different runs may give different answers, both are correct)
- Language standard defines it
- Is not defined everywhere (yay, UB)
- Parameterized. That is, compiler/machine specific behavior of some functionality of the abstract machine must be defined by the compiler devs.

Compiler uses these properties for optimizations mostly on the basis of there being no side-effects

:::important[The so called "as-if" rule]
More formally, the compiler can do anything as long as the observable behavior of the abstract machine (program) does not change. This is a necessary condition but not sufficient.
:::

Observable behavior consists of:

- Order of data written to file.
- Accessing volatile glvalues.
- I/O device interactions.

A lack of side-effects does not necessitate the compiler to start optimizing things away. In the example below, the return value cannot be optimized away since there is an external linkage of the `i` variable and the integer being returned via the return slot might be getting assigned to a volatile glvalue.

:::note[Fun fact]
In C++ we consider the scope of connections that the compiler can see as a *translation unit*, unlike C where the file is considered a translation unit.
:::

```cpp
int check(int i) { /*External linkage*/
    return i * i + i * 8 + 42; // Cant just optimize this away
}
```

:::tip
You can usually think about this by ascertaining if something *could be* assigned to a volatile glvalue.
:::

### UB is a guarantee we give to the compiler, to do absolutely anything

If UB exists in code, not only does the compiler turn blind to that specific LOC, it is free to do anything in the code preceding the location of UB. It can remove everything and consequently lead to an infinite loop as shown below.

This responsibility is on the programmer ONLY, since they have assured the compiler that this is UB and that they guarantee that this UB will never be triggered. Thus an *as-if* behavior is triggered that makes the compiler act as if the UB is never reached. Also, [fun stuff linked here](https://lore.kernel.org/llvm/20250426200513.GA427956@ax162/).

:::important
For the compiler, UB doesn't exist. So if something goes wrong, then the responsibily is on the programmer. The compiler will usually follow a simple mechanism and remove the entire path leading upto UB by assuming that it is unreachable. This will either happen due to the as-if rule seeing no side-effects or because it is on the path to UB.
:::

![The loop is completely removed and turned into an infinite loop due to the uninitialized value UB](image-1.png)

Another example of the compiler assuming that UB cannot be reached at all.

```cpp
int foo(bool c) {
    int x, y; // x is uninitialized, UB!
    y = c ? x : 42; // The compiler removes the case where we use x at all

    // The above line becomes y = 42, and finally, constant folded to ignore
    // c all the time and just return 42. Thus, foo(true) == 42
    return y;
}
```

### Erroneous behaviour and erroneous values (C++ 26)

```cpp
char a;
char b [[indeterminate]];

char c = a + 1; // c contains an erroneous value, since uninitialized vars are EB not UB
char d = b + 1; // This is UB, since we mark b's uninitialization with the UB attribute "indeterminate"

unsigned char e; // EB
unsigned char f = e; // EB, but now f gets "erroneous value"
assert(e == f); // This evaluates to true since both are erroneous
```

Note that in the above example, `c` can still get optimized away due to the UBness of `b` in the next line `char d = b + 1`; NOT by any fault of `a` or `c` (being undefined/assigned to undefined). EB is allowed to be done, AND recommended by the compiler to diagnose, garbage in garbage out.

### Homework

![Homework given at the end of lecture 1](image-2.png)
:::note
Submitting a patch for 1.1 will earn more points
:::

#### Solution for H.W[1.1]

TBD

#### Solution for H.W[1.2]

:::warning
I am not entirely sure if this answer suffices all expectations.
:::

```cpp
#include <iostream>
#include <limits>

template<typename T>
int what(T a) {
    return std::numeric_limits<T>::max();
}

void ujjwal() {
    int *a = nullptr;
    std::cout << what(*a);
}

void lecture() {
    int *a = nullptr;
    std::cout << typeid(*a).name();
}

int main() {
    ujjwal();
    lecture();  

    /* Output seen for Clang and GCC both
    2147483647i
    */  
}
```

[You may check the code generated for this here](https://godbolt.org/z/Ec4cPzqnq)

#### Solution for H.W[1.3]

```cpp
#include <limits>

constexpr int BAD_VAL = -1;

int foo_wrong(int *a, int base, int off) {
    if (off > 0 and base > base + off) {
        return BAD_VAL;
    }
    // Above got removed because overflow can never
    // happen and the condition is treated as false
    // and removed by UB blindness

    return a[base + off];
}

int foo_ok(int *a, int base, int off) {
    if (off > 0 and base > std::numeric_limits<int>::max() - off) {
        return BAD_VAL;
    }
    // Does not depend on the overflow to act, instead
    // the equation base + off > MAX is rearranged!

    return a[base + off];
}
```

[You may check the code generated for this here](https://godbolt.org/z/1Waf8eqc7)

### Extra essential viewing

:::note
At some point I realized that I wasn't very clear on the specific behavior of signed and unsigned overflows while watching Chandler's talk, I took a detour with the [blog on overflow by Ian](https://www.airs.com/blog/archives/120)
:::

#### Rabbit hole: Ian's blog on overflow and some more digging

> [Read it here](https://www.airs.com/blog/archives/120)

Signed integer overflow is considered UB by the standard. So, a C/C++ program that contains signed overflow is considered wrong. Thus, the compiler is free to assume that a given program will not contain signed integer overflow, and if it does anything can be done to it. OTOH, unsigned integer overflow is indeed defined behavior.

*But... why? they are either BOTH UB or BOTH IB/DB right?* No, there are a few caveats with the treatment of a signed integer. Different processors treat signed integer representation of a binary differently. Some use the 2's complement method, while some use 1's complement. Thus, the language standard cannot define a set behavior on signed overflow if the underlying hardware changes. With the unsigned integer, it is obvious that the same bit representation will be treated exactly the same across machines. Thus, overflow can be treated as a "wrap around" (modulo MAX + 1).

*No chance that we can define the signed integer overflow behavior?* Nope, the hardware differences are too wide. By fixing the behavior to one style, we would be antagonizing the users of hardware that is non-conformant to the standard. Plus, far too many optimizations exist due to UB which would otherwise be completely lost. For example here, `if (i + 1 < i) { ... }  // always false if overflow can't happen` can be replaced by the compiler to be `false` directly. If you use flags like `-fwrapv` or `-fno-strict-overflow` then indeed the behavior is fixed at a performance cost. Use builtins like `__builtin_add_overflow(a, b, &result)` (detect overflow safely).

#### CppCon 2016: Chandler Carruth “Garbage In, Garbage Out: Arguing about Undefined Behavior..."

> [Watch it on YouTube here](https://www.youtube.com/watch?v=yG1OZ69H_-o)

:::tip
The summary of the first half of the talk is essentially that *UB is NOT a compiler bug, it is a human violation of the contracts provided by the standard.
:::

That's it, we must start treating UB as a lack of the user's (read: programmer's) knowledge about the working of the language. When they break the contract that the language provides (e.g: dereference a null pointer), they made that error, and as specified by the standard, all bets are off. Sure, UB *could* allow the compiler to optimize the generated asm. However, it is still a valid behaviour even if the final behaviour of the code is not what the user wanted.

### Okay then, lets widen all of our contracts and define all behavior

#### Define all behavior

This problem is simply intractable. consider a function `Node findSink( const vector<Node*>& graph )`. If this is defined in a way that recurses to find a node that is a sink, then upon being given a graph which is cyclic any of the following could happen:

- The recursion forces the stack to grow to an overflow, and it gets killed by the OS
- The implementation is optimizable to a tail recursion, which causes there to be an infinite loop.
- Some node pointer corrupts due to the growing stack and a dereference goes on to hit a guarded page, leading to a segfault.

Can you define this behaviour? No you cannot, it has to be platform/hardware/compiler defined. Matter of fact, all three cases are (usually) possible to hit in most cases so the result is non-deterministic across different runs of the binary. Remember, UB does not guarantee consistent behaviour across runs -- all bets are off.

#### Widen all contracts

In the previous example, the only way to do that would be to add a runtime pre-condition contract that first processes the graph to find a cycle. This needs extra memory, this needs extra runtime performance cost. Not all users are going to be happy with that. In fact, such a pre-condition might be essentially impossible to write. To add a pre-condition that disallows inputs going into an infinite loop is reducible to solving the halting problem. So, we are stuck with a logical bottleneck to eliminating UB.

### Pay for what you use: Narrow contracts when we need them, are okay

Principles for narrow contracts:

- Checkable (probabilistically) at runtime
- Provides immense value as a trade-off
- Should not break existing code (or coding practices that are widely accepted)

TBD

#### CppCon 2017: Piotr Padlewski “Undefined Behaviour is awesome!”

TBD

## Lecture 2: Strings

:::tip
THE best and greatest example of generic programming are string...
:::

### History

Whether you write a "hello world" program in 2025, in the most bleeding edge module and `std::print` syntax or in the 1983 `std::printf` version; the string literal continues to remain common. In fact, the C-style string literals put withing double quotes are used till date. Why?

### What are strings exactly?

Initially, the notion to "pass" strings around would require some way of knowing what the string was. Imagine encoding strings with their length attached as a prefix to them. This would require some kind of a compiler magic type that would define the interaction of this string type with all other types in the standard.

Instead, the compiler devs of the time came up with a brilliant idea. Null-terminated strings and treating string literals as `const char*`. In fact, EVERY suffix of the string was automatically another `const char *` by design, and this only took one extra character worth of memory to null-terminate itself thus, not requiring a prefix-ed length. This solution is really cool, there are often caveats like the one shown below.

```cpp
auto t = "Hello, World";
static_assert(sizeof(t) == 8); // t is a const char *
static_assert(sizeof("Hello, World") == 14); // the passed in thing is a null-terminated string literal
// This is because the string literal would be stored as an array of bytes in the .rodata section of
// the code. The sizeof thus finds the number of elements present in this array including the null-term
// When we store the const char* in a variable the variable can simply point to it now.
```

### Suprises and C-style strings

Seemingly "safe" and "common" functions like the `strcpy` function are in fact VERY unsafe. This is because passing in a non null-terminated string into the src ptr will make the implementation endlessly look for a null termination character that makes the program behave badly.

#### Band-aid solution

Make a "safe" `strcpy` which also accepts a limit value; i.e., `strncpy`. Really? We still can't make a `strnlen` right? What would that do? Give up after "n" characters? In fact, this still doesn't help improve the security at all!
![humorous xkcd panel shows that a user can just pass in a large "n" value and get a raw output that shows other things in the buffer.](image-3.png)

Begs the question, why do we not get guarantees with our C-string API (if you can call it one). Simple, the C-string has no length invariant in it. We can add invariants simply using...

#### OOPS `:O`

We can preserve object invariance, by writing length as a private member of the string that can only be modified by the members of the object. Specifically, encapsulation is what allows this.

There are MANY (read: MANYYYYY) string classes that have been written. For this reason, we abandon the idea to rewrite the string class any better than the current alternatives to `std::string`; we study `std::string` and how it is built.

### Deep dive into the `std::string` class

Fundamentally similar to what `std::vector<>` does. A pointer to manually managed memory which contains the real payload. And a `size` variable to track the real length alongwith a `capacity` variable to track the remaining allocated buffer on the managed memory. More convenient is that the string class itself automaticall manages memory, and owns the buffer.

:::warning
Counterintuitively, we still store a null-terminating character at exactly `buffer[size]`. This is for the reasons of backwards compatibility. Infact, `string.c_str()` returns a `const char*`.

Also, please don't use `string.data()`, since it returns a simple `char*` allowing the internal pointer to be modified without encapsulation (DANGEROUS).
:::

#### Suspicious stuff here

Yes, the string class owns the buffer. Allocations on the buffer will be done on the managed memory... usually the heap. What happens if we make these allocations BEFORE `main()`? Worse, what happens if there's a `bad_alloc()` BEFORE `main()`? The example below shows a case where this is indeed possible.

```cpp
static const std::string s =
    "String so big that it won't "
    "fit as a simple list of symbols. "
    "Proceed with heap alloc :("

void foo(const std::string& arg) {
    /* blah blah with arg */
}

int main() {
    foo(s); // heap alloc seen BEFORE main()
    // leads to a jmp __abrt() in the asm. BAD
}
```

Okay, let's do a band-aid fix. What if, instead of a `static std::string`, we use a `static const char*`? The actual payload "String so big..." will be stored in `.rodata` and the the pointer `s` itself is stored on `.data/.bss`. But there's something even more annoying here.

Any time `arg` receives a `const char*`, it must temporarily materialize it in the body of `foo()` for each call. Every call of `foo()` does an alloc and then a dealloc after arg goes out of scope. This is a serious performance penalty.

#### `std::string_view` solves this!

It is a non-owning pointer to a string. Thus, the following code does not have any heap allocations before main, and calling `foo()` does not temporarily materialize heap-allocated objects every call.

```cpp
static constinit const std::string_view s =
    "String so big that it won't "
    "fit as a simple list of symbols. "
    "Proceed with heap alloc :("

void foo(const std::string_view arg) {
    /* blah blah with arg */
}

int main() {
    foo(s); // No heap alloc, no performance penalty
    // The actual owner is the static storage duration
    // Essentially, .rodata
}
```

:::tip
We throw in a `constinit` qualification to our `static std::string_view`. This guarantees that if the compiler is unable to use compile-time initialization for this, it will throw an error. Essentially, this initialization will be done before any dynamic initialization; and will error out at compile-time if it is not possible to do so.
:::

### Deep dive into `std::string_view`

It simply contains a pointer to the to the beginning of the view, alongwith a `size` that denotes the size of the view. However, the pointer to the beginning of the view does NOT own the memory being pointed to, and is thus, not responsible for the memory management of the same.

#### Suspicious stuff here (again!)

Unfortunately, the `std::string_view` (and other classes like it) pretends to be a value. It isn't. What happens when the value dangles? Can a value itself be an xvalue? Logically, it shouldn't. Consider this:

```cpp
std::string_view sv_bad = std::string("Hello World");
```

`std::string("Hello World!")` is an xvalue since in the very next line, this value reaches the end of its lifetime. But `sv_bad` must bind to it. This breakes value semantics! Many other classes exist with value semantics but _can_ dangle.

:::tip
A Rule of Thumb for `view`-like types

Use `std::string_view` in the following cases only:

- As a function parameter: `std::string identity(std::string_view sv) { return sv; };`. This is a safe option since the actual owner is outside the scope. The caller guarantees no-dangling.
- As for-loop initializers: `for (std::string_view e : elems) { /**/ };`. Again, the owner of each e is outside the scope of the for-loop.


If you see `std::string_view` anywhere else, beware! Double check the lifetimes.
:::

### Performance notes

#### CoW

Often, there are copies of the same string in the program, even in the simple case shown below. The same string "Hello\0" is stored in memory twice. Wasting the memory and some performance on extra allocations and deallocations.

```cpp
std::string s1 = "Hello";
std::string s2 = s1; // Invokes a copy constructor.
```

A typical optimization possible here is "CoW" a.k.a. "copy on write". Only increment refcounts on a control block that actually owns the string. When something is edited, the refcount reduces and the actual allocation of a new control block is done. Otherwise, only refcounts are added or subtracted and the string points to the control block.

This doesn't solve everything though. The memory savings come at the cost of an extra pointer indirection. The fewer allocations comes at the cost of thread safety issues and atomic operation requirement on the control block's refcount.

However, there is a reason that CoW strings aren't in the standard anymore: **pointer invalidation**. Very basic operations can invalidate pointers with CoW string!

```cpp
cow::string s("str");
const char* p = s.data();

{
    cow::string s2(s);
    s[0] = 'S'; // CoW triggered!
}

// No guarantee that the p is still valid due to
// reallocation
// For non-CoW strings, p would have been valid.
std::cout << *p << '\n';
```

Editing a single character in a copy of the original string, invalidated the pointer to the original string! This is why the standard in C++11 changed and forbade the `operator[]` from invalidating poiners to the string.

#### SSO

Well, the string block itself has a pointer and two unsigned ints. That consumes around 24 bytes (compiler/hardware dependent behaviour). Why not store really tiny strings (i.e., within 24 bytes) on the stack? This *could* be implemented as a union with size checks ensuring the right dispatch. However, GCC strings (ver >= 5) store the `std::string` as shown below.

![Memory layout of GCC's string for version >= 5](image-4.png)

### A final implementation wrinkle

Since we can store "x bytes" in memory, different strings like UTF-8 and UTF-32 might need repeated implementations. Matter of fact, the 89 public member functions (as mentioned in the video) would have to repeated for each. Enter: generic types!

We will make a `basic_string<CharT>` type that cares only about bytes. And then all other types become conveniences, `using string = basic_string<char>` and `using u16string = basic_string<u16chat_t>` among others.

Moreover, to decouple characted-level semantics (how are characters compared? how does `to_upper()` work?) from the container-level semantics (manage the data pointer, size etc); we pass in another policy class `basic_string<CharT, Traits>`. This can allow us to write custom traits that can, for example, use vectorized instructions for certain traits, or obfuscate credit-card numbers automatically while reading them; etc.

:::caution 
Allocation of a single character is not a trait of the char itself. It is that of the container since that determines WHERE will the character be allocated.
:::

Which means that even the allocator must be separated! The final, real class becomes `basic_string<CharT, Traits, Allocator>`. This is super important because often, the default heap-based allocators might not be the best for a given workload; and swapping out a different allocator would be as simple as passing in a different `Allocator` templ arg.

### Homework

#### Write your own CoW-string template class. Measure advantage over `std::string` on some benchmarks
TBD

#### Write a template class string_twine for O(logN) concatenation of several `string_view`s
TBD

#### Compilers are optimizing `std::string` worse than `std::vector`. Investigate [here](https://godbolt.org/z/Tfh6zfa6P)

TBD

### CppCon 2016: Nicholas Ormrod “The strange details of std::string at Facebook"

> [Watch it on YouTube here](https://www.youtube.com/watch?v=kPR8h4-qZdk)

GCC (version < 5) string is actually only a single data pointer... Where are the other details then? BEHIND the actual payload of the string! Moreover, for the empty string, GCC maintains a global variable that is a 25 byte array of zeroes that is the actual empty string.

![gcc old string implementation visualized as a single data pointer pointing to the start of payload AND the end of size/capacity/refcount](image-5.png)

Andrei Alexandrescu suggested the fbstring, which does SSO (with the data/size/capacity) triplet in the stack now. It stores the remaining capacity at the end. The beauty of this design is that when all 23 bytes are occupied, and the 24th byte is the null-terminator. The value of the remaining capacity doubles as the null-termination: 0!!

I don't even know how to describe the insane jemalloc + kernel + null terminator UB bug. I'll just rewatch the video from that timestamp [here](https://youtu.be/kPR8h4-qZdk?si=K5_Ry-JRWi_lWhXf&t=1260). 

Conclusion? Strings are a lot richer than what we imagine:) . And that the next great string implementation will come with its own trade-offs.

## Lecture 3: Overload sets

:::tip
The first step is to get the algorithm right. The second step is to figure out which sorts of things (types) it works for:p
:::

### Illustrative example: Raising a number to a power

We intend to get the algorithm right first (binary exponentiation)

```cpp
unsigned nth_power(unsigned x, unsigned n) {
    unsigned acc = 1;
    if (x < 2 || n == 1) return x;
    while (n > 0) {
        if (n & 0x1) {
            acc = acc * x;
            n -= 1;
        }

        x = x * x;
        n >>= 1;
    }

    return acc;
}
```

#### Naive generalization: templated type of "x"

```cpp
template <typename T>
T nth_power(T x, unsigned n) {
    T acc = 1;
    if (x < 2 || n == 1) return x;
    while (n > 0) {
        if (n & 0x1) {
            acc = acc * x;
            n -= 1;
        }

        x = x * x;
        n >>= 1;
    }

    return acc;
}
```

While this solutions "feels" like it is more generic than the hardcoded unsigned type. It fails due to two glaring issues. First, is the issue of the `T` which accepts just about any type. However, not all types can support the operations that `nth_power` needs for `T`, like the initial value of `acc` isn't necessarily `1`.
For example, a `Matrix` class would require `acc` to be an identity matrix instead of a scalar value. Second, the check `x < 2` doesn't logically work for `signed` types either (`-2 ^ 10 != -2`).

#### Improving the naive generalization

To improve this, we assume is that all `T` will have an identity type associated with it (`id<T>`).

```cpp
template <typename T>
T nth_power(T x, unsigned n) {
    T acc = id<T>();
    if (x == acc || n == 1) return x;
    
    /*Same as before*/
}
```

This is a really strong assumption though. You can't assume global functions to be required unless they are extremely standard. For example, it is alright to require a `.begin()` since it is very idiomatic in C++. But assuming the existence of such an odd function is not.

Can we solve this by making `id<T>` a part of the type_traits? Well this runs into a similar issue. It is better than having a global requirement and it also decouples the existence of the identity from the type itself and puts it into a `Traits` template parameter. But this isn't how the library solves its problems. Think: Where else do we need an identity type in the standard library, how does it solve the identity issue?

#### Inspired from `std::accumulate`

The standard handles this by passing in the identity value as a function argument to separate these concerns from the type itself.

```cpp
template <typename T>
T nth_power(T x, T acc, unsigned n) {
    while (n > 0) {
        if (n & 0x1) {
            acc = acc * x;
            n -= 1;
        }

        x = x * x;
        n >>= 1;
    }

    return acc;
}
```

Also, to improve the quality of usage, we can define full specializations of more commonly used types like `unsigned`:

```cpp
template <typename T>
T nth_power(T x, T acc, unsigned n) {
    /* Same as before */
}

unsigned nth_power(unsigned x, unsigned n) {
    if (x < 2u || n == 1u) return x;
    return nth_power<unsigned>(x, 1u, n);
}
```

:::tip
Notice how the "clean" part of the algorithm gets separated out into a generic function with very little casework. The casework that is type specific, gets handed out to full specializations of wrapper calls.

This is why the building blocks of generic programming are these **overload sets**!
:::

We notice examples of overload sets being used in the std library all the time. As shown here, the overload sets of the comparison function of `const char*` with `basic_string<...>` avoid an extra copy of the string literal.

![stdlib example of the library using overload sets to save an extra copy for basic_string comparison](image-6.png)

### General principles for the design of overload sets

#### Examples of good design

Different but related types: Ensures consistent behaviour and ensures that the compiler is not creating a temporary `std::string`.

- `void foo(const char* s)`
- `void foo(std::string s) { Foo(s.c_str()); }`

Different number of parameters

- `auto s1 = twine("Hello", name).str();`
- `auto s2 = twine("Hello", name, " ", surname).str();`

Optimizations

- `void vector<T>::push_back(const T&)`
- `void vector<T>::push_back(T&&)`

#### Some rules of thumb

- A person shouldn't need a deep understanding of the C++ overload resolution or of the C++ type system to simply use the function.
- Noone should be forced to do deep overload resolution in their head.
- Each overload set should **roughly** do the same thing.
- Try to encode requirements of the generic function using `requires` constraints.
- You may create overload sets using `requires` clauses on each type. SFINAE will pick the appropriate overload, and if nothing works, each failing condition will be displayed which is good for diagnostics.

### Fun fact function name mangling

Look at the overloads of `Foo` shown below. They should be mangled to the same name and lead to a redifinition or ambiguity error. Instead, this is perfectly normal and behaves as expected.

```cpp
template <typename T>
requires (sizeof(T) > 4)
void Foo(T x) {}

template <typename T>
requires (sizeof(T) <= 4)
void Foo(T x) {}
```
 
This works because according to the standards two functions with the same name are considered to be equivalent iff:

- They are in the same namespace.
- Their parameter sets are equivalent.
- **Their trailing requires clauses are equivalent.**

The compiler treats the requires clause as an integral part of the function interface but NOT as part of the mangled function symbol. This is done by simply checking for viable overloads by the compiler. In this case, the two versions are for mutually exclusive types. Thus, **for a given `T`** only one definition is valid. So even though `nm` should in theory show the same mangled name, the linker doesn't care since for a given type only one type would be valid.

#### Compiler doesn't know that a condition is "more restrictive"

If an overload requires a condition $A$ to be true, while another overload requires a condition $B \sub A$ to be true. The compiler will find both sets to be viable overloads but it won't know that the latter is more restrictive (*it is easy to confuse this with specializations of a template*).

In fact, using concepts is even better here! Complex constraints can be used to check *validity* of an expression as opposed to the boolean evaluated values.

```cpp

template <typename T>
consteval int somepred() { return 67; }

// simple constraint
template <typename T>
requires (somepred<T>() == 42)
bool foo(T&& lhs, T&& rhs);

// complex constraint
template <typename T>
requires requires(T t) { somepred<T>() == 42; }
bool foo(T&& lhs, T&& rhs);
```

The first one will evaluate the requires clause at compile time and emit false. The second one however simply checks that the enclosed expression is indeed valid. Since the expression is valid (regardless of its boolean-ness), the clause is satisfied!

:::caution
I don't think I am clear on the four types of requires clauses and how they differ. Honestly, the syntax is throwing me off a bit so I will return to this later using maybe some other video!
:::

### concepts

There are 4 types of requires expressions: simple, type, compound and nested. To make these long `requires` clauses more readable and clean, C++20 introduces `concept`. These concepts can be given names, making their intent very clear. It is a compile-time predicate, so it doesn't need to be *called* like a function. The `concept` itself is a value!

:::important 
Recursive concepts and constraints on concepts by other predicates are not allowed. This is *probably* done to disallow two sources of Turing complete programs (meta programming) by the committee.

We can still put other predicates on a concept using conjuction and mitigate it, but explicit recursion is simply disallowed on concepts.
:::

Moreover, we can constrain member functions and even constructors inside a class using concepts/requires. Compiler can also partially order concepts to help pick the "most restrictive" viable overload set. How does compiler understand partial specialization? How does it understand that `Ord` is more restrictive than `Ord || Void`.

### Compiler's understanding of `concept`'s constraints

All `concept`s are a series of atomic constraints joined by logical operations (`||`, `&&`; etc.) **with the short-circuiting rules working as usual**. The compiler is able to determine a `subsumes` relation between the constraint sets of two viable overload sets. The set that subsumes all others is automatically the most restrictive viable overload set.

Ideally, we would have preferred to see the subsumes condition to be that: `P subsumes Q <=> (Q => P)`. However, we are not in the real world, and semantic meaning is often going to be too hard for the compiler to determine; leading to false negatives on ill-formed types. In the old RFCs for `concepts` they didn't just account for syntactice, but also semantic requirements using keywords like `axioms`.

### Homework

#### Design a realistic overload set for generic function `nth_power` using constraints. Account for integers, floats and matrices.

TBD

#### Try to figure out what is going wrong [here](godbolt.org/z/Kq1GG7hr)

TBD

![homework problem 3.2](image-7.png)

### Suggested reading

I have noted them down in decreasing order of relevance for myself.

#### Nicolai Josuttis, Back to basics: concepts, Cppcon 2024

#### Andrew Sutton, Concepts in 60, Cppcon 2018

#### Titus Winters, Modern C++ design (2 parts), Cppcon 2018
