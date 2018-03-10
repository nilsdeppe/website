---
layout: post
title:  "Using Clang's Feedback to Vectorize Code"
date:   2016-05-20 19:50:38
categories: clang vectorization c++ cpp
---

https://felix.abecassis.me/2012/08/sse-vectorizing-conditional-code/
https://msdn.microsoft.com/en-us/library/708ya3be(v=vs.100)
https://msdn.microsoft.com/en-us/library/x5c07e2a(v=vs.100).aspx
https://felix.abecassis.me/2011/09/cpp-getting-started-with-sse/
http://agner.org/optimize/
http://agner.org/optimize/optimizing_cpp.pdf
http://www.linuxjournal.com/content/introduction-gcc-compiler-intrinsics-vector-processing?page=0,2

Vectorization is using single-instruction multiple-data operations on
a CPU. We will initially focus on using classes in C++ and vectorizing
the code with feedback from the auto-vectorizer in clang and GCC. In
order to enable vectorization we will compile with `-O3` optimization
level. It is possible to turn on just the vectorizer but since we are
interested in what the compiler does for release settings we stick to
`-O3`. If you are using clang you can turn on vectorization feedback by
specifying the flags `-Rpass-analysis=loop-vectorize` to get detailed
feedback as to why a loop failed to be vectorized and
`-Rpass=loop-vectorize` to find out which loops were successfully
vectorized. When using GCC the flags to specify are
`-ftree-vectorize -ftree-vectorizer-verbose=7 -fopt-info-vec`. The sample
code I include is just to illustrate vectorization. Often there are
unitiliazed variables and other problems with the code. I feel that
these can be omitted in favor of clarity and readability of the
vectorization aspects of the code.

Now let's look at an example. We want to vectorize the vector-vector
addition `c = a + b`. Here is the sample code we are interested in:
{% highlight c++ %}
#include <vector>

int main() {
  const int ms = 10000;
  std::vector<int> a(ms), b(ms), c(ms);
  for (int i = 0; i < a.size(); ++i) {
    c[i] = a[i] + b[i];
  }
  return c[ms - 1];
}
{% endhighlight %}
The `return c[ms-1];` was necessary to prevent clang from optimizing
away the loop since the resulting vector `c` is never actually used
otherwise. Now we might expect the compiler to vectorize the above
loop but you should never assume what the compiler does. Always check
explicitly whether or not the loop is getting vectorized by looking
at the vectorization reports. In this particular case clang tells us
that
{% highlight bash %}
./t.cpp:8:3: remark: vectorized loop (vectorization factor: 4, unrolling interleave factor: 2)
      [-Rpass=loop-vectorize]
  for (int i = 0; i < ms; ++i) {
  ^
{% endhighlight %}
In this case the compiler did vectorize the loop, just as expected.

Let's look at another example. Let's replace the addition by a division,
that is, `c[i] = a[i] / b[i]`. Compiling with vectorization reports
enabled clang tells us
{% highlight bash %}
./t.cpp:8:3: remark: unrolled with interleaving factor 2 (vectorization not beneficial)
      [-Rpass=loop-vectorize]
  for (int i = 0; i < ms; ++i) {
  ^
{% endhighlight %}
We see the loop was unrolled but not vectorized. Now what happens if we
try multiplication instead of division? We get the same result that
we got for addition: the loop gets vectorized. The lesson to take away
is that you should never assume a loop is vectorized but that you
must read the vectorization reports.

Admittedly the above examples are rather contrived and uninteresting.
Let us consider a more realistic example, time evolving the 3D scalar
wave equation. Instead of worrying about how to perform the time
evolution we will focus on how to evaluate the right hand side of
the evolution equations, which is what typically takes up most
of the work. If the data is stored in a class that contains
vectors of each variable, the field $$\Psi$$, the momentum
$\Pi=\partial_t\Psi$ and the auxiliary variable $\Phi_i=partial_i\Psi$.
The class would have functions that return variables, for example
`const Data& pi() const { return mVariables[1]; }`. Since the compiler
does not look into each function in the loop such a call would prevent
the compiler from making any optimizations.

There are a lot of really great resources to help you conceptually
understand what vectorization does. The really short version is that
the CPU can perform the same operation on a lot of data very
efficiently. By vectorizing your code you increase the number of
operations that the CPU can do "at once" and therefore increase
performance.

An easy way to time the performance increase you get in the loop or
from any code changes that are easily encapsulated is using `std::chrono`.
Since any unit of time is appropriate as long as the total run time of
the code your timing is on the order of several seconds we can..

Vectorization is using single-instruction multiple-data operations on a CPU. Consider having vector<double>'s `A`,`B` and `C` and you want to do C=A+B. Writing the code as

{% highlight c++ %}
for (int i = 0; i < A.size(); ++i) {
  C[i] = A[i] + B[i];
}
{% endhighlight %}

is not guaranteed to result in vectorization, even if you explicit specify that you want the loop vectorized using `#pragma clang loop vectorize(enable) interleave(enable)` (Clang) or `#pragma ivdep` (GCC and Intel). This is because the compiler does not assume that the for loop cannot change the size of `A`. You must explicitly guarantee this for the compiler to possibly vectorize the loop. This done simplest by defining a `const int ms = A.size()` and having the loop check `i < ms`. That is,

{% highlight c++ %}
const int ms = A.size();
for (int i = 0; i < ms; ++i) {
  C[i] = A[i] + B[i];
}
{% endhighlight %}

As important, or possibly even more important is knowing when the compiler has successfully vectorized a loop and when it has not. When attempting to optimize code you must always test how your change affects the performance and the resulting code. I will go over the latter first.

Unfortunately the exact process of getting vectorization reports varies between compilers. I will go over how to do this with Clang, which should be sufficient that with the documentation of the GCC and Intel compilers that you will be able to do the same with them. First, to enable vectorization with Clang you must compile with `-O3`. While you can enable just vectorization, we are interested in how our code will behave in release builds. Clang offers three different types of vectorization reports, the most detailed being enable by specifying the flag `-Rpass-analysis=loop-vectorize`, which will give feedback on which loops were not optimized and why not. Here is an example output from clang,

{% highlight c++ %}
Gradient.cpp:44:29: remark: loop not vectorized: loop control
      flow is not understood by vectorizer [-Rpass-analysis=loop-vectorize]
        for (int s = 0; s < mesh.size(); ++s) {
                            ^
{% endhighlight %}

What the compiler is telling us is that it is not sure if `mesh.size()` is guaranteed not to change during iterations. Here declaring a `const int` as described above results in the loop being vectorized and a speedup by a factor of four to five.
