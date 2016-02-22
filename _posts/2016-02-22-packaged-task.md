---
layout: post
title:  "Packaged task"
date:   2016-02-22
categories: concurrency
---

The `{%raw%}std::async{%endraw%}` is not the only way to associate a
`{%raw%}std::future{%endraw%}` with a task. Using the
`{%raw%}std::packaged_task{%endraw%}` is also a possible solution.

Packaged task
-------------

We would like to represent the result of the function 

{% highlight c++ %}
int complicated_computation(double, bool);
{% endhighlight %}

with a future using a packaged task. This is done in three steps. 

### *Creation* ###

First, we create a packaged task:

{% highlight c++ %}
std::packaged_task< int(double, bool) > task(complicated_computation);
{% endhighlight %}

The constructor of the `{%raw%}std::packaged_task{%endraw%}` requires a function
(callable target) as an argument. The template parameter is the signature of the
function. The signature is `{%raw%}int(double, bool){%endraw%}` because
`{%raw%}complicated_computation{%endraw%}` returns `{%raw%}int{%endraw%}` and
takes arguments of types `{%raw%}double{%endraw%}` and `{%raw%}bool{%endraw%}`. 

### *Future* ###

The packaged task has a member function
`{%raw%}.get_future(){%endraw%}` which returns a future. 

{% highlight c++ %}
std::future< int > future = task.get_future();
{% endhighlight %}

The template parameter of the `{%raw%}std::future{%endraw%}` must be the same as
the return type of the function which was the argument for the packaged task
constructor. In our example, the template parameter must be
`{%raw%}int{%endraw%}` because `{%raw%}complicated_computation{%endraw%}` (which
returns `{%raw%}int{%endraw%}`) was the argument for the `{%raw%}task{%endraw%}`
constructor.

### *Execution* ###

The packaged task is a callable object. The `{%raw%}task{%endraw%}` forwards the
arguments to the supplied function.

{% highlight c++ %}
task(2.5, true);
{% endhighlight %}

The upper call runs the `{%raw%}complicated_computation{%endraw%}` with
arguments `{%raw%}(2.5, true){%endraw%}` and stores the result of the function
in the future. 

We can pass the packaged task around and call it at a different place than where
it was created. This is the benefit of packaged tasks: decoupling the creation
of the future with the execution of the task.

#### *Typical usage* ####

The typical usage of a packaged task is:

* creating the packaged task with a function,

* retrieving the future from the packaged task, 

* passing the packaged task elsewhere,

* invoking the packaged task. 


Temperature tomorrow revisited
------------------------------

We will use a packaged task on the example from [the previous article](/blog/2016/02/futures.html).

The story was: A couple, wife and husband, are going on a picnic tomorrow. The
wife would like to know what will be the temperature of the weather. Therefore,
she asks her husband to look it up.

The two functions from the last example 

{% highlight c++ %}
void make_break(int millisec);

int temperature();
{% endhighlight %}

stay the same. The `{%raw%}make_break(){%endraw%}` stops current thread for
given amount of milliseconds and the `{%raw%}temperature(){%endraw%}` returns the
temperature, which the husband looked up. The `{%raw%}main{%endraw%}` function
is different

{% highlight c++ %}
int main()
{
    std::cout << "Wife:    Tomorrow, we are going on a picnic.\n" 
              << "         What will be the weather...\n" 
              << "         \"What will be the "
              << "temperature tomorrow?\"" << std::endl;
    
    std::packaged_task< int() > task(temperature);
    std::future<int> answer = task.get_future();
    std::thread thread(std::move(task));
    thread.detach();
    
    make_break(2);
    
    std::cout << "Wife:    I should pack for tomorrow." << std::endl;
    
    make_break(2);

    std::cout << "Wife:    Hopefully my husband can figure out the weather soon."
              << std::endl;
    
    int temp = answer.get();

    std::cout << "Wife:    Finally, tomorrow will be " << temp << "... Em...\n"
              << "         \"In which units is the answer?\"" 
              << std::endl;

    return 0;
}
{% endhighlight %}

The interesting part is where the constructor of `{%raw%}std::package_task{%endraw%}`
creates a packaged task with the
`{%raw%}temperature(){%endraw%}`. Then, the future `{%raw%}answer{%endraw%}` is
retrieved from the `{%raw%}task{%endraw%}`. Afterwards, we pass the task to a
separate thread. The packaged task is not copyable, therefore
`{%raw%}std::move{%endraw%}` moves it to the thread. The thread is detached and
computes the result asynchronously.

If the `{%raw%}temperature{%endraw%}` would have additional input arguments,
they would be listed in the `{%raw%}std::thread{%endraw%}` constructor. 

The entire source code is available
[here](https://github.com/jakaspeh/concurrency/blob/master/packagedTask.cpp). The
output of the program is:

{% highlight bash %}
$ ./packagedTask
Wife:    Tomorrow, we are going on a picnic.
         What will be the weather...
         "What will be the temperature tomorrow?"
Husband: Hm, is the weather forecast in the newspaper?
         Eh, we don't have a newspaper at home...
Wife:    I should pack for tomorrow.
Husband: I will look it up on the internet!
Wife:    Hopefully my husband can figure out the weather soon.
Husband: Here it is, it says tomorrow will be 40.
Wife:    Finally, tomorrow will be 40... Em...
         "In which units is the answer?"
{% endhighlight %}


This is a possible usage of a packaged task but it is not a common one. Using
`{%raw%}std::async{%endraw%}` is a lot easier in this case. But it is useful to
start exploring packaged tasks with a familiar example. 

Summary 
-------

The `{%raw%}std::packaged_task{%endraw%}` is another way to associate a task
with a `{%raw%}std::future{%endraw%}` -- the future representation of the result
of the task.


In the next article, we will give another example of usage of packaged
tasks. It will take advantage of the fact that the execution of the task and the
retrieval of the future from the packaged task is not so tightly connected as in
`{%raw%}std::async{%endraw%}`. 

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/packagedTask.cpp)

* [std::packaged_task](http://en.cppreference.com/w/cpp/thread/packaged_task)

