---
layout: post
title:  "Why are plain threads dangerous"
date:   2015-12-28
categories: concurrency
---

When we start a thread, there are two options. Either we will wait for the
thread to finish or we will not wait for it. A mechanism for waiting is 
a `{%raw%}.join(){%endraw%}` member function. This is what we did in the [first
blog post](/blog/2015/12/hello-concurrent-world.html). The
`{%raw%}.detach(){%endraw%}` member function is the other option: not waiting
for the thread to finish.

Therefore, we must choose between these two options.  If we don't detach or join
the thread, the program will crash! If you don't believe me, comment a
`{%raw%}.join(){%endraw%}` call in
[helloWorld.cpp](https://github.com/jakaspeh/concurrency/blob/master/helloWorld.cpp),
compile and run the program. You should get an error message:

{% highlight bash %}
...
terminate called without an active exception
Aborted (core dumped)
...
{% endhighlight %}

Two options
-----------

Thus, we **must** call either `{%raw%}join{%endraw%}` or
`{%raw%}detach{%endraw%}`. The second option is not problematic. The
`{%raw%}.detach(){%endraw%}` is invoked right after the construction of a
thread.

The first approach, however, has some problems. Invoking the
`{%raw%}.join(){%endraw%}` right after constructor doesn't make sense, because
there is no need for the new thread in this case.  Therefore, there is some code
between the construction of the thread and the `{%raw%}.join(){%endraw%}` call.

{% highlight c++ %}
int main()
{
    std::thread t(function);

    ...
    some code
    ...
    
    t.join();

    return 0;
}
{% endhighlight %}

But this code can emit an exception, which consequently means that the
`{%raw%}.join(){%endraw%}` will not be invoked and the program will crash.

![Exception before join](/pics/exception.png)

The solution is a `{%raw%}Guard{%endraw%}` class. Let us have a look at it.

{% highlight c++ %}
#include <iostream>
#include <thread>

class Guard
{
public:    
    Guard(std::thread& t)
        : thread(t)
    { }
    
    ~Guard()
    {
        if (thread.joinable())
        {
            thread.join();
        }
    }
    
    Guard(Guard& other)=delete;
    Guard& operator=(const Guard& rhs)=delete;

private:
    std::thread& thread;
};

void function()
{
    std::cout << "I'm inside function." << std::endl;
}

int main()
{
    std::thread t(function);
    Guard guard(t);

    return 0;
}
{% endhighlight %}

An object of the class has a reference to a thread. The destructor checks if the
thread is joinable and then joins it. A thread is joinable if
`{%raw%}.join(){%endraw%}` or `{%raw%}.detach(){%endraw%}` were not called
yet. A guard object can not be copied, because we declared copy constructor and
copy assignment operator with `{%raw%}=delete{%endraw%}`. Inability to copy
means that the object can not outlive the scope of referenced thread.

This class guarantees that `{%raw%}.join(){%endraw%}` member function will be
called in any case. When the function ends, the destructors of all objects in
the scope are called before the program exits. This also happens if the function
raises an exception and ends prematurely.

Acknowledgment
--------------

Thanks to **Nino Bašić** for comments and suggestions. Now the post looks more
organized!

Summary
-------

We learned why are the plain threads dangerous and how to make them safer with a 
`{%raw%}Guard{%endraw%}` class.

In the next blog post, we will look how to pass an argument to a thread. 



Links: 

- [Source code](https://github.com/jakaspeh/concurrency/blob/master/guard.cpp)

- [std::thread::join](http://en.cppreference.com/w/cpp/thread/thread/join)

- [std::thread::detach](http://en.cppreference.com/w/cpp/thread/thread/detach)

- [std::thread::joinable](http://en.cppreference.com/w/cpp/thread/thread/joinable)