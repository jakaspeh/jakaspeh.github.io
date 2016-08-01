---
layout: post
title:  "OpenMP: Cancel and cancellation points"
date:   2016-08-01
categories: cpp, openmp
---

In [the previous article](/blog/2016/07/omp-thread-cancellation-model.html), we
explained the OpenMP cancellation model.

This article has three parts. First, we summarize the model. Afterwards, we
explain the OpenMP constructs for the cancellation. And in the end, we
demonstrate the usage of the constructs with an example.


Thread cancellation model
-------------------------

Let us assume that we have a parallel loop. We would like to cancel this loop
early. How can we do it?

For a sequential loop, we would use something similar to the break
statement. But the parallel case is more complicated. However, OpenMP provides
the cancellation model. We can use this model for canceling parallel loops.

OpenMP has a cancel construct which we can put in a parallel loop. This
construct imitates the "parallel break statement".

When a thread reaches the cancel construct, the thread activates the
cancellation and exits the loop.

The other threads check if the cancellation is activated at the cancellation
points. If the cancellation is activated then the other threads also exit the
loop. Otherwise, they continue.

Let us illustrate this model with an example.

Example
=======

The following figure presents the outline of the example.

![Parallel break](/pics/omp_break_parallel.png)

There are three threads: the blue, the green and the brown thread. They
parallelize the loop. They may encounter `{%raw%}CP{%endraw%}`, which stands for
a cancellation point, or `{%raw%}BR{%endraw%}`, which stands for the "parallel
break statement" (the cancel construct).

The cancellation is not activated at the beginning of the loop. We illustrate
this fact on the right side, with the answer to the question: "Should thread
break?".

The green thread first encounters a cancellation point. Since the cancellation is
not activated, the thread continues executing the loop.

Then, the blue thread reaches the "parallel break statement". The thread
activates the cancellation and exits the loop.

Afterwards, the brown thread reaches a cancellation point. Since the
cancellation is activated, the thread exits the loop.

Finally, the green thread reaches its second cancellation point. The thread
exits the loop, because the cancellation is activated.

If you are interested in a more detailed description of the cancellation model,
look at [this article](/blog/2016/07/omp-thread-cancellation-model.html).


Cancel and cancellation points
------------------------------

Now, we are familiar with the OpenMP cancellation model. The next step is to use
this model in our programs.

In order to use it, we have to do the following steps:

* We have to enable the cancellation features of OpenMP. 
* We have to activate the cancellation.
* And we have to set up the cancellation points. 

We describe each of these steps in the next sections. 


Enabling the cancellation
=========================

The cancellation of the parallel loops is an expensive operation. OpenMP
performs it only if the environmental variable 

{% highlight c++ %}
OMP_CANCELLATION
{% endhighlight %}

is set to `{%raw%}true{%endraw%}`. We can check the value of the
`{%raw%}OMP_CANCELLATION{%endraw%}` with the

{% highlight c++ %}
omp_get_cancellation() 
{% endhighlight %}

function. 

The first time I wanted to use the cancellation features of OpenMP, I forgot to
set the `{%raw%}OMP_CANCELLATION{%endraw%}`. The program was still correct but the
performance was below expectation. You should learn from my mistake and do not
forget to set the environmental variable.

Activate the cancellation
=========================

The cancel construct activates the cancellation. In the next example

{% highlight c++ %}
#pragma omp parallel for
for (...)
{
    ...

    if (condition)
    {
        ...
        #pragma omp cancel for 
    }
}
{% endhighlight %}

the cancel construct activates the cancellation of the for loop.

Again, OpenMP ignores the cancel constructs if the
`{%raw%}OMP_CANCELLATION{%endraw%}` is not set to `{%raw%}true{%endraw%}`.

Setting up the cancellation points
==================================

OpenMP sets the cancellation points at the following locations

* `{%raw%}cancel{%endraw%}` constructs
* implicit and explicit barriers 
* and `{%raw%}cancellation points{%endraw%}`. 

Since OpenMP sets a cancellation point at the `{%raw%}cancel{%endraw%}`
constructs, a thread which activates the cancellation also immediately breaks out
of the loop.

