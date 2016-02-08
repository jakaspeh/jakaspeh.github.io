---
layout: post
title:  "Difference between std::lock_guard and std::unique_lock"
date:   2016-02-08
categories: concurrency
---

One way of preventing data races between the threads is to use mutexes.

A mutex is usually associated with a resource. The thread, which locks the
mutex, has granted access to the resource. No other thread can then lock the
mutex because it is already locked (look figure below). Consequently, no other
thread has an access to the resource guarded by the locked mutex. This is the
mutual exclusion: only one thread has access to the resource at any given time.

![Mutex](/pics/mutex.png)

We already spoke about the problems which appear when using mutexes in our code:
remember [Mutex and deadlock](/blog/2016/01/deadlock.html). There, we introduced
the `{%raw%}std::lock_guard{%endraw%}` class. But when we synchronized threads
with a condition variable, we used similar class:
`{%raw%}std::unique_lock{%endraw%}`. What is the difference between these two
classes?

The difference
--------------

One of the differences between `{%raw%}std::lock_guard{%endraw%}` and
`{%raw%}std::unique_lock{%endraw%}` is that the programmer **is able** to unlock
`{%raw%}std::unique_lock{%endraw%}`, but she/he **is not able** to unlock
`{%raw%}std::lock_guard{%endraw%}`. Let's explain it in more detail. 

### std::lock_guard

If you have an object 

{% highlight c++ %}
std::lock_guard guard1(mutex);
{% endhighlight %}

then the constructor of `{%raw%}guard1{%endraw%}` locks the mutex. At the end of
`{%raw%}guard1{%endraw%}`'s life, the destructor unlocks the mutex. There is no
other possibility. In fact, the `{%raw%}std::lock_guard{%endraw%}` class doesn't
have any other member function.

### std::unique_lock

On the other hand, we have an object of `{%raw%}std::unique_lock{%endraw%}`.

{% highlight c++ %}
std::unique_lock guard2(mutex);
{% endhighlight %}

There are similarities with `{%raw%}std::lock_guard{%endraw%}` class. The
constructor of `{%raw%}guard2{%endraw%}` also locks the mutex and the destructor
of `{%raw%}guard2{%endraw%}` also unlocks the mutex. But the
`{%raw%}std::unique_lock{%endraw%}` has additional functionalities.

The programmer is able to unlock the mutex with the help of the guard object

{% highlight c++ %}
guard2.unlock();
{% endhighlight %}

This means that the programmer can unlock the mutex before the
`{%raw%}guard2{%endraw%}`'s life ends. After the mutex was unlocked, the
programmer can also lock it again

{% highlight c++ %}
guard2.lock();
{% endhighlight %}

We should mention that the `{%raw%}std::unique_lock{%endraw%}`  has also
some other member functions. You can look it up
[here](http://en.cppreference.com/w/cpp/thread/unique_lock).

When to use std::unique_lock ?
------------------------------

There are at least two reasons for using
`{%raw%}std::unique_lock{%endraw%}`. Sometimes we are forced to use it: other
functions require it as an input. And other times using
`{%raw%}std::unique_lock{%endraw%}` allows us to have more parallelizable code.


### Higher parallelization

Let's say that we have a long function. First part of the function accesses some
shared resource and the second part locally processes the resource. 

{% highlight c++ %}
std::vector< int > vector; // shared between threads

...

int function(...)
{
    ...
    Getting int from the shared vector.
    ...
       
    ...
    Long, complicated computation with int. 
    This part does not depend on the vector. 
    ... 

}
{% endhighlight %}

A mutex must be locked just in the first part of the function, because we access
the element of the vector. In the second part, the mutex doesn't need to be
locked anymore (because we don't access any shared variable).

{% highlight c++ %}
std::vector< int > vector; // shared between threads
std::mutex mutex; 

...

int function(...)
{
    ...
    std::unique_lock guard(mutex);
    Getting int from the shared vector.
    ...
       
    ...
    guard.unlock();
    Long, complicated computation with int. 
    This part does not depend on the vector. 
    ... 

}
{% endhighlight %}

In fact, it is preferable that the mutex is not locked in the second part,
because then other threads can lock it. In principle, we would like that the
locks last as little time as possible. This minimizes the time when threads are
waiting to get a lock on the mutex and not doing any useful work. We obtain more
parallelizable code.

### Using functions that requires std::unique_lock

In [Condition variable](/blog/2016/01/condition-variable.html), we had to use
the `{%raw%}std::unique_lock{%endraw%}`, because
`{%raw%}std::condition_variable::wait(...){%endraw%}` requires
`{%raw%}std::unique_lock{%endraw%}` as an input.

The `{%raw%}std::condition_variable::wait(...){%endraw%}` unlocks the mutex and
waits for the `{%raw%}std::condition_variable.notify_one(){%endraw%}` member
function call. Then, `{%raw%}wait(...){%endraw%}` reacquires the lock and
proceeds.

We recognize that `{%raw%}wait(...){%endraw%}` member function requires
`{%raw%}std::unique_lock{%endraw%}`. The function can not use usual
`{%raw%}std::lock_guard{%endraw%}`, because it unlocks/locks the mutex.

When to use std::lock_guard ?
------------------------------

The `{%raw%}std::unique_lock{%endraw%}` has all of the functionalities of the
`{%raw%}std::lock_guard{%endraw%}`. Everything which is possible to do with
`{%raw%}std::lock_guard{%endraw%}` is also possible to do with
`{%raw%}std::unique_lock{%endraw%}`. So, when should we use
`{%raw%}std::lock_guard{%endraw%}`?

The rule of thumb is to always use `{%raw%}std::lock_guard{%endraw%}`. But if we
need some higher level functionalities, which are available by
`{%raw%}std::unique_lock{%endraw%}`, then we should use the
`{%raw%}std::unique_lock{%endraw%}`.

Summary
-------

We learned the differences between the `{%raw%}std::lock_guard{%endraw%}` and
the `{%raw%}std::unique_lock{%endraw%}`. We also listed some situations where we
should use the `{%raw%}std::unique_lock{%endraw%}`.


Links:

* [std::lock_guard](http://en.cppreference.com/w/cpp/thread/lock_guard)
* [std::unique_lock](http://en.cppreference.com/w/cpp/thread/unique_lock)