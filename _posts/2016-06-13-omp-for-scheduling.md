---
layout: post
title:  "OpenMP: For & Scheduling"
date:   2016-06-13
categories: cpp, openmp
---

OpenMP specialty is a parallelization of loops. The loop construct
enables the parallelization.

{% highlight c++ %}
#pragma omp parallel for
for (...)
{ ... }
{% endhighlight %}

OpenMP then takes care about all the details of the parallelization. It creates
a team of threads and distributes the iterations between the threads.

In this article, we explore how OpenMP schedules the iterations between the
threads and how can we change this behavior. First, we examine different ways to
specify a scheduling type of a loop. Then, we describe five different scheduling
types.


Scheduling
----------

We describe how a programmer can determine a scheduling type of a loop. 


Explicit
========

If the loop construct has explicit `{%raw%}schedule{%endraw%}` clause

{% highlight c++ %}
#pragma omp parallel for schedule(scheduling-type)
for (...)
{ ... }
{% endhighlight %}

then OpenMP uses `{%raw%}scheduling-type{%endraw%}` for
scheduling the iterations of the for loop.


Runtime
=======

If the `{%raw%}scheduling-type{%endraw%}` (in the schedule clause of the loop
construct) is equal to `{%raw%}runtime{%endraw%}` then OpenMP determines the
scheduling by the internal control variable `{%raw%}run-sched-var{%endraw%}`. We
can set this variable by setting the environment variable
`{%raw%}OMP_SCHEDULE{%endraw%}` to the desired scheduling type. For example, in
`{%raw%}bash{%endraw%}`-like terminals, we can do

{% highlight c++ %}
$ export OMP_SCHEDULE=sheduling-type
{% endhighlight %}

Another way to specify `{%raw%}run-sched-var{%endraw%}` is to set it with
`{%raw%}omp_set_schedule{%endraw%}` function.

{% highlight c++ %}
...
omp_set_schedule(sheduling-type);
...
{% endhighlight %}


Default
=======

If the loop construct does not have an explicit `{%raw%}schedule{%endraw%}`
clause

{% highlight c++ %}
#pragma omp parallel for 
for (...)
{ ... }
{% endhighlight %}

then OpenMP uses the default scheduling type. It is defined by the internal
control variable `{%raw%}def-sched-var{%endraw%}` and it is implementation
dependent.

Summary
=======

The following figure summarizes how OpenMP determines a scheduling type of a for
loop.

![Determining a scheduling type](/pics/omp_scheduling.png)



The scheduling types
--------------------

We can choose between five different scheduling types:

* `{%raw%}static{%endraw%}`,
* `{%raw%}dynamic{%endraw%}`,
* `{%raw%}guided{%endraw%}`,
* `{%raw%}auto{%endraw%}` and
* `{%raw%}runtime{%endraw%}`. 


Static
======

The `{%raw%}schedule(static, chunk-size){%endraw%}` clause of the loop construct
specifies that the for loop has the static scheduling type. OpenMP divides the
iterations into chunks of size `{%raw%}chunk-size{%endraw%}` and it distributes
the chunks to threads in a circular order. 

When no `{%raw%}chunk-size{%endraw%}` is specified, OpenMP divides iterations
into chunks that are approximately equal in size and it distributes at most one
chunk to each thread.

Here are three examples of static scheduling.

{% highlight c++ %}
schedule(static):      
****************                                                
                ****************                                
                                ****************                
                                                ****************
{% endhighlight %}

{% highlight c++ %}
schedule(static, 4):   
****            ****            ****            ****            
    ****            ****            ****            ****        
        ****            ****            ****            ****    
            ****            ****            ****            ****
{% endhighlight %}

{% highlight c++ %}
schedule(static, 8):   
********                        ********                        
        ********                        ********                
                ********                        ********        
                        ********                        ********
{% endhighlight %}

Let me explain the examples. We parallelized a for loop with 64 iterations and
we used four threads to parallelize the for loop. Each row of stars in the
examples represents a thread. Each column represents an iteration.

The first example (`{%raw%}schedule(static){%endraw%}`) has 16 stars in the
first row. This means that the first tread executes iterations 1, 2, 3, ..., 15
and 16. The second row has 16 blanks and then 16 stars. This means that the
second thread executes iterations 17, 18, 19, ..., 31, 32. Similar applies to
the threads three and four.

We see that for `{%raw%}schedule(static){%endraw%}` OpenMP divides iterations
into four chunks of size 16 and it distributes them to four threads. For
`{%raw%}schedule(static, 4){%endraw%}`  and `{%raw%}schedule(static,
8){%endraw%}` OpenMP divides iterations into chunks of
size 4 and 8, respectively. 

The static scheduling type is appropriate when all iterations have the same
computational cost.

Dynamic
=======

The `{%raw%}schedule(dynamic, chunk-size){%endraw%}` clause of the loop
construct specifies that the for loop has the dynamic scheduling type. OpenMP
divides the iterations into chunks of size `{%raw%}chunk-size{%endraw%}`. Each
thread executes a chunk of iterations and then requests another chunk until
there are no more chunks available.

There is no particular order in which the chunks are distributed to the
threads. The order changes each time when we execute the for loop.

If we do not specify `{%raw%}chunk-size{%endraw%}`, it defaults to one.

Here are four examples of dynamic scheduling. 


{% highlight c++ %}
schedule(dynamic):     
*   ** **  * * *  *      *  *    **   *  *  * *       *  *   *  
  *       *     *    * *     * *   *    *        * *   *    *   
 *       *    *     * *   *   *     *  *       *  *  *  *  *   *
   *  *     *    * *    *  *    *    *    ** *  *   *     *   * 
{% endhighlight %}

