---
layout: post
title:  "OpenMP: Introduction"
date:   2016-04-25
categories: concurrency
---

When we write parallel/concurrent programs, we start with a sequential
version. The next step is to completely change the program by introducing
special parts of code, which make the program parallel. OpenMP helps us to write
programs which doesn't require a lot of modification to the original sequential
version. If the compiler does not support OpenMP, the program ideally still runs
in a sequential manner.

OpenMP
------

[OpenMP](http://openmp.org/wp/) is a programming model for 
parallel programming with a shared memory. It is [a specification /
API](http://www.openmp.org/mp-documents/openmp-4.5.pdf). The implementers of the
compilers look at the specification and they implement it. Therefore, the
compilers know how to compile a program which uses OpenMP.

In order to enable OpenMP for the C++ compiler, we must add the flag
`{%raw%}-fopenmp{%endraw%}` to the other compilation flags. For the GNU compiler, it
would look like this

{% highlight c++ %}
g++ program.cpp -fopenmp -o program
{% endhighlight %}

Parallelization
---------------

The multithreading in OpenMP programs is supported by the so-called [fork-join
programming model](https://en.wikipedia.org/wiki/Fork%E2%80%93join_model). The
idea is that at the beginning, we have one initial thread. When the initial
thread encounters OpenMP parallel construct, the thread creates (forks) a team
of threads, which runs in parallel. At the end of the parallel construct, the
team is joined and only the initial thread continues.

![Fork-Join](/pics/fork_join.png)

Programming in OpenMP
---------------------

There are three ways to use OpenMP functionalities:

* OpenMP API provides a set of functions. These are just like the ordinary C/C++
  functions. They all start with `{%raw%}omp_{%endraw%}`. An example of a
  function is `{%raw%}omp_get_thread_num(){%endraw%}`, which returns the
  identification number of the current thread. 

* Second way to use the OpenMP is via the environmental variables. They all start
  with `{%raw%}OMP_{%endraw%}`. An example is
  `{%raw%}OMP_NUM_THREADS{%endraw%}`, which sets the number of threads the
  program can use.

* The last way is to use the so-called pragmas. They all start
  with `{%raw%}#pragma omp{%endraw%}`. An example is 
  {% highlight c++ %}
#pragma omp parallel
{ 
    // structured block
}
  {% endhighlight %}
  which starts parallel execution of the structured block. 
  

First OpenMP program
--------------------

The first program creates a parallel region. 

{% highlight c++ %}
#include <iostream>
#include <omp.h>


int main ()  
{
    
#pragma omp parallel 
    {
        std::cout << "Thread = " << omp_get_thread_num()
                  << std::endl;


        if (0 == omp_get_thread_num()) 
        {
            std::cout << "Number of threads = "<< omp_get_num_threads()
                      << std::endl;
        }

    }
    
    return 0;
}
{% endhighlight %}

Since we use the function from the OpenMP specification, we must include the
header `{%raw%}<omp.h>{%endraw%}`. The `{%raw%}#pragma omp parallel{%endraw%}`
creates a team of threads. Then, the program prints the thread number of each
thread. If the thread number is equal to zero, the program prints the number of
threads in the team. The compiler chooses the number of threads for us if we do
not specify it.

The output of the program might look like:

{% highlight c++ %}
$ ./openmpStart 
Thread = 6
Thread = 2
Thread = 4
Thread = 0
Number of threads = 8
Thread = 7
Thread = 1
Thread = 5
Thread = 3
{% endhighlight %}

or

{% highlight c++ %}
$ ./openmpStart 
Thread = Thread = Thread = 1
Thread = Thread = 4Thread = 3
0
Number of threads = 8
Thread = 7
Thread = 2
6
5
{% endhighlight %}

We have a data race. The OpenMP does not automatically figure out that multiple
threads access `{%raw%}std::cout{%endraw%}`. It is a responsibility of the
programmer to state this in the code.

Setting the number of threads
-----------------------------

We can manually set the number of threads via the environment variable. For
example, in `{%raw%}bash{%endraw%}`-like terminal, we can do

{% highlight c++ %}
$ export OMP_NUM_THREADS=4
{% endhighlight %}

and then the output of the program might look like

{% highlight c++ %}
$ ./openmpStart 
Thread = 0
Number of threads = 4
Thread = 3
Thread = 2
Thread = 1
{% endhighlight %}

Summary
-------

In this article, we looked at the basics of the OpenMP specification. With the
help of the OpenMP, we wrote a simple multithreaded program.

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/openmpStart.cpp)
* [OpenMP](http://openmp.org/wp/)


