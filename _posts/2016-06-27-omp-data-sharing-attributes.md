---
layout: post
title:  "OpenMP: Data-Sharing Rules"
date:   2016-06-27
categories: cpp, openmp
---


A variable in an OpenMP parallel region can be either shared or private. If a
variable is shared, then there exists one instance of this variable which is
shared among all threads. If a variable is private, then each thread in a team of
threads has its own local copy of the private variable.

In this article, we look how OpenMP specifies if a variable is shared or private. 


Implicit Rules
--------------

OpenMP has a set of rules, which deduce the data-sharing attributes of variables.

For example, let us consider the following snippet of code.

{% highlight c++ %}
int i = 0;
int n = 10;
int a = 7;

#pragma omp parallel for 
for (i = 0; i < n; i++)
{
    int b = a + i;
    ...
}
{% endhighlight %}

There are four variables `{%raw%}i{%endraw%}`, `{%raw%}n{%endraw%}`,
`{%raw%}a{%endraw%}` and `{%raw%}b{%endraw%}`. 

The data-sharing attribute of variables, which are declared outside the
parallel region, is usually shared. Therefore, `{%raw%}n{%endraw%}` and
`{%raw%}a{%endraw%}` are shared variables.

The loop iteration variables, however, are private by default. Therefore,
`{%raw%}i{%endraw%}` is private.

The variables which are declared locally within the parallel region are
private. Thus `{%raw%}b{%endraw%}` is private.

I recommend to declare the loop iteration variables inside the parallel
region. In this case, it is clearer that this variables are private. The upper
snippet of code then looks like:

{% highlight c++ %}
int n = 10;                 // shared
int a = 7;                  // shared

#pragma omp parallel for 
for (int i = 0; i < n; i++) // i private
{
    int b = a + i;          // b private
    ...
}
{% endhighlight %}


Explicit rules
--------------

We can explicitly set the data-sharing attribute of a variable. 

Shared
======

The `{%raw%}shared(list){%endraw%}` clause declares that all the variables in
`{%raw%}list{%endraw%}` are shared. In the next example

{% highlight c++ %}
#pragma omp parallel for shared(n, a)
for (int i = 0; i < n; i++)
{
    int b = a + i;
    ...        
}
{% endhighlight %}

`{%raw%}n{%endraw%}` and `{%raw%}a{%endraw%}` are shared variables. 

OpenMP does not put any restriction to prevent data races between shared
variables. This is a responsibility of a programmer.

Shared variables introduce an overhead, because one instance of a variable is
shared between multiple threads. Therefore, it is often best to minimize the
number of shared variables when a good performance is desired.

Private
=======

The `{%raw%}private(list){%endraw%}` clause declares that all the variables in
`{%raw%}list{%endraw%}` are private. In the next example

{% highlight c++ %}
#pragma omp parallel for shared(n, a) private(b)
for (int i = 0; i < n; i++)
{
    b = a + i;
    ...
}
{% endhighlight %}

`{%raw%}b{%endraw%}` is a private variable. When a variable is declared private,
OpenMP replicates this variable and assigns its local copy to each thread.

The behavior of private variables is sometimes unintuitive. Let us assume that a
private variable has a value before a parallel region. However, the value of the
variable at the beginning of the parallel region is undefined. Additionally, the
value of the variable is undefined also after the parallel region.

For example:

{% highlight c++ %}
int p = 0; 
// the value of p is 0

#pragma omp parallel private(p)
{
    // the value of p is undefined
    p = omp_get_thread_num();
    // the value of p is defined
    ...
}
// the value of p is undefined
{% endhighlight %}

On several occasions, we can avoid listing private variables in the OpenMP
constructs by declaring them inside a parallel region. For example, instead of

{% highlight c++ %}
int p;
#pragma omp parallel private(p)
{
    p = omp_get_thread_num();
}
{% endhighlight %}

we can do 

{% highlight c++ %}
#pragma omp parallel
{
    int p = omp_get_thread_num();
    ...
}
{% endhighlight %}