{% highlight c++ %}
schedule(dynamic, 1):  
    *    *     *        *   *    * *  *  *         *  * *  * *  
*  *  *   * *     *  * * *    * *      *   ***  *   *         * 
 *   *  *  *  *    ** *    *      *  *  * *   *  *   *   *      
  *    *     *  **        *  * *    *          *  *    *  * *  *
{% endhighlight %}

{% highlight c++ %}
schedule(dynamic, 4):  
            ****                    ****                    ****
****            ****    ****            ****        ****        
    ****            ****    ****            ****        ****    
        ****                    ****            ****            
{% endhighlight %}

{% highlight c++ %}
schedule(dynamic, 8):  
                ********                                ********
                        ********        ********                
********                        ********        ********        
        ********                                                
{% endhighlight %}

We can see that for `{%raw%}schedule(dynamic){%endraw%}` and
`{%raw%}schedule(dynamic, 1){%endraw%}` OpenMP determines similar
scheduling. The size of chunks is equal to one in both instances. The
distribution of chunks between the threads is arbitrary. 

For `{%raw%}schedule(dynamic, 4){%endraw%}` and `{%raw%}schedule(dynamic,
8){%endraw%}` OpenMP divides iterations into chunks of size four and eight,
respectively. The distribution of chunks to the threads has no pattern. 

The dynamic scheduling type is appropriate when the iterations require different
computational costs. This means that the iterations are poorly balanced between
each other. The dynamic scheduling type has higher overhead then the static
scheduling type because it dynamically distributes the iterations during the
runtime.

Guided
======

The guided scheduling type is similar to the dynamic scheduling type. OpenMP
again divides the iterations into chunks. Each thread executes a chunk of
iterations and then requests another chunk until there are no more chunks
available.

The difference with the dynamic scheduling type is in the size of chunks. The
size of a chunk is proportional to the number of unassigned iterations divided
by the number of the threads. Therefore the size of the chunks decreases. 

The minimum size of a chunk is set by `{%raw%}chunk-size{%endraw%}`. We
determine it in the scheduling clause: `{%raw%}schedule(guided,
chunk-size){%endraw%}`. However, the chunk which contains the last iterations
may have smaller size than `{%raw%}chunk-size{%endraw%}`.

If we do not specify `{%raw%}chunk-size{%endraw%}`, it defaults to one.

Here are four examples of the guided scheduling. 

{% highlight c++ %}
schedule(guided):      
                            *********                        *  
                ************                     *******  ***   
                                     *******                   *
****************                            *****       **    * 
{% endhighlight %}


{% highlight c++ %}
schedule(guided, 2):   
                ************                     ****     **    
                                     *******         ***    **  
                            *********                           
****************                            *****       **    **
{% endhighlight %}


{% highlight c++ %}
schedule(guided, 4):   
                                     *******                    
                ************                     ****    ****   
                            *********                           
****************                            *****    ****    ***
{% endhighlight %}


{% highlight c++ %}
schedule(guided, 8):   
                ************                 ********        ***
****************                                                
                                     ********                   
                            *********                ********
{% endhighlight %}

We can see that the size of the chunks is decreasing. First chunk has always 16
iterations. This is because the for loop has 64 iterations and we use 4 threads
to parallelize the for loop. If we divide `{%raw%}64 / 4{%endraw%}`, we get 16. 

We can also see that the minimum chunk size is determined in the schedule
clause. The only exception is the last chunk. Its size might be lower then the
prescribed minimum size.

The guided scheduling type is appropriate when the iterations are poorly balanced
between each other. The initial chunks are larger, because they reduce
overhead. The smaller chunks fills the schedule towards the end of the
computation and improve load balancing. This scheduling type is especially
appropriate when poor load balancing occurs toward the end of the computation.

Auto
====

The auto scheduling type delegates the decision of the scheduling to the
compiler and/or runtime system. 

In the following example, the compiler/system determined the static scheduling.

{% highlight c++ %}
schedule(auto):        
****************                                                
                ****************                                
                                ****************                
                                                ****************
{% endhighlight %}

Runtime
=======

The runtime scheduling type defers the decision about the scheduling until the
runtime. We already described different ways of specifying the scheduling type in
this case. One option is with the environment variable
`{%raw%}OMP_SCHEDULE{%endraw%}` and the other option is with the function
`{%raw%}omp_set_schedule{%endraw%}`.

Default
=======

If we do not specify the scheduling type in a for loop

{% highlight c++ %}
#pragma omp parallel for 
for (...)
{ ... }
{% endhighlight %}

OpenMP uses the default scheduling type (defined by the internal control
variable `{%raw%}def-sched-var{%endraw%}`).

If I do not specify the scheduling type in my machine, I get the following
result.

{% highlight c++ %}
default:               
****************                                                
                ****************                                
                                ****************                
                                                ****************
{% endhighlight %}

We recognize that the default scheduling type in my machine is the static
scheduling type.

Summary
=======

We learned different ways to specify a scheduling type of a loop. We also
described five different scheduling types.

Links:

* [Source code of all presented examples](https://github.com/jakaspeh/concurrency/blob/master/ompForSchedule.cpp)
* More information about the scheduling is available in the 
  [OpenMP specification](http://www.openmp.org/mp-documents/openmp-4.5.pdf)
  on the page 60. 


Jaka's Corner OpenMP series:

* [OpenMP: Introduction](/blog/2016/04/omp-introduction.html)
* [OpenMP: For](/blog/2016/05/omp-for.html)
* [OpenMP: Sections](/blog/2016/05/omp-sections.html)
* [OpenMP: Monte Carlo method for Pi](/blog/2016/05/omp-monte-carlo-pi.html)
* [OpenMP: For & Reduction](/blog/2016/06/omp-for-reduction.html)



