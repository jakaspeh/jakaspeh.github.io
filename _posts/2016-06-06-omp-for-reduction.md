---
layout: post
title:  "OpenMP: For & Reduction"
date:   2016-06-06
categories: cpp
---

When I was researching for this article, I stumbled into [this Wikipedia
web page](https://en.wikipedia.org/wiki/Reduction). It shows many meanings of a
reduction. I clicked on several entries from this page and I realized that
a reduction simplifies something complex into something which is understandable
and uncomplicated.

Therefore, we can argue that the following example is a reduction

{% highlight c++ %}
sum = a[0] + a[1] + a[2] + a[3] + a[4] + a[5] + a[6] + a[7] + a[8] + a[9]
{% endhighlight %}

It simplifies the long expression 

{% highlight c++ %}
a[0] + a[1] + a[2] + a[3] + a[4] + a[5] + a[6] + a[7] + a[8] + a[9]
{% endhighlight %}

into something shorter -- `{%raw%}sum{%endraw%}`. A similar argument
argues that also the following example is a reduction

{% highlight c++ %}
product = a[0] * a[1] * a[2] * a[3] * a[4] * a[5] * a[6] * a[7] * a[8] * a[9]
{% endhighlight %}

It reduces the long expression into the simpler one. 

Loops
-----

In C++, we can write the previous examples with for loops:

{% highlight c++ %}
sum = 0;
for (auto i = 0; i < 10; i++)
{
    sum += a[i]
}
{% endhighlight %}

and 

{% highlight c++ %}
product = 1;
for (auto i = 0; i < 10; i++)
{
    product *= a[i]
}
{% endhighlight %}

In fact, if we have an appropriate operation `{%raw%}o{%endraw%}`, we can rewrite

{% highlight c++ %}
reduction = a[0] o a[1] o a[2] o a[3] o a[4] o a[5] o a[6] o a[7] o a[8] o a[9]
{% endhighlight %}

into a for loop

{% highlight c++ %}
// initialize reduction
for (int i = 0; i < 10; i++)
{
    reduction = reduction o a[i]
}
{% endhighlight %}

OpenMP and reduction
--------------------

OpenMP is specialized into parallelization of for loops. But a parallelization
of the previous for loops is tricky. There is a shared variable
(`{%raw%}sum{%endraw%}` / `{%raw%}product{%endraw%}` /
`{%raw%}reduction{%endraw%}`) which is modified in every iteration. If we are
not careful when parallelizing such for loops, we might introduce data races.

For example, if we would parallelize the previous for loops by simply adding 

{% highlight c++ %}
#pragma omp parallel for 
{% endhighlight %}

we would introduce a data race, because multiple threads could try to update the
shared variable at the same time.

But for loops which represent a reduction are quite common. Therefore, OpenMP
has the special `{%raw%}reduction{%endraw%}` clause which can express the
reduction of a for loop.

In order to specify the reduction in OpenMP, we must provide 

* an operation (`{%raw%}+{%endraw%}` / `{%raw%}*{%endraw%}` /
  `{%raw%}o{%endraw%}`)
* and a reduction variable (`{%raw%}sum{%endraw%}` / `{%raw%}product{%endraw%}` /
  `{%raw%}reduction{%endraw%}`). This variable holds the result of the
  computation.

For example, the following for loop

{% highlight c++ %}
sum = 0;
for (auto i = 0; i < 10; i++)
{
    sum += a[i]
}
{% endhighlight %}

can be parallelized with the `{%raw%}reduction{%endraw%}` clause where the
operation is equal to `{%raw%}+{%endraw%}` and the reduction variable is equal
to `{%raw%}sum{%endraw%}`:

{% highlight c++ %}
sum = 0;
#pragma omp parallel for shared(sum, a) reduction(+: sum)
for (auto i = 0; i < 10; i++)
{
    sum += a[i]
}
{% endhighlight %}


Operators and variables
-----------------------

OpenMP is able to perform a reduction for the following operators

{% highlight c++ %}
+, -, *, &, |, ^, &&, ||
{% endhighlight %}

Let `{%raw%}o{%endraw%}` be one of the previous operators and let
`{%raw%}x{%endraw%}` be a reduction variable. Further, let
`{%raw%}expr{%endraw%}` be an expression which does not depend on
`{%raw%}x{%endraw%}`. OpenMP specifies which statements are supported in the
definition of the reduction. This statements occur in the body of the for
loop. The statements are

* `{%raw%}x = x o expr{%endraw%}`. 

For example, if `{%raw%}o == +{%endraw%}` and `{%raw%}x == sum{%endraw%}` then
the statement is `{%raw%}sum = sum + expr{%endraw%}`.

* `{%raw%}x o= expr{%endraw%}`.

For example, if `{%raw%}o == +{%endraw%}` and `{%raw%}x == sum{%endraw%}` then
the statement is `{%raw%}sum += expr{%endraw%}`.

* `{%raw%}x = expr o x{%endraw%}`.

In this case the operator `{%raw%}o{%endraw%}` is not allowed to be `{%raw%}-{%endraw%}`.

There are some additional supported statements if the operator is
`{%raw%}+{%endraw%}` or `{%raw%}-{%endraw%}`. They are

* `{%raw%}x++{%endraw%}`,
* `{%raw%}++x{%endraw%}`,
* `{%raw%}x--{%endraw%}`,
* `{%raw%}--x{%endraw%}`. 

If the reduction operator `{%raw%}o{%endraw%}` is one of the operators listed
above and if the statement for the reduction variable `{%raw%}x{%endraw%}` is
one of the statements listed above, then we can express the reduction with the
loop construct:

{% highlight c++ %}
#pragma omp for reduction(o: x)
for (...) { ... }
{% endhighlight %}

More information about the specification of the reduction clause is available
[in the OpenMP specification on the page
201](http://www.openmp.org/mp-documents/openmp-4.5.pdf).

Implementation
--------------

How does OpenMP parallelize a for loop declared with a reduction clause?  OpenMP
creates a team of threads and then shares the iterations of the for loop between
the threads. Each thread has its own **local** copy of the reduction
variable. The thread modifies only the local copy of this variable. Therefore,
there is no data race. When the threads join together, all the local copies of
the reduction variable are combined to the global shared variable.

For example, let us parallelize the following for loop 

{% highlight c++ %}
sum = 0;
#pragma omp parallel for shared(sum, a) reduction(+: sum)
for (auto i = 0; i < 9; i++)
{
    sum += a[i]
}
{% endhighlight %}

and let there be three threads in the team of threads. Each thread has
`{%raw%}sumloc{%endraw%}`, which is a local copy of the reduction
variable. The threads then perform the following computations

* Thread 1

{% highlight c++ %}
sumloc_1 = a[0] + a[1] + a[2]
{% endhighlight %}

* Thread 2

{% highlight c++ %}
sumloc_2 = a[3] + a[4] + a[5]
{% endhighlight %}

* Thread 3

{% highlight c++ %}
sumloc_3 = a[6] + a[7] + a[8]
{% endhighlight %}

In the end, when the treads join together, OpenMP reduces local copies to the
shared reduction variable

{% highlight c++ %}
sum = sumloc_1 + sumloc_2 + sumloc_3
{% endhighlight %}

The following figure is another representation of this process.

![Example of reduction](/pics/reduction_clause.png)

Summary
-------

We learned how to use the `{%raw%}reduction{%endraw%}` clause.  We looked at the
specifications of the clause and familiarized ourselves with its implementational
details.

Jaka's Corner OpenMP series:

* [OpenMP: Introduction](/blog/2016/04/omp-introduction.html)
* [OpenMP: For](/blog/2016/05/omp-for.html)
* [OpenMP: Sections](/blog/2016/05/omp-sections.html)
* [OpenMP: Monte Carlo method for Pi](/blog/2016/05/omp-monte-carlo-pi.html)




