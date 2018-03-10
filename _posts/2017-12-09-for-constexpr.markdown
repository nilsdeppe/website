---
layout: post
title:  "The Missing for constexpr"
date:   2017-12-09 12:00:00
categories: c++ cpp template metaprogramming
comments: True
permalink: posts/for-constexpr
shortUrl: https://goo.gl/xEnTMR
is_post: true
---

With C++17 we finally get `if constexpr`. However, I have started running
into a lot of cases where I want a `for constexpr` that is guaranteed to be
evaluated at compile time. My use case is effectively compile time
indexing of a multi-index `tuple`-like datastructure. This means that
arbitrary nesting (within instantiation depth limits at least) must be
possible, and it must be possible to have an inner for loop index depend
on an outer for loop index. We will use C++14 to implement `for constexpr`.

### Interface Design {#interface_design}

First let's think about the interface. A function template named
`for_constexpr` which takes an invokable object as its only argument
seems like a good start.
We pass `std::integral_constant`s of the current loop iteration indices
to the invokable object, which means generic
lambas will work with our interface. What all that basically means
is something like:

{% highlight cpp linenos %}
for_constexpr(
    [](auto I, auto J, auto K) {
      std::cout << I << ',' << J << ',' << K << "   " << I + 3 * (J + 3 * K)
                << '\n';
    });
{% endhighlight %}

Now we need to figure out how to set the bounds for the nested for loops
and how to determine how many nestings there are. Ideally I'd like
something like the following:

{% highlight cpp linenos %}
for_constexpr<for_bounds<0, 3>, for_bounds<0, 3>, for_bounds<0, 2>>(
    [](auto I, auto J, auto K) {
      std::cout << I << ',' << J << ',' << K << "   " << I + 3 * (J + 3 * K)
                << '\n';
    });
{% endhighlight %}

You might look at this and ask
"Why not just pass the index bounds as `<0, 3, 0, 3, 0, 2>`?" Well, consider a
different case where we only want the third index to loop from `J` to 2. How do
we specify that case?
That is, how do we write the loop structure:

{% highlight cpp linenos %}
for (size_t i = 0; i < 3 ++i) {
  for (size_t j = 0; j < 3 ++j) {
    for (size_t k = j; k < 2 ++k) {
      // Do stuff here
    }
  }
}
{% endhighlight %}

By passing a parameter pack of `for_bounds` we can intersperse `for_symmetric`
types. A `for_symmetric`'s first template parameter is the numerical index
(starting from 0) of the index we are symmetrizing over, and the second the upper bound. That is,
we would write the above nested for loop construct as:

{% highlight cpp linenos %}
for_constexpr<for_bounds<0, 3>, for_bounds<0, 3>, for_symmetric<1, 2>>(
    [](auto I, auto J, auto K) {
      std::cout << I << ',' << J << ',' << K << "   " << I + 3 * (J + 3 * K)
                << '\n';
    });
{% endhighlight %}

We can similarly symmetrize over `I` using `for_symmetric<0, 2>`, or
symmetrize the `J` index over `I` using:

{% highlight cpp linenos %}
for_constexpr<for_bounds<0, 3>, for_symmetric<0, 3>, for_bounds<0, 2>>(
    [](auto I, auto J, auto K) {
      std::cout << I << ',' << J << ',' << K << "   " << I + 3 * (J + 3 * K)
                << '\n';
    });
{% endhighlight %}

Okay, we now have a nice interface that is scalable to any number of nested
loops. What else do we need to consider? What about for loops that count
down? Well, the user can just use `N - 1 - I` instead of `I` in the lambda
(here `I` goes from `0` to `N - 1`). So at least our interface seems
to be able to handle the type of
loops we are interested in, now we just need to implement the thing.

### Single Loop

Before writing a bunch of template-heavy C++ we should write down our
requirements.

1. As non-recursive as possible. Function recursion depth is
$$\mathcal{O}(N)$$ where $$N$$ is the number of nested loops. Generating
the `index_sequence` should be recursive to a depth of
$$\mathcal{O}(\log(M))$$ where $$M$$ is the number of values in the sequence.

2. Bounds must not be required to start at zero.

3. A range of 10 numbers starting at one million must be as efficient
   as a range of 10 starting at zero.

4. Zero runtime recursion.

5. Able to specify symmetrized loops.

6. Avoid `std::enable_if`

Alright, so now that we have some constraints let's first write `for_bounds`
and `for_symmetric`. These are implemented as:

