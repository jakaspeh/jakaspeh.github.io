---
layout: post
title:  "OpenMP: Critical construct and zero matrix entries"
date:   2016-07-18
categories: cpp, openmp
---

The critical construct takes care that a program executes the associated block
of code by a single thread at a time. It is not possible that multiple threads
execute this block of code at the same time.

The syntax of the critical construct is 

{% highlight c++ %}
#pragma omp critical
{
    // block of code
}
{% endhighlight %}

In this article, we use the critical construct to avoid data races. 

Has a matrix a zero entry?
--------------------------

We should figure out if a given matrix has a zero entry. For all
non-mathematicians out there, a matrix is just a rectangular array of numbers. 

![3 x 4 matrix](/pics/matrix34.png)

Each number is one entry. And we should determine if one of the entries is equal
to zero.

We write a simple program. The program first initializes a boolean variable
`{%raw%}has_zero{%endraw%}` to `{%raw%}false{%endraw%}`. Then the program loops
through each entry and sets the boolean variable to `{%raw%}true{%endraw%}` if
the entry is equal to zero.

If, in the end, `{%raw%}has_zero{%endraw%}` is equal to `{%raw%}true{%endraw%}`,
then the matrix has a zero entry. If `{%raw%}has_zero{%endraw%}` is equal to
`{%raw%}false{%endraw%}`, then the matrix does not have any zero entries.

{% highlight c++ %}
bool has_zero = false;
for (int row = 0; row != rows; row++)
{
    for (int col = 0; col != cols; col++)
    {
        if (matrix(row, col) == 0)
        {
            has_zero = true;
        }
    }
}
{% endhighlight %}

Ok, this was not a hard problem. But what if the number of entries of the matrix
is very high. We might benefit from a parallel execution of the program.
Therefore, let us parallelize it.

Parallelization
---------------

We decide to parallelize the outer loop. We identify all the shared
variables in the parallel region: `{%raw%}rows{%endraw%}`,
`{%raw%}cols{%endraw%}`, `{%raw%}matrix{%endraw%}` and
`{%raw%}has_zero{%endraw%}`. 

The three variables `{%raw%}rows{%endraw%}`, `{%raw%}cols{%endraw%}` and
`{%raw%}matrix{%endraw%}` are read-only variables. They do not cause any
problems. 

On the other hand, the program modifies `{%raw%}has_zero{%endraw%}`. This means
that multiple threads might modify this variable at the same time. Here is a
potential **data race**. We must "critically" do something about it!

It is critical!
---------------

OpenMP provides some synchronization mechanisms, which can help us to resolve
data races.  The critical construct is one of them. It takes
care that only one tread executes selected section of code. It is not possible
that multiple treads execute this section at the same time.

This is exactly what we need. If only one thread at a time modifies
`{%raw%}has_zero{%endraw%}`, then there is no data race. 

The parallelized version of the program now looks like this:

{% highlight c++ %}
bool has_zero = false;
#pragma omp parallel default(none) shared(rows, cols, matrix, has_zero)
{
    #pragma omp for 
    for (int row = 0; row != rows; row++)
    {
        for (int col = 0; col != cols; col++)
        {
            if (matrix(row, col) == 0)
            {
                #pragma omp critical
                {
                    has_zero = true;
                }
            }
        }
    }
}
{% endhighlight %}

Validation
----------

We implement both versions of the program and compare the execution time for a
matrix of size `{%raw%}1000 x 1000{%endraw%}`. You can access the source code
[here](https://github.com/jakaspeh/concurrency/blob/master/ompCritical.cpp) and
play with it.

{% highlight c++ %}
$ ./ompCritical 
Sequential implementation: 
Time: 0.0130267
Parallel implementation: 
Time: 0.00215875
{% endhighlight %}

There is some benefit in the parallel version since its execution time is lower
than the time of the sequential version.

To all "angry" readers
----------------------

Ok, I know. You are getting angry and you think: "What an idiot! He should use
the break statement!" Yes, this is a valid concern. Using the break statement in
the sequential version would definitely improve the performance. It is also
possible to parallelize such a loop. 

We will address these issues in the next article ;-).

Summary
-------

We parallelized a program which checks if a matrix has a zero entry. We used the
critical construct to ensure that the program is data race free. 

In the next article, we will improve the current program by using the break
statement.

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/ompCritical.cpp)
* [The critical construct [OpenMP API, page 149]](http://www.openmp.org/mp-documents/openmp-4.5.pdf)

Jaka's Corner OpenMP series:

* [OpenMP: Introduction](/blog/2016/04/omp-introduction.html)
* [OpenMP: For](/blog/2016/05/omp-for.html)
* [OpenMP: Sections](/blog/2016/05/omp-sections.html)
* [OpenMP: Monte Carlo method for Pi](/blog/2016/05/omp-monte-carlo-pi.html)
* [OpenMP: For & Reduction](/blog/2016/06/omp-for-reduction.html)
* [OpenMP: For & Scheduling](/blog/2016/06/omp-for-scheduling.html)
* [OpenMP: Data-Sharing Rules](/blog/2016/06/omp-data-sharing-attributes.html)
* [OpenMP: default(none) and const variables](/blog/2016/07/omp-default-none-and-shared.html)
* [OpenMP: Barrier](/blog/2016/07/omp-barrier.html)


