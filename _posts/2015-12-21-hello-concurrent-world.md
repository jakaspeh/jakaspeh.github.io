---
layout: post
title:  "Hello concurrent world"
date:   2015-12-21
categories: concurrency
---

Our aim is to write a simple concurrent C++ program using C++11 standard
library.  More precisely, we would like to write a program that executes two or
more separate instructions at the same time.

What is a thread?
-----------------

We use threads to write concurrent programs. But what is a thread? If you are
looking over the internet, people have many answers to this question. Everyone has
its own explanation, but I was not able to find a simple answer. Therefore, I will
give my simple definition of threads (I don't claim that it is the right one).

A thread is a set of instructions for a computer. When you execute the program,
it creates the main thread. The program is concurrent, if the main thread spawns
other threads. 

Don't worry if this doesn't make any sense now. We will get better understanding
about the threads through examples. 

First concurrent program
------------------------

We would like to have a program which does two different tasks -- each in their
own thread. As always, the main function will start the main thread. The main
thread will spawn two additional threads, which will execute their tasks.

![First concurrent program](/pics/two_threads.png)

The first concurrent program `{%raw%}helloWorld.cpp{%endraw%}` is listed below.

{% highlight c++ %}
#include <iostream>
#include <thread>
#include <chrono>

void function_1()
{
    for (int i = 0; i != 4; i++)
    {
        std::cout << "Function 1 i = " << i << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
}


void function_2()
{
    for (int j = 0; j != 4; j++)
    {
        std::cout << "                   Function 2 j = " 
                  << j << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
}


int main()
{
    std::thread thread_1(function_1);
    std::thread thread_2(function_2);
    
    thread_1.join();
    thread_2.join();
    
    return 0;
}
{% endhighlight %}

We must compile it with C++ compiler which supports C++11 standard. Additionally,
we must add the flags -pthread -std=c++11. For example, on Linux, the command
looks like:

{% highlight bash %}
$ g++ helloWorld.cpp -pthread -std=c++11 -o helloWorld
{% endhighlight %}

The output of the program is 
{% highlight bash %}
$ ./helloWorld
Function 1 i = 0
                   Function 2 j = 0
                   Function 2 j = 1
Function 1 i = 1
                   Function 2 j = 2
                   Function 2 j = 3
Function 1 i = 2
Function 1 i = 3
{% endhighlight %}


Let's explain it.  We included `{%raw%}<thread>{%endraw%}` and
`{%raw%}<chrono>{%endraw%}` libraries. First one allows us to write concurrent
programs and the second one is used for managing the time.

Then, there are two functions, which are quite similar. Both are iterating
certain number of times. In each iteration, they print some information to
standard output and then wait for 1 or 0.5 second.

The most interesting part is the main function. It creates two threads. Input
argument for a thread constructur is a function. The function will be executed
immediately in the thread. (`{%raw%}function_1{%endraw%}` will be executed
immediately in `{%raw%}thread_1{%endraw%}`.) Then the `{%raw%}.join(){%endraw%}`
member function is called. This forces current thread to wait for the other one.
(`{%raw%}thread_1.join(){%endraw%}` forces main thread to wait for the
`{%raw%}thread_1{%endraw%}` to finish.) If we don't `{%raw%}.join(){%endraw%}`
our threads, we end up in the land of undefined behavior. 

Summary
-------

We used threads to write our first concurrent program. But using plain threads is
considered a bad programming practice. However, my opinion is that it is good to
know them because they are a basic building block of concurrent programs. 

In the next articles we will describe how to pass a function with arguments to a
thread and investigate why are the plain threads dangerous. 

Links: 

* [Code](https://github.com/jakaspeh/concurrency/blob/master/helloWorld.cpp)

* [std::thread](http://en.cppreference.com/w/cpp/thread/thread)

* [std::this_thread::sleep_for](http://en.cppreference.com/w/cpp/thread/sleep_for)