{% highlight cpp linenos %}
template <size_t Lower, size_t Upper>
struct for_bounds {
  static constexpr const size_t lower = Lower;
  static constexpr const size_t upper = Upper;
};

template <size_t Index, size_t Upper>
struct for_symmetric {};
{% endhighlight %}

Let's start our `for_constexpr` journey gently: a single for loop,
no nesting, but arbitrary ranges.

{% highlight cpp linenos %}
namespace for_constexpr_detail {
template <size_t lower, size_t... Is, class F>
void for_constexpr_impl(F&& f,
    std::index_sequence<Is...> /*meta*/) {
  (void)std::initializer_list<char>{
      ((void)f(std::integral_constant<size_t, Is + lower>{}),
       '0')...};
}
}  // namespace for_constexpr_detail

template <class Bounds0, class F>
void for_constexpr(F&& f) {
  for_constexpr_detail::for_constexpr_impl<Bounds0::lower>(
      std::forward<F>(f),
      std::make_index_sequence<Bounds0::upper - Bounds0::lower>{});
}
{% endhighlight %}

Let's go over what is happening here. As always seems to be the case
recently, the code is rather dense. The template parameters to
`for_constexpr` are hopefully somewhat straightforward. The first is
a `for_bounds` that must be explicitly specified, while the second is the
deduced invokable. The `for_constexpr` function forwards on information
to the `_impl`
and generates an `index_sequence` from 0 to $$M - 1$$ where $$M$$ is
`Bounds0::upper - Bounds0::lower`. This ensures that our second requirement
above is satisfied. The `_impl` unpacks the `index_sequence` into an
`initializer_list`, which is guaranteed to be evaluated left to right, so
our index value is increasing upon each call. Adding `lower` bound to `Is`
is necessary to increase the `Is` back to the range the user specified.
And that's it. That's a single loop implementation of `for constexpr`
that allows arbitrary ranges and is effectively non-recursive.

### Nested Loops and Symmetrizing