OpenMP also sets the cancellation points at the implicit and explicit
barriers. We explained how OpenMP sets these barriers [in this
article](/blog/2016/07/omp-barrier.html).

We can also explicitly set the cancellation points. If we activated the
cancellation of the for loop with 

{% highlight c++ %}
#pragma omp cancel for
{% endhighlight %}

we can set the cancellation point for this region with

{% highlight c++ %}
#pragma omp cancellation point for
{% endhighlight %}

Again, OpenMP ignores the cancellation points if 
`{%raw%}OMP_CANCELLATION{%endraw%}` is not set to `{%raw%}true{%endraw%}`.

Because a compiler sets some cancellation points automatically, we can still
use the cancellation model without setting them explicitly. However, we might
significantly improve the performance of a program if we do set them.

Disclaimer
=========

In the examples above, we assumed that we are canceling a loop. Therefore, we
always had the `{%raw%}for{%endraw%}` option in the cancel construct and in the
cancellation point construct.

But there are other possibilities. We could also cancel 

* parallel region,
* sections or
* `{%raw%}taskgroup{%endraw%}`. 

If we cancel a parallel region, then the constructs would be

{% highlight c++ %}
#pragma omp cancel parallel
{% endhighlight %}

and 

{% highlight c++ %}
#pragma omp cancellation point parallel
{% endhighlight %}


Example: has a matrix a zero entry?
-----------------------------------

In [one of the previous articles](/blog/2016/07/omp-critical.html), we wrote a
parallel program which checks if a matrix has a zero entry. However, there was
an idea to improve the program by using the break statement.

Let us implement this idea. We start by writing the sequential version of the
program. Then we parallelize it.

Sequential version
==================

The idea of this solution is to "break" out of the loops as soon as we know that
a zero entry exists. The sequential solution is then:

{% highlight c++ %}
bool has_zero = false;
for (int row = 0; row != rows; row++)
{
    for (int col = 0; col != cols; col++)
    {
        if (matrix(row, col) == 0)
        {
            has_zero = true;
            break;
        }
    }

    if (has_zero)
    {
        break;
    }   
}
{% endhighlight %}

The program loops through each row and column of a matrix. If an entry of the
matrix is equal to zero, the program exits the loops. Since the first break
statement only breaks out of the inner loop, we must also include the second
break statement, which breaks out of the outer loop. 

The results is stored in `{%raw%}has_zero{%endraw%}` variable. If the matrix has
a zero entry, then `{%raw%}has_zero{%endraw%}` is `{%raw%}true{%endraw%}`,
otherwise `{%raw%}has_zero {%endraw%}` is `{%raw%}false{%endraw%}`.

OK, now let us parallelize this program. 

Parallel version
================

The idea of this version is to parallelize the outer most loop and to use the
cancel construct to break out of the loop. The code looks like this:

{% highlight c++ %}
bool has_zero = false;
#pragma omp parallel default(none) shared(matrix, has_zero)
{
    #pragma omp for 
    for (int row = 0; row < rows; row++)
    {
        for (int col = 0; col < cols; col++)
        {
            if (matrix(row, col) == 0)
            {
                #pragma omp critical
                {
                    has_zero = true;
                }
                #pragma omp cancel for 
            }
        }
    }
}
{% endhighlight %}

The parallel region has two shared variables: `{%raw%}matrix{%endraw%}` and
`{%raw%}has_zero{%endraw%}`. 

The matrix is read only variable, therefore it does not introduce any problems.

However, multiple threads might modify `{%raw%}has_zero{%endraw%}`. In order to
avoid data races, we use [the critical
construct](/blog/2016/07/omp-critical.html).

If an entry is equal to zero, then a thread sets `{%raw%}has_zero{%endraw%}` to
`{%raw%}true{%endraw%}`, activates the cancellation and exits the loop.  This is
similar to the break statement in the sequential version.

OpenMP also inserts two cancellation points. One is at the cancel construct and
the other one is in the end of the for loop.

Improvement
===========

The previous version is correct, but we might improve its performance by adding
some cancellation points. However, if a thread encounters them too often, the
performance might be worse.

With this in mind, we do not add a cancellation point in the inner loop. In this
case, a thread would have to check it after accessing each entry of the matrix.

