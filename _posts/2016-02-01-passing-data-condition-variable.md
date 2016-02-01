---
layout: post
title:  "Passing data with condition variable"
date:   2016-02-01
categories: concurrency
---

Condition variables were the topic of the [previous blog
post](/blog/2016/01/condition-variable.html). I recommend reading it before
starting with this blog post.

Today, we will see another use case of a condition variable: sharing data
between the threads.


Work, work, work
----------------

Let's imagine the usual working environment. There is a boss who delegates
tasks, and there is a worker who carries out the tasks. We can associate the
people with the two different processes. One process is delegating the work and
the other one is doing the work. In order to have a successful company, these
processes must be synchronized.

Let's transform the boss--worker example into a concurrent code. We need to keep
in mind two important facts while reading the code below.

* The two threads will represent the boss and the worker.

* The condition variable will synchronize the tasks between the two threads. 

We start with include statements and global variables. 

{% highlight c++ %}
#include <iostream>
#include <string>
#include <queue>
#include <random>
#include <chrono>

#include <thread>
#include <mutex>
#include <condition_variable>

std::mutex MUT;
std::queue <std::string> TASKS;
std::condition_variable CV;

int seed = std::chrono::system_clock::now().time_since_epoch().count();

std::default_random_engine GENERATOR(seed);
std::uniform_int_distribution<int> DISTRIBUTION(0, 4);
std::vector<std::string> TODO = {"Write program",
                                 "Fix bug",
                                 "Write unit tests",
                                 "Bring me coffee",
                                 "Clean my car"};
{% endhighlight %}

Condition variable is associated with:

* The mutex `{%raw%}MUT{%endraw%}`, which helps with the synchronization.

* The queue `{%raw%}TASKS{%endraw%}`, which passes data between the two
threads.

There is also a `{%raw%}TODO{%endraw%}` list and a random distribution. The boss
will randomly choose a task from the `{%raw%}TODO{%endraw%}` list and assign it
to the worker. 

The next section of the code defines a behavior of the boss.

{% highlight c++ %}
void add_task(const std::string& task)
{
    std::lock_guard<std::mutex> guard(MUT);
    std::cout << "Boss delegates: " << task << std::endl;
    TASKS.push(task);
    CV.notify_one();
}

void add_random_task()
{
    add_task(TODO[DISTRIBUTION(GENERATOR)]);
}

void go_home()
{
    add_task("go home");
}

void take_break()
{
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
}
                                 
void boss()
{
    for (int i = 0; i != 5; i++)
    {
        add_random_task();
        take_break();
    }
    
    go_home();
}
{% endhighlight %}

The `{%raw%}boss(){%endraw%}` function assigns five tasks to the worker and then
instructs the worker to go home. Of course the boss takes multiple breaks
between her/his work. 

The `{%raw%}add_task{%endraw%}()` function is the most interesting for us. It
gets a string with the name of the task. Then, it locks the
`{%raw%}MUT{%endraw%}`, because we will modify the shared queue by pushing the
task onto the `{%raw%}TASKS{%endraw%}`.  At the end, the function notifies the
waiting thread with the `{%raw%}notify_one(){%endraw%}` member function of the
condition variable. 

The next section of the code defines a behavior of the worker.
    
{% highlight c++ %}
void worker()
{
    while (true)
    {
        std::unique_lock<std::mutex> guard(MUT);
        CV.wait(guard, []{return !TASKS.empty();});
        std::string task = TASKS.front();
        TASKS.pop();
        
        std::cout << "Worker doing:   " << task << std::endl;
        
        if (task == "go home")
        {
            break;
        }
    }
}
{% endhighlight %}

The function contains an infinite loop. The second line of the body of the loop
guarantees that the `{%raw%}TASKS{%endraw%}` queue is not empty. This means that
the boss already assigned some tasks. We explained the behavior of the
`{%raw%}wait(){%endraw%}` member function in [the previous blog
post](/blog/2016/01/condition-variable.html). Check it out if you don't
understand the guarantee.

Afterwards, the function gets and prints a task from the queue. The loop
terminates, if the boss commands the worker to go home.

The `{%raw%}worker(){%endraw%}` basically waits for the tasks and prints them
until it gets the special task: `{%raw%}"go home"{%endraw%}`. Then the function
returns. This is an example of how to wait and process data which comes in chunks
from another thread.

The main function creates two threads which represent the boss and the worker.

{% highlight c++ %}
int main()
{
    std::thread t1(boss);
    std::thread t2(worker);
    
    t1.join();
    t2.join();

    return 0;
}
{% endhighlight %}

You can look at the entire source code
[here](https://github.com/jakaspeh/concurrency/blob/master/conditionVariable2.cpp).

Running the example
-------------------

One possible output might be the following.
{% highlight bash %}
$ ./conditionVariable2
Boss delegates: Write program
Worker doing:   Write program
Boss delegates: Fix bug
Worker doing:   Fix bug
Boss delegates: Write program
Worker doing:   Write program
Boss delegates: Write unit tests
Worker doing:   Write unit tests
Boss delegates: Bring me coffee
Worker doing:   Bring me coffee
Boss delegates: go home
Worker doing:   go home
{% endhighlight %}

And the corresponding timeline is in the picture below.

![Condition variable with mother and son.](/pics/boss_worker_condition_variable.png)

The red color represents the boss and the blue color represents the worker. The
transmission of the data with the queue is displayed with the black color in the
middle.

When running the example, comment out the function
`{%raw%}take_break(){%endraw%}` in the `{%raw%}boss(){%endraw%}`
and run the program multiple times. Then figure out what happens.

Summary
-------

We learned how to pass data between the threads using a queue and a condition
variable. 
        
Links:

* [Source
  code](https://github.com/jakaspeh/concurrency/blob/master/conditionVariable2.cpp)
* [Jaka's Corner: Condition variable](/blog/2016/01/condition-variable.html)