---
layout: post
title:  "Lambdas and Overloads and Type Traits, Oh My!"
date:   2017-09-10 12:00:00
categories: c++ cpp template metaprogramming
comments: True
permalink: posts/local-type-traits
shortUrl: https://goo.gl/HcgRxf
is_post: true
---

In this post I will discuss overloading generic lambdas in C++14, using SFINAE
with lambdas, using tagged dispatch for "pseudo-SFINAE", and finally using
overloading and SFINAE to create local type traits inside functions. All of
these are useful when writing generic code. Overloading generic lambdas,
for example, proves to be extremely useful when implementing a stream operator
for `std::tuple`. I'm sure you can think of reasons why a type trait is useful
only in the current scope and writing a `struct` elsewhere for it is overkill
(e.g. checking for the existence of a member function of a template parameter).

### Overloaded Lambdas

C++ provides us with the first ingredient needed for overloaded lambdas: the
lambdas. The second ingredient that we need is a method of overloading the call
operator of a lambda. Unfortunately, there is no language support for this
so we must implement this ourselves using existing language facilities. If we
have a class `overloader` that derives
off some other class, `B`, then we can bring `B`'s call operator into scope
with `using B::operator();`. Let me be concrete about what I mean:

{% highlight cpp linenos %}
template <class F>
struct overloader : F {
  using F::operator();
};
{% endhighlight %}

This gets us what we want for one non-function invokable (a lambda or function
object). However, we want to be able to overload the call operator N times,
so we need to use a variadic struct template---C++17 makes this
problem quite [easy]({% post_url 2017-09-02-variadic-using %}). In C++11/14
we can use recursion with fast-tracking
to implement `overloader`, and given that there will typically only be a
handful of overloads the recursive approach is fine. Let's see what this
looks like:

{% highlight cpp linenos %}
template <class... Fs>
struct overloader;

template <class F1, class F2, class... Fs>
struct overloader<F1, F2, Fs...> : F1, F2, overloader<Fs...> {
  constexpr overloader(F1 f1, F2 f2, Fs... fs)
      : F1(std::move(f1)),
        F2(std::move(f2)),
        overloader<Fs...>(std::move(fs)...) {}

  using F1::operator();
  using F2::operator();
  using overloader<Fs...>::operator();
};

template <class F>
struct overloader<F> : F {
  constexpr explicit overloader(F f) : F(std::move(f)) {}

  using F::operator();
};

namespace overloader_details {
struct no_such_type;
}  // namespace overloader_details

template <>
struct overloader<> {
  void operator()(overloader_details::no_such_type const* const  /*unused*/) {}
};
{% endhighlight %}

More fast-tracks can easily be added, e.g. for four and eight invokables.

Alright, let's walk through what is happening in the above code. First
we declare a variadic struct template, `overloader` on line 1. Next, on line 4
we provide a specialization that matches for two or more `F`s and derives off
`F1`, `F2`, and `overloader<Fs...>`. This is the simplest fast-track we can have:
it peels off two template parameters instead of one. The constructor receives
the invokables and moves them into the base classes. Finally, we bring the
call operators from `F1`, `F2` and `overloader<Fs...>` into scope by using
three using-declarations. Because we recurse through the parameter pack
`Fs...`, by bringing the call operators of `overloader<Fs...>` into scope
we also bring all of the call operators of `Fs...` into scope.

The second specialization on line 16 is for when there is only a single
invokable. The code is straightforward compared with the fast-track case.
The sentinel specialization on line 27 is different: we need to provide a call
operator so that `using overloader<>::operator()` is a valid expression.
However, this call operator overload should never be called and the best
way to ensure that is to have a pointer to an undefined (but declared) type.
If someone tries to create the type `overloader_detail::no_such_type` they
will be greeted with a compiler error.

I claim that the above will do exactly what we need: allow us to overload
the call operator of (generic) lambdas. Let's make life a little easier and
define the utility function `make_overloader`:

{% highlight cpp linenos %}
template <class... Fs>
constexpr overloader<Fs...> make_overloader(Fs... fs) {
  return overloader<Fs...>{std::move(fs)...};
}
{% endhighlight %}

The point of this utility function is to do the template argument deduction
for us. That is, it deduces the template parameters to `overloader`, the
`Fs...`, allowing us to simply pass them to the function. This is
especially useful for lambdas whose types are not so straightforward
to figure out. Okay, so now let's see this in action!

{% highlight cpp linenos %}
const auto lambdas = make_overloader(
    [](int a) { std::cout << "int: " << a << '\n'; },
    [](std::string a) { std::cout << "string: " << a << '\n'; });
lambdas(1);
lambdas("this is a string");
{% endhighlight %}

This will print out

{% highlight text %}
int: 1
string: this is a string
{% endhighlight %}

### SFINAE With Overloaded Lambdas

