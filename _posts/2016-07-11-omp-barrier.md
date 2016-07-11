---
layout: post
title:  "OpenMP: Barrier"
date:   2016-07-11
categories: cpp, openmp
---

What is a barrier?

It is a point in the execution of a program where threads wait for each
other. No thread is allowed to continue until all threads in a team reach the
barrier.

Basically, a barrier is a synchronization point in a program. We can visualize it
with a wall.

![A barrier](/pics/omp_barrier.png)

In the figure, the red threads are waiting at the wall for the blue threads. The
red threads can not go beyond the wall. They can proceed only when all threads
reach the wall.

Pros and cons
-------------

The main reason for a barrier in a program is to avoid data races and to ensure
the correctness of the program. 

Of course there are some downsides. Each synchronization is a threat for
performance. When a thread waits for other threads, it does not do any useful
work and it spends valuable resources.

Another problem might occur if we are not carefully inserting barriers. As soon
as one thread reaches the barrier then all threads in the team must reach the
barrier. Otherwise, the threads waiting at the barrier will wait forever (except
if we use a cancel construct, but this is a topic for another article).

The following figure shows how a couple of blue threads avoids the barrier. In
this case, the red threads will wait forever for the blue threads.

![Barrier cheaters.](/pics/omp_barrier_cheaters.png)

How to add a barrier to a program?
----------------------------------

We can explicitly insert a barrier in a program by adding the barrier construct:

{% highlight c++ %}
#pragma omp barrier
{% endhighlight %}

This is an explicit way of adding a barrier.

There are also many other situations, where a compiler inserts a barrier instead
of us. This happens because many OpenMP constructs imply a barrier. For example,
the parallel construct implies a barrier in the end of the parallel region. The
loop construct implies a barrier in the end of the loop. The single construct
implies a barrier in the end of the single region.

However, there are also OpenMP constructs which do not imply a barrier. The
master construct is such example. This construct is very similar to the single
construct: the code inside the master construct is executed by only one (master)
thread. But the difference is that the master construct does not imply a barrier
while the single construct does.

How can we figure out which constructs imply a barrier and which do not?

We have to look at the [OpenMP
specification](http://www.openmp.org/mp-documents/openmp-4.5.pdf). The
description of each construct contains the information about the existence of
the barrier.


Avoiding the implicit barriers
------------------------------

A natural question that arises is: Can we omit the implicit barriers?

This depends on the constructs. Some constructs support the removal of a
barrier, while the others do not support such a feature. Again, [OpenMP
specification](http://www.openmp.org/mp-documents/openmp-4.5.pdf) can tell us if
a construct supports this feature.


The loop construct supports the removal of a barrier. A programmer can then omit
the barrier by adding `{%raw%}nowait{%endraw%}` clause to the loop construct.

{% highlight c++ %}
#pragma omp parallel
{
    #pragma omp for nowait
    for (...)
    {
        // for loop body
    }

    // next instructions
}
{% endhighlight %}

In this case the thread that finishes early proceeds straight to the next
instruction and does not wait for the other threads in the team.

Using the nowait clause can improve the performance of a program. But we must be
careful, because removing a barrier might introduce a data race.

An example
----------

[In the article about the single construct](/blog/2016/06/omp-single.html), we
presented several programs which accumulate the salaries of all employees in two
companies. The third version was the following:

{% highlight c++ %}
#pragma omp parallel shared(salaries1, salaries2)
{
    #pragma omp for reduction(+: salaries1)
    for (int employee = 0; employee < 25000; employee++)
    {
        salaries1 += fetchTheSalary(employee, Co::Company1);
    }

    #pragma omp single
    {
        std::cout << "Salaries1: " << salaries1 << std::endl;
    }

    #pragma omp for reduction(+: salaries1)
    for (int employee = 0; employee < 25000; employee++)
    {
        salaries2 += fetchTheSalary(employee, Co::Company2);
    }
}

std::cout << "Salaries2: " << salaries2 << std::endl;
{% endhighlight %}

[Mats Brorsson](https://www.linkedin.com/in/matsbrorsson) [commented on
LinkedIn](https://www.linkedin.com/hp/update/6151752513759506432) that this
version suffers from oversynchronization (read as: it has too many barriers). I
agree!  Therefore, we now explain the problem with the program and different
solutions to the problem.

The key is to notice where are the implicit barriers. They are 

* in the end of the parallel region,
* in the end of the first for loop,
* in the end of the single construct and
* in the end of the second for loop. 

![Examples of barriers](/pics/omp_barrier_example.png)

Let us analyze each barrier. 

The first barrier is in the end of the first for loop. If we omit the barrier
there, we might introduce a data race. This is because the next instruction
after the for loop accesses the reduction variable:
`{%raw%}salaries1{%endraw%}`. Without the barrier, one thread might access
`{%raw%}salaries1{%endraw%}` for printing while some other thread might still
update the value of the `{%raw%}salaries1{%endraw%}`. Therefore, we should not
add `{%raw%}nowait{%endraw%}` clause to the first for loop. 

The second barrier is in the end of the single construct. In the single
construct, the program prints the value of `{%raw%}salaries1{%endraw%}`. This is
the last time when the program reads/writes `{%raw%}salaries1{%endraw%}`. The
next instructions already compute `{%raw%}salaries2{%endraw%}`. Because of this
independence, we can safely remove the barrier in the end of the single
construct. We can do this by inserting the nowait clause.

There is also another option. We can replace the single construct with the
master construct. The master construct is very similar to the single
construct. The main differences are that the master construct is executed by the
master thread and that the master construct does not imply a barrier.

There are two more barriers left. They are both in the end of the parallel
region. The parallel construct does not support the nowait clause. Thus, the
only possibility to eliminate the barrier is in the end of the second loop. The
elimination does not introduce a data race, because there exists the barrier of
the parallel construct, which synchronizes the threads. Therefore, it is safe to
omit the implicit barrier in the end of the second loop. Note that a
compiler might do this automatically.

Summary
-------

We studied barriers. We explained how to add a barrier to a program and how a
compiler adds implicit barriers to a program. We showed how to omit an implicit
barrier with the nowait clause. 

In the end, we analyzed implicit barriers of an example. The valid removals of
barriers might improve the efficiency of a program. Of course, we should measure
it to check if this really is the case.

Thanks to [Mats Brorsson](https://www.linkedin.com/in/matsbrorsson) for giving
me the idea for this article.

Links:

* [The barrier construct, OpenMP specification, page
   151](http://www.openmp.org/mp-documents/openmp-4.5.pdf) 
* [The master construct, OpenMP specification, page
  148](http://www.openmp.org/mp-documents/openmp-4.5.pdf)

Jaka's Corner OpenMP series:

* [OpenMP: Introduction](/blog/2016/04/omp-introduction.html)
* [OpenMP: For](/blog/2016/05/omp-for.html)
* [OpenMP: Sections](/blog/2016/05/omp-sections.html)
* [OpenMP: Monte Carlo method for Pi](/blog/2016/05/omp-monte-carlo-pi.html)
* [OpenMP: For & Reduction](/blog/2016/06/omp-for-reduction.html)
* [OpenMP: For & Scheduling](/blog/2016/06/omp-for-scheduling.html)
* [OpenMP: Data-Sharing Rules](/blog/2016/06/omp-data-sharing-attributes.html)
* [OpenMP: default(none) and const variables](/blog/2016/07/omp-default-none-and-shared.html)