Therefore, we add a cancellation point in the end of the outer loop. In this
case, a thread checks the cancellation point after processing the entire row of
the matrix.

The final code looks like this:

{% highlight c++ %}
bool has_zero = false;
#pragma omp parallel default(none) shared(matrix, has_zero)
{
    #pragma omp for 
    for (int row = 0; row < rows; row++)
    {
        for (int col = 0; col < cols; col++)
        {
            if (matrix(row, col) == 0)
            {
                #pragma omp critical
                {
                    has_zero = true;
                }
                #pragma omp cancel for 
            }
        }    
        #pragma omp cancellation point for
    }
}
{% endhighlight %}

The source code of all three versions is available
[here](https://github.com/jakaspeh/concurrency/blob/master/ompCancel.cpp).

Comparisons
===========

We compare the execution time of the sequential implementation, the
implementation without explicit cancellation points and the implementation with
a cancellation point.

The size of the matrix is `{%raw%}8000 x 8000{%endraw%}`. Each entry in the
matrix is equal to one, except the entry at the position `{%raw%}(7000,
10){%endraw%}`, which is equal to zero.

The first comparison gives us the following results:

{% highlight c++ %}
$ ./ompCancel 
Does OpenMP allow cancel construct: 0
Sequential implementation: 
Time: 0.03917
Parallel implementation (without cancellation point): 
Time: 0.0351697
Parallel implementation (with cancellation point): 
Time: 0.031086
{% endhighlight %}

In this experiment, we forgot to set `{%raw%}OMP_CANCELLATION{%endraw%}` to
`{%raw%}true{%endraw%}`. The execution times are quite similar but still
different enough that they might confuse us :-).

Now, we do the comparison the right way:

{% highlight c++ %}
$ export OMP_CANCELLATION=true
$ ./ompCancel 
Does OpenMP allow cancel construct: 1
Sequential implementation: 
Time: 0.0412532
Parallel implementation (without cancellation point): 
Time: 0.0188502
Parallel implementation (with cancellation point): 
Time: 1.81924e-05
{% endhighlight %}

We can see that the parallel implementation without explicit cancellation points
is faster than the sequential version. But the parallel implementation which has
the additional cancellation point heavily outperforms the other two
implementations.

#### Confession

I have a confession to make. In the previous example I chose such a matrix
that the parallel implementations outperformed the sequential one.

If we choose a different matrix, we get different results. 

For example, we get the following results for a matrix of the size `{%raw%}8000
x 8000{%endraw%}` with only one zero entry, which is at the position `{%raw%}(0,
100){%endraw%}`:

{% highlight c++ %}
$ export OMP_CANCELLATION=true
$ ./ompCancel 
Does OpenMP allow cancel construct: 1
Sequential implementation: 
Time: 2.391e-07
Parallel implementation (without cancellation point): 
Time: 0.0196065
Parallel implementation (with cancellation point): 
Time: 1.80495e-05
{% endhighlight %}

For this matrix, the sequential version is the fastest.

#### Conclusion

The performance of the paralellization heavily depends on the input.

This means that we should analyze the performance for expected sets of inputs
before we choose a final implementation.

Summary
-------

In this article, we first summarized the cancellation model. Then, we explained
the two main constructs of the model: `{%raw%}cancel{%endraw%}` and
`{%raw%}cancellation point{%endraw%}`. In the end, we demonstrated the usage of
the constructs with an example. We parallelized a program which checks if a
matrix has a zero entry.

The parallel implementation does not always outperform the sequential one. We
should analyze how the performance behaves for the desired sets of inputs. Then
we can decide if it is worth using the parallel implementation.

Links: 

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/ompCancel.cpp)
* [The cancel construct (OpenMP API, page 172)](http://www.openmp.org/mp-documents/openmp-4.5.pdf)
* [The cancellation point construct (OpenMP API, page 176)](http://www.openmp.org/mp-documents/openmp-4.5.pdf)

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
* [OpenMP: OpenMP: Critical construct and zero matrix entries](/blog/2016/07/omp-critical.html)
* [OpenMP: Thread cancellation model](/blog/2016/07/omp-thread-cancellation-model.html)

