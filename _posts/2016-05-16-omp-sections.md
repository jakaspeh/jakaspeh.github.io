---
layout: post
title:  "OpenMP: Sections"
date:   2016-05-16
categories: cpp
---

We have several tasks which can be executed in parallel.  Is it possible to run
each task in its own thread?

We [already know that this is possible with the C++11 standard
library](/blog/2015/12/hello-concurrent-world.html). If we have two
functions

{% highlight c++ %}
void function_1();

void function_2();
{% endhighlight %}

we can run them in their own threads:

{% highlight c++ %}
std::thread thread_1(function_1);
std::thread thread_2(function_2);
    
thread_1.join();
thread_2.join();
{% endhighlight %}

Is something similar possible with OpenMP? The answer is: yes. 

Sections
--------

The section construct is one way to distribute different tasks to different
threads. The syntax of this construct is:

{% highlight c++ %}
#pragma omp parallel 
{
    #pragma omp sections
    {
        #pragma omp section
        {
            // structured block 1
        }
        
        #pragma omp section
        {
            // structured block 2
        }

        #pragma omp section
        {
            // structured block 3
        }           
        
        ...
    }
}
{% endhighlight %}

The `{%raw%}parallel{%endraw%}` construct creates a team of threads which
execute in parallel.

The `{%raw%}sections{%endraw%}` construct indicates the start of the
construct. It contains several `{%raw%}section{%endraw%}` constructs. Each
`{%raw%}section{%endraw%}` marks the different block, which represents a
task. Be aware, the use of singular and plural nouns (`{%raw%}section{%endraw%}`
/ `{%raw%}sections{%endraw%}`) is important!

The `{%raw%}sections{%endraw%}` construct distributes the blocks/tasks between
existing threads. The requirement is that each block must be independent of the
other blocks. Then each thread executes one block at a time. Each block is
executed only once by one thread. 

There are no assumptions about the order of the execution. The distribution
of the tasks to the threads is implementation defined. If there are more tasks
than threads, some of the threads will execute multiple blocks. If there are
more threads than tasks, some of the threads will be idle.

If there is only one `{%raw%}sections{%endraw%}` construct inside the
`{%raw%}parallel{%endraw%}` construct

{% highlight c++ %}
#pragma omp parallel
{
    #pragma omp sections
    {
        ...
    }
}
{% endhighlight %}

we can simplify both constructs into the combined construct

{% highlight c++ %}
#pragma omp parallel sections
{
    ...
}
{% endhighlight %}

The documentation for the `{%raw%}sections{%endraw%}` construct is available in
the [OpenMP specification](http://www.openmp.org/mp-documents/openmp-4.5.pdf) on
the page 65.

Example
-------

Let us present a simple example. 

We have two functions

{% highlight c++ %}
void function_1()
{
    for (int i = 0; i != 3; i++)
    {
        std::cout << "Function 1 (i = " << i << ")" << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
}

void function_2()
{
    for (int j = 0; j != 4; j++)
    {
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
        std::cout << "                   Function 2 "
                  << "(j = " << j << ")" << std::endl;
    }
}
{% endhighlight %}

which print some information. We use the `{%raw%}sleep_for{%endraw%}` function
to delay the printing between the iterations. 

The main function is 

{% highlight c++ %}
int main()
{
    #pragma omp parallel sections
    {    
        #pragma omp section
        function_1();
            
        #pragma omp section
        function_2();
    }

    return 0;
}
{% endhighlight %}

At the beginning, the combined construct creates a team of threads. The OpenMP
distributes the function calls `{%raw%}function_1();{%endraw%}` and
`{%raw%}function_2();{%endraw%}` between the available threads. 

If the number of threads is higher than one, then the program executes the
function calls in parallel. If the number of threads is equal to one, than the
program executes the function calls sequentially.

The timeline of the program is presented in the picture (if the number of
threads is higher than one).

![Sections](/pics/section_construct.png)

The output of the program is 

{% highlight c++ %}
$ ./ompSection 
Function 1 (i = 0)
                   Function 2 (j = 0)
Function 1 (i = 1)
                   Function 2 (j = 1)
                   Function 2 (j = 2)
Function 1 (i = 2)
                   Function 2 (j = 3)
{% endhighlight %}

We can clearly see that the functions are executed in parallel.

We can also run the program with only one thread. In this case, the output is

{% highlight c++ %}
$ export OMP_NUM_THREADS=1
$ ./ompSection 
Function 1 (i = 0)
Function 1 (i = 1)
Function 1 (i = 2)
                   Function 2 (j = 0)
                   Function 2 (j = 1)
                   Function 2 (j = 2)
                   Function 2 (j = 3)

{% endhighlight %}

Summary
-------

In this article, we learned how to run several tasks in parallel with the OpenMP
API.

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/ompSection.cpp)
* [Jaka's Corner: Introduction to OpenMP](/blog/2016/04/omp-introduction.html)
* [Jaka's Corner: OpenMP: For](/blog/2016/05/omp-for.html)


