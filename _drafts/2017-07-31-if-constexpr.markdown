---
layout: post
title:  "C++17 is Big for Generic Programming"
date:   2018-07-31 12:00:00
categories: c++ cpp template metaprogramming typelist
comments: True
permalink: posts/if-constexpr-tmp
shortUrl: https://goo.gl/LdRg45
is_post: true
---

I've noticed that C++17 is sometimes criticized and pinned as being a
small release. Sure, we didn't get Concepts, Modules, or Ranges,
but that doesn't mean the features we did get won't change the way
we write code in a dramatic way. In this post I'm going to focus
on `if constexpr` and how this completely remodels how we can and
should write generic code.

One very common problem is iterating over a parameter pack, such as the
elements in a tuple. A generic algorithm for iterating over a tuple
can be written, but this is not as useful and common as I'd like.
Unfortunately the result is that we are often stuck with writing the
recursion by hand. Let's compare how we can do this in C++11/14 vs.
C++17. In C++11/14 we have [^1]

```cpp
template <size_t Count = 0,
          typename... Elements,
          typename std::enable_if_t<Count == sizeof...(Elements)>* = nullptr>
constexpr void iterate_tuple_11(
    const std::tuple<Elements...>& tupull) noexcept {}

template <size_t Count = 0,
          typename... Elements,
          typename std::enable_if_t<Count != sizeof...(Elements)>* = nullptr>
constexpr void iterate_tuple_11(const std::tuple<Elements...>& tupull) {
  std::cout << "Element: " << Count << " " << std::get<Count>(tupull) << "\n";
  iterate_tuple_11<Count + 1>(tupull);
}

void example_iterate_tuple_11() {
  std::cout << "\nC++11/14:\n";
  iterate_tuple_11(std::make_tuple("first", 2, 3.14));
}
```

In C++17 this can be simplified using `if constexpr` as follows

```cpp
template <size_t Count = 0, typename... Elements>
constexpr void iterate_tuple_17(const std::tuple<Elements...>& tupull) {
  std::cout << "Element: " << Count << " " << std::get<Count>(tupull) << "\n";
  if constexpr(Count + 1 != sizeof...(Elements)) {
    iterate_tuple_17<Count + 1>(tupull);
  }
}

void example_iterate_tuple_17() {
  std::cout << "\nC++17:\n";
  iterate_tuple_17(std::make_tuple("first", 2, 3.14));
}
```

Let's now look at the interesting example where we want the body
of `func` to change depending on some property of the current
element being operated on. We've already gone over iterating over a tuple
so let's just focus first on "changing" the body of `func` depending
on some type property. We can achieve this sort of behavior pre-C++17
using `std::enable_if`, but it's a lot more work than what we do here.


```cpp
struct single_call {
  void operator()(const double) { std::cout << "single call\n"; }
};
struct double_call {
  void operator()(const double, const double) { std::cout << "double call\n"; }
};

int main() {
  auto lambda = [](auto func) {
    if constexpr(std::is_invocable_v<decltype(func), double, double>) {
      func(1.3, 8.9);
    } else {
      func(7.8);
    }
  };
  lambda(single_call{});
  lambda(double_call{});
  return 0;
}
```
[^1]: I use the C++14 type aliases
      `std::enable_if_t<bool, typename> = typename std::enable_if<bool, typename>::type`. 
      By making this substitution the code will be C++11 compliant.
