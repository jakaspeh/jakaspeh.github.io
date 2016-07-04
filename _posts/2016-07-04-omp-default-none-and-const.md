---
layout: post
title:  "OpenMP: default(none) and const variables"
date:   2016-07-04
categories: cpp, openmp
---

In [the previous article](/blog/2016/06/omp-data-sharing-attributes.html), we
gave a recommendation to use the `{%raw%}default(none){%endraw%}` clause
whenever possible.

The `{%raw%}default(none){%endraw%}` clause forces a programmer to explicitly
specify the data-sharing attributes of all variables in a parallel region. Using
this clause then forces the programmer to think about data-sharing attributes.

This is beneficial because the code is clearer and has less bugs.

But following this principle might be difficult in some cases. In the next example,

{% highlight c++ %}
const int n = 10;
const int a = 7;

#pragma omp parallel for default(none) shared(a, n)
for (int i = 0; i < n; i++)
{
    int b = a + i;
    ...
}
{% endhighlight %}

we list `{%raw%}a{%endraw%}` and `{%raw%}n{%endraw%}` in the shared clause,
because they are shared variables.

Unfortunately, the `{%raw%}gcc{%endraw%}` compiler does not compile the code.
The compiler terminates with the following compilation error.

{% highlight c++ %}
$ g++ -fopenmp -Wall -pthread ompDefaultNone.cpp -o ompDefaultNone
In function "int main()":
error: "a" is predetermined "shared" for "shared"
#pragma omp parallel for default(none) shared(a, n)
{% endhighlight %}

The same happens with the versions `{%raw%}4.8.4{%endraw%}`,
`{%raw%}4.9.3{%endraw%}`, `{%raw%}5.3.0{%endraw%}` and `{%raw%}6.1.1{%endraw%}`
of the `{%raw%}gcc{%endraw%}` compiler. They all give the same error.

So, what is happening?

The variables `{%raw%}a{%endraw%}` and `{%raw%}n{%endraw%}` are const
variables. Since they can not be modified, the compiler implicitly sets their
data-sharing attribute to shared. This makes sense, because there is arguably no
need to make a private copy of a constant variable for each thread.

Then we, as programmers, explicitly set the data-sharing attribute of the const
variables to shared.  We do it by listing the variables in the shared clause.

Here the problem happens. 

The compiler complains that we set the data-sharing attribute of
`{%raw%}a{%endraw%}` to shared, but its data-sharing attribute was already
predetermined to be shared. This is what the compiler wants to tell us with:

{% highlight c++ %}
error: "a" is predetermined "shared" for "shared"
{% endhighlight %}

It looks like that the compiler does not accept explicit information about the
data-sharing attribute when the compiler knows it already.

The solution is to omit the const variables from the shared clause (because
their data-sharing attribute is already determined) and to still use the
`{%raw%}default(none){%endraw%}` clause (because of its benefits). 

Then, the solution of the previous example looks like this:

{% highlight c++ %}
const int n = 10;
const int a = 7;

#pragma omp parallel for default(none) 
for (int i = 0; i < n; i++)
{
    int b = a + i;
    ...
}
{% endhighlight %}

Now, the code compiles. The const variables are shared inside the parallel
region and we still have the benefits of the `{%raw%}default(none){%endraw%}`
clause for the non-const variables.

Summary
-------

In this article, we addressed a problem with the `{%raw%}default(none){%endraw%}`
clause and const variables. We also presented a solution to the problem.


Link:

* [Stackoverflow: OpenMP: predetermined 'shared' for
  'shared'?](http://stackoverflow.com/questions/13199398/openmp-predetermined-shared-for-shared)

Jaka's Corner OpenMP series:

* [OpenMP: Introduction](/blog/2016/04/omp-introduction.html)
* [OpenMP: For](/blog/2016/05/omp-for.html)
* [OpenMP: Sections](/blog/2016/05/omp-sections.html)
* [OpenMP: Monte Carlo method for Pi](/blog/2016/05/omp-monte-carlo-pi.html)
* [OpenMP: For & Reduction](/blog/2016/06/omp-for-reduction.html)
* [OpenMP: For & Scheduling](/blog/2016/06/omp-for-scheduling.html)
* [OpenMP: Single](/blog/2016/06/omp-single.html)
* [OpenMP: Data-Sharing Rules](/blog/2016/06/omp-data-sharing-attributes.html)