When writing generic code we sometimes want/need to
use SFINAE with a lambda. One example is when writing a stream operator for
`std::tuple` where it is necessary to support types that do not have a stream
operator defined. Let's use SFINAE to select an overload depending on whether a
class has a member `func(int)`, for which we can easily write a type trait:

{% highlight cpp linenos %}
template <class...>
using void_t = void;

template <class T, class = void_t<>>
struct has_func : std::false_type {};
template <class T>
struct has_func<T,
                void_t<decltype(std::declval<T>().func(std::declval<int>()))>>
    : std::true_type {};
    
template <class T>
constexpr bool has_func_v = has_func<T>::value;
{% endhighlight %}

I'm assuming you are familiar with writing type traits like this, but if not
then I explain a litte bit of what's happening with the `decltype()` call on line
8 in the section
["From Overloaded Lambdas to Type Traits"](#from-overloaded-lambdas-to-type-traits)
below. To use the type trait for SFINAE of an overloaded lambda we use the
trailing return type syntax as follows:

{% highlight cpp linenos %}
template <class T>
void check_for_func_member(T t) {
  constexpr auto my_lambdas = make_overloader(
      [](auto s) -> std::enable_if_t<has_func_v<decltype(s)>> {
        std::cout << "Has func(int) member using SFINAE\n";
      },
      [](auto s) -> std::enable_if_t<not has_func_v<decltype(s)>> {
        std::cout << "Has no func(int) member using SFINAE\n";
      });
  my_lambdas(t);
}
{% endhighlight %}

The return type is always `void` but which function is called depends on whether
the first or second overload has a substitution failure in the return type.
The resulting behavior is the same as using SFINAE with any regular function.

There is another clever way to have what I call "pseudo-SFINAE" with lambdas.
In this case we write:

{% highlight cpp linenos %}
template <class T>
void check_for_func_member_overload(T t) {
  constexpr auto my_lambdas = make_overloader(
      [](auto s, std::true_type /*meta*/) {
        std::cout << "Has func(int) member using pseudo-SFINAE\n";
      },
      [](auto s, std::false_type /*meta*/) {
        std::cout << "Has no func(int) member using pseudo-SFINAE\n";
      });
  my_lambdas(t, typename has_func<T>::type{});
}
{% endhighlight %}

We select which overload we call based on what type
`typename has_func<T>::type` is. If it is `true_type` we select the first,
and if it is `false_type` we select the second. This is effectively tagged
dispatch--using metaprogramming to select which function to resolve to.
This approach has the major advantage that it is, in my opinion, much easier to
understand than the SFINAE case.

### Understanding The Overloaded Lambdas

We are almost ready to get to the fun stuff---metaprogramming. Before that we
need to understand a little bit more about generic lambdas and overloading.
One thing you may not be aware of is that you can use the trailing return type
syntax to specify the return type of a lambda. What I mean is you can write
`[]() -> double { return 1; }` and the return type will be `double`, not `int`.
We will now (ab)use this to write local type traits. First let's write
a trait that checks for the existence of a member function `func(int)`.
Here is the implementation of the type trait:

{% highlight cpp linenos %}
constexpr auto has_func = make_overloader(
    [](auto t, int) -> decltype(
        (void)std::declval<decltype(t)>().func(std::declval<int>()),
        std::true_type{}) { return std::true_type{}; },
    [](auto...) { return std::false_type{}; });
{% endhighlight %}

You are probably wondering what is happening here. First, we are
creating an `overloader` of two lambdas. Let's talk about the lambda
on line 5 first. The lambda

{% highlight cpp linenos %}
[](auto...) { return std::false_type{}; }
{% endhighlight %}

translates to

{% highlight cpp linenos %}
struct LAMBDA_SECOND {
  template <class... Ts>
  std::false_type operator()(Ts...) const { return std::false_type{}; }
};
{% endhighlight %}

Similarly, the lambda on lines 2-4,

{% highlight cpp linenos %}
[](auto t,
   int) -> decltype((void)std::declval<decltype(t)>().func(std::declval<int>()),
                    std::true_type{}) { return std::true_type{}; }
{% endhighlight %}

translates to

{% highlight cpp linenos %}
struct LAMBDA_FIRST {
  template <class T>
  auto operator()(T t, int)
      -> decltype((void)std::declval<T>().func(std::declval<int>()),
                  std::true_type{}) const {
    return std::true_type{};
  }
};
{% endhighlight %}

Thus, the resulting overloaded lambda is

{% highlight cpp linenos %}
struct LAMBDA_OVERLOADER {
  template <class T>
  auto operator()(T t, int)
      -> decltype((void)std::declval<T>().func(std::declval<int>()),
                  std::true_type{}) const {
    return std::true_type{};
  }

  template <class... Ts>
  std::false_type operator()(Ts...) const { return std::false_type{}; }
};
{% endhighlight %}

Hopefully it is now clear how to think about the overloaded lambdas.

I'll quickly point out that the `(void)` on line 3 of the first code block
in this section is necessary to avoid code injection from overloaded
comma operators. That is, if someone were to define
`void operator,(int, std::true_type) {}` then the expression passed to the
`decltype()` operator on line 2 and 3 of the first block in this section
will be ill-formed.

### From Overloaded Lambdas to Type Traits

Now that we have the required code, let's see what happens when we use the type
trait `has_func`. The simplest thing we can do is:

{% highlight cpp linenos %}
struct my_type1 {
  int func(int a) { return 2 * a; }
};
struct my_type2 {};

template <class Trait, class... Type>
constexpr bool local_trait_v =
    decltype(std::declval<Trait>()(std::declval<Type>()..., 0))::value;

template <class T>
void local_type_trait_example1(T /*t*/) {
  constexpr auto has_func_impl = make_overloader(
      [](auto t, int) -> decltype(
          (void)std::declval<decltype(t)>().func(std::declval<int>()),
          std::true_type{}) { return std::true_type{}; },
      [](auto...) { return std::false_type{}; });
  using has_func_member = decltype(has_func_impl);

  std::cout << "Has func(int) member function: " << std::boolalpha
            << local_trait_v<has_func_member, T> << "\n";
}
int main() {
  local_type_trait_example1(my_type1{});
  local_type_trait_example1(my_type2{});
}
{% endhighlight %}

which prints out

{% highlight text %}
Has func(int) member function: true
Has func(int) member function: false
{% endhighlight %}

First I'll explain `local_trait_v` (line 7), which is simply a helper
`constexpr` variable used
to reduce the amount of typing necessary to evaluate local type traits. It
takes the trait to evaluate as its first template parameter and the types to
pass to the trait as the parameter pack. Because lambdas cannot
appear in an unevaluated context we must declare the variable `has_func_impl`
rather than having the `make_overloader` call be inside the `decltype()` call
on line 17.

You should now be able to follow the call to `local_trait_v` on line 20 back to
overload resolution of the call operator. So how does the compiler actually
decide which return type to deduce from the call operator? Well, if the
expression to `decltype()` in the trailing return type on line 13 is evaluable,
i.e. `decltype(t)` has a member function `func(int)`, then the first overload is
preferred if and only if the second argument to the call operator is an `int`.
This is why `local_trait_v` calls `(type, 0)`: the `0` matches `int`,
which is a better match than `auto...` in the call operator on line 16. However,
if the call were `(type, 0L)`, the `0L` would match `auto...` because then there
is no implicit cast to from `long` to `int`. If you have written a lot of type
traits then this trick will be familiar, though frequently C-style variadic
functions are used, which I'm intentionally avoiding.

### Another Local Type Trait Example

Another interesting class of type traits are the `is_`, for example
`is_std_map`. Let's write that one and use it! Here is the implementation:

{% highlight cpp linenos %}
void local_type_trait_example2() {
  constexpr auto is_std_map = make_overloader(
      [](auto t, int) -> std::enable_if_t<
          std::is_same<decltype(t),
                       std::map<typename decltype(t)::key_type,
                                typename decltype(t)::mapped_type,
                                typename decltype(t)::key_compare,
                                typename decltype(t)::allocator_type>>::value,
          std::true_type> { return std::true_type{}; },
      [](auto...) { return std::false_type{}; });

  std::map<int, double> b;
  std::unordered_map<int, double> c;
  std::vector<int> d;
  std::cout << "Is a map: " << decltype(is_std_map(b, 0))::value << "\n";
  std::cout << "Is a map: " << decltype(is_std_map(c, 0))::value << "\n";
  std::cout << "Is a map: " << decltype(is_std_map(d, 0))::value << "\n";
}
{% endhighlight %}

The trait checks that the member aliases `key_type`, `mapped_type`,
`key_compare` and `allocator_type` exist, and then that a `std::map`
with the same template parameters is the same type as what was passed in to `t`.
This is definitely not the most common way of implementing the type trait if
one were using a `struct`, but the result is perfectly adequate and works
within the restrictions of overloaded lambdas.

### Summary

In this post we explored how to overload the call operator of a (generic)
lambda, use SFINAE and tagged dispatch for "pseudo-SFINAE" in overloaded
lambdas, and (ab)use these to implement type traits locally within
functions. The result is several different ways of performing fairly complex
metaprogramming tasks within a narrower scope than was previously possible.
This helps to simplify the amount of code that needs to be analyzed when
reasoning about what a function does. The
biggest gain from these methods is for library implementers who write a lot
of generic code. I have shared a complete, working example of the code
snippets shown here on my
[GitHub](https://github.com/nilsdeppe/template-metaprogramming-tutorials)
in the file `local-type-traits.cpp`.

I hope you enjoyed the post and thanks for reading!
