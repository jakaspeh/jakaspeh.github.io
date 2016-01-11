---
layout: post
title:  "Data race and mutex"
date:   2016-01-11
categories: concurrency
---

Data race
---------

The problem, which we had in [the last blog post](/blog/2016/01/arguments.html)
(`{%raw%}arguments.cpp{%endraw%}`), was the messy output of the program. Namely,
there were two sentences in one line and also some blank lines appeared:

{% highlight bash %}
$ ./arguments 
Today is a beautiful day.And sun is shining.

Today is a beautiful day.And sun is shining.
And sun is shining.
And sun is shining.
And sun is shining.
And sun is shining.

Today is a beautiful day.
Today is a beautiful day.
Today is a beautiful day.
Today is a beautiful day.
{% endhighlight %}

You might experience different outputs when you run the program on your 
computer. This problem is so typical for concurrent programming that it has its
own name: **data race**. A data race happens when two threads try to change a
resource at the same time. They are racing which one will change the resource
first. The end result depends on the winner of the race.

![Data race](/pics/data_race.png)

In `{%raw%}arguments.cpp{%endraw%}`, threads `{%raw%}t1{%endraw%}` and
`{%raw%}t2{%endraw%}` are racing for the resource
`{%raw%}std::cout{%endraw%}`. The threads change internal state of
`{%raw%}std::cout{%endraw%}` -- the data which will be printed to the standard
output.

One might argue that this is not a problematic thing: the program still prints
everything on the screen, just the ordering is a bit off. But the data race can
be more dangerous than "messy printing". Let's look at the
`{%raw%}dataRace.cpp{%endraw%}`.

{% highlight c++ %}
#include <iostream>
#include <thread>
#include <vector>

void countEven(const std::vector<int>& numbers,
               int& numEven)
{
    for (const auto number : numbers)
    {
        if (number % 2 == 0)
        {
            numEven++;
        }
    }
}

int main()
{
    std::vector<int> numbers1 = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::vector<int> numbers2 = {11, 12, 13, 14, 15, 16, 17, 18};
    
    int numEven = 0;
    
    std::thread t1(countEven, std::ref(numbers1), std::ref(numEven));
    std::thread t2(countEven, std::ref(numbers2), std::ref(numEven));    

    t1.join();
    t2.join();
    
    std::cout << "Total number of even numbers is: " << n << std::endl;
    return 0;
}
{% endhighlight %}

The function `{%raw%}countEven{%endraw%}` gets a vector of ints and a reference
`{%raw%}numEven{%endraw%}` which represents the number of odd numbers so
far. The function loops through the vector and for each odd entry increments
the `{%raw%}numEven{%endraw%}`. The main function constructs two
vectors of ints and initializes `{%raw%}numEven{%endraw%}` to zero. Then for
each vector, the `{%raw%}countEven{%endraw%}` function concurrently counts the number
of odd entries.

Let's test the program. Here are two of my several experiments:

{% highlight bash %}
$ ./dataRace 
Total number of even numbers is: 9
$ ./dataRace 
Total number of even numbers is: 5
{% endhighlight %}

Nooo, the results are different! This happens because of a data race. Two threads
are racing for the `{%raw%}numEven{%endraw%}` and when both are tying to
increment it at the same time, strange stuff happens and the outcome is
unpredictable. This is much worse than the example from the previous blog post. 

There are several solutions. We could count separately in every thread and then
sum results together in the main thread. But if we would still like to count with
one variable, we must use a mutex.

The solution: mutex
-------------------

The mutex stands for **mut**ual **ex**clusion and puts a restriction on access of a
resource. Before accessing the resource, you lock the mutex, and after you are
done accessing, you unlock the mutex. A thread, which wants to lock a mutex, has
two possibilities

* If the mutex is not locked, the thread locks it.
* If the mutex is locked, the thread waits until the mutex is unlocked and then
  locks it.

A resource, which is guarded by a mutex, can not be a reason for a data
race. Only one thread at a time can access the resource, all the other threads
must wait for the unlock of the mutex.

Let's make `{%raw%}dataRace.cpp{%endraw%}` data race free with a mutex:

{% highlight c++ %}
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>

std::mutex increment;

void countEven(const std::vector<int>& numbers,
               int& numEven)
{
    for (const auto n : numbers)
    {
        if (n % 2 == 0)
        {
            increment.lock();
            numEven++;
            increment.unlock();
        }
    }
}

int main()
{ ...
{% endhighlight %}

In order to use mutexes, we must include them into the program with
`{%raw%}#include <mutex>{%endraw%}` statement. We declare a global mutex, lock
it before incrementing and unlock it afterwards. The main function is
unchanged. This makes our program data race free. For an exercise, you can make
the `{%raw%}arguments.cpp{%endraw%}` from the previous blog post data race free.

Usually, we don't use global mutexes, but we group them and the protected
resource together in a class. This also makes the relation between the mutex and
the resource more apparent. 

A mutex looks like a very useful tool in concurrent programming. But, it is also
a source of problems. In the next blog post, we will explore common
problems associated with mutexes. 

Summary
-------

We defined and explored data races. One possible solution for data races are
mutexes.

In the next blog post, we will look more deeply into mutexes. 

Links:

* Source code: [dataRace.cpp](https://github.com/jakaspeh/concurrency/blob/master/dataRace.cpp)

* [std::mutex](http://en.cppreference.com/w/cpp/thread/mutex)






