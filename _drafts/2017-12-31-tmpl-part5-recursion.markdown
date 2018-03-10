---
layout: post
title:  "Template Metaprogramming Part 5 - Iteration"
date:   2018-09-31 12:00:00
categories: c++ cpp template metaprogramming typelist brigand
comments: True
permalink: posts/tmpl-part5
shortUrl: https://goo.gl/LdRg45
is_post: true
---

In this post I will discuss iteration. There are several different methods
of iteration each with their own pros and cons. Odin Holmes has done a lot of
great work on improving performance of metaprogramming. I will do my best
to include his new findings in this post and apologize for anything I
don't get quite right. I think it is important to discuss these new ideas
and to explore how they affect TMP.

First, it is important to distinguish between two variants of the for or
while construct: ones that make available the iteration index and ones
that simply perform some action given the current value, and possibly
an accumulated state. I guess that makes three variants, two of which
are readily available in all TMP libraries: `fold` and `transform`.

So let's start by looking at some rather arbitrary and contrived
examples. Let's say we have a typelist of fundamentals,

```cpp
using my_fund_list = typelist<double, int, char, double, bool,
                              bool, short, char>;
```

Now let's say we want to check each metavalue and see if it is
a double, returning `std::true_type` if it is and `std::false_type`
if it's not. That is, we are feeding in `my_fund_list` and getting
out a `typelist` that contains `std::true_type` and
`std::false_type`. What we need is to somehow unpack each element
of the typelst one at a time and transform it into a differnt
type.