Now for the really fun stuff: nesting and symmetrizing loops. Let's first
look at the generalized `for_constexpr` before dealing with the `_impl`.
It turns out we need two overloads: the single loop case and the multi-loop
case (you'll see why when we write the `_impl`s). These are:

{% highlight cpp linenos %}
template <class Bounds0, class F>
 void for_constexpr(F&& f) {
  for_constexpr_detail::for_constexpr_impl<Bounds0::lower>(
      std::forward<F>(f),
      std::make_index_sequence<Bounds0::upper - Bounds0::lower>{});
}

template <class Bounds0, class Bounds1, class... Bounds, class F>
 void for_constexpr(F&& f) {
  for_constexpr_detail::for_constexpr_impl<Bounds0::lower, Bounds...>(
      std::forward<F>(f), Bounds1{},
      std::make_index_sequence<Bounds0::upper - Bounds0::lower>{});
}
{% endhighlight %}

In the nested loop case (bottom) we peel off the first two indices immediately,
but handle them differently. The first index is immediately looped over, while
the second (`Bounds1`) is passed as the second argument to the `_impl`. The
reason for that is this way we can overload `_impl` on whether `Bounds1` is
a `for_bounds` or a `for_symmetric`. Other than that, both are the same
except that the nested version also forwards the remaining `Bounds...` as a
parameter pack.

The `_impl`s are a bit scarier this time around. Let's first look at
just the `for_bounds` implementations:

{% highlight cpp linenos %}
// Base case
template <size_t lower, size_t... Is, class F, class... IntegralConstants>
ALWAYS_INLINE constexpr void for_constexpr_impl(
    F&& f, std::index_sequence<Is...> /*meta*/, IntegralConstants&&... v) {
  (void)std::initializer_list<char>{
      ((void)f(v..., std::integral_constant<size_t, Is + lower>{}),
       '0')...};
}

// Cases of second last loop
template <size_t lower, size_t BoundsNextLower, size_t BoundsNextUpper,
          size_t... Is, class F, class... IntegralConstants>
void for_constexpr_impl(
    F&& f, for_bounds<BoundsNextLower, BoundsNextUpper> /*meta*/,
    std::index_sequence<Is...> /*meta*/, IntegralConstants&&... v) {
  (void)std::initializer_list<char>{
      ((void)for_constexpr_impl<BoundsNextLower>(
           std::forward<F>(f),
           std::make_index_sequence<BoundsNextUpper - BoundsNextLower>{},
           v..., std::integral_constant<size_t, Is + lower>{}),
       '0')...};
}

// Handle cases of more than two nested loops
template <size_t lower, class Bounds1, class... Bounds, size_t BoundsNextLower,
          size_t BoundsNextUpper, size_t... Is, class F,
          class... IntegralConstants>
void for_constexpr_impl(
    F&& f, for_bounds<BoundsNextLower, BoundsNextUpper> /*meta*/,
    std::index_sequence<Is...> /*meta*/, IntegralConstants&&... v) {
  (void)std::initializer_list<char>{
      ((void)for_constexpr_impl<BoundsNextLower, Bounds...>(
           std::forward<F>(f), Bounds1{},
           std::make_index_sequence<BoundsNextUpper - BoundsNextLower>{},
           v..., std::integral_constant<size_t, Is + lower>{}),
       '0')...};
}
{% endhighlight %}

Okay, so things are a bit more involved now. Let's look at the base case
first. The only difference in the base case is that now an additional
parameter pack is used to forward the `std::integral_constant`s that are
used to pass around the indices at compile time. These are in the same
order as the `for_bounds` in the call to `for_constexpr`.

The second function evaluates the second most nested loop, and then
calls the base case. The body is otherwise almost identical to the general
case, so let's discuss that. The arguments to the general case are the invokable
to later be called, the next `for_bounds`, the `index_sequence` over the
current loop, and finally the indices from the outer loops. The `for_bounds`
is taken
as an argument to allow selecting between non-symmetrized and symmetrized
loops (the symmetrized version takes a `for_symmetric` and is described
below). The standard parameter pack expansion into an `initializer_list`
is present in the body, and each element of the pack expansion involves
another call to `for_constexpr_impl`, where the next `for_bounds` (or
`for_symmetric`) is passed to the `_impl`. The indices of the loop are
appended to the `v`s pack in the call to `for_constexpr_impl` to build
the full list for the base case.

The only difference between `for_bounds` and `for_symmetric` is that in
the symmetric case the `BoundsNextLower` usages are replaced by:

{% highlight cpp linenos %}
std::get<BoundsNextIndex>(std::make_tuple(
                          IntegralConstants::value...,
                          Is + lower))
{% endhighlight %}

where `BoundsNextIndex` is the `Index` template parameter of the
`for_symmetric`. The code builds a `std::tuple<size_t...>` and then retrieves
the element that was requested to be symmetrized over (the first argument to
`for_symmetric`). That's it, we have implemented nested arbitrary-range
constexpr for loops, a `for_constexpr`. All examples in
the [Interface Design](#interface_design) section will now work.

Finally, I'll note that to achieve the "zero runtime recursion" goal
the functions must all be decorated with `ALWAYS_INLINE`, defined as

{% highlight cpp linenos %}
#define ALWAYS_INLINE __attribute__((always_inline)) inline
{% endhighlight %}

when using Clang and GCC. The lambda can also be inlined by using

{% highlight cpp linenos %}
#define JUST_ALWAYS_INLINE __attribute__((always_inline))
{% endhighlight %}

As I've done with my last several posts, the entire code is
available on [GitHub][github].

### Upper and Lower Bounded Symmetric Loops

Near the final stages of the design I decided that it is
straightforward to allow loops that range from 0 to the
bounding loop and loops that range from the bounding loop
to a specified upper bound. I described the latter above,
however the code I share on [GitHub][github] supports both
types of bounded loops. The code has Doxygen comments that
together with this post should be enough to make the code
useful.

### Summary

In this post we implemented nested compile time for loops, a
`for constexpr` in analogy to the `if constexpr` in C++17.
The main achievement is that it allows trivial iteration over `std::tuple`s
or generalized multi-index compile time containers. It also provides a fairly
trivial way to do explicit loop unrolling (be sure to benchmark that the
unrolled loop is faster!). The best summary
I think is to show one of the motivating code blocks again:

{% highlight cpp linenos %}
for_constexpr<for_bounds<0, 3>, for_bounds<0, 3>, for_symmetric<1, 2>>(
    [](auto I, auto J, auto K) {
      std::cout << I << ',' << J << ',' << K << "   " << I + 3 * (J + 3 * K)
                << '\n';
    });
{% endhighlight %}

I've shared the code on [GitHub][github] and I hope you enjoyed this post!

[github]: https://github.com/nilsdeppe/template-metaprogramming-tutorials/blob/master/for_constexpr.cpp