I highly encourage declaring private variables inside a parallel region
whenever possible. This guideline simplifies the code and increases its
readability.


Default 
=======

There are two versions of the `{%raw%}default{%endraw%}` clause. First, we focus
on `{%raw%}default(shared){%endraw%}` option and then we consider
`{%raw%}default(none){%endraw%}` clause. 

These two versions are specific for C++ programmers of OpenMP. There are some
additional `{%raw%}default{%endraw%}` possibilities for Fortran programmers. For
more details look at [the OpenMP specification on the page
189](http://www.openmp.org/mp-documents/openmp-4.5.pdf).

### Default (shared) ###

The `{%raw%}default(shared){%endraw%}` clause sets the data-sharing attributes
of all variables in the construct to shared. In the following example

{% highlight c++ %}
int a, b, c, n;
...

#pragma omp parallel for default(shared)
for (int i = 0; i < n; i++)
{
    // using a, b, c
}
{% endhighlight %}

`{%raw%}a{%endraw%}`, `{%raw%}b{%endraw%}`, `{%raw%}c{%endraw%}` and
`{%raw%}n{%endraw%}` are shared variables.

Another usage of `{%raw%}default(shared){%endraw%}` clause is to specify the
data-sharing attributes of the majority of the variables and then additionally
define the private variables. Such usage is presented below:

{% highlight c++ %}
int a, b, c, n;

#pragma omp parallel for default(shared) private(a, b)
for (int i = 0; i < n; i++)
{
    // a and b are private variables
    // c and n are shared variables 
}
{% endhighlight %}


### Default (none) ###

The `{%raw%}default(none){%endraw%}` clause forces a programmer to explicitly
specify the data-sharing attributes of all variables. 

A distracted programmer might write the following piece of code

{% highlight c++ %}
int n = 10;
std::vector<int> vector(n);
int a = 10;

#pragma omp parallel for default(none) shared(n, vector)
for (int i = 0; i < n; i++)
{
    vector[i] = i * a;
}
{% endhighlight %}

But then the compiler would complain

{% highlight c++ %}
error: ‘a’ not specified in enclosing parallel
         vector[i] = i * a;
                       ^
error: enclosing parallel
     #pragma omp parallel for default(none) shared(n, vector)
             ^
{% endhighlight %}

The reason for the unhappy compiler is that the programmer used
`{%raw%}default(none){%endraw%}` clause and then she/he forgot to explicitly
specify the data-sharing attribute of `{%raw%}a{%endraw%}`. The correct version of the
program would be

{% highlight c++ %}
int n = 10;
std::vector<int> vector(n);
int a = 10;

#pragma omp parallel for default(none) shared(n, vector, a)
for (int i = 0; i < n; i++)
{
    vector[i] = i * a;
}
{% endhighlight %}


Good practices
--------------

I warmly encourage that a programmer follows the next two guidelines. 

The first guideline is to **always write parallel regions with the 
`{%raw%}default(none){%endraw%}` clause**. This forces the programmer to
explicitly think about the data-sharing attributes of all variables.

The second guideline is to **declare private variables inside parallel regions
whenever possible**. This guideline improves the readability of the code and
makes it clearer.

Summary
-------

We studied the implicit and explicit rules for deducing the data-sharing
attributes of variables. We also recommended a couple of good practices that a
programmer should follow.


Links:

* [Data-Sharing Attribute Clauses [OpenMP API, page
  188]](http://www.openmp.org/mp-documents/openmp-4.5.pdf)

Jaka's Corner OpenMP series:

* [OpenMP: Introduction](/blog/2016/04/omp-introduction.html)
* [OpenMP: For](/blog/2016/05/omp-for.html)
* [OpenMP: Sections](/blog/2016/05/omp-sections.html)
* [OpenMP: Monte Carlo method for Pi](/blog/2016/05/omp-monte-carlo-pi.html)
* [OpenMP: For & Reduction](/blog/2016/06/omp-for-reduction.html)
* [OpenMP: For & Scheduling](/blog/2016/06/omp-for-scheduling.html)
* [OpenMP: Single](/blog/2016/06/omp-single.html)


