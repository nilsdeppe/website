---
layout: post
title:  "Variadic Using and How to Use It"
date:   2017-09-02 00:21:00
categories: c++ cpp template metaprogramming using variadic
comments: True
permalink: posts/variadic-using
shortUrl: https://goo.gl/Ckyz2c
is_post: true
---

C++17 allows parameter pack expansion inside `using` statements. This feature
is sometimes called *variadic-using* and the paper is
[here](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0195r2.html).
Unfortunately, I found searching for "variadic using" with Google to not be all
that helpful. In this post I will provide a concrete example of what this
feature can be used for, since it might seem like a bit of a strange idea at
first.

The idea for this post came about from wanting to use call operator overloading
in a lambda expression, which is not something provided by the language.
Basically, I would like to write a lambda expression with two call operators 
defined that take different arguments. I can do this for a struct, but that
cannot be done inline like lambda expressions. This problem is easily solved
with a *variadic-using*. Here is the code:

{% highlight cpp linenos %}
template <class... Lambdas>
struct overload : Lambdas... {
  explicit overload(Lambdas... lambdas) : Lambdas(std::move(lambdas))... {}

  using Lambdas::operator()...;
};

template <class... Lambdas>
overload<Lambdas...> make_overload(Lambdas... lambdas) {
  return overload<Lambdas...>(std::move(lambdas)...);
}
{% endhighlight %}

The `make_overload` is a helper function to do the type deduction for us,
similar in spirit to `std::make_tuple`.[^1] What line 5
(the *variadic-using*) does in pull in the call operator
of all the `Lambdas` into the scope of `overload` so that they are
available for overload resolution.

Now what this allows you to do is use overloading as a pseudo-SFINAE by
having the lambdas take as an argument a `std::integral_constant<bool, B>`
and having a type trait fill the value of `B` at the call site. Here
is some code that uses this:

{% highlight cpp linenos %}
#include <iostream>
#include <type_traits>

template <class... Lambdas>
struct overload : Lambdas... {
  explicit overload(Lambdas... lambdas) : Lambdas(std::move(lambdas))... {}

  using Lambdas::operator()...;
};

template <class... Lambdas>
overload<Lambdas...> make_overload(Lambdas... lambdas) {
  return overload<Lambdas...>(std::move(lambdas)...);
}

struct my_type1 {
  int func(int a) { return 2 * a; }
};

struct my_type2 {};

template <class T, class = std::void_t<>>
struct has_func : std::false_type {};

template <class T>
struct has_func<
    T, std::void_t<decltype(std::declval<T>().func(std::declval<int>()))>>
    : std::true_type {};

static_assert(has_func<my_type1>::value, "");
static_assert(not has_func<my_type2>::value, "");

template <class T>
void func(T t) {
  auto my_lambdas = make_overload(
      [](auto& f, std::integral_constant<bool, true>) {
        std::cout << "True " << f.func(2) << "\n";
      },
      [](auto& f, std::integral_constant<bool, false>) {
        std::cout << "False\n";
      });
  my_lambdas(t, std::integral_constant<bool, has_func<T>::value>{});
}

int main() {
  auto my_lambdas = make_overload(
      [](std::integral_constant<bool, true>) { std::cout << "True\n"; },
      [](std::integral_constant<bool, false>) { std::cout << "False\n"; });
  my_lambdas(std::integral_constant<bool, true>{});
  my_lambdas(std::integral_constant<bool, false>{});

  func(my_type1{});
  func(my_type2{});
}
{% endhighlight %}

This prints out:

{% highlight shell linenos %}
True
False
True 4
False
{% endhighlight %}

What you notice is that the type trait `has_func` is used on line 42, which
then calls the correct generic lambda in `my_lambdas`. Of course, you are not
limited to selecting between two lambdas, you can select between arbitrary
many and then use a single `std::integral_constant<int, I>` or multiple
`std::integral_constant<bool, B>` arguments. At the call
site things are little trickier since you need to do something like
`std::integral_constant<int, trait1<T>::value + 2 * trait2::value>` for
the former, and
`std::integral_constant<bool, trait1<T>::value>{},
std::integral_constant<bool, trait2<T>::value>{}` for the latter.

#### Summary

In this post we explored *variadic-using* declarations to allow SFINAE-like
behavior in generic lambdas. A similar implementation is possible in C++14
but that's for another post. Overloading for SFINAE-like behavior greatly
extends the usefulness and usability of lambda expressions, since
overloading of functions and SFINAE are common and powerful techniques
for developing generic code. The code I showed above is available on my
[GitHub](https://github.com/nilsdeppe/template-metaprogramming-tutorials)
with the name `variadic-using.cpp`.

[^1]: `make_overload` can be replaced with a deduction guideline, but that
      is not yet supported in a release build of Clang, so I am avoiding it
      for now.
